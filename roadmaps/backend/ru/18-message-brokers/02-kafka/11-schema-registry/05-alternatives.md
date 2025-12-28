# Альтернативы реестру схем

[prev: 04-compatibility-rules](./04-compatibility-rules.md) | [next: 01-kafka-streams](../12-stream-processing/01-kafka-streams.md)

---

## Описание

Хотя Confluent Schema Registry является наиболее популярным решением для управления схемами в Kafka, существуют альтернативные подходы и инструменты. Выбор зависит от требований проекта: необходимости поддержки определённых форматов, интеграции с существующей инфраструктурой, лицензионных ограничений и операционных особенностей.

## Ключевые концепции

### Сравнение решений

| Характеристика | Confluent SR | Apicurio | Karapace | AWS Glue | Без реестра |
|----------------|--------------|----------|----------|----------|-------------|
| Лицензия | Community Edition (бесплатно) / Commercial | Apache 2.0 | Apache 2.0 | AWS | - |
| Avro | Да | Да | Да | Да | Да |
| Protobuf | Да | Да | Да | Нет | Да |
| JSON Schema | Да | Да | Да | Нет | Да |
| OpenAPI | Нет | Да | Нет | Нет | - |
| GraphQL | Нет | Да | Нет | Нет | - |
| API совместимость | - | Confluent | Confluent | Нет | - |
| Хранение | Kafka | Kafka/SQL/Infinispan | Kafka | AWS | Git/файлы |
| Облачный сервис | Confluent Cloud | OpenShift | Нет | Да | - |

### Критерии выбора

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Дерево выбора реестра схем                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Нужен ли реестр схем вообще?                                          │
│  ├── Нет → Схемы в Git + валидация в CI/CD                             │
│  │                                                                      │
│  └── Да → Используете AWS?                                              │
│           ├── Да, только Avro → AWS Glue Schema Registry               │
│           │                                                             │
│           └── Нет → Нужна ли коммерческая поддержка?                   │
│                     ├── Да → Confluent Schema Registry                 │
│                     │                                                   │
│                     └── Нет → OpenAPI/GraphQL схемы?                   │
│                               ├── Да → Apicurio Registry               │
│                               │                                         │
│                               └── Нет → Karapace                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Apicurio Registry

### Описание

Apicurio Registry — это открытое решение от Red Hat для управления схемами и API артефактами. Поддерживает множество форматов включая Avro, Protobuf, JSON Schema, OpenAPI, AsyncAPI и GraphQL. Может использоваться как замена Confluent Schema Registry с полной совместимостью API.

### Ключевые особенности

- **Множество форматов**: Avro, Protobuf, JSON Schema, OpenAPI, AsyncAPI, GraphQL, WSDL, XML Schema
- **Совместимость с Confluent API**: Drop-in замена для существующих клиентов
- **Гибкое хранение**: Kafka, PostgreSQL, Infinispan, In-memory
- **Интеграция с OpenShift**: Оператор для Kubernetes/OpenShift
- **CNCF проект**: Часть экосистемы Cloud Native

### Архитектура

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Apicurio Registry                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                       REST API                                 │ │
│  │  /apis/registry/v2  |  /apis/ccompat/v7 (Confluent compat)    │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                              │                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                    Core Registry                               │ │
│  │  • Artifact management                                         │ │
│  │  • Version control                                             │ │
│  │  • Compatibility checking                                      │ │
│  │  • Content validation                                          │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                              │                                     │
│  ┌───────────┬───────────────┬───────────────┬──────────────────┐ │
│  │   Kafka   │  PostgreSQL   │   Infinispan  │    In-Memory     │ │
│  │  Storage  │    Storage    │    Storage    │     Storage      │ │
│  └───────────┴───────────────┴───────────────┴──────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Примеры кода

#### Docker Compose для Apicurio

