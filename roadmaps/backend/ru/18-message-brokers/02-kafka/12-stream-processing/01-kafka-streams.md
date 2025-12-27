# Kafka Streams

## Описание

**Kafka Streams** - это клиентская библиотека для построения приложений и микросервисов, где входные и выходные данные хранятся в кластерах Apache Kafka. Библиотека позволяет писать масштабируемые, эластичные и отказоустойчивые приложения для потоковой обработки данных.

### Ключевые особенности

- **Простота развертывания** - работает как обычное Java-приложение, не требует отдельного кластера
- **Горизонтальное масштабирование** - запуск нескольких экземпляров автоматически распределяет нагрузку
- **Exactly-once семантика** - гарантия обработки каждого сообщения ровно один раз
- **Отказоустойчивость** - автоматическое восстановление при сбоях
- **Stateful и Stateless обработка** - поддержка операций с состоянием и без

## Ключевые концепции

### Топология обработки

Kafka Streams строит **топологию** - направленный ациклический граф (DAG) из узлов-процессоров:

```
Source Processor --> Stream Processor --> Stream Processor --> Sink Processor
     (читает)           (обрабатывает)       (обрабатывает)       (пишет)
```

### KStream vs KTable

| Характеристика | KStream | KTable |
|---------------|---------|--------|
| Семантика | Поток записей (insert) | Журнал изменений (upsert) |
| Дубликаты ключей | Все записи сохраняются | Последнее значение по ключу |
| Применение | События, логи | Справочники, агрегаты |

### GlobalKTable

**GlobalKTable** - реплицируется на все экземпляры приложения целиком. Используется для небольших справочников, к которым нужен доступ из любой партиции.

### State Stores

Хранилища состояния для stateful операций:

- **RocksDB** (по умолчанию) - персистентное хранилище на диске
- **In-Memory** - для небольших объемов данных

## Примеры кода

### Подключение зависимости (Maven)

```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-streams</artifactId>
    <version>3.6.0</version>
</dependency>
```

### Базовая конфигурация

```java
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.common.serialization.Serdes;
import java.util.Properties;

Properties props = new Properties();
props.put(StreamsConfig.APPLICATION_ID_CONFIG, "my-stream-app");
props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

// Для exactly-once семантики
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE_V2);
```

### Streams DSL: Простой пример фильтрации

```java
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.KStream;

StreamsBuilder builder = new StreamsBuilder();

// Читаем из топика
KStream<String, String> source = builder.stream("input-topic");

// Фильтруем и трансформируем
KStream<String, String> filtered = source
    .filter((key, value) -> value != null && value.length() > 10)
    .mapValues(value -> value.toUpperCase());

// Записываем в выходной топик
filtered.to("output-topic");

// Создаем и запускаем приложение
KafkaStreams streams = new KafkaStreams(builder.build(), props);
streams.start();

// Graceful shutdown
Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
```

### Streams DSL: Агрегация (Word Count)

```java
import org.apache.kafka.streams.kstream.*;
import org.apache.kafka.common.serialization.Serdes;
import java.util.Arrays;

StreamsBuilder builder = new StreamsBuilder();

KStream<String, String> textLines = builder.stream("text-input");

KTable<String, Long> wordCounts = textLines
    // Разбиваем строки на слова
    .flatMapValues(line -> Arrays.asList(line.toLowerCase().split("\\W+")))
    // Фильтруем пустые строки
    .filter((key, word) -> word != null && !word.isEmpty())
    // Группируем по слову (меняем ключ)
    .groupBy((key, word) -> word)
    // Подсчитываем количество
    .count(Materialized.as("word-count-store"));

// Выводим результат в топик
wordCounts.toStream().to("word-count-output",
    Produced.with(Serdes.String(), Serdes.Long()));
```

### Streams DSL: Windowing (оконные операции)

```java
import org.apache.kafka.streams.kstream.TimeWindows;
import org.apache.kafka.streams.kstream.Windowed;
import java.time.Duration;

StreamsBuilder builder = new StreamsBuilder();

KStream<String, String> events = builder.stream("events");

// Подсчет событий за каждые 5 минут
KTable<Windowed<String>, Long> windowedCounts = events
    .groupByKey()
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
    .count();

// Вывод с информацией об окне
windowedCounts.toStream()
    .map((windowedKey, count) -> {
        String key = windowedKey.key();
        long windowStart = windowedKey.window().start();
        long windowEnd = windowedKey.window().end();
        String newKey = String.format("%s@%d-%d", key, windowStart, windowEnd);
        return KeyValue.pair(newKey, count);
    })
    .to("windowed-counts");
```

