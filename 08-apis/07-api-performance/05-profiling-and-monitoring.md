# Profiling and Monitoring

## Введение

Profiling (профилирование) и Monitoring (мониторинг) — это практики анализа производительности приложения для выявления узких мест, отслеживания метрик и обеспечения стабильной работы системы.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Profiling vs Monitoring                      │
├─────────────────────────────────────────────────────────────────┤
│ Profiling:                     │ Monitoring:                    │
│ - Детальный анализ             │ - Непрерывное наблюдение       │
│ - Во время разработки/тестов   │ - В production                 │
│ - Высокий overhead             │ - Низкий overhead              │
│ - Поиск bottlenecks            │ - Отслеживание состояния       │
│ - Одноразовые сессии           │ - 24/7 сбор данных             │
└─────────────────────────────────────────────────────────────────┘
```

## Python Profiling

### cProfile — CPU Profiling

```python
import cProfile
import pstats
from io import StringIO
from functools import wraps
from typing import Callable

# Базовое использование cProfile
def profile_function():
    profiler = cProfile.Profile()
    profiler.enable()

    # Код для профилирования
    result = expensive_operation()

    profiler.disable()

    # Вывод статистики
    stream = StringIO()
    stats = pstats.Stats(profiler, stream=stream)
    stats.sort_stats('cumulative')
    stats.print_stats(20)  # Top 20 функций

    print(stream.getvalue())
    return result

# Декоратор для профилирования
def profile(output_file: str = None):
    """Декоратор для профилирования функции"""
    def decorator(func: Callable):
        @wraps(func)
        def wrapper(*args, **kwargs):
            profiler = cProfile.Profile()
            profiler.enable()

            try:
                result = func(*args, **kwargs)
                return result
            finally:
                profiler.disable()

                if output_file:
                    profiler.dump_stats(output_file)
                else:
                    stats = pstats.Stats(profiler)
                    stats.sort_stats('cumulative')
                    stats.print_stats(10)

        return wrapper
    return decorator

# Использование
@profile(output_file='profile_output.prof')
def process_data(data: list):
    # Какая-то тяжёлая операция
    return [item * 2 for item in data]

# Анализ результатов
# python -m pstats profile_output.prof
# sort cumulative
# stats 20
```

### line_profiler — построчное профилирование

```python
# pip install line_profiler

# Использование с декоратором @profile
# Запуск: kernprof -l -v script.py

@profile  # Этот декоратор добавляется line_profiler
def slow_function():
    result = []
    for i in range(10000):
        result.append(i ** 2)  # Эта строка может быть медленной
    return sum(result)

# Программный вариант
from line_profiler import LineProfiler

def analyze_function(func, *args, **kwargs):
    """Анализ функции построчно"""
    lp = LineProfiler()
    lp.add_function(func)

    # Также профилируем вызываемые функции
    lp.add_function(helper_function)

    wrapped = lp(func)
    result = wrapped(*args, **kwargs)

    lp.print_stats()
    return result

# Пример вывода:
# Line #      Hits         Time  Per Hit   % Time  Line Contents
# ==============================================================
#      3         1          2.0      2.0      0.0  def slow_function():
#      4         1          1.0      1.0      0.0      result = []
#      5     10001      15234.0      1.5     45.2      for i in range(10000):
#      6     10000      18456.0      1.8     54.8          result.append(i ** 2)
#      7         1          3.0      3.0      0.0      return sum(result)
```

### memory_profiler — профилирование памяти

```python
# pip install memory_profiler

from memory_profiler import profile, memory_usage

@profile
def memory_intensive_function():
    """Функция для анализа использования памяти"""
    # Создаём большой список
    big_list = [i ** 2 for i in range(1000000)]

    # Создаём ещё один
    another_list = big_list.copy()

    # Удаляем первый
    del big_list

    return sum(another_list)

# Программный анализ
def analyze_memory(func, *args, **kwargs):
    """Анализ использования памяти"""
    mem_usage = memory_usage((func, args, kwargs), interval=0.1)

    return {
        'min_mb': min(mem_usage),
        'max_mb': max(mem_usage),
        'diff_mb': max(mem_usage) - min(mem_usage)
    }

# Отслеживание утечек памяти
import tracemalloc

