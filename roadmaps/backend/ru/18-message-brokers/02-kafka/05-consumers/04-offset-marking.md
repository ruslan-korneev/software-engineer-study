# Маркировка местонахождения (Offsets)

[prev: 03-tracing](./03-tracing.md) | [next: 05-compacted-topic-reading](./05-compacted-topic-reading.md)

---

## Описание

Offset (смещение) — это уникальный идентификатор позиции сообщения в партиции Kafka. Каждое сообщение в партиции имеет последовательный номер, начиная с 0. Consumer использует offset для отслеживания прогресса чтения — какие сообщения уже обработаны, а какие ещё предстоит прочитать. Правильное управление offset'ами критически важно для обеспечения гарантий доставки сообщений: at-least-once, at-most-once или exactly-once.

## Ключевые концепции

### Типы Offset

```
Партиция:
┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ 0  │ 1  │ 2  │ 3  │ 4  │ 5  │ 6  │ 7  │ 8  │ 9  │
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
  ▲                        ▲                   ▲
  │                        │                   │
Log Start               Committed           Log End
Offset                  Offset              Offset
(earliest)              (position)          (latest)
```

| Тип Offset | Описание |
|------------|----------|
| **Log Start Offset** | Самый старый offset в партиции (earliest) |
| **Log End Offset (LEO)** | Следующий offset для записи (latest) |
| **Committed Offset** | Последний закоммиченный offset consumer'а |
| **Current Position** | Текущая позиция чтения consumer'а |
| **Consumer Lag** | LEO - Committed Offset |

### Хранение Offset

Offset'ы хранятся в специальном внутреннем топике Kafka:
- **Топик**: `__consumer_offsets`
- **Ключ**: `(group.id, topic, partition)`
- **Значение**: `offset, metadata, timestamp`
- **Retention**: По умолчанию 7 дней

### Стратегии коммита

| Стратегия | Гарантия | Использование |
|-----------|----------|---------------|
| **Auto Commit** | At-least-once (примерно) | Простые случаи |
| **Sync Commit** | At-least-once | Надёжная обработка |
| **Async Commit** | At-least-once | Производительность |
| **Transactional** | Exactly-once | Критичные данные |

## Примеры кода

### Автоматический коммит (Auto Commit)

```java
import org.apache.kafka.clients.consumer.*;
import java.time.Duration;
import java.util.Collections;
import java.util.Properties;

public class AutoCommitExample {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "auto-commit-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");

        // Включение автоматического коммита
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);
        props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, 5000); // 5 секунд

        // Поведение при отсутствии сохранённого offset
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        // earliest - читать с начала
        // latest - читать только новые
        // none - выбросить исключение

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(Collections.singletonList("my-topic"));

            while (true) {
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("Offset: %d, Value: %s%n",
                            record.offset(), record.value());
                    processRecord(record);
                }
                // Коммит происходит автоматически каждые 5 секунд
            }
        }
    }

    private static void processRecord(ConsumerRecord<String, String> record) {
        // Обработка
    }
}
```

### Синхронный коммит (Sync Commit)

```java
import org.apache.kafka.clients.consumer.*;
import java.time.Duration;
import java.util.Collections;
import java.util.Properties;

public class SyncCommitExample {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "sync-commit-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");

        // Отключение автоматического коммита
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(Collections.singletonList("my-topic"));

            while (true) {
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    processRecord(record);
                }

                if (!records.isEmpty()) {
                    try {
                        // Синхронный коммит — блокирует до подтверждения
                        consumer.commitSync();
                        System.out.println("Offset committed successfully");
                    } catch (CommitFailedException e) {
                        System.err.println("Commit failed: " + e.getMessage());
                        // Обработка ошибки коммита
                    }
                }
            }
        }
    }

    private static void processRecord(ConsumerRecord<String, String> record) {
        System.out.printf("Processing: partition=%d, offset=%d%n",
                record.partition(), record.offset());
    }
}
```

