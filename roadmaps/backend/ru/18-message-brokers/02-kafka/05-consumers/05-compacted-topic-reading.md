# Чтение из сжатой темы и Ребалансировка (Rebalancing)

[prev: 04-offset-marking](./04-offset-marking.md) | [next: 06-factory-requirements](./06-factory-requirements.md)

---

## Описание

Ребалансировка (Rebalancing) — это процесс перераспределения партиций между consumer'ами в группе. Она происходит при добавлении/удалении consumer'ов, изменении подписки на топики или при сбоях. Во время ребалансировки потребление данных приостанавливается, поэтому важно минимизировать её влияние на систему.

Compacted Topics (сжатые топики) — это топики с политикой хранения по ключу, где Kafka сохраняет только последнее значение для каждого ключа. Чтение из таких топиков имеет свои особенности.

## Ключевые концепции

### Причины ребалансировки

```
┌─────────────────────────────────────────────────────────────┐
│                    Триггеры ребалансировки                  │
├─────────────────────────────────────────────────────────────┤
│ 1. Consumer присоединяется к группе                         │
│ 2. Consumer покидает группу (graceful shutdown)             │
│ 3. Consumer "умирает" (пропуск heartbeat)                   │
│ 4. Consumer не вызывает poll() вовремя                      │
│ 5. Изменение подписки (subscribe с новым списком топиков)   │
│ 6. Добавление партиций в топик                              │
│ 7. Координатор группы обнаруживает новые метаданные         │
└─────────────────────────────────────────────────────────────┘
```

### Протокол ребалансировки

```
                 Consumer 1                Coordinator              Consumer 2
                     │                          │                       │
                     │      JoinGroup           │                       │
                     │─────────────────────────>│                       │
                     │                          │      JoinGroup        │
                     │                          │<──────────────────────│
                     │                          │                       │
                     │   JoinGroup Response     │   JoinGroup Response  │
                     │   (Leader assignment)    │   (Follower)          │
                     │<─────────────────────────│──────────────────────>│
                     │                          │                       │
                     │      SyncGroup           │      SyncGroup        │
                     │  (with assignments)      │   (empty)             │
                     │─────────────────────────>│<──────────────────────│
                     │                          │                       │
                     │   SyncGroup Response     │   SyncGroup Response  │
                     │   (partition list)       │   (partition list)    │
                     │<─────────────────────────│──────────────────────>│
```

### Типы ребалансировки

| Тип | Описание | Производительность |
|-----|----------|-------------------|
| **Eager (Stop-the-World)** | Все партиции отзываются, затем переназначаются | Низкая |
| **Cooperative (Incremental)** | Только затронутые партиции переназначаются | Высокая |

### Роли участников

| Роль | Описание |
|------|----------|
| **Group Coordinator** | Брокер, управляющий протоколом группы |
| **Group Leader** | Consumer, вычисляющий назначение партиций |
| **Group Member** | Обычный участник группы |

## Примеры кода

### Обработка ребалансировки с ConsumerRebalanceListener

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;

import java.time.Duration;
import java.util.*;

public class RebalanceListenerExample {

    private KafkaConsumer<String, String> consumer;
    private Map<TopicPartition, Long> currentOffsets = new HashMap<>();

    public void run() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "rebalance-example-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        consumer = new KafkaConsumer<>(props);

