# Push Gateways

## Введение

**Push Gateway** — это промежуточный компонент в архитектуре Prometheus, который позволяет отправлять (push) метрики вместо стандартного подхода с опросом (pull). Это особенно полезно для короткоживущих задач, batch-процессов и скриптов, которые завершаются до того, как Prometheus успеет собрать их метрики.

## Проблема короткоживущих задач

### Стандартная модель Pull

В обычной архитектуре Prometheus:
1. Prometheus периодически опрашивает (scrape) целевые эндпоинты
2. Интервал опроса обычно составляет 15-60 секунд
3. Приложение должно быть доступно в момент опроса

```
┌─────────────┐         scrape          ┌─────────────────┐
│  Prometheus │ ───────────────────────▶│   Application   │
│             │      каждые 15 сек      │  /metrics       │
└─────────────┘                         └─────────────────┘
```

### Проблема

Если задача выполняется менее 15 секунд, Prometheus может не успеть собрать её метрики:

```
Время:     0s         5s         10s        15s        30s
           │          │          │          │          │
Задача:    ████████████
           start      end
                                            │
                                     Prometheus scrape
                                     (задача уже завершена!)
```

## Решение: Push Gateway

Push Gateway действует как промежуточный кэш метрик:

```
┌─────────────┐                    ┌──────────────┐                ┌─────────────────┐
│ Batch Job   │ ──── push ────────▶│ Push Gateway │◀──── scrape ───│   Prometheus    │
│ (5 секунд)  │                    │              │                │                 │
└─────────────┘                    └──────────────┘                └─────────────────┘
       │                                  │
       │                                  │
       ▼                                  ▼
   Задача                          Метрики хранятся
   завершается                     до следующего push
```

## Установка и запуск Push Gateway

### Docker

```bash
# Запуск Push Gateway
docker run -d \
  --name pushgateway \
  -p 9091:9091 \
  prom/pushgateway

# Проверка работоспособности
curl http://localhost:9091/metrics
```

### Docker Compose

```yaml
version: '3.8'

services:
  pushgateway:
    image: prom/pushgateway:latest
    container_name: pushgateway
    ports:
      - "9091:9091"
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    restart: unless-stopped
```

### Конфигурация Prometheus для scrape Push Gateway

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'pushgateway'
    honor_labels: true  # Важно! Сохраняет оригинальные labels от job
    static_configs:
      - targets: ['pushgateway:9091']
```

## Отправка метрик в Push Gateway

### Python

```python
from prometheus_client import CollectorRegistry, Counter, Gauge, Histogram, push_to_gateway
import time
import random

def run_batch_job():
    """Пример batch-задачи с отправкой метрик в Push Gateway"""

    # Создаём отдельный реестр для batch-задачи
    registry = CollectorRegistry()

    # Определяем метрики
    job_duration = Gauge(
        'batch_job_duration_seconds',
        'Длительность выполнения batch-задачи',
        registry=registry
    )

    records_processed = Counter(
        'batch_job_records_processed_total',
        'Количество обработанных записей',
        ['status'],
        registry=registry
    )

    processing_time = Histogram(
        'batch_job_record_processing_seconds',
        'Время обработки одной записи',
        registry=registry,
        buckets=[0.001, 0.005, 0.01, 0.05, 0.1, 0.5]
    )

    job_last_success = Gauge(
        'batch_job_last_success_timestamp',
        'Временная метка последнего успешного выполнения',
        registry=registry
    )

    # Выполняем работу
    start_time = time.time()

    try:
        for i in range(100):
            record_start = time.time()

            # Симуляция обработки записи
            time.sleep(random.uniform(0.001, 0.01))

            # Симуляция ошибок (10% записей)
            if random.random() < 0.1:
                records_processed.labels(status='error').inc()
            else:
                records_processed.labels(status='success').inc()

            processing_time.observe(time.time() - record_start)

        # Успешное завершение
        job_last_success.set(time.time())

    finally:
        # Записываем общую длительность
        job_duration.set(time.time() - start_time)

        # Отправляем метрики в Push Gateway
        push_to_gateway(
            'localhost:9091',           # Адрес Push Gateway
            job='daily_report_job',     # Имя задачи
            registry=registry,
            grouping_key={              # Дополнительные ключи группировки
                'instance': 'worker-1'
            }
        )

    print("Batch job completed, metrics pushed!")

