# Архитектура Prometheus

## Введение

Prometheus — это система мониторинга и оповещения с открытым исходным кодом, изначально разработанная в SoundCloud. С 2016 года Prometheus является частью Cloud Native Computing Foundation (CNCF). Prometheus использует pull-модель сбора метрик, что означает, что сервер Prometheus активно опрашивает целевые системы для получения метрик.

## Основные компоненты архитектуры

### 1. Prometheus Server

Prometheus Server — это ядро системы, которое выполняет три основные функции:

```
┌─────────────────────────────────────────────────────────┐
│                   PROMETHEUS SERVER                      │
├─────────────────┬─────────────────┬─────────────────────┤
│   Retrieval     │    TSDB         │    HTTP Server      │
│   (Scraping)    │   (Storage)     │    (PromQL)         │
└─────────────────┴─────────────────┴─────────────────────┘
```

**Retrieval (Сбор данных):**
- Отвечает за обнаружение целей (service discovery)
- Выполняет HTTP-запросы к эндпоинтам `/metrics`
- Парсит метрики в формате Prometheus

**TSDB (Time Series Database):**
- Хранит метрики в локальной базе данных временных рядов
- Использует эффективное сжатие данных
- Поддерживает настраиваемый период хранения данных

**HTTP Server:**
- Предоставляет API для выполнения PromQL-запросов
- Обеспечивает веб-интерфейс для визуализации
- Позволяет интегрироваться с внешними системами (Grafana)

### 2. Экспортёры (Exporters)

Экспортёры — это агенты, которые собирают метрики из различных систем и предоставляют их в формате Prometheus.

```yaml
# Популярные экспортёры
node_exporter:       # Метрики операционной системы (CPU, RAM, Disk)
  port: 9100

postgres_exporter:   # Метрики PostgreSQL
  port: 9187

redis_exporter:      # Метрики Redis
  port: 9121

nginx_exporter:      # Метрики Nginx
  port: 9113

blackbox_exporter:   # Проверка доступности сервисов
  port: 9115
```

**Пример вывода Node Exporter:**

```text
# HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 1234567.89
node_cpu_seconds_total{cpu="0",mode="user"} 45678.12
node_cpu_seconds_total{cpu="0",mode="system"} 12345.67

# HELP node_memory_MemTotal_bytes Memory information field MemTotal_bytes.
# TYPE node_memory_MemTotal_bytes gauge
node_memory_MemTotal_bytes 1.6777216e+10

# HELP node_filesystem_avail_bytes Filesystem space available to non-root users.
# TYPE node_filesystem_avail_bytes gauge
node_filesystem_avail_bytes{device="/dev/sda1",mountpoint="/"} 5.36870912e+10
```

### 3. Pushgateway

Pushgateway используется для сценариев, когда pull-модель не подходит:

```
┌──────────────┐     push      ┌──────────────┐     pull     ┌────────────┐
│  Short-lived │ ──────────────▶│  Pushgateway │◀─────────────│ Prometheus │
│     Jobs     │               │              │              │   Server   │
└──────────────┘               └──────────────┘              └────────────┘
```

**Когда использовать Pushgateway:**
- Batch-задания (cron jobs)
- Короткоживущие задачи
- Задачи за NAT или firewall

**Пример отправки метрик в Pushgateway:**

```bash
# Отправка метрики через curl
echo "job_duration_seconds 42" | curl --data-binary @- \
  http://pushgateway:9091/metrics/job/backup_job/instance/server1

# Отправка нескольких метрик
cat <<EOF | curl --data-binary @- http://pushgateway:9091/metrics/job/backup_job
# TYPE backup_size_bytes gauge
backup_size_bytes 1234567890
# TYPE backup_duration_seconds gauge
backup_duration_seconds 120
EOF
```

### 4. Alertmanager

Alertmanager обрабатывает оповещения, отправленные сервером Prometheus:

```
┌────────────┐  alerts   ┌──────────────┐  notifications  ┌─────────────┐
│ Prometheus │ ─────────▶│ Alertmanager │ ───────────────▶│  Receivers  │
│   Server   │           │              │                 │ (Slack/PD)  │
└────────────┘           └──────────────┘                 └─────────────┘
```

**Основные функции Alertmanager:**
- **Группировка (Grouping)** — объединение связанных алертов
- **Подавление (Inhibition)** — скрытие менее важных алертов при наличии более критичных
- **Тишина (Silences)** — временное отключение алертов
- **Маршрутизация** — направление алертов в нужные каналы

### 5. Service Discovery

Prometheus поддерживает различные механизмы обнаружения целей:

```yaml
# Пример конфигурации service discovery
scrape_configs:
  # Статическая конфигурация
  - job_name: 'static_targets'
    static_configs:
      - targets: ['localhost:9090', 'localhost:9100']

  # Kubernetes service discovery
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

  # Consul service discovery
  - job_name: 'consul_services'
    consul_sd_configs:
      - server: 'consul:8500'
        services: []

  # AWS EC2 service discovery
  - job_name: 'ec2_instances'
    ec2_sd_configs:
      - region: us-east-1
        access_key: AKIAXXXXXXXX
        secret_key: XXXXXXXXXXXX
        port: 9100

  # DNS service discovery
  - job_name: 'dns_discovery'
    dns_sd_configs:
      - names: ['_prometheus._tcp.example.com']
        type: SRV
```

## Типы метрик

