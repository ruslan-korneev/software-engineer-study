# Мониторинг и Observability

[prev: 24-performance-antipatterns](./24-performance-antipatterns.md) | [next: 26-cloud-design-patterns](./26-cloud-design-patterns.md)

---

## Введение

Мониторинг — это практика сбора, анализа и использования информации о состоянии системы для обеспечения её надёжной работы. В современных распределённых системах мониторинг становится критически важным компонентом, позволяющим:

- Обнаруживать проблемы до того, как они повлияют на пользователей
- Быстро находить корневые причины инцидентов
- Понимать поведение системы под нагрузкой
- Принимать обоснованные решения о масштабировании
- Обеспечивать SLA и SLO

**Observability** (наблюдаемость) — это свойство системы, которое определяет, насколько хорошо можно понять её внутреннее состояние по внешним выходным данным. Три столпа observability: **метрики**, **логи**, **трейсы**.

---

## Три столпа Observability

### 1. Метрики (Metrics)

Метрики — это числовые измерения, собираемые через регулярные интервалы времени. Они показывают агрегированную картину состояния системы.

#### Типы метрик

```python
from prometheus_client import Counter, Gauge, Histogram, Summary

# Counter — монотонно возрастающее значение
# Используется для: запросов, ошибок, обработанных сообщений
http_requests_total = Counter(
    'http_requests_total',
    'Total number of HTTP requests',
    ['method', 'endpoint', 'status']
)

# Gauge — значение, которое может увеличиваться и уменьшаться
# Используется для: температуры, памяти, количества активных соединений
active_connections = Gauge(
    'active_connections',
    'Number of active connections'
)

current_temperature = Gauge(
    'temperature_celsius',
    'Current temperature in Celsius'
)

# Histogram — распределение значений по bucket'ам
# Используется для: latency, размера запросов
request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration in seconds',
    ['method', 'endpoint'],
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

# Summary — похож на Histogram, но вычисляет квантили на клиенте
request_duration_summary = Summary(
    'http_request_duration_summary_seconds',
    'HTTP request duration summary',
    ['method', 'endpoint']
)
```

#### RED Method (для сервисов)

```python
from prometheus_client import Counter, Histogram
import time
from functools import wraps

# Rate — количество запросов в секунду
requests_total = Counter(
    'requests_total',
    'Total requests',
    ['service', 'endpoint']
)

# Errors — количество ошибок
errors_total = Counter(
    'errors_total',
    'Total errors',
    ['service', 'endpoint', 'error_type']
)

# Duration — время обработки
request_duration = Histogram(
    'request_duration_seconds',
    'Request duration',
    ['service', 'endpoint']
)

def track_request(service: str, endpoint: str):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            requests_total.labels(service=service, endpoint=endpoint).inc()

            start = time.time()
            try:
                result = func(*args, **kwargs)
                return result
            except Exception as e:
                errors_total.labels(
                    service=service,
                    endpoint=endpoint,
                    error_type=type(e).__name__
                ).inc()
                raise
            finally:
                duration = time.time() - start
                request_duration.labels(
                    service=service,
                    endpoint=endpoint
                ).observe(duration)

        return wrapper
    return decorator

# Использование
@track_request(service="user-service", endpoint="/users")
def get_users():
    # ... логика
    pass
```

#### USE Method (для ресурсов)

```python
import psutil
from prometheus_client import Gauge

# Utilization — процент использования ресурса
cpu_utilization = Gauge('cpu_utilization_percent', 'CPU utilization percentage')
memory_utilization = Gauge('memory_utilization_percent', 'Memory utilization percentage')
disk_utilization = Gauge('disk_utilization_percent', 'Disk utilization percentage', ['mount'])

# Saturation — степень насыщения (очередь ожидания)
cpu_load_average = Gauge('cpu_load_average', 'CPU load average', ['period'])
disk_io_queue = Gauge('disk_io_queue_length', 'Disk I/O queue length', ['device'])

# Errors — количество ошибок ресурса
network_errors = Gauge('network_errors_total', 'Network errors', ['interface', 'type'])

def collect_system_metrics():
    # CPU
    cpu_utilization.set(psutil.cpu_percent())

    load_avg = psutil.getloadavg()
    cpu_load_average.labels(period='1m').set(load_avg[0])
    cpu_load_average.labels(period='5m').set(load_avg[1])
    cpu_load_average.labels(period='15m').set(load_avg[2])

    # Memory
    memory = psutil.virtual_memory()
    memory_utilization.set(memory.percent)

    # Disk
    for partition in psutil.disk_partitions():
        usage = psutil.disk_usage(partition.mountpoint)
        disk_utilization.labels(mount=partition.mountpoint).set(usage.percent)
```

