# Метрики Kafka

## Описание

Метрики Apache Kafka предоставляют детальную информацию о состоянии и производительности кластера. Kafka экспортирует метрики через JMX (Java Management Extensions), которые можно собирать с помощью различных инструментов мониторинга: Prometheus, Datadog, Grafana и других.

Правильное отслеживание метрик позволяет выявлять проблемы до их критического влияния на систему, планировать масштабирование и оптимизировать производительность.

## Ключевые концепции

### Категории метрик

| Категория | Описание | Важность |
|-----------|----------|----------|
| Broker metrics | Состояние и производительность брокера | Критическая |
| Topic metrics | Метрики по топикам | Высокая |
| Partition metrics | Метрики партиций и репликации | Критическая |
| Producer metrics | Метрики продюсеров | Средняя |
| Consumer metrics | Метрики консьюмеров | Высокая |
| Network metrics | Сетевые метрики | Высокая |
| JVM metrics | Метрики Java VM | Средняя |

### Ключевые метрики для мониторинга

| Метрика | Описание | Порог для алерта |
|---------|----------|------------------|
| UnderReplicatedPartitions | Партиции с неполной репликацией | > 0 |
| ActiveControllerCount | Количество активных контроллеров | != 1 |
| OfflinePartitionsCount | Оффлайн партиции | > 0 |
| RequestsPerSec | Количество запросов в секунду | Зависит от нагрузки |
| BytesInPerSec | Входящий трафик | 80% от пропускной способности |
| BytesOutPerSec | Исходящий трафик | 80% от пропускной способности |
| ConsumerLag | Отставание консьюмеров | > настраиваемый порог |

## Примеры

### Настройка JMX для экспорта метрик

```bash
# Переменные окружения для JMX
export KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote \
  -Dcom.sun.management.jmxremote.port=9999 \
  -Dcom.sun.management.jmxremote.rmi.port=9999 \
  -Dcom.sun.management.jmxremote.authenticate=false \
  -Dcom.sun.management.jmxremote.ssl=false \
  -Djava.rmi.server.hostname=localhost"

# Для production с аутентификацией
export KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote \
  -Dcom.sun.management.jmxremote.port=9999 \
  -Dcom.sun.management.jmxremote.rmi.port=9999 \
  -Dcom.sun.management.jmxremote.authenticate=true \
  -Dcom.sun.management.jmxremote.ssl=true \
  -Dcom.sun.management.jmxremote.password.file=/opt/kafka/config/jmxremote.password \
  -Dcom.sun.management.jmxremote.access.file=/opt/kafka/config/jmxremote.access \
  -Djava.rmi.server.hostname=broker1.example.com"
```

### Основные метрики брокера

```
# Метрики здоровья кластера
kafka.controller:type=KafkaController,name=ActiveControllerCount
kafka.controller:type=KafkaController,name=OfflinePartitionsCount

# Метрики репликации
kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
kafka.server:type=ReplicaManager,name=UnderMinIsrPartitionCount
kafka.server:type=ReplicaManager,name=PartitionCount
kafka.server:type=ReplicaManager,name=LeaderCount

# Метрики запросов
kafka.network:type=RequestMetrics,name=RequestsPerSec,request={Produce|FetchConsumer|FetchFollower}
kafka.network:type=RequestMetrics,name=TotalTimeMs,request={Produce|FetchConsumer|FetchFollower}
kafka.network:type=RequestMetrics,name=RequestQueueTimeMs,request={Produce|FetchConsumer|FetchFollower}
kafka.network:type=RequestMetrics,name=LocalTimeMs,request={Produce|FetchConsumer|FetchFollower}
kafka.network:type=RequestMetrics,name=RemoteTimeMs,request={Produce|FetchConsumer|FetchFollower}
kafka.network:type=RequestMetrics,name=ResponseQueueTimeMs,request={Produce|FetchConsumer|FetchFollower}
kafka.network:type=RequestMetrics,name=ResponseSendTimeMs,request={Produce|FetchConsumer|FetchFollower}

# Метрики пропускной способности
kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec
kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec
kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec
kafka.server:type=BrokerTopicMetrics,name=TotalProduceRequestsPerSec
kafka.server:type=BrokerTopicMetrics,name=TotalFetchRequestsPerSec
kafka.server:type=BrokerTopicMetrics,name=FailedProduceRequestsPerSec
kafka.server:type=BrokerTopicMetrics,name=FailedFetchRequestsPerSec

# Метрики логов
kafka.log:type=LogFlushStats,name=LogFlushRateAndTimeMs
kafka.log:type=Log,name=Size,topic=*,partition=*
kafka.log:type=Log,name=NumLogSegments,topic=*,partition=*

# Метрики сети
kafka.network:type=SocketServer,name=NetworkProcessorAvgIdlePercent
kafka.server:type=KafkaRequestHandlerPool,name=RequestHandlerAvgIdlePercent
```

