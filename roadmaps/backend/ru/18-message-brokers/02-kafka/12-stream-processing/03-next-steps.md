# Куда пойти дальше

[prev: 02-ksqldb](./02-ksqldb.md) | [next: 01-websockets](../../../19-real-time-data/01-websockets.md)

---

## Описание

После изучения основ потоковой обработки с Kafka Streams и ksqlDB, важно углубить знания в продвинутых паттернах обработки потоков, архитектурных решениях и экосистеме инструментов.

## Ключевые концепции

### Паттерны потоковой обработки (Stream Processing Patterns)

#### 1. Aggregation (Агрегация)

Накопление и суммирование данных по ключу или временному окну.

```java
// Kafka Streams - подсчет событий по категории
KTable<String, Long> categoryCounts = events
    .groupBy((key, event) -> event.getCategory())
    .count(Materialized.as("category-counts"));

// Kafka Streams - сумма с оконной агрегацией
KTable<Windowed<String>, Double> hourlyRevenue = orders
    .groupByKey()
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofHours(1)))
    .aggregate(
        () -> 0.0,
        (key, order, total) -> total + order.getAmount(),
        Materialized.with(Serdes.String(), Serdes.Double())
    );
```

```sql
-- ksqlDB - агрегация
CREATE TABLE category_stats AS
    SELECT category,
           COUNT(*) AS event_count,
           SUM(amount) AS total_amount,
           AVG(amount) AS avg_amount
    FROM events
    GROUP BY category
    EMIT CHANGES;
```

#### 2. Joins (Соединения)

Объединение данных из нескольких источников.

**Stream-Stream Join** - объединение двух потоков во временном окне:

```java
// Объединение заказов и платежей
KStream<String, OrderWithPayment> joined = orders.join(
    payments,
    (order, payment) -> new OrderWithPayment(order, payment),
    JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(5))
);
```

**Stream-Table Join** - обогащение потока данными из таблицы:

```java
// Обогащение событий данными пользователя
KStream<String, EnrichedEvent> enriched = events.join(
    usersTable,
    (event, user) -> new EnrichedEvent(event, user)
);
```

**Table-Table Join** - соединение двух таблиц:

```java
// Объединение профилей и настроек
KTable<String, UserWithSettings> combined = profiles.join(
    settings,
    (profile, setting) -> new UserWithSettings(profile, setting)
);
```

#### 3. Windowing (Оконные операции)

**Tumbling Windows** - непересекающиеся фиксированные окна:

```java
// Подсчет за каждые 5 минут
.windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
```

**Hopping Windows** - скользящие окна с перекрытием:

```java
// 10-минутные окна с шагом 1 минута
.windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(10))
    .advanceBy(Duration.ofMinutes(1)))
```

**Session Windows** - динамические окна на основе активности:

```java
// Сессии с gap 30 минут
.windowedBy(SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMinutes(30)))
```

**Sliding Windows** - окна с точной границей:

```java
// Для join операций
JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(5))
```

#### 4. Event-Time Processing

Обработка на основе времени события, а не времени обработки:

```java
// Настройка timestamp extractor
props.put(StreamsConfig.DEFAULT_TIMESTAMP_EXTRACTOR_CLASS_CONFIG,
    WallclockTimestampExtractor.class);

// Кастомный extractor
public class OrderTimestampExtractor implements TimestampExtractor {
    @Override
    public long extract(ConsumerRecord<Object, Object> record, long partitionTime) {
        Order order = (Order) record.value();
        return order.getOrderTime().toEpochMilli();
    }
}
```

#### 5. Late Events и Watermarks

Обработка опоздавших событий:

```java
// Настройка grace period для окна
TimeWindows.ofSizeAndGrace(
    Duration.ofMinutes(5),
    Duration.ofMinutes(1)  // Допуск опоздания на 1 минуту
);

// Suppress для ожидания финального результата
.suppress(Suppressed.untilWindowCloses(
    Suppressed.BufferConfig.unbounded()
))
```

#### 6. Exactly-Once Semantics

Гарантия обработки каждого сообщения ровно один раз:

```java
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG,
    StreamsConfig.EXACTLY_ONCE_V2);
```

### Архитектурные паттерны

#### Event Sourcing

Хранение всех изменений состояния как последовательности событий:

