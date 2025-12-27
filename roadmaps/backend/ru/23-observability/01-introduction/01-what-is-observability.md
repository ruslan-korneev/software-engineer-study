# Что такое Observability

## Введение

**Observability** (наблюдаемость) — это способность понять внутреннее состояние системы, анализируя её внешние выходные данные. Термин пришёл из теории управления, где он означает возможность определить внутреннее состояние системы по её внешним выводам.

В контексте программного обеспечения observability позволяет ответить на вопросы:
- Что происходит внутри системы прямо сейчас?
- Почему система ведёт себя определённым образом?
- Где возникла проблема?
- Как это влияет на пользователей?

## Observability vs Monitoring

Важно понимать различие между этими понятиями:

| Аспект | Monitoring | Observability |
|--------|------------|---------------|
| Подход | Реактивный | Проактивный |
| Вопросы | "Что сломалось?" | "Почему сломалось?" |
| Данные | Предопределённые метрики | Произвольные запросы |
| Проблемы | Известные неизвестные | Неизвестные неизвестные |
| Контекст | Ограниченный | Богатый, с корреляцией |

**Monitoring** отвечает на заранее известные вопросы: "Работает ли сервис?", "Какая нагрузка на CPU?".

**Observability** позволяет исследовать систему и отвечать на вопросы, которые вы не предвидели заранее.

```python
# Пример: простой мониторинг vs observability

# Monitoring: заранее определённая метрика
cpu_usage = get_cpu_usage()
if cpu_usage > 80:
    send_alert("CPU usage high!")

# Observability: богатый контекст для исследования
def process_request(request):
    with tracer.start_span("process_request") as span:
        span.set_attribute("user_id", request.user_id)
        span.set_attribute("request_type", request.type)
        span.set_attribute("payload_size", len(request.payload))

        # Теперь можно спросить: "Какие запросы от user_id=123
        # были медленными за последний час и почему?"
        result = handle_request(request)

        span.set_attribute("response_code", result.code)
        return result
```

## Три столпа Observability

Observability строится на трёх ключевых типах телеметрических данных:

### 1. Метрики (Metrics)

Метрики — это числовые измерения, агрегированные во времени.

```python
from prometheus_client import Counter, Histogram, Gauge

# Counter — только растущее значение
requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

# Histogram — распределение значений
request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint'],
    buckets=[0.01, 0.05, 0.1, 0.5, 1.0, 5.0]
)

# Gauge — значение, которое может расти и падать
active_connections = Gauge(
    'active_connections',
    'Number of active connections'
)

# Использование
def handle_request(method, endpoint):
    with request_duration.labels(method=method, endpoint=endpoint).time():
        active_connections.inc()
        try:
            result = process()
            requests_total.labels(
                method=method,
                endpoint=endpoint,
                status=200
            ).inc()
            return result
        finally:
            active_connections.dec()
```

**Характеристики метрик:**
- Низкая стоимость хранения (агрегированные данные)
- Быстрые запросы
- Хороши для алертинга и дашбордов
- Потеря детализации при агрегации

### 2. Логи (Logs)

Логи — это записи дискретных событий с временной меткой.

```python
import structlog
import json

# Структурированное логирование
logger = structlog.get_logger()

def process_order(order_id: str, user_id: str, items: list):
    logger.info(
        "order_processing_started",
        order_id=order_id,
        user_id=user_id,
        items_count=len(items),
        total_amount=sum(item.price for item in items)
    )

    try:
        result = payment_service.charge(order_id)
        logger.info(
            "payment_processed",
            order_id=order_id,
            payment_id=result.payment_id,
            status="success"
        )
    except PaymentError as e:
        logger.error(
            "payment_failed",
            order_id=order_id,
            error_type=type(e).__name__,
            error_message=str(e),
            retry_possible=e.is_retryable
        )
        raise
```

**Пример структурированного лога (JSON):**
```json
{
  "timestamp": "2024-01-15T10:30:45.123Z",
  "level": "info",
  "event": "order_processing_started",
  "order_id": "ord_123456",
  "user_id": "usr_789",
  "items_count": 3,
  "total_amount": 150.00,
  "service": "order-service",
  "trace_id": "abc123def456"
}
```

**Характеристики логов:**
- Высокая детализация
- Значительный объём данных
- Сложность анализа без структурирования
- Контекст конкретного события

### 3. Трейсы (Traces)

Трейсы отслеживают путь запроса через распределённую систему.

```python
from opentelemetry import trace
from opentelemetry.trace import Status, StatusCode

tracer = trace.get_tracer(__name__)

def handle_api_request(request):
    # Создаём корневой span
    with tracer.start_as_current_span("handle_api_request") as span:
        span.set_attribute("http.method", request.method)
        span.set_attribute("http.url", request.url)
        span.set_attribute("user.id", request.user_id)

        # Валидация — дочерний span
        with tracer.start_as_current_span("validate_request") as validation_span:
            if not validate(request):
                validation_span.set_status(Status(StatusCode.ERROR))
                span.set_attribute("error", True)
                return error_response()

        # Обращение к БД — дочерний span
        with tracer.start_as_current_span("database_query") as db_span:
            db_span.set_attribute("db.system", "postgresql")
            db_span.set_attribute("db.statement", "SELECT * FROM users WHERE id = ?")
            user = fetch_user(request.user_id)

        # Вызов внешнего сервиса — дочерний span
        with tracer.start_as_current_span("external_service_call") as ext_span:
            ext_span.set_attribute("peer.service", "payment-service")
            result = call_payment_service(user, request)

        return result
```