        // Подписка с обработчиком ребалансировки
        consumer.subscribe(Collections.singletonList("my-topic"), new ConsumerRebalanceListener() {

            @Override
            public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
                // Вызывается ПЕРЕД ребалансировкой
                // Здесь нужно закоммитить текущие offset'ы
                System.out.println("Partitions revoked: " + partitions);

                if (!currentOffsets.isEmpty()) {
                    Map<TopicPartition, OffsetAndMetadata> offsetsToCommit = new HashMap<>();
                    for (Map.Entry<TopicPartition, Long> entry : currentOffsets.entrySet()) {
                        offsetsToCommit.put(entry.getKey(),
                                new OffsetAndMetadata(entry.getValue() + 1));
                    }

                    try {
                        consumer.commitSync(offsetsToCommit);
                        System.out.println("Committed offsets before rebalance: " + offsetsToCommit);
                    } catch (Exception e) {
                        System.err.println("Failed to commit during rebalance: " + e.getMessage());
                    }

                    currentOffsets.clear();
                }

                // Освобождение ресурсов, связанных с партициями
                cleanupPartitionResources(partitions);
            }

            @Override
            public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
                // Вызывается ПОСЛЕ ребалансировки
                // Здесь можно инициализировать ресурсы для новых партиций
                System.out.println("Partitions assigned: " + partitions);

                // Опционально: восстановить offset из внешнего хранилища
                for (TopicPartition partition : partitions) {
                    Long savedOffset = loadOffsetFromDatabase(partition);
                    if (savedOffset != null) {
                        consumer.seek(partition, savedOffset);
                        System.out.println("Seeking to saved offset: " + savedOffset);
                    }
                }

                initializePartitionResources(partitions);
            }
        });

        try {
            while (true) {
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    processRecord(record);

                    // Отслеживаем текущий offset
                    currentOffsets.put(
                            new TopicPartition(record.topic(), record.partition()),
                            record.offset()
                    );
                }

                // Периодический коммит
                if (!currentOffsets.isEmpty()) {
                    consumer.commitAsync();
                }
            }
        } finally {
            consumer.close();
        }
    }

    private void processRecord(ConsumerRecord<String, String> record) {
        System.out.printf("Processing: partition=%d, offset=%d%n",
                record.partition(), record.offset());
    }

    private void cleanupPartitionResources(Collection<TopicPartition> partitions) {
        // Закрытие соединений, файлов и т.д.
        System.out.println("Cleaning up resources for: " + partitions);
    }

    private void initializePartitionResources(Collection<TopicPartition> partitions) {
        // Открытие соединений, инициализация состояния
        System.out.println("Initializing resources for: " + partitions);
    }

    private Long loadOffsetFromDatabase(TopicPartition partition) {
        // Загрузка сохранённого offset из внешнего хранилища
        return null; // Вернуть сохранённый offset или null
    }
}
```

### Cooperative Rebalancing (инкрементальная ребалансировка)

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;

import java.time.Duration;
import java.util.*;

public class CooperativeRebalanceExample {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "cooperative-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");

        // Включение Cooperative ребалансировки
        props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
                  CooperativeStickyAssignor.class.getName());

        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {

            consumer.subscribe(Collections.singletonList("my-topic"),
                    new ConsumerRebalanceListener() {

                @Override
                public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
                    // При Cooperative: только отзываемые партиции
                    // Остальные продолжают обрабатываться!
                    System.out.println("Partitions being revoked: " + partitions);

                    // Коммит только для отзываемых партиций
                    if (!partitions.isEmpty()) {
                        Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
                        for (TopicPartition partition : partitions) {
                            long position = consumer.position(partition);
                            offsets.put(partition, new OffsetAndMetadata(position));
                        }
                        consumer.commitSync(offsets);
                    }
                }

                @Override
                public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
                    // Новые партиции добавлены
                    System.out.println("Partitions assigned: " + partitions);
                }

                @Override
                public void onPartitionsLost(Collection<TopicPartition> partitions) {
                    // Партиции потеряны без вызова onPartitionsRevoked
                    // (например, при таймауте сессии)
                    System.out.println("Partitions lost: " + partitions);
                    // НЕ пытаемся коммитить — партиции уже переназначены
                }
            });

            while (true) {
                // Poll продолжает работать во время Cooperative ребалансировки
                // для партиций, которые не затронуты
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("Partition %d, Offset %d: %s%n",
                            record.partition(), record.offset(), record.value());
                }

                if (!records.isEmpty()) {
                    consumer.commitSync();
                }
            }
        }
    }
}
```

### Static Membership для предотвращения ребалансировки

