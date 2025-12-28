# Окружения с несколькими кластерами (Multi-Cluster)

[prev: 05-architectures](./05-architectures.md) | [next: 07-cloud-container-storage](./07-cloud-container-storage.md)

---

## Описание

Multi-cluster архитектура Kafka позволяет организовать распределённую инфраструктуру для обеспечения высокой доступности, disaster recovery, географического распределения данных и изоляции рабочих нагрузок. В этом разделе рассматриваются паттерны и инструменты для работы с несколькими кластерами Kafka.

## Ключевые концепции

### Причины использования мульти-кластера

1. **Disaster Recovery (DR)** — резервный кластер в другом регионе/ДЦ
2. **Geographic Distribution** — данные ближе к пользователям
3. **Workload Isolation** — разделение prod/staging/dev
4. **Regulatory Compliance** — данные в определённой юрисдикции
5. **Migration** — переход на новую версию Kafka
6. **Aggregation** — сбор данных из множества источников

### Топологии мульти-кластера

#### Active-Passive (DR)

```
┌─────────────────────────────────────────────────────────────┐
│                       Region: US-East                        │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                  Primary Cluster                       │  │
│  │  ┌──────┐ ┌──────┐ ┌──────┐                           │  │
│  │  │Broker│ │Broker│ │Broker│  ← Producers/Consumers    │  │
│  │  │  1   │ │  2   │ │  3   │                           │  │
│  │  └──────┘ └──────┘ └──────┘                           │  │
│  └───────────────────────────┬───────────────────────────┘  │
│                              │                               │
└──────────────────────────────┼───────────────────────────────┘
                               │ MirrorMaker 2
                               │ (Async Replication)
                               ▼
┌──────────────────────────────────────────────────────────────┐
│                       Region: US-West                         │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                  Standby Cluster (DR)                   │  │
│  │  ┌──────┐ ┌──────┐ ┌──────┐                            │  │
│  │  │Broker│ │Broker│ │Broker│  (Read-only, failover)     │  │
│  │  │  1   │ │  2   │ │  3   │                            │  │
│  │  └──────┘ └──────┘ └──────┘                            │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

#### Active-Active (Geo-Distribution)

```
┌─────────────────────────────────────────────────────────────┐
│                       Region: Europe                         │
│  ┌───────────────────────────────────────────────────────┐  │
│  │               Cluster EU                               │  │
│  │  ┌──────┐ ┌──────┐ ┌──────┐                           │  │
│  │  │Broker│ │Broker│ │Broker│  ← EU Producers           │  │
│  │  │  1   │ │  2   │ │  3   │  ← EU Consumers           │  │
│  │  └──────┘ └──────┘ └──────┘                           │  │
│  └───────────────────────────────────────────────────────┘  │
└──────────────────────────────┬───────────────────────────────┘
                               │
                    MirrorMaker 2 (Bidirectional)
                               │
┌──────────────────────────────┴───────────────────────────────┐
│                       Region: US                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │               Cluster US                                │  │
│  │  ┌──────┐ ┌──────┐ ┌──────┐                            │  │
│  │  │Broker│ │Broker│ │Broker│  ← US Producers            │  │
│  │  │  1   │ │  2   │ │  3   │  ← US Consumers            │  │
│  │  └──────┘ └──────┘ └──────┘                            │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

#### Hub-and-Spoke (Aggregation)

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Cluster    │     │  Cluster    │     │  Cluster    │
│  Region A   │     │  Region B   │     │  Region C   │
│  (Spoke)    │     │  (Spoke)    │     │  (Spoke)    │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       │   MirrorMaker 2   │   MirrorMaker 2   │
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                           ▼
                ┌──────────────────────┐
                │    Central Cluster    │
                │        (Hub)          │
                │                       │
                │  • Analytics          │
                │  • ML Training        │
                │  • Global Reports     │
                └──────────────────────┘
