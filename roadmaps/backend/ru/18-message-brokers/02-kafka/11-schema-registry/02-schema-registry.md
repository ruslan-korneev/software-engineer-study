# Реестр схем (Schema Registry)

[prev: 01-kafka-maturity-model](./01-kafka-maturity-model.md) | [next: 03-registry-components](./03-registry-components.md)

---

## Описание

Schema Registry — это централизованный сервис для управления схемами данных в Apache Kafka. Он хранит версионированные схемы Avro, Protobuf и JSON Schema, обеспечивая согласованность формата сообщений между продюсерами и консьюмерами. Schema Registry разработан компанией Confluent и является ключевым компонентом для построения надежных событийно-ориентированных архитектур.

### Зачем нужен Schema Registry?

В распределенных системах продюсеры и консьюмеры часто разрабатываются разными командами и деплоятся независимо. Без централизованного управления схемами возникают проблемы:

1. **Несовместимость форматов**: Продюсер изменил структуру сообщения, а консьюмер не обновлен
2. **Отсутствие документации**: Непонятно, какие поля содержит сообщение
3. **Ошибки десериализации**: Runtime-ошибки при парсинге сообщений
4. **Сложность миграции**: Невозможно безопасно эволюционировать схемы

Schema Registry решает эти проблемы через:
- Централизованное хранение схем
- Версионирование с проверкой совместимости
- Автоматическую валидацию при сериализации

## Ключевые концепции

### Архитектура Schema Registry

```
┌─────────────────────────────────────────────────────────────────┐
│                        Schema Registry                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    REST API (:8081)                      │   │
│  │    GET/POST /subjects  |  GET /schemas  |  GET /config   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Schema Storage (Kafka Topic)                │   │
│  │                  _schemas (compacted)                    │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
          │                                         │
          ▼                                         ▼
┌──────────────────┐                    ┌──────────────────┐
│    Producer      │                    │    Consumer      │
│  (Avro/Protobuf) │                    │  (Avro/Protobuf) │
└──────────────────┘                    └──────────────────┘
          │                                         │
          ▼                                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Kafka Cluster                            │
│                    [Topic: user-events]                          │
└─────────────────────────────────────────────────────────────────┘
```

### Основные компоненты

#### Subject (Субъект)
Subject — это область видимости для схемы, обычно связанная с топиком Kafka. По умолчанию используется стратегия именования:
- `<topic-name>-key` — для схемы ключа
- `<topic-name>-value` — для схемы значения

```
Топик: orders
├── orders-key (схема ключа: order_id)
└── orders-value (схема значения: Order)
```

#### Schema ID
Каждая схема при регистрации получает уникальный глобальный идентификатор. Этот ID включается в сериализованное сообщение:

```
┌─────────────────────────────────────────────┐
│            Kafka Message Value              │
├──────┬────────────┬─────────────────────────┤
│ 0x00 │ Schema ID  │     Avro/Protobuf Data  │
│(1 b) │  (4 bytes) │     (variable length)   │
└──────┴────────────┴─────────────────────────┘
   │        │
   │        └── ID схемы в Schema Registry
   └── Magic byte (всегда 0)
```

#### Version (Версия)
Каждый subject может иметь множество версий схемы. Версии нумеруются начиная с 1.

### Стратегии именования субъектов

| Стратегия | Формат | Использование |
|-----------|--------|---------------|
| TopicNameStrategy | `<topic>-<key\|value>` | По умолчанию, одна схема на топик |
| RecordNameStrategy | `<namespace>.<record>` | Одна схема для типа записи |
| TopicRecordNameStrategy | `<topic>-<namespace>.<record>` | Комбинация топика и типа |

## Примеры кода

### Запуск Schema Registry с Docker

```yaml
# docker-compose.yml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.0
    depends_on:
      - kafka
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:9092
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
```

### REST API примеры

```bash
# Получить список всех субъектов
curl -X GET http://localhost:8081/subjects

# Зарегистрировать новую схему Avro
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{
    "schema": "{\"type\":\"record\",\"name\":\"User\",\"fields\":[{\"name\":\"id\",\"type\":\"long\"},{\"name\":\"name\",\"type\":\"string\"}]}"
  }' \
  http://localhost:8081/subjects/users-value/versions

# Ответ: {"id": 1}

# Получить схему по ID
curl -X GET http://localhost:8081/schemas/ids/1

# Получить все версии схемы для субъекта
curl -X GET http://localhost:8081/subjects/users-value/versions

# Получить конкретную версию схемы
curl -X GET http://localhost:8081/subjects/users-value/versions/1

# Получить последнюю версию схемы
curl -X GET http://localhost:8081/subjects/users-value/versions/latest

# Проверить совместимость новой схемы
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{
    "schema": "{\"type\":\"record\",\"name\":\"User\",\"fields\":[{\"name\":\"id\",\"type\":\"long\"},{\"name\":\"name\",\"type\":\"string\"},{\"name\":\"email\",\"type\":[\"null\",\"string\"],\"default\":null}]}"
  }' \
  http://localhost:8081/compatibility/subjects/users-value/versions/latest

# Получить текущий уровень совместимости
curl -X GET http://localhost:8081/config/users-value

# Установить уровень совместимости для субъекта
curl -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"compatibility": "BACKWARD"}' \
  http://localhost:8081/config/users-value
```