**Структура трейса:**
```
Trace ID: abc123
│
├── Span: handle_api_request (100ms)
│   ├── Span: validate_request (5ms)
│   ├── Span: database_query (20ms)
│   │   └── attributes: db.system=postgresql
│   └── Span: external_service_call (70ms)
│       └── attributes: peer.service=payment-service
```

**Характеристики трейсов:**
- Показывают причинно-следственные связи
- Визуализируют латентность каждого этапа
- Связывают события между сервисами
- Помогают найти узкие места

## Корреляция данных

Настоящая сила observability — в связывании всех трёх типов данных:

```python
import uuid
from contextvars import ContextVar

# Единый trace_id для корреляции
trace_id_var: ContextVar[str] = ContextVar('trace_id')

class ObservabilityMiddleware:
    def __call__(self, request):
        # Генерируем или получаем trace_id
        trace_id = request.headers.get('X-Trace-ID') or str(uuid.uuid4())
        trace_id_var.set(trace_id)

        # Теперь все метрики, логи и span'ы используют один trace_id
        with tracer.start_as_current_span("request") as span:
            span.set_attribute("trace_id", trace_id)

            logger.info(
                "request_started",
                trace_id=trace_id,
                path=request.path
            )

            request_counter.labels(
                path=request.path,
                trace_id=trace_id  # В реальности — через exemplars
            ).inc()

            return self.handle(request)
```

## Best Practices

### 1. Инструментируйте заранее
```python
# Хорошо: инструментация встроена в код
class OrderService:
    def __init__(self, tracer, logger, metrics):
        self.tracer = tracer
        self.logger = logger
        self.order_counter = metrics.counter("orders_created")

    def create_order(self, order_data):
        with self.tracer.start_span("create_order") as span:
            span.set_attributes(order_data.to_attributes())
            self.logger.info("creating_order", **order_data.to_dict())
            result = self._create(order_data)
            self.order_counter.inc()
            return result
```

### 2. Используйте семантические конвенции
```python
# OpenTelemetry Semantic Conventions
span.set_attribute("http.method", "POST")
span.set_attribute("http.status_code", 200)
span.set_attribute("http.url", "/api/orders")
span.set_attribute("db.system", "postgresql")
span.set_attribute("db.operation", "INSERT")
```

### 3. Добавляйте бизнес-контекст
```python
span.set_attribute("order.type", "subscription")
span.set_attribute("order.value", 99.99)
span.set_attribute("customer.tier", "premium")
```

### 4. Используйте уровни логирования правильно
```python
logger.debug("Детали для отладки")  # Только в development
logger.info("Значимые бизнес-события")  # Нормальный поток
logger.warning("Потенциальные проблемы")  # Требует внимания
logger.error("Ошибки, требующие действий")  # Нужна реакция
logger.critical("Критические сбои")  # Немедленная реакция
```

## Частые ошибки

### 1. Слишком много или слишком мало данных
```python
# Плохо: логируем всё подряд
for item in items:
    logger.info(f"Processing item {item}")  # Миллионы логов!

# Хорошо: агрегируем и логируем значимое
logger.info(
    "batch_processed",
    items_count=len(items),
    success_count=success,
    error_count=errors
)
```

### 2. Отсутствие контекста
```python
# Плохо: бесполезный лог
logger.error("Error occurred")

# Хорошо: полный контекст
logger.error(
    "payment_processing_failed",
    order_id=order.id,
    user_id=user.id,
    error_type="timeout",
    retry_attempt=3,
    exc_info=True
)
```

### 3. Высокая кардинальность меток
```python
# Плохо: user_id как метка = миллионы временных рядов
requests.labels(user_id=request.user_id).inc()

# Хорошо: user_id в span/log, не в метках
requests.labels(endpoint=request.path).inc()
span.set_attribute("user_id", request.user_id)
```

## Инструменты Observability

### Сбор и хранение
- **Prometheus** — метрики
- **Loki, Elasticsearch** — логи
- **Jaeger, Tempo, Zipkin** — трейсы
- **OpenTelemetry** — унифицированный сбор

### Визуализация
- **Grafana** — дашборды для всех типов данных
- **Kibana** — анализ логов

### Коммерческие решения
- Datadog, New Relic, Dynatrace — all-in-one платформы
- Honeycomb — продвинутая работа с трейсами

## Заключение

Observability — это не просто набор инструментов, а культура и подход к разработке. Ключевые принципы:

1. **Инструментируйте код с самого начала**
2. **Связывайте все типы данных через trace_id**
3. **Добавляйте богатый контекст**
4. **Используйте стандарты (OpenTelemetry)**
5. **Думайте о том, какие вопросы придётся задавать**

В следующих разделах мы подробно рассмотрим каждый из трёх столпов observability и практические инструменты для их реализации.
