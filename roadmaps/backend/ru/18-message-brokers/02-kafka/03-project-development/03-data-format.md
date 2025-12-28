# Формат представления данных

[prev: 02-sensor-events](./02-sensor-events.md) | [next: 01-example](../04-producers/01-example.md)

---

## Описание

Выбор формата сериализации данных в Apache Kafka критически важен для производительности, совместимости и масштабируемости системы. В этом разделе мы рассмотрим основные форматы данных: JSON, Apache Avro, Protocol Buffers и их применение в реальных проектах.

## Сравнение форматов сериализации

| Формат | Размер | Скорость | Типизация | Схема | Читаемость |
|--------|--------|----------|-----------|-------|------------|
| JSON | Большой | Средняя | Динамическая | Опционально | Высокая |
| Avro | Маленький | Высокая | Строгая | Обязательна | Низкая |
| Protobuf | Маленький | Очень высокая | Строгая | Обязательна | Низкая |
| MessagePack | Маленький | Высокая | Динамическая | Нет | Низкая |

## JSON Сериализация

### Описание

JSON - самый распространенный формат для начала работы с Kafka благодаря простоте и читаемости. Однако он имеет недостатки: большой размер сообщений и отсутствие встроенной валидации схемы.

### Реализация JSON Serializer

```python
# src/serializers/json_serializer.py
import json
from typing import Any, Optional, Type, TypeVar
from dataclasses import asdict, is_dataclass
from datetime import datetime, date
from enum import Enum
import logging

logger = logging.getLogger(__name__)

T = TypeVar('T')


class CustomJSONEncoder(json.JSONEncoder):
    """Кастомный JSON энкодер с поддержкой дополнительных типов."""

    def default(self, obj: Any) -> Any:
        if isinstance(obj, datetime):
            return obj.isoformat()
        if isinstance(obj, date):
            return obj.isoformat()
        if isinstance(obj, Enum):
            return obj.value
        if is_dataclass(obj):
            return asdict(obj)
        if hasattr(obj, 'to_dict'):
            return obj.to_dict()
        return super().default(obj)


class JSONSerializer:
    """Сериализатор для JSON формата."""

    def __init__(self, encoding: str = "utf-8"):
        self.encoding = encoding
        self.encoder = CustomJSONEncoder

    def serialize(self, data: Any) -> bytes:
        """Сериализация объекта в JSON bytes."""
        try:
            json_str = json.dumps(data, cls=self.encoder, ensure_ascii=False)
            return json_str.encode(self.encoding)
        except (TypeError, ValueError) as e:
            logger.error(f"Ошибка сериализации JSON: {e}")
            raise

    def deserialize(self, data: bytes, target_class: Optional[Type[T]] = None) -> Any:
        """Десериализация JSON bytes в объект."""
        try:
            parsed = json.loads(data.decode(self.encoding))

            if target_class and hasattr(target_class, 'from_dict'):
                return target_class.from_dict(parsed)

            return parsed
        except (json.JSONDecodeError, UnicodeDecodeError) as e:
            logger.error(f"Ошибка десериализации JSON: {e}")
            raise


class KafkaJSONSerializer:
    """Сериализатор для использования с confluent-kafka."""

    def __init__(self):
        self.serializer = JSONSerializer()

    def __call__(self, obj: Any, ctx=None) -> bytes:
        """Callable для использования в Producer/Consumer."""
        if obj is None:
            return None
        return self.serializer.serialize(obj)


class KafkaJSONDeserializer:
    """Десериализатор для использования с confluent-kafka."""

    def __init__(self, target_class: Optional[Type] = None):
        self.serializer = JSONSerializer()
        self.target_class = target_class

    def __call__(self, data: bytes, ctx=None) -> Any:
        """Callable для использования в Consumer."""
        if data is None:
            return None
        return self.serializer.deserialize(data, self.target_class)
```

### Использование JSON с Producer/Consumer

