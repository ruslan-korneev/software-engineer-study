# Архитектуры с использованием Kafka

## Описание

Kafka — это не просто брокер сообщений, а платформа для построения распределённых систем обработки данных в реальном времени. В этом разделе рассматриваются основные архитектурные паттерны использования Kafka как системы хранения и обработки данных.

## Ключевые концепции

### Kafka как центральная нервная система

```
┌─────────────────────────────────────────────────────────────────┐
│                        Data Sources                              │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────────────┐  │
│  │ Apps │ │ IoT  │ │ Logs │ │ DBs  │ │ APIs │ │ External     │  │
│  └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ │   Systems    │  │
│     │        │        │        │        │     └───────┬──────┘  │
└─────┼────────┼────────┼────────┼────────┼─────────────┼─────────┘
      │        │        │        │        │             │
      └────────┴────────┴───┬────┴────────┴─────────────┘
                            │
                    ┌───────▼───────┐
                    │               │
                    │  Apache Kafka │
                    │   (Storage)   │
                    │               │
                    └───────┬───────┘
                            │
      ┌────────┬────────┬───┴────┬────────┬─────────────┐
      │        │        │        │        │             │
┌─────▼────┐ ┌─▼────┐ ┌─▼────┐ ┌─▼────┐ ┌─▼────┐ ┌─────▼─────┐
│ Stream   │ │ Data │ │ OLAP │ │ ML   │ │Search│ │Microserv. │
│Processing│ │ Lake │ │ DBs  │ │Pipel.│ │Engine│ │           │
└──────────┘ └──────┘ └──────┘ └──────┘ └──────┘ └───────────┘
```

## Архитектурные паттерны

### 1. Event Sourcing

Все изменения состояния сохраняются как последовательность событий:

```
┌─────────────────────────────────────────────────────┐
│                   Event Store (Kafka)               │
├─────────────────────────────────────────────────────┤
│ OrderCreated → ItemAdded → ItemRemoved → OrderPaid  │
└─────────────────────────────────────────────────────┘
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
    ┌───────────┐ ┌───────────┐ ┌───────────┐
    │ Orders    │ │ Analytics │ │ Audit     │
    │ Service   │ │ Service   │ │ Service   │
    │           │ │           │ │           │
    │ (rebuild  │ │ (read     │ │ (archive) │
    │  state)   │ │  events)  │ │           │
    └───────────┘ └───────────┘ └───────────┘
```

**Конфигурация топика для Event Sourcing:**

```bash
kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic order-events \
  --partitions 12 \
  --replication-factor 3 \
  --config cleanup.policy=compact \
  --config retention.ms=-1 \
  --config min.compaction.lag.ms=3600000
```

**Пример события:**

```java
@Data
public class OrderEvent {
    private String eventId;
    private String orderId;  // Partition key
    private String eventType;
    private Instant timestamp;
    private Map<String, Object> payload;
}

// Producer
public void publishEvent(OrderEvent event) {
    ProducerRecord<String, OrderEvent> record =
        new ProducerRecord<>("order-events", event.getOrderId(), event);

    producer.send(record).get();  // Синхронно для гарантий
}
```

### 2. CQRS (Command Query Responsibility Segregation)

Разделение записи и чтения:

```
┌────────────────────────────────────────────────────────────┐
│                      Commands                               │
│  ┌────────┐      ┌────────────┐      ┌───────────────┐     │
│  │ Create │  →   │  Command   │  →   │    Kafka      │     │
│  │ Update │      │  Handler   │      │  (Write Log)  │     │
│  │ Delete │      └────────────┘      └───────┬───────┘     │
│  └────────┘                                  │             │
└──────────────────────────────────────────────┼─────────────┘
                                               │
                              ┌────────────────┼────────────────┐
                              │                │                │
                              ▼                ▼                ▼
                      ┌───────────┐    ┌───────────┐    ┌───────────┐
                      │PostgreSQL │    │   Redis   │    │Elastic    │
                      │(Relations)│    │  (Cache)  │    │Search     │
                      └─────┬─────┘    └─────┬─────┘    └─────┬─────┘
                            │                │                │
┌───────────────────────────┼────────────────┼────────────────┼─────┐
│                           ▼                ▼                ▼     │
│  ┌────────┐      ┌────────────────────────────────────────────┐  │
│  │  List  │  ←   │              Query Handler                 │  │
│  │  Get   │      │         (Read Optimized Views)             │  │
│  │ Search │      └────────────────────────────────────────────┘  │
│  └────────┘                                                      │
│                           Queries                                │
└──────────────────────────────────────────────────────────────────┘
```

**Kafka Streams для материализации view:**

