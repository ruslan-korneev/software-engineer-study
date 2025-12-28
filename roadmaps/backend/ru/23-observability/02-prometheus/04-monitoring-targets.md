# Мониторинг целей (Targets)

[prev: 03-promql-basics](./03-promql-basics.md) | [next: 01-node-exporter](../03-exporters/01-node-exporter.md)

---

## Введение

В Prometheus "цель" (target) — это эндпоинт, с которого сервер собирает метрики. Правильная настройка мониторинга целей включает выбор экспортёров, настройку service discovery, инструментирование приложений и создание информативных дашбордов.

## Экспортёры (Exporters)

Экспортёры — это агенты, которые собирают метрики из различных систем и предоставляют их в формате Prometheus.

### Node Exporter

Сбор метрик операционной системы Linux/Unix.

```bash
# Установка
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xvfz node_exporter-1.7.0.linux-amd64.tar.gz
./node_exporter

# Docker
docker run -d \
  --name node_exporter \
  --net host \
  --pid host \
  -v /:/host:ro,rslave \
  quay.io/prometheus/node-exporter:latest \
  --path.rootfs=/host
```

**Конфигурация в Prometheus:**

```yaml
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets:
          - 'server1:9100'
          - 'server2:9100'
    relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):\d+'
        target_label: instance
        replacement: '${1}'
```

**Основные метрики Node Exporter:**

```promql
# CPU
node_cpu_seconds_total{mode="idle"}
node_cpu_seconds_total{mode="user"}
node_cpu_seconds_total{mode="system"}
node_cpu_seconds_total{mode="iowait"}

# Memory
node_memory_MemTotal_bytes
node_memory_MemAvailable_bytes
node_memory_MemFree_bytes
node_memory_Buffers_bytes
node_memory_Cached_bytes
node_memory_SwapTotal_bytes
node_memory_SwapFree_bytes

# Disk
node_filesystem_size_bytes
node_filesystem_avail_bytes
node_filesystem_free_bytes
node_disk_read_bytes_total
node_disk_written_bytes_total
node_disk_io_time_seconds_total

# Network
node_network_receive_bytes_total
node_network_transmit_bytes_total
node_network_receive_packets_total
node_network_transmit_packets_total
node_network_receive_errs_total
node_network_transmit_errs_total

# Load
node_load1
node_load5
node_load15

# Uptime
node_boot_time_seconds
```

**Полезные запросы:**

```promql
# CPU Usage (%)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory Usage (%)
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Disk Usage (%)
(1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}
  / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"}) * 100

# Network Throughput (MB/s)
rate(node_network_receive_bytes_total[5m]) / 1024 / 1024
rate(node_network_transmit_bytes_total[5m]) / 1024 / 1024

# Uptime (days)
(time() - node_boot_time_seconds) / 86400
```

### Blackbox Exporter

Мониторинг доступности сервисов (HTTP, TCP, ICMP, DNS).

```yaml
# blackbox.yml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200, 201, 202, 204]
      method: GET
      follow_redirects: true
      fail_if_ssl: false
      fail_if_not_ssl: false
      tls_config:
        insecure_skip_verify: false

  http_post_2xx:
    prober: http
    timeout: 5s
    http:
      method: POST
      headers:
        Content-Type: application/json
      body: '{"test": "data"}'

  http_basic_auth:
    prober: http
    http:
      basic_auth:
        username: "user"
        password: "password"

  tcp_connect:
    prober: tcp
    timeout: 5s

  icmp_ping:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: ip4

  dns_lookup:
    prober: dns
    timeout: 5s
    dns:
      query_name: "example.com"
      query_type: "A"
```

**Конфигурация в Prometheus:**

```yaml
scrape_configs:
  - job_name: 'blackbox_http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://example.com
          - https://api.example.com/health
          - http://internal-service:8080
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115

  - job_name: 'blackbox_tcp'
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
      - targets:
          - redis:6379
          - postgres:5432
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

**Основные метрики Blackbox:**

```promql
# Доступность
probe_success