### Асинхронный коммит (Async Commit)

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;

import java.time.Duration;
import java.util.*;

public class AsyncCommitExample {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "async-commit-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(Collections.singletonList("my-topic"));

            while (true) {
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    processRecord(record);
                }

                if (!records.isEmpty()) {
                    // Асинхронный коммит — не блокирует
                    consumer.commitAsync((offsets, exception) -> {
                        if (exception != null) {
                            System.err.printf("Commit failed for offsets %s: %s%n",
                                    offsets, exception.getMessage());
                        } else {
                            System.out.printf("Committed offsets: %s%n", offsets);
                        }
                    });
                }
            }
        }
    }

    private static void processRecord(ConsumerRecord<String, String> record) {
        // Обработка
    }
}
```

### Коммит конкретных Offset'ов

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;

import java.time.Duration;
import java.util.*;

public class SpecificOffsetCommitExample {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "specific-offset-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(Collections.singletonList("my-topic"));

            int count = 0;
            Map<TopicPartition, OffsetAndMetadata> currentOffsets = new HashMap<>();

            while (true) {
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    processRecord(record);

                    // Запоминаем offset для каждой партиции
                    // ВАЖНО: коммитим offset + 1 (следующий для чтения)
                    currentOffsets.put(
                            new TopicPartition(record.topic(), record.partition()),
                            new OffsetAndMetadata(record.offset() + 1, "processed")
                    );

                    count++;

                    // Коммит каждые 100 записей
                    if (count % 100 == 0) {
                        consumer.commitSync(currentOffsets);
                        System.out.printf("Committed offsets: %s%n", currentOffsets);
                        currentOffsets.clear();
                    }
                }
            }
        }
    }

    private static void processRecord(ConsumerRecord<String, String> record) {
        System.out.printf("Processing: partition=%d, offset=%d%n",
                record.partition(), record.offset());
    }
}
```

### Ручное управление позицией (Seek)

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;

import java.time.Duration;
import java.util.*;

public class SeekExample {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "seek-example-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            String topic = "my-topic";

            // Ручное назначение партиций (без group management)
            TopicPartition partition0 = new TopicPartition(topic, 0);
            TopicPartition partition1 = new TopicPartition(topic, 1);
            consumer.assign(Arrays.asList(partition0, partition1));

            // Переход к началу партиции
            consumer.seekToBeginning(Collections.singletonList(partition0));

            // Переход к концу партиции
            consumer.seekToEnd(Collections.singletonList(partition1));

            // Переход к конкретному offset
            consumer.seek(partition0, 100);

            // Получение текущей позиции
            long position0 = consumer.position(partition0);
            long position1 = consumer.position(partition1);
            System.out.printf("Positions: partition0=%d, partition1=%d%n",
                    position0, position1);

            // Получение committed offset
            Map<TopicPartition, OffsetAndMetadata> committed =
                    consumer.committed(new HashSet<>(Arrays.asList(partition0, partition1)));
            System.out.printf("Committed: %s%n", committed);