```java
// Поток событий как источник истины
KStream<String, DomainEvent> events = builder.stream("domain-events");

// Восстановление состояния через агрегацию
KTable<String, EntityState> currentState = events
    .groupByKey()
    .aggregate(
        EntityState::new,
        (key, event, state) -> state.apply(event),
        Materialized.as("entity-state-store")
    );
```

#### CQRS (Command Query Responsibility Segregation)

Разделение команд записи и запросов чтения:

```
Commands --> Kafka Topic --> Stream Processing --> Materialized Views
                                                         |
                                                         v
                                                   Read Queries
```

#### Saga Pattern

Распределенные транзакции через события:

```java
// Оркестратор саги
KStream<String, SagaCommand> commands = builder.stream("saga-commands");

KStream<String, SagaEvent>[] branches = commands
    .branch(
        (k, v) -> v.getType() == CommandType.START_ORDER,
        (k, v) -> v.getType() == CommandType.RESERVE_INVENTORY,
        (k, v) -> v.getType() == CommandType.PROCESS_PAYMENT,
        (k, v) -> v.getType() == CommandType.COMPLETE_ORDER
    );

// Обработка каждого шага саги
branches[0].mapValues(cmd -> processOrderStart(cmd)).to("saga-events");
branches[1].mapValues(cmd -> processInventory(cmd)).to("saga-events");
// ...
```

## Примеры кода

### Продвинутый пример: Real-time Analytics Pipeline

```java
public class RealTimeAnalytics {

    public static void main(String[] args) {
        StreamsBuilder builder = new StreamsBuilder();

        // Входной поток событий
        KStream<String, ClickEvent> clicks = builder.stream("clicks",
            Consumed.with(Serdes.String(), clickEventSerde));

        // 1. Обогащение данными пользователя
        KTable<String, UserProfile> users = builder.table("users",
            Consumed.with(Serdes.String(), userProfileSerde));

        KStream<String, EnrichedClick> enrichedClicks = clicks
            .selectKey((k, v) -> v.getUserId())
            .join(users, EnrichedClick::new);

        // 2. Агрегация по категориям с 5-минутным окном
        KTable<Windowed<String>, CategoryStats> categoryStats = enrichedClicks
            .groupBy((k, v) -> v.getCategory())
            .windowedBy(TimeWindows.ofSizeAndGrace(
                Duration.ofMinutes(5),
                Duration.ofMinutes(1)))
            .aggregate(
                CategoryStats::new,
                (key, click, stats) -> stats.add(click),
                Materialized.with(Serdes.String(), categoryStatsSerde)
            );

        // 3. Вывод результатов
        categoryStats.toStream()
            .map((windowedKey, stats) -> KeyValue.pair(
                windowedKey.key() + "@" + windowedKey.window().start(),
                stats
            ))
            .to("category-stats-output");

        // 4. Детекция аномалий
        enrichedClicks
            .groupBy((k, v) -> v.getUserId())
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
            .count()
            .toStream()
            .filter((windowedKey, count) -> count > 100)  // Более 100 кликов в минуту
            .mapValues((windowedKey, count) ->
                new AnomalyAlert(windowedKey.key(), "high_click_rate", count))
            .to("anomaly-alerts");

        KafkaStreams streams = new KafkaStreams(builder.build(), getConfig());
        streams.start();
    }
}
```

### Продвинутый пример: Multi-way Join

```java
// Объединение трех источников данных
StreamsBuilder builder = new StreamsBuilder();

KStream<String, Order> orders = builder.stream("orders");
KTable<String, Customer> customers = builder.table("customers");
KTable<String, Product> products = builder.table("products");

// Сначала обогащаем данными клиента
KStream<String, OrderWithCustomer> ordersWithCustomer = orders
    .selectKey((k, order) -> order.getCustomerId())
    .join(customers, OrderWithCustomer::new);

// Затем данными продукта
KStream<String, FullOrder> fullOrders = ordersWithCustomer
    .selectKey((k, owc) -> owc.getOrder().getProductId())
    .join(products, (owc, product) ->
        new FullOrder(owc.getOrder(), owc.getCustomer(), product));

fullOrders.to("full-orders");
```

### Продвинутый пример: Custom State Store