```java
import org.apache.kafka.clients.consumer.*;

import java.time.Duration;
import java.util.*;

public class StaticMembershipExample {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "static-membership-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");

        // Static Membership: уникальный идентификатор для этого consumer
        // При перезапуске с тем же ID — ребалансировка не происходит
        String instanceId = System.getenv("HOSTNAME"); // или "consumer-1"
        props.put(ConsumerConfig.GROUP_INSTANCE_ID_CONFIG, instanceId);

        // Увеличенный session timeout для static members
        // Consumer может отсутствовать до session.timeout без ребалансировки
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 300000); // 5 минут

        // Heartbeat interval
        props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 30000); // 30 секунд

        System.out.printf("Starting consumer with instance.id: %s%n", instanceId);

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(Collections.singletonList("my-topic"));

            while (true) {
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("[%s] Partition %d: %s%n",
                            instanceId, record.partition(), record.value());
                }
            }
        }
    }
}
```

### Чтение из Compacted Topic

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;

import java.time.Duration;
import java.util.*;

public class CompactedTopicReader {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "compacted-reader");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");

        // Для compacted topics важно читать с начала при первом запуске
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        // Хранилище текущего состояния (ключ -> последнее значение)
        Map<String, String> currentState = new HashMap<>();

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {

            // Compacted topic для хранения конфигураций
            String compactedTopic = "user-preferences";
            consumer.subscribe(Collections.singletonList(compactedTopic));

            // Первоначальная загрузка состояния
            System.out.println("Loading initial state from compacted topic...");
            boolean initialLoadComplete = false;
            Map<TopicPartition, Long> endOffsets = null;

            while (!initialLoadComplete) {
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofMillis(1000));

                if (endOffsets == null) {
                    endOffsets = consumer.endOffsets(consumer.assignment());
                }

                for (ConsumerRecord<String, String> record : records) {
                    if (record.value() == null) {
                        // Tombstone — удаление ключа
                        currentState.remove(record.key());
                        System.out.printf("Deleted key: %s%n", record.key());
                    } else {
                        // Обновление значения
                        currentState.put(record.key(), record.value());
                        System.out.printf("Updated: %s = %s%n",
                                record.key(), record.value());
                    }
                }

                // Проверка достижения конца всех партиций
                initialLoadComplete = true;
                for (TopicPartition partition : consumer.assignment()) {
                    long position = consumer.position(partition);
                    long endOffset = endOffsets.getOrDefault(partition, 0L);
                    if (position < endOffset) {
                        initialLoadComplete = false;
                        break;
                    }
                }
            }

            System.out.println("Initial state loaded: " + currentState);
            consumer.commitSync();

            // Продолжаем слушать обновления
            System.out.println("Listening for updates...");
            while (true) {
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    if (record.value() == null) {
                        currentState.remove(record.key());
                        System.out.printf("Deleted: %s%n", record.key());
                    } else {
                        String oldValue = currentState.put(record.key(), record.value());
                        System.out.printf("Updated: %s = %s (was: %s)%n",
                                record.key(), record.value(), oldValue);
                    }
                }

                if (!records.isEmpty()) {
                    consumer.commitSync();
                }
            }
        }
    }
}
```

### Ребалансировка на Python

```python
from confluent_kafka import Consumer, TopicPartition
import threading

class RebalanceHandler:
    def __init__(self, consumer):
        self.consumer = consumer
        self.current_offsets = {}

    def on_assign(self, consumer, partitions):
        """Вызывается после назначения партиций"""
        print(f"Partitions assigned: {[p.partition for p in partitions]}")

        # Опционально: восстановление offset из внешнего хранилища
        for partition in partitions:
            saved_offset = self.load_offset(partition)
            if saved_offset is not None:
                partition.offset = saved_offset

        # Установка позиций
        consumer.assign(partitions)

    def on_revoke(self, consumer, partitions):
        """Вызывается перед отзывом партиций"""
        print(f"Partitions revoked: {[p.partition for p in partitions]}")

        # Коммит текущих offset'ов
        if self.current_offsets:
            try:
                consumer.commit(asynchronous=False)
                print("Committed offsets before rebalance")
            except Exception as e:
                print(f"Failed to commit: {e}")

        self.current_offsets.clear()

    def on_lost(self, consumer, partitions):
        """Вызывается при потере партиций (Cooperative)"""
        print(f"Partitions lost: {[p.partition for p in partitions]}")
        # Не пытаемся коммитить — партиции уже переназначены

    def load_offset(self, partition):
        """Загрузка offset из внешнего хранилища"""
        return None