### Streams DSL: Join операции

```java
import org.apache.kafka.streams.kstream.*;
import java.time.Duration;

StreamsBuilder builder = new StreamsBuilder();

// Поток заказов
KStream<String, Order> orders = builder.stream("orders");

// Таблица пользователей
KTable<String, User> users = builder.table("users");

// Stream-Table Join: обогащение заказов данными пользователя
KStream<String, EnrichedOrder> enrichedOrders = orders.join(
    users,
    (order, user) -> new EnrichedOrder(order, user),
    Joined.with(Serdes.String(), orderSerde, userSerde)
);

// Stream-Stream Join: объединение событий за временное окно
KStream<String, Payment> payments = builder.stream("payments");

KStream<String, OrderWithPayment> ordersWithPayments = orders.join(
    payments,
    (order, payment) -> new OrderWithPayment(order, payment),
    JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(5)),
    StreamJoined.with(Serdes.String(), orderSerde, paymentSerde)
);
```

### Processor API: Низкоуровневый контроль

```java
import org.apache.kafka.streams.processor.api.*;
import org.apache.kafka.streams.state.KeyValueStore;

public class CustomProcessor implements Processor<String, String, String, String> {
    private ProcessorContext<String, String> context;
    private KeyValueStore<String, Integer> stateStore;

    @Override
    public void init(ProcessorContext<String, String> context) {
        this.context = context;
        this.stateStore = context.getStateStore("my-state-store");

        // Периодическая обработка каждые 30 секунд
        context.schedule(
            Duration.ofSeconds(30),
            PunctuationType.WALL_CLOCK_TIME,
            this::punctuate
        );
    }

    @Override
    public void process(Record<String, String> record) {
        String key = record.key();
        String value = record.value();

        // Получаем текущий счетчик
        Integer count = stateStore.get(key);
        count = (count == null) ? 1 : count + 1;
        stateStore.put(key, count);

        // Отправляем обработанную запись
        context.forward(new Record<>(key, value + ":" + count, record.timestamp()));
    }

    private void punctuate(long timestamp) {
        // Периодическая логика (например, flush или агрегация)
        try (var iterator = stateStore.all()) {
            while (iterator.hasNext()) {
                var entry = iterator.next();
                context.forward(new Record<>(entry.key, "periodic:" + entry.value, timestamp));
            }
        }
    }

    @Override
    public void close() {
        // Освобождение ресурсов
    }
}
```

### Использование Processor API в топологии

```java
import org.apache.kafka.streams.Topology;
import org.apache.kafka.streams.state.Stores;
import org.apache.kafka.streams.state.KeyValueBytesStoreSupplier;

Topology topology = new Topology();

// Добавляем state store
KeyValueBytesStoreSupplier storeSupplier = Stores.persistentKeyValueStore("my-state-store");

topology.addSource("source", "input-topic")
        .addProcessor("processor", CustomProcessor::new, "source")
        .addStateStore(
            Stores.keyValueStoreBuilder(storeSupplier, Serdes.String(), Serdes.Integer()),
            "processor"
        )
        .addSink("sink", "output-topic", "processor");

KafkaStreams streams = new KafkaStreams(topology, props);
streams.start();
```

### Interactive Queries: Запросы к состоянию

```java
import org.apache.kafka.streams.StoreQueryParameters;
import org.apache.kafka.streams.state.QueryableStoreTypes;
import org.apache.kafka.streams.state.ReadOnlyKeyValueStore;

// Получение доступа к state store для чтения
ReadOnlyKeyValueStore<String, Long> keyValueStore = streams.store(
    StoreQueryParameters.fromNameAndType(
        "word-count-store",
        QueryableStoreTypes.keyValueStore()
    )
);

// Запрос значения по ключу
Long count = keyValueStore.get("hello");
System.out.println("Count for 'hello': " + count);

// Итерация по всем значениям
try (var iterator = keyValueStore.all()) {
    while (iterator.hasNext()) {
        var entry = iterator.next();
        System.out.println(entry.key + " = " + entry.value);
    }
}
```

