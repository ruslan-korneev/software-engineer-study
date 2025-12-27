# Сжатые темы (Compacted Topics)

## Описание

**Compacted Topic (сжатый топик)** — это специальный тип топика в Apache Kafka с политикой очистки `cleanup.policy=compact`. В отличие от обычных топиков, где данные удаляются по времени или размеру, в сжатых топиках Kafka сохраняет только **последнее значение для каждого ключа**.

### Основная идея

```
До компактификации:
Offset:  0    1    2    3    4    5    6    7
Key:    [A]  [B]  [A]  [C]  [B]  [A]  [C]  [D]
Value:  [v1] [v1] [v2] [v1] [v2] [v3] [v2] [v1]

После компактификации:
Offset:  5    6    7
Key:    [A]  [C]  [D]
Value:  [v3] [v2] [v1]

Сохраняется только последнее значение для каждого уникального ключа!
```

### Зачем нужны сжатые топики?

- **Хранение состояния**: текущее состояние объектов (пользователи, заказы, конфигурации)
- **Changelog для баз данных**: CDC (Change Data Capture)
- **Event Sourcing**: компактный снапшот состояния
- **Конфигурации**: хранение актуальных настроек
- **Кэширование**: распределенный кэш состояния

## Ключевые концепции

### Структура хранения

```
Partition Log:
┌──────────────────────────────────────────────────────────┐
│  Head (активный сегмент)  │  Tail (компактируемые)       │
│  ─────────────────────    │  ───────────────────────     │
│  [K1:V5] [K3:V2] [K1:V6]  │  [K1:V4] [K2:V1] [K3:V1]     │
│       ↑                    │           ↑                  │
│  Новые записи              │    Подлежат компактификации  │
└──────────────────────────────────────────────────────────┘

После компактификации Tail:
│  [K1:V4] [K2:V1] [K3:V1]  →  [K2:V1]
│  (K1 и K3 есть в Head, K2 уникален)
```

### Параметры конфигурации

| Параметр | Описание | Значение по умолчанию |
|----------|----------|----------------------|
| `cleanup.policy` | Политика очистки | `delete` |
| `min.cleanable.dirty.ratio` | Минимальное соотношение "грязных" записей | 0.5 |
| `min.compaction.lag.ms` | Минимальная задержка перед компактификацией | 0 |
| `max.compaction.lag.ms` | Максимальная задержка компактификации | 9223372036854775807 |
| `delete.retention.ms` | Время хранения tombstone | 86400000 (24 часа) |
| `segment.ms` | Время жизни сегмента | 604800000 (7 дней) |

### Tombstone (маркер удаления)

**Tombstone** — это сообщение с ключом и `null` значением. Оно означает, что запись с данным ключом должна быть удалена.

```
// Создание tombstone
producer.send(new ProducerRecord<>("users", "user-123", null));

// После компактификации:
// - Все предыдущие значения для "user-123" удалены
// - Tombstone хранится delete.retention.ms миллисекунд
// - Затем tombstone тоже удаляется
```

### Гарантии компактификации

1. **Порядок сохраняется**: offset'ы не меняются, удаляются только дубликаты
2. **Актуальность**: для каждого ключа гарантируется последнее значение
3. **Задержка**: компактификация происходит в фоне, не мгновенно
4. **Неизменность Head**: активный сегмент не компактируется

## Примеры

### Создание сжатого топика

```bash
# Через CLI
kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic user-profiles \
  --partitions 6 \
  --replication-factor 3 \
  --config cleanup.policy=compact \
  --config min.cleanable.dirty.ratio=0.3 \
  --config delete.retention.ms=86400000 \
  --config segment.ms=3600000

# Комбинированная политика (compact + delete)
kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic orders-state \
  --partitions 3 \
  --replication-factor 3 \
  --config cleanup.policy=compact,delete \
  --config retention.ms=604800000 \
  --config min.cleanable.dirty.ratio=0.5
```

### Изменение политики существующего топика

