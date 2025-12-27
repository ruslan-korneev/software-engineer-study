# Тестирование с EmbeddedKafkaCluster

## Описание

**EmbeddedKafka** — это встраиваемый Kafka-брокер для локального тестирования, который позволяет запускать полноценный Kafka-кластер в памяти без необходимости внешней инфраструктуры. Это критически важный инструмент для написания интеграционных тестов приложений, работающих с Kafka.

### Зачем нужен EmbeddedKafka?

- **Изолированное тестирование**: тесты не зависят от внешних сервисов
- **Скорость**: быстрый запуск и остановка кластера
- **Воспроизводимость**: одинаковое поведение на любом окружении
- **CI/CD интеграция**: легко интегрируется в пайплайны
- **Параллельный запуск**: каждый тест может иметь свой кластер

### Варианты реализации

| Библиотека | Экосистема | Особенности |
|------------|------------|-------------|
| `spring-kafka-test` | Spring | Аннотация `@EmbeddedKafka` |
| `kafka-junit` | JUnit 4/5 | Rule/Extension для JUnit |
| `testcontainers` | Любая | Docker-контейнер с реальным Kafka |
| `embedded-kafka` | Scala | Нативная поддержка Scala |

## Ключевые концепции

### Архитектура тестирования

```
┌─────────────────────────────────────────────────────────────┐
│                      Test Class                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐  │
│  │  Producer   │───→│ EmbeddedKafka│───→│    Consumer     │  │
│  │   (Test)    │    │   Broker     │    │    (SUT)        │  │
│  └─────────────┘    └─────────────┘    └─────────────────┘  │
│         ↑                  │                    │            │
│         │                  ↓                    ↓            │
│         │           ┌─────────────┐      ┌──────────┐       │
│         └───────────│    Topic    │      │ Assertions│       │
│                     └─────────────┘      └──────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### Жизненный цикл теста

```
1. @BeforeAll  → Запуск EmbeddedKafka
2. @BeforeEach → Создание топиков, очистка состояния
3. Test        → Отправка/получение сообщений
4. @AfterEach  → Удаление топиков
5. @AfterAll   → Остановка EmbeddedKafka
```

## Примеры

### Spring Boot с @EmbeddedKafka

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka-test</artifactId>
    <scope>test</scope>
</dependency>
```

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.test.context.EmbeddedKafka;
import org.springframework.kafka.test.EmbeddedKafkaBroker;
import org.springframework.test.annotation.DirtiesContext;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@DirtiesContext
@EmbeddedKafka(
    partitions = 3,
    topics = {"test-topic", "orders"},
    brokerProperties = {
        "listeners=PLAINTEXT://localhost:9092",
        "port=9092",
        "log.dir=/tmp/kafka-logs"
    }
)
class KafkaIntegrationTest {

    @Autowired
    private EmbeddedKafkaBroker embeddedKafka;

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    private OrderConsumer orderConsumer;

    @Test
    void shouldSendAndReceiveMessage() throws Exception {
        // Given
        String topic = "test-topic";
        String message = "Hello, Kafka!";

        // When
        kafkaTemplate.send(topic, message).get();

        // Then
        // Ожидаем получения сообщения консьюмером
        Thread.sleep(1000);
        assertThat(orderConsumer.getReceivedMessages()).contains(message);
    }

    @Test
    void shouldSendToSpecificPartition() throws Exception {
        // Given
        String topic = "orders";
        String key = "order-123";
        String value = "{\"orderId\": \"123\", \"status\": \"created\"}";

        // When
        var result = kafkaTemplate.send(topic, key, value).get();

        // Then
        assertThat(result.getRecordMetadata().topic()).isEqualTo(topic);
        assertThat(result.getRecordMetadata().partition()).isGreaterThanOrEqualTo(0);
    }
}
```

### Конфигурация теста с ConsumerRecords

```java
import org.apache.kafka.clients.consumer.Consumer;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.test.utils.KafkaTestUtils;

import java.time.Duration;
import java.util.Collections;
import java.util.Map;

@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"test-topic"})
class KafkaConsumerTest {

    @Autowired
    private EmbeddedKafkaBroker embeddedKafka;

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Test
    void shouldConsumeMessage() {
        // Создаем test consumer
        Map<String, Object> consumerProps = KafkaTestUtils.consumerProps(
            "test-group", "true", embeddedKafka);
        consumerProps.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        DefaultKafkaConsumerFactory<String, String> consumerFactory =
            new DefaultKafkaConsumerFactory<>(
                consumerProps,
                new StringDeserializer(),
                new StringDeserializer()
            );

        Consumer<String, String> consumer = consumerFactory.createConsumer();
        consumer.subscribe(Collections.singletonList("test-topic"));

        // Отправляем сообщение
        kafkaTemplate.send("test-topic", "key1", "value1");
        kafkaTemplate.flush();

        // Читаем сообщение
        ConsumerRecords<String, String> records =
            KafkaTestUtils.getRecords(consumer, Duration.ofSeconds(10));

        assertThat(records.count()).isEqualTo(1);
        assertThat(records.iterator().next().value()).isEqualTo("value1");

        consumer.close();
    }
}
```

### JUnit 5 Extension для Kafka

```java
import org.apache.kafka.clients.admin.AdminClient;
import org.apache.kafka.clients.admin.NewTopic;
import org.junit.jupiter.api.extension.*;

