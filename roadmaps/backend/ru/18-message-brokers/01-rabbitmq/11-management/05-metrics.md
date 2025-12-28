# Метрики

[prev: 04-logging](./04-logging.md) | [next: 01-amqp-091](../12-protocols/01-amqp-091.md)

---

## Обзор

Метрики RabbitMQ предоставляют количественные данные о состоянии и производительности брокера. Правильный сбор и анализ метрик позволяет:

- Отслеживать производительность в реальном времени
- Выявлять узкие места и проблемы
- Планировать capacity
- Настраивать алерты для проактивного реагирования

## Источники метрик

### Management API

```bash
# Общая статистика
curl -u admin:password http://localhost:15672/api/overview

# Метрики нод
curl -u admin:password http://localhost:15672/api/nodes

# Метрики очередей
curl -u admin:password http://localhost:15672/api/queues

# Метрики соединений
curl -u admin:password http://localhost:15672/api/connections

# Метрики каналов
curl -u admin:password http://localhost:15672/api/channels
```

### Prometheus Endpoint

```bash
# Включение плагина
rabbitmq-plugins enable rabbitmq_prometheus

# Базовые метрики
curl http://localhost:15692/metrics

# Детальные метрики по объектам
curl http://localhost:15692/metrics/per-object

# Метрики в JSON (для отладки)
curl http://localhost:15692/metrics/detailed
```

## Категории метрик

### 1. Метрики очередей

```promql
# Общее количество сообщений
rabbitmq_queue_messages

# Сообщения, готовые к доставке
rabbitmq_queue_messages_ready

# Неподтвержденные сообщения
rabbitmq_queue_messages_unacked

# Количество consumers
rabbitmq_queue_consumers

# Память, используемая очередью
rabbitmq_queue_process_memory_bytes

# Сообщения в RAM
rabbitmq_queue_messages_ram

# Сообщения на диске
rabbitmq_queue_messages_persistent
```

**CLI эквивалент:**
```bash
rabbitmqctl list_queues name messages messages_ready messages_unacked consumers memory
```

### 2. Метрики сообщений (rates)

```promql
# Скорость публикации (всего)
rabbitmq_channel_messages_published_total

# Скорость доставки
rabbitmq_channel_messages_delivered_total

# Скорость подтверждений
rabbitmq_channel_messages_acked_total

# Скорость redelivery
rabbitmq_channel_messages_redelivered_total

# Unroutable сообщения
rabbitmq_channel_messages_unroutable_returned_total
rabbitmq_channel_messages_unroutable_dropped_total

# Вычисление rate (Prometheus)
rate(rabbitmq_channel_messages_published_total[5m])
```

### 3. Метрики соединений

```promql
# Общее количество соединений
rabbitmq_connections

# Соединения по состояниям
rabbitmq_connection_state{state="running"}
rabbitmq_connection_state{state="blocked"}

# Каналы на соединение
rabbitmq_connection_channels

# Трафик соединений
rabbitmq_connection_received_bytes_total
rabbitmq_connection_sent_bytes_total
```

**CLI эквивалент:**
```bash
rabbitmqctl list_connections name state channels recv_oct send_oct
```

### 4. Метрики каналов

```promql
# Общее количество каналов
rabbitmq_channels

# Prefetch на канале
rabbitmq_channel_prefetch

# Unconfirmed сообщения (publisher confirms)
rabbitmq_channel_messages_unconfirmed

# Consumer count на канале
rabbitmq_channel_consumers
```

### 5. Метрики ноды

```promql
# Использование памяти
rabbitmq_process_resident_memory_bytes

# Лимит памяти
rabbitmq_resident_memory_limit_bytes

# Процент использования памяти
rabbitmq_process_resident_memory_bytes / rabbitmq_resident_memory_limit_bytes

# File descriptors
rabbitmq_process_open_fds
rabbitmq_process_max_fds

# Sockets
rabbitmq_process_open_sockets
rabbitmq_process_max_sockets

# Disk space
rabbitmq_disk_space_available_bytes
rabbitmq_disk_space_available_limit_bytes

# Erlang processes
rabbitmq_erlang_processes_used
rabbitmq_erlang_processes_limit

# Erlang atoms
rabbitmq_erlang_atoms_used
rabbitmq_erlang_atoms_limit
```

