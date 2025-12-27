# Другие экспортёры Prometheus

## Введение

Экосистема Prometheus включает сотни экспортёров для различных систем и сервисов. Экспортёры можно разделить на несколько категорий:

- **Инфраструктурные** — серверы, контейнеры, сети
- **Базы данных** — PostgreSQL, MySQL, MongoDB, Redis
- **Очереди сообщений** — Kafka, RabbitMQ
- **Веб-серверы и прокси** — Nginx, HAProxy, Apache
- **Мониторинг приложений** — Blackbox, JMX
- **Облачные сервисы** — AWS, GCP, Azure

В этом разделе мы рассмотрим наиболее популярные экспортёры.

## PostgreSQL Exporter

### Описание

**postgres_exporter** собирает метрики из PostgreSQL: производительность запросов, состояние репликации, использование ресурсов и другие.

### Установка

```bash
# Бинарный файл
wget https://github.com/prometheus-community/postgres_exporter/releases/download/v0.15.0/postgres_exporter-0.15.0.linux-amd64.tar.gz
tar xvfz postgres_exporter-0.15.0.linux-amd64.tar.gz
sudo mv postgres_exporter-0.15.0.linux-amd64/postgres_exporter /usr/local/bin/

# Docker
docker run -d \
  --name postgres-exporter \
  -p 9187:9187 \
  -e DATA_SOURCE_NAME="postgresql://user:password@postgres:5432/postgres?sslmode=disable" \
  quay.io/prometheuscommunity/postgres-exporter
```

### Конфигурация

```bash
# Через переменную окружения
export DATA_SOURCE_NAME="postgresql://prometheus:password@localhost:5432/postgres?sslmode=disable"
postgres_exporter

# Или через флаг
postgres_exporter --config.file=/etc/postgres_exporter/queries.yaml
```

### Создание пользователя в PostgreSQL

```sql
-- Создаём пользователя для мониторинга
CREATE USER prometheus WITH PASSWORD 'secure_password';

-- Даём права на чтение статистики
GRANT pg_monitor TO prometheus;

-- Или для старых версий PostgreSQL (<10)
GRANT SELECT ON pg_stat_database TO prometheus;
GRANT SELECT ON pg_stat_replication TO prometheus;
GRANT SELECT ON pg_stat_activity TO prometheus;
```

### Пользовательские запросы

```yaml
# /etc/postgres_exporter/queries.yaml
pg_postmaster:
  query: "SELECT pg_postmaster_start_time as start_time_seconds from pg_postmaster_start_time()"
  master: true
  metrics:
    - start_time_seconds:
        usage: "GAUGE"
        description: "Time at which postmaster started"

pg_database_size:
  query: "SELECT datname, pg_database_size(datname) as size_bytes FROM pg_database WHERE datistemplate = false"
  metrics:
    - datname:
        usage: "LABEL"
        description: "Database name"
    - size_bytes:
        usage: "GAUGE"
        description: "Database size in bytes"
```

### Основные метрики

| Метрика | Описание |
|---------|----------|
| `pg_up` | Доступность PostgreSQL |
| `pg_stat_activity_count` | Количество подключений |
| `pg_stat_database_tup_fetched` | Строк прочитано |
| `pg_stat_database_tup_inserted` | Строк вставлено |
| `pg_stat_replication_lag` | Отставание репликации |
| `pg_settings_max_connections` | Максимальное количество подключений |

### PromQL запросы

```promql
# Использование подключений
pg_stat_activity_count{datname="mydb"} / pg_settings_max_connections

# Запросов в секунду
rate(pg_stat_database_xact_commit[5m]) + rate(pg_stat_database_xact_rollback[5m])

# Отставание репликации в байтах
pg_replication_lag_bytes
```

## MySQL Exporter

### Установка

```bash
# Docker
docker run -d \
  --name mysql-exporter \
  -p 9104:9104 \
  -e DATA_SOURCE_NAME="exporter:password@(mysql:3306)/" \
  prom/mysqld-exporter
```

### Создание пользователя MySQL

```sql
CREATE USER 'exporter'@'%' IDENTIFIED BY 'password';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';
FLUSH PRIVILEGES;
```

### Основные метрики