```java
StreamsBuilder builder = new StreamsBuilder();

KStream<String, OrderEvent> events =
    builder.stream("order-events");

// Материализация в KTable
KTable<String, Order> orders = events
    .groupByKey()
    .aggregate(
        Order::new,
        (key, event, order) -> order.apply(event),
        Materialized.<String, Order, KeyValueStore<Bytes, byte[]>>as("orders-store")
            .withKeySerde(Serdes.String())
            .withValueSerde(orderSerde)
    );

// Интерактивные запросы
ReadOnlyKeyValueStore<String, Order> store =
    streams.store(
        StoreQueryParameters.fromNameAndType(
            "orders-store",
            QueryableStoreTypes.keyValueStore()
        )
    );

Order order = store.get("order-123");
```

### 3. Lambda Architecture

Объединение batch и stream processing:

```
                    ┌─────────────────┐
                    │   Data Source   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │      Kafka      │
                    │  (Raw Events)   │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
           ▼                 ▼                 ▼
┌──────────────────┐ ┌───────────────┐ ┌───────────────┐
│   Batch Layer    │ │ Speed Layer   │ │ Serving Layer │
│                  │ │               │ │               │
│  ┌────────────┐  │ │ ┌───────────┐ │ │ ┌───────────┐ │
│  │   Spark    │  │ │ │  Kafka    │ │ │ │ Cassandra │ │
│  │   (Daily)  │  │ │ │  Streams  │ │ │ │    or     │ │
│  └──────┬─────┘  │ │ │ (Real-time│ │ │ │  Druid    │ │
│         │        │ │ └─────┬─────┘ │ │ └─────┬─────┘ │
│         ▼        │ │       │       │ │       │       │
│  ┌────────────┐  │ │       │       │ │       │       │
│  │   HDFS/    │  │ │       │       │ │       │       │
│  │    S3      │  │ │       │       │ │       │       │
│  └──────┬─────┘  │ │       │       │ │       │       │
│         │        │ │       │       │ │       │       │
└─────────┼────────┘ └───────┼───────┘ └───────┼───────┘
          │                  │                 │
          └──────────────────┴─────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Query Service  │
                    │ (Batch + Speed) │
                    └─────────────────┘
```

### 4. Kappa Architecture

Упрощённая версия Lambda — только stream processing:

```
           ┌─────────────────────────────────────────────┐
           │              Data Sources                    │
           └──────────────────┬──────────────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │                   │
                    │   Apache Kafka    │
                    │  (Immutable Log)  │
                    │                   │
                    └─────────┬─────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
      ┌───────────────┐ ┌───────────┐ ┌───────────────┐
      │ Stream App v1 │ │Stream App │ │ Stream App v3 │
      │   (Legacy)    │ │    v2     │ │   (Current)   │
      └───────┬───────┘ └─────┬─────┘ └───────┬───────┘
              │               │               │
              └───────────────┼───────────────┘
                              │
                    ┌─────────▼─────────┐
                    │   Serving Layer   │
                    │  (DB/Cache/etc)   │
                    └───────────────────┘
```

**Преимущества:**
- Единая кодовая база для batch и stream
- Reprocessing через replay из Kafka
- Версионирование обработки

### 5. Data Mesh с Kafka

```
┌─────────────────────────────────────────────────────────────────┐
│                        Domain Teams                              │
├─────────────────┬─────────────────┬─────────────────────────────┤
│  Orders Domain  │  Users Domain   │      Products Domain        │
│                 │                 │                             │
│ ┌─────────────┐ │ ┌─────────────┐ │ ┌─────────────────────────┐ │
│ │ order-      │ │ │ user-       │ │ │ product-events          │ │
│ │ events      │ │ │ events      │ │ │ product-catalog         │ │
│ │ order-      │ │ │ user-       │ │ │ inventory-updates       │ │
│ │ snapshots   │ │ │ profiles    │ │ │                         │ │
│ └──────┬──────┘ │ └──────┬──────┘ │ └───────────┬─────────────┘ │
│        │        │        │        │             │               │
└────────┼────────┴────────┼────────┴─────────────┼───────────────┘
         │                 │                      │
         └─────────────────┼──────────────────────┘
                           │
              ┌────────────▼────────────┐
              │    Kafka (Data Plane)   │
              │                         │
              │  • Schema Registry      │
              │  • Access Control       │
              │  • Data Catalog         │
              │  • Lineage Tracking     │
              └────────────┬────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
         ▼                 ▼                 ▼
┌─────────────────┐ ┌─────────────┐ ┌─────────────────┐
│ Analytics Team  │ │   ML Team   │ │ Reporting Team  │
│ (Consumers)     │ │ (Consumers) │ │  (Consumers)    │
└─────────────────┘ └─────────────┘ └─────────────────┘
```

### 6. Change Data Capture (CDC)