```bash
# Изменить на compact
kafka-configs.sh --alter \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name my-topic \
  --add-config cleanup.policy=compact

# Проверить конфигурацию
kafka-configs.sh --describe \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name my-topic
```

### Java: работа со сжатыми топиками

```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.clients.admin.*;
import java.util.*;

public class CompactedTopicExample {

    private static final String TOPIC = "user-profiles";
    private static final String BOOTSTRAP_SERVERS = "localhost:9092";

    // Создание сжатого топика программно
    public static void createCompactedTopic() {
        Properties props = new Properties();
        props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);

        try (AdminClient adminClient = AdminClient.create(props)) {

            Map<String, String> topicConfig = new HashMap<>();
            topicConfig.put("cleanup.policy", "compact");
            topicConfig.put("min.cleanable.dirty.ratio", "0.3");
            topicConfig.put("delete.retention.ms", "86400000");
            topicConfig.put("segment.ms", "3600000");

            NewTopic newTopic = new NewTopic(TOPIC, 6, (short) 3)
                .configs(topicConfig);

            adminClient.createTopics(Collections.singleton(newTopic)).all().get();
            System.out.println("Compacted topic created: " + TOPIC);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // Отправка данных в сжатый топик
    public static void sendUserProfile(String userId, String profileJson) {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
            "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
            "org.apache.kafka.common.serialization.StringSerializer");
        // Важно для компактификации
        props.put(ProducerConfig.ACKS_CONFIG, "all");

        try (KafkaProducer<String, String> producer = new KafkaProducer<>(props)) {

            // Ключ обязателен для сжатых топиков!
            ProducerRecord<String, String> record = new ProducerRecord<>(
                TOPIC,
                userId,      // Ключ - идентификатор пользователя
                profileJson  // Значение - профиль
            );

            producer.send(record, (metadata, exception) -> {
                if (exception == null) {
                    System.out.printf("Sent: key=%s, partition=%d, offset=%d%n",
                        userId, metadata.partition(), metadata.offset());
                } else {
                    exception.printStackTrace();
                }
            });

        }
    }

    // Удаление записи (отправка tombstone)
    public static void deleteUserProfile(String userId) {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
            "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
            "org.apache.kafka.common.serialization.StringSerializer");

        try (KafkaProducer<String, String> producer = new KafkaProducer<>(props)) {

            // Tombstone: ключ + null значение
            ProducerRecord<String, String> tombstone = new ProducerRecord<>(
                TOPIC,
                userId,
                null  // null значение = удаление
            );

            producer.send(tombstone).get();
            System.out.println("Tombstone sent for user: " + userId);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // Чтение всех актуальных профилей (от начала)
    public static Map<String, String> readAllProfiles() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
            "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
            "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");

        Map<String, String> profiles = new HashMap<>();

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {

            // Назначаем все партиции
            List<TopicPartition> partitions = consumer.partitionsFor(TOPIC)
                .stream()
                .map(info -> new TopicPartition(TOPIC, info.partition()))
                .toList();

            consumer.assign(partitions);
            consumer.seekToBeginning(partitions);

            // Определяем конечные offset'ы
            Map<TopicPartition, Long> endOffsets = consumer.endOffsets(partitions);
            Map<TopicPartition, Boolean> finished = new HashMap<>();
            partitions.forEach(tp -> finished.put(tp, false));

            while (finished.containsValue(false)) {
                ConsumerRecords<String, String> records =
                    consumer.poll(Duration.ofMillis(100));

                for (ConsumerRecord<String, String> record : records) {
                    if (record.value() != null) {
                        // Обновляем или добавляем профиль
                        profiles.put(record.key(), record.value());
                    } else {
                        // Tombstone - удаляем профиль
                        profiles.remove(record.key());
                    }

                    // Проверяем, достигли ли конца
                    TopicPartition tp = new TopicPartition(record.topic(), record.partition());
                    if (record.offset() + 1 >= endOffsets.get(tp)) {
                        finished.put(tp, true);
                    }
                }
            }

        }

        return profiles;
    }

    public static void main(String[] args) {
        // Создаем топик
        createCompactedTopic();

        // Отправляем профили
        sendUserProfile("user-1", "{\"name\": \"Alice\", \"age\": 25}");
        sendUserProfile("user-2", "{\"name\": \"Bob\", \"age\": 30}");
        sendUserProfile("user-1", "{\"name\": \"Alice\", \"age\": 26}"); // Обновление

        // Удаляем пользователя
        deleteUserProfile("user-2");

        // Читаем все актуальные профили
        Map<String, String> profiles = readAllProfiles();
        profiles.forEach((k, v) -> System.out.println(k + " -> " + v));
        // Вывод: user-1 -> {"name": "Alice", "age": 26}
    }
}
```

