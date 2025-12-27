# Мониторинг

## Обзор

Мониторинг RabbitMQ критически важен для обеспечения надежности системы обмена сообщениями. Эффективный мониторинг позволяет:

- Обнаруживать проблемы до их критического влияния
- Планировать масштабирование на основе трендов
- Быстро диагностировать и устранять инциденты
- Оптимизировать производительность

## Встроенные механизмы мониторинга

### Management UI Statistics

Management плагин собирает статистику автоматически:

```ini
# rabbitmq.conf - настройка сбора статистики
collect_statistics = fine  # fine, coarse, или none
collect_statistics_interval = 5000  # миллисекунды
```

### Event Exchange

RabbitMQ публикует события в специальный exchange:

```bash
# Включение event exchange плагина
rabbitmq-plugins enable rabbitmq_event_exchange
```

События публикуются в exchange `amq.rabbitmq.event`:

```python
import pika

connection = pika.BlockingConnection()
channel = connection.channel()

# Создаем очередь для событий
result = channel.queue_declare(queue='', exclusive=True)
queue_name = result.method.queue

# Подписываемся на события очередей
channel.queue_bind(
    exchange='amq.rabbitmq.event',
    queue=queue_name,
    routing_key='queue.*'  # queue.created, queue.deleted, etc.
)

# Типы событий:
# connection.created, connection.closed
# channel.created, channel.closed
# queue.created, queue.deleted
# binding.created, binding.deleted
# consumer.created, consumer.deleted
# vhost.created, vhost.deleted
# user.created, user.deleted, user.authentication.success/failure
```

## Health Checks

### Базовые проверки

```bash
# Проверка работоспособности ноды
rabbitmq-diagnostics check_running

# Проверка alarms (память, диск)
rabbitmq-diagnostics check_local_alarms

# Проверка доступности портов
rabbitmq-diagnostics check_port_connectivity

# Проверка virtual hosts
rabbitmq-diagnostics check_virtual_hosts

# Проверка сертификатов
rabbitmq-diagnostics check_certificate_expiration --within 30days
```

### HTTP Health Check Endpoint

```bash
# Простая проверка через API (возвращает 200 если ok)
curl -f http://localhost:15672/api/healthchecks/node

# Детальная проверка
curl -u guest:guest http://localhost:15672/api/health/checks/virtual-hosts

# Проверка alarms
curl -u guest:guest http://localhost:15672/api/health/checks/alarms

# Проверка порта
curl -u guest:guest http://localhost:15672/api/health/checks/port-listener/5672
```

### Kubernetes Health Probes

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: rabbitmq
          livenessProbe:
            exec:
              command:
                - rabbitmq-diagnostics
                - -q
                - check_running
            initialDelaySeconds: 30
            periodSeconds: 60
            timeoutSeconds: 10
          readinessProbe:
            exec:
              command:
                - rabbitmq-diagnostics
                - -q
                - check_running
                - "&&"
                - rabbitmq-diagnostics
                - -q
                - check_local_alarms
            initialDelaySeconds: 20
            periodSeconds: 10
            timeoutSeconds: 10
```

## Prometheus Integration

### Установка

```bash
# Включение Prometheus плагина
rabbitmq-plugins enable rabbitmq_prometheus
```

Метрики доступны на порту 15692:
- `http://localhost:15692/metrics` — метрики RabbitMQ
- `http://localhost:15692/metrics/per-object` — детальные метрики по объектам

### Конфигурация Prometheus

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'rabbitmq'
    static_configs:
      - targets: ['rabbitmq-node1:15692', 'rabbitmq-node2:15692']
    scrape_interval: 15s
    scrape_timeout: 10s
    metrics_path: /metrics
```

### Ключевые метрики Prometheus

```promql
# Общее количество сообщений в очередях
sum(rabbitmq_queue_messages)

# Количество готовых к доставке сообщений
sum(rabbitmq_queue_messages_ready)

# Количество unacknowledged сообщений
sum(rabbitmq_queue_messages_unacked)

# Скорость публикации (сообщений в секунду)
sum(rate(rabbitmq_channel_messages_published_total[5m]))

# Скорость доставки
sum(rate(rabbitmq_channel_messages_delivered_total[5m]))

# Скорость подтверждений
sum(rate(rabbitmq_channel_messages_acked_total[5m]))

