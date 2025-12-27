# Разделы (Partitions)

## Описание

**Partition (партиция/раздел)** — это базовая единица параллелизма в Apache Kafka. Каждый топик разделен на одну или несколько партиций, которые представляют собой упорядоченные, неизменяемые последовательности записей (лог).

Партиции позволяют:
- **Горизонтально масштабировать** хранение и обработку данных
- **Параллельно обрабатывать** сообщения несколькими консьюмерами
- **Распределять нагрузку** между брокерами кластера
- **Гарантировать порядок** сообщений в пределах одной партиции

### Архитектура партиции

```
Topic: orders (3 partitions)

Partition 0:          [0] [1] [2] [3] [4] [5] [6] [7] → Writes
                       ↑                           ↑
                    oldest                      newest
                    offset                      offset

Partition 1:          [0] [1] [2] [3] [4] [5] → Writes

Partition 2:          [0] [1] [2] [3] [4] [5] [6] [7] [8] [9] → Writes
```

## Ключевые концепции

### Offset (смещение)

**Offset** — уникальный последовательный идентификатор записи в партиции. Это 64-битное число, которое монотонно возрастает для каждой новой записи.

```
Partition 0:
Offset:    0     1     2     3     4     5
          ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐
Data:     │ A │ │ B │ │ C │ │ D │ │ E │ │ F │
          └───┘ └───┘ └───┘ └───┘ └───┘ └───┘
                              ↑
                    Consumer position (offset 3)
```

### Структура хранения на диске

```
/kafka-logs/
├── orders-0/                    # Партиция 0 топика orders
│   ├── 00000000000000000000.log # Сегмент лога
│   ├── 00000000000000000000.index
│   ├── 00000000000000000000.timeindex
│   ├── 00000000000012345678.log # Следующий сегмент
│   ├── 00000000000012345678.index
│   └── 00000000000012345678.timeindex
├── orders-1/                    # Партиция 1
│   └── ...
└── orders-2/                    # Партиция 2
    └── ...
```

### Leader и Replicas

Каждая партиция имеет:
- **Leader** — брокер, обрабатывающий все операции чтения/записи
- **Followers** — брокеры, реплицирующие данные для отказоустойчивости
- **ISR (In-Sync Replicas)** — реплики, синхронизированные с лидером

```
Topic: orders, Partition: 0

Broker 1 (Leader):    [0] [1] [2] [3] [4] [5]
                                            ↓ Replicate
Broker 2 (Follower):  [0] [1] [2] [3] [4] [5]  ✓ In ISR
                                            ↓ Replicate
Broker 3 (Follower):  [0] [1] [2] [3] [4]      ✗ Out of ISR (отстает)
```

### High Watermark и Log End Offset

```
Partition 0:
                     High Watermark        Log End Offset
                          ↓                      ↓
Offset:    0  1  2  3  4  5  6  7  8  9  10 11 12
          ├──┴──┴──┴──┴──┴──┤──┴──┴──┴──┴──┴──┤
          │   Committed     │   Uncommitted    │
          │   (safe to read)│   (not replicated)

- High Watermark: последний offset, реплицированный на все ISR
- Log End Offset: последний записанный offset на лидере
```

## Примеры

### Просмотр информации о партициях

```bash
# Детальная информация о партициях топика
kafka-topics.sh --describe \
  --bootstrap-server localhost:9092 \
  --topic orders

# Вывод:
# Topic: orders    TopicId: abc123    PartitionCount: 3    ReplicationFactor: 3
# Topic: orders    Partition: 0    Leader: 1    Replicas: 1,2,3    Isr: 1,2,3
# Topic: orders    Partition: 1    Leader: 2    Replicas: 2,3,1    Isr: 2,3,1
# Topic: orders    Partition: 2    Leader: 3    Replicas: 3,1,2    Isr: 3,1,2

# Получение offset'ов для партиций
kafka-run-class.sh kafka.tools.GetOffsetShell \
  --broker-list localhost:9092 \
  --topic orders

# Или через kafka-get-offsets (новый инструмент)
kafka-get-offsets.sh --bootstrap-server localhost:9092 \
  --topic orders
```

### Работа с партициями в Java