import java.util.*;

public class EmbeddedKafkaExtension implements BeforeAllCallback, AfterAllCallback {

    private EmbeddedKafkaCluster kafka;

    @Override
    public void beforeAll(ExtensionContext context) throws Exception {
        kafka = new EmbeddedKafkaCluster(1);
        kafka.start();

        // Регистрируем в контексте теста
        context.getStore(ExtensionContext.Namespace.GLOBAL)
            .put("kafka", kafka);
    }

    @Override
    public void afterAll(ExtensionContext context) throws Exception {
        if (kafka != null) {
            kafka.stop();
        }
    }
}

// Использование
@ExtendWith(EmbeddedKafkaExtension.class)
class MyKafkaTest {

    @Test
    void testWithEmbeddedKafka(EmbeddedKafkaCluster kafka) {
        // Тест с embedded Kafka
    }
}
```

### Testcontainers для Kafka

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>kafka</artifactId>
    <version>1.19.3</version>
    <scope>test</scope>
</dependency>
```

```java
import org.testcontainers.containers.KafkaContainer;
import org.testcontainers.utility.DockerImageName;
import org.junit.jupiter.api.*;

class KafkaTestcontainersTest {

    static KafkaContainer kafka;

    @BeforeAll
    static void startKafka() {
        kafka = new KafkaContainer(
            DockerImageName.parse("confluentinc/cp-kafka:7.5.0")
        );
        kafka.start();
    }

    @AfterAll
    static void stopKafka() {
        kafka.stop();
    }

    @Test
    void shouldProduceAndConsume() {
        String bootstrapServers = kafka.getBootstrapServers();

        // Настраиваем producer
        Properties producerProps = new Properties();
        producerProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        producerProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
            StringSerializer.class);
        producerProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
            StringSerializer.class);

        try (KafkaProducer<String, String> producer = new KafkaProducer<>(producerProps)) {
            producer.send(new ProducerRecord<>("test", "key", "value")).get();
        }

        // Настраиваем consumer
        Properties consumerProps = new Properties();
        consumerProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        consumerProps.put(ConsumerConfig.GROUP_ID_CONFIG, "test-group");
        consumerProps.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        consumerProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
            StringDeserializer.class);
        consumerProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
            StringDeserializer.class);

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerProps)) {
            consumer.subscribe(Collections.singletonList("test"));

            ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(10));

            assertThat(records.count()).isEqualTo(1);
        }
    }
}
```

### Тестирование с динамическим портом

```java
@SpringBootTest
@EmbeddedKafka(
    partitions = 1,
    brokerProperties = {"listeners=PLAINTEXT://localhost:0"}  // Динамический порт
)
@TestPropertySource(properties = {
    "spring.kafka.bootstrap-servers=${spring.embedded.kafka.brokers}"
})
class DynamicPortKafkaTest {

    @Autowired
    private EmbeddedKafkaBroker embeddedKafka;

    @Test
    void shouldWorkWithDynamicPort() {
        String brokers = embeddedKafka.getBrokersAsString();
        System.out.println("Kafka brokers: " + brokers);
        // Тест
    }
}
```

### Тестирование Kafka Streams

```java
import org.apache.kafka.streams.*;
import org.apache.kafka.streams.test.TestRecord;

class KafkaStreamsTest {

    private TopologyTestDriver testDriver;
    private TestInputTopic<String, String> inputTopic;
    private TestOutputTopic<String, String> outputTopic;

    @BeforeEach
    void setup() {
        // Создаем топологию
        StreamsBuilder builder = new StreamsBuilder();
        builder.<String, String>stream("input-topic")
            .mapValues(value -> value.toUpperCase())
            .to("output-topic");

        Topology topology = builder.build();

        // Конфигурация для тестов
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "test-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "dummy:1234");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG,
            Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG,
            Serdes.String().getClass());

        // Создаем test driver
        testDriver = new TopologyTestDriver(topology, props);

        inputTopic = testDriver.createInputTopic(
            "input-topic",
            new StringSerializer(),
            new StringSerializer()
        );

        outputTopic = testDriver.createOutputTopic(
            "output-topic",
            new StringDeserializer(),
            new StringDeserializer()
        );
    }

    @AfterEach
    void tearDown() {
        testDriver.close();
    }

    @Test
    void shouldTransformToUpperCase() {
        // Given
        inputTopic.pipeInput("key1", "hello");
        inputTopic.pipeInput("key2", "world");

        // When & Then
        TestRecord<String, String> record1 = outputTopic.readRecord();
        assertThat(record1.getValue()).isEqualTo("HELLO");

        TestRecord<String, String> record2 = outputTopic.readRecord();
        assertThat(record2.getValue()).isEqualTo("WORLD");

        assertThat(outputTopic.isEmpty()).isTrue();
    }
}
```

