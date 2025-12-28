# Как взаимодействуют потребители (Consumer Groups)

[prev: 01-example](./01-example.md) | [next: 03-tracing](./03-tracing.md)

---

## Описание

Consumer Group (группа потребителей) — это механизм масштабирования и отказоустойчивости в Apache Kafka, позволяющий нескольким потребителям совместно обрабатывать сообщения из топика. Каждая партиция топика назначается только одному потребителю в группе, что обеспечивает параллельную обработку и исключает дублирование. Группы потребителей — ключевая концепция для построения масштабируемых систем обработки данных.

## Ключевые концепции

### Принцип работы Consumer Group

```
Топик: orders (3 партиции)
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Partition 0 │  │ Partition 1 │  │ Partition 2 │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       ▼                ▼                ▼
┌─────────────────────────────────────────────┐
│           Consumer Group: order-processors  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│  │Consumer 1│ │Consumer 2│ │Consumer 3│    │
│  │  (P0)    │ │  (P1)    │ │  (P2)    │    │
│  └──────────┘ └──────────┘ └──────────┘    │
└─────────────────────────────────────────────┘
```

### Основные правила

1. **Одна партиция — один потребитель**: Каждая партиция назначается только одному consumer в группе
2. **Один потребитель — много партиций**: Consumer может читать из нескольких партиций
3. **Избыточные потребители простаивают**: Если consumers больше, чем партиций, лишние не получают данных
4. **Разные группы независимы**: Разные группы читают одни и те же данные независимо друг от друга

### Роли в Consumer Group

| Компонент | Описание |
|-----------|----------|
| **Group Coordinator** | Брокер Kafka, управляющий группой |
| **Group Leader** | Consumer, определяющий распределение партиций |
| **Group Member** | Любой consumer в группе |

### Идентификация группы

```java
// group.id — уникальный идентификатор группы
props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-processing-group");

// group.instance.id — статический идентификатор для consumer (опционально)
props.put(ConsumerConfig.GROUP_INSTANCE_ID_CONFIG, "consumer-instance-1");
```

## Примеры кода

### Базовый пример Consumer Group

```java
import org.apache.kafka.clients.consumer.*;
import java.time.Duration;
import java.util.Collections;
import java.util.Properties;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ConsumerGroupExample {

    private static final String TOPIC = "orders";
    private static final String GROUP_ID = "order-processors";
    private static final int NUM_CONSUMERS = 3;

    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(NUM_CONSUMERS);

        for (int i = 0; i < NUM_CONSUMERS; i++) {
            final int consumerId = i;
            executor.submit(() -> runConsumer(consumerId));
        }
    }

    private static void runConsumer(int consumerId) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, GROUP_ID);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(Collections.singletonList(TOPIC));

            System.out.printf("Consumer %d запущен в группе %s%n", consumerId, GROUP_ID);

            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("[Consumer %d] Partition: %d, Offset: %d, Value: %s%n",
                            consumerId, record.partition(), record.offset(), record.value());
                }
            }
        }
    }
}
```

### Несколько Consumer Groups для одного топика

```java
public class MultipleConsumerGroupsExample {

    public static void main(String[] args) {
        // Группа 1: Аналитика — читает все сообщения для построения отчётов
        startConsumerGroup("analytics-group", "Analytics");

        // Группа 2: Уведомления — читает все сообщения для отправки уведомлений
        startConsumerGroup("notifications-group", "Notifications");

        // Группа 3: Аудит — читает все сообщения для логирования
        startConsumerGroup("audit-group", "Audit");

        // Каждая группа получит ВСЕ сообщения независимо
    }

    private static void startConsumerGroup(String groupId, String purpose) {
        new Thread(() -> {
            Properties props = createConsumerProps(groupId);

            try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
                consumer.subscribe(Collections.singletonList("orders"));

                System.out.printf("Группа %s (%s) запущена%n", groupId, purpose);

                while (true) {
                    ConsumerRecords<String, String> records =
                            consumer.poll(Duration.ofMillis(1000));

                    for (ConsumerRecord<String, String> record : records) {
                        System.out.printf("[%s] Обработка: %s%n", purpose, record.value());
                    }
                }
            }
        }).start();
    }

    private static Properties createConsumerProps(String groupId) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        return props;
    }
}
```

### Static Membership (статическое членство)

```java
public class StaticMembershipExample {

    public static void main(String[] args) {
        // Static membership предотвращает ребалансировку при кратковременных отключениях

        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "static-group");

        // Статический идентификатор consumer — при перезапуске сохраняет партиции
        props.put(ConsumerConfig.GROUP_INSTANCE_ID_CONFIG, "consumer-pod-1");

        // Увеличенный таймаут сессии для static members
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 60000); // 1 минута

        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(Collections.singletonList("events"));

            // При перезапуске в пределах session.timeout.ms
            // consumer получит те же партиции без ребалансировки
            while (true) {
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofMillis(1000));
                // обработка
            }
        }
    }
}
```

