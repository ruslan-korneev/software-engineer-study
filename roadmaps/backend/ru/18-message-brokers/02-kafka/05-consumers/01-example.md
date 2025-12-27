# Пример потребителя (Consumer API)

## Описание

Kafka Consumer API — это набор интерфейсов и классов для чтения данных из топиков Apache Kafka. Consumer (потребитель) подписывается на один или несколько топиков и получает сообщения, опубликованные продюсерами. Consumer API предоставляет гибкий механизм для управления чтением данных, включая контроль над позицией чтения (offset), группировку потребителей и обработку ошибок.

## Ключевые концепции

### Основные компоненты Consumer API

1. **KafkaConsumer** — главный класс для создания потребителя
2. **ConsumerRecord** — объект, представляющий одно сообщение
3. **ConsumerRecords** — коллекция сообщений, полученных за один poll()
4. **TopicPartition** — идентификатор партиции топика
5. **OffsetAndMetadata** — позиция чтения с метаданными

### Жизненный цикл Consumer

```
Создание → Подписка → Poll Loop → Обработка → Commit → Закрытие
```

### Обязательные конфигурации

| Параметр | Описание |
|----------|----------|
| `bootstrap.servers` | Адреса брокеров Kafka |
| `key.deserializer` | Десериализатор для ключей |
| `value.deserializer` | Десериализатор для значений |
| `group.id` | Идентификатор группы потребителей |

## Примеры кода

### Базовый пример Consumer на Java

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.Collections;
import java.util.Properties;

public class SimpleConsumerExample {

    public static void main(String[] args) {
        // Конфигурация потребителя
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "my-consumer-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());

        // Дополнительные настройки
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");

        // Создание потребителя
        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {

            // Подписка на топик
            consumer.subscribe(Collections.singletonList("my-topic"));

            // Poll loop — основной цикл обработки
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("Получено сообщение: topic=%s, partition=%d, offset=%d, key=%s, value=%s%n",
                            record.topic(),
                            record.partition(),
                            record.offset(),
                            record.key(),
                            record.value());

                    // Обработка сообщения
                    processMessage(record);
                }

                // Ручной коммит после обработки
                consumer.commitSync();
            }
        }
    }

    private static void processMessage(ConsumerRecord<String, String> record) {
        // Бизнес-логика обработки сообщения
        System.out.println("Обработка: " + record.value());
    }
}
```

### Consumer на Python (confluent-kafka)

```python
from confluent_kafka import Consumer, KafkaError, KafkaException
import sys

def create_consumer():
    """Создание и настройка Consumer"""
    conf = {
        'bootstrap.servers': 'localhost:9092',
        'group.id': 'my-python-group',
        'auto.offset.reset': 'earliest',
        'enable.auto.commit': False,
    }

    return Consumer(conf)

def consume_messages(consumer, topics):
    """Основной цикл потребления сообщений"""
    try:
        consumer.subscribe(topics)

        while True:
            msg = consumer.poll(timeout=1.0)

            if msg is None:
                continue

            if msg.error():
                if msg.error().code() == KafkaError._PARTITION_EOF:
                    # Достигнут конец партиции
                    print(f'Конец партиции {msg.topic()} [{msg.partition()}]')
                else:
                    raise KafkaException(msg.error())
            else:
                # Успешное получение сообщения
                print(f'Получено: topic={msg.topic()}, '
                      f'partition={msg.partition()}, '
                      f'offset={msg.offset()}, '
                      f'key={msg.key()}, '
                      f'value={msg.value().decode("utf-8")}')

                # Обработка сообщения
                process_message(msg)

                # Ручной коммит
                consumer.commit(asynchronous=False)

    except KeyboardInterrupt:
        pass
    finally:
        consumer.close()

def process_message(msg):
    """Обработка полученного сообщения"""
    print(f"Обработка: {msg.value().decode('utf-8')}")

if __name__ == '__main__':
    consumer = create_consumer()
    consume_messages(consumer, ['my-topic'])
```

### Consumer с ручным назначением партиций

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;

import java.time.Duration;
import java.util.Arrays;
import java.util.Properties;

public class ManualPartitionAssignment {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        // group.id не обязателен при ручном назначении

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {

            // Ручное назначение конкретных партиций
            TopicPartition partition0 = new TopicPartition("my-topic", 0);
            TopicPartition partition1 = new TopicPartition("my-topic", 1);
            consumer.assign(Arrays.asList(partition0, partition1));

            // Можно установить начальную позицию
            consumer.seekToBeginning(Arrays.asList(partition0));
            consumer.seek(partition1, 100); // Начать с offset 100

            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("Partition: %d, Offset: %d, Value: %s%n",
                            record.partition(), record.offset(), record.value());
                }
            }
        }
    }
}
```

### Consumer с десериализацией JSON

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.kafka.common.serialization.Deserializer;

import java.util.Map;

public class JsonDeserializer<T> implements Deserializer<T> {

    private final ObjectMapper objectMapper = new ObjectMapper();
    private Class<T> type;