## Best Practices

### Конфигурация и настройка

1. **Количество потоков обработки**
   ```java
   // Количество потоков = количество партиций входного топика
   props.put(StreamsConfig.NUM_STREAM_THREADS_CONFIG, 4);
   ```

2. **Настройка коммитов**
   ```java
   // Интервал коммита offset (по умолчанию 30 секунд)
   props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 1000);
   ```

3. **Настройка кэширования**
   ```java
   // Размер кэша записи (уменьшает количество записей в state store)
   props.put(StreamsConfig.STATESTORE_CACHE_MAX_BYTES_CONFIG, 10 * 1024 * 1024);
   ```

### Обработка ошибок

```java
// Обработчик ошибок десериализации
props.put(
    StreamsConfig.DEFAULT_DESERIALIZATION_EXCEPTION_HANDLER_CLASS_CONFIG,
    LogAndContinueExceptionHandler.class
);

// Обработчик ошибок продюсера
props.put(
    StreamsConfig.DEFAULT_PRODUCTION_EXCEPTION_HANDLER_CLASS_CONFIG,
    DefaultProductionExceptionHandler.class
);

// Глобальный обработчик необработанных исключений
streams.setUncaughtExceptionHandler((exception) -> {
    log.error("Uncaught exception in stream thread", exception);
    // REPLACE_THREAD - заменить поток
    // SHUTDOWN_CLIENT - остановить приложение
    // SHUTDOWN_APPLICATION - остановить все экземпляры
    return StreamsUncaughtExceptionHandler.StreamThreadExceptionResponse.REPLACE_THREAD;
});
```

### Мониторинг

```java
// Получение метрик
streams.metrics().forEach((metricName, metric) -> {
    System.out.println(metricName + " = " + metric.metricValue());
});

// Отслеживание состояния приложения
streams.setStateListener((newState, oldState) -> {
    System.out.println("State changed from " + oldState + " to " + newState);
});
```

### Graceful Shutdown

```java
// Корректное завершение работы
CountDownLatch latch = new CountDownLatch(1);

Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    streams.close(Duration.ofSeconds(30));
    latch.countDown();
}));

try {
    streams.start();
    latch.await();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

### Рекомендации по проектированию

1. **Избегайте внешних вызовов** в процессорах - используйте async patterns или обогащение через join
2. **Правильно выбирайте ключи** - ключ определяет партицию и возможности join
3. **Используйте Avro/Protobuf** для сериализации сложных объектов
4. **Настройте changelog топики** - используйте compaction и retention
5. **Тестируйте с TopologyTestDriver** - unit-тесты без реального Kafka

### Тестирование

```java
import org.apache.kafka.streams.TopologyTestDriver;
import org.apache.kafka.streams.TestInputTopic;
import org.apache.kafka.streams.TestOutputTopic;

@Test
void testWordCount() {
    StreamsBuilder builder = new StreamsBuilder();
    // ... построение топологии ...

    try (TopologyTestDriver testDriver = new TopologyTestDriver(builder.build(), props)) {
        TestInputTopic<String, String> inputTopic = testDriver.createInputTopic(
            "input-topic",
            new StringSerializer(),
            new StringSerializer()
        );

        TestOutputTopic<String, Long> outputTopic = testDriver.createOutputTopic(
            "output-topic",
            new StringDeserializer(),
            new LongDeserializer()
        );

        inputTopic.pipeInput("key", "hello world hello");

        Map<String, Long> results = outputTopic.readKeyValuesToMap();
        assertEquals(2L, results.get("hello"));
        assertEquals(1L, results.get("world"));
    }
}
```

## Типы окон (Window Types)

| Тип окна | Описание | Использование |
|----------|----------|---------------|
| **Tumbling** | Непересекающиеся окна фиксированного размера | Агрегация за периоды |
| **Hopping** | Скользящие окна с перекрытием | Сглаживание данных |
| **Sliding** | Окна с фиксированной разницей во времени | Join операции |
| **Session** | Динамические окна на основе активности | Анализ сессий |

```java
// Tumbling Window: 5-минутные окна без перекрытия
TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5))

// Hopping Window: 5-минутные окна с шагом 1 минута
TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5))
    .advanceBy(Duration.ofMinutes(1))

// Session Window: группировка по активности с gap 10 минут
SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMinutes(10))
```