# Время ответа
probe_duration_seconds

# HTTP статус код
probe_http_status_code

# SSL информация
probe_ssl_earliest_cert_expiry
probe_http_ssl

# DNS
probe_dns_lookup_time_seconds
```

**Алерты для Blackbox:**

```yaml
groups:
  - name: blackbox_alerts
    rules:
      - alert: EndpointDown
        expr: probe_success == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Endpoint {{ $labels.instance }} is down"

      - alert: SSLCertExpiringSoon
        expr: probe_ssl_earliest_cert_expiry - time() < 86400 * 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "SSL certificate expires in less than 30 days"
          description: "Certificate for {{ $labels.instance }} expires in {{ $value | humanizeDuration }}"

      - alert: SlowEndpoint
        expr: probe_duration_seconds > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Endpoint {{ $labels.instance }} is slow"
```

### PostgreSQL Exporter

```bash
# Docker
docker run -d \
  --name postgres_exporter \
  -p 9187:9187 \
  -e DATA_SOURCE_NAME="postgresql://user:password@postgres:5432/database?sslmode=disable" \
  quay.io/prometheuscommunity/postgres-exporter
```

**Основные метрики:**

```promql
# Соединения
pg_stat_activity_count{state="active"}
pg_stat_activity_count{state="idle"}
pg_settings_max_connections

# Транзакции
pg_stat_database_xact_commit
pg_stat_database_xact_rollback

# Блокировки
pg_locks_count

# Размер БД
pg_database_size_bytes

# Репликация
pg_replication_lag
pg_stat_replication_pg_current_wal_lsn_bytes
pg_stat_replication_pg_wal_lsn_diff

# Cache hit ratio
pg_stat_database_blks_hit / (pg_stat_database_blks_hit + pg_stat_database_blks_read)
```

### Redis Exporter

```bash
docker run -d \
  --name redis_exporter \
  -p 9121:9121 \
  oliver006/redis_exporter \
  --redis.addr=redis://redis:6379
```

**Основные метрики:**

```promql
# Память
redis_memory_used_bytes
redis_memory_max_bytes

# Соединения
redis_connected_clients
redis_blocked_clients

# Операции
redis_commands_processed_total
redis_keyspace_hits_total
redis_keyspace_misses_total

# Ключи
redis_db_keys

# Репликация
redis_connected_slaves
redis_master_repl_offset
```

### Nginx Exporter

```bash
# nginx.conf - включение stub_status
server {
    listen 8080;
    location /nginx_status {
        stub_status on;
        allow 127.0.0.1;
        deny all;
    }
}

# Docker
docker run -d \
  --name nginx_exporter \
  -p 9113:9113 \
  nginx/nginx-prometheus-exporter:0.11.0 \
  -nginx.scrape-uri=http://nginx:8080/nginx_status
```

**Основные метрики:**

```promql
# Соединения
nginx_connections_active
nginx_connections_accepted
nginx_connections_handled
nginx_connections_reading
nginx_connections_writing
nginx_connections_waiting

# Запросы
nginx_http_requests_total
```

### MongoDB Exporter

```bash
docker run -d \
  --name mongodb_exporter \
  -p 9216:9216 \
  percona/mongodb_exporter:0.40 \
  --mongodb.uri=mongodb://user:password@mongodb:27017
```

### Kafka Exporter

```bash
docker run -d \
  --name kafka_exporter \
  -p 9308:9308 \
  danielqsj/kafka-exporter \
  --kafka.server=kafka:9092
```

**Основные метрики:**

```promql
# Lag консьюмеров
kafka_consumergroup_lag
kafka_consumergroup_current_offset

# Партиции
kafka_topic_partitions

# Брокеры
kafka_brokers
```

## Инструментирование приложений

### Python (prometheus_client)

```python
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time

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
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
)

ACTIVE_REQUESTS = Gauge(
    'http_requests_in_progress',
    'Number of HTTP requests in progress',
    ['method', 'endpoint']
)

