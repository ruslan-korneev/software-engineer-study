# Создание кастомных метрик

## Введение

Кастомные метрики — это метрики, которые вы определяете и инструментируете в своём приложении для отслеживания бизнес-логики, производительности и поведения системы. В отличие от стандартных системных метрик (CPU, память, сеть), кастомные метрики позволяют измерять специфичные для вашего приложения показатели.

## Типы метрик

### 1. Counter (Счётчик)

Counter — это метрика, которая только увеличивается (или сбрасывается при перезапуске). Используется для подсчёта событий.

**Примеры использования:**
- Количество HTTP-запросов
- Количество ошибок
- Количество обработанных заказов

#### Python (prometheus_client)

```python
from prometheus_client import Counter, start_http_server
import time
import random

# Создаём счётчик с labels
http_requests_total = Counter(
    'http_requests_total',
    'Общее количество HTTP запросов',
    ['method', 'endpoint', 'status_code']
)

# Счётчик ошибок
errors_total = Counter(
    'application_errors_total',
    'Общее количество ошибок приложения',
    ['error_type']
)

def handle_request(method, endpoint):
    """Обработка запроса с инструментированием"""
    try:
        # Симуляция обработки запроса
        if random.random() < 0.1:
            raise ValueError("Случайная ошибка")

        # Увеличиваем счётчик успешных запросов
        http_requests_total.labels(
            method=method,
            endpoint=endpoint,
            status_code='200'
        ).inc()

    except ValueError as e:
        # Увеличиваем счётчик ошибок
        errors_total.labels(error_type='ValueError').inc()
        http_requests_total.labels(
            method=method,
            endpoint=endpoint,
            status_code='500'
        ).inc()

if __name__ == '__main__':
    # Запускаем HTTP-сервер для экспорта метрик
    start_http_server(8000)

    while True:
        handle_request('GET', '/api/users')
        time.sleep(0.1)
```

#### Go (prometheus/client_golang)

```go
package main

import (
    "math/rand"
    "net/http"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Общее количество HTTP запросов",
        },
        []string{"method", "endpoint", "status_code"},
    )

    errorsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "application_errors_total",
            Help: "Общее количество ошибок приложения",
        },
        []string{"error_type"},
    )
)

func handleRequest(method, endpoint string) {
    // Симуляция обработки с возможной ошибкой
    if rand.Float64() < 0.1 {
        errorsTotal.WithLabelValues("random_error").Inc()
        httpRequestsTotal.WithLabelValues(method, endpoint, "500").Inc()
        return
    }

    httpRequestsTotal.WithLabelValues(method, endpoint, "200").Inc()
}

func main() {
    // Обработчик для метрик Prometheus
    http.Handle("/metrics", promhttp.Handler())

    go func() {
        for {
            handleRequest("GET", "/api/users")
            time.Sleep(100 * time.Millisecond)
        }
    }()

    http.ListenAndServe(":8000", nil)
}
```

#### Java (Micrometer)

```java
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.prometheus.PrometheusConfig;
import io.micrometer.prometheus.PrometheusMeterRegistry;

public class CustomMetricsExample {

    private final Counter httpRequestsTotal;
    private final Counter errorsTotal;

    public CustomMetricsExample(MeterRegistry registry) {
        // Создание счётчиков с тегами
        this.httpRequestsTotal = Counter.builder("http_requests_total")
            .description("Общее количество HTTP запросов")
            .tags("method", "GET", "endpoint", "/api/users")
            .register(registry);

        this.errorsTotal = Counter.builder("application_errors_total")
            .description("Общее количество ошибок приложения")
            .tag("error_type", "runtime")
            .register(registry);
    }

    public void handleRequest() {
        try {
            // Логика обработки запроса
            if (Math.random() < 0.1) {
                throw new RuntimeException("Случайная ошибка");
            }
            httpRequestsTotal.increment();
        } catch (RuntimeException e) {
            errorsTotal.increment();
        }
    }

    public static void main(String[] args) {
        PrometheusMeterRegistry registry = new PrometheusMeterRegistry(PrometheusConfig.DEFAULT);
        CustomMetricsExample example = new CustomMetricsExample(registry);

        // Симуляция запросов
        for (int i = 0; i < 100; i++) {
            example.handleRequest();
        }

        // Вывод метрик в формате Prometheus
        System.out.println(registry.scrape());
    }
}
```

### 2. Gauge (Измеритель)