```yaml
# docker-compose.yml
version: '3.8'
services:
  apicurio:
    image: apicurio/apicurio-registry-mem:2.4.12.Final
    ports:
      - "8080:8080"
    environment:
      REGISTRY_AUTH_ANONYMOUS_READ_ACCESS_ENABLED: "true"
      LOG_LEVEL: INFO

  # Для использования с Kafka storage
  apicurio-kafka:
    image: apicurio/apicurio-registry-kafkasql:2.4.12.Final
    ports:
      - "8080:8080"
    environment:
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      REGISTRY_AUTH_ANONYMOUS_READ_ACCESS_ENABLED: "true"

  # Для использования с PostgreSQL
  apicurio-sql:
    image: apicurio/apicurio-registry-sql:2.4.12.Final
    ports:
      - "8080:8080"
    environment:
      REGISTRY_DATASOURCE_URL: jdbc:postgresql://postgres:5432/apicurio
      REGISTRY_DATASOURCE_USERNAME: apicurio
      REGISTRY_DATASOURCE_PASSWORD: password
    depends_on:
      - postgres

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: apicurio
      POSTGRES_USER: apicurio
      POSTGRES_PASSWORD: password
```

#### Использование Apicurio API

```bash
# Создание артефакта (схемы)
curl -X POST "http://localhost:8080/apis/registry/v2/groups/default/artifacts" \
  -H "Content-Type: application/json; artifactType=AVRO" \
  -H "X-Registry-ArtifactId: users-value" \
  --data '{
    "type": "record",
    "name": "User",
    "fields": [
      {"name": "id", "type": "long"},
      {"name": "name", "type": "string"}
    ]
  }'

# Получение артефакта
curl "http://localhost:8080/apis/registry/v2/groups/default/artifacts/users-value"

# Создание новой версии
curl -X POST "http://localhost:8080/apis/registry/v2/groups/default/artifacts/users-value/versions" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "record",
    "name": "User",
    "fields": [
      {"name": "id", "type": "long"},
      {"name": "name", "type": "string"},
      {"name": "email", "type": ["null", "string"], "default": null}
    ]
  }'

# Confluent-совместимый API
curl "http://localhost:8080/apis/ccompat/v7/subjects"
curl "http://localhost:8080/apis/ccompat/v7/subjects/users-value/versions"
```

#### Python: Использование с Confluent клиентом

```python
from confluent_kafka import Producer
from confluent_kafka.serialization import SerializationContext, MessageField
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer

# Apicurio совместим с Confluent API
schema_registry_conf = {
    # Используем Confluent-совместимый endpoint
    'url': 'http://localhost:8080/apis/ccompat/v7'
}

schema_registry_client = SchemaRegistryClient(schema_registry_conf)

user_schema = """
{
    "type": "record",
    "name": "User",
    "namespace": "com.example",
    "fields": [
        {"name": "id", "type": "long"},
        {"name": "name", "type": "string"}
    ]
}
"""

avro_serializer = AvroSerializer(
    schema_registry_client,
    user_schema,
    lambda obj, ctx: obj
)

producer = Producer({'bootstrap.servers': 'localhost:9092'})

user = {"id": 1, "name": "Иван"}
producer.produce(
    topic='users',
    value=avro_serializer(user, SerializationContext('users', MessageField.VALUE))
)
producer.flush()
```

#### Работа с OpenAPI схемами

```python
import requests

# Регистрация OpenAPI спецификации
openapi_spec = """
openapi: 3.0.0
info:
  title: User API
  version: 1.0.0
paths:
  /users:
    get:
      summary: Get users
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/User'
components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: integer
        name:
          type: string
"""

response = requests.post(
    "http://localhost:8080/apis/registry/v2/groups/default/artifacts",
    headers={
        "Content-Type": "application/x-yaml; artifactType=OPENAPI",
        "X-Registry-ArtifactId": "user-api"
    },
    data=openapi_spec
)
print(f"Created: {response.json()}")
```

---

## Karapace

### Описание

Karapace — это открытый проект от Aiven, предоставляющий совместимую с Confluent реализацию Schema Registry и REST Proxy для Kafka. Написан на Python, что делает его легко расширяемым и отлаживаемым.

### Ключевые особенности

- **100% совместимость** с Confluent Schema Registry API
- **Встроенный Kafka REST Proxy**: Produce/consume через HTTP
- **Написан на Python**: Легко модифицировать и отлаживать
- **Легковесный**: Меньше требований к ресурсам
- **Открытый исходный код**: Apache 2.0 лицензия