### Prometheus JMX Exporter конфигурация

```yaml
# prometheus-jmx-config.yml

lowercaseOutputName: true
lowercaseOutputLabelNames: true
whitelistObjectNames:
  - "kafka.controller:*"
  - "kafka.server:*"
  - "kafka.network:*"
  - "kafka.log:*"
  - "kafka.cluster:*"

rules:
  # Метрики контроллера
  - pattern: kafka.controller<type=KafkaController, name=(.+)><>Value
    name: kafka_controller_$1
    type: GAUGE

  - pattern: kafka.controller<type=ControllerStats, name=(.+)><>Count
    name: kafka_controller_stats_$1_total
    type: COUNTER

  # Метрики брокера
  - pattern: kafka.server<type=ReplicaManager, name=(.+)><>Value
    name: kafka_server_replica_manager_$1
    type: GAUGE

  - pattern: kafka.server<type=BrokerTopicMetrics, name=(.+)><>Count
    name: kafka_server_broker_topic_metrics_$1_total
    type: COUNTER

  - pattern: kafka.server<type=BrokerTopicMetrics, name=(.+), topic=(.+)><>Count
    name: kafka_server_broker_topic_metrics_$1_total
    type: COUNTER
    labels:
      topic: "$2"

  - pattern: kafka.server<type=BrokerTopicMetrics, name=(.+)><>OneMinuteRate
    name: kafka_server_broker_topic_metrics_$1_rate
    type: GAUGE

  - pattern: kafka.server<type=BrokerTopicMetrics, name=(.+), topic=(.+)><>OneMinuteRate
    name: kafka_server_broker_topic_metrics_$1_rate
    type: GAUGE
    labels:
      topic: "$2"

  # Метрики запросов
  - pattern: kafka.network<type=RequestMetrics, name=(.+), request=(.+), error=(.+)><>Count
    name: kafka_network_request_metrics_$1_total
    type: COUNTER
    labels:
      request: "$2"
      error: "$3"

  - pattern: kafka.network<type=RequestMetrics, name=(.+), request=(.+)><>Count
    name: kafka_network_request_metrics_$1_total
    type: COUNTER
    labels:
      request: "$2"

  - pattern: kafka.network<type=RequestMetrics, name=(.+), request=(.+)><>(\d+)thPercentile
    name: kafka_network_request_metrics_$1_percentile
    type: GAUGE
    labels:
      request: "$2"
      percentile: "$3"

  # Метрики партиций
  - pattern: kafka.cluster<type=Partition, name=(.+), topic=(.+), partition=(.+)><>Value
    name: kafka_cluster_partition_$1
    type: GAUGE
    labels:
      topic: "$2"
      partition: "$3"

  # Метрики логов
  - pattern: kafka.log<type=Log, name=(.+), topic=(.+), partition=(.+)><>Value
    name: kafka_log_$1
    type: GAUGE
    labels:
      topic: "$2"
      partition: "$3"

  - pattern: kafka.log<type=LogFlushStats, name=(.+)><>Count
    name: kafka_log_flush_$1_total
    type: COUNTER

  # Сетевые метрики
  - pattern: kafka.network<type=SocketServer, name=(.+)><>Value
    name: kafka_network_socket_server_$1
    type: GAUGE

  - pattern: kafka.server<type=KafkaRequestHandlerPool, name=(.+)><>Value
    name: kafka_server_request_handler_$1
    type: GAUGE

  # Consumer group метрики
  - pattern: kafka.server<type=FetcherLagMetrics, name=ConsumerLag, clientId=(.+), topic=(.+), partition=(.+)><>Value
    name: kafka_server_consumer_lag
    type: GAUGE
    labels:
      client_id: "$1"
      topic: "$2"
      partition: "$3"
```

### Запуск JMX Exporter

```bash
# Добавить в KAFKA_OPTS
KAFKA_OPTS="-javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent.jar=7071:/opt/jmx_exporter/kafka.yml"
```

### Prometheus конфигурация для сбора метрик