### Мониторинг Consumer Group через AdminClient

```java
import org.apache.kafka.clients.admin.*;
import org.apache.kafka.common.TopicPartition;

import java.util.*;
import java.util.concurrent.ExecutionException;

public class ConsumerGroupMonitoring {

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Properties props = new Properties();
        props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

        try (AdminClient admin = AdminClient.create(props)) {

            // 1. Список всех consumer groups
            ListConsumerGroupsResult groupsResult = admin.listConsumerGroups();
            System.out.println("Consumer Groups:");
            for (ConsumerGroupListing group : groupsResult.all().get()) {
                System.out.printf("  - %s (state: %s)%n",
                        group.groupId(), group.state().orElse(null));
            }

            // 2. Детали конкретной группы
            String groupId = "order-processors";
            DescribeConsumerGroupsResult describeResult =
                    admin.describeConsumerGroups(Collections.singletonList(groupId));

            ConsumerGroupDescription description = describeResult.all().get().get(groupId);
            System.out.printf("%nГруппа: %s%n", groupId);
            System.out.printf("Состояние: %s%n", description.state());
            System.out.printf("Координатор: %s%n", description.coordinator());

            System.out.println("Участники:");
            for (MemberDescription member : description.members()) {
                System.out.printf("  - %s (host: %s)%n",
                        member.consumerId(), member.host());
                System.out.printf("    Партиции: %s%n", member.assignment().topicPartitions());
            }

            // 3. Лаг потребления
            ListConsumerGroupOffsetsResult offsetsResult =
                    admin.listConsumerGroupOffsets(groupId);

            Map<TopicPartition, OffsetAndMetadata> offsets =
                    offsetsResult.partitionsToOffsetAndMetadata().get();

            System.out.println("\nТекущие offset'ы:");
            for (Map.Entry<TopicPartition, OffsetAndMetadata> entry : offsets.entrySet()) {
                System.out.printf("  %s: %d%n",
                        entry.getKey(), entry.getValue().offset());
            }
        }
    }
}
```

### Consumer Group на Python

```python
from confluent_kafka import Consumer, KafkaException
from threading import Thread
import time

def create_consumer(group_id, consumer_id):
    """Создание consumer в указанной группе"""
    conf = {
        'bootstrap.servers': 'localhost:9092',
        'group.id': group_id,
        'auto.offset.reset': 'earliest',
        'enable.auto.commit': True,
    }

    consumer = Consumer(conf)
    consumer.subscribe(['orders'])

    print(f"Consumer {consumer_id} запущен в группе {group_id}")

    try:
        while True:
            msg = consumer.poll(timeout=1.0)

            if msg is None:
                continue
            if msg.error():
                print(f"Ошибка: {msg.error()}")
                continue

            print(f"[Consumer {consumer_id}] "
                  f"Partition: {msg.partition()}, "
                  f"Offset: {msg.offset()}, "
                  f"Value: {msg.value().decode('utf-8')}")

    except KeyboardInterrupt:
        pass
    finally:
        consumer.close()

def run_consumer_group(group_id, num_consumers):
    """Запуск группы потребителей"""
    threads = []

    for i in range(num_consumers):
        thread = Thread(target=create_consumer, args=(group_id, i))
        thread.daemon = True
        thread.start()
        threads.append(thread)

    return threads

if __name__ == '__main__':
    # Запуск группы из 3 потребителей
    threads = run_consumer_group("python-order-processors", 3)

    # Ожидание
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        print("Завершение...")
```

## Best Practices

### 1. Правильное количество потребителей

```
Оптимальное соотношение:
- Consumers <= Partitions (идеально: Consumers == Partitions)
- Избегайте простаивающих consumer'ов

Пример:
- Топик с 6 партициями
- Максимум 6 активных consumer'ов в группе
- 7-й consumer будет простаивать
```

### 2. Выбор стратегии назначения партиций

```java
// Range — последовательное распределение (по умолчанию)
props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
          "org.apache.kafka.clients.consumer.RangeAssignor");

// RoundRobin — равномерное распределение
props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
          "org.apache.kafka.clients.consumer.RoundRobinAssignor");

// Sticky — минимизация перемещений при ребалансировке
props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
          "org.apache.kafka.clients.consumer.StickyAssignor");

// CooperativeSticky — инкрементальная ребалансировка (рекомендуется)
props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
          "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");
```

### 3. Использование Static Membership в Kubernetes