### Python: Продюсер с Avro и Schema Registry

```python
from confluent_kafka import Producer
from confluent_kafka.serialization import SerializationContext, MessageField
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer

# Схема Avro для пользователя
user_schema_str = """
{
    "type": "record",
    "name": "User",
    "namespace": "com.example",
    "fields": [
        {"name": "id", "type": "long"},
        {"name": "name", "type": "string"},
        {"name": "email", "type": ["null", "string"], "default": null},
        {"name": "created_at", "type": {"type": "long", "logicalType": "timestamp-millis"}}
    ]
}
"""

# Конфигурация Schema Registry
schema_registry_conf = {'url': 'http://localhost:8081'}
schema_registry_client = SchemaRegistryClient(schema_registry_conf)

# Функция преобразования объекта в словарь
def user_to_dict(user, ctx):
    return {
        "id": user.id,
        "name": user.name,
        "email": user.email,
        "created_at": int(user.created_at.timestamp() * 1000)
    }

# Создание сериализатора (схема автоматически регистрируется)
avro_serializer = AvroSerializer(
    schema_registry_client,
    user_schema_str,
    user_to_dict
)

# Конфигурация продюсера
producer_conf = {
    'bootstrap.servers': 'localhost:9092',
    'client.id': 'user-producer'
}
producer = Producer(producer_conf)

# Отправка сообщения
class User:
    def __init__(self, id, name, email, created_at):
        self.id = id
        self.name = name
        self.email = email
        self.created_at = created_at

from datetime import datetime

user = User(
    id=1,
    name="Иван Петров",
    email="ivan@example.com",
    created_at=datetime.now()
)

def delivery_callback(err, msg):
    if err:
        print(f"Ошибка доставки: {err}")
    else:
        print(f"Сообщение доставлено в {msg.topic()} [{msg.partition()}] @ {msg.offset()}")

producer.produce(
    topic='users',
    key=str(user.id),
    value=avro_serializer(user, SerializationContext('users', MessageField.VALUE)),
    on_delivery=delivery_callback
)

producer.flush()
```

### Python: Консьюмер с Avro и Schema Registry

```python
from confluent_kafka import Consumer
from confluent_kafka.serialization import SerializationContext, MessageField
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroDeserializer

# Конфигурация Schema Registry
schema_registry_conf = {'url': 'http://localhost:8081'}
schema_registry_client = SchemaRegistryClient(schema_registry_conf)

# Функция преобразования словаря в объект
def dict_to_user(obj, ctx):
    if obj is None:
        return None
    return User(
        id=obj['id'],
        name=obj['name'],
        email=obj.get('email'),
        created_at=datetime.fromtimestamp(obj['created_at'] / 1000)
    )

# Создание десериализатора (схема загружается автоматически по ID)
avro_deserializer = AvroDeserializer(
    schema_registry_client,
    None,  # Схема будет получена из сообщения
    dict_to_user
)

# Конфигурация консьюмера
consumer_conf = {
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'user-consumer-group',
    'auto.offset.reset': 'earliest'
}
consumer = Consumer(consumer_conf)
consumer.subscribe(['users'])

try:
    while True:
        msg = consumer.poll(1.0)
        if msg is None:
            continue
        if msg.error():
            print(f"Ошибка консьюмера: {msg.error()}")
            continue

        user = avro_deserializer(
            msg.value(),
            SerializationContext(msg.topic(), MessageField.VALUE)
        )
        print(f"Получен пользователь: {user.name} ({user.email})")

except KeyboardInterrupt:
    pass
finally:
    consumer.close()
```

### Java: Использование Schema Registry

```java
import io.confluent.kafka.serializers.KafkaAvroSerializer;
import io.confluent.kafka.serializers.KafkaAvroDeserializer;
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.clients.consumer.*;
import org.apache.avro.generic.GenericRecord;
import org.apache.avro.generic.GenericData;
import org.apache.avro.Schema;

import java.util.Properties;
import java.util.Collections;

public class SchemaRegistryExample {

    public static void main(String[] args) {
        // Producer с Avro
        Properties producerProps = new Properties();
        producerProps.put("bootstrap.servers", "localhost:9092");
        producerProps.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        producerProps.put("value.serializer", KafkaAvroSerializer.class.getName());
        producerProps.put("schema.registry.url", "http://localhost:8081");

        // Определение схемы
        String schemaString = """
            {
                "type": "record",
                "name": "User",
                "fields": [
                    {"name": "id", "type": "long"},
                    {"name": "name", "type": "string"},
                    {"name": "email", "type": ["null", "string"], "default": null}
                ]
            }
            """;

        Schema.Parser parser = new Schema.Parser();
        Schema schema = parser.parse(schemaString);

        // Создание записи
        GenericRecord user = new GenericData.Record(schema);
        user.put("id", 1L);
        user.put("name", "Иван Петров");
        user.put("email", "ivan@example.com");

        try (Producer<String, GenericRecord> producer = new KafkaProducer<>(producerProps)) {
            ProducerRecord<String, GenericRecord> record =
                new ProducerRecord<>("users", "1", user);
            producer.send(record).get();
            System.out.println("Сообщение отправлено");
        } catch (Exception e) {
            e.printStackTrace();
        }

        // Consumer с Avro
        Properties consumerProps = new Properties();
        consumerProps.put("bootstrap.servers", "localhost:9092");
        consumerProps.put("group.id", "user-consumer");
        consumerProps.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        consumerProps.put("value.deserializer", KafkaAvroDeserializer.class.getName());
        consumerProps.put("schema.registry.url", "http://localhost:8081");
        consumerProps.put("specific.avro.reader", "false");

        try (Consumer<String, GenericRecord> consumer = new KafkaConsumer<>(consumerProps)) {
            consumer.subscribe(Collections.singletonList("users"));

            while (true) {
                ConsumerRecords<String, GenericRecord> records = consumer.poll(Duration.ofMillis(100));
                for (ConsumerRecord<String, GenericRecord> record : records) {
                    GenericRecord value = record.value();
                    System.out.printf("Получен пользователь: %s (%s)%n",
                        value.get("name"), value.get("email"));
                }
            }
        }
    }
}
```