            // Чтение данных
            while (true) {
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("Partition: %d, Offset: %d, Value: %s%n",
                            record.partition(), record.offset(), record.value());
                }
            }
        }
    }
}
```

### Поиск по времени (Offset by Timestamp)

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;

import java.time.Duration;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.*;

public class OffsetByTimestampExample {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "timestamp-seek-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            String topic = "my-topic";

            // Получаем информацию о партициях
            consumer.subscribe(Collections.singletonList(topic));
            consumer.poll(Duration.ofMillis(0)); // Инициализация назначения

            Set<TopicPartition> assignedPartitions = consumer.assignment();

            // Время: 1 час назад
            long oneHourAgo = Instant.now()
                    .minus(1, ChronoUnit.HOURS)
                    .toEpochMilli();

            // Создаём map timestamp для каждой партиции
            Map<TopicPartition, Long> timestampsToSearch = new HashMap<>();
            for (TopicPartition partition : assignedPartitions) {
                timestampsToSearch.put(partition, oneHourAgo);
            }

            // Поиск offset'ов по времени
            Map<TopicPartition, OffsetAndTimestamp> offsetsForTimes =
                    consumer.offsetsForTimes(timestampsToSearch);

            // Переход к найденным offset'ам
            for (Map.Entry<TopicPartition, OffsetAndTimestamp> entry :
                    offsetsForTimes.entrySet()) {

                TopicPartition partition = entry.getKey();
                OffsetAndTimestamp offsetAndTimestamp = entry.getValue();

                if (offsetAndTimestamp != null) {
                    consumer.seek(partition, offsetAndTimestamp.offset());
                    System.out.printf("Partition %s: seeking to offset %d (timestamp: %d)%n",
                            partition, offsetAndTimestamp.offset(),
                            offsetAndTimestamp.timestamp());
                } else {
                    System.out.printf("Partition %s: no messages after timestamp%n", partition);
                    consumer.seekToEnd(Collections.singletonList(partition));
                }
            }

            // Чтение сообщений с найденной позиции
            while (true) {
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("Timestamp: %d, Offset: %d, Value: %s%n",
                            record.timestamp(), record.offset(), record.value());
                }
            }
        }
    }
}
```

### Комбинированный коммит (Async + Sync)

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.errors.WakeupException;

import java.time.Duration;
import java.util.Collections;
import java.util.Properties;
import java.util.concurrent.atomic.AtomicBoolean;

public class CombinedCommitExample {

    private static final AtomicBoolean running = new AtomicBoolean(true);

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "combined-commit-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);

        // Shutdown hook
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            running.set(false);
            consumer.wakeup();
        }));

        try {
            consumer.subscribe(Collections.singletonList("my-topic"));

            while (running.get()) {
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    processRecord(record);
                }

                if (!records.isEmpty()) {
                    // Асинхронный коммит для производительности
                    consumer.commitAsync((offsets, exception) -> {
                        if (exception != null) {
                            // Логируем, но не прерываем
                            System.err.println("Async commit failed: " + exception.getMessage());
                        }
                    });
                }
            }
        } catch (WakeupException e) {
            if (running.get()) {
                throw e;
            }
        } finally {
            try {
                // Синхронный коммит при закрытии для надёжности
                consumer.commitSync();
                System.out.println("Final commit successful");
            } catch (Exception e) {
                System.err.println("Final commit failed: " + e.getMessage());
            } finally {
                consumer.close();
                System.out.println("Consumer closed");
            }
        }
    }

    private static void processRecord(ConsumerRecord<String, String> record) {
        System.out.printf("Processing: offset=%d%n", record.offset());
    }
}
```

### Offset управление на Python

```python
from confluent_kafka import Consumer, TopicPartition, OFFSET_BEGINNING, OFFSET_END
import time

def create_consumer():
    conf = {
        'bootstrap.servers': 'localhost:9092',
        'group.id': 'python-offset-group',
        'auto.offset.reset': 'earliest',
        'enable.auto.commit': False,
    }
    return Consumer(conf)

def manual_commit_example():
    """Пример ручного коммита"""
    consumer = create_consumer()
    consumer.subscribe(['my-topic'])

    try:
        while True:
            msg = consumer.poll(timeout=1.0)

            if msg is None:
                continue
            if msg.error():
                print(f"Error: {msg.error()}")
                continue

            print(f"Processing: partition={msg.partition()}, offset={msg.offset()}")
            process_message(msg)

            # Синхронный коммит
            consumer.commit(asynchronous=False)
            print(f"Committed offset: {msg.offset() + 1}")

    except KeyboardInterrupt:
        pass
    finally:
        consumer.close()