```java
// В Kubernetes используйте имя pod'а как instance.id
String podName = System.getenv("HOSTNAME"); // kubernetes pod name
props.put(ConsumerConfig.GROUP_INSTANCE_ID_CONFIG, podName);

// Увеличьте session.timeout для плановых перезапусков
props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 300000); // 5 минут
```

### 4. Мониторинг Consumer Lag

```bash
# Команда для проверки лага
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --describe --group order-processors

# Пример вывода:
# GROUP           TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# order-processors orders   0          1000            1050            50
# order-processors orders   1          2000            2000            0
# order-processors orders   2          1500            1600            100
```

### 5. Изоляция по назначению

```java
// Разделяйте группы по функциональному назначению
// Не объединяйте разную логику в одну группу

// Группа для обработки заказов
props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-processing");

// Отдельная группа для отправки email
props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-email-notifications");

// Отдельная группа для аналитики
props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-analytics");
```

## Распространённые ошибки

### 1. Больше consumers, чем партиций

```
НЕПРАВИЛЬНО:
Топик: 3 партиции
Consumer Group: 5 consumers
Результат: 2 consumer'а простаивают, ресурсы тратятся впустую

ПРАВИЛЬНО:
Увеличьте количество партиций ДО добавления consumer'ов
или используйте меньше consumer'ов
```

### 2. Одинаковый group.id для разной логики

```java
// НЕПРАВИЛЬНО - разная логика в одной группе
// Сообщение обработается только ОДНИМ consumer'ом!

// Consumer 1: отправка email
props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-handlers");

// Consumer 2: обновление статистики (ТА ЖЕ группа!)
props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-handlers");

// ПРАВИЛЬНО - разные группы для разной логики
props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-email-sender");
props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-statistics-updater");
```

### 3. Игнорирование ребалансировки

```java
// НЕПРАВИЛЬНО - не обрабатывается ребалансировка
consumer.subscribe(Collections.singletonList("topic"));

// ПРАВИЛЬНО - обработка ребалансировки
consumer.subscribe(Collections.singletonList("topic"), new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // Коммит перед потерей партиций
        consumer.commitSync();
        // Освобождение ресурсов, связанных с партициями
    }

    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // Инициализация для новых партиций
        System.out.println("Назначены: " + partitions);
    }
});
```

### 4. Отсутствие мониторинга лага

```java
// Реализуйте мониторинг consumer lag
// Используйте метрики Kafka или внешние системы (Prometheus, Grafana)

// Метрики для мониторинга:
// - records-lag-max: максимальный лаг среди партиций
// - records-consumed-rate: скорость потребления
// - fetch-rate: частота fetch-запросов
```

### 5. Неправильная обработка перезапуска

```java
// НЕПРАВИЛЬНО - потеря offset'ов при аварийном завершении
while (true) {
    records = consumer.poll(Duration.ofMillis(1000));
    process(records);
    // Коммит пропущен!
}

// ПРАВИЛЬНО - graceful shutdown с коммитом
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    consumer.wakeup();
}));

try {
    while (true) {
        records = consumer.poll(Duration.ofMillis(1000));
        process(records);
        consumer.commitSync();
    }
} catch (WakeupException e) {
    // Ожидаемое исключение при shutdown
} finally {
    consumer.commitSync(); // Финальный коммит
    consumer.close();
}
```

## Сценарии масштабирования

### Горизонтальное масштабирование

```
Начальное состояние:
- Топик: 6 партиций
- Consumer Group: 2 consumers
- Распределение: Consumer1 (P0, P1, P2), Consumer2 (P3, P4, P5)

После добавления consumer:
- Топик: 6 партиций
- Consumer Group: 3 consumers
- Распределение: Consumer1 (P0, P1), Consumer2 (P2, P3), Consumer3 (P4, P5)

Максимальное масштабирование:
- Топик: 6 партиций
- Consumer Group: 6 consumers
- Распределение: каждый consumer получает 1 партицию
```

### Увеличение партиций для масштабирования

```bash
# Добавление партиций в топик
kafka-topics.sh --bootstrap-server localhost:9092 \
    --alter --topic orders --partitions 12

# Теперь можно масштабировать до 12 consumer'ов
```

## Дополнительные ресурсы

- [Kafka Consumer Groups Documentation](https://kafka.apache.org/documentation/#intro_consumers)
- [Confluent Consumer Group Guide](https://docs.confluent.io/platform/current/clients/consumer.html#consumer-groups)
- [Kafka Consumer Group Management](https://kafka.apache.org/documentation/#impl_consumer)

---

[prev: 01-example](./01-example.md) | [next: 03-tracing](./03-tracing.md)