### Тестирование Producer с MockProducer

```java
import org.apache.kafka.clients.producer.MockProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.serialization.StringSerializer;

class MockProducerTest {

    private MockProducer<String, String> mockProducer;
    private OrderService orderService;

    @BeforeEach
    void setup() {
        mockProducer = new MockProducer<>(
            true,  // autoComplete
            new StringSerializer(),
            new StringSerializer()
        );

        orderService = new OrderService(mockProducer);
    }

    @Test
    void shouldSendOrderCreatedEvent() {
        // Given
        Order order = new Order("order-123", "user-456", 100.0);

        // When
        orderService.createOrder(order);

        // Then
        List<ProducerRecord<String, String>> history = mockProducer.history();
        assertThat(history).hasSize(1);

        ProducerRecord<String, String> record = history.get(0);
        assertThat(record.topic()).isEqualTo("orders");
        assertThat(record.key()).isEqualTo("order-123");
        assertThat(record.value()).contains("order-123");
    }

    @Test
    void shouldHandleProducerError() {
        // Создаем producer без auto-complete
        MockProducer<String, String> errorProducer = new MockProducer<>(
            false,
            new StringSerializer(),
            new StringSerializer()
        );

        OrderService service = new OrderService(errorProducer);

        // When
        CompletableFuture<Void> future = service.createOrderAsync(
            new Order("order-456", "user-789", 50.0)
        );

        // Симулируем ошибку
        RuntimeException error = new RuntimeException("Kafka error");
        errorProducer.errorNext(error);

        // Then
        assertThatThrownBy(future::join)
            .hasCauseInstanceOf(RuntimeException.class);
    }
}
```

### Тестирование Consumer с MockConsumer

```java
import org.apache.kafka.clients.consumer.MockConsumer;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.OffsetResetStrategy;
import org.apache.kafka.common.TopicPartition;

class MockConsumerTest {

    private MockConsumer<String, String> mockConsumer;
    private OrderProcessor orderProcessor;

    @BeforeEach
    void setup() {
        mockConsumer = new MockConsumer<>(OffsetResetStrategy.EARLIEST);
        orderProcessor = new OrderProcessor(mockConsumer);
    }

    @Test
    void shouldProcessOrders() {
        // Setup: assign partition и установить offset
        TopicPartition tp = new TopicPartition("orders", 0);
        mockConsumer.assign(Collections.singletonList(tp));

        HashMap<TopicPartition, Long> beginningOffsets = new HashMap<>();
        beginningOffsets.put(tp, 0L);
        mockConsumer.updateBeginningOffsets(beginningOffsets);

        // Добавляем тестовые записи
        mockConsumer.addRecord(new ConsumerRecord<>(
            "orders", 0, 0L, "order-1", "{\"orderId\":\"1\",\"amount\":100}"
        ));
        mockConsumer.addRecord(new ConsumerRecord<>(
            "orders", 0, 1L, "order-2", "{\"orderId\":\"2\",\"amount\":200}"
        ));

        // When
        List<Order> processedOrders = orderProcessor.processNextBatch();

        // Then
        assertThat(processedOrders).hasSize(2);
        assertThat(processedOrders.get(0).getOrderId()).isEqualTo("1");
        assertThat(processedOrders.get(1).getOrderId()).isEqualTo("2");
    }

    @Test
    void shouldHandleRebalance() {
        TopicPartition tp0 = new TopicPartition("orders", 0);
        TopicPartition tp1 = new TopicPartition("orders", 1);

        // Симулируем rebalance
        mockConsumer.rebalance(Arrays.asList(tp0, tp1));

        HashMap<TopicPartition, Long> offsets = new HashMap<>();
        offsets.put(tp0, 0L);
        offsets.put(tp1, 0L);
        mockConsumer.updateBeginningOffsets(offsets);

        assertThat(mockConsumer.assignment()).containsExactlyInAnyOrder(tp0, tp1);
    }
}
```

### Python: тестирование с pytest и kafka-python