def seek_example():
    """Пример использования seek"""
    consumer = create_consumer()

    # Ручное назначение партиций
    partitions = [
        TopicPartition('my-topic', 0),
        TopicPartition('my-topic', 1),
    ]
    consumer.assign(partitions)

    # Переход к началу
    consumer.seek(TopicPartition('my-topic', 0), OFFSET_BEGINNING)

    # Переход к конкретному offset
    consumer.seek(TopicPartition('my-topic', 1), 100)

    # Получение текущих позиций
    for tp in partitions:
        position = consumer.position([tp])
        print(f"Position for {tp}: {position}")

    try:
        while True:
            msg = consumer.poll(timeout=1.0)
            if msg and not msg.error():
                print(f"Partition: {msg.partition()}, Offset: {msg.offset()}")
    except KeyboardInterrupt:
        pass
    finally:
        consumer.close()

def offset_by_timestamp_example():
    """Поиск offset по времени"""
    consumer = create_consumer()
    consumer.subscribe(['my-topic'])

    # Нужен poll для инициализации
    consumer.poll(timeout=0)

    assignment = consumer.assignment()
    print(f"Assigned partitions: {assignment}")

    # Время: 1 час назад (в миллисекундах)
    one_hour_ago = int((time.time() - 3600) * 1000)

    # Подготовка запроса
    topic_partitions = [
        TopicPartition(tp.topic, tp.partition, one_hour_ago)
        for tp in assignment
    ]

    # Поиск offset'ов
    offsets = consumer.offsets_for_times(topic_partitions)

    for tp in offsets:
        if tp.offset >= 0:
            print(f"Seeking {tp.topic}-{tp.partition} to offset {tp.offset}")
            consumer.seek(tp)
        else:
            print(f"No offset found for {tp.topic}-{tp.partition}")

    try:
        while True:
            msg = consumer.poll(timeout=1.0)
            if msg and not msg.error():
                print(f"Timestamp: {msg.timestamp()}, Offset: {msg.offset()}")
    except KeyboardInterrupt:
        pass
    finally:
        consumer.close()

def process_message(msg):
    """Обработка сообщения"""
    print(f"Value: {msg.value().decode('utf-8')}")

if __name__ == '__main__':
    manual_commit_example()
```

## Best Practices

### 1. Выбор стратегии коммита

```java
// At-most-once: коммит ДО обработки
consumer.commitSync();
processRecord(record); // Если упадёт — сообщение потеряно

// At-least-once: коммит ПОСЛЕ обработки (рекомендуется)
processRecord(record);
consumer.commitSync(); // Если упадёт — сообщение обработается повторно

// Exactly-once: использование транзакций
props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
```

### 2. Обработка ошибки коммита

```java
try {
    consumer.commitSync();
} catch (CommitFailedException e) {
    // Ребалансировка произошла — партиции потеряны
    // Повторный коммит бессмысленен
    log.warn("Commit failed, partition reassigned: {}", e.getMessage());
} catch (RetriableException e) {
    // Можно повторить
    retryCommit(consumer);
}
```

### 3. Мониторинг Consumer Lag

```bash
# Командная строка
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --describe --group my-group

# Вывод:
# TOPIC      PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# my-topic   0          1000            1100            100
# my-topic   1          2000            2000            0
```

### 4. Сохранение offset во внешнее хранилище

```java
// Для exactly-once семантики с внешней БД
try {
    for (ConsumerRecord<String, String> record : records) {
        // Начало транзакции БД
        database.beginTransaction();

        // Обработка и сохранение результата
        database.save(processRecord(record));

        // Сохранение offset в той же транзакции
        database.saveOffset(record.topic(), record.partition(), record.offset() + 1);

        // Коммит транзакции
        database.commit();
    }

    // Коммит Kafka offset не нужен — читаем из БД при старте
} catch (Exception e) {
    database.rollback();
    throw e;
}