```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.*;

import java.util.*;
import java.time.Duration;

public class PartitionExample {

    // Отправка в конкретную партицию
    public void sendToSpecificPartition(String topic, int partition, String message) {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
            "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
            "org.apache.kafka.common.serialization.StringSerializer");

        try (KafkaProducer<String, String> producer = new KafkaProducer<>(props)) {

            // Явное указание партиции
            ProducerRecord<String, String> record = new ProducerRecord<>(
                topic,
                partition,  // Номер партиции
                null,       // Key (может быть null)
                message     // Value
            );

            producer.send(record, (metadata, exception) -> {
                if (exception == null) {
                    System.out.printf("Sent to partition %d, offset %d%n",
                        metadata.partition(), metadata.offset());
                }
            });
        }
    }

    // Чтение из конкретных партиций (assign)
    public void readFromSpecificPartitions(String topic, List<Integer> partitions) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
            "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
            "org.apache.kafka.common.serialization.StringDeserializer");
        // Не указываем group.id для assign

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {

            // Создаем список TopicPartition
            List<TopicPartition> topicPartitions = new ArrayList<>();
            for (int partition : partitions) {
                topicPartitions.add(new TopicPartition(topic, partition));
            }

            // Назначаем партиции (без использования consumer group)
            consumer.assign(topicPartitions);

            // Позиционируем на начало
            consumer.seekToBeginning(topicPartitions);

            while (true) {
                ConsumerRecords<String, String> records =
                    consumer.poll(Duration.ofMillis(100));

                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("Partition: %d, Offset: %d, Value: %s%n",
                        record.partition(), record.offset(), record.value());
                }
            }
        }
    }

    // Получение информации о партициях
    public void getPartitionInfo(String topic) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
            "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
            "org.apache.kafka.common.serialization.StringDeserializer");

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {

            // Получаем информацию о партициях
            List<PartitionInfo> partitions = consumer.partitionsFor(topic);

            for (PartitionInfo info : partitions) {
                System.out.printf("Partition: %d, Leader: %s, Replicas: %s, ISR: %s%n",
                    info.partition(),
                    info.leader(),
                    Arrays.toString(info.replicas()),
                    Arrays.toString(info.inSyncReplicas())
                );
            }

            // Получаем offset'ы
            List<TopicPartition> topicPartitions = partitions.stream()
                .map(p -> new TopicPartition(topic, p.partition()))
                .toList();

            Map<TopicPartition, Long> beginningOffsets =
                consumer.beginningOffsets(topicPartitions);
            Map<TopicPartition, Long> endOffsets =
                consumer.endOffsets(topicPartitions);

            for (TopicPartition tp : topicPartitions) {
                System.out.printf("Partition %d: begin=%d, end=%d, messages=%d%n",
                    tp.partition(),
                    beginningOffsets.get(tp),
                    endOffsets.get(tp),
                    endOffsets.get(tp) - beginningOffsets.get(tp)
                );
            }
        }
    }
}
```

### Работа с партициями в Python