```promql
# QPS (Queries Per Second)
rate(mysql_global_status_queries[5m])

# Медленные запросы
rate(mysql_global_status_slow_queries[5m])

# Использование подключений
mysql_global_status_threads_connected / mysql_global_variables_max_connections

# Размер баз данных
mysql_info_schema_table_size_bytes
```

## Redis Exporter

### Установка

```bash
# Docker
docker run -d \
  --name redis-exporter \
  -p 9121:9121 \
  oliver006/redis_exporter:latest \
  --redis.addr=redis://redis:6379

# С паролем
docker run -d \
  --name redis-exporter \
  -p 9121:9121 \
  -e REDIS_ADDR=redis://redis:6379 \
  -e REDIS_PASSWORD=your_password \
  oliver006/redis_exporter:latest
```

### Основные метрики

| Метрика | Описание |
|---------|----------|
| `redis_up` | Доступность Redis |
| `redis_connected_clients` | Подключённые клиенты |
| `redis_memory_used_bytes` | Используемая память |
| `redis_commands_processed_total` | Обработанных команд |
| `redis_keyspace_hits_total` | Попаданий в кэш |
| `redis_keyspace_misses_total` | Промахов кэша |

### PromQL запросы

```promql
# Hit rate (попадания в кэш)
rate(redis_keyspace_hits_total[5m]) / (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))

# Команд в секунду
rate(redis_commands_processed_total[5m])

# Использование памяти
redis_memory_used_bytes / redis_memory_max_bytes
```

## MongoDB Exporter

### Установка

```bash
docker run -d \
  --name mongodb-exporter \
  -p 9216:9216 \
  percona/mongodb_exporter:0.40 \
  --mongodb.uri=mongodb://user:password@mongo:27017
```

### Основные метрики

```promql
# Операции в секунду
rate(mongodb_op_counters_total[5m])

# Текущие подключения
mongodb_connections{state="current"}

# Размер данных
mongodb_dbstats_dataSize
```

## Kafka Exporter

### Установка

```bash
docker run -d \
  --name kafka-exporter \
  -p 9308:9308 \
  danielqsj/kafka-exporter:latest \
  --kafka.server=kafka:9092
```

### Основные метрики

| Метрика | Описание |
|---------|----------|
| `kafka_consumergroup_lag` | Отставание consumer group |
| `kafka_topic_partitions` | Количество партиций топика |
| `kafka_brokers` | Количество брокеров |
| `kafka_topic_partition_current_offset` | Текущий offset партиции |

### PromQL запросы

```promql
# Consumer lag (отставание)
kafka_consumergroup_lag

# Сообщений в секунду
rate(kafka_topic_partition_current_offset[5m])

# Топики с высоким лагом
kafka_consumergroup_lag > 1000
```

## RabbitMQ Exporter

RabbitMQ имеет встроенную поддержку Prometheus через плагин.

### Включение плагина

```bash
rabbitmq-plugins enable rabbitmq_prometheus
```

### Основные метрики

```promql
# Сообщений в очереди
rabbitmq_queue_messages

# Потребителей на очереди
rabbitmq_queue_consumers

# Сообщений в секунду
rate(rabbitmq_queue_messages_published_total[5m])

# Готовых к доставке
rabbitmq_queue_messages_ready
```

## Blackbox Exporter

### Описание

**Blackbox Exporter** позволяет проверять доступность внешних сервисов (HTTP, TCP, ICMP, DNS).

### Установка

```bash
docker run -d \
  --name blackbox-exporter \
  -p 9115:9115 \
  -v /etc/blackbox_exporter:/config \
  prom/blackbox-exporter:latest \
  --config.file=/config/blackbox.yml
```

### Конфигурация

```yaml
# /etc/blackbox_exporter/blackbox.yml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200, 201, 204]
      method: GET
      follow_redirects: true
      preferred_ip_protocol: "ip4"

  http_post_2xx:
    prober: http
    timeout: 5s
    http:
      method: POST
      headers:
        Content-Type: application/json
      body: '{"test": true}'

  tcp_connect:
    prober: tcp
    timeout: 5s

  icmp:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: "ip4"

  dns:
    prober: dns
    timeout: 5s
    dns:
      query_name: "example.com"
      query_type: "A"
```

### Конфигурация Prometheus