```

## MirrorMaker 2 (MM2)

### Архитектура MM2

MirrorMaker 2 построен на Kafka Connect:

```
┌───────────────────────────────────────────────────────────────┐
│                    MirrorMaker 2                               │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌───────────────┐ │
│  │ MirrorSource    │  │ MirrorCheckpoint│  │ MirrorHeartbeat│ │
│  │ Connector       │  │ Connector       │  │ Connector     │ │
│  │                 │  │                 │  │               │ │
│  │ • Copy topics   │  │ • Sync consumer │  │ • Monitoring  │ │
│  │ • Preserve      │  │   offsets       │  │ • Latency     │ │
│  │   offsets       │  │                 │  │   tracking    │ │
│  └─────────────────┘  └─────────────────┘  └───────────────┘ │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### Базовая конфигурация MM2

```properties
# mm2.properties

# Определение кластеров
clusters = primary, secondary

# Bootstrap серверы
primary.bootstrap.servers = kafka-primary-1:9092,kafka-primary-2:9092,kafka-primary-3:9092
secondary.bootstrap.servers = kafka-secondary-1:9092,kafka-secondary-2:9092,kafka-secondary-3:9092

# Направление репликации: primary → secondary
primary->secondary.enabled = true
primary->secondary.topics = .*
primary->secondary.topics.exclude = __.*

# Обратная репликация (для active-active)
secondary->primary.enabled = false

# Consumer offset синхронизация
sync.group.offsets.enabled = true
sync.group.offsets.interval.seconds = 5

# Heartbeat
emit.heartbeats.enabled = true
emit.heartbeats.interval.seconds = 5

# Checkpoints для failover
emit.checkpoints.enabled = true
emit.checkpoints.interval.seconds = 5

# Replication policy
replication.policy.class = org.apache.kafka.connect.mirror.DefaultReplicationPolicy
# Или IdentityReplicationPolicy для сохранения оригинальных имён топиков
```

### Запуск MM2

```bash
# Standalone режим
connect-mirror-maker.sh mm2.properties

# Distributed режим (рекомендуется для продакшена)
connect-distributed.sh connect-mm2-distributed.properties
```

### Distributed конфигурация

```properties
# connect-mm2-distributed.properties

bootstrap.servers = kafka-primary-1:9092,kafka-primary-2:9092

group.id = mm2-cluster

key.converter = org.apache.kafka.connect.converters.ByteArrayConverter
value.converter = org.apache.kafka.connect.converters.ByteArrayConverter

offset.storage.topic = mm2-offsets
offset.storage.replication.factor = 3

config.storage.topic = mm2-configs
config.storage.replication.factor = 3

status.storage.topic = mm2-status
status.storage.replication.factor = 3
```

### Политики репликации

#### DefaultReplicationPolicy

Добавляет prefix с именем source кластера:

```
Source: primary  Topic: events
Target: secondary  Topic: primary.events
```

#### IdentityReplicationPolicy

Сохраняет оригинальные имена топиков:

```
Source: primary  Topic: events
Target: secondary  Topic: events
```

```properties
replication.policy.class = org.apache.kafka.connect.mirror.IdentityReplicationPolicy
```

**Важно:** IdentityReplicationPolicy требует осторожности при bidirectional репликации для избежания циклов.

## Примеры конфигурации

### Active-Passive DR Setup

```properties
# mm2-dr.properties

clusters = prod, dr

prod.bootstrap.servers = prod-kafka-1:9092,prod-kafka-2:9092,prod-kafka-3:9092
dr.bootstrap.servers = dr-kafka-1:9092,dr-kafka-2:9092,dr-kafka-3:9092

# Security (если используется)
prod.security.protocol = SASL_SSL
prod.sasl.mechanism = PLAIN
prod.sasl.jaas.config = org.apache.kafka.common.security.plain.PlainLoginModule required \
    username="mm2" \
    password="secret";

dr.security.protocol = SASL_SSL
dr.sasl.mechanism = PLAIN
dr.sasl.jaas.config = org.apache.kafka.common.security.plain.PlainLoginModule required \
    username="mm2" \
    password="secret";

# Репликация prod → dr
prod->dr.enabled = true
prod->dr.topics = .*
prod->dr.topics.exclude = __.*, .*\.internal

# Настройки producer для DR
prod->dr.producer.acks = all
prod->dr.producer.retries = 2147483647
prod->dr.producer.max.in.flight.requests.per.connection = 1

# Consumer offset sync
sync.group.offsets.enabled = true
sync.group.offsets.interval.seconds = 10
refresh.groups.interval.seconds = 30

# Checkpoints для failover
emit.checkpoints.enabled = true
emit.checkpoints.interval.seconds = 10
```