```python
from kafka import KafkaProducer, KafkaConsumer, TopicPartition
from kafka.admin import KafkaAdminClient

def send_to_partition(bootstrap_servers: str,
                      topic: str,
                      partition: int,
                      message: str):
    """Отправка сообщения в конкретную партицию"""

    producer = KafkaProducer(
        bootstrap_servers=bootstrap_servers,
        value_serializer=lambda v: v.encode('utf-8')
    )

    # Явное указание партиции
    future = producer.send(topic, value=message, partition=partition)

    # Ожидание подтверждения
    record_metadata = future.get(timeout=10)

    print(f"Sent to {record_metadata.topic}:{record_metadata.partition} "
          f"at offset {record_metadata.offset}")

    producer.close()


def read_from_partitions(bootstrap_servers: str,
                         topic: str,
                         partitions: list):
    """Чтение из конкретных партиций"""

    consumer = KafkaConsumer(
        bootstrap_servers=bootstrap_servers,
        value_deserializer=lambda v: v.decode('utf-8'),
        auto_offset_reset='earliest'
    )

    # Назначаем конкретные партиции
    topic_partitions = [TopicPartition(topic, p) for p in partitions]
    consumer.assign(topic_partitions)

    # Позиционируем на начало
    consumer.seek_to_beginning(*topic_partitions)

    try:
        for message in consumer:
            print(f"Partition: {message.partition}, "
                  f"Offset: {message.offset}, "
                  f"Value: {message.value}")
    finally:
        consumer.close()


def get_partition_info(bootstrap_servers: str, topic: str):
    """Получение информации о партициях"""

    consumer = KafkaConsumer(bootstrap_servers=bootstrap_servers)

    # Информация о партициях
    partitions = consumer.partitions_for_topic(topic)
    print(f"Topic {topic} has {len(partitions)} partitions: {partitions}")

    # Offset'ы для каждой партиции
    topic_partitions = [TopicPartition(topic, p) for p in partitions]

    beginning = consumer.beginning_offsets(topic_partitions)
    end = consumer.end_offsets(topic_partitions)

    for tp in topic_partitions:
        messages = end[tp] - beginning[tp]
        print(f"Partition {tp.partition}: "
              f"begin={beginning[tp]}, end={end[tp]}, messages={messages}")

    consumer.close()


def seek_to_offset(bootstrap_servers: str,
                   topic: str,
                   partition: int,
                   offset: int):
    """Позиционирование на конкретный offset"""

    consumer = KafkaConsumer(
        bootstrap_servers=bootstrap_servers,
        value_deserializer=lambda v: v.decode('utf-8')
    )

    tp = TopicPartition(topic, partition)
    consumer.assign([tp])

    # Позиционируемся на нужный offset
    consumer.seek(tp, offset)

    # Читаем 10 сообщений
    count = 0
    for message in consumer:
        print(f"Offset {message.offset}: {message.value}")
        count += 1
        if count >= 10:
            break

    consumer.close()


# Использование
if __name__ == "__main__":
    servers = "localhost:9092"

    get_partition_info(servers, "orders")
    send_to_partition(servers, "orders", 0, "Message for partition 0")
    read_from_partitions(servers, "orders", [0, 1])
```

### Параллельная обработка партиций

```java
import java.util.concurrent.*;

public class ParallelPartitionProcessor {

    private final String bootstrapServers;
    private final String topic;
    private final int numPartitions;
    private final ExecutorService executor;

    public ParallelPartitionProcessor(String bootstrapServers,
                                       String topic,
                                       int numPartitions) {
        this.bootstrapServers = bootstrapServers;
        this.topic = topic;
        this.numPartitions = numPartitions;
        this.executor = Executors.newFixedThreadPool(numPartitions);
    }

    public void startProcessing() {
        // Запускаем по одному потоку на партицию
        for (int partition = 0; partition < numPartitions; partition++) {
            final int p = partition;
            executor.submit(() -> processPartition(p));
        }
    }

    private void processPartition(int partition) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
            "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
            "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        TopicPartition tp = new TopicPartition(topic, partition);
        consumer.assign(Collections.singletonList(tp));

        try {
            while (!Thread.currentThread().isInterrupted()) {
                ConsumerRecords<String, String> records =
                    consumer.poll(Duration.ofMillis(100));

                for (ConsumerRecord<String, String> record : records) {
                    // Обработка сообщения
                    processMessage(record);
                }

                consumer.commitSync();
            }
        } finally {
            consumer.close();
        }
    }

    private void processMessage(ConsumerRecord<String, String> record) {
        System.out.printf("[Thread-%s] Partition: %d, Offset: %d, Value: %s%n",
            Thread.currentThread().getName(),
            record.partition(),
            record.offset(),
            record.value()
        );
    }

    public void shutdown() {
        executor.shutdownNow();
    }
}
```

## Best Practices

### 1. Выбор количества партиций

```
Рекомендации:
- Минимум: количество консьюмеров в группе
- Максимум: не более 10,000 партиций на брокер
- Оптимально: 2-3x от ожидаемого числа консьюмеров

Примеры:
- Низкая нагрузка (< 10 MB/s): 3-6 партиций
- Средняя нагрузка (10-100 MB/s): 6-12 партиций
- Высокая нагрузка (> 100 MB/s): 12-50+ партиций
```

### 2. Распределение партиций по брокерам

```bash
# Проверка распределения лидеров
kafka-topics.sh --describe \
  --bootstrap-server localhost:9092 \
  --topic orders | grep Leader

# Ручное переназначение партиций (при необходимости)
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --execute
```