```yaml
scrape_configs:
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://google.com
          - https://example.com
          - https://api.myservice.com/health
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115

  - job_name: 'blackbox-tcp'
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
      - targets:
          - postgres:5432
          - redis:6379
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

### Основные метрики Blackbox

```promql
# Успешность проверки
probe_success

# Время ответа
probe_duration_seconds

# HTTP статус код
probe_http_status_code

# Срок действия SSL сертификата
probe_ssl_earliest_cert_expiry - time()

# DNS lookup время
probe_dns_lookup_time_seconds
```

### Алерты для Blackbox

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
        expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "SSL certificate expires in {{ $value }} days"

      - alert: SlowEndpoint
        expr: probe_duration_seconds > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Endpoint {{ $labels.instance }} is slow"
```

## Pushgateway

### Описание

**Pushgateway** позволяет пушить метрики в Prometheus для короткоживущих задач (cron jobs, batch jobs), которые не могут быть опрошены стандартным pull-методом.

### Установка

```bash
docker run -d \
  --name pushgateway \
  -p 9091:9091 \
  prom/pushgateway:latest
```

### Отправка метрик

```bash
# Простая метрика
echo "my_job_last_success $(date +%s)" | curl --data-binary @- http://pushgateway:9091/metrics/job/my_batch_job

# Несколько метрик
cat <<EOF | curl --data-binary @- http://pushgateway:9091/metrics/job/backup/instance/db01
# TYPE backup_duration_seconds gauge
backup_duration_seconds 125.5
# TYPE backup_size_bytes gauge
backup_size_bytes 1073741824
# TYPE backup_last_success_timestamp gauge
backup_last_success_timestamp $(date +%s)
EOF

# С дополнительными метками
echo "batch_job_duration_seconds 23.5" | curl --data-binary @- http://pushgateway:9091/metrics/job/my_job/instance/server1/status/success
```

### Пример скрипта для batch job

```bash
#!/bin/bash
# backup.sh

PUSHGATEWAY_URL="http://pushgateway:9091"
JOB_NAME="database_backup"
INSTANCE="$(hostname)"

# Фиксируем время начала
start_time=$(date +%s)

# Выполняем бэкап
if pg_dump mydb > /backup/mydb_$(date +%Y%m%d).sql; then
    status="success"
    backup_size=$(stat -c%s /backup/mydb_$(date +%Y%m%d).sql)
else
    status="failure"
    backup_size=0
fi

# Вычисляем длительность
end_time=$(date +%s)
duration=$((end_time - start_time))

# Отправляем метрики
cat <<EOF | curl --data-binary @- "${PUSHGATEWAY_URL}/metrics/job/${JOB_NAME}/instance/${INSTANCE}"
# TYPE backup_duration_seconds gauge
backup_duration_seconds ${duration}
# TYPE backup_size_bytes gauge
backup_size_bytes ${backup_size}
# TYPE backup_last_run_timestamp gauge
backup_last_run_timestamp ${end_time}
# TYPE backup_success gauge
backup_success $([ "$status" = "success" ] && echo 1 || echo 0)
EOF
```

### Конфигурация Prometheus

```yaml
scrape_configs:
  - job_name: 'pushgateway'
    honor_labels: true
    static_configs:
      - targets: ['pushgateway:9091']
```

> **Важно**: `honor_labels: true` необходим, чтобы метки из pushgateway не перезаписывались.

### Удаление метрик

```bash
# Удалить все метрики job
curl -X DELETE http://pushgateway:9091/metrics/job/my_job

# Удалить метрики конкретного instance
curl -X DELETE http://pushgateway:9091/metrics/job/my_job/instance/server1
```

## cAdvisor (Container Advisor)

### Описание

**cAdvisor** собирает метрики контейнеров: CPU, память, сеть, I/O.

### Установка

```bash
docker run -d \
  --name cadvisor \
  -p 8080:8080 \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  gcr.io/cadvisor/cadvisor:latest
```

### Основные метрики

```promql
# CPU использование контейнера
rate(container_cpu_usage_seconds_total{name="my_container"}[5m])

# Память контейнера
container_memory_usage_bytes{name="my_container"}

# Сетевой трафик
rate(container_network_receive_bytes_total[5m])
rate(container_network_transmit_bytes_total[5m])
```

## JMX Exporter (для Java приложений)

### Описание