def find_memory_leaks():
    """Поиск утечек памяти"""
    tracemalloc.start()

    # Выполняем код
    process_something()

    # Делаем snapshot
    snapshot = tracemalloc.take_snapshot()
    top_stats = snapshot.statistics('lineno')

    print("Top 10 memory allocations:")
    for stat in top_stats[:10]:
        print(stat)

    tracemalloc.stop()

# Сравнение snapshots для обнаружения утечек
def compare_memory_snapshots():
    tracemalloc.start()

    snapshot1 = tracemalloc.take_snapshot()

    # Выполняем операции
    data = load_data()
    process_data(data)
    del data  # Должна освободиться память

    snapshot2 = tracemalloc.take_snapshot()

    # Сравниваем
    top_stats = snapshot2.compare_to(snapshot1, 'lineno')

    print("Memory changes:")
    for stat in top_stats[:10]:
        print(stat)
```

### Async Profiling

```python
import asyncio
import time
from typing import Dict, List
from dataclasses import dataclass, field
from contextlib import asynccontextmanager

@dataclass
class AsyncProfiler:
    """Профилировщик для async кода"""

    timings: Dict[str, List[float]] = field(default_factory=dict)

    @asynccontextmanager
    async def measure(self, name: str):
        """Измерение времени выполнения async операции"""
        start = time.perf_counter()
        try:
            yield
        finally:
            elapsed = time.perf_counter() - start
            if name not in self.timings:
                self.timings[name] = []
            self.timings[name].append(elapsed)

    def get_stats(self) -> Dict[str, dict]:
        """Получение статистики"""
        import statistics

        result = {}
        for name, times in self.timings.items():
            result[name] = {
                'count': len(times),
                'total': sum(times),
                'mean': statistics.mean(times),
                'median': statistics.median(times),
                'min': min(times),
                'max': max(times),
                'stdev': statistics.stdev(times) if len(times) > 1 else 0
            }
        return result

profiler = AsyncProfiler()

async def fetch_data(url: str):
    async with profiler.measure('http_request'):
        # HTTP запрос
        async with httpx.AsyncClient() as client:
            response = await client.get(url)
            return response.json()

async def process_with_profiling():
    async with profiler.measure('db_query'):
        await db.fetch_all(query)

    async with profiler.measure('cache_lookup'):
        await cache.get(key)

    # Получаем статистику
    stats = profiler.get_stats()
    print(stats)
```

## Application Performance Monitoring (APM)

### Sentry Performance

```python
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration
from sentry_sdk.integrations.sqlalchemy import SqlalchemyIntegration
from sentry_sdk.integrations.redis import RedisIntegration

# Инициализация Sentry
sentry_sdk.init(
    dsn="https://xxx@sentry.io/xxx",
    integrations=[
        FastApiIntegration(),
        SqlalchemyIntegration(),
        RedisIntegration(),
    ],
    # Процент транзакций для трассировки
    traces_sample_rate=0.1,  # 10%
    # Профилирование
    profiles_sample_rate=0.1,
    environment="production",
)

# Ручное создание транзакций
from sentry_sdk import start_transaction, start_span

async def process_order(order_id: str):
    with start_transaction(op="task", name="process_order") as transaction:
        with start_span(op="db", description="fetch_order"):
            order = await db.get_order(order_id)

        with start_span(op="http", description="payment_api"):
            await process_payment(order)

        with start_span(op="db", description="update_order"):
            await db.update_order_status(order_id, "completed")

        # Добавляем контекст
        transaction.set_tag("order_type", order.type)
        transaction.set_data("order_total", order.total)
```

### OpenTelemetry

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.instrumentation.redis import RedisInstrumentor

# Настройка TracerProvider
provider = TracerProvider()
processor = BatchSpanProcessor(
    OTLPSpanExporter(endpoint="http://jaeger:4317")
)
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

# Автоматическая инструментация
FastAPIInstrumentor.instrument()
HTTPXClientInstrumentor().instrument()
SQLAlchemyInstrumentor().instrument(engine=engine)
RedisInstrumentor().instrument()

# Ручная инструментация
tracer = trace.get_tracer(__name__)

async def complex_operation(data: dict):
    with tracer.start_as_current_span("complex_operation") as span:
        span.set_attribute("data.size", len(data))

        with tracer.start_as_current_span("validate"):
            validated = validate_data(data)

        with tracer.start_as_current_span("process"):
            result = await process_data(validated)

        with tracer.start_as_current_span("save"):
            await save_result(result)

        span.set_attribute("result.count", len(result))
        return result
```