Gauge — метрика, которая может как увеличиваться, так и уменьшаться. Используется для измерения текущего состояния.

**Примеры использования:**
- Текущее количество активных соединений
- Размер очереди
- Температура
- Использование памяти

#### Python

```python
from prometheus_client import Gauge, start_http_server
import threading
import time
import random

# Gauge для отслеживания активных соединений
active_connections = Gauge(
    'active_connections',
    'Количество активных соединений',
    ['service']
)

# Gauge для размера очереди
queue_size = Gauge(
    'queue_size',
    'Текущий размер очереди задач',
    ['queue_name']
)

# Gauge для использования памяти (в байтах)
memory_usage_bytes = Gauge(
    'memory_usage_bytes',
    'Использование памяти приложением'
)

class ConnectionPool:
    def __init__(self, service_name):
        self.service_name = service_name
        self.connections = 0

    def connect(self):
        self.connections += 1
        active_connections.labels(service=self.service_name).set(self.connections)

    def disconnect(self):
        self.connections -= 1
        active_connections.labels(service=self.service_name).set(self.connections)

class TaskQueue:
    def __init__(self, name):
        self.name = name
        self.tasks = []

    def add_task(self, task):
        self.tasks.append(task)
        queue_size.labels(queue_name=self.name).set(len(self.tasks))

    def process_task(self):
        if self.tasks:
            task = self.tasks.pop(0)
            queue_size.labels(queue_name=self.name).set(len(self.tasks))
            return task
        return None

# Использование декоратора in_progress для отслеживания выполняющихся функций
in_progress_requests = Gauge(
    'in_progress_requests',
    'Количество запросов в обработке'
)

@in_progress_requests.track_inprogress()
def process_request():
    """Функция автоматически увеличивает gauge при входе и уменьшает при выходе"""
    time.sleep(random.uniform(0.1, 0.5))

if __name__ == '__main__':
    start_http_server(8000)

    pool = ConnectionPool('database')
    queue = TaskQueue('email')

    while True:
        # Симуляция работы
        if random.random() > 0.5:
            pool.connect()
        else:
            pool.disconnect() if pool.connections > 0 else None

        queue.add_task(f"task_{random.randint(1, 100)}")
        if random.random() > 0.3:
            queue.process_task()

        time.sleep(0.5)
```

#### Go

```go
package main

import (
    "math/rand"
    "net/http"
    "sync"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    activeConnections = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "active_connections",
            Help: "Количество активных соединений",
        },
        []string{"service"},
    )

    queueSize = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "queue_size",
            Help: "Текущий размер очереди задач",
        },
        []string{"queue_name"},
    )

    inProgressRequests = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "in_progress_requests",
            Help: "Количество запросов в обработке",
        },
    )
)

type ConnectionPool struct {
    mu          sync.Mutex
    connections int
    service     string
}

func (p *ConnectionPool) Connect() {
    p.mu.Lock()
    defer p.mu.Unlock()
    p.connections++
    activeConnections.WithLabelValues(p.service).Set(float64(p.connections))
}

func (p *ConnectionPool) Disconnect() {
    p.mu.Lock()
    defer p.mu.Unlock()
    if p.connections > 0 {
        p.connections--
        activeConnections.WithLabelValues(p.service).Set(float64(p.connections))
    }
}

func processRequest() {
    inProgressRequests.Inc()
    defer inProgressRequests.Dec()

    // Симуляция обработки
    time.Sleep(time.Duration(rand.Intn(500)) * time.Millisecond)
}

func main() {
    http.Handle("/metrics", promhttp.Handler())

    pool := &ConnectionPool{service: "database"}

    go func() {
        for {
            if rand.Float64() > 0.5 {
                pool.Connect()
            } else {
                pool.Disconnect()
            }
            time.Sleep(500 * time.Millisecond)
        }
    }()

    http.ListenAndServe(":8000", nil)
}
```

### 3. Histogram (Гистограмма)

Histogram — метрика для измерения распределения значений. Автоматически предоставляет бакеты (buckets), сумму и количество.

**Примеры использования:**
- Время ответа API
- Размер запросов
- Длительность выполнения задач

#### Python