// При старте восстанавливаем позицию из БД
long savedOffset = database.getOffset(topic, partition);
consumer.seek(new TopicPartition(topic, partition), savedOffset);
```

### 5. Использование metadata в коммите

```java
// Сохранение дополнительной информации
Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
offsets.put(
    new TopicPartition("my-topic", 0),
    new OffsetAndMetadata(
        record.offset() + 1,
        "processed_at=" + Instant.now().toString() + ",batch_id=" + batchId
    )
);
consumer.commitSync(offsets);

// Чтение metadata
Map<TopicPartition, OffsetAndMetadata> committed = consumer.committed(offsets.keySet());
String metadata = committed.get(partition).metadata();
```

## Распространённые ошибки

### 1. Коммит неправильного offset

```java
// НЕПРАВИЛЬНО — коммит текущего offset
consumer.commitSync(Collections.singletonMap(
    new TopicPartition(record.topic(), record.partition()),
    new OffsetAndMetadata(record.offset()) // Этот offset уже обработан!
));

// ПРАВИЛЬНО — коммит следующего offset
consumer.commitSync(Collections.singletonMap(
    new TopicPartition(record.topic(), record.partition()),
    new OffsetAndMetadata(record.offset() + 1) // Следующий для чтения
));
```

### 2. Коммит до обработки

```java
// НЕПРАВИЛЬНО — at-most-once семантика
for (ConsumerRecord<String, String> record : records) {
    consumer.commitSync(); // Коммит ДО обработки
    processRecord(record); // Если упадёт — данные потеряны
}

// ПРАВИЛЬНО — at-least-once семантика
for (ConsumerRecord<String, String> record : records) {
    processRecord(record);
}
consumer.commitSync(); // Коммит ПОСЛЕ обработки
```

### 3. Игнорирование исключений при коммите

```java
// НЕПРАВИЛЬНО
consumer.commitAsync(); // Ошибки игнорируются

// ПРАВИЛЬНО
consumer.commitAsync((offsets, exception) -> {
    if (exception != null) {
        log.error("Commit failed for offsets {}: {}",
                offsets, exception.getMessage());
        // Можно попробовать sync commit или записать для retry
    }
});
```

### 4. Auto commit с долгой обработкой

```java
// НЕПРАВИЛЬНО
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);
props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, 5000);

for (ConsumerRecord<String, String> record : records) {
    Thread.sleep(10000); // Долгая обработка
    // Auto commit может произойти до завершения обработки!
}

// ПРАВИЛЬНО — отключить auto commit при долгой обработке
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
```

### 5. Коммит в неправильном месте при ребалансировке

```java
// НЕПРАВИЛЬНО — коммит после потери партиции
consumer.subscribe(Collections.singletonList("topic"));

// ПРАВИЛЬНО — коммит в onPartitionsRevoked
consumer.subscribe(Collections.singletonList("topic"), new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // Коммит ДО потери партиций
        consumer.commitSync();
    }

    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // Можно восстановить позицию из внешнего хранилища
    }
});
```

## Сравнение стратегий коммита

| Критерий | Auto Commit | Sync Commit | Async Commit | External Store |
|----------|-------------|-------------|--------------|----------------|
| Гарантия | At-least-once* | At-least-once | At-least-once | Exactly-once |
| Производительность | Высокая | Низкая | Высокая | Средняя |
| Сложность | Низкая | Низкая | Средняя | Высокая |
| Контроль | Нет | Полный | Полный | Полный |
| Надёжность | Низкая | Высокая | Средняя | Высокая |

*Auto commit не гарантирует at-least-once при сбоях

## Дополнительные ресурсы

- [Kafka Consumer Offset Management](https://kafka.apache.org/documentation/#consumerconfigs)
- [Confluent Offset Management Guide](https://docs.confluent.io/platform/current/clients/consumer.html#offset-management)
- [Exactly-once Semantics in Kafka](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/)

---

[prev: 03-tracing](./03-tracing.md) | [next: 05-compacted-topic-reading](./05-compacted-topic-reading.md)