### Custom Metrics с Prometheus

```python
from prometheus_client import (
    Counter, Histogram, Gauge, Summary,
    generate_latest, CONTENT_TYPE_LATEST
)
from fastapi import FastAPI, Request
from starlette.responses import Response
import time

app = FastAPI()

# Определение метрик
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint'],
    buckets=[.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10]
)

REQUESTS_IN_PROGRESS = Gauge(
    'http_requests_in_progress',
    'Number of requests in progress',
    ['method', 'endpoint']
)

DB_QUERY_DURATION = Histogram(
    'db_query_duration_seconds',
    'Database query duration',
    ['query_type', 'table'],
    buckets=[.001, .005, .01, .025, .05, .1, .25, .5, 1]
)

CACHE_OPERATIONS = Counter(
    'cache_operations_total',
    'Cache operations',
    ['operation', 'result']  # result: hit, miss
)

# Middleware для сбора метрик
@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    method = request.method
    endpoint = request.url.path

    REQUESTS_IN_PROGRESS.labels(method=method, endpoint=endpoint).inc()

    start_time = time.perf_counter()

    try:
        response = await call_next(request)
        status = response.status_code
    except Exception as e:
        status = 500
        raise
    finally:
        duration = time.perf_counter() - start_time

        REQUEST_COUNT.labels(
            method=method,
            endpoint=endpoint,
            status=status
        ).inc()

        REQUEST_LATENCY.labels(
            method=method,
            endpoint=endpoint
        ).observe(duration)

        REQUESTS_IN_PROGRESS.labels(method=method, endpoint=endpoint).dec()

    return response

# Эндпоинт для Prometheus
@app.get("/metrics")
async def metrics():
    return Response(
        content=generate_latest(),
        media_type=CONTENT_TYPE_LATEST
    )

# Декоратор для метрик БД
def track_db_query(query_type: str, table: str):
    def decorator(func):
        async def wrapper(*args, **kwargs):
            start = time.perf_counter()
            try:
                return await func(*args, **kwargs)
            finally:
                duration = time.perf_counter() - start
                DB_QUERY_DURATION.labels(
                    query_type=query_type,
                    table=table
                ).observe(duration)
        return wrapper
    return decorator

@track_db_query("select", "users")
async def get_user(user_id: int):
    return await db.fetch_one(
        "SELECT * FROM users WHERE id = :id",
        {"id": user_id}
    )
```

## Distributed Tracing

### Jaeger Integration

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.resources import Resource
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.propagate import set_global_textmap
from opentelemetry.propagators.b3 import B3MultiFormat

# Конфигурация
resource = Resource.create({
    "service.name": "api-service",
    "service.version": "1.0.0",
    "deployment.environment": "production"
})

provider = TracerProvider(resource=resource)

jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger-agent",
    agent_port=6831,
)

provider.add_span_processor(BatchSpanProcessor(jaeger_exporter))
trace.set_tracer_provider(provider)

# B3 propagation для микросервисов
set_global_textmap(B3MultiFormat())

tracer = trace.get_tracer(__name__)

# Пример трейса через несколько сервисов
async def api_endpoint(request: Request):
    with tracer.start_as_current_span("api_request") as span:
        span.set_attribute("http.method", request.method)
        span.set_attribute("http.url", str(request.url))

        # Вызов другого сервиса
        async with httpx.AsyncClient() as client:
            # Заголовки трейсинга добавляются автоматически
            response = await client.post(
                "http://payment-service/process",
                json={"amount": 100}
            )

        span.set_attribute("payment.status", response.status_code)
        return response.json()
```

### Context Propagation

```python
from opentelemetry import context, propagate
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator
import httpx

propagator = TraceContextTextMapPropagator()