```python
from prometheus_client import Histogram, start_http_server
import time
import random

# Гистограмма для времени ответа HTTP
http_request_duration_seconds = Histogram(
    'http_request_duration_seconds',
    'Длительность HTTP запроса в секундах',
    ['method', 'endpoint'],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

# Гистограмма для размера ответа
http_response_size_bytes = Histogram(
    'http_response_size_bytes',
    'Размер HTTP ответа в байтах',
    ['endpoint'],
    buckets=[100, 500, 1000, 5000, 10000, 50000, 100000, 500000, 1000000]
)

def process_request(method, endpoint):
    """Обработка запроса с измерением времени"""

    # Способ 1: Ручное измерение
    start_time = time.time()

    # Симуляция обработки
    time.sleep(random.uniform(0.01, 0.5))
    response_size = random.randint(100, 10000)

    duration = time.time() - start_time
    http_request_duration_seconds.labels(
        method=method,
        endpoint=endpoint
    ).observe(duration)

    http_response_size_bytes.labels(endpoint=endpoint).observe(response_size)

    return response_size

# Способ 2: Использование декоратора
@http_request_duration_seconds.labels(method='GET', endpoint='/api/items').time()
def get_items():
    """Декоратор автоматически измеряет время выполнения"""
    time.sleep(random.uniform(0.01, 0.3))
    return [{'id': i} for i in range(10)]

# Способ 3: Контекстный менеджер
def update_item(item_id):
    with http_request_duration_seconds.labels(
        method='PUT',
        endpoint='/api/items'
    ).time():
        time.sleep(random.uniform(0.05, 0.2))
        return {'id': item_id, 'updated': True}

if __name__ == '__main__':
    start_http_server(8000)

    while True:
        process_request('GET', '/api/users')
        get_items()
        update_item(random.randint(1, 100))
        time.sleep(0.1)
```

#### Go

```go
package main

import (
    "math/rand"
    "net/http"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "Длительность HTTP запроса в секундах",
            Buckets: []float64{0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0},
        },
        []string{"method", "endpoint"},
    )

    httpResponseSize = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_response_size_bytes",
            Help:    "Размер HTTP ответа в байтах",
            Buckets: prometheus.ExponentialBuckets(100, 2, 10), // 100, 200, 400, 800...
        },
        []string{"endpoint"},
    )
)

func processRequest(method, endpoint string) {
    start := time.Now()

    // Симуляция обработки
    time.Sleep(time.Duration(rand.Intn(500)) * time.Millisecond)
    responseSize := rand.Intn(10000) + 100

    duration := time.Since(start).Seconds()
    httpRequestDuration.WithLabelValues(method, endpoint).Observe(duration)
    httpResponseSize.WithLabelValues(endpoint).Observe(float64(responseSize))
}

// Middleware для автоматического измерения
func instrumentHandler(handler http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        timer := prometheus.NewTimer(
            httpRequestDuration.WithLabelValues(r.Method, r.URL.Path),
        )
        defer timer.ObserveDuration()

        handler.ServeHTTP(w, r)
    })
}

func main() {
    http.Handle("/metrics", promhttp.Handler())

    go func() {
        for {
            processRequest("GET", "/api/users")
            time.Sleep(100 * time.Millisecond)
        }
    }()

    http.ListenAndServe(":8000", nil)
}
```

### 4. Summary (Саммари)

Summary похож на Histogram, но вычисляет квантили на стороне клиента. Менее гибкий для агрегации, но точнее для конкретных процентилей.

#### Python

```python
from prometheus_client import Summary, start_http_server
import time
import random

# Summary с квантилями
request_latency = Summary(
    'request_latency_seconds',
    'Латентность запросов',
    ['service']
)

# Summary с кастомными квантилями (требует prometheus_client >= 0.15)
# Примечание: в стандартной библиотеке Python квантили не настраиваются,
# они вычисляются на стороне Prometheus

@request_latency.labels(service='api').time()
def api_request():
    time.sleep(random.uniform(0.01, 0.5))

if __name__ == '__main__':
    start_http_server(8000)

    while True:
        api_request()
        time.sleep(0.1)
```

#### Go

```go
package main

import (
    "math/rand"
    "net/http"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    requestLatency = promauto.NewSummaryVec(
        prometheus.SummaryOpts{
            Name: "request_latency_seconds",
            Help: "Латентность запросов",
            Objectives: map[float64]float64{
                0.5:  0.05,  // 50-й процентиль с погрешностью 5%
                0.9:  0.01,  // 90-й процентиль с погрешностью 1%
                0.99: 0.001, // 99-й процентиль с погрешностью 0.1%
            },
        },
        []string{"service"},
    )
)

func apiRequest() {
    timer := prometheus.NewTimer(requestLatency.WithLabelValues("api"))
    defer timer.ObserveDuration()

    time.Sleep(time.Duration(rand.Intn(500)) * time.Millisecond)
}

func main() {
    http.Handle("/metrics", promhttp.Handler())

    go func() {
        for {
            apiRequest()
            time.Sleep(100 * time.Millisecond)
        }
    }()

    http.ListenAndServe(":8000", nil)
}
```