if __name__ == '__main__':
    run_batch_job()
```

### Удаление метрик после успешного выполнения

```python
from prometheus_client import delete_from_gateway

def cleanup_metrics():
    """Удаление метрик из Push Gateway после обработки"""
    delete_from_gateway(
        'localhost:9091',
        job='daily_report_job',
        grouping_key={'instance': 'worker-1'}
    )
```

### Go

```go
package main

import (
    "fmt"
    "math/rand"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/push"
)

func main() {
    // Создаём реестр
    registry := prometheus.NewRegistry()

    // Определяем метрики
    jobDuration := prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "batch_job_duration_seconds",
        Help: "Длительность выполнения batch-задачи",
    })

    recordsProcessed := prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "batch_job_records_processed_total",
            Help: "Количество обработанных записей",
        },
        []string{"status"},
    )

    processingTime := prometheus.NewHistogram(prometheus.HistogramOpts{
        Name:    "batch_job_record_processing_seconds",
        Help:    "Время обработки одной записи",
        Buckets: []float64{0.001, 0.005, 0.01, 0.05, 0.1, 0.5},
    })

    lastSuccess := prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "batch_job_last_success_timestamp",
        Help: "Временная метка последнего успешного выполнения",
    })

    // Регистрируем метрики
    registry.MustRegister(jobDuration, recordsProcessed, processingTime, lastSuccess)

    // Выполняем работу
    startTime := time.Now()

    for i := 0; i < 100; i++ {
        recordStart := time.Now()

        // Симуляция обработки
        time.Sleep(time.Duration(rand.Intn(10)) * time.Millisecond)

        // Симуляция ошибок
        if rand.Float64() < 0.1 {
            recordsProcessed.WithLabelValues("error").Inc()
        } else {
            recordsProcessed.WithLabelValues("success").Inc()
        }

        processingTime.Observe(time.Since(recordStart).Seconds())
    }

    // Записываем результаты
    jobDuration.Set(time.Since(startTime).Seconds())
    lastSuccess.SetToCurrentTime()

    // Отправляем в Push Gateway
    pusher := push.New("http://localhost:9091", "daily_report_job").
        Grouping("instance", "worker-1").
        Gatherer(registry)

    if err := pusher.Push(); err != nil {
        fmt.Printf("Could not push to Pushgateway: %v\n", err)
    } else {
        fmt.Println("Metrics pushed successfully!")
    }
}
```

### Java

```java
import io.prometheus.client.CollectorRegistry;
import io.prometheus.client.Counter;
import io.prometheus.client.Gauge;
import io.prometheus.client.Histogram;
import io.prometheus.client.exporter.PushGateway;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;
import java.util.Random;

public class BatchJobWithPushGateway {

    public static void main(String[] args) {
        // Создаём отдельный реестр
        CollectorRegistry registry = new CollectorRegistry();

        // Определяем метрики
        Gauge jobDuration = Gauge.build()
            .name("batch_job_duration_seconds")
            .help("Длительность выполнения batch-задачи")
            .register(registry);

        Counter recordsProcessed = Counter.build()
            .name("batch_job_records_processed_total")
            .help("Количество обработанных записей")
            .labelNames("status")
            .register(registry);

        Histogram processingTime = Histogram.build()
            .name("batch_job_record_processing_seconds")
            .help("Время обработки одной записи")
            .buckets(0.001, 0.005, 0.01, 0.05, 0.1, 0.5)
            .register(registry);

        Gauge lastSuccess = Gauge.build()
            .name("batch_job_last_success_timestamp")
            .help("Временная метка последнего успешного выполнения")
            .register(registry);

        // Выполняем работу
        long startTime = System.currentTimeMillis();
        Random random = new Random();

        for (int i = 0; i < 100; i++) {
            long recordStart = System.nanoTime();

            try {
                // Симуляция обработки
                Thread.sleep(random.nextInt(10));
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }

            // Симуляция ошибок
            if (random.nextDouble() < 0.1) {
                recordsProcessed.labels("error").inc();
            } else {
                recordsProcessed.labels("success").inc();
            }

            double durationSeconds = (System.nanoTime() - recordStart) / 1_000_000_000.0;
            processingTime.observe(durationSeconds);
        }

        // Записываем результаты
        jobDuration.set((System.currentTimeMillis() - startTime) / 1000.0);
        lastSuccess.setToCurrentTime();

        // Отправляем в Push Gateway
        PushGateway pushGateway = new PushGateway("localhost:9091");

        Map<String, String> groupingKey = new HashMap<>();
        groupingKey.put("instance", "worker-1");

        try {
            pushGateway.pushAdd(registry, "daily_report_job", groupingKey);
            System.out.println("Metrics pushed successfully!");
        } catch (IOException e) {
            System.err.println("Could not push to Pushgateway: " + e.getMessage());
        }
    }
}
```

## Использование HTTP API напрямую

Push Gateway также поддерживает отправку метрик через HTTP API без клиентских библиотек:

### Bash / cURL

```bash
# Отправка простой метрики
cat <<EOF | curl --data-binary @- http://localhost:9091/metrics/job/my_batch_job
# TYPE my_job_duration_seconds gauge
my_job_duration_seconds 42.5
# TYPE my_job_records_processed counter
my_job_records_processed 1000
EOF

