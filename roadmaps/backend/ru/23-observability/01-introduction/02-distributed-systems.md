# Observability в распределённых системах

## Введение

Распределённые системы — это системы, где компоненты расположены на разных машинах и взаимодействуют через сеть. Microservices, serverless-функции, контейнеризованные приложения — всё это примеры распределённых архитектур.

**Почему observability критически важна для распределённых систем?**

В монолите один запрос обрабатывается в одном процессе. В распределённой системе запрос может пройти через десятки сервисов, очередей сообщений и баз данных. Без правильной observability понять, что происходит — практически невозможно.

## Проблемы распределённых систем

### 1. Распределённая латентность

```
Пользователь → API Gateway → Auth Service → User Service → Order Service → Payment Service → DB
                 ↓              ↓              ↓              ↓                ↓
              5ms           50ms          30ms          100ms            200ms

Общее время: 385ms — но где проблема?
```

```python
from opentelemetry import trace
from opentelemetry.propagate import inject, extract

tracer = trace.get_tracer(__name__)

# API Gateway — начало цепочки
async def api_gateway_handler(request):
    with tracer.start_as_current_span("api_gateway") as span:
        span.set_attribute("http.method", request.method)
        span.set_attribute("http.url", request.url)

        # Передаём контекст трейса в заголовки
        headers = {}
        inject(headers)

        # Вызываем Auth Service с контекстом
        auth_response = await http_client.post(
            "http://auth-service/validate",
            headers=headers
        )

        if auth_response.status == 200:
            # Вызываем User Service
            user_response = await http_client.get(
                f"http://user-service/users/{request.user_id}",
                headers=headers
            )
            return user_response

# Auth Service — получаем и продолжаем трейс
async def auth_service_handler(request):
    # Извлекаем контекст из заголовков
    context = extract(request.headers)

    with tracer.start_as_current_span(
        "auth_validate",
        context=context
    ) as span:
        span.set_attribute("auth.method", "jwt")
        token = request.headers.get("Authorization")
        result = validate_token(token)
        span.set_attribute("auth.valid", result.is_valid)
        return result
```

### 2. Частичные сбои

В распределённой системе отдельные компоненты могут отказывать независимо:

```python
from opentelemetry import trace
from opentelemetry.trace import Status, StatusCode
import structlog

logger = structlog.get_logger()
tracer = trace.get_tracer(__name__)

class OrderService:
    async def create_order(self, order_data):
        with tracer.start_as_current_span("create_order") as span:
            order_id = generate_order_id()
            span.set_attribute("order.id", order_id)

            # Шаг 1: Создаём заказ
            try:
                await self.order_repo.save(order_data)
                span.add_event("order_saved")
            except DatabaseError as e:
                span.set_status(Status(StatusCode.ERROR, "DB error"))
                span.record_exception(e)
                raise

            # Шаг 2: Резервируем товар — может упасть!
            try:
                with tracer.start_as_current_span("reserve_inventory") as inv_span:
                    await self.inventory_client.reserve(order_data.items)
                    inv_span.set_attribute("reservation.status", "success")
            except InventoryServiceUnavailable as e:
                span.add_event("inventory_service_unavailable")
                logger.warning(
                    "inventory_reservation_failed",
                    order_id=order_id,
                    error=str(e),
                    fallback="async_reservation"
                )
                # Fallback: ставим в очередь для повторной попытки
                await self.queue.publish("inventory_reservations", order_data)

            # Шаг 3: Отправляем уведомление — некритично
            try:
                with tracer.start_as_current_span("send_notification") as notif_span:
                    await self.notification_client.send(order_data.user_id)
            except Exception as e:
                # Логируем, но не падаем
                logger.error(
                    "notification_failed",
                    order_id=order_id,
                    error=str(e)
                )

            return order_id
```

### 3. Асинхронные взаимодействия

Очереди сообщений и события усложняют отслеживание:

```python
from opentelemetry import trace
from opentelemetry.propagate import inject, extract
import json

tracer = trace.get_tracer(__name__)

# Producer: отправляем сообщение с контекстом трейса
async def publish_order_event(order):
    with tracer.start_as_current_span("publish_order_created") as span:
        span.set_attribute("messaging.system", "rabbitmq")
        span.set_attribute("messaging.destination", "orders")
        span.set_attribute("order.id", order.id)

        # Внедряем контекст трейса в заголовки сообщения
        headers = {}
        inject(headers)

        message = {
            "event": "order_created",
            "order_id": order.id,
            "data": order.to_dict(),
            "trace_context": headers  # Сохраняем для consumer'а
        }

        await rabbitmq.publish(
            exchange="orders",
            routing_key="order.created",
            body=json.dumps(message),
            headers=headers
        )

# Consumer: восстанавливаем контекст трейса
async def handle_order_created(message):
    # Извлекаем контекст из сообщения
    trace_context = message.get("trace_context", {})
    parent_context = extract(trace_context)

    # Создаём span как продолжение исходного трейса
    with tracer.start_as_current_span(
        "process_order_created",
        context=parent_context,
        kind=trace.SpanKind.CONSUMER
    ) as span:
        span.set_attribute("messaging.system", "rabbitmq")
        span.set_attribute("messaging.operation", "receive")
        span.set_attribute("order.id", message["order_id"])

        # Время в очереди
        enqueue_time = message.get("enqueued_at")
        if enqueue_time:
            queue_latency = time.time() - enqueue_time
            span.set_attribute("messaging.queue_latency_ms", queue_latency * 1000)

        await process_order(message["data"])
```