### Active-Active с IdentityReplicationPolicy

```properties
# mm2-active-active.properties

clusters = us, eu

us.bootstrap.servers = us-kafka-1:9092,us-kafka-2:9092
eu.bootstrap.servers = eu-kafka-1:9092,eu-kafka-2:9092

# Bidirectional
us->eu.enabled = true
eu->us.enabled = true

# Exclude internal и replicated topics
us->eu.topics = (?!eu\.).*
eu->us.topics = (?!us\.).*

# Сохранение оригинальных имён с prefix
replication.policy.class = org.apache.kafka.connect.mirror.DefaultReplicationPolicy
replication.policy.separator = .

# Offset translation
sync.group.offsets.enabled = true
emit.checkpoints.enabled = true
```

### Aggregation Hub Setup

```properties
# mm2-aggregation.properties

clusters = region-a, region-b, region-c, hub

region-a.bootstrap.servers = region-a-kafka:9092
region-b.bootstrap.servers = region-b-kafka:9092
region-c.bootstrap.servers = region-c-kafka:9092
hub.bootstrap.servers = hub-kafka-1:9092,hub-kafka-2:9092,hub-kafka-3:9092

# All regions → hub
region-a->hub.enabled = true
region-b->hub.enabled = true
region-c->hub.enabled = true

# Topics to aggregate
region-a->hub.topics = events, metrics, logs
region-b->hub.topics = events, metrics, logs
region-c->hub.topics = events, metrics, logs

# No reverse replication
hub->region-a.enabled = false
hub->region-b.enabled = false
hub->region-c.enabled = false
```

## Failover процедуры

### Автоматический Failover

```bash
#!/bin/bash
# failover-to-dr.sh

PRIMARY="prod-kafka:9092"
DR="dr-kafka:9092"
CONSUMER_GROUP="my-app"

echo "Starting failover to DR cluster..."

# 1. Проверка доступности DR кластера
if ! kafka-broker-api-versions.sh --bootstrap-server $DR > /dev/null 2>&1; then
    echo "ERROR: DR cluster is not reachable"
    exit 1
fi

# 2. Получение translated offsets из checkpoints
kafka-console-consumer.sh \
    --bootstrap-server $DR \
    --topic mm2-offset-syncs.dr.internal \
    --from-beginning \
    --max-messages 1000 \
    --property print.key=true > /tmp/offset-translation.txt

# 3. Применение offsets для consumer group
# (требует кастомного скрипта или использования MM2 checkpoint API)

# 4. Переключение приложений
echo "Update application configs to use: $DR"
echo "Failover complete!"
```

### Manual Offset Translation

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.*;

public class OffsetTranslator {

    public Map<TopicPartition, Long> translateOffsets(
            String sourceCluster,
            String targetBootstrap,
            String consumerGroup) {

        Map<TopicPartition, Long> translatedOffsets = new HashMap<>();

        // Чтение checkpoint топика
        String checkpointTopic = sourceCluster + ".checkpoints.internal";

        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, targetBootstrap);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "offset-translator");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(Collections.singletonList(checkpointTopic));

            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));

                for (ConsumerRecord<String, String> record : records) {
                    // Parse checkpoint record
                    Checkpoint checkpoint = parseCheckpoint(record);

                    if (checkpoint.consumerGroup().equals(consumerGroup)) {
                        TopicPartition tp = new TopicPartition(
                            checkpoint.targetTopic(),
                            checkpoint.targetPartition()
                        );
                        translatedOffsets.put(tp, checkpoint.targetOffset());
                    }
                }

                if (records.isEmpty()) break;
            }
        }

        return translatedOffsets;
    }
}
```

## Мониторинг мульти-кластера

### Ключевые метрики

```yaml
# Prometheus метрики MM2