# Отправка метрик с labels
cat <<EOF | curl --data-binary @- http://localhost:9091/metrics/job/my_batch_job/instance/worker-1
# TYPE batch_job_success gauge
batch_job_success 1
# TYPE batch_job_last_run_timestamp gauge
batch_job_last_run_timestamp $(date +%s)
EOF

# Удаление метрик для конкретной job
curl -X DELETE http://localhost:9091/metrics/job/my_batch_job/instance/worker-1

# Удаление всех метрик для job
curl -X DELETE http://localhost:9091/metrics/job/my_batch_job
```

### Python (requests)

```python
import requests
import time

def push_metrics_raw():
    """Отправка метрик через HTTP API без prometheus_client"""

    job_name = "simple_batch_job"
    instance = "worker-1"

    metrics = f"""
# TYPE batch_job_duration_seconds gauge
batch_job_duration_seconds 15.5
# TYPE batch_job_success gauge
batch_job_success 1
# TYPE batch_job_last_run_timestamp gauge
batch_job_last_run_timestamp {time.time()}
# TYPE batch_job_records_total counter
batch_job_records_total 500
"""

    url = f"http://localhost:9091/metrics/job/{job_name}/instance/{instance}"

    response = requests.post(
        url,
        data=metrics.strip(),
        headers={'Content-Type': 'text/plain'}
    )

    if response.status_code == 200:
        print("Metrics pushed successfully!")
    else:
        print(f"Failed to push metrics: {response.status_code}")

if __name__ == '__main__':
    push_metrics_raw()
```

## Группировка метрик (Grouping Keys)

Push Gateway использует ключи группировки для идентификации источников метрик:

```python
from prometheus_client import push_to_gateway, CollectorRegistry, Gauge

registry = CollectorRegistry()
gauge = Gauge('example_metric', 'Example', registry=registry)
gauge.set(42)

# Базовая группировка по job
push_to_gateway('localhost:9091', job='my_job', registry=registry)

# Дополнительные ключи группировки
push_to_gateway(
    'localhost:9091',
    job='my_job',
    registry=registry,
    grouping_key={
        'instance': 'worker-1',
        'environment': 'production',
        'region': 'us-east-1'
    }
)
```

### URL-структура Push Gateway

```
POST /metrics/job/<JOB_NAME>[/label1/value1][/label2/value2]...

Примеры:
POST /metrics/job/backup_job
POST /metrics/job/backup_job/instance/db-server-1
POST /metrics/job/backup_job/instance/db-server-1/database/users
```

## push vs pushAdd

```python
from prometheus_client import push_to_gateway, pushadd_to_gateway

# push_to_gateway - ЗАМЕНЯЕТ все метрики для данной группы
push_to_gateway('localhost:9091', job='my_job', registry=registry)

# pushadd_to_gateway - ДОБАВЛЯЕТ/ОБНОВЛЯЕТ метрики, не удаляя существующие
pushadd_to_gateway('localhost:9091', job='my_job', registry=registry)
```

**Когда использовать:**
- `push` - для задач, которые генерируют полный набор метрик при каждом запуске
- `pushadd` - для инкрементального обновления метрик

## Best Practices

### 1. Используйте отдельный реестр

```python
# ПРАВИЛЬНО: отдельный реестр для каждой batch-задачи
from prometheus_client import CollectorRegistry, Gauge, push_to_gateway