def run_consumer_with_rebalance():
    conf = {
        'bootstrap.servers': 'localhost:9092',
        'group.id': 'python-rebalance-group',
        'auto.offset.reset': 'earliest',
        'enable.auto.commit': False,
        # Cooperative rebalancing
        'partition.assignment.strategy': 'cooperative-sticky',
    }

    consumer = Consumer(conf)
    handler = RebalanceHandler(consumer)

    # Подписка с обработчиками ребалансировки
    consumer.subscribe(
        ['my-topic'],
        on_assign=handler.on_assign,
        on_revoke=handler.on_revoke,
        on_lost=handler.on_lost
    )

    try:
        while True:
            msg = consumer.poll(timeout=1.0)

            if msg is None:
                continue
            if msg.error():
                print(f"Error: {msg.error()}")
                continue

            print(f"Partition {msg.partition()}, Offset {msg.offset()}: {msg.value()}")

            # Отслеживание offset
            handler.current_offsets[msg.partition()] = msg.offset()

            # Периодический коммит
            consumer.commit(asynchronous=True)

    except KeyboardInterrupt:
        pass
    finally:
        # Финальный коммит
        consumer.commit(asynchronous=False)
        consumer.close()

if __name__ == '__main__':
    run_consumer_with_rebalance()
```

## Best Practices

### 1. Минимизация времени ребалансировки

```java
// Используйте Cooperative Sticky Assignor
props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
          CooperativeStickyAssignor.class.getName());

// Или несколько стратегий с приоритетом
props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
          "org.apache.kafka.clients.consumer.CooperativeStickyAssignor," +
          "org.apache.kafka.clients.consumer.RangeAssignor");
```

### 2. Static Membership в Kubernetes

```yaml
# kubernetes deployment
spec:
  template:
    spec:
      containers:
        - name: kafka-consumer
          env:
            - name: KAFKA_GROUP_INSTANCE_ID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name  # pod name
```

```java
// В Java коде
String instanceId = System.getenv("KAFKA_GROUP_INSTANCE_ID");
props.put(ConsumerConfig.GROUP_INSTANCE_ID_CONFIG, instanceId);
props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 300000);
```

### 3. Правильная обработка onPartitionsLost

```java
consumer.subscribe(topics, new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // Graceful — можно коммитить
        commitCurrentOffsets(partitions);
    }

    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // Инициализация
    }

    @Override
    public void onPartitionsLost(Collection<TopicPartition> partitions) {
        // НЕ graceful — партиции уже переназначены!
        // НЕ коммитим, только очищаем локальное состояние
        cleanupLocalState(partitions);
    }
});
```

### 4. Мониторинг ребалансировок

```java
// Метрики для мониторинга
// rebalance-total: общее количество ребалансировок
// rebalance-latency-avg: среднее время ребалансировки
// last-rebalance-seconds-ago: время с последней ребалансировки

Map<MetricName, ? extends Metric> metrics = consumer.metrics();
for (Map.Entry<MetricName, ? extends Metric> entry : metrics.entrySet()) {
    if (entry.getKey().name().contains("rebalance")) {
        System.out.printf("%s: %s%n",
                entry.getKey().name(), entry.getValue().metricValue());
    }
}
```

### 5. Чтение Compacted Topic для восстановления состояния

```java
// Паттерн: чтение до конца для построения полного состояния
public Map<String, String> loadStateFromCompactedTopic(String topic) {
    Properties props = createConsumerProps();
    props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

    Map<String, String> state = new HashMap<>();

    try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
        // Назначаем все партиции вручную
        List<TopicPartition> partitions = consumer.partitionsFor(topic)
                .stream()
                .map(info -> new TopicPartition(topic, info.partition()))
                .collect(Collectors.toList());

        consumer.assign(partitions);
        consumer.seekToBeginning(partitions);

        Map<TopicPartition, Long> endOffsets = consumer.endOffsets(partitions);

        while (true) {
            ConsumerRecords<String, String> records =
                    consumer.poll(Duration.ofMillis(1000));

            for (ConsumerRecord<String, String> record : records) {
                if (record.value() == null) {
                    state.remove(record.key()); // tombstone
                } else {
                    state.put(record.key(), record.value());
                }
            }

            // Проверка достижения конца
            boolean done = true;
            for (TopicPartition partition : partitions) {
                if (consumer.position(partition) < endOffsets.get(partition)) {
                    done = false;
                    break;
                }
            }
            if (done) break;
        }
    }

    return state;
}
```

## Распространённые ошибки

### 1. Слишком долгая обработка в onPartitionsRevoked

```java
// НЕПРАВИЛЬНО — задержка ребалансировки
@Override
public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
    saveToSlowDatabase(currentState); // Может занять минуты!
    consumer.commitSync();
}