### 2. Логирование (Logging)

Логи — это текстовые записи о событиях в системе. Они предоставляют контекст и детали, которые недоступны в метриках.

#### Структурированное логирование

```python
import structlog
import logging
from datetime import datetime
import json

# Настройка structlog
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
        structlog.processors.JSONRenderer()
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    wrapper_class=structlog.stdlib.BoundLogger,
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger()

# Использование
def process_order(order_id: str, user_id: str, items: list):
    log = logger.bind(
        order_id=order_id,
        user_id=user_id,
        item_count=len(items)
    )

    log.info("order_processing_started")

    try:
        # Обработка заказа
        total = sum(item['price'] * item['quantity'] for item in items)

        log.info(
            "order_total_calculated",
            total=total,
            currency="USD"
        )

        # Отправка в платёжную систему
        payment_result = process_payment(order_id, total)

        log.info(
            "payment_processed",
            payment_id=payment_result['id'],
            status=payment_result['status']
        )

        return {"status": "success", "order_id": order_id}

    except PaymentError as e:
        log.error(
            "payment_failed",
            error_type=type(e).__name__,
            error_message=str(e)
        )
        raise
    except Exception as e:
        log.exception(
            "order_processing_failed",
            error_type=type(e).__name__
        )
        raise
```

#### Уровни логирования

```python
import logging

# Конфигурация уровней
LOGGING_CONFIG = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'json': {
            '()': 'pythonjsonlogger.jsonlogger.JsonFormatter',
            'format': '%(asctime)s %(name)s %(levelname)s %(message)s'
        }
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'json',
            'level': 'DEBUG'
        },
        'file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/var/log/app/app.log',
            'maxBytes': 10485760,  # 10MB
            'backupCount': 5,
            'formatter': 'json',
            'level': 'INFO'
        }
    },
    'loggers': {
        '': {  # Root logger
            'handlers': ['console', 'file'],
            'level': 'DEBUG'
        },
        'sqlalchemy': {
            'handlers': ['file'],
            'level': 'WARNING',
            'propagate': False
        }
    }
}

# Когда использовать какой уровень:
logger = logging.getLogger(__name__)

# DEBUG - детальная информация для отладки
logger.debug("Cache lookup", extra={"key": cache_key, "hit": hit})

# INFO - подтверждение нормальной работы
logger.info("User logged in", extra={"user_id": user_id})

# WARNING - что-то неожиданное, но система работает
logger.warning("Rate limit approaching", extra={"current": 95, "limit": 100})

# ERROR - ошибка, которая помешала выполнить операцию
logger.error("Failed to send email", extra={"recipient": email, "error": str(e)})

# CRITICAL - критическая ошибка, система может быть недоступна
logger.critical("Database connection pool exhausted")
```

#### Контекст запроса (Correlation ID)

```python
import uuid
from contextvars import ContextVar
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware

# Контекстная переменная для correlation ID
correlation_id_var: ContextVar[str] = ContextVar('correlation_id', default='')

class CorrelationIdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Получаем или генерируем correlation ID
        correlation_id = request.headers.get('X-Correlation-ID', str(uuid.uuid4()))
        correlation_id_var.set(correlation_id)

        # Добавляем в response headers
        response = await call_next(request)
        response.headers['X-Correlation-ID'] = correlation_id

        return response

# Логгер с автоматическим добавлением correlation ID
class CorrelationLogger:
    def __init__(self, name: str):
        self.logger = structlog.get_logger(name)

    def _log(self, level: str, message: str, **kwargs):
        correlation_id = correlation_id_var.get()
        getattr(self.logger, level)(
            message,
            correlation_id=correlation_id,
            **kwargs
        )

    def info(self, message: str, **kwargs):
        self._log('info', message, **kwargs)

    def error(self, message: str, **kwargs):
        self._log('error', message, **kwargs)

    # ... другие уровни

# Использование
logger = CorrelationLogger(__name__)

async def process_request(data: dict):
    logger.info("Processing request", data_size=len(data))

    # Вызов другого сервиса - передаём correlation ID
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "http://other-service/api",
            json=data,
            headers={"X-Correlation-ID": correlation_id_var.get()}
        )
```

