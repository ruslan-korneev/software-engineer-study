# Performance Metrics

## Введение

Метрики производительности API — это количественные показатели, которые позволяют оценить, насколько эффективно работает ваш API. Без метрик невозможно понять текущее состояние системы, обнаружить деградацию производительности и принимать обоснованные решения об оптимизации.

## Ключевые метрики производительности

### 1. Latency (Задержка)

Задержка — это время между отправкой запроса и получением ответа.

#### Виды задержки

```
Client Latency = Network Latency + Server Processing Time + Network Latency (response)
                        ↓                    ↓                      ↓
                   Time to First Byte    Application Logic    Response Transmission
```

#### Важные перцентили

```python
import numpy as np
from dataclasses import dataclass
from typing import List

@dataclass
class LatencyMetrics:
    """Класс для расчёта метрик задержки"""

    def calculate_percentiles(self, latencies: List[float]) -> dict:
        """
        Расчёт перцентилей задержки

        p50 (медиана) — типичный опыт пользователя
        p95 — опыт 95% пользователей
        p99 — опыт 99% пользователей (показывает "хвост" распределения)
        p99.9 — критичен для high-traffic систем
        """
        return {
            "p50": np.percentile(latencies, 50),
            "p95": np.percentile(latencies, 95),
            "p99": np.percentile(latencies, 99),
            "p99.9": np.percentile(latencies, 99.9),
            "average": np.mean(latencies),
            "max": np.max(latencies)
        }

# Пример использования
latencies = [12, 15, 18, 20, 22, 25, 30, 45, 100, 500]  # в миллисекундах
metrics = LatencyMetrics()
result = metrics.calculate_percentiles(latencies)

# Результат: average=78.7ms, но p50=21ms — среднее искажено выбросами!
```

#### Почему p99 важнее среднего

```
Распределение задержек:
                                                              ┌─ p99
█████████████████████████████████████████████████████         │
█████████████████████████████████████████████████████████████ │
████████████████████████████████████████████████████████████████████ ─┤
████████████████████████████████████████████████████████████████ ─┤   │
                    ↑                                 ↑        p95│   │
                   p50                              p90            │   │

При 1M запросов/день, p99 = 500ms означает 10,000 медленных запросов!
```

### 2. Throughput (Пропускная способность)

Throughput — количество запросов, которые система может обработать за единицу времени.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta
from collections import deque
import threading
import time

@dataclass
class ThroughputCounter:
    """Счётчик пропускной способности с скользящим окном"""

    def __init__(self, window_seconds: int = 60):
        self.window_seconds = window_seconds
        self.requests = deque()
        self.lock = threading.Lock()

    def record_request(self):
        """Регистрация нового запроса"""
        now = datetime.now()
        with self.lock:
            self.requests.append(now)
            self._cleanup(now)

    def _cleanup(self, now: datetime):
        """Удаление устаревших записей"""
        cutoff = now - timedelta(seconds=self.window_seconds)
        while self.requests and self.requests[0] < cutoff:
            self.requests.popleft()

    def get_rps(self) -> float:
        """Получение текущего RPS (requests per second)"""
        with self.lock:
            self._cleanup(datetime.now())
            return len(self.requests) / self.window_seconds

# Метрики пропускной способности
class ThroughputMetrics:
    """
    RPS (Requests Per Second) — базовая метрика
    RPM (Requests Per Minute) — для менее нагруженных систем
    QPS (Queries Per Second) — для баз данных
    TPS (Transactions Per Second) — для транзакционных систем
    """

    def __init__(self):
        self.counter = ThroughputCounter()

    def calculate_capacity(self, current_rps: float,
                          max_rps: float) -> dict:
        """Расчёт загрузки системы"""
        utilization = (current_rps / max_rps) * 100
        headroom = max_rps - current_rps

        return {
            "current_rps": current_rps,
            "max_rps": max_rps,
            "utilization_percent": utilization,
            "headroom_rps": headroom,
            "status": self._get_status(utilization)
        }

    def _get_status(self, utilization: float) -> str:
        if utilization < 50:
            return "healthy"
        elif utilization < 75:
            return "moderate"
        elif utilization < 90:
            return "warning"
        else:
            return "critical"