## Best Practices (Лучшие практики)

### Именование метрик

```python
# ПРАВИЛЬНО: snake_case, понятные имена с единицами измерения
http_requests_total                    # счётчик запросов
http_request_duration_seconds          # длительность в секундах
process_memory_bytes                   # память в байтах
queue_length                           # длина очереди (без единиц)

# НЕПРАВИЛЬНО
httpRequests                           # camelCase
request_time                           # неясные единицы измерения
http_request_duration_milliseconds     # миллисекунды (лучше секунды)
RequestCount                           # PascalCase
```

### Использование Labels

```python
# ПРАВИЛЬНО: ограниченная кардинальность
http_requests_total.labels(method='GET', endpoint='/api/users', status='200')

# НЕПРАВИЛЬНО: высокая кардинальность
http_requests_total.labels(
    user_id='12345',           # Уникальные значения!
    request_id='abc-123-xyz',  # Каждый запрос уникален!
    timestamp='1234567890'     # Бесконечная кардинальность!
)
```

### Структурирование метрик

```python
from prometheus_client import Counter, Histogram, Gauge, CollectorRegistry

class ServiceMetrics:
    """Класс для группировки метрик сервиса"""

    def __init__(self, registry: CollectorRegistry = None):
        registry = registry or CollectorRegistry()

        # HTTP метрики
        self.http_requests = Counter(
            'http_requests_total',
            'Общее количество HTTP запросов',
            ['method', 'endpoint', 'status'],
            registry=registry
        )

        self.http_duration = Histogram(
            'http_request_duration_seconds',
            'Длительность HTTP запросов',
            ['method', 'endpoint'],
            registry=registry
        )

        # Бизнес-метрики
        self.orders_created = Counter(
            'orders_created_total',
            'Количество созданных заказов',
            ['payment_method'],
            registry=registry
        )

        self.orders_in_progress = Gauge(
            'orders_in_progress',
            'Количество заказов в обработке',
            registry=registry
        )

    def record_request(self, method, endpoint, status, duration):
        """Запись метрик HTTP запроса"""
        self.http_requests.labels(
            method=method,
            endpoint=endpoint,
            status=status
        ).inc()

        self.http_duration.labels(
            method=method,
            endpoint=endpoint
        ).observe(duration)
```

## Частые ошибки

### 1. Высокая кардинальность labels

```python
# ОШИБКА: user_id создаёт миллионы уникальных комбинаций
http_requests.labels(user_id=user.id).inc()

# ПРАВИЛЬНО: используйте агрегированные категории
http_requests.labels(user_type=user.subscription_type).inc()
```

### 2. Использование Counter вместо Gauge

```python
# ОШИБКА: активные сессии могут уменьшаться
active_sessions = Counter('active_sessions', '...')  # Counter только увеличивается!

# ПРАВИЛЬНО
active_sessions = Gauge('active_sessions', '...')
```

### 3. Неправильные бакеты гистограммы

```python
# ОШИБКА: слишком большие бакеты для времени ответа API
http_duration = Histogram(
    'http_request_duration_seconds',
    'Duration',
    buckets=[1, 5, 10, 30, 60]  # Секунды? Для API это слишком много!
)

# ПРАВИЛЬНО: бакеты соответствуют ожидаемому времени ответа
http_duration = Histogram(
    'http_request_duration_seconds',
    'Duration',
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5]
)
```

### 4. Забытые метрики при ошибках

```python
# ОШИБКА: метрики не записываются при исключении
def process():
    start = time.time()
    try:
        do_work()
        duration.observe(time.time() - start)  # Пропускается при ошибке!
    except Exception:
        raise

# ПРАВИЛЬНО: используйте finally или контекстный менеджер
def process():
    start = time.time()
    try:
        do_work()
    finally:
        duration.observe(time.time() - start)
```

## Полезные ссылки

- [Prometheus Best Practices](https://prometheus.io/docs/practices/naming/)
- [Prometheus Python Client](https://github.com/prometheus/client_python)
- [Prometheus Go Client](https://github.com/prometheus/client_golang)
- [Micrometer Documentation](https://micrometer.io/docs)