### 3. Трейсинг (Distributed Tracing)

Трейсинг позволяет отслеживать путь запроса через все сервисы распределённой системы.

#### Концепции

```
Trace: Полный путь запроса через систему
  └── Span A (Gateway, 100ms)
        ├── Span B (User Service, 30ms)
        │     └── Span C (Database, 10ms)
        └── Span D (Order Service, 50ms)
              ├── Span E (Inventory Service, 20ms)
              └── Span F (Payment Service, 25ms)
```

#### OpenTelemetry

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.propagate import set_global_textmap
from opentelemetry.propagators.b3 import B3MultiFormat
import httpx

# Настройка провайдера
def setup_tracing(service_name: str):
    # Создаём провайдер
    provider = TracerProvider()

    # Добавляем экспортёр (отправка в Jaeger/Zipkin/etc)
    exporter = OTLPSpanExporter(
        endpoint="http://jaeger:4317",
        insecure=True
    )
    provider.add_span_processor(BatchSpanProcessor(exporter))

    # Устанавливаем глобально
    trace.set_tracer_provider(provider)

    # Настраиваем propagation (B3 для совместимости с Zipkin)
    set_global_textmap(B3MultiFormat())

    return trace.get_tracer(service_name)

# Автоматическая инструментация
def instrument_app(app):
    FastAPIInstrumentor.instrument_app(app)
    HTTPXClientInstrumentor().instrument()
    SQLAlchemyInstrumentor().instrument()

# Ручное создание span'ов
tracer = setup_tracing("order-service")

async def process_order(order_id: str, items: list):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        span.set_attribute("order.item_count", len(items))

        # Валидация
        with tracer.start_as_current_span("validate_items"):
            for item in items:
                validate_item(item)

        # Расчёт стоимости
        with tracer.start_as_current_span("calculate_total") as calc_span:
            total = sum(item['price'] * item['quantity'] for item in items)
            calc_span.set_attribute("order.total", total)

        # Вызов внешнего сервиса (автоматически инструментирован)
        async with httpx.AsyncClient() as client:
            with tracer.start_as_current_span("call_payment_service"):
                response = await client.post(
                    "http://payment-service/process",
                    json={"order_id": order_id, "amount": total}
                )

        span.set_attribute("order.status", "completed")
        return {"order_id": order_id, "status": "success"}
```

#### Baggage (передача контекста)

```python
from opentelemetry import baggage
from opentelemetry.context import attach, detach

def set_user_context(user_id: str, tenant_id: str):
    """Устанавливает контекст пользователя для всего trace"""
    ctx = baggage.set_baggage("user.id", user_id)
    ctx = baggage.set_baggage("tenant.id", tenant_id, context=ctx)
    return attach(ctx)

def get_user_context():
    """Получает контекст пользователя"""
    return {
        "user_id": baggage.get_baggage("user.id"),
        "tenant_id": baggage.get_baggage("tenant.id")
    }

# Middleware для FastAPI
class BaggageMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        user_id = request.headers.get("X-User-ID")
        tenant_id = request.headers.get("X-Tenant-ID")

        if user_id and tenant_id:
            token = set_user_context(user_id, tenant_id)
            try:
                response = await call_next(request)
            finally:
                detach(token)
        else:
            response = await call_next(request)

        return response
```

---

## Инструменты мониторинга

### Prometheus

Prometheus — это система мониторинга и оповещения с pull-моделью сбора метрик.

```python
from prometheus_client import start_http_server, Counter, Histogram, Gauge
from prometheus_client import CollectorRegistry, generate_latest, CONTENT_TYPE_LATEST
from fastapi import FastAPI, Response
import time

# Создание метрик
REQUEST_COUNT = Counter(
    'app_requests_total',
    'Total app requests',
    ['method', 'endpoint', 'status']
)