# Использование в приложении
@app.route('/api/users')
def get_users():
    method = request.method
    endpoint = '/api/users'

    ACTIVE_REQUESTS.labels(method=method, endpoint=endpoint).inc()

    with REQUEST_LATENCY.labels(method=method, endpoint=endpoint).time():
        try:
            result = fetch_users()
            REQUEST_COUNT.labels(method=method, endpoint=endpoint, status='200').inc()
            return result
        except Exception as e:
            REQUEST_COUNT.labels(method=method, endpoint=endpoint, status='500').inc()
            raise
        finally:
            ACTIVE_REQUESTS.labels(method=method, endpoint=endpoint).dec()

# Запуск сервера метрик
if __name__ == '__main__':
    start_http_server(8000)  # Метрики на порту 8000
    app.run(port=5000)
```

### FastAPI с prometheus-fastapi-instrumentator

```python
from fastapi import FastAPI
from prometheus_fastapi_instrumentator import Instrumentator

app = FastAPI()

# Автоматическое инструментирование
Instrumentator().instrument(app).expose(app)

@app.get("/api/users")
async def get_users():
    return {"users": []}
```

### Go

```go
package main

import (
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
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )

    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "endpoint"},
    )
)

func metricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // Wrap ResponseWriter to capture status code
        wrapped := &statusRecorder{ResponseWriter: w, status: 200}
        next.ServeHTTP(wrapped, r)

        duration := time.Since(start).Seconds()

        httpRequestsTotal.WithLabelValues(
            r.Method,
            r.URL.Path,
            http.StatusText(wrapped.status),
        ).Inc()

        httpRequestDuration.WithLabelValues(
            r.Method,
            r.URL.Path,
        ).Observe(duration)
    })
}

func main() {
    http.Handle("/metrics", promhttp.Handler())
    http.Handle("/", metricsMiddleware(mainHandler))
    http.ListenAndServe(":8080", nil)
}
```

### Node.js (prom-client)

```javascript
const express = require('express');
const promClient = require('prom-client');

const app = express();

// Сбор стандартных метрик Node.js
promClient.collectDefaultMetrics();

// Кастомные метрики
const httpRequestsTotal = new promClient.Counter({
    name: 'http_requests_total',
    help: 'Total number of HTTP requests',
    labelNames: ['method', 'path', 'status']
});

const httpRequestDuration = new promClient.Histogram({
    name: 'http_request_duration_seconds',
    help: 'HTTP request duration',
    labelNames: ['method', 'path'],
    buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
});

// Middleware
app.use((req, res, next) => {
    const start = Date.now();

    res.on('finish', () => {
        const duration = (Date.now() - start) / 1000;

        httpRequestsTotal.labels(req.method, req.path, res.statusCode).inc();
        httpRequestDuration.labels(req.method, req.path).observe(duration);
    });

    next();
});

// Эндпоинт метрик
app.get('/metrics', async (req, res) => {
    res.set('Content-Type', promClient.register.contentType);
    res.send(await promClient.register.metrics());
});

app.listen(3000);
```

## Service Discovery

### Kubernetes Service Discovery

```yaml
scrape_configs:
  # Мониторинг подов
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Только поды с аннотацией prometheus.io/scrape=true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

      # Путь к метрикам из аннотации
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

      # Порт из аннотации
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__

      # Схема (http/https)
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)

      # Лейблы
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app

  # Мониторинг сервисов
  - job_name: 'kubernetes-services'
    kubernetes_sd_configs:
      - role: service
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_service_name]
        target_label: service

  # Мониторинг нод
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics
```

**Аннотации для пода:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/path: "/metrics"
    prometheus.io/port: "8080"
    prometheus.io/scheme: "http"
```

### Consul Service Discovery

```yaml
scrape_configs:
  - job_name: 'consul-services'
    consul_sd_configs:
      - server: 'consul.example.com:8500'
        services: []
        tags:
          - prometheus
    relabel_configs:
      - source_labels: [__meta_consul_service]
        target_label: job
      - source_labels: [__meta_consul_service_id]
        target_label: instance
      - source_labels: [__meta_consul_dc]
        target_label: datacenter
      - source_labels: [__meta_consul_tags]
        regex: .*,env=([^,]+),.*
        replacement: ${1}
        target_label: env
```