**CLI эквивалент:**
```bash
rabbitmq-diagnostics memory_breakdown --unit mb
rabbitmq-diagnostics file_handle_breakdown
```

### 6. Метрики кластера

```promql
# Количество нод в кластере
rabbitmq_cluster_members

# Partitions
rabbitmq_cluster_partitions

# Running nodes
rabbitmq_cluster_nodes_running

# Raft метрики (quorum queues)
rabbitmq_raft_term_total
rabbitmq_raft_log_snapshot_index
rabbitmq_raft_log_last_applied_index
rabbitmq_raft_log_commit_index
rabbitmq_raft_entry_commit_latency_seconds
```

### 7. Метрики Alarms

```promql
# Memory alarm
rabbitmq_alarms_memory_used_watermark

# Disk alarm
rabbitmq_alarms_free_disk_space_watermark

# File descriptor alarm
rabbitmq_alarms_file_descriptor_limit
```

## Ключевые метрики для мониторинга

### Критические (требуют немедленного внимания)

| Метрика | Threshold | Описание |
|---------|-----------|----------|
| Memory alarm | = 1 | Превышен лимит памяти |
| Disk alarm | = 1 | Мало свободного места |
| Node down | = 0 | Нода недоступна |
| Partitions | > 0 | Network partition |

### Важные (требуют внимания)

| Метрика | Threshold | Описание |
|---------|-----------|----------|
| Queue depth | > 10000 | Накопление сообщений |
| Unacked messages | > 1000 | Медленные consumers |
| No consumers | = 0 при messages > 0 | Очередь без обработчиков |
| FD usage | > 80% | Мало file descriptors |
| Memory usage | > 80% | Высокое потребление памяти |

### Информационные (для анализа)

| Метрика | Использование |
|---------|---------------|
| Message rates | Анализ нагрузки |
| Connection count | Планирование capacity |
| Consumer count | Балансировка нагрузки |

## Настройка алертов

### Prometheus Alerting Rules

```yaml
# rabbitmq_alerts.yml
groups:
  - name: rabbitmq_critical
    rules:
      # Memory Alarm
      - alert: RabbitMQMemoryAlarm
        expr: rabbitmq_alarms_memory_used_watermark == 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "RabbitMQ memory alarm on {{ $labels.instance }}"
          description: "Memory usage exceeded high watermark"
          runbook: "https://wiki/runbooks/rabbitmq-memory-alarm"

      # Disk Alarm
      - alert: RabbitMQDiskAlarm
        expr: rabbitmq_alarms_free_disk_space_watermark == 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "RabbitMQ disk alarm on {{ $labels.instance }}"
          description: "Free disk space below threshold"

      # Node Down
      - alert: RabbitMQNodeDown
        expr: up{job="rabbitmq"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "RabbitMQ node {{ $labels.instance }} is down"

  - name: rabbitmq_warning
    rules:
      # High Queue Depth
      - alert: RabbitMQQueueDepthHigh
        expr: rabbitmq_queue_messages > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High message count in queue {{ $labels.queue }}"
          description: "Queue has {{ $value }} messages"

      # High Unacked Messages
      - alert: RabbitMQUnackedHigh
        expr: rabbitmq_queue_messages_unacked > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High unacked count in queue {{ $labels.queue }}"
          description: "{{ $value }} unacked messages"

      # No Consumers
      - alert: RabbitMQNoConsumers
        expr: >
          rabbitmq_queue_consumers == 0
          and rabbitmq_queue_messages > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Queue {{ $labels.queue }} has no consumers"

      # High Memory Usage
      - alert: RabbitMQMemoryHigh
        expr: >
          rabbitmq_process_resident_memory_bytes
          / rabbitmq_resident_memory_limit_bytes > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value | humanizePercentage }}"

      # High FD Usage
      - alert: RabbitMQFileDescriptorsHigh
        expr: >
          rabbitmq_process_open_fds
          / rabbitmq_process_max_fds > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High file descriptor usage"

      # Message Rate Drop
      - alert: RabbitMQPublishRateDrop
        expr: >
          rate(rabbitmq_channel_messages_published_total[5m])
          < rate(rabbitmq_channel_messages_published_total[1h] offset 1d) * 0.5
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Significant drop in publish rate"
```