registry = CollectorRegistry()
metric = Gauge('my_metric', 'Description', registry=registry)
push_to_gateway('localhost:9091', job='my_job', registry=registry)

# НЕПРАВИЛЬНО: использование глобального реестра
from prometheus_client import Gauge, REGISTRY, push_to_gateway

metric = Gauge('my_metric', 'Description')  # Глобальный реестр
push_to_gateway('localhost:9091', job='my_job', registry=REGISTRY)
# Отправит ВСЕ метрики процесса, включая системные!
```

### 2. Всегда устанавливайте timestamp последнего успеха

```python
last_success = Gauge(
    'job_last_success_timestamp',
    'Время последнего успешного выполнения',
    registry=registry
)

try:
    do_work()
    last_success.set(time.time())  # Только при успехе!
finally:
    push_to_gateway(...)
```

### 3. Используйте honor_labels в Prometheus

```yaml
scrape_configs:
  - job_name: 'pushgateway'
    honor_labels: true  # Сохраняет оригинальные labels от источника
    static_configs:
      - targets: ['pushgateway:9091']
```

### 4. Очищайте старые метрики

```python
from prometheus_client import delete_from_gateway

# После успешной обработки данных, если метрики больше не нужны
def cleanup_old_metrics():
    delete_from_gateway(
        'localhost:9091',
        job='old_job',
        grouping_key={'instance': 'old-worker'}
    )
```

### 5. Обрабатывайте ошибки при push

```python
import logging
from prometheus_client import push_to_gateway
from prometheus_client.exposition import basic_auth_handler
import urllib.error

def safe_push(registry, job_name, gateway='localhost:9091'):
    """Безопасная отправка метрик с обработкой ошибок"""
    try:
        push_to_gateway(gateway, job=job_name, registry=registry)
        logging.info(f"Metrics pushed successfully for job: {job_name}")
        return True
    except urllib.error.URLError as e:
        logging.error(f"Failed to push metrics: Connection error - {e}")
        return False
    except Exception as e:
        logging.error(f"Failed to push metrics: {e}")
        return False
```

## Частые ошибки

### 1. Использование Push Gateway для долгоживущих сервисов

```python
# НЕПРАВИЛЬНО: постоянно пушить метрики из работающего сервиса
while True:
    push_to_gateway(...)  # Это антипаттерн!
    time.sleep(15)

# ПРАВИЛЬНО: для долгоживущих сервисов используйте стандартный /metrics endpoint
from prometheus_client import start_http_server
start_http_server(8000)  # Prometheus сам будет scrape'ить
```

### 2. Забыть установить honor_labels

Без `honor_labels: true` Prometheus перезапишет label `job` своим значением из конфигурации.

### 3. Не очищать устаревшие метрики

```python
# Метрики остаются в Push Gateway навсегда, пока их не удалят!
# Это может привести к "зомби-метрикам" от давно завершённых задач

# Решение: удаляйте метрики после обработки или используйте TTL
```

### 4. Высокая нагрузка на Push Gateway

```python
# НЕПРАВИЛЬНО: push после каждой записи
for record in records:
    process(record)
    push_to_gateway(...)  # Слишком частые push!

# ПРАВИЛЬНО: один push в конце работы
for record in records:
    process(record)
push_to_gateway(...)  # Один раз
```

## Альтернативы Push Gateway

### 1. Prometheus Remote Write

Для высоконагруженных систем можно использовать remote write напрямую.

### 2. Statsd Exporter

Для приложений, которые уже используют StatsD протокол.

### 3. Textfile Collector (Node Exporter)

```bash
# Записываем метрики в файл
cat <<EOF > /var/lib/node_exporter/textfile_collector/batch_job.prom
# TYPE batch_job_success gauge
batch_job_success 1
# TYPE batch_job_duration_seconds gauge
batch_job_duration_seconds 42.5
EOF

# Node Exporter автоматически подхватит эти метрики
```

## Полезные ссылки

- [Prometheus Push Gateway](https://github.com/prometheus/pushgateway)
- [When to use the Push Gateway](https://prometheus.io/docs/practices/pushing/)
- [Push Gateway API](https://github.com/prometheus/pushgateway#api)