### AWS EC2 Service Discovery

```yaml
scrape_configs:
  - job_name: 'aws-ec2'
    ec2_sd_configs:
      - region: us-east-1
        access_key: AKIAXXXXXXXX
        secret_key: XXXXXXXXXXXX
        port: 9100
        filters:
          - name: tag:Environment
            values:
              - production
          - name: instance-state-name
            values:
              - running
    relabel_configs:
      - source_labels: [__meta_ec2_tag_Name]
        target_label: instance
      - source_labels: [__meta_ec2_instance_id]
        target_label: instance_id
      - source_labels: [__meta_ec2_availability_zone]
        target_label: availability_zone
```

### File-based Service Discovery

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'file_sd'
    file_sd_configs:
      - files:
          - '/etc/prometheus/targets/*.json'
          - '/etc/prometheus/targets/*.yml'
        refresh_interval: 30s
```

```json
// /etc/prometheus/targets/web-servers.json
[
  {
    "targets": ["web1.example.com:9100", "web2.example.com:9100"],
    "labels": {
      "env": "production",
      "role": "webserver",
      "team": "platform"
    }
  },
  {
    "targets": ["web-staging.example.com:9100"],
    "labels": {
      "env": "staging",
      "role": "webserver",
      "team": "platform"
    }
  }
]
```

## Создание дашбордов

### Структура дашборда

```
┌────────────────────────────────────────────────────────────────────┐
│                        Service Overview                             │
├─────────────────┬─────────────────┬─────────────────┬──────────────┤
│   Total RPS     │   Error Rate    │   P99 Latency   │   Uptime     │
│     1,234       │      0.5%       │     120ms       │   99.99%     │
├─────────────────┴─────────────────┴─────────────────┴──────────────┤
│                     Request Rate (RPS)                              │
│  ▁▂▃▄▅▆▇█▇▆▅▄▃▂▁▂▃▄▅▆▇█▇▆▅▄▃▂▁▂▃▄▅▆▇█▇▆▅▄▃▂▁                      │
├────────────────────────────────────────────────────────────────────┤
│                     Latency Distribution                            │
│  P50: 25ms  P90: 80ms  P99: 120ms  P999: 250ms                     │
├───────────────────────────────────┬────────────────────────────────┤
│         CPU Usage                 │       Memory Usage              │
│  ▂▃▄▅▆▇█▇▆▅▄▃▂▁▂▃▄               │  ▅▅▅▅▆▆▆▆▇▇▇▇█████              │
├───────────────────────────────────┴────────────────────────────────┤
│                    Active Instances                                 │
│  instance-1: ✓  instance-2: ✓  instance-3: ✓  instance-4: ✓        │
└────────────────────────────────────────────────────────────────────┘
```

### Grafana Dashboard JSON

```json
{
  "title": "Application Dashboard",
  "panels": [
    {
      "title": "Request Rate",
      "type": "timeseries",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{job=\"api\"}[5m]))",
          "legendFormat": "Total RPS"
        },
        {
          "expr": "sum by (status) (rate(http_requests_total{job=\"api\"}[5m]))",
          "legendFormat": "{{status}}"
        }
      ]
    },
    {
      "title": "Error Rate",
      "type": "stat",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{job=\"api\",status=~\"5..\"}[5m])) / sum(rate(http_requests_total{job=\"api\"}[5m])) * 100",
          "legendFormat": "Error %"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "thresholds": {
            "steps": [
              {"value": 0, "color": "green"},
              {"value": 1, "color": "yellow"},
              {"value": 5, "color": "red"}
            ]
          }
        }
      }
    },
    {
      "title": "Latency Percentiles",
      "type": "timeseries",
      "targets": [
        {
          "expr": "histogram_quantile(0.50, sum by (le) (rate(http_request_duration_seconds_bucket{job=\"api\"}[5m])))",
          "legendFormat": "P50"
        },
        {
          "expr": "histogram_quantile(0.90, sum by (le) (rate(http_request_duration_seconds_bucket{job=\"api\"}[5m])))",
          "legendFormat": "P90"
        },
        {
          "expr": "histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{job=\"api\"}[5m])))",
          "legendFormat": "P99"
        }
      ]
    }
  ]
}
```

## USE и RED методологии

### USE Method (Utilization, Saturation, Errors)

Для ресурсов (CPU, Memory, Disk, Network):

```promql
# CPU
# Utilization
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Saturation (load average vs CPU count)
node_load1 / count by (instance) (node_cpu_seconds_total{mode="idle"})