**JMX Exporter** экспортирует метрики Java приложений через JMX в формат Prometheus.

### Установка (как Java agent)

```bash
# Скачиваем
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar

# Запускаем Java приложение с агентом
java -javaagent:./jmx_prometheus_javaagent-0.20.0.jar=8080:config.yaml -jar myapp.jar
```

### Конфигурация

```yaml
# config.yaml
lowercaseOutputName: true
lowercaseOutputLabelNames: true

rules:
  # Стандартные JVM метрики
  - pattern: 'java.lang<type=Memory><(\w+)>(\w+)'
    name: jvm_memory_$1_$2
    type: GAUGE

  - pattern: 'java.lang<type=GarbageCollector, name=(.+)><>CollectionCount'
    name: jvm_gc_collection_count
    labels:
      gc: $1
    type: COUNTER

  # Tomcat метрики
  - pattern: 'Catalina<type=ThreadPool, name="(.+)"><>currentThreadCount'
    name: tomcat_threadpool_current_threads
    labels:
      pool: $1
```

## Таблица популярных экспортёров

| Экспортёр | Порт | Назначение |
|-----------|------|------------|
| Node Exporter | 9100 | ОС и железо |
| Nginx Exporter | 9113 | Nginx |
| PostgreSQL Exporter | 9187 | PostgreSQL |
| MySQL Exporter | 9104 | MySQL |
| Redis Exporter | 9121 | Redis |
| MongoDB Exporter | 9216 | MongoDB |
| Kafka Exporter | 9308 | Apache Kafka |
| Blackbox Exporter | 9115 | Внешние проверки |
| Pushgateway | 9091 | Push метрик |
| cAdvisor | 8080 | Docker контейнеры |
| HAProxy Exporter | 9101 | HAProxy |
| Elasticsearch Exporter | 9114 | Elasticsearch |

## Написание собственного экспортёра

Если для вашей системы нет готового экспортёра, вы можете написать свой.

### Пример на Python

```python
from prometheus_client import start_http_server, Gauge, Counter
import time
import random

# Определяем метрики
REQUEST_COUNT = Counter('myapp_requests_total', 'Total requests', ['method', 'endpoint'])
REQUEST_LATENCY = Gauge('myapp_request_latency_seconds', 'Request latency', ['endpoint'])
ACTIVE_USERS = Gauge('myapp_active_users', 'Number of active users')

def collect_metrics():
    """Сбор метрик из вашего приложения."""
    # Здесь логика сбора реальных метрик
    REQUEST_COUNT.labels(method='GET', endpoint='/api/users').inc()
    REQUEST_LATENCY.labels(endpoint='/api/users').set(random.uniform(0.1, 0.5))
    ACTIVE_USERS.set(random.randint(100, 500))

if __name__ == '__main__':
    # Запускаем HTTP сервер на порту 8000
    start_http_server(8000)
    print("Exporter running on port 8000")

    while True:
        collect_metrics()
        time.sleep(15)
```

### Пример на Go

```go
package main

import (
    "net/http"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    requestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "myapp_requests_total",
            Help: "Total number of requests",
        },
        []string{"method", "endpoint"},
    )

    activeConnections = prometheus.NewGauge(
        prometheus.GaugeOpts{
            Name: "myapp_active_connections",
            Help: "Number of active connections",
        },
    )
)

func init() {
    prometheus.MustRegister(requestsTotal)
    prometheus.MustRegister(activeConnections)
}

func main() {
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":8000", nil)
}
```

## Best Practices

### 1. Стандартизация портов

Придерживайтесь стандартных портов для экспортёров (9100 для Node, 9187 для PostgreSQL и т.д.).

### 2. Безопасность

```yaml
# Используйте TLS и аутентификацию
scrape_configs:
  - job_name: 'secure-exporter'
    scheme: https
    tls_config:
      ca_file: /etc/prometheus/ca.crt
    basic_auth:
      username: prometheus
      password_file: /etc/prometheus/password
```

### 3. Service Discovery

Вместо статических targets используйте Service Discovery:

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

## Полезные ссылки

- [Официальный список экспортёров](https://prometheus.io/docs/instrumenting/exporters/)
- [Prometheus Community GitHub](https://github.com/prometheus-community)
- [Awesome Prometheus](https://github.com/roaldnefs/awesome-prometheus)