```

### 3. Error Rate (Частота ошибок)

```python
from enum import Enum
from collections import defaultdict
from typing import Dict

class ErrorCategory(Enum):
    CLIENT_ERROR = "4xx"      # Ошибки клиента
    SERVER_ERROR = "5xx"      # Ошибки сервера
    TIMEOUT = "timeout"       # Таймауты
    NETWORK = "network"       # Сетевые ошибки

class ErrorRateTracker:
    """Отслеживание частоты ошибок"""

    def __init__(self):
        self.total_requests = 0
        self.errors_by_category: Dict[ErrorCategory, int] = defaultdict(int)
        self.errors_by_endpoint: Dict[str, int] = defaultdict(int)

    def record(self, endpoint: str, status_code: int, error: bool = False):
        """Регистрация результата запроса"""
        self.total_requests += 1

        if error or status_code >= 400:
            category = self._categorize(status_code, error)
            self.errors_by_category[category] += 1
            self.errors_by_endpoint[endpoint] += 1

    def _categorize(self, status_code: int, is_error: bool) -> ErrorCategory:
        if 400 <= status_code < 500:
            return ErrorCategory.CLIENT_ERROR
        elif status_code >= 500:
            return ErrorCategory.SERVER_ERROR
        elif is_error:
            return ErrorCategory.NETWORK
        return ErrorCategory.TIMEOUT

    def get_error_rates(self) -> dict:
        """Расчёт коэффициентов ошибок"""
        if self.total_requests == 0:
            return {"total_error_rate": 0}

        total_errors = sum(self.errors_by_category.values())

        return {
            "total_error_rate": (total_errors / self.total_requests) * 100,
            "server_error_rate": (
                self.errors_by_category[ErrorCategory.SERVER_ERROR] /
                self.total_requests
            ) * 100,
            "client_error_rate": (
                self.errors_by_category[ErrorCategory.CLIENT_ERROR] /
                self.total_requests
            ) * 100,
            "availability": (
                (self.total_requests - self.errors_by_category[ErrorCategory.SERVER_ERROR]) /
                self.total_requests
            ) * 100
        }

# SLA примеры:
# 99.9% availability = максимум 8.76 часов downtime в год
# 99.99% availability = максимум 52.56 минут downtime в год
# 99.999% availability = максимум 5.26 минут downtime в год
```

### 4. Saturation (Насыщение)

Насыщение показывает, насколько близка система к исчерпанию ресурсов.

```python
import psutil
from dataclasses import dataclass
from typing import Optional

@dataclass
class ResourceSaturation:
    """Мониторинг насыщения ресурсов"""

    @staticmethod
    def get_system_saturation() -> dict:
        """Получение метрик насыщения системы"""
        cpu = psutil.cpu_percent(interval=1)
        memory = psutil.virtual_memory()
        disk = psutil.disk_usage('/')

        # Сетевые соединения
        connections = len(psutil.net_connections())

        return {
            "cpu": {
                "usage_percent": cpu,
                "status": "critical" if cpu > 90 else "warning" if cpu > 70 else "healthy"
            },
            "memory": {
                "usage_percent": memory.percent,
                "available_gb": memory.available / (1024**3),
                "status": "critical" if memory.percent > 90 else "warning" if memory.percent > 80 else "healthy"
            },
            "disk": {
                "usage_percent": disk.percent,
                "free_gb": disk.free / (1024**3),
                "status": "critical" if disk.percent > 95 else "warning" if disk.percent > 85 else "healthy"
            },
            "connections": {
                "active": connections,
                "status": "warning" if connections > 10000 else "healthy"
            }
        }

# Database-specific saturation
class DatabaseSaturation:
    """Метрики насыщения базы данных"""

    def __init__(self, connection_pool):
        self.pool = connection_pool

    def get_metrics(self) -> dict:
        return {
            "connection_pool": {
                "active": self.pool.active_connections,
                "idle": self.pool.idle_connections,
                "max": self.pool.max_connections,
                "utilization": (self.pool.active_connections / self.pool.max_connections) * 100,
                "waiting_requests": self.pool.waiting_queue_size
            }
        }