```java
public class CustomAggregator implements Processor<String, Event, String, AggregatedResult> {
    private KeyValueStore<String, AggregationState> stateStore;
    private ProcessorContext<String, AggregatedResult> context;

    @Override
    public void init(ProcessorContext<String, AggregatedResult> context) {
        this.context = context;
        this.stateStore = context.getStateStore("aggregation-store");

        // Периодический вывод результатов
        context.schedule(
            Duration.ofSeconds(10),
            PunctuationType.WALL_CLOCK_TIME,
            this::emitResults
        );
    }

    @Override
    public void process(Record<String, Event> record) {
        String key = record.key();
        Event event = record.value();

        AggregationState state = stateStore.get(key);
        if (state == null) {
            state = new AggregationState();
        }

        state.update(event);
        stateStore.put(key, state);
    }

    private void emitResults(long timestamp) {
        try (KeyValueIterator<String, AggregationState> iter = stateStore.all()) {
            while (iter.hasNext()) {
                KeyValue<String, AggregationState> entry = iter.next();
                if (entry.value.isComplete()) {
                    context.forward(new Record<>(
                        entry.key,
                        entry.value.toResult(),
                        timestamp
                    ));
                    stateStore.delete(entry.key);
                }
            }
        }
    }
}
```

## Best Practices

### Масштабирование

1. **Количество партиций** - определяет максимальный параллелизм
2. **Количество потоков** - `num.stream.threads` должно соответствовать партициям
3. **State store размер** - планируйте место на диске для RocksDB
4. **Распределение нагрузки** - используйте правильные ключи для равномерного распределения

### Отказоустойчивость

```java
// Настройка standby replicas
props.put(StreamsConfig.NUM_STANDBY_REPLICAS_CONFIG, 1);

// Обработка ошибок
streams.setUncaughtExceptionHandler(exception -> {
    log.error("Stream error", exception);
    return StreamsUncaughtExceptionHandler.StreamThreadExceptionResponse.REPLACE_THREAD;
});
```

### Мониторинг

Ключевые метрики для отслеживания:

- `process-rate` - скорость обработки записей
- `process-latency` - задержка обработки
- `commit-latency` - время коммита
- `poll-records` - количество записей за poll
- `state-store-size` - размер state store

### Тестирование

```java
// Unit тестирование с TopologyTestDriver
@Test
void testAggregation() {
    try (TopologyTestDriver driver = new TopologyTestDriver(topology, props)) {
        TestInputTopic<String, Event> input = driver.createInputTopic(
            "events", new StringSerializer(), eventSerializer);

        TestOutputTopic<String, Long> output = driver.createOutputTopic(
            "counts", new StringDeserializer(), new LongDeserializer());

        input.pipeInput("key1", new Event("type1"));
        input.pipeInput("key1", new Event("type1"));

        assertEquals(2L, output.readValue());
    }
}
```

## Дополнительные инструменты и технологии

### Apache Flink

Альтернатива Kafka Streams для сложной потоковой обработки:
- Поддержка event time и watermarks
- Savepoints для управляемой миграции
- CEP (Complex Event Processing)

### Apache Spark Structured Streaming

Для интеграции с batch-обработкой:
- Единый API для batch и stream
- Интеграция с ML pipeline
- SQL поддержка

### Confluent Platform

Расширения экосистемы Kafka:
- Schema Registry для управления схемами
- Kafka Connect для интеграций
- Control Center для мониторинга

## Ресурсы для изучения

### Документация

- [Kafka Streams Developer Guide](https://kafka.apache.org/documentation/streams/)
- [ksqlDB Documentation](https://docs.ksqldb.io/)
- [Confluent Developer](https://developer.confluent.io/)

### Книги

- "Kafka: The Definitive Guide" - Neha Narkhede et al.
- "Designing Data-Intensive Applications" - Martin Kleppmann
- "Streaming Systems" - Tyler Akidau et al.

### Курсы

- Confluent Kafka Streams 101
- Confluent ksqlDB 101
- Apache Kafka for Developers

### Практика

1. Создайте end-to-end pipeline с обогащением данных
2. Реализуйте real-time dashboard с оконными агрегациями
3. Постройте систему детекции аномалий
4. Реализуйте Event Sourcing + CQRS архитектуру

## Следующие шаги

1. **Изучите продвинутые паттерны** - Event Sourcing, CQRS, Saga
2. **Практикуйтесь с production-сценариями** - масштабирование, восстановление
3. **Освойте мониторинг** - настройка алертов, метрики
4. **Изучите альтернативы** - Flink, Spark Streaming для расширения кругозора
5. **Участвуйте в сообществе** - Confluent Community, Stack Overflow

---

[prev: 02-ksqldb](./02-ksqldb.md) | [next: 01-websockets](../../../19-real-time-data/01-websockets.md)