```python
# Пример использования JSON сериализации
from confluent_kafka import Producer, Consumer
from src.serializers.json_serializer import JSONSerializer

serializer = JSONSerializer()

# Producer
producer = Producer({"bootstrap.servers": "localhost:9092"})

event = {
    "sensor_id": "temp-001",
    "value": 23.5,
    "timestamp": datetime.utcnow().isoformat()
}

producer.produce(
    topic="sensor-events",
    key=b"temp-001",
    value=serializer.serialize(event)
)
producer.flush()

# Consumer
consumer = Consumer({
    "bootstrap.servers": "localhost:9092",
    "group.id": "json-consumer",
    "auto.offset.reset": "earliest"
})
consumer.subscribe(["sensor-events"])

msg = consumer.poll(1.0)
if msg and not msg.error():
    data = serializer.deserialize(msg.value())
    print(f"Получено: {data}")
```

## Apache Avro

### Описание

Avro - это компактный бинарный формат с поддержкой схем. Схемы хранятся отдельно от данных, что обеспечивает эффективную сериализацию и эволюцию схем.

### Установка зависимостей

```bash
pip install fastavro confluent-kafka[avro]
```

### Определение Avro схемы

```python
# src/schemas/sensor_event.avsc
SENSOR_EVENT_SCHEMA = {
    "type": "record",
    "name": "SensorEvent",
    "namespace": "com.example.sensors",
    "fields": [
        {"name": "event_id", "type": "string"},
        {"name": "sensor_id", "type": "string"},
        {"name": "sensor_type", "type": {
            "type": "enum",
            "name": "SensorType",
            "symbols": ["TEMPERATURE", "HUMIDITY", "PRESSURE", "CO2", "LIGHT"]
        }},
        {"name": "value", "type": "double"},
        {"name": "unit", "type": "string"},
        {"name": "timestamp", "type": {
            "type": "long",
            "logicalType": "timestamp-millis"
        }},
        {"name": "location", "type": ["null", "string"], "default": None},
        {"name": "metadata", "type": {
            "type": "map",
            "values": "string"
        }, "default": {}}
    ]
}
```

### Avro Serializer

```python
# src/serializers/avro_serializer.py
import io
from typing import Any, Dict, Optional
from fastavro import writer, reader, parse_schema
from fastavro.schema import load_schema
import logging

logger = logging.getLogger(__name__)


class AvroSerializer:
    """Сериализатор для Apache Avro формата."""

    def __init__(self, schema: Dict):
        self.schema = parse_schema(schema)

    def serialize(self, data: Dict) -> bytes:
        """Сериализация словаря в Avro bytes."""
        buffer = io.BytesIO()
        try:
            writer(buffer, self.schema, [data])
            return buffer.getvalue()
        except Exception as e:
            logger.error(f"Ошибка сериализации Avro: {e}")
            raise

    def deserialize(self, data: bytes) -> Dict:
        """Десериализация Avro bytes в словарь."""
        buffer = io.BytesIO(data)
        try:
            records = list(reader(buffer, self.schema))
            return records[0] if records else {}
        except Exception as e:
            logger.error(f"Ошибка десериализации Avro: {e}")
            raise


class SchemaRegistryAvroSerializer:
    """Avro сериализатор с поддержкой Schema Registry."""

    def __init__(
        self,
        schema_registry_url: str,
        schema: Dict,
        subject_name: str
    ):
        from confluent_kafka.schema_registry import SchemaRegistryClient
        from confluent_kafka.schema_registry.avro import AvroSerializer as ConfluentAvroSerializer

        self.schema_registry_client = SchemaRegistryClient({
            "url": schema_registry_url
        })

        self.serializer = ConfluentAvroSerializer(
            schema_registry_client=self.schema_registry_client,
            schema_str=str(schema),
            to_dict=lambda obj, ctx: obj if isinstance(obj, dict) else obj.to_dict()
        )

    def __call__(self, obj: Any, ctx=None) -> bytes:
        """Сериализация объекта."""
        return self.serializer(obj, ctx)


class SchemaRegistryAvroDeserializer:
    """Avro десериализатор с поддержкой Schema Registry."""

    def __init__(self, schema_registry_url: str):
        from confluent_kafka.schema_registry import SchemaRegistryClient
        from confluent_kafka.schema_registry.avro import AvroDeserializer as ConfluentAvroDeserializer

        self.schema_registry_client = SchemaRegistryClient({
            "url": schema_registry_url
        })

        self.deserializer = ConfluentAvroDeserializer(
            schema_registry_client=self.schema_registry_client
        )

    def __call__(self, data: bytes, ctx=None) -> Dict:
        """Десериализация данных."""
        return self.deserializer(data, ctx)
```