### Архитектура

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Karapace                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │                    Schema Registry API                          ││
│  │         /subjects, /schemas, /config, /compatibility            ││
│  └─────────────────────────────────────────────────────────────────┘│
│                              │                                      │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │                    REST Proxy API                               ││
│  │     /topics, /consumers, /brokers (produce/consume HTTP)        ││
│  └─────────────────────────────────────────────────────────────────┘│
│                              │                                      │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │                    Kafka Storage                                ││
│  │              _schemas topic (compacted)                         ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Примеры кода

#### Docker Compose для Karapace

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

  karapace-registry:
    image: ghcr.io/aiven/karapace:latest
    ports:
      - "8081:8081"
    depends_on:
      - kafka
    environment:
      KARAPACE_ADVERTISED_HOSTNAME: karapace-registry
      KARAPACE_BOOTSTRAP_URI: kafka:9092
      KARAPACE_PORT: 8081
      KARAPACE_HOST: 0.0.0.0
      KARAPACE_CLIENT_ID: karapace
      KARAPACE_GROUP_ID: karapace-registry
      KARAPACE_MASTER_ELIGIBILITY: "true"
      KARAPACE_TOPIC_NAME: _schemas
      KARAPACE_LOG_LEVEL: INFO
      KARAPACE_COMPATIBILITY: BACKWARD

  karapace-rest:
    image: ghcr.io/aiven/karapace:latest
    ports:
      - "8082:8082"
    depends_on:
      - kafka
      - karapace-registry
    command: ["python", "-m", "karapace.kafka_rest_apis"]
    environment:
      KARAPACE_ADVERTISED_HOSTNAME: karapace-rest
      KARAPACE_BOOTSTRAP_URI: kafka:9092
      KARAPACE_PORT: 8082
      KARAPACE_HOST: 0.0.0.0
      KARAPACE_REGISTRY_HOST: karapace-registry
      KARAPACE_REGISTRY_PORT: 8081
      KARAPACE_LOG_LEVEL: INFO
```

#### Использование REST Proxy

```bash
# Produce сообщение через REST API
curl -X POST "http://localhost:8082/topics/users" \
  -H "Content-Type: application/vnd.kafka.avro.v2+json" \
  --data '{
    "value_schema": "{\"type\":\"record\",\"name\":\"User\",\"fields\":[{\"name\":\"id\",\"type\":\"long\"},{\"name\":\"name\",\"type\":\"string\"}]}",
    "records": [
      {"value": {"id": 1, "name": "Иван"}},
      {"value": {"id": 2, "name": "Мария"}}
    ]
  }'

# Создать consumer group
curl -X POST "http://localhost:8082/consumers/my-group" \
  -H "Content-Type: application/vnd.kafka.v2+json" \
  --data '{
    "name": "my-consumer",
    "format": "avro",
    "auto.offset.reset": "earliest"
  }'

# Подписаться на топик
curl -X POST "http://localhost:8082/consumers/my-group/instances/my-consumer/subscription" \
  -H "Content-Type: application/vnd.kafka.v2+json" \
  --data '{"topics": ["users"]}'

# Получить сообщения
curl "http://localhost:8082/consumers/my-group/instances/my-consumer/records" \
  -H "Accept: application/vnd.kafka.avro.v2+json"
```

#### Python: Использование Karapace

```python
from confluent_kafka import Producer, Consumer
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer, AvroDeserializer

# Karapace полностью совместим с Confluent клиентами
schema_registry_conf = {'url': 'http://localhost:8081'}
client = SchemaRegistryClient(schema_registry_conf)

# Работа идентична Confluent Schema Registry
schema = """
{
    "type": "record",
    "name": "Event",
    "fields": [
        {"name": "id", "type": "string"},
        {"name": "timestamp", "type": "long"},
        {"name": "data", "type": ["null", "string"], "default": null}
    ]
}
"""

serializer = AvroSerializer(client, schema, lambda obj, ctx: obj)

producer = Producer({'bootstrap.servers': 'localhost:9092'})

event = {
    "id": "evt-001",
    "timestamp": 1705312800000,
    "data": "some data"
}