## Grafana Dashboards

### Основной Dashboard

```json
{
  "dashboard": {
    "title": "RabbitMQ Overview",
    "panels": [
      {
        "title": "Cluster Health",
        "type": "stat",
        "gridPos": {"x": 0, "y": 0, "w": 4, "h": 4},
        "targets": [{
          "expr": "sum(up{job='rabbitmq'})",
          "legendFormat": "Nodes Up"
        }],
        "options": {
          "colorMode": "value",
          "graphMode": "none"
        }
      },
      {
        "title": "Total Messages",
        "type": "stat",
        "gridPos": {"x": 4, "y": 0, "w": 4, "h": 4},
        "targets": [{
          "expr": "sum(rabbitmq_queue_messages)",
          "legendFormat": "Messages"
        }]
      },
      {
        "title": "Connections",
        "type": "stat",
        "gridPos": {"x": 8, "y": 0, "w": 4, "h": 4},
        "targets": [{
          "expr": "sum(rabbitmq_connections)",
          "legendFormat": "Connections"
        }]
      },
      {
        "title": "Memory Usage",
        "type": "gauge",
        "gridPos": {"x": 12, "y": 0, "w": 4, "h": 4},
        "targets": [{
          "expr": "sum(rabbitmq_process_resident_memory_bytes) / sum(rabbitmq_resident_memory_limit_bytes) * 100"
        }],
        "options": {
          "thresholds": [
            {"color": "green", "value": 0},
            {"color": "yellow", "value": 60},
            {"color": "red", "value": 80}
          ]
        }
      },
      {
        "title": "Message Rates",
        "type": "graph",
        "gridPos": {"x": 0, "y": 4, "w": 12, "h": 8},
        "targets": [
          {
            "expr": "sum(rate(rabbitmq_channel_messages_published_total[5m]))",
            "legendFormat": "Published"
          },
          {
            "expr": "sum(rate(rabbitmq_channel_messages_delivered_total[5m]))",
            "legendFormat": "Delivered"
          },
          {
            "expr": "sum(rate(rabbitmq_channel_messages_acked_total[5m]))",
            "legendFormat": "Acknowledged"
          }
        ]
      },
      {
        "title": "Queue Depths",
        "type": "graph",
        "gridPos": {"x": 0, "y": 12, "w": 12, "h": 8},
        "targets": [{
          "expr": "rabbitmq_queue_messages",
          "legendFormat": "{{ queue }}"
        }]
      },
      {
        "title": "Top 10 Queues by Messages",
        "type": "table",
        "gridPos": {"x": 12, "y": 4, "w": 12, "h": 8},
        "targets": [{
          "expr": "topk(10, rabbitmq_queue_messages)",
          "format": "table",
          "instant": true
        }],
        "transformations": [{
          "type": "organize",
          "config": {
            "includeByName": ["queue", "Value"],
            "renameByName": {
              "queue": "Queue",
              "Value": "Messages"
            }
          }
        }]
      }
    ]
  }
}
```

### Dashboard для Quorum Queues

```json
{
  "dashboard": {
    "title": "RabbitMQ Quorum Queues",
    "panels": [
      {
        "title": "Raft Term",
        "type": "graph",
        "targets": [{
          "expr": "rabbitmq_raft_term_total",
          "legendFormat": "{{ queue }}"
        }]
      },
      {
        "title": "Commit Latency",
        "type": "heatmap",
        "targets": [{
          "expr": "rate(rabbitmq_raft_entry_commit_latency_seconds_bucket[5m])",
          "format": "heatmap"
        }]
      },
      {
        "title": "Log Index Gap",
        "type": "graph",
        "targets": [{
          "expr": "rabbitmq_raft_log_commit_index - rabbitmq_raft_log_last_applied_index",
          "legendFormat": "{{ queue }}"
        }]
      }
    ]
  }
}
```

## Сбор метрик

### Python скрипт