# Количество соединений
rabbitmq_connections

# Количество каналов
rabbitmq_channels

# Использование памяти
rabbitmq_process_resident_memory_bytes

# Использование дискового пространства
rabbitmq_disk_space_available_bytes

# File descriptors
rabbitmq_process_open_fds / rabbitmq_process_max_fds
```

### Prometheus Alerting Rules

```yaml
# rabbitmq_alerts.yml
groups:
  - name: rabbitmq
    rules:
      # Алерт на высокое количество сообщений в очереди
      - alert: RabbitMQQueueMessagesHigh
        expr: rabbitmq_queue_messages > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High queue depth on {{ $labels.queue }}"
          description: "Queue {{ $labels.queue }} has {{ $value }} messages"

      # Алерт на Memory Alarm
      - alert: RabbitMQMemoryAlarm
        expr: rabbitmq_alarms_memory_used_watermark == 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "RabbitMQ memory alarm triggered"
          description: "Memory usage exceeded high watermark"

      # Алерт на Disk Alarm
      - alert: RabbitMQDiskAlarm
        expr: rabbitmq_alarms_free_disk_space_watermark == 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "RabbitMQ disk alarm triggered"
          description: "Free disk space below threshold"

      # Алерт на отсутствие consumers
      - alert: RabbitMQNoConsumers
        expr: rabbitmq_queue_consumers == 0 and rabbitmq_queue_messages > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Queue {{ $labels.queue }} has no consumers"
          description: "Queue has messages but no active consumers"

      # Алерт на рост unacked сообщений
      - alert: RabbitMQUnackedMessagesHigh
        expr: rabbitmq_queue_messages_unacked > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High unacked messages on {{ $labels.queue }}"

      # Алерт на недоступность ноды
      - alert: RabbitMQNodeDown
        expr: up{job="rabbitmq"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "RabbitMQ node is down"

      # Алерт на высокое использование FD
      - alert: RabbitMQFileDescriptorsHigh
        expr: rabbitmq_process_open_fds / rabbitmq_process_max_fds > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High file descriptor usage"
```

## Grafana Dashboards

### Официальный Dashboard

```bash
# Импорт официального dashboard
# Dashboard ID: 10991 (RabbitMQ Overview)
# Dashboard ID: 11340 (RabbitMQ Cluster Overview)
```

### Пример Custom Dashboard

```json
{
  "dashboard": {
    "title": "RabbitMQ Monitoring",
    "panels": [
      {
        "title": "Total Messages",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(rabbitmq_queue_messages)",
            "legendFormat": "Messages"
          }
        ]
      },
      {
        "title": "Message Rates",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(rabbitmq_channel_messages_published_total[5m]))",
            "legendFormat": "Published"
          },
          {
            "expr": "sum(rate(rabbitmq_channel_messages_delivered_total[5m]))",
            "legendFormat": "Delivered"
          }
        ]
      },
      {
        "title": "Queue Depth",
        "type": "graph",
        "targets": [
          {
            "expr": "rabbitmq_queue_messages",
            "legendFormat": "{{ queue }}"
          }
        ]
      },
      {
        "title": "Memory Usage",
        "type": "gauge",
        "targets": [
          {
            "expr": "rabbitmq_process_resident_memory_bytes / 1024 / 1024 / 1024",
            "legendFormat": "Memory (GB)"
          }
        ]
      }
    ]
  }
}
```

## Мониторинг через API

### Python скрипт мониторинга

```python
import requests
from datetime import datetime