// ПРАВИЛЬНО — быстрая обработка
@Override
public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
    consumer.commitSync(); // Быстро
    // Сохранение в БД — асинхронно или в основном потоке
}
```

### 2. Игнорирование Cooperative протокола

```java
// НЕПРАВИЛЬНО — потеря данных при Cooperative
@Override
public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
    // При Cooperative revoked содержит только отзываемые партиции!
    // Остальные продолжают обрабатываться
    consumer.commitSync(); // Коммит ВСЕХ партиций — неправильно
}

// ПРАВИЛЬНО
@Override
public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
    // Коммит только отзываемых партиций
    Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
    for (TopicPartition partition : partitions) {
        offsets.put(partition, new OffsetAndMetadata(consumer.position(partition)));
    }
    consumer.commitSync(offsets);
}
```

### 3. Использование assign() с subscribe()

```java
// НЕПРАВИЛЬНО — IllegalStateException
consumer.subscribe(Collections.singletonList("topic"));
consumer.assign(Arrays.asList(new TopicPartition("topic", 0)));

// ПРАВИЛЬНО — использовать только один способ
// Вариант 1: subscribe (с group management и ребалансировкой)
consumer.subscribe(Collections.singletonList("topic"));

// Вариант 2: assign (без group management, без ребалансировки)
consumer.assign(Arrays.asList(new TopicPartition("topic", 0)));
```

### 4. Неправильная работа с tombstone в compacted topics

```java
// НЕПРАВИЛЬНО — игнорирование null значений
for (ConsumerRecord<String, String> record : records) {
    state.put(record.key(), record.value()); // NPE при tombstone!
}

// ПРАВИЛЬНО — обработка tombstone
for (ConsumerRecord<String, String> record : records) {
    if (record.value() == null) {
        state.remove(record.key()); // Удаление
    } else {
        state.put(record.key(), record.value());
    }
}
```

### 5. Неполное чтение compacted topic

```java
// НЕПРАВИЛЬНО — чтение с latest пропустит историю
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "latest");

// ПРАВИЛЬНО — чтение с earliest для полного состояния
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
```

## Сравнение стратегий назначения партиций

| Стратегия | Ребалансировка | Производительность | Использование |
|-----------|----------------|-------------------|---------------|
| **RangeAssignor** | Eager | Средняя | По умолчанию (старый) |
| **RoundRobinAssignor** | Eager | Хорошая | Равномерное распределение |
| **StickyAssignor** | Eager | Хорошая | Минимизация перемещений |
| **CooperativeStickyAssignor** | Cooperative | Отличная | Рекомендуется |

## Дополнительные ресурсы

- [Kafka Rebalancing Protocol](https://kafka.apache.org/documentation/#consumerconfigs)
- [Cooperative Rebalancing KIP-429](https://cwiki.apache.org/confluence/display/KAFKA/KIP-429)
- [Compacted Topics Documentation](https://kafka.apache.org/documentation/#compaction)
- [Static Membership KIP-345](https://cwiki.apache.org/confluence/display/KAFKA/KIP-345)

---

[prev: 04-offset-marking](./04-offset-marking.md) | [next: 06-factory-requirements](./06-factory-requirements.md)