### Пример использования Avro

```python
# Пример с Avro и Schema Registry
from confluent_kafka import Producer, Consumer
from confluent_kafka.serialization import SerializationContext, MessageField

# Схема
schema = {
    "type": "record",
    "name": "SensorEvent",
    "fields": [
        {"name": "sensor_id", "type": "string"},
        {"name": "value", "type": "double"},
        {"name": "timestamp", "type": "long"}
    ]
}

# Инициализация с Schema Registry
serializer = SchemaRegistryAvroSerializer(
    schema_registry_url="http://localhost:8081",
    schema=schema,
    subject_name="sensor-events-value"
)

# Producer
producer = Producer({"bootstrap.servers": "localhost:9092"})

event = {
    "sensor_id": "temp-001",
    "value": 23.5,
    "timestamp": int(datetime.utcnow().timestamp() * 1000)
}

producer.produce(
    topic="sensor-events",
    key=b"temp-001",
    value=serializer(event, SerializationContext("sensor-events", MessageField.VALUE))
)
producer.flush()
```

## Protocol Buffers (Protobuf)

### Описание

Protocol Buffers - это бинарный формат от Google, обеспечивающий максимальную скорость сериализации и минимальный размер сообщений.

### Установка

```bash
pip install protobuf grpcio-tools
```

### Определение .proto файла

```protobuf
// src/proto/sensor_event.proto
syntax = "proto3";

package sensors;

enum SensorType {
    UNKNOWN = 0;
    TEMPERATURE = 1;
    HUMIDITY = 2;
    PRESSURE = 3;
    CO2 = 4;
    LIGHT = 5;
}

message SensorEvent {
    string event_id = 1;
    string sensor_id = 2;
    SensorType sensor_type = 3;
    double value = 4;
    string unit = 5;
    int64 timestamp = 6;
    optional string location = 7;
    map<string, string> metadata = 8;
}

message SensorEventBatch {
    repeated SensorEvent events = 1;
}
```

### Генерация Python классов

```bash
# Генерация Python файлов из .proto
python -m grpc_tools.protoc \
    -I./src/proto \
    --python_out=./src/proto \
    ./src/proto/sensor_event.proto
```

### Protobuf Serializer

```python
# src/serializers/protobuf_serializer.py
from typing import Type, TypeVar
from google.protobuf.message import Message
import logging

logger = logging.getLogger(__name__)

T = TypeVar('T', bound=Message)


class ProtobufSerializer:
    """Сериализатор для Protocol Buffers."""

    def serialize(self, message: Message) -> bytes:
        """Сериализация protobuf сообщения в bytes."""
        try:
            return message.SerializeToString()
        except Exception as e:
            logger.error(f"Ошибка сериализации Protobuf: {e}")
            raise

    def deserialize(self, data: bytes, message_class: Type[T]) -> T:
        """Десериализация bytes в protobuf сообщение."""
        try:
            message = message_class()
            message.ParseFromString(data)
            return message
        except Exception as e:
            logger.error(f"Ошибка десериализации Protobuf: {e}")
            raise


class KafkaProtobufSerializer:
    """Kafka-совместимый Protobuf сериализатор."""

    def __init__(self):
        self.serializer = ProtobufSerializer()

    def __call__(self, obj: Message, ctx=None) -> bytes:
        """Сериализация для Kafka."""
        if obj is None:
            return None
        return self.serializer.serialize(obj)


class KafkaProtobufDeserializer:
    """Kafka-совместимый Protobuf десериализатор."""

    def __init__(self, message_class: Type[Message]):
        self.serializer = ProtobufSerializer()
        self.message_class = message_class

    def __call__(self, data: bytes, ctx=None) -> Message:
        """Десериализация из Kafka."""
        if data is None:
            return None
        return self.serializer.deserialize(data, self.message_class)
```