## Distributed Tracing: основа observability в распределённых системах

### Концепции

**Trace** — полный путь запроса через систему.
**Span** — единица работы внутри трейса (вызов функции, HTTP-запрос, запрос к БД).
**Context Propagation** — передача контекста трейса между сервисами.

```
Trace ID: trace-abc-123
├── Span: API Gateway (span-1)
│   └── Parent: none (root span)
│
├── Span: Auth Service (span-2)
│   └── Parent: span-1
│
├── Span: User Service (span-3)
│   └── Parent: span-1
│   │
│   ├── Span: Cache lookup (span-4)
│   │   └── Parent: span-3
│   │
│   └── Span: Database query (span-5)
│       └── Parent: span-3
│
└── Span: Order Service (span-6)
    └── Parent: span-1
```

### Propagation форматы

```python
# W3C Trace Context — стандарт
# Headers:
# traceparent: 00-trace_id-span_id-flags
# tracestate: vendor1=value1,vendor2=value2

# Пример traceparent:
# 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
#  │  │                                 │                  │
#  │  │                                 │                  └─ Flags (01 = sampled)
#  │  │                                 └─ Parent Span ID (16 hex chars)
#  │  └─ Trace ID (32 hex chars)
#  └─ Version

from opentelemetry.propagate import set_global_textmap
from opentelemetry.propagators.composite import CompositePropagator
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator
from opentelemetry.baggage.propagation import W3CBaggagePropagator

# Настройка propagator'ов
set_global_textmap(
    CompositePropagator([
        TraceContextTextMapPropagator(),
        W3CBaggagePropagator()
    ])
)
```

### Baggage — передача данных между сервисами

```python
from opentelemetry import baggage
from opentelemetry.baggage.propagation import W3CBaggagePropagator

# Устанавливаем baggage в начале цепочки
def handle_request(request):
    ctx = baggage.set_baggage("user.tier", "premium")
    ctx = baggage.set_baggage("request.region", "eu-west")

    with tracer.start_as_current_span("handle_request", context=ctx):
        # Baggage будет передан во все последующие сервисы
        call_downstream_services()

# В любом downstream-сервисе
def process_in_downstream():
    # Получаем baggage из контекста
    user_tier = baggage.get_baggage("user.tier")
    region = baggage.get_baggage("request.region")

    # Используем для принятия решений
    if user_tier == "premium":
        use_fast_path()
```

## Корреляция данных в распределённой системе

### Единый Trace ID во всех типах телеметрии

```python
import structlog
from opentelemetry import trace
from prometheus_client import Counter, Histogram

# Настраиваем логгер с trace context
def add_trace_context(logger, method_name, event_dict):
    span = trace.get_current_span()
    if span:
        ctx = span.get_span_context()
        event_dict["trace_id"] = format(ctx.trace_id, "032x")
        event_dict["span_id"] = format(ctx.span_id, "016x")
    return event_dict

structlog.configure(
    processors=[
        add_trace_context,
        structlog.processors.JSONRenderer()
    ]
)

logger = structlog.get_logger()

# Метрики с exemplars (Prometheus)
request_duration = Histogram(
    "request_duration_seconds",
    "Request duration",
    ["service", "endpoint"]
)

def handle_request():
    span = trace.get_current_span()
    trace_id = format(span.get_span_context().trace_id, "032x")

    with request_duration.labels(
        service="order-service",
        endpoint="/orders"
    ).time():
        # Exemplar позволяет связать метрику с конкретным trace_id
        # (требует поддержки в Prometheus/Grafana)

        logger.info(
            "processing_request",
            endpoint="/orders"
            # trace_id добавится автоматически
        )

        result = process()
        return result
```

### Service Mesh и Observability

Service mesh (Istio, Linkerd) автоматически добавляет observability:

```yaml
# Istio автоматически:
# 1. Собирает метрики (request count, latency, errors)
# 2. Propagates trace headers
# 3. Логирует access logs

# Пример Istio метрик
istio_requests_total{
  source_workload="frontend",
  destination_workload="orders",
  response_code="200"
}

istio_request_duration_milliseconds_bucket{
  source_workload="frontend",
  destination_workload="orders",
  le="100"
}
```