class RabbitMQMonitor:
    def __init__(self, host, user, password):
        self.base_url = f"http://{host}:15672/api"
        self.auth = (user, password)

    def get_overview(self):
        """Получить общую статистику"""
        response = requests.get(f"{self.base_url}/overview", auth=self.auth)
        return response.json()

    def get_queues(self, vhost='/'):
        """Получить список очередей"""
        vhost_encoded = vhost.replace('/', '%2F')
        response = requests.get(
            f"{self.base_url}/queues/{vhost_encoded}",
            auth=self.auth
        )
        return response.json()

    def get_connections(self):
        """Получить список соединений"""
        response = requests.get(f"{self.base_url}/connections", auth=self.auth)
        return response.json()

    def check_queue_depth(self, threshold=10000):
        """Проверить глубину очередей"""
        alerts = []
        for queue in self.get_queues():
            if queue['messages'] > threshold:
                alerts.append({
                    'queue': queue['name'],
                    'messages': queue['messages'],
                    'consumers': queue['consumers']
                })
        return alerts

    def check_memory_alarm(self):
        """Проверить memory alarm"""
        overview = self.get_overview()
        for node in overview.get('node_stats', []):
            if node.get('mem_alarm'):
                return True
        return False

    def check_disk_alarm(self):
        """Проверить disk alarm"""
        overview = self.get_overview()
        for node in overview.get('node_stats', []):
            if node.get('disk_free_alarm'):
                return True
        return False

    def get_message_rates(self):
        """Получить скорости обработки сообщений"""
        overview = self.get_overview()
        rates = overview.get('message_stats', {})
        return {
            'publish_rate': rates.get('publish_details', {}).get('rate', 0),
            'deliver_rate': rates.get('deliver_details', {}).get('rate', 0),
            'ack_rate': rates.get('ack_details', {}).get('rate', 0)
        }


# Использование
if __name__ == '__main__':
    monitor = RabbitMQMonitor('localhost', 'admin', 'password')

    # Проверка очередей
    alerts = monitor.check_queue_depth(5000)
    for alert in alerts:
        print(f"WARNING: Queue {alert['queue']} has {alert['messages']} messages")

    # Проверка alarms
    if monitor.check_memory_alarm():
        print("CRITICAL: Memory alarm triggered!")

    if monitor.check_disk_alarm():
        print("CRITICAL: Disk alarm triggered!")

    # Вывод rates
    rates = monitor.get_message_rates()
    print(f"Publish rate: {rates['publish_rate']:.2f} msg/s")
    print(f"Deliver rate: {rates['deliver_rate']:.2f} msg/s")
```

## Telegraf Integration

```toml
# telegraf.conf
[[inputs.rabbitmq]]
  url = "http://localhost:15672"
  username = "admin"
  password = "password"

  # Собираемые метрики
  queues = [".*"]  # regex для очередей
  exchanges = [".*"]

  # Теги
  [inputs.rabbitmq.tags]
    environment = "production"

[[outputs.influxdb_v2]]
  urls = ["http://influxdb:8086"]
  token = "your-token"
  organization = "your-org"
  bucket = "rabbitmq"
```

## Мониторинг Quorum Queues

```bash
# Статус quorum queue
rabbitmq-diagnostics quorum_status queue_name

# Проверка leader
rabbitmqctl list_queues name type leader members online

# Мониторинг Raft
rabbitmq-diagnostics raft_overview
```

```promql
# Prometheus метрики для quorum queues
rabbitmq_raft_term_total
rabbitmq_raft_log_snapshot_index
rabbitmq_raft_log_last_applied_index
rabbitmq_raft_log_commit_index
```

## Best Practices мониторинга

### Ключевые метрики для алертов

| Метрика | Threshold | Severity |
|---------|-----------|----------|
| Memory alarm | triggered | Critical |
| Disk alarm | triggered | Critical |
| Node down | any node | Critical |
| Queue depth | > 10000 | Warning |
| Unacked messages | > 1000 | Warning |
| No consumers | > 5 min | Warning |
| FD usage | > 80% | Warning |
| Connection count | > 80% of limit | Warning |

### Рекомендации

1. **Мониторьте тренды, а не только пороги** — рост очередей может указывать на проблемы до срабатывания алертов

2. **Настройте алерты на anomalies** — резкое изменение message rate может указывать на проблемы

3. **Используйте per-queue мониторинг** для критичных очередей

4. **Храните метрики долгосрочно** для анализа трендов и capacity planning

5. **Мониторьте клиентские метрики** в дополнение к серверным

6. **Настройте escalation** — разные уровни severity для разных проблем

7. **Документируйте runbooks** для каждого типа алерта

## Заключение

Эффективный мониторинг RabbitMQ включает:

- **Health checks** для проверки работоспособности
- **Prometheus metrics** для детального мониторинга
- **Grafana dashboards** для визуализации
- **Alerting rules** для оповещения о проблемах
- **Custom скрипты** для специфических проверок

Правильно настроенный мониторинг позволяет предотвращать инциденты и быстро реагировать на проблемы.