```json
// reassignment.json
{
  "version": 1,
  "partitions": [
    {"topic": "orders", "partition": 0, "replicas": [1, 2, 3]},
    {"topic": "orders", "partition": 1, "replicas": [2, 3, 1]},
    {"topic": "orders", "partition": 2, "replicas": [3, 1, 2]}
  ]
}
```

### 3. Мониторинг партиций

```java
// Проверка отставания партиций
public Map<TopicPartition, Long> getPartitionLag(String groupId, String topic) {
    Properties props = new Properties();
    props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

    try (AdminClient adminClient = AdminClient.create(props)) {

        // Получаем текущие offset'ы consumer group
        ListConsumerGroupOffsetsResult offsetsResult =
            adminClient.listConsumerGroupOffsets(groupId);

        Map<TopicPartition, OffsetAndMetadata> committedOffsets =
            offsetsResult.partitionsToOffsetAndMetadata().get();

        // Получаем end offset'ы
        Map<TopicPartition, Long> endOffsets = new HashMap<>();
        try (KafkaConsumer<String, String> consumer = createConsumer()) {
            List<TopicPartition> partitions = consumer.partitionsFor(topic)
                .stream()
                .map(p -> new TopicPartition(topic, p.partition()))
                .toList();
            endOffsets = consumer.endOffsets(partitions);
        }

        // Вычисляем lag
        Map<TopicPartition, Long> lag = new HashMap<>();
        for (Map.Entry<TopicPartition, Long> entry : endOffsets.entrySet()) {
            TopicPartition tp = entry.getKey();
            long endOffset = entry.getValue();
            long committedOffset = committedOffsets.containsKey(tp)
                ? committedOffsets.get(tp).offset()
                : 0;
            lag.put(tp, endOffset - committedOffset);
        }

        return lag;
    }
}
```

### 4. Гарантия порядка сообщений

```java
// Порядок гарантирован ТОЛЬКО в пределах одной партиции!

// Правильно: все заказы пользователя идут в одну партицию
ProducerRecord<String, String> record = new ProducerRecord<>(
    "orders",
    userId,    // Key определяет партицию
    orderJson  // Value
);

// Неправильно: без ключа порядок не гарантирован
ProducerRecord<String, String> record = new ProducerRecord<>(
    "orders",
    orderJson  // Без ключа - round-robin между партициями
);
```

### 5. Обработка увеличения партиций

```java
// При увеличении партиций ключи могут попасть в другие партиции!

// До увеличения (3 партиции):
// hash("user-123") % 3 = 1

// После увеличения (6 партиций):
// hash("user-123") % 6 = 4  // Другая партиция!

// Решение: использовать custom partitioner с consistent hashing
```

### 6. Настройка сегментов

```properties
# Конфигурация топика для оптимизации сегментов
segment.bytes=1073741824      # 1 GB
segment.ms=604800000          # 7 дней

# Для топиков с высокой нагрузкой
segment.bytes=536870912       # 512 MB - чаще ротация
segment.ms=86400000           # 1 день
```

### 7. Under-replicated партиции

```bash
# Проверка under-replicated партиций
kafka-topics.sh --describe \
  --bootstrap-server localhost:9092 \
  --under-replicated-partitions

# Мониторинг метрики
kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
```

## Типичные проблемы

### 1. Неравномерное распределение данных (Hot Partition)

```
Проблема: одна партиция получает больше данных
Причина: плохой выбор ключа или неравномерное распределение ключей
Решение: использовать составные ключи или custom partitioner
```

### 2. Потеря порядка при масштабировании

```
Проблема: порядок сообщений нарушается после добавления партиций
Причина: изменение hash(key) % numPartitions
Решение: не увеличивать партиции для топиков с ordering-зависимой логикой
```

### 3. Consumer отставание (Lag)

```
Проблема: консьюмер не успевает обрабатывать сообщения
Решение:
- Увеличить количество консьюмеров (≤ количества партиций)
- Оптимизировать обработку сообщений
- Увеличить количество партиций
```

## Связанные темы

- [Топики](./01-topics.md)
- [Тестирование с EmbeddedKafka](./03-embedded-kafka-testing.md)
- [Сжатые топики](./04-compacted-topics.md)