    public JsonDeserializer(Class<T> type) {
        this.type = type;
    }

    @Override
    public void configure(Map<String, ?> configs, boolean isKey) {
        // Можно получить тип из конфигурации
    }

    @Override
    public T deserialize(String topic, byte[] data) {
        if (data == null) {
            return null;
        }
        try {
            return objectMapper.readValue(data, type);
        } catch (Exception e) {
            throw new RuntimeException("Ошибка десериализации JSON", e);
        }
    }

    @Override
    public void close() {
        // Освобождение ресурсов
    }
}

// Использование
// props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
```

## Best Practices

### 1. Правильное закрытие Consumer

```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    System.out.println("Закрытие consumer...");
    consumer.wakeup(); // Прерывает poll()
}));

try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
        // обработка
    }
} catch (WakeupException e) {
    // Игнорируем - это ожидаемое исключение при shutdown
} finally {
    consumer.close(); // Всегда закрываем consumer
}
```

### 2. Обработка в отдельном потоке

```java
// Consumer НЕ thread-safe!
// Используйте один consumer на поток
ExecutorService executor = Executors.newFixedThreadPool(3);

for (int i = 0; i < 3; i++) {
    executor.submit(() -> {
        // Каждый поток создаёт свой consumer
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        // ... poll loop
    });
}
```

### 3. Оптимальные настройки для производительности

```java
props.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 1024);      // Мин. размер fetch
props.put(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, 500);     // Макс. ожидание
props.put(ConsumerConfig.MAX_PARTITION_FETCH_BYTES_CONFIG, 1048576); // 1MB на партицию
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);      // Макс. записей за poll
```

### 4. Использование subscribe с RebalanceListener

```java
consumer.subscribe(Collections.singletonList("my-topic"), new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // Вызывается перед ребалансировкой
        // Хорошее место для коммита текущих offset
        consumer.commitSync();
        System.out.println("Отозваны партиции: " + partitions);
    }

    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // Вызывается после назначения новых партиций
        System.out.println("Назначены партиции: " + partitions);
    }
});
```

## Распространённые ошибки

### 1. Забыть указать group.id

```java
// НЕПРАВИЛЬНО - будет ошибка при использовании subscribe()
Properties props = new Properties();
props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
// Нет group.id!

// ПРАВИЛЬНО
props.put(ConsumerConfig.GROUP_ID_CONFIG, "my-group");
```

### 2. Долгая обработка в poll loop

```java
// НЕПРАВИЛЬНО - может привести к ребалансировке
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    for (ConsumerRecord<String, String> record : records) {
        Thread.sleep(60000); // Слишком долгая обработка!
    }
}

// ПРАВИЛЬНО - используйте асинхронную обработку или увеличьте max.poll.interval.ms
props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000); // 5 минут
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 10); // Меньше записей за раз
```

### 3. Использование consumer из нескольких потоков

```java
// НЕПРАВИЛЬНО - Consumer не thread-safe!
ExecutorService executor = Executors.newFixedThreadPool(2);
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);

executor.submit(() -> consumer.poll(Duration.ofMillis(1000))); // Поток 1
executor.submit(() -> consumer.commitSync());                   // Поток 2

// ПРАВИЛЬНО - один consumer на поток или использовать wakeup()
```

### 4. Игнорирование ошибок десериализации

```java
// ПРАВИЛЬНО - обработка ошибок десериализации
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
          ErrorHandlingDeserializer.class);
props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS,
          StringDeserializer.class);
props.put(ErrorHandlingDeserializer.VALUE_FUNCTION,
          (record, exception) -> {
              log.error("Ошибка десериализации", exception);
              return null; // или специальное значение
          });
```

### 5. Коммит до обработки

```java
// НЕПРАВИЛЬНО - потеря данных при сбое после коммита
consumer.commitSync();
processRecords(records); // Если упадёт здесь - данные потеряны

// ПРАВИЛЬНО - коммит после успешной обработки
processRecords(records);
consumer.commitSync();
```

## Конфигурационные параметры Consumer

| Параметр | По умолчанию | Описание |
|----------|--------------|----------|
| `session.timeout.ms` | 45000 | Таймаут сессии с брокером |
| `heartbeat.interval.ms` | 3000 | Интервал heartbeat |
| `max.poll.interval.ms` | 300000 | Макс. интервал между poll() |
| `auto.offset.reset` | latest | Поведение при отсутствии offset |
| `enable.auto.commit` | true | Автоматический коммит |
| `auto.commit.interval.ms` | 5000 | Интервал автокоммита |
| `fetch.min.bytes` | 1 | Минимальный размер fetch |
| `fetch.max.wait.ms` | 500 | Макс. ожидание fetch |
| `max.poll.records` | 500 | Макс. записей за poll() |

## Дополнительные ресурсы

- [Apache Kafka Documentation - Consumer](https://kafka.apache.org/documentation/#consumerapi)
- [Confluent Kafka Consumer Guide](https://docs.confluent.io/platform/current/clients/consumer.html)
- [Kafka Consumer Javadoc](https://kafka.apache.org/35/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html)