REQUEST_LATENCY = Histogram(
    'app_request_latency_seconds',
    'Request latency',
    ['method', 'endpoint'],
    buckets=[0.01, 0.025, 0.05, 0.075, 0.1, 0.25, 0.5, 0.75, 1.0, 2.5, 5.0]
)

ACTIVE_REQUESTS = Gauge(
    'app_active_requests',
    'Number of active requests'
)

app = FastAPI()

# Middleware для сбора метрик
@app.middleware("http")
async def metrics_middleware(request, call_next):
    ACTIVE_REQUESTS.inc()
    start_time = time.time()

    response = await call_next(request)

    duration = time.time() - start_time
    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()

    REQUEST_LATENCY.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(duration)

    ACTIVE_REQUESTS.dec()
    return response

# Endpoint для Prometheus
@app.get("/metrics")
def metrics():
    return Response(
        content=generate_latest(),
        media_type=CONTENT_TYPE_LATEST
    )
```

#### Prometheus конфигурация

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - '/etc/prometheus/rules/*.yml'

scrape_configs:
  - job_name: 'app'
    static_configs:
      - targets: ['app:8000']
    metrics_path: /metrics
    scrape_interval: 10s

  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
```

### Grafana

Grafana — платформа для визуализации метрик и создания дашбордов.

```python
# Пример запросов PromQL для Grafana

# Rate запросов в секунду
rate(app_requests_total[5m])

# 95-й перцентиль латентности
histogram_quantile(0.95, rate(app_request_latency_seconds_bucket[5m]))

# Error rate
sum(rate(app_requests_total{status=~"5.."}[5m])) / sum(rate(app_requests_total[5m]))

# Доступность (успешные запросы)
sum(rate(app_requests_total{status=~"2.."}[5m])) / sum(rate(app_requests_total[5m])) * 100
```

#### Grafana Dashboard as Code

```python
from grafanalib.core import (
    Dashboard, Graph, Row, Target,
    YAxes, YAxis, SECONDS_FORMAT,
    OPS_FORMAT, PERCENT_FORMAT
)
import json

# Программное создание дашборда
dashboard = Dashboard(
    title="Application Dashboard",
    rows=[
        Row(panels=[
            Graph(
                title="Request Rate",
                targets=[
                    Target(
                        expr='sum(rate(app_requests_total[5m])) by (endpoint)',
                        legendFormat='{{endpoint}}'
                    )
                ],
                yAxes=YAxes(
                    YAxis(format=OPS_FORMAT),
                    YAxis(format=OPS_FORMAT)
                )
            ),
            Graph(
                title="Error Rate",
                targets=[
                    Target(
                        expr='sum(rate(app_requests_total{status=~"5.."}[5m])) / sum(rate(app_requests_total[5m])) * 100',
                        legendFormat='Error %'
                    )
                ],
                yAxes=YAxes(
                    YAxis(format=PERCENT_FORMAT),
                    YAxis(format=PERCENT_FORMAT)
                )
            )
        ]),
        Row(panels=[
            Graph(
                title="Request Latency (p95)",
                targets=[
                    Target(
                        expr='histogram_quantile(0.95, sum(rate(app_request_latency_seconds_bucket[5m])) by (le, endpoint))',
                        legendFormat='{{endpoint}}'
                    )
                ],
                yAxes=YAxes(
                    YAxis(format=SECONDS_FORMAT),
                    YAxis(format=SECONDS_FORMAT)
                )
            )
        ])
    ]
).auto_panel_ids()

# Экспорт в JSON
print(json.dumps(dashboard.to_json_data(), indent=2))
```

### ELK Stack (Elasticsearch, Logstash, Kibana)

ELK Stack — популярное решение для централизованного логирования.

```python
# Отправка логов в Logstash через TCP
import logging
import logstash

# Настройка handler'а
host = 'logstash'
port = 5000

logger = logging.getLogger('python-logstash-logger')
logger.setLevel(logging.INFO)
logger.addHandler(logstash.TCPLogstashHandler(host, port, version=1))

# Использование
logger.info('Application started', extra={
    'service': 'user-service',
    'environment': 'production'
})

logger.error('Database connection failed', extra={
    'service': 'user-service',
    'database': 'users_db',
    'error_code': 'ECONNREFUSED'
})
```