```yaml
# prometheus.yml

global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'kafka'
    static_configs:
      - targets:
        - 'kafka-broker-1:7071'
        - 'kafka-broker-2:7071'
        - 'kafka-broker-3:7071'
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '([^:]+):\d+'
        replacement: '$1'

  - job_name: 'kafka-exporter'
    static_configs:
      - targets: ['kafka-exporter:9308']

  - job_name: 'zookeeper'
    static_configs:
      - targets:
        - 'zookeeper-1:7072'
        - 'zookeeper-2:7072'
        - 'zookeeper-3:7072'
```

### Alertmanager правила

```yaml
# kafka-alerts.yml

groups:
  - name: kafka-alerts
    rules:
      # Критические алерты
      - alert: KafkaOfflinePartitions
        expr: kafka_controller_offline_partitions_count > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Kafka has offline partitions"
          description: "{{ $value }} partitions are offline on {{ $labels.instance }}"

      - alert: KafkaUnderReplicatedPartitions
        expr: kafka_server_replica_manager_under_replicated_partitions > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Kafka has under-replicated partitions"
          description: "{{ $value }} partitions are under-replicated on {{ $labels.instance }}"

      - alert: KafkaNoActiveController
        expr: sum(kafka_controller_active_controller_count) != 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Kafka cluster has no active controller"
          description: "The Kafka cluster has {{ $value }} active controllers, expected 1"

      # Производительность
      - alert: KafkaHighRequestLatency
        expr: kafka_network_request_metrics_total_time_ms{quantile="0.99"} > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High Kafka request latency"
          description: "99th percentile latency is {{ $value }}ms for {{ $labels.request }}"

      - alert: KafkaHighProduceLatency
        expr: kafka_network_request_metrics_total_time_ms{request="Produce", quantile="0.99"} > 500
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High Kafka produce latency"
          description: "99th percentile produce latency is {{ $value }}ms"

      # Consumer lag
      - alert: KafkaConsumerLag
        expr: kafka_consumergroup_lag > 10000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High consumer lag detected"
          description: "Consumer group {{ $labels.consumergroup }} has lag {{ $value }} on topic {{ $labels.topic }}"

      - alert: KafkaConsumerLagCritical
        expr: kafka_consumergroup_lag > 100000
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Critical consumer lag detected"
          description: "Consumer group {{ $labels.consumergroup }} has critical lag {{ $value }}"

      # Ресурсы
      - alert: KafkaNetworkProcessorIdleLow
        expr: kafka_network_socket_server_network_processor_avg_idle_percent < 0.3
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Kafka network processor idle is low"
          description: "Network processor idle is {{ $value | humanizePercentage }} on {{ $labels.instance }}"

      - alert: KafkaRequestHandlerIdleLow
        expr: kafka_server_request_handler_avg_idle_percent < 0.3
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Kafka request handler idle is low"
          description: "Request handler idle is {{ $value | humanizePercentage }} on {{ $labels.instance }}"

      # Диск
      - alert: KafkaLogDirDiskUsageHigh
        expr: (1 - (node_filesystem_avail_bytes{mountpoint="/var/lib/kafka"} / node_filesystem_size_bytes{mountpoint="/var/lib/kafka"})) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Kafka log directory disk usage is high"
          description: "Disk usage is {{ $value | humanizePercentage }} on {{ $labels.instance }}"

      # JVM
      - alert: KafkaJvmMemoryHigh
        expr: (jvm_memory_bytes_used{area="heap"} / jvm_memory_bytes_max{area="heap"}) > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Kafka JVM heap usage is high"
          description: "JVM heap usage is {{ $value | humanizePercentage }} on {{ $labels.instance }}"
```

### Grafana Dashboard запросы

```yaml
# Примеры PromQL запросов для Grafana

# Messages in/out per second
rate(kafka_server_broker_topic_metrics_messages_in_total[5m])
rate(kafka_server_broker_topic_metrics_bytes_out_total[5m])

# Request rate by type
rate(kafka_network_request_metrics_requests_total[5m])

# Request latency percentiles
histogram_quantile(0.99, rate(kafka_network_request_metrics_total_time_ms_bucket[5m]))

# Under-replicated partitions
sum(kafka_server_replica_manager_under_replicated_partitions)

# Active controller
sum(kafka_controller_active_controller_count)

# Consumer lag
sum by (consumergroup, topic) (kafka_consumergroup_lag)

# Partition count per broker
sum by (instance) (kafka_server_replica_manager_partition_count)

# Leader count per broker
sum by (instance) (kafka_server_replica_manager_leader_count)

# Network/Request handler utilization
1 - kafka_network_socket_server_network_processor_avg_idle_percent
1 - kafka_server_request_handler_avg_idle_percent

# ISR shrinks/expands
rate(kafka_server_replica_manager_isr_shrinks_total[5m])
rate(kafka_server_replica_manager_isr_expands_total[5m])
```