class TracingMiddleware:
    """Middleware для propagation контекста трейсинга"""

    async def __call__(self, request: Request, call_next):
        # Извлекаем контекст из входящих заголовков
        carrier = dict(request.headers)
        ctx = propagator.extract(carrier)

        # Устанавливаем контекст
        token = context.attach(ctx)

        try:
            with tracer.start_as_current_span(
                f"{request.method} {request.url.path}"
            ) as span:
                span.set_attribute("http.method", request.method)
                span.set_attribute("http.url", str(request.url))

                response = await call_next(request)

                span.set_attribute("http.status_code", response.status_code)
                return response
        finally:
            context.detach(token)

# Исходящие запросы
async def call_external_service(data: dict):
    """Передача контекста трейсинга в исходящие запросы"""
    headers = {}
    propagator.inject(headers)  # Добавляем trace headers

    async with httpx.AsyncClient() as client:
        response = await client.post(
            "http://external-service/api",
            json=data,
            headers=headers  # Передаём контекст
        )
        return response.json()
```

## Logging Best Practices

### Structured Logging

```python
import structlog
import logging
from typing import Any, Dict

# Конфигурация structlog
structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.JSONRenderer()
    ],
    wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
    context_class=dict,
    logger_factory=structlog.PrintLoggerFactory(),
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger()

# Использование
async def process_request(request_id: str, user_id: str):
    # Добавляем контекст, который будет во всех логах
    log = logger.bind(
        request_id=request_id,
        user_id=user_id
    )

    log.info("processing_started")

    try:
        result = await do_work()
        log.info("processing_completed", result_count=len(result))
        return result
    except Exception as e:
        log.error("processing_failed", error=str(e), exc_info=True)
        raise

# Вывод:
# {
#   "request_id": "abc-123",
#   "user_id": "user-456",
#   "event": "processing_completed",
#   "result_count": 42,
#   "level": "info",
#   "timestamp": "2024-01-15T10:30:00Z"
# }
```

### Request Logging Middleware

```python
import time
import uuid
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware

class RequestLoggingMiddleware(BaseHTTPMiddleware):
    """Middleware для логирования запросов"""

    async def dispatch(self, request: Request, call_next):
        request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
        start_time = time.perf_counter()

        # Привязываем контекст к логгеру
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(
            request_id=request_id,
            method=request.method,
            path=request.url.path,
            client_ip=request.client.host if request.client else None,
        )

        logger.info("request_started")

        try:
            response = await call_next(request)

            duration = time.perf_counter() - start_time
            logger.info(
                "request_completed",
                status_code=response.status_code,
                duration_ms=round(duration * 1000, 2)
            )

            # Добавляем request_id в ответ
            response.headers["X-Request-ID"] = request_id
            return response

        except Exception as e:
            duration = time.perf_counter() - start_time
            logger.error(
                "request_failed",
                error=str(e),
                duration_ms=round(duration * 1000, 2),
                exc_info=True
            )
            raise
```

### Correlation ID

```python
from contextvars import ContextVar
from typing import Optional

# Context variable для correlation ID
correlation_id_var: ContextVar[Optional[str]] = ContextVar(
    'correlation_id',
    default=None
)

class CorrelationIDMiddleware:
    """Middleware для управления Correlation ID"""

    HEADER_NAME = "X-Correlation-ID"

    async def __call__(self, request: Request, call_next):
        # Получаем или генерируем correlation ID
        correlation_id = request.headers.get(
            self.HEADER_NAME,
            str(uuid.uuid4())
        )

        # Устанавливаем в context var
        token = correlation_id_var.set(correlation_id)

        try:
            response = await call_next(request)
            response.headers[self.HEADER_NAME] = correlation_id
            return response
        finally:
            correlation_id_var.reset(token)

# Использование в любом месте кода
def get_correlation_id() -> str:
    return correlation_id_var.get() or str(uuid.uuid4())

# В логах
logger.info(
    "operation_completed",
    correlation_id=get_correlation_id()
)

# В исходящих запросах
async def call_service():
    async with httpx.AsyncClient() as client:
        return await client.get(
            "http://service/api",
            headers={"X-Correlation-ID": get_correlation_id()}
        )