#### Logstash конфигурация

```ruby
# logstash.conf
input {
  tcp {
    port => 5000
    codec => json
  }

  beats {
    port => 5044
  }
}

filter {
  # Парсинг timestamp
  date {
    match => ["@timestamp", "ISO8601"]
    target => "@timestamp"
  }

  # Добавление geo-данных по IP
  geoip {
    source => "client_ip"
    target => "geoip"
  }

  # Обогащение данных
  mutate {
    add_field => {
      "environment" => "${ENVIRONMENT:development}"
    }
  }

  # Парсинг стектрейса
  if [message] =~ /Traceback/ {
    mutate {
      add_tag => ["exception"]
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
}
```

---

## Алертинг (Alerting)

### Определение алертов

```yaml
# prometheus/rules/alerts.yml
groups:
  - name: application
    rules:
      # Высокий Error Rate
      - alert: HighErrorRate
        expr: |
          sum(rate(app_requests_total{status=~"5.."}[5m])) /
          sum(rate(app_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | printf \"%.2f\" }}% (threshold: 5%)"

      # Высокая латентность
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, sum(rate(app_request_latency_seconds_bucket[5m])) by (le)) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "95th percentile latency is {{ $value | printf \"%.2f\" }}s"

      # Сервис недоступен
      - alert: ServiceDown
        expr: up{job="app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.instance }} is down"
          description: "The service has been unreachable for more than 1 minute"

  - name: infrastructure
    rules:
      # Высокое использование CPU
      - alert: HighCPUUsage
        expr: |
          100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | printf \"%.1f\" }}%"

      # Мало свободной памяти
      - alert: LowMemory
        expr: |
          (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 < 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Low memory on {{ $labels.instance }}"
          description: "Available memory is {{ $value | printf \"%.1f\" }}%"

      # Диск заполняется
      - alert: DiskSpaceRunningOut
        expr: |
          (node_filesystem_avail_bytes{fstype!~"tmpfs"} /
           node_filesystem_size_bytes{fstype!~"tmpfs"}) * 100 < 10
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Disk space low on {{ $labels.instance }}"
          description: "Disk {{ $labels.mountpoint }} has {{ $value | printf \"%.1f\" }}% free"
```

### Alertmanager конфигурация

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@example.com'
  smtp_auth_username: 'alerts@example.com'
  smtp_auth_password: 'password'
  slack_api_url: 'https://hooks.slack.com/services/xxx/yyy/zzz'

route:
  receiver: 'default'
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    # Critical алерты - сразу в Slack и PagerDuty
    - match:
        severity: critical
      receiver: 'critical-alerts'
      continue: true

    # Warning алерты - только в Slack
    - match:
        severity: warning
      receiver: 'warning-alerts'

receivers:
  - name: 'default'
    email_configs:
      - to: 'team@example.com'

  - name: 'critical-alerts'
    slack_configs:
      - channel: '#alerts-critical'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true
    pagerduty_configs:
      - service_key: 'your-pagerduty-key'

  - name: 'warning-alerts'
    slack_configs:
      - channel: '#alerts-warning'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

inhibit_rules:
  # Если сервис недоступен, не шлём алерты о его метриках
  - source_match:
      alertname: 'ServiceDown'
    target_match:
      severity: 'warning'
    equal: ['instance']
```

### On-Call практики

```python
# Пример интеграции с PagerDuty
import requests
from datetime import datetime

class PagerDutyClient:
    def __init__(self, routing_key: str):
        self.routing_key = routing_key
        self.base_url = "https://events.pagerduty.com/v2/enqueue"

    def trigger_alert(
        self,
        summary: str,
        severity: str = "critical",
        source: str = "monitoring",
        custom_details: dict = None
    ):
        payload = {
            "routing_key": self.routing_key,
            "event_action": "trigger",
            "payload": {
                "summary": summary,
                "severity": severity,
                "source": source,
                "timestamp": datetime.utcnow().isoformat(),
                "custom_details": custom_details or {}
            }
        }

        response = requests.post(self.base_url, json=payload)
        return response.json()

    def resolve_alert(self, dedup_key: str):
        payload = {
            "routing_key": self.routing_key,
            "event_action": "resolve",
            "dedup_key": dedup_key
        }

        response = requests.post(self.base_url, json=payload)
        return response.json()