### Python: работа со сжатыми топиками

```python
from kafka import KafkaProducer, KafkaConsumer, TopicPartition
from kafka.admin import KafkaAdminClient, NewTopic, ConfigResource, ConfigResourceType
import json
from typing import Dict, Optional


class CompactedTopicManager:
    """Менеджер для работы со сжатыми топиками"""

    def __init__(self, bootstrap_servers: str):
        self.bootstrap_servers = bootstrap_servers

    def create_compacted_topic(self,
                               topic_name: str,
                               num_partitions: int = 3,
                               replication_factor: int = 1) -> None:
        """Создание сжатого топика"""

        admin = KafkaAdminClient(bootstrap_servers=self.bootstrap_servers)

        topic = NewTopic(
            name=topic_name,
            num_partitions=num_partitions,
            replication_factor=replication_factor,
            topic_configs={
                'cleanup.policy': 'compact',
                'min.cleanable.dirty.ratio': '0.3',
                'delete.retention.ms': '86400000',
                'segment.ms': '3600000'
            }
        )

        try:
            admin.create_topics([topic])
            print(f"Compacted topic '{topic_name}' created")
        except Exception as e:
            print(f"Error creating topic: {e}")
        finally:
            admin.close()


class UserProfileService:
    """Сервис для работы с профилями пользователей через сжатый топик"""

    def __init__(self, bootstrap_servers: str, topic: str = 'user-profiles'):
        self.bootstrap_servers = bootstrap_servers
        self.topic = topic
        self._init_producer()

    def _init_producer(self):
        self.producer = KafkaProducer(
            bootstrap_servers=self.bootstrap_servers,
            key_serializer=lambda k: k.encode('utf-8'),
            value_serializer=lambda v: json.dumps(v).encode('utf-8') if v else None,
            acks='all'
        )

    def save_profile(self, user_id: str, profile: dict) -> None:
        """Сохранение профиля пользователя"""
        future = self.producer.send(self.topic, key=user_id, value=profile)
        result = future.get(timeout=10)
        print(f"Profile saved: {user_id} at offset {result.offset}")

    def delete_profile(self, user_id: str) -> None:
        """Удаление профиля (отправка tombstone)"""
        future = self.producer.send(self.topic, key=user_id, value=None)
        result = future.get(timeout=10)
        print(f"Profile deleted (tombstone): {user_id} at offset {result.offset}")

    def get_all_profiles(self) -> Dict[str, dict]:
        """Чтение всех актуальных профилей"""
        consumer = KafkaConsumer(
            bootstrap_servers=self.bootstrap_servers,
            key_deserializer=lambda k: k.decode('utf-8'),
            value_deserializer=lambda v: json.loads(v.decode('utf-8')) if v else None,
            auto_offset_reset='earliest',
            enable_auto_commit=False
        )

        # Получаем все партиции
        partitions = consumer.partitions_for_topic(self.topic)
        if not partitions:
            consumer.close()
            return {}

        topic_partitions = [TopicPartition(self.topic, p) for p in partitions]
        consumer.assign(topic_partitions)
        consumer.seek_to_beginning(*topic_partitions)

        # Получаем конечные offset'ы
        end_offsets = consumer.end_offsets(topic_partitions)

        profiles: Dict[str, dict] = {}
        finished = {tp: False for tp in topic_partitions}

        # Если топик пустой
        if all(end_offsets[tp] == 0 for tp in topic_partitions):
            consumer.close()
            return profiles

        while not all(finished.values()):
            records = consumer.poll(timeout_ms=1000)

            for tp, messages in records.items():
                for message in messages:
                    if message.value is not None:
                        profiles[message.key] = message.value
                    else:
                        # Tombstone - удаляем
                        profiles.pop(message.key, None)

                    if message.offset + 1 >= end_offsets[tp]:
                        finished[tp] = True

            # Проверяем пустые партиции
            for tp in topic_partitions:
                if end_offsets[tp] == 0:
                    finished[tp] = True

        consumer.close()
        return profiles

    def get_profile(self, user_id: str) -> Optional[dict]:
        """Получение профиля конкретного пользователя"""
        profiles = self.get_all_profiles()
        return profiles.get(user_id)

    def close(self):
        self.producer.close()


# Использование
if __name__ == "__main__":
    bootstrap_servers = "localhost:9092"
    topic = "user-profiles"

    # Создаем топик
    manager = CompactedTopicManager(bootstrap_servers)
    manager.create_compacted_topic(topic)

    # Работаем с профилями
    service = UserProfileService(bootstrap_servers, topic)

    try:
        # Сохраняем профили
        service.save_profile("user-001", {
            "name": "Иван Петров",
            "email": "ivan@example.com",
            "age": 30
        })

        service.save_profile("user-002", {
            "name": "Мария Сидорова",
            "email": "maria@example.com",
            "age": 25
        })

        # Обновляем профиль
        service.save_profile("user-001", {
            "name": "Иван Петров",
            "email": "ivan.petrov@example.com",  # Изменили email
            "age": 31
        })

        # Удаляем профиль
        service.delete_profile("user-002")

        # Читаем все профили
        print("\nАктуальные профили:")
        profiles = service.get_all_profiles()
        for user_id, profile in profiles.items():
            print(f"  {user_id}: {profile}")

    finally:
        service.close()
```