producer.produce(
    'events',
    value=serializer(event, SerializationContext('events', MessageField.VALUE))
)
producer.flush()
```

---

## AWS Glue Schema Registry

### Описание

AWS Glue Schema Registry — это управляемый сервис AWS для хранения и управления схемами. Интегрируется с AWS экосистемой: Kinesis, MSK (Managed Kafka), Glue ETL, Lambda.

### Ключевые особенности

- **Управляемый сервис**: Не требует управления инфраструктурой
- **Интеграция с AWS**: MSK, Kinesis, Glue, Lambda
- **IAM интеграция**: Контроль доступа через AWS IAM
- **Только Avro**: Не поддерживает Protobuf и JSON Schema
- **Оплата по использованию**: Billing за API вызовы

### Архитектура

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AWS Glue Schema Registry                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   AWS Glue Catalog                           │   │
│  │           (Schemas, Versions, Registries)                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                     │
│                              ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Schema Registry API                       │   │
│  │     CreateSchema, RegisterSchemaVersion, GetSchemaVersion    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                     │
│        ┌─────────────────────┼─────────────────────┐              │
│        ▼                     ▼                     ▼              │
│  ┌───────────┐         ┌───────────┐         ┌───────────┐       │
│  │    MSK    │         │  Kinesis  │         │   Glue    │       │
│  │  (Kafka)  │         │  Streams  │         │   Jobs    │       │
│  └───────────┘         └───────────┘         └───────────┘       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Примеры кода

#### Создание реестра и схемы через AWS CLI

```bash
# Создать реестр
aws glue create-registry \
  --registry-name my-registry \
  --description "Реестр для Kafka схем"

# Создать схему
aws glue create-schema \
  --registry-id RegistryName=my-registry \
  --schema-name users \
  --data-format AVRO \
  --compatibility BACKWARD \
  --schema-definition '{
    "type": "record",
    "name": "User",
    "namespace": "com.example",
    "fields": [
      {"name": "id", "type": "long"},
      {"name": "name", "type": "string"}
    ]
  }'

# Зарегистрировать новую версию
aws glue register-schema-version \
  --schema-id SchemaName=users,RegistryName=my-registry \
  --schema-definition '{
    "type": "record",
    "name": "User",
    "namespace": "com.example",
    "fields": [
      {"name": "id", "type": "long"},
      {"name": "name", "type": "string"},
      {"name": "email", "type": ["null", "string"], "default": null}
    ]
  }'

# Получить версию схемы
aws glue get-schema-version \
  --schema-id SchemaName=users,RegistryName=my-registry \
  --schema-version-number LatestVersion
```

#### Python: Использование с Kafka

```python
from aws_schema_registry import DataAndSchema, SchemaRegistryClient
from aws_schema_registry.avro import AvroSchema
from confluent_kafka import Producer, Consumer
import json

# Конфигурация AWS Schema Registry клиента
glue_client = SchemaRegistryClient(
    {
        'region': 'us-east-1',
        'registry': 'my-registry'
    }
)

# Определение схемы
user_schema = AvroSchema("""{
    "type": "record",
    "name": "User",
    "namespace": "com.example",
    "fields": [
        {"name": "id", "type": "long"},
        {"name": "name", "type": "string"}
    ]
}""")

# Producer
producer = Producer({'bootstrap.servers': 'b-1.msk-cluster.kafka.us-east-1.amazonaws.com:9092'})

user_data = {"id": 1, "name": "Иван"}

# Сериализация с Glue Schema Registry
data_and_schema = DataAndSchema(data=user_data, schema=user_schema)
serialized = glue_client.encode(
    schema_name='users',
    data_and_schema=data_and_schema
)

producer.produce('users', value=serialized)
producer.flush()

# Consumer
consumer = Consumer({
    'bootstrap.servers': 'b-1.msk-cluster.kafka.us-east-1.amazonaws.com:9092',
    'group.id': 'my-group',
    'auto.offset.reset': 'earliest'
})
consumer.subscribe(['users'])

msg = consumer.poll(1.0)
if msg:
    # Десериализация
    decoded = glue_client.decode(msg.value())
    print(f"Received: {decoded.data}")
```

#### Java: Использование AWS Glue Schema Registry

```java
import com.amazonaws.services.schemaregistry.serializers.avro.AWSKafkaAvroSerializer;
import com.amazonaws.services.schemaregistry.deserializers.avro.AWSKafkaAvroDeserializer;
import com.amazonaws.services.schemaregistry.utils.AWSSchemaRegistryConstants;
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.clients.consumer.*;
import org.apache.avro.generic.GenericRecord;
import org.apache.avro.generic.GenericData;
import org.apache.avro.Schema;