# Использование
pagerduty = PagerDutyClient("your-routing-key")

# При обнаружении критической проблемы
pagerduty.trigger_alert(
    summary="Database connection pool exhausted",
    severity="critical",
    source="user-service",
    custom_details={
        "active_connections": 100,
        "max_connections": 100,
        "waiting_requests": 50
    }
)
```

---

## SLI, SLO, SLA

### Service Level Indicators (SLI)

```python
from prometheus_client import Counter, Histogram
from dataclasses import dataclass
from typing import Optional

# Определение SLI
@dataclass
class SLIDefinition:
    name: str
    description: str
    metric_name: str
    good_events_query: str
    total_events_query: str

# Основные SLI
availability_sli = SLIDefinition(
    name="availability",
    description="Percentage of successful requests",
    metric_name="http_requests_total",
    good_events_query='sum(rate(http_requests_total{status=~"2.."}[5m]))',
    total_events_query='sum(rate(http_requests_total[5m]))'
)

latency_sli = SLIDefinition(
    name="latency",
    description="Percentage of requests faster than threshold",
    metric_name="http_request_duration_seconds",
    good_events_query='sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m]))',
    total_events_query='sum(rate(http_request_duration_seconds_count[5m]))'
)

# Код для расчёта SLI
class SLICalculator:
    def __init__(self, prometheus_client):
        self.prometheus = prometheus_client

    def calculate_sli(self, sli: SLIDefinition, time_range: str = "5m") -> float:
        good_events = self.prometheus.query(sli.good_events_query)
        total_events = self.prometheus.query(sli.total_events_query)

        if total_events == 0:
            return 1.0  # Если нет запросов, считаем SLI = 100%

        return good_events / total_events
```

### Service Level Objectives (SLO)

```yaml
# slo.yaml - Определение SLO
slos:
  - name: "API Availability"
    sli: availability
    target: 0.999  # 99.9%
    window: "30d"

  - name: "API Latency"
    sli: latency
    description: "99% of requests complete within 300ms"
    target: 0.99
    threshold: 0.3  # 300ms
    window: "30d"

  - name: "Checkout Success Rate"
    sli: checkout_success
    target: 0.995  # 99.5%
    window: "30d"
```

```python
# SLO мониторинг и Error Budget
from dataclasses import dataclass
from datetime import datetime, timedelta

@dataclass
class SLO:
    name: str
    target: float  # 0.999 = 99.9%
    window_days: int = 30

class ErrorBudgetCalculator:
    def __init__(self, prometheus_client):
        self.prometheus = prometheus_client

    def calculate_error_budget(self, slo: SLO, current_sli: float) -> dict:
        """
        Рассчитывает Error Budget

        Error Budget = (1 - SLO target) * time_window

        Например, для SLO 99.9% за 30 дней:
        Error Budget = 0.001 * 30 дней = 43.2 минуты downtime
        """
        error_budget_total = 1 - slo.target
        error_budget_consumed = max(0, slo.target - current_sli)
        error_budget_remaining = error_budget_total - error_budget_consumed

        # В минутах
        window_minutes = slo.window_days * 24 * 60
        budget_total_minutes = error_budget_total * window_minutes
        budget_consumed_minutes = error_budget_consumed * window_minutes
        budget_remaining_minutes = error_budget_remaining * window_minutes

        return {
            "slo_target": slo.target,
            "current_sli": current_sli,
            "error_budget_total": error_budget_total,
            "error_budget_consumed": error_budget_consumed,
            "error_budget_remaining": error_budget_remaining,
            "error_budget_remaining_percent": (error_budget_remaining / error_budget_total) * 100,
            "budget_minutes_remaining": budget_remaining_minutes,
            "is_healthy": current_sli >= slo.target
        }

# Пример использования
slo = SLO(name="API Availability", target=0.999, window_days=30)
calculator = ErrorBudgetCalculator(prometheus_client)