### Пример использования Protobuf

```python
# Пример с Protobuf
from src.proto import sensor_event_pb2
from src.serializers.protobuf_serializer import ProtobufSerializer
from confluent_kafka import Producer
import time

serializer = ProtobufSerializer()

# Создание события
event = sensor_event_pb2.SensorEvent(
    event_id="evt-001",
    sensor_id="temp-001",
    sensor_type=sensor_event_pb2.TEMPERATURE,
    value=23.5,
    unit="°C",
    timestamp=int(time.time() * 1000),
    location="Building A"
)
event.metadata["firmware"] = "1.2.3"

# Сериализация
data = serializer.serialize(event)
print(f"Размер сообщения: {len(data)} bytes")

# Десериализация
decoded = serializer.deserialize(data, sensor_event_pb2.SensorEvent)
print(f"Sensor ID: {decoded.sensor_id}, Value: {decoded.value}")

# Отправка в Kafka
producer = Producer({"bootstrap.servers": "localhost:9092"})
producer.produce(
    topic="sensor-events-proto",
    key=b"temp-001",
    value=data
)
producer.flush()
```

## Schema Registry

### Описание

Schema Registry - это централизованное хранилище схем, которое обеспечивает совместимость версий и валидацию данных.

### Конфигурация Schema Registry в Docker

```yaml
# docker/docker-compose.yml (дополнение)
services:
  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.0
    hostname: schema-registry
    container_name: schema-registry
    depends_on:
      - kafka
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:29092
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
      SCHEMA_REGISTRY_SCHEMA_COMPATIBILITY_LEVEL: BACKWARD
```

### Работа с Schema Registry

```python
# src/schemas/registry.py
import requests
from typing import Dict, Optional, List
import json
import logging

logger = logging.getLogger(__name__)


class SchemaRegistryClient:
    """Клиент для работы с Schema Registry."""

    def __init__(self, base_url: str = "http://localhost:8081"):
        self.base_url = base_url.rstrip("/")

    def register_schema(
        self,
        subject: str,
        schema: Dict,
        schema_type: str = "AVRO"
    ) -> int:
        """Регистрация новой схемы."""
        url = f"{self.base_url}/subjects/{subject}/versions"

        payload = {
            "schemaType": schema_type,
            "schema": json.dumps(schema) if isinstance(schema, dict) else schema
        }

        response = requests.post(url, json=payload)
        response.raise_for_status()

        schema_id = response.json()["id"]
        logger.info(f"Схема зарегистрирована: subject={subject}, id={schema_id}")

        return schema_id

    def get_schema(self, schema_id: int) -> Dict:
        """Получение схемы по ID."""
        url = f"{self.base_url}/schemas/ids/{schema_id}"

        response = requests.get(url)
        response.raise_for_status()

        return json.loads(response.json()["schema"])

    def get_latest_schema(self, subject: str) -> Dict:
        """Получение последней версии схемы."""
        url = f"{self.base_url}/subjects/{subject}/versions/latest"

        response = requests.get(url)
        response.raise_for_status()

        result = response.json()
        return {
            "id": result["id"],
            "version": result["version"],
            "schema": json.loads(result["schema"])
        }

    def get_all_subjects(self) -> List[str]:
        """Получение списка всех subjects."""
        url = f"{self.base_url}/subjects"

        response = requests.get(url)
        response.raise_for_status()

        return response.json()

    def check_compatibility(
        self,
        subject: str,
        schema: Dict,
        version: str = "latest"
    ) -> bool:
        """Проверка совместимости схемы."""
        url = f"{self.base_url}/compatibility/subjects/{subject}/versions/{version}"

        payload = {
            "schema": json.dumps(schema) if isinstance(schema, dict) else schema
        }

        response = requests.post(url, json=payload)

        if response.status_code == 200:
            return response.json().get("is_compatible", False)
        return False

    def set_compatibility(
        self,
        subject: str,
        level: str = "BACKWARD"
    ) -> None:
        """Установка уровня совместимости для subject."""
        url = f"{self.base_url}/config/{subject}"

        payload = {"compatibility": level}

        response = requests.put(url, json=payload)
        response.raise_for_status()

        logger.info(f"Уровень совместимости для {subject} установлен: {level}")


# Пример использования
def setup_schemas():
    """Регистрация схем при запуске приложения."""
    client = SchemaRegistryClient("http://localhost:8081")

    # Регистрация схемы для событий датчиков
    sensor_schema = {
        "type": "record",
        "name": "SensorEvent",
        "namespace": "com.example.sensors",
        "fields": [
            {"name": "event_id", "type": "string"},
            {"name": "sensor_id", "type": "string"},
            {"name": "value", "type": "double"},
            {"name": "timestamp", "type": "long"}
        ]
    }

    try:
        schema_id = client.register_schema(
            subject="sensor-events-value",
            schema=sensor_schema
        )
        print(f"Схема зарегистрирована с ID: {schema_id}")

        # Проверка совместимости обновленной схемы
        updated_schema = {
            **sensor_schema,
            "fields": sensor_schema["fields"] + [
                {"name": "unit", "type": ["null", "string"], "default": None}
            ]
        }

        is_compatible = client.check_compatibility(
            subject="sensor-events-value",
            schema=updated_schema
        )
        print(f"Новая схема совместима: {is_compatible}")

    except Exception as e:
        print(f"Ошибка: {e}")
```