```
┌─────────────────────────────────────────────────────────────┐
│                     Source Databases                         │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐ │
│  │  PostgreSQL  │ │    MySQL     │ │      MongoDB         │ │
│  │  (Orders)    │ │   (Users)    │ │    (Products)        │ │
│  └──────┬───────┘ └──────┬───────┘ └──────────┬───────────┘ │
│         │                │                    │             │
└─────────┼────────────────┼────────────────────┼─────────────┘
          │                │                    │
          ▼                ▼                    ▼
┌─────────────────────────────────────────────────────────────┐
│                      Debezium                                │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐ │
│  │ PostgreSQL   │ │   MySQL      │ │    MongoDB           │ │
│  │ Connector    │ │  Connector   │ │    Connector         │ │
│  └──────┬───────┘ └──────┬───────┘ └──────────┬───────────┘ │
│         │                │                    │             │
└─────────┼────────────────┼────────────────────┼─────────────┘
          │                │                    │
          └────────────────┼────────────────────┘
                           │
                  ┌────────▼────────┐
                  │     Kafka       │
                  │                 │
                  │ • orders.cdc    │
                  │ • users.cdc     │
                  │ • products.cdc  │
                  └────────┬────────┘
                           │
       ┌───────────────────┼───────────────────┐
       │                   │                   │
       ▼                   ▼                   ▼
┌─────────────┐    ┌───────────────┐   ┌─────────────────┐
│ Data Lake   │    │ Elasticsearch │   │  Target DBs     │
│ (S3/HDFS)   │    │ (Search)      │   │  (Replication)  │
└─────────────┘    └───────────────┘   └─────────────────┘
```

**Debezium конфигурация:**

```json
{
  "name": "postgres-cdc-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "tasks.max": "1",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "${file:/secrets/db.properties:password}",
    "database.dbname": "orders_db",
    "topic.prefix": "cdc",
    "table.include.list": "public.orders,public.order_items",

    "publication.autocreate.mode": "filtered",
    "slot.name": "debezium_orders",

    "transforms": "route",
    "transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.route.regex": "cdc.public.(.*)",
    "transforms.route.replacement": "orders.$1.events"
  }
}
```

## Примеры конфигурации

### Kafka для Event Sourcing

```properties
# server.properties

# Длительное хранение для event store
log.retention.ms=-1
log.retention.bytes=-1

# Compaction для снэпшотов
log.cleanup.policy=compact

# Гарантии доставки
min.insync.replicas=2
unclean.leader.election.enable=false

# Производительность
num.io.threads=16
num.network.threads=8
```

### Kafka Streams для CQRS

```java
Properties props = new Properties();
props.put(StreamsConfig.APPLICATION_ID_CONFIG, "cqrs-materializer");
props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, "exactly_once_v2");
props.put(StreamsConfig.STATE_DIR_CONFIG, "/data/kafka-streams");
props.put(StreamsConfig.NUM_STREAM_THREADS_CONFIG, 4);

// Настройка state store
props.put(StreamsConfig.ROCKSDB_CONFIG_SETTER_CLASS_CONFIG,
    CustomRocksDBConfig.class);
```

### Топология для Lambda Architecture

```java
// Speed layer: real-time aggregation
KStream<String, PageView> views = builder.stream("page-views");

KTable<Windowed<String>, Long> realtimeCounts = views
    .groupBy((key, value) -> value.getPageId())
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
    .count(Materialized.as("realtime-counts"));

// Результат в serving layer topic
realtimeCounts
    .toStream()
    .to("page-view-counts-5min");
```

## Best Practices

### 1. Выбор архитектуры

| Сценарий | Рекомендуемая архитектура |
|----------|--------------------------|
| Аудит и compliance | Event Sourcing |
| Разные модели чтения | CQRS |
| Batch + Real-time | Kappa (предпочтительно) |
| Синхронизация БД | CDC с Debezium |
| Множество доменов | Data Mesh |

### 2. Гранулярность топиков

```bash
# Плохо: один топик для всего
events

# Хорошо: разделение по домену и типу
orders.created
orders.updated
orders.cancelled
users.registered
users.profile-updated
```

### 3. Версионирование событий

```java
// Используйте Schema Registry
public class OrderEventV1 {
    private String orderId;
    private List<String> items;
}

public class OrderEventV2 {
    private String orderId;
    private List<OrderItem> items;  // Расширенная структура
    private BigDecimal totalAmount;
}
```

### 4. Идемпотентность обработки

```java
@Transactional
public void processEvent(OrderEvent event) {
    // Проверка на дублирование
    if (processedEvents.contains(event.getEventId())) {
        log.info("Event already processed: {}", event.getEventId());
        return;
    }

    // Обработка
    orderService.apply(event);

    // Запись об обработке
    processedEvents.add(event.getEventId());
}
```

### 5. Мониторинг архитектуры

```yaml
# Prometheus метрики для отслеживания
kafka_streams_state_store_size_bytes
kafka_consumer_records_lag_max
kafka_producer_record_send_rate
```

## Связанные темы

- [Срок хранения данных](./01-data-retention.md) — retention для разных паттернов
- [Инструменты](./03-tools.md) — CLI для управления
- [Мульти-кластер](./06-multi-cluster.md) — распределённые архитектуры
- [Облачное хранение](./07-cloud-container-storage.md) — интеграция с облаком