```python
import pytest
from kafka import KafkaProducer, KafkaConsumer
from kafka.admin import KafkaAdminClient, NewTopic
from testcontainers.kafka import KafkaContainer
import json


@pytest.fixture(scope="session")
def kafka_container():
    """Запуск Kafka контейнера для тестов"""
    with KafkaContainer() as kafka:
        yield kafka


@pytest.fixture
def kafka_producer(kafka_container):
    """Создание producer для тестов"""
    producer = KafkaProducer(
        bootstrap_servers=kafka_container.get_bootstrap_server(),
        value_serializer=lambda v: json.dumps(v).encode('utf-8'),
        key_serializer=lambda k: k.encode('utf-8') if k else None
    )
    yield producer
    producer.close()


@pytest.fixture
def kafka_consumer(kafka_container):
    """Создание consumer для тестов"""
    consumer = KafkaConsumer(
        'test-topic',
        bootstrap_servers=kafka_container.get_bootstrap_server(),
        auto_offset_reset='earliest',
        group_id='test-group',
        value_deserializer=lambda v: json.loads(v.decode('utf-8')),
        consumer_timeout_ms=5000
    )
    yield consumer
    consumer.close()


def test_produce_and_consume(kafka_producer, kafka_consumer):
    """Тест отправки и получения сообщения"""
    # Given
    message = {"order_id": "123", "amount": 100.0}

    # When
    kafka_producer.send('test-topic', key='order-123', value=message)
    kafka_producer.flush()

    # Then
    messages = list(kafka_consumer)
    assert len(messages) == 1
    assert messages[0].value == message
    assert messages[0].key.decode('utf-8') == 'order-123'


def test_partition_assignment(kafka_container):
    """Тест назначения партиций"""
    admin = KafkaAdminClient(
        bootstrap_servers=kafka_container.get_bootstrap_server()
    )

    # Создаем топик с 3 партициями
    topic = NewTopic(name='partitioned-topic', num_partitions=3, replication_factor=1)
    admin.create_topics([topic])

    # Проверяем
    consumer = KafkaConsumer(
        bootstrap_servers=kafka_container.get_bootstrap_server()
    )
    partitions = consumer.partitions_for_topic('partitioned-topic')

    assert len(partitions) == 3

    consumer.close()
    admin.close()
```

## Best Practices

### 1. Изоляция тестов

```java
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
class IsolatedKafkaTest {
    // Каждый тест получает чистый контекст
}

// Или использовать уникальные имена топиков
@BeforeEach
void setup() {
    String uniqueTopic = "test-" + UUID.randomUUID();
    // Использовать uniqueTopic в тесте
}
```

### 2. Ожидание сообщений

```java
// Неправильно: Thread.sleep()
Thread.sleep(5000);

// Правильно: использовать утилиты ожидания
import static org.awaitility.Awaitility.*;

await()
    .atMost(Duration.ofSeconds(10))
    .pollInterval(Duration.ofMillis(100))
    .until(() -> messageReceived);

// Или KafkaTestUtils
ConsumerRecords<String, String> records = KafkaTestUtils.getRecords(
    consumer,
    Duration.ofSeconds(10)
);
```

### 3. Очистка ресурсов

```java
@AfterEach
void cleanup() {
    // Удаляем созданные топики
    adminClient.deleteTopics(Arrays.asList("test-topic")).all().get();

    // Закрываем consumer'ы
    if (consumer != null) {
        consumer.close();
    }
}
```

### 4. Конфигурация для быстрых тестов

```java
@EmbeddedKafka(
    brokerProperties = {
        "offsets.topic.replication.factor=1",
        "transaction.state.log.replication.factor=1",
        "transaction.state.log.min.isr=1",
        "min.insync.replicas=1",
        "default.replication.factor=1"
    }
)
```

### 5. Тестирование ошибок

```java
@Test
void shouldHandleDeserializationError() {
    // Отправляем "битое" сообщение
    kafkaTemplate.send("test-topic", "invalid-json");

    // Проверяем обработку ошибки
    await().atMost(Duration.ofSeconds(5))
        .until(() -> errorHandler.getErrorCount() > 0);
}
```

### 6. Профилирование тестов

```java
@Test
@Timeout(value = 30, unit = TimeUnit.SECONDS)
void shouldCompleteWithinTimeout() {
    // Тест с таймаутом
}
```

## Типичные ошибки

1. **Не дождались записи сообщений** - используйте `flush()` и `get()`
2. **Конфликт портов** - используйте динамические порты
3. **Утечка ресурсов** - закрывайте consumer'ы и producer'ы
4. **Зависимость от порядка тестов** - изолируйте тесты
5. **Слишком большой timeout** - замедляет CI/CD

## Связанные темы

- [Топики](./01-topics.md)
- [Партиции](./02-partitions.md)
- [Сжатые топики](./04-compacted-topics.md)