```

## Методология RED и USE

### RED Method (для микросервисов)

```python
from prometheus_client import Counter, Histogram, Gauge
import time
from functools import wraps

# Rate — количество запросов в секунду
request_count = Counter(
    'api_requests_total',
    'Total API requests',
    ['method', 'endpoint', 'status']
)

# Errors — количество ошибочных запросов
error_count = Counter(
    'api_errors_total',
    'Total API errors',
    ['method', 'endpoint', 'error_type']
)

# Duration — распределение времени ответа
request_duration = Histogram(
    'api_request_duration_seconds',
    'API request duration in seconds',
    ['method', 'endpoint'],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

def red_metrics(endpoint: str):
    """Декоратор для сбора RED метрик"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            start_time = time.time()
            status = "success"

            try:
                result = await func(*args, **kwargs)
                return result
            except Exception as e:
                status = "error"
                error_count.labels(
                    method="POST",
                    endpoint=endpoint,
                    error_type=type(e).__name__
                ).inc()
                raise
            finally:
                duration = time.time() - start_time

                request_count.labels(
                    method="POST",
                    endpoint=endpoint,
                    status=status
                ).inc()

                request_duration.labels(
                    method="POST",
                    endpoint=endpoint
                ).observe(duration)

        return wrapper
    return decorator

# Использование
@red_metrics("/api/users")
async def create_user(user_data: dict):
    # Логика создания пользователя
    pass
```

### USE Method (для ресурсов)

```python
from dataclasses import dataclass
from typing import Dict, Any

@dataclass
class USEMetrics:
    """
    USE Method:
    - Utilization — процент времени, когда ресурс занят
    - Saturation — степень переполнения (очередь ожидания)
    - Errors — количество ошибок
    """

    resource_name: str
    utilization: float      # 0.0 - 1.0
    saturation: float       # 0.0 - 1.0 (или queue length)
    errors: int

    def get_status(self) -> str:
        if self.errors > 0:
            return "errors_detected"
        if self.saturation > 0.8:
            return "saturated"
        if self.utilization > 0.9:
            return "high_utilization"
        if self.utilization > 0.7:
            return "moderate_utilization"
        return "healthy"

    def to_dict(self) -> Dict[str, Any]:
        return {
            "resource": self.resource_name,
            "utilization_percent": self.utilization * 100,
            "saturation_percent": self.saturation * 100,
            "errors": self.errors,
            "status": self.get_status()
        }

# Пример для CPU
cpu_metrics = USEMetrics(
    resource_name="cpu",
    utilization=0.75,
    saturation=0.1,  # run queue length normalized
    errors=0
)

# Пример для сетевого интерфейса
network_metrics = USEMetrics(
    resource_name="eth0",
    utilization=0.45,
    saturation=0.02,  # transmit queue drops
    errors=5
)
```

## Apdex Score (Application Performance Index)

```python
from enum import Enum
from typing import List

class SatisfactionLevel(Enum):
    SATISFIED = "satisfied"
    TOLERATING = "tolerating"
    FRUSTRATED = "frustrated"

class ApdexCalculator:
    """
    Apdex — стандартизированная метрика удовлетворённости пользователей

    Формула: Apdex = (Satisfied + Tolerating/2) / Total

    - Satisfied: время ответа <= T (target)
    - Tolerating: T < время ответа <= 4T
    - Frustrated: время ответа > 4T
    """

    def __init__(self, target_ms: float = 500):
        """
        Args:
            target_ms: Целевое время ответа в миллисекундах
        """
        self.target = target_ms
        self.frustrated_threshold = target_ms * 4

    def classify(self, response_time_ms: float) -> SatisfactionLevel:
        """Классификация времени ответа"""
        if response_time_ms <= self.target:
            return SatisfactionLevel.SATISFIED
        elif response_time_ms <= self.frustrated_threshold:
            return SatisfactionLevel.TOLERATING
        else:
            return SatisfactionLevel.FRUSTRATED

    def calculate(self, response_times: List[float]) -> dict:
        """Расчёт Apdex score"""
        if not response_times:
            return {"apdex": 1.0, "interpretation": "No data"}

        satisfied = 0
        tolerating = 0
        frustrated = 0

        for rt in response_times:
            level = self.classify(rt)
            if level == SatisfactionLevel.SATISFIED:
                satisfied += 1
            elif level == SatisfactionLevel.TOLERATING:
                tolerating += 1
            else:
                frustrated += 1

        total = len(response_times)
        apdex = (satisfied + tolerating / 2) / total

        return {
            "apdex": round(apdex, 3),
            "satisfied_count": satisfied,
            "tolerating_count": tolerating,
            "frustrated_count": frustrated,
            "interpretation": self._interpret(apdex)
        }

    def _interpret(self, apdex: float) -> str:
        if apdex >= 0.94:
            return "Excellent"
        elif apdex >= 0.85:
            return "Good"
        elif apdex >= 0.7:
            return "Fair"
        elif apdex >= 0.5:
            return "Poor"
        else:
            return "Unacceptable"

# Пример использования
calculator = ApdexCalculator(target_ms=200)
response_times = [50, 100, 150, 200, 300, 500, 800, 1200, 150, 180]
result = calculator.calculate(response_times)
# result = {"apdex": 0.75, "interpretation": "Fair", ...}
```

## Инструменты для сбора метрик

### Prometheus + Grafana

```yaml
# prometheus.yml - конфигурация Prometheus
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'api-service'
    static_configs:
      - targets: ['localhost:8000']
    metrics_path: '/metrics'

  - job_name: 'api-service-production'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: api-service
        action: keep
```

```python
# FastAPI с Prometheus метриками
from fastapi import FastAPI, Request
from prometheus_client import (
    Counter, Histogram, Gauge,
    generate_latest, CONTENT_TYPE_LATEST
)
from starlette.responses import Response
import time

app = FastAPI()

# Определение метрик
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status_code']
)

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint'],
    buckets=[0.01, 0.05, 0.1, 0.5, 1.0, 5.0]
)

REQUESTS_IN_PROGRESS = Gauge(
    'http_requests_in_progress',
    'HTTP requests currently in progress',
    ['method', 'endpoint']
)

@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    method = request.method
    endpoint = request.url.path

    REQUESTS_IN_PROGRESS.labels(method=method, endpoint=endpoint).inc()

    start_time = time.time()
    response = await call_next(request)
    duration = time.time() - start_time

    REQUEST_COUNT.labels(
        method=method,
        endpoint=endpoint,
        status_code=response.status_code
    ).inc()

    REQUEST_LATENCY.labels(
        method=method,
        endpoint=endpoint
    ).observe(duration)

    REQUESTS_IN_PROGRESS.labels(method=method, endpoint=endpoint).dec()

    return response

@app.get("/metrics")
async def metrics():
    return Response(
        content=generate_latest(),
        media_type=CONTENT_TYPE_LATEST
    )
```

### StatsD / Datadog

```python
from datadog import DogStatsd
import time
from functools import wraps

statsd = DogStatsd(host="localhost", port=8125)

def track_metrics(metric_name: str):
    """Декоратор для отправки метрик в Datadog"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            tags = [f"function:{func.__name__}"]

            # Increment counter
            statsd.increment(f"{metric_name}.calls", tags=tags)

            start_time = time.time()
            try:
                result = await func(*args, **kwargs)
                statsd.increment(f"{metric_name}.success", tags=tags)
                return result
            except Exception as e:
                statsd.increment(
                    f"{metric_name}.errors",
                    tags=tags + [f"error:{type(e).__name__}"]
                )
                raise
            finally:
                duration_ms = (time.time() - start_time) * 1000
                statsd.histogram(f"{metric_name}.duration", duration_ms, tags=tags)

        return wrapper
    return decorator
```

## Best Practices

### 1. Выбор правильных метрик

```python
class MetricsStrategy:
    """
    Стратегия выбора метрик для разных уровней системы
    """

    FRONTEND_METRICS = {
        "core_web_vitals": [
            "LCP",  # Largest Contentful Paint
            "FID",  # First Input Delay
            "CLS",  # Cumulative Layout Shift
        ],
        "user_experience": [
            "time_to_first_byte",
            "time_to_interactive",
            "error_rate",
        ]
    }

    API_METRICS = {
        "red_method": [
            "request_rate",
            "error_rate",
            "duration_percentiles",
        ],
        "business": [
            "requests_by_endpoint",
            "requests_by_user_tier",
            "revenue_impacting_errors",
        ]
    }

    INFRASTRUCTURE_METRICS = {
        "use_method": [
            "cpu_utilization",
            "memory_utilization",
            "disk_io_saturation",
            "network_saturation",
        ],
        "capacity": [
            "connection_pool_usage",
            "thread_pool_usage",
            "queue_depth",
        ]
    }
```

### 2. Установка SLO (Service Level Objectives)

```python
from dataclasses import dataclass
from typing import List, Callable
from datetime import datetime, timedelta

@dataclass
class SLO:
    """Service Level Objective"""
    name: str
    target: float  # например, 99.9
    window: timedelta  # период измерения
    metric_query: str  # запрос для получения метрики

    def calculate_error_budget(self, current_availability: float) -> dict:
        """Расчёт error budget"""
        allowed_failures = 100 - self.target  # например, 0.1%
        current_failures = 100 - current_availability

        budget_remaining = allowed_failures - current_failures
        budget_consumed = (current_failures / allowed_failures) * 100 if allowed_failures > 0 else 100

        return {
            "slo_target": f"{self.target}%",
            "current_availability": f"{current_availability}%",
            "error_budget_total": f"{allowed_failures}%",
            "error_budget_consumed": f"{budget_consumed:.1f}%",
            "error_budget_remaining": f"{budget_remaining:.3f}%",
            "status": "healthy" if budget_consumed < 80 else "warning" if budget_consumed < 100 else "exhausted"
        }

# Пример SLO
api_availability_slo = SLO(
    name="API Availability",
    target=99.9,
    window=timedelta(days=30),
    metric_query='avg(rate(http_requests_total{status!~"5.."}[5m])) / avg(rate(http_requests_total[5m]))'
)

latency_slo = SLO(
    name="P99 Latency",
    target=99.0,  # 99% запросов должны быть < 500ms
    window=timedelta(days=7),
    metric_query='histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) < 0.5'
)
```

### 3. Алертинг на основе метрик

```yaml
# Prometheus alerting rules
groups:
  - name: api-performance
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m])) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} for the last 5 minutes"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
          ) > 1.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "P99 latency is {{ $value }}s"

      - alert: LowThroughput
        expr: |
          sum(rate(http_requests_total[5m])) < 10
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Low throughput detected"
          description: "Request rate is {{ $value }} req/s"
```

## Визуализация метрик

### Grafana Dashboard JSON

```json
{
  "title": "API Performance Overview",
  "panels": [
    {
      "title": "Request Rate",
      "type": "graph",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total[5m]))",
          "legendFormat": "Total RPS"
        }
      ]
    },
    {
      "title": "Latency Percentiles",
      "type": "graph",
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
      "title": "Error Rate",
      "type": "stat",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m])) * 100",
          "legendFormat": "Error %"
        }
      ]
    }
  ]
}
```

## Заключение

Эффективный мониторинг производительности API требует:

1. **Правильного выбора метрик** — используйте RED для сервисов, USE для инфраструктуры
2. **Перцентилей вместо средних** — p99 показывает реальный опыт пользователей
3. **SLO и Error Budget** — определите цели и отслеживайте их выполнение
4. **Автоматического алертинга** — реагируйте на проблемы до того, как их заметят пользователи
5. **Визуализации** — используйте dashboards для быстрого понимания состояния системы

Метрики — это основа для всех последующих оптимизаций производительности.