Prometheus поддерживает четыре основных типа метрик:

### Counter (Счётчик)

Монотонно возрастающее значение, которое может только увеличиваться или сбрасываться в ноль:

```python
# Python пример с prometheus_client
from prometheus_client import Counter

http_requests_total = Counter(
    'http_requests_total',
    'Total number of HTTP requests',
    ['method', 'endpoint', 'status']
)

# Использование
http_requests_total.labels(method='GET', endpoint='/api/users', status='200').inc()
```

### Gauge (Измеритель)

Значение, которое может произвольно увеличиваться и уменьшаться:

```python
from prometheus_client import Gauge

active_connections = Gauge(
    'active_connections',
    'Number of active connections'
)

# Использование
active_connections.inc()   # +1
active_connections.dec()   # -1
active_connections.set(42) # Установить значение
```

### Histogram (Гистограмма)

Распределение значений по заранее определённым бакетам:

```python
from prometheus_client import Histogram

request_latency = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
)

# Использование
with request_latency.time():
    process_request()
```

### Summary (Сводка)

Похожа на гистограмму, но вычисляет квантили на стороне клиента:

```python
from prometheus_client import Summary

request_latency_summary = Summary(
    'http_request_duration_seconds_summary',
    'HTTP request latency summary'
)

# Использование
@request_latency_summary.time()
def process_request():
    pass
```

## Поток данных в Prometheus

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DATA FLOW                                          │
│                                                                              │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                                 │
│  │ App with │   │  Node    │   │ Database │                                 │
│  │ /metrics │   │ Exporter │   │ Exporter │                                 │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘                                 │
│       │              │              │                                        │
│       └──────────────┴──────────────┘                                        │
│                      │                                                       │
│                      ▼ scrape (pull)                                         │
│              ┌───────────────┐                                               │
│              │   Prometheus  │                                               │
│              │    Server     │                                               │
│              └───────┬───────┘                                               │
│                      │                                                       │
│         ┌────────────┼────────────┐                                          │
│         │            │            │                                          │
│         ▼            ▼            ▼                                          │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐                                    │
│   │  TSDB    │ │ AlertMgr │ │  Grafana │                                    │
│   │ Storage  │ │          │ │          │                                    │
│   └──────────┘ └──────────┘ └──────────┘                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Формат метрик

Prometheus использует простой текстовый формат для метрик:

```text
# HELP metric_name Description of the metric
# TYPE metric_name type
metric_name{label1="value1",label2="value2"} value timestamp

# Пример
# HELP http_requests_total The total number of HTTP requests.
# TYPE http_requests_total counter
http_requests_total{method="post",code="200"} 1027 1395066363000
http_requests_total{method="post",code="400"} 3 1395066363000
```

## Хранение данных

### Локальное хранилище (TSDB)

```
data/
├── 01BKGV7JBM69T2G1BGBGM6KB12/  # Block directory
│   ├── chunks/                   # Compressed time series data
│   ├── index                     # Index file
│   ├── meta.json                 # Block metadata
│   └── tombstones                # Deleted data markers
├── 01BKGTZQ1SYQJTR4PB43C8PD98/
├── chunks_head/                  # In-memory chunk snapshots
├── wal/                          # Write-ahead log
└── lock                          # Lock file
```

### Remote Storage

Для долгосрочного хранения данных можно использовать remote storage:

```yaml
# prometheus.yml
remote_write:
  - url: "http://thanos-receive:19291/api/v1/receive"
    queue_config:
      max_samples_per_send: 1000
      batch_send_deadline: 5s

remote_read:
  - url: "http://thanos-query:10901/api/v1/read"
```

## Best Practices

### 1. Именование метрик

```text
# Правильно
http_requests_total           # Counter с суффиксом _total
http_request_duration_seconds # Histogram с единицей измерения
process_cpu_seconds_total     # Использование стандартных единиц

# Неправильно
httpRequests                  # CamelCase
http_requests_count           # Неконсистентный суффикс
http_request_duration_ms      # Нестандартная единица
```

### 2. Использование лейблов

```python
# Хорошо: лейблы с низкой кардинальностью
http_requests_total{method="GET", status="200", endpoint="/api/users"}

# Плохо: лейблы с высокой кардинальностью
http_requests_total{user_id="12345", session_id="abc123"}  # Избегайте!
```

### 3. Настройка retention

```yaml
# prometheus.yml или через флаги
--storage.tsdb.retention.time=15d   # Хранить данные 15 дней
--storage.tsdb.retention.size=50GB  # Ограничение по размеру
```

## Частые ошибки

1. **Высокая кардинальность лейблов** — использование уникальных значений (user_id, request_id) в лейблах приводит к экспоненциальному росту временных рядов

2. **Неправильный тип метрики** — использование Gauge вместо Counter для монотонно растущих значений

3. **Игнорирование staleness** — Prometheus помечает метрики как stale через 5 минут отсутствия скрейпинга

4. **Слишком частый scrape_interval** — частый сбор метрик увеличивает нагрузку и объём данных

5. **Отсутствие recording rules** — вычисление сложных запросов на лету вместо предварительного расчёта

## Заключение

Архитектура Prometheus спроектирована для надёжного мониторинга современных распределённых систем. Pull-модель сбора метрик, эффективное хранение временных рядов и мощный язык запросов PromQL делают Prometheus стандартом де-факто для мониторинга в cloud-native средах.