```

## Alerting

### Prometheus Alerting Rules

```yaml
# prometheus/alerts.yml
groups:
  - name: api_alerts
    rules:
      # High Error Rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m])) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate: {{ $value | humanizePercentage }}"
          description: "Error rate is above 1% for the last 5 minutes"
          runbook_url: "https://wiki.example.com/runbooks/high-error-rate"

      # High Latency
      - alert: HighP99Latency
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
          ) > 1.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High P99 latency: {{ $value | humanizeDuration }}"
          description: "P99 latency is above 1 second"

      # Low Throughput
      - alert: LowThroughput
        expr: |
          sum(rate(http_requests_total[5m])) < 10
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Low throughput: {{ $value }} req/s"
          description: "Request rate is unusually low"

      # High Memory Usage
      - alert: HighMemoryUsage
        expr: |
          process_resident_memory_bytes / 1024 / 1024 > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage: {{ $value | humanize1024 }}B"

      # Database Connection Pool Exhaustion
      - alert: DBConnectionPoolExhausted
        expr: |
          db_pool_connections_used / db_pool_connections_max > 0.9
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "DB connection pool > 90% used"
```

### AlertManager Configuration

```yaml
# alertmanager.yml
global:
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alerts@example.com'
  slack_api_url: 'https://hooks.slack.com/services/xxx'

route:
  receiver: 'default'
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true

    - match:
        severity: critical
      receiver: 'slack-critical'

    - match:
        severity: warning
      receiver: 'slack-warning'

receivers:
  - name: 'default'
    email_configs:
      - to: 'team@example.com'

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'xxx'
        severity: '{{ .CommonLabels.severity }}'

  - name: 'slack-critical'
    slack_configs:
      - channel: '#alerts-critical'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'

  - name: 'slack-warning'
    slack_configs:
      - channel: '#alerts-warning'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
```

## Dashboards

### Grafana Dashboard для API

```json
{
  "title": "API Performance Dashboard",
  "panels": [
    {
      "title": "Request Rate",
      "type": "graph",
      "gridPos": {"x": 0, "y": 0, "w": 12, "h": 8},
      "targets": [
        {
          "expr": "sum(rate(http_requests_total[5m])) by (method)",
          "legendFormat": "{{method}}"
        }
      ]
    },
    {
      "title": "Error Rate",
      "type": "graph",
      "gridPos": {"x": 12, "y": 0, "w": 12, "h": 8},
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m])) * 100",
          "legendFormat": "Error %"
        }
      ],
      "alert": {
        "name": "High Error Rate",
        "conditions": [
          {"evaluator": {"params": [1], "type": "gt"}}
        ]
      }
    },
    {
      "title": "Latency Percentiles",
      "type": "graph",
      "gridPos": {"x": 0, "y": 8, "w": 12, "h": 8},
      "targets": [
        {
          "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
          "legendFormat": "p50"
        },
        {
          "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
          "legendFormat": "p95"
        },
        {
          "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
          "legendFormat": "p99"
        }
      ]
    },
    {
      "title": "Top Endpoints by Latency",
      "type": "table",
      "gridPos": {"x": 12, "y": 8, "w": 12, "h": 8},
      "targets": [
        {
          "expr": "topk(10, histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint)))",
          "format": "table"
        }
      ]
    }
  ]
}
```

## Health Checks

### Comprehensive Health Check

```python
from fastapi import FastAPI, Response
from pydantic import BaseModel
from typing import Dict, Optional
from enum import Enum
import asyncio
import time