import java.util.Properties;

public class GlueSchemaRegistryExample {

    public static void main(String[] args) {
        // Producer конфигурация
        Properties producerProps = new Properties();
        producerProps.put("bootstrap.servers", "b-1.msk-cluster.kafka.us-east-1.amazonaws.com:9092");
        producerProps.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        producerProps.put("value.serializer", AWSKafkaAvroSerializer.class.getName());

        // AWS Glue специфичные настройки
        producerProps.put(AWSSchemaRegistryConstants.AWS_REGION, "us-east-1");
        producerProps.put(AWSSchemaRegistryConstants.REGISTRY_NAME, "my-registry");
        producerProps.put(AWSSchemaRegistryConstants.SCHEMA_NAME, "users");
        producerProps.put(AWSSchemaRegistryConstants.COMPATIBILITY_SETTING, "BACKWARD");
        producerProps.put(AWSSchemaRegistryConstants.SCHEMA_AUTO_REGISTRATION_SETTING, true);

        // Создание записи
        String schemaString = """
            {
                "type": "record",
                "name": "User",
                "namespace": "com.example",
                "fields": [
                    {"name": "id", "type": "long"},
                    {"name": "name", "type": "string"}
                ]
            }
            """;

        Schema schema = new Schema.Parser().parse(schemaString);
        GenericRecord user = new GenericData.Record(schema);
        user.put("id", 1L);
        user.put("name", "Иван");

        try (Producer<String, GenericRecord> producer = new KafkaProducer<>(producerProps)) {
            ProducerRecord<String, GenericRecord> record = new ProducerRecord<>("users", "1", user);
            producer.send(record).get();
            System.out.println("Sent successfully");
        } catch (Exception e) {
            e.printStackTrace();
        }

        // Consumer конфигурация
        Properties consumerProps = new Properties();
        consumerProps.put("bootstrap.servers", "b-1.msk-cluster.kafka.us-east-1.amazonaws.com:9092");
        consumerProps.put("group.id", "my-consumer-group");
        consumerProps.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        consumerProps.put("value.deserializer", AWSKafkaAvroDeserializer.class.getName());
        consumerProps.put(AWSSchemaRegistryConstants.AWS_REGION, "us-east-1");
        consumerProps.put(AWSSchemaRegistryConstants.REGISTRY_NAME, "my-registry");

        try (Consumer<String, GenericRecord> consumer = new KafkaConsumer<>(consumerProps)) {
            consumer.subscribe(Collections.singletonList("users"));

            ConsumerRecords<String, GenericRecord> records = consumer.poll(Duration.ofSeconds(10));
            for (ConsumerRecord<String, GenericRecord> record : records) {
                System.out.printf("User: %s%n", record.value().get("name"));
            }
        }
    }
}
```

---

## Схемы без реестра

### Описание

В некоторых случаях можно обойтись без централизованного реестра схем, используя альтернативные подходы к управлению схемами.

### Подходы

#### 1. Схемы в Git репозитории

```
schemas/
├── avro/
│   ├── users.avsc
│   ├── orders.avsc
│   └── events.avsc
├── protobuf/
│   ├── users.proto
│   └── orders.proto
└── json-schema/
    └── config.json
```

```python
# Загрузка схемы из файла
import json
from pathlib import Path

def load_avro_schema(schema_name: str) -> str:
    schema_path = Path("schemas/avro") / f"{schema_name}.avsc"
    with open(schema_path) as f:
        return f.read()

# Использование
from confluent_kafka.schema_registry.avro import AvroSerializer

user_schema = load_avro_schema("users")
serializer = AvroSerializer(None, user_schema)  # Без Schema Registry
```

#### 2. Схемы как часть сообщения

```python
import json
from confluent_kafka import Producer

# Включение схемы в каждое сообщение (не рекомендуется для production)
message = {
    "_schema": {
        "type": "record",
        "name": "User",
        "fields": [
            {"name": "id", "type": "long"},
            {"name": "name", "type": "string"}
        ]
    },
    "data": {
        "id": 1,
        "name": "Иван"
    }
}