### Скрипт для получения метрик через JMX

```bash
#!/bin/bash
# get-kafka-metrics.sh

KAFKA_JMX_HOST=${1:-localhost}
KAFKA_JMX_PORT=${2:-9999}

# Использование jmxterm
echo "beans" | java -jar jmxterm.jar -l $KAFKA_JMX_HOST:$KAFKA_JMX_PORT -n

# Получение конкретной метрики
echo "get -b kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec Count" | \
    java -jar jmxterm.jar -l $KAFKA_JMX_HOST:$KAFKA_JMX_PORT -n

# Получение всех метрик репликации
echo "get -d kafka.server -b type=ReplicaManager,name=* Value" | \
    java -jar jmxterm.jar -l $KAFKA_JMX_HOST:$KAFKA_JMX_PORT -n
```

### Python скрипт для сбора метрик

```python
#!/usr/bin/env python3
# kafka_metrics_collector.py

from kafka import KafkaAdminClient
from kafka.admin import ConfigResource, ConfigResourceType
import json

def get_cluster_metrics(bootstrap_servers):
    admin = KafkaAdminClient(bootstrap_servers=bootstrap_servers)

    # Получение информации о кластере
    cluster_metadata = admin.describe_cluster()
    print(f"Cluster ID: {cluster_metadata['cluster_id']}")
    print(f"Controller: {cluster_metadata['controller_id']}")
    print(f"Brokers: {len(cluster_metadata['brokers'])}")

    # Получение информации о топиках
    topics = admin.list_topics()
    topic_descriptions = admin.describe_topics(topics)

    metrics = {
        'cluster_id': cluster_metadata['cluster_id'],
        'controller_id': cluster_metadata['controller_id'],
        'broker_count': len(cluster_metadata['brokers']),
        'topic_count': len(topics),
        'topics': {}
    }

    for topic in topic_descriptions:
        topic_name = topic['topic']
        partitions = topic['partitions']

        partition_count = len(partitions)
        replica_count = sum(len(p['replicas']) for p in partitions)
        isr_count = sum(len(p['isr']) for p in partitions)

        under_replicated = sum(
            1 for p in partitions
            if len(p['isr']) < len(p['replicas'])
        )

        metrics['topics'][topic_name] = {
            'partitions': partition_count,
            'replicas': replica_count,
            'isr': isr_count,
            'under_replicated': under_replicated
        }

    admin.close()
    return metrics

if __name__ == '__main__':
    metrics = get_cluster_metrics(['localhost:9092'])
    print(json.dumps(metrics, indent=2))
```

## Best Practices

### Ключевые метрики для SLA

1. **Доступность**: ActiveControllerCount, OfflinePartitionsCount
2. **Надёжность**: UnderReplicatedPartitions, ISRShrinkRate
3. **Производительность**: RequestLatency, MessagesPerSec
4. **Потребление**: ConsumerLag, BytesInPerSec, BytesOutPerSec

### Рекомендации по настройке алертов

```yaml
# Приоритеты алертов
Critical (P1):
  - OfflinePartitionsCount > 0
  - ActiveControllerCount != 1
  - UnderReplicatedPartitions > 0 (длительное время)
  - Disk usage > 95%

Warning (P2):
  - UnderReplicatedPartitions > 0 (кратковременно)
  - High request latency
  - Consumer lag growing
  - Network processor idle < 30%

Info (P3):
  - ISR shrinks/expands
  - Leader elections
  - Log flush latency
```

### Базовые пороги

| Метрика | Warning | Critical |
|---------|---------|----------|
| UnderReplicatedPartitions | > 0 (5 мин) | > 0 (15 мин) |
| ConsumerLag | > 10,000 | > 100,000 |
| RequestLatency (p99) | > 500ms | > 1000ms |
| Disk usage | > 75% | > 90% |
| Network processor idle | < 40% | < 20% |
| JVM heap usage | > 80% | > 90% |

### Retention метрик

```yaml
# Рекомендуемые периоды хранения
- Raw metrics: 2 недели
- 5-минутные агрегаты: 3 месяца
- Часовые агрегаты: 1 год
- Дневные агрегаты: 3 года
```