class HealthStatus(str, Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    UNHEALTHY = "unhealthy"

class ComponentHealth(BaseModel):
    status: HealthStatus
    latency_ms: Optional[float] = None
    message: Optional[str] = None
    details: Optional[Dict] = None

class SystemHealth(BaseModel):
    status: HealthStatus
    version: str
    uptime_seconds: float
    components: Dict[str, ComponentHealth]

app = FastAPI()
start_time = time.time()

async def check_database() -> ComponentHealth:
    """Проверка базы данных"""
    start = time.perf_counter()
    try:
        await db.execute("SELECT 1")
        latency = (time.perf_counter() - start) * 1000

        return ComponentHealth(
            status=HealthStatus.HEALTHY if latency < 100 else HealthStatus.DEGRADED,
            latency_ms=round(latency, 2),
            message="Connected" if latency < 100 else "Slow response"
        )
    except Exception as e:
        return ComponentHealth(
            status=HealthStatus.UNHEALTHY,
            message=str(e)
        )

async def check_redis() -> ComponentHealth:
    """Проверка Redis"""
    start = time.perf_counter()
    try:
        await redis.ping()
        latency = (time.perf_counter() - start) * 1000

        info = await redis.info()
        return ComponentHealth(
            status=HealthStatus.HEALTHY,
            latency_ms=round(latency, 2),
            details={
                "connected_clients": info.get("connected_clients"),
                "used_memory_human": info.get("used_memory_human")
            }
        )
    except Exception as e:
        return ComponentHealth(
            status=HealthStatus.UNHEALTHY,
            message=str(e)
        )

async def check_external_api() -> ComponentHealth:
    """Проверка внешнего API"""
    start = time.perf_counter()
    try:
        async with httpx.AsyncClient(timeout=5.0) as client:
            response = await client.get("https://api.external.com/health")
            latency = (time.perf_counter() - start) * 1000

            if response.status_code == 200:
                return ComponentHealth(
                    status=HealthStatus.HEALTHY,
                    latency_ms=round(latency, 2)
                )
            else:
                return ComponentHealth(
                    status=HealthStatus.DEGRADED,
                    latency_ms=round(latency, 2),
                    message=f"Status code: {response.status_code}"
                )
    except Exception as e:
        return ComponentHealth(
            status=HealthStatus.UNHEALTHY,
            message=str(e)
        )

@app.get("/health", response_model=SystemHealth)
async def health_check(response: Response):
    """Comprehensive health check"""
    components = await asyncio.gather(
        check_database(),
        check_redis(),
        check_external_api(),
    )

    component_dict = {
        "database": components[0],
        "redis": components[1],
        "external_api": components[2],
    }

    # Определяем общий статус
    statuses = [c.status for c in components]
    if HealthStatus.UNHEALTHY in statuses:
        overall_status = HealthStatus.UNHEALTHY
        response.status_code = 503
    elif HealthStatus.DEGRADED in statuses:
        overall_status = HealthStatus.DEGRADED
        response.status_code = 200
    else:
        overall_status = HealthStatus.HEALTHY
        response.status_code = 200

    return SystemHealth(
        status=overall_status,
        version="1.0.0",
        uptime_seconds=round(time.time() - start_time, 2),
        components=component_dict
    )

@app.get("/health/live")
async def liveness():
    """Kubernetes liveness probe — процесс жив"""
    return {"status": "alive"}

@app.get("/health/ready")
async def readiness(response: Response):
    """Kubernetes readiness probe — готов принимать трафик"""
    db_health = await check_database()
    if db_health.status == HealthStatus.UNHEALTHY:
        response.status_code = 503
        return {"status": "not_ready", "reason": "database unavailable"}
    return {"status": "ready"}
```

## Best Practices

```python
"""
Best Practices для Profiling и Monitoring:

1. Профилирование:
   - Профилируйте в conditions, близких к production
   - Используйте sampling для снижения overhead
   - Фокусируйтесь на hot paths (критичные пути выполнения)
   - Сохраняйте профили для сравнения

2. Метрики:
   - Используйте RED method (Rate, Errors, Duration)
   - Добавляйте business metrics
   - Избегайте high cardinality labels
   - Используйте histograms вместо summary

3. Трейсинг:
   - Propagate context между сервисами
   - Добавляйте meaningful attributes
   - Sampling для снижения объёма данных
   - Записывайте errors и exceptions

4. Логирование:
   - Structured logging (JSON)
   - Correlation ID в каждом логе
   - Log levels правильно
   - Не логируйте sensitive data

5. Alerting:
   - Alert на симптомы, не причины
   - Избегайте alert fatigue
   - Включайте runbook URLs
   - Тестируйте алерты регулярно
"""
```

## Заключение

Эффективный мониторинг и профилирование требуют:

1. **Профилирования во время разработки** — cProfile, line_profiler, memory_profiler
2. **APM в production** — Sentry, OpenTelemetry, custom metrics
3. **Distributed tracing** — Jaeger, Zipkin для микросервисов
4. **Structured logging** — structlog, correlation IDs
5. **Alerting** — Prometheus + AlertManager, meaningful alerts
6. **Dashboards** — Grafana для визуализации

Мониторинг — это не просто сбор данных, а система принятия решений на основе этих данных.