## Паттерны Observability для распределённых систем

### 1. Health Checks с деталями

```python
from fastapi import FastAPI
from enum import Enum

app = FastAPI()

class HealthStatus(Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    UNHEALTHY = "unhealthy"

@app.get("/health")
async def health_check():
    checks = {}
    overall_status = HealthStatus.HEALTHY

    # Проверяем базу данных
    try:
        await db.execute("SELECT 1")
        checks["database"] = {"status": "healthy", "latency_ms": 5}
    except Exception as e:
        checks["database"] = {"status": "unhealthy", "error": str(e)}
        overall_status = HealthStatus.UNHEALTHY

    # Проверяем Redis
    try:
        await redis.ping()
        checks["cache"] = {"status": "healthy", "latency_ms": 1}
    except Exception as e:
        checks["cache"] = {"status": "degraded", "error": str(e)}
        if overall_status == HealthStatus.HEALTHY:
            overall_status = HealthStatus.DEGRADED

    # Проверяем downstream-сервисы
    try:
        resp = await http_client.get(
            "http://payment-service/health",
            timeout=2.0
        )
        checks["payment_service"] = {
            "status": "healthy" if resp.status == 200 else "degraded"
        }
    except Exception as e:
        checks["payment_service"] = {"status": "unknown", "error": str(e)}

    return {
        "status": overall_status.value,
        "checks": checks,
        "version": "1.2.3",
        "timestamp": datetime.utcnow().isoformat()
    }
```

### 2. Circuit Breaker с метриками

```python
from circuitbreaker import circuit
from prometheus_client import Counter, Gauge

circuit_state = Gauge(
    "circuit_breaker_state",
    "Circuit breaker state (0=closed, 1=open, 2=half-open)",
    ["service"]
)

circuit_failures = Counter(
    "circuit_breaker_failures_total",
    "Circuit breaker failure count",
    ["service", "error_type"]
)

class ObservableCircuitBreaker:
    def __init__(self, service_name: str, failure_threshold: int = 5):
        self.service_name = service_name
        self.failure_threshold = failure_threshold
        self.failures = 0
        self.state = "closed"

    async def call(self, func, *args, **kwargs):
        if self.state == "open":
            circuit_state.labels(service=self.service_name).set(1)
            raise CircuitOpenError(f"Circuit for {self.service_name} is open")

        try:
            with tracer.start_as_current_span(f"circuit_call_{self.service_name}") as span:
                span.set_attribute("circuit.state", self.state)
                result = await func(*args, **kwargs)
                self.on_success()
                return result
        except Exception as e:
            self.on_failure(e)
            raise

    def on_success(self):
        self.failures = 0
        if self.state == "half-open":
            self.state = "closed"
            circuit_state.labels(service=self.service_name).set(0)
            logger.info(
                "circuit_closed",
                service=self.service_name
            )

    def on_failure(self, error):
        self.failures += 1
        circuit_failures.labels(
            service=self.service_name,
            error_type=type(error).__name__
        ).inc()

        if self.failures >= self.failure_threshold:
            self.state = "open"
            circuit_state.labels(service=self.service_name).set(1)
            logger.warning(
                "circuit_opened",
                service=self.service_name,
                failures=self.failures
            )
```

### 3. Retry с Observability

```python
from tenacity import retry, stop_after_attempt, wait_exponential

def log_retry(retry_state):
    span = trace.get_current_span()
    span.add_event(
        "retry_attempt",
        {
            "attempt": retry_state.attempt_number,
            "wait_time": retry_state.next_action.sleep if retry_state.next_action else 0
        }
    )

    logger.warning(
        "retry_attempt",
        attempt=retry_state.attempt_number,
        function=retry_state.fn.__name__,
        error=str(retry_state.outcome.exception()) if retry_state.outcome else None
    )

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    before_sleep=log_retry
)
async def call_external_service(data):
    with tracer.start_as_current_span("external_service_call") as span:
        span.set_attribute("retry.enabled", True)
        response = await http_client.post(
            "http://external-service/api",
            json=data
        )
        if response.status >= 500:
            raise ExternalServiceError(response.status)
        return response
```

## Sampling стратегии

В высоконагруженных системах нельзя собирать все трейсы:

```python
from opentelemetry.sdk.trace.sampling import (
    TraceIdRatioBased,
    ParentBased,
    ALWAYS_ON,
    ALWAYS_OFF
)

# 1. Probabilistic sampling — 10% запросов
sampler = TraceIdRatioBased(0.1)

# 2. Parent-based — следуем решению родителя
sampler = ParentBased(root=TraceIdRatioBased(0.1))

# 3. Adaptive sampling — умный sampling
class AdaptiveSampler:
    def __init__(self, target_rate: float = 100):  # traces per second
        self.target_rate = target_rate
        self.current_rate = 0
        self.window_start = time.time()
        self.traces_in_window = 0

    def should_sample(self, span_context, trace_id, name, attributes):
        # Всегда сэмплируем ошибки
        if attributes.get("error"):
            return True

        # Всегда сэмплируем медленные запросы
        if attributes.get("duration_hint_ms", 0) > 1000:
            return True

        # Rate limiting для остальных
        current_time = time.time()
        if current_time - self.window_start > 1.0:
            self.current_rate = self.traces_in_window
            self.traces_in_window = 0
            self.window_start = current_time

        if self.traces_in_window < self.target_rate:
            self.traces_in_window += 1
            return True

        return False
```

## Best Practices

### 1. Именование span'ов

```python
# Плохо: слишком общее
span = tracer.start_span("process")
span = tracer.start_span("handle")

# Хорошо: конкретное и понятное
span = tracer.start_span("OrderService.create_order")
span = tracer.start_span("HTTP GET /api/users/{id}")
span = tracer.start_span("PostgreSQL SELECT users")
```

### 2. Атрибуты span'ов

```python
# Используйте семантические конвенции OpenTelemetry
span.set_attribute("http.method", "POST")
span.set_attribute("http.url", "/api/orders")
span.set_attribute("http.status_code", 201)
span.set_attribute("http.request_content_length", 1024)

span.set_attribute("db.system", "postgresql")
span.set_attribute("db.name", "orders")
span.set_attribute("db.operation", "INSERT")
span.set_attribute("db.statement", "INSERT INTO orders...")

span.set_attribute("rpc.system", "grpc")
span.set_attribute("rpc.service", "OrderService")
span.set_attribute("rpc.method", "CreateOrder")
```

### 3. Обработка ошибок

```python
try:
    result = await process()
except Exception as e:
    span.set_status(Status(StatusCode.ERROR, str(e)))
    span.record_exception(e)

    # Добавляем контекст для отладки
    span.set_attribute("error.type", type(e).__name__)
    span.set_attribute("error.message", str(e))

    logger.error(
        "processing_failed",
        error_type=type(e).__name__,
        error_message=str(e),
        exc_info=True
    )
    raise
```

## Частые ошибки

### 1. Потеря контекста между сервисами

```python
# Плохо: контекст не передаётся
async def call_service():
    response = await http.get("http://other-service/api")  # Trace обрывается!

# Хорошо: передаём контекст
async def call_service():
    headers = {}
    inject(headers)  # Внедряем trace context
    response = await http.get(
        "http://other-service/api",
        headers=headers
    )
```

### 2. Слишком много span'ов

```python
# Плохо: span на каждую мелкую операцию
for item in items:
    with tracer.start_span(f"process_item_{item.id}"):  # 10000 spans!
        process(item)

# Хорошо: агрегируем
with tracer.start_span("process_items_batch") as span:
    span.set_attribute("batch.size", len(items))
    for item in items:
        process(item)
    span.set_attribute("batch.processed", len(items))
```

### 3. Чувствительные данные в span'ах

```python
# Плохо: PII в атрибутах
span.set_attribute("user.email", "john@example.com")
span.set_attribute("user.ssn", "123-45-6789")

# Хорошо: только ID и хешированные данные
span.set_attribute("user.id", user.id)
span.set_attribute("user.email_hash", hash_pii(user.email))
```

## Инструменты для распределённых систем

### Трейсинг
- **Jaeger** — open-source, хорошо масштабируется
- **Zipkin** — простой, легковесный
- **Tempo** — от Grafana, интеграция с Loki и Prometheus
- **AWS X-Ray** — для AWS-инфраструктуры

### Унифицированный сбор
- **OpenTelemetry** — стандарт индустрии, vendor-neutral
- **OpenTelemetry Collector** — агрегация и маршрутизация телеметрии

### Service Mesh
- **Istio** — полнофункциональный, Envoy-based
- **Linkerd** — легковесный, простой
- **Consul Connect** — от HashiCorp

## Заключение

Observability в распределённых системах требует:

1. **Distributed Tracing** — отслеживание запросов через все сервисы
2. **Context Propagation** — передача trace context между компонентами
3. **Корреляция** — связывание логов, метрик и трейсов
4. **Sampling** — умный отбор данных для хранения
5. **Стандартизация** — использование OpenTelemetry и семантических конвенций

Без observability распределённая система превращается в "чёрный ящик", где поиск проблем становится угадыванием. С правильной observability вы можете:

- Найти причину проблемы за минуты, а не часы
- Понять, какой сервис влияет на латентность
- Отследить путь конкретного запроса через всю систему
- Выявить деградацию до того, как она повлияет на пользователей