result = calculator.calculate_error_budget(slo, current_sli=0.998)
print(f"Error Budget remaining: {result['budget_minutes_remaining']:.1f} minutes")
print(f"Error Budget remaining: {result['error_budget_remaining_percent']:.1f}%")
```

### Burn Rate Alerts

```yaml
# Алерты на основе burn rate
groups:
  - name: slo-alerts
    rules:
      # Быстрый burn rate (2% за 1 час)
      - alert: SLOBurnRateFast
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[1h])) /
            sum(rate(http_requests_total[1h]))
          ) > (14.4 * 0.001)  # 14.4x burn rate
        for: 2m
        labels:
          severity: critical
          window: 1h
        annotations:
          summary: "High error burn rate (fast)"
          description: "Error budget will be exhausted in ~1 hour at current rate"

      # Средний burn rate (5% за 6 часов)
      - alert: SLOBurnRateMedium
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[6h])) /
            sum(rate(http_requests_total[6h]))
          ) > (6 * 0.001)  # 6x burn rate
        for: 15m
        labels:
          severity: warning
          window: 6h
        annotations:
          summary: "Elevated error burn rate (medium)"
          description: "Error budget will be exhausted in ~6 hours at current rate"
```

---

## Best Practices

### 1. Naming Conventions

```python
# Метрики - snake_case с суффиксами
# Суффиксы: _total (counter), _seconds (duration), _bytes (size), _ratio

http_requests_total  # Counter
http_request_duration_seconds  # Histogram
process_memory_bytes  # Gauge
cache_hit_ratio  # Gauge (0-1)

# Labels - lowercase, без спецсимволов
http_requests_total{method="GET", endpoint="/users", status="200"}
```

### 2. Cardinality Control

```python
# Плохо - высокая cardinality
http_requests_total{user_id="123456", session_id="abc..."}  # Миллионы значений!

# Хорошо - ограниченная cardinality
http_requests_total{method="GET", status="200", endpoint="/api/users"}
```

### 3. Dashboard Design

```
Golden Signals Dashboard Layout:
┌─────────────────────────────────────────────────────────────┐
│  Header: Service Name | Time Range | Refresh               │
├─────────────────────────────────────────────────────────────┤
│  Row 1: Key Metrics (Single Stats)                         │
│  [Uptime] [RPS] [Error Rate] [P95 Latency] [Active Users]   │
├─────────────────────────────────────────────────────────────┤
│  Row 2: Traffic                                             │
│  [Requests/sec by endpoint] [Requests by status code]       │
├─────────────────────────────────────────────────────────────┤
│  Row 3: Latency                                             │
│  [P50/P95/P99 latency] [Latency heatmap]                    │
├─────────────────────────────────────────────────────────────┤
│  Row 4: Errors                                              │
│  [Error rate] [Errors by type] [Error logs]                 │
├─────────────────────────────────────────────────────────────┤
│  Row 5: Saturation                                          │
│  [CPU] [Memory] [Connections] [Thread pool]                 │
└─────────────────────────────────────────────────────────────┘
```

### 4. Log Levels Best Practices

```python
# DEBUG - только для разработки
logger.debug(f"Cache key: {key}, value size: {len(value)}")

# INFO - важные бизнес-события
logger.info("User created", extra={"user_id": user_id, "email": email})

# WARNING - потенциальные проблемы
logger.warning("Rate limit approaching", extra={"current": 95, "limit": 100})

# ERROR - ошибки, требующие внимания
logger.error("Payment failed", extra={"order_id": order_id, "error": str(e)})

# CRITICAL - система неработоспособна
logger.critical("Database connection lost")
```

---

## Заключение

Эффективный мониторинг — это не просто сбор метрик, а построение целостной системы наблюдаемости:

1. **Три столпа observability** работают вместе:
   - Метрики показывают ЧТО происходит
   - Логи объясняют ПОЧЕМУ это происходит
   - Трейсы показывают ГДЕ это происходит

2. **Правильные инструменты** для правильных задач:
   - Prometheus для метрик
   - ELK/Loki для логов
   - Jaeger/Zipkin для трейсинга
   - Grafana для визуализации

3. **SLO-driven подход**:
   - Определите что важно для пользователей
   - Установите измеримые цели
   - Мониторьте Error Budget
   - Алертите на burn rate, не на симптомы

4. **Культура** важнее инструментов:
   - Инструментируйте код с самого начала
   - Создавайте runbooks для алертов
   - Проводите post-mortems
   - Постоянно улучшайте дашборды