```python
import requests
import time
from prometheus_client import start_http_server, Gauge

# Создаем метрики
queue_messages = Gauge(
    'rabbitmq_queue_messages_custom',
    'Number of messages in queue',
    ['queue', 'vhost']
)

queue_consumers = Gauge(
    'rabbitmq_queue_consumers_custom',
    'Number of consumers',
    ['queue', 'vhost']
)

class RabbitMQMetricsCollector:
    def __init__(self, host, user, password):
        self.url = f"http://{host}:15672/api"
        self.auth = (user, password)

    def collect_queue_metrics(self):
        response = requests.get(f"{self.url}/queues", auth=self.auth)
        queues = response.json()

        for q in queues:
            queue_messages.labels(
                queue=q['name'],
                vhost=q['vhost']
            ).set(q.get('messages', 0))

            queue_consumers.labels(
                queue=q['name'],
                vhost=q['vhost']
            ).set(q.get('consumers', 0))

    def run(self, interval=15):
        while True:
            try:
                self.collect_queue_metrics()
            except Exception as e:
                print(f"Error collecting metrics: {e}")
            time.sleep(interval)


if __name__ == '__main__':
    # Запускаем HTTP сервер для Prometheus
    start_http_server(8000)

    collector = RabbitMQMetricsCollector(
        'localhost',
        'admin',
        'password'
    )
    collector.run()
```

### Telegraf конфигурация

```toml
# telegraf.conf
[[inputs.rabbitmq]]
  url = "http://localhost:15672"
  username = "admin"
  password = "password"

  # Интервал сбора
  interval = "10s"

  # Собираемые объекты
  queues = [".*"]
  exchanges = [".*"]

  # Исключения
  queue_name_exclude = ["amq\\..*"]

  # Теги
  [inputs.rabbitmq.tags]
    environment = "production"
    cluster = "main"

[[outputs.prometheus_client]]
  listen = ":9273"
  metric_version = 2
```

## Анализ метрик

### Capacity Planning

```promql
# Прогноз роста очередей (линейная регрессия)
predict_linear(
  rabbitmq_queue_messages[1h],
  3600
)

# Средняя нагрузка за неделю
avg_over_time(
  rabbitmq_channel_messages_published_total[7d]
)

# Максимальная нагрузка за месяц
max_over_time(
  rabbitmq_connections[30d]
)

# Percentiles времени обработки
histogram_quantile(
  0.99,
  rate(rabbitmq_raft_entry_commit_latency_seconds_bucket[5m])
)
```

### Troubleshooting Queries

```promql
# Очереди с проблемами (messages growing, low consumers)
rabbitmq_queue_messages > 1000
and rabbitmq_queue_consumers < 2

# Соединения с высоким трафиком
topk(10, rate(rabbitmq_connection_received_bytes_total[5m]))

# Каналы с большим prefetch
topk(10, rabbitmq_channel_prefetch)

# Очереди с memory issues
topk(10, rabbitmq_queue_process_memory_bytes)
```

## Best Practices

### Рекомендации по метрикам

1. **Собирайте метрики с интервалом 10-30 секунд** — достаточно для мониторинга, не создает нагрузку

2. **Используйте per-object метрики осторожно** — при большом количестве очередей создают высокую cardinality

3. **Храните метрики долгосрочно** — для анализа трендов и capacity planning

4. **Настройте recording rules** для часто используемых запросов:
```yaml
groups:
  - name: rabbitmq_recording
    rules:
      - record: rabbitmq:messages:total
        expr: sum(rabbitmq_queue_messages)

      - record: rabbitmq:publish_rate:5m
        expr: sum(rate(rabbitmq_channel_messages_published_total[5m]))
```

5. **Используйте агрегацию** для dashboards с большим количеством очередей

### Retention рекомендации

| Granularity | Retention | Использование |
|-------------|-----------|---------------|
| Raw (10s) | 2 недели | Real-time мониторинг |
| 1 min | 1 месяц | Troubleshooting |
| 5 min | 3 месяца | Trend analysis |
| 1 hour | 1 год | Capacity planning |

## Заключение

Эффективный мониторинг метрик RabbitMQ включает:

- **Prometheus** для сбора и хранения метрик
- **Grafana** для визуализации
- **Alerting** для проактивного реагирования
- **Recording rules** для оптимизации запросов
- **Long-term storage** для анализа трендов

Правильно настроенные метрики позволяют обеспечить надежную работу брокера сообщений и быстро реагировать на проблемы.

---

[prev: 04-logging](./04-logging.md) | [next: 01-amqp-091](../12-protocols/01-amqp-091.md)