## Конфигурация Schema Registry

### Основные параметры

```properties
# Listener для REST API
listeners=http://0.0.0.0:8081

# Kafka bootstrap servers
kafkastore.bootstrap.servers=PLAINTEXT://kafka:9092

# Топик для хранения схем (по умолчанию _schemas)
kafkastore.topic=_schemas

# Уровень совместимости по умолчанию
schema.compatibility.level=BACKWARD

# Количество реплик для топика _schemas
kafkastore.topic.replication.factor=3

# Timeout для соединения с Kafka
kafkastore.timeout.ms=500

# Режим работы: PRIMARY, SECONDARY, READWRITE, READONLY
mode=READWRITE
```

### Безопасность

```properties
# SSL для REST API
ssl.keystore.location=/path/to/keystore.jks
ssl.keystore.password=keystore-password
ssl.key.password=key-password

# SSL для Kafka
kafkastore.ssl.keystore.location=/path/to/keystore.jks
kafkastore.ssl.keystore.password=keystore-password
kafkastore.ssl.truststore.location=/path/to/truststore.jks
kafkastore.ssl.truststore.password=truststore-password

# Basic Auth
authentication.method=BASIC
authentication.roles=admin,developer,readonly
authentication.realm=SchemaRegistry-Props
```

## Best Practices

### Проектирование схем

1. **Используйте namespace**:
   ```json
   {
     "namespace": "com.company.domain.events",
     "name": "UserCreated",
     "type": "record"
   }
   ```

2. **Добавляйте значения по умолчанию**:
   ```json
   {"name": "email", "type": ["null", "string"], "default": null}
   ```

3. **Используйте логические типы для дат**:
   ```json
   {"name": "created_at", "type": {"type": "long", "logicalType": "timestamp-millis"}}
   ```

4. **Документируйте поля**:
   ```json
   {"name": "status", "type": "string", "doc": "Статус заказа: PENDING, CONFIRMED, SHIPPED, DELIVERED"}
   ```

### Управление версиями

1. **Начинайте с BACKWARD совместимости** — это режим по умолчанию
2. **Используйте FULL для критичных топиков** — гарантирует совместимость в обе стороны
3. **Тестируйте совместимость перед деплоем** — используйте API `/compatibility`
4. **Храните схемы в Git** вместе с кодом приложения

### Операционные рекомендации

| Рекомендация | Описание |
|--------------|----------|
| Высокая доступность | Запускайте минимум 3 инстанса Schema Registry |
| Мониторинг | Отслеживайте метрики через JMX |
| Бэкапы | Регулярно бэкапьте топик _schemas |
| Тестовое окружение | Используйте отдельный Schema Registry для dev/test |
| CI/CD интеграция | Проверяйте совместимость схем в пайплайне |

### Частые ошибки

1. **Изменение типа поля** — нарушает совместимость
2. **Удаление обязательного поля** — ломает старых консьюмеров
3. **Добавление поля без default** — ломает старых продюсеров
4. **Игнорирование namespace** — конфликты имен схем
5. **Отсутствие мониторинга** — проблемы обнаруживаются слишком поздно

## Интеграция с другими инструментами

### Kafka Connect

```json
{
  "name": "jdbc-source",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://localhost:8081"
  }
}
```

### ksqlDB

```sql
-- ksqlDB автоматически использует Schema Registry
CREATE STREAM users (
  id BIGINT,
  name VARCHAR,
  email VARCHAR
) WITH (
  KAFKA_TOPIC='users',
  VALUE_FORMAT='AVRO'
);
```

### Kafka Streams

```java
Properties props = new Properties();
props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, SpecificAvroSerde.class);
props.put("schema.registry.url", "http://localhost:8081");
```

---

[prev: 01-kafka-maturity-model](./01-kafka-maturity-model.md) | [next: 03-registry-components](./03-registry-components.md)