### Spring Boot: конфигурация сжатого топика

```java
import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration
public class CompactedTopicsConfig {

    @Bean
    public NewTopic userProfilesTopic() {
        return TopicBuilder.name("user-profiles")
            .partitions(6)
            .replicas(3)
            .compact()  // Эквивалентно cleanup.policy=compact
            .config("min.cleanable.dirty.ratio", "0.3")
            .config("delete.retention.ms", "86400000")
            .config("segment.ms", "3600000")
            .build();
    }

    @Bean
    public NewTopic orderStateTopic() {
        return TopicBuilder.name("order-state")
            .partitions(12)
            .replicas(3)
            .config("cleanup.policy", "compact,delete")  // Компактификация + TTL
            .config("retention.ms", "604800000")
            .config("min.cleanable.dirty.ratio", "0.5")
            .build();
    }
}
```

### Kafka Streams: использование сжатых топиков

```java
import org.apache.kafka.streams.*;
import org.apache.kafka.streams.kstream.*;
import org.apache.kafka.streams.state.*;

public class CompactedTopicStreamsExample {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "user-aggregator");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();

        // Читаем из обычного топика событий
        KStream<String, String> userEvents = builder.stream("user-events");

        // Группируем по ключу и агрегируем
        KTable<String, String> userProfiles = userEvents
            .groupByKey()
            .reduce(
                (oldValue, newValue) -> newValue,  // Берем последнее значение
                Materialized.<String, String, KeyValueStore<Bytes, byte[]>>as("user-profiles-store")
                    .withKeySerde(Serdes.String())
                    .withValueSerde(Serdes.String())
            );

        // Записываем в сжатый топик
        userProfiles.toStream().to("user-profiles-compacted");

        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();

        // Graceful shutdown
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```