## Best Practices

### Выбор формата

1. **JSON**: Используйте для прототипирования и случаев, когда важна читаемость
2. **Avro**: Рекомендуется для большинства продакшен систем с Schema Registry
3. **Protobuf**: Используйте при критических требованиях к производительности

### Schema Evolution

1. **Добавление полей**: Всегда добавляйте значения по умолчанию
2. **Удаление полей**: Используйте backward-compatible удаление
3. **Изменение типов**: Избегайте, если возможно
4. **Переименование**: Используйте aliases в Avro

### Совместимость схем

```
BACKWARD  - новые схемы могут читать старые данные
FORWARD   - старые схемы могут читать новые данные
FULL      - BACKWARD + FORWARD
NONE      - совместимость не проверяется
```

### Производительность

1. **Кэширование схем**: Используйте локальный кэш для Schema Registry
2. **Batch сериализация**: Группируйте сообщения для эффективности
3. **Compression**: Используйте сжатие для больших сообщений

### Безопасность

1. **Валидация**: Всегда валидируйте данные перед отправкой
2. **Санитизация**: Очищайте входные данные
3. **Шифрование**: Используйте encryption-at-rest для чувствительных данных

## Сравнение производительности

```python
# Бенчмарк сериализации
import time

def benchmark_serialization(serializer, data, iterations=10000):
    """Бенчмарк сериализатора."""
    start = time.time()
    for _ in range(iterations):
        serialized = serializer.serialize(data)
    elapsed = time.time() - start

    return {
        "total_time": elapsed,
        "ops_per_sec": iterations / elapsed,
        "message_size": len(serialized)
    }

# Типичные результаты (примерные):
# JSON:     ~50,000 ops/sec, ~200 bytes
# Avro:     ~150,000 ops/sec, ~80 bytes
# Protobuf: ~300,000 ops/sec, ~60 bytes
```

## Дополнительные ресурсы

- [Apache Avro Specification](https://avro.apache.org/docs/current/spec.html)
- [Protocol Buffers Guide](https://developers.google.com/protocol-buffers/docs/pythontutorial)
- [Confluent Schema Registry](https://docs.confluent.io/platform/current/schema-registry/index.html)
- [Schema Evolution Best Practices](https://docs.confluent.io/platform/current/schema-registry/avro.html)

---

[prev: 02-sensor-events](./02-sensor-events.md) | [next: 01-example](../04-producers/01-example.md)