# Lag между кластерами
kafka_connect_mirror_source_connector_replication_latency_ms

# Records replicated
kafka_connect_mirror_source_connector_record_count

# Errors
kafka_connect_mirror_source_connector_error_count

# Heartbeat lag (показывает задержку репликации)
kafka_connect_mirror_heartbeat_connector_heartbeat_latency_ms
```

### Grafana Dashboard

```json
{
  "panels": [
    {
      "title": "Replication Lag (ms)",
      "targets": [{
        "expr": "kafka_connect_mirror_source_connector_replication_latency_ms{source_cluster='prod'}"
      }]
    },
    {
      "title": "Records Replicated/sec",
      "targets": [{
        "expr": "rate(kafka_connect_mirror_source_connector_record_count[5m])"
      }]
    },
    {
      "title": "Heartbeat Status",
      "targets": [{
        "expr": "time() - kafka_connect_mirror_heartbeat_connector_last_heartbeat_timestamp"
      }]
    }
  ]
}
```

### Alerting Rules

```yaml
# prometheus-rules.yaml

groups:
  - name: kafka-mm2
    rules:
      - alert: MM2ReplicationLagHigh
        expr: kafka_connect_mirror_source_connector_replication_latency_ms > 60000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MM2 replication lag is high"

      - alert: MM2HeartbeatMissing
        expr: time() - kafka_connect_mirror_heartbeat_connector_last_heartbeat_timestamp > 300
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "MM2 heartbeat missing - replication may be down"

      - alert: MM2ConnectorDown
        expr: kafka_connect_connector_state{connector=~".*MirrorSource.*"} != 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MM2 connector is not running"
```

## Best Practices

### 1. Сетевые требования

```
Рекомендации:
• Минимальная bandwidth: 100 Mbps между кластерами
• Latency: < 100ms для comfortable репликации
• Стабильное соединение (избегайте flapping)

Расчёт bandwidth:
Required BW = (Message Rate × Avg Message Size × Replication Factor) / 0.8

Пример:
10,000 msg/s × 1KB × 1 / 0.8 = 12.5 MB/s ≈ 100 Mbps
```

### 2. Topic Naming Conventions

```bash
# Хорошая практика: включать регион в имя
us.orders.events
eu.orders.events
global.config

# После MM2 репликации:
# В EU кластере: us.orders.events (реплика из US)
```

### 3. Consumer Group Strategy

```java
// Используйте разные group.id для разных регионов
Properties props = new Properties();
props.put(ConsumerConfig.GROUP_ID_CONFIG,
    "my-app-" + System.getenv("REGION"));  // my-app-us, my-app-eu
```

### 4. Handling Conflicts in Active-Active

```java
// Last-write-wins strategy
public Order mergeOrders(Order local, Order remote) {
    if (remote.getUpdatedAt().isAfter(local.getUpdatedAt())) {
        return remote;
    }
    return local;
}

// Или используйте CRDT (Conflict-free Replicated Data Types)
```

### 5. Testing Failover

```bash
#!/bin/bash
# test-failover.sh

echo "=== Failover Test Procedure ==="

# 1. Записать текущий lag
echo "Current replication lag:"
kafka-consumer-groups.sh --bootstrap-server dr-kafka:9092 \
    --describe --group __mm2_offsets

# 2. Остановить primary (симуляция)
echo "Simulating primary failure..."

# 3. Переключить consumers
echo "Switching consumers to DR..."

# 4. Проверить continuity
echo "Verifying message continuity..."

# 5. Восстановить primary
echo "Restoring primary cluster..."

# 6. Failback
echo "Failing back to primary..."
```

## Связанные темы

- [Срок хранения данных](./01-data-retention.md) — retention в мульти-кластере
- [Возврат данных](./04-data-return.md) — восстановление после failover
- [Архитектуры](./05-architectures.md) — паттерны с мульти-кластером
- [Облачное хранение](./07-cloud-container-storage.md) — cross-region репликация

---

[prev: 05-architectures](./05-architectures.md) | [next: 07-cloud-container-storage](./07-cloud-container-storage.md)