### CDC (Change Data Capture) с Debezium

```java
// Debezium автоматически использует сжатые топики для CDC

// Пример записи из Debezium (упрощенно):
// Key: {"id": 123}
// Value: {
//   "before": {"id": 123, "name": "Old Name", "email": "old@mail.com"},
//   "after": {"id": 123, "name": "New Name", "email": "new@mail.com"},
//   "op": "u",  // update
//   "ts_ms": 1634567890123
// }

// При удалении записи:
// Key: {"id": 123}
// Value: null  // Tombstone
```

## Best Practices

### 1. Всегда используйте ключи

```java
// Правильно: с ключом
producer.send(new ProducerRecord<>("compacted-topic", "user-123", profile));

// Неправильно: без ключа (не будет работать компактификация!)
producer.send(new ProducerRecord<>("compacted-topic", profile));
```

### 2. Выбор ключа

```java
// Хорошо: уникальный идентификатор сущности
String key = entity.getId();

// Хорошо: составной ключ
String key = tenantId + ":" + entityId;

// Плохо: временная метка или случайное значение
String key = UUID.randomUUID().toString();  // Каждое сообщение уникально!
```

### 3. Размер сообщений

```properties
# Для больших сообщений увеличьте лимиты
max.message.bytes=10485760
replica.fetch.max.bytes=10485760

# Используйте сжатие
compression.type=lz4
```

### 4. Настройка компактификации

```properties
# Для частых обновлений
min.cleanable.dirty.ratio=0.3    # Чаще компактификация
segment.ms=3600000               # Меньше сегменты

# Для редких обновлений
min.cleanable.dirty.ratio=0.5    # Реже компактификация
segment.ms=86400000              # Большие сегменты
```

### 5. Мониторинг компактификации

```bash
# Проверка размера лога
kafka-log-dirs.sh --describe \
  --bootstrap-server localhost:9092 \
  --topic-list user-profiles

# Метрики компактификации
kafka.log:type=LogCleaner,name=max-clean-time-secs
kafka.log:type=LogCleaner,name=cleaner-recopy-percent
```

### 6. Обработка tombstone

```java
// Всегда проверяйте на null!
for (ConsumerRecord<String, String> record : records) {
    if (record.value() == null) {
        // Это tombstone - удаляем из локального состояния
        localCache.remove(record.key());
    } else {
        // Обычная запись
        localCache.put(record.key(), record.value());
    }
}
```

### 7. Комбинированная политика

```bash
# cleanup.policy=compact,delete
# - Компактификация: сохраняет последнее значение для ключа
# - Удаление: удаляет записи старше retention.ms

kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic orders-state \
  --config cleanup.policy=compact,delete \
  --config retention.ms=604800000
```

## Типичные ошибки

### 1. Отсутствие ключа

```
Проблема: сообщения без ключа не компактифицируются
Решение: всегда указывайте ключ
```

### 2. Слишком маленький delete.retention.ms

```
Проблема: tombstone удаляются раньше, чем консьюмер их прочитает
Решение: увеличьте delete.retention.ms
```

### 3. Большие сообщения

```
Проблема: компактификация замедляется
Решение: разбивайте данные, используйте сжатие
```

### 4. Ожидание мгновенной компактификации

```
Проблема: старые значения все еще видны после записи нового
Причина: компактификация работает в фоне
Решение: используйте логику "последняя запись выигрывает" в приложении
```

## Сценарии использования

| Сценарий | Политика | Параметры |
|----------|----------|-----------|
| User profiles | compact | min.dirty.ratio=0.3 |
| Order state | compact,delete | retention.ms=7d |
| Config store | compact | delete.retention.ms=7d |
| CDC changelog | compact | segment.ms=1h |
| Session cache | compact,delete | retention.ms=24h |

## Связанные темы

- [Топики](./01-topics.md)
- [Партиции](./02-partitions.md)
- [Тестирование с EmbeddedKafka](./03-embedded-kafka-testing.md)