# Errors
rate(node_cpu_guest_seconds_total[5m])

# Memory
# Utilization
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Saturation
rate(node_vmstat_pgmajfault[5m])

# Errors
increase(node_memory_numa_alloc_fail[1h])

# Disk
# Utilization
rate(node_disk_io_time_seconds_total[5m])

# Saturation
node_disk_io_time_weighted_seconds_total

# Errors
rate(node_disk_read_errors_total[5m]) + rate(node_disk_write_errors_total[5m])
```

### RED Method (Rate, Errors, Duration)

Для сервисов:

```promql
# Rate (RPS)
sum(rate(http_requests_total[5m]))

# Errors (%)
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m])) * 100

# Duration (latency)
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
)
```

## Best Practices

### 1. Стандартизация лейблов

```yaml
# Используйте консистентные имена лейблов
job: "api-server"        # Имя сервиса
instance: "10.0.0.1:8080" # Адрес инстанса
env: "production"         # Окружение
region: "us-east-1"       # Регион
team: "platform"          # Команда-владелец
```

### 2. Группировка таргетов

```yaml
scrape_configs:
  - job_name: 'production-api'
    static_configs:
      - targets: ['api-1:8080', 'api-2:8080']
        labels:
          env: production

  - job_name: 'staging-api'
    static_configs:
      - targets: ['api-staging:8080']
        labels:
          env: staging
```

### 3. Использование Recording Rules

```yaml
groups:
  - name: slo_recording_rules
    rules:
      - record: job:slo:availability:ratio
        expr: |
          sum(rate(http_requests_total{status!~"5.."}[5m]))
          / sum(rate(http_requests_total[5m]))

      - record: job:slo:latency_p99:seconds
        expr: |
          histogram_quantile(0.99,
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          )
```

### 4. Настройка алертов

```yaml
groups:
  - name: slo_alerts
    rules:
      - alert: SLOAvailabilityBreach
        expr: job:slo:availability:ratio < 0.999
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "SLO availability breach for {{ $labels.job }}"
          description: "Availability is {{ $value | humanizePercentage }}"

      - alert: SLOLatencyBreach
        expr: job:slo:latency_p99:seconds > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "SLO latency breach for {{ $labels.job }}"
          description: "P99 latency is {{ $value | humanizeDuration }}"
```

## Частые ошибки

1. **Неправильный scrape_interval** — слишком частый сбор увеличивает нагрузку
2. **Отсутствие timeout** — зависшие таргеты блокируют скрейпинг
3. **Высокая кардинальность** — использование user_id, request_id в лейблах
4. **Игнорирование up метрики** — всегда мониторьте доступность таргетов
5. **Отсутствие relabeling** — некорректные лейблы усложняют запросы

## Отладка проблем с таргетами

```bash
# Проверка доступности таргета
curl http://target:port/metrics

# Проверка статуса в Prometheus UI
http://prometheus:9090/targets

# Проверка service discovery
http://prometheus:9090/service-discovery

# Проверка конфигурации
promtool check config prometheus.yml

# Логи Prometheus
journalctl -u prometheus -f
```

## Заключение

Правильная настройка мониторинга целей — основа эффективной системы наблюдаемости. Используйте подходящие экспортёры, настройте service discovery для автоматического обнаружения сервисов, инструментируйте свои приложения и создавайте информативные дашборды на основе USE/RED методологий.