producer = Producer({'bootstrap.servers': 'localhost:9092'})
producer.produce('users', value=json.dumps(message).encode())
```

#### 3. Версионирование топиков

```
# Разные топики для разных версий схемы
users-v1   # Старая схема
users-v2   # Новая схема
users-v3   # Ещё более новая схема

# Или с namespace
com.example.users.v1
com.example.users.v2
```

#### 4. Header-based версионирование

```python
from confluent_kafka import Producer
import json

producer = Producer({'bootstrap.servers': 'localhost:9092'})

# Версия схемы в headers
headers = [
    ('schema-version', b'2'),
    ('schema-name', b'users')
]

message = json.dumps({"id": 1, "name": "Иван", "email": "ivan@example.com"})

producer.produce(
    'users',
    value=message.encode(),
    headers=headers
)
```

### CI/CD валидация без реестра

```yaml
# .github/workflows/schema-validation.yml
name: Schema Validation

on:
  pull_request:
    paths:
      - 'schemas/**'

jobs:
  validate-avro:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 2

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install avro-python3 jsonschema

      - name: Validate Avro schemas
        run: |
          python - << 'EOF'
          import json
          import avro.schema
          from pathlib import Path

          for schema_file in Path('schemas/avro').glob('*.avsc'):
              print(f"Validating {schema_file}...")
              with open(schema_file) as f:
                  schema_str = f.read()
              try:
                  avro.schema.parse(schema_str)
                  print(f"  ✓ Valid")
              except Exception as e:
                  print(f"  ✗ Invalid: {e}")
                  exit(1)
          EOF

      - name: Check backward compatibility
        run: |
          python - << 'EOF'
          import subprocess
          import json
          from pathlib import Path

          # Получаем изменённые файлы
          result = subprocess.run(
              ['git', 'diff', '--name-only', 'HEAD~1'],
              capture_output=True, text=True
          )
          changed_files = result.stdout.strip().split('\n')

          for file in changed_files:
              if file.startswith('schemas/avro/') and file.endswith('.avsc'):
                  # Получаем предыдущую версию
                  old_result = subprocess.run(
                      ['git', 'show', f'HEAD~1:{file}'],
                      capture_output=True, text=True
                  )

                  if old_result.returncode == 0:
                      # Проверяем совместимость
                      # (здесь нужна реализация проверки совместимости)
                      print(f"Checking compatibility for {file}")
          EOF
```

---

## Best Practices

### Выбор решения

| Критерий | Рекомендация |
|----------|--------------|
| Production с поддержкой | Confluent Schema Registry |
| OpenAPI/GraphQL схемы | Apicurio Registry |
| Экономия бюджета | Karapace или Apicurio |
| AWS native | AWS Glue Schema Registry |
| Простые проекты | Схемы в Git |
| Микросервисы с разными языками | Protobuf + любой реестр |

### Миграция между решениями

```python
# Скрипт миграции схем из Confluent в Apicurio
import requests

confluent_url = "http://confluent-sr:8081"
apicurio_url = "http://apicurio:8080/apis/registry/v2"

# Получить все субъекты из Confluent
subjects = requests.get(f"{confluent_url}/subjects").json()

for subject in subjects:
    # Получить схему
    schema_resp = requests.get(f"{confluent_url}/subjects/{subject}/versions/latest")
    schema_data = schema_resp.json()

    # Загрузить в Apicurio
    requests.post(
        f"{apicurio_url}/groups/default/artifacts",
        headers={
            "Content-Type": "application/json; artifactType=AVRO",
            "X-Registry-ArtifactId": subject
        },
        data=schema_data['schema']
    )
    print(f"Migrated: {subject}")
```

### Типичные ошибки при выборе

1. **Выбор AWS Glue для Protobuf** — не поддерживается
2. **Недооценка операционных затрат** — self-hosted требует поддержки
3. **Игнорирование совместимости API** — миграция может быть сложной
4. **Выбор решения без учёта роста** — масштабируемость важна

---

[prev: 04-compatibility-rules](./04-compatibility-rules.md) | [next: 01-kafka-streams](../12-stream-processing/01-kafka-streams.md)
