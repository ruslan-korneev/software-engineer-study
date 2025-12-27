# Компоненты реестра схем

## Описание

Schema Registry поддерживает три основных формата сериализации данных: Apache Avro, Protocol Buffers (Protobuf) и JSON Schema. Каждый формат имеет свои преимущества и особенности использования. Выбор формата зависит от требований проекта: производительности, совместимости с существующими системами, удобства разработки и экосистемы инструментов.

## Ключевые концепции

### Сравнение форматов

| Характеристика | Avro | Protobuf | JSON Schema |
|----------------|------|----------|-------------|
| Формат данных | Бинарный | Бинарный | Текстовый (JSON) |
| Размер сообщения | Компактный | Очень компактный | Большой |
| Скорость сериализации | Высокая | Очень высокая | Средняя |
| Читаемость | Требует схему | Требует схему | Читаемый |
| Эволюция схем | Отличная | Хорошая | Хорошая |
| Кодогенерация | Опциональная | Обязательная | Опциональная |
| Экосистема Kafka | Нативная | Полная поддержка | Полная поддержка |
| Языковая поддержка | JVM, Python, C | Широкая (30+ языков) | Универсальная |

### Когда использовать какой формат

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Выбор формата сериализации                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐   Hadoop/Spark экосистема                         │
│  │    Avro     │   Динамические схемы                               │
│  │             │   Нужна генерация схем из данных                   │
│  └─────────────┘                                                    │
│                                                                     │
│  ┌─────────────┐   Микросервисы с разными языками                  │
│  │  Protobuf   │   Максимальная производительность                  │
│  │             │   gRPC интеграция                                  │
│  └─────────────┘                                                    │
│                                                                     │
│  ┌─────────────┐   Простая интеграция                              │
│  │ JSON Schema │   Отладка и мониторинг                             │
│  │             │   REST API совместимость                           │
│  └─────────────┘                                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Apache Avro

### Описание

Apache Avro — это система сериализации данных, разработанная в рамках проекта Apache Hadoop. Avro использует JSON для определения схем и бинарный формат для хранения данных. Ключевая особенность — схема хранится вместе с данными или отдельно в Schema Registry, что позволяет динамически работать со схемами без кодогенерации.

### Структура схемы Avro

```json
{
    "type": "record",
    "name": "Order",
    "namespace": "com.example.events",
    "doc": "Событие создания заказа",
    "fields": [
        {
            "name": "order_id",
            "type": "string",
            "doc": "Уникальный идентификатор заказа"
        },
        {
            "name": "customer_id",
            "type": "long"
        },
        {
            "name": "amount",
            "type": {
                "type": "bytes",
                "logicalType": "decimal",
                "precision": 10,
                "scale": 2
            }
        },
        {
            "name": "status",
            "type": {
                "type": "enum",
                "name": "OrderStatus",
                "symbols": ["PENDING", "CONFIRMED", "SHIPPED", "DELIVERED", "CANCELLED"]
            }
        },
        {
            "name": "items",
            "type": {
                "type": "array",
                "items": {
                    "type": "record",
                    "name": "OrderItem",
                    "fields": [
                        {"name": "product_id", "type": "string"},
                        {"name": "quantity", "type": "int"},
                        {"name": "price", "type": "double"}
                    ]
                }
            }
        },
        {
            "name": "shipping_address",
            "type": ["null", {
                "type": "record",
                "name": "Address",
                "fields": [
                    {"name": "street", "type": "string"},
                    {"name": "city", "type": "string"},
                    {"name": "country", "type": "string"},
                    {"name": "postal_code", "type": "string"}
                ]
            }],
            "default": null
        },
        {
            "name": "created_at",
            "type": {
                "type": "long",
                "logicalType": "timestamp-millis"
            }
        },
        {
            "name": "metadata",
            "type": ["null", {"type": "map", "values": "string"}],
            "default": null
        }
    ]
}
```

### Типы данных Avro

#### Примитивные типы

| Тип | Описание | Пример |
|-----|----------|--------|
| `null` | Отсутствие значения | `null` |
| `boolean` | Булево значение | `true`, `false` |
| `int` | 32-битное целое | `42` |
| `long` | 64-битное целое | `1234567890123` |
| `float` | 32-битное с плавающей точкой | `3.14` |
| `double` | 64-битное с плавающей точкой | `3.14159265359` |
| `bytes` | Последовательность байт | `"\u00FF\u00FE"` |
| `string` | Строка Unicode | `"Привет"` |

#### Сложные типы

```json
// Record - структура с именованными полями
{
    "type": "record",
    "name": "User",
    "fields": [
        {"name": "id", "type": "long"},
        {"name": "name", "type": "string"}
    ]
}

// Enum - перечисление
{
    "type": "enum",
    "name": "Color",
    "symbols": ["RED", "GREEN", "BLUE"]
}

// Array - массив
{
    "type": "array",
    "items": "string"
}

// Map - словарь со строковыми ключами
{
    "type": "map",
    "values": "long"
}

// Union - объединение типов (nullable поле)
["null", "string"]

// Fixed - фиксированное количество байт
{
    "type": "fixed",
    "name": "UUID",
    "size": 16
}
```

#### Логические типы

```json
// Дата (количество дней с 1970-01-01)
{"type": "int", "logicalType": "date"}

// Время в миллисекундах
{"type": "int", "logicalType": "time-millis"}

// Время в микросекундах
{"type": "long", "logicalType": "time-micros"}

// Timestamp в миллисекундах
{"type": "long", "logicalType": "timestamp-millis"}

// Timestamp в микросекундах
{"type": "long", "logicalType": "timestamp-micros"}

// Decimal с фиксированной точностью
{
    "type": "bytes",
    "logicalType": "decimal",
    "precision": 10,
    "scale": 2
}

// UUID
{"type": "string", "logicalType": "uuid"}
```

### Примеры кода Avro

#### Python: Работа с Avro

```python
from confluent_kafka import Producer, Consumer
from confluent_kafka.serialization import SerializationContext, MessageField
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer, AvroDeserializer
from dataclasses import dataclass
from typing import Optional, List, Dict
from datetime import datetime
from decimal import Decimal

# Определение классов данных
@dataclass
class OrderItem:
    product_id: str
    quantity: int
    price: float

@dataclass
class Address:
    street: str
    city: str
    country: str
    postal_code: str

@dataclass
class Order:
    order_id: str
    customer_id: int
    amount: Decimal
    status: str
    items: List[OrderItem]
    shipping_address: Optional[Address]
    created_at: datetime
    metadata: Optional[Dict[str, str]] = None

# Схема Avro
order_schema = """
{
    "type": "record",
    "name": "Order",
    "namespace": "com.example.events",
    "fields": [
        {"name": "order_id", "type": "string"},
        {"name": "customer_id", "type": "long"},
        {"name": "amount", "type": {"type": "bytes", "logicalType": "decimal", "precision": 10, "scale": 2}},
        {"name": "status", "type": {"type": "enum", "name": "OrderStatus", "symbols": ["PENDING", "CONFIRMED", "SHIPPED", "DELIVERED", "CANCELLED"]}},
        {"name": "items", "type": {"type": "array", "items": {"type": "record", "name": "OrderItem", "fields": [{"name": "product_id", "type": "string"}, {"name": "quantity", "type": "int"}, {"name": "price", "type": "double"}]}}},
        {"name": "shipping_address", "type": ["null", {"type": "record", "name": "Address", "fields": [{"name": "street", "type": "string"}, {"name": "city", "type": "string"}, {"name": "country", "type": "string"}, {"name": "postal_code", "type": "string"}]}], "default": null},
        {"name": "created_at", "type": {"type": "long", "logicalType": "timestamp-millis"}},
        {"name": "metadata", "type": ["null", {"type": "map", "values": "string"}], "default": null}
    ]
}
"""

def order_to_dict(order: Order, ctx) -> dict:
    """Преобразование Order в словарь для сериализации"""
    return {
        "order_id": order.order_id,
        "customer_id": order.customer_id,
        "amount": str(order.amount).encode(),  # Decimal как bytes
        "status": order.status,
        "items": [
            {"product_id": item.product_id, "quantity": item.quantity, "price": item.price}
            for item in order.items
        ],
        "shipping_address": {
            "street": order.shipping_address.street,
            "city": order.shipping_address.city,
            "country": order.shipping_address.country,
            "postal_code": order.shipping_address.postal_code
        } if order.shipping_address else None,
        "created_at": int(order.created_at.timestamp() * 1000),
        "metadata": order.metadata
    }

def dict_to_order(obj: dict, ctx) -> Order:
    """Преобразование словаря в Order при десериализации"""
    if obj is None:
        return None

    items = [
        OrderItem(
            product_id=item["product_id"],
            quantity=item["quantity"],
            price=item["price"]
        )
        for item in obj["items"]
    ]

    address = None
    if obj.get("shipping_address"):
        addr = obj["shipping_address"]
        address = Address(
            street=addr["street"],
            city=addr["city"],
            country=addr["country"],
            postal_code=addr["postal_code"]
        )

    return Order(
        order_id=obj["order_id"],
        customer_id=obj["customer_id"],
        amount=Decimal(obj["amount"].decode() if isinstance(obj["amount"], bytes) else str(obj["amount"])),
        status=obj["status"],
        items=items,
        shipping_address=address,
        created_at=datetime.fromtimestamp(obj["created_at"] / 1000),
        metadata=obj.get("metadata")
    )

# Настройка Schema Registry
schema_registry_client = SchemaRegistryClient({'url': 'http://localhost:8081'})

# Создание сериализатора и десериализатора
avro_serializer = AvroSerializer(schema_registry_client, order_schema, order_to_dict)
avro_deserializer = AvroDeserializer(schema_registry_client, order_schema, dict_to_order)

# Пример использования Producer
producer = Producer({'bootstrap.servers': 'localhost:9092'})

order = Order(
    order_id="ORD-001",
    customer_id=12345,
    amount=Decimal("99.99"),
    status="PENDING",
    items=[
        OrderItem("PROD-1", 2, 29.99),
        OrderItem("PROD-2", 1, 39.99)
    ],
    shipping_address=Address("ул. Ленина, 1", "Москва", "Россия", "101000"),
    created_at=datetime.now(),
    metadata={"source": "web", "campaign": "summer_sale"}
)

producer.produce(
    topic='orders',
    key=order.order_id,
    value=avro_serializer(order, SerializationContext('orders', MessageField.VALUE))
)
producer.flush()
print(f"Заказ {order.order_id} отправлен")
```

#### Java: Generic и Specific Records

```java
import org.apache.avro.Schema;
import org.apache.avro.generic.GenericData;
import org.apache.avro.generic.GenericRecord;
import io.confluent.kafka.serializers.KafkaAvroSerializer;
import org.apache.kafka.clients.producer.*;

import java.util.Properties;

public class AvroGenericExample {

    private static final String SCHEMA_STRING = """
        {
            "type": "record",
            "name": "User",
            "namespace": "com.example",
            "fields": [
                {"name": "id", "type": "long"},
                {"name": "name", "type": "string"},
                {"name": "email", "type": ["null", "string"], "default": null}
            ]
        }
        """;

    public static void main(String[] args) throws Exception {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", KafkaAvroSerializer.class.getName());
        props.put("schema.registry.url", "http://localhost:8081");

        Schema.Parser parser = new Schema.Parser();
        Schema schema = parser.parse(SCHEMA_STRING);

        // Создание Generic Record (без кодогенерации)
        GenericRecord user = new GenericData.Record(schema);
        user.put("id", 1L);
        user.put("name", "Иван Петров");
        user.put("email", "ivan@example.com");

        try (Producer<String, GenericRecord> producer = new KafkaProducer<>(props)) {
            ProducerRecord<String, GenericRecord> record =
                new ProducerRecord<>("users", "1", user);
            producer.send(record).get();
            System.out.println("Пользователь отправлен");
        }
    }
}
```

---

## Protocol Buffers (Protobuf)

### Описание

Protocol Buffers (Protobuf) — это механизм сериализации структурированных данных, разработанный Google. Protobuf использует IDL (Interface Definition Language) для определения структуры данных и генерирует код для различных языков программирования. Формат известен высокой производительностью и компактностью.

### Структура .proto файла

```protobuf
// order.proto
syntax = "proto3";

package com.example.events;

option java_package = "com.example.events";
option java_outer_classname = "OrderProtos";

import "google/protobuf/timestamp.proto";

// Перечисление статуса заказа
enum OrderStatus {
    ORDER_STATUS_UNSPECIFIED = 0;
    ORDER_STATUS_PENDING = 1;
    ORDER_STATUS_CONFIRMED = 2;
    ORDER_STATUS_SHIPPED = 3;
    ORDER_STATUS_DELIVERED = 4;
    ORDER_STATUS_CANCELLED = 5;
}

// Адрес доставки
message Address {
    string street = 1;
    string city = 2;
    string country = 3;
    string postal_code = 4;
}

// Элемент заказа
message OrderItem {
    string product_id = 1;
    int32 quantity = 2;
    double price = 3;
}

// Основное сообщение заказа
message Order {
    // Уникальный идентификатор заказа
    string order_id = 1;

    // ID клиента
    int64 customer_id = 2;

    // Сумма заказа (в копейках/центах)
    int64 amount_cents = 3;

    // Статус заказа
    OrderStatus status = 4;

    // Список товаров
    repeated OrderItem items = 5;

    // Адрес доставки (опциональный)
    optional Address shipping_address = 6;

    // Время создания
    google.protobuf.Timestamp created_at = 7;

    // Метаданные
    map<string, string> metadata = 8;

    // Зарезервированные поля для будущего использования
    reserved 9, 10;
    reserved "legacy_field";
}

// Событие создания заказа
message OrderCreatedEvent {
    Order order = 1;
    string correlation_id = 2;
}
```

### Типы данных Protobuf

#### Скалярные типы

| Protobuf тип | Описание | Python | Java |
|--------------|----------|--------|------|
| `double` | 64-bit float | `float` | `double` |
| `float` | 32-bit float | `float` | `float` |
| `int32` | 32-bit signed | `int` | `int` |
| `int64` | 64-bit signed | `int` | `long` |
| `uint32` | 32-bit unsigned | `int` | `int` |
| `uint64` | 64-bit unsigned | `int` | `long` |
| `sint32` | ZigZag encoded | `int` | `int` |
| `sint64` | ZigZag encoded | `int` | `long` |
| `fixed32` | 4 bytes | `int` | `int` |
| `fixed64` | 8 bytes | `int` | `long` |
| `sfixed32` | 4 bytes signed | `int` | `int` |
| `sfixed64` | 8 bytes signed | `int` | `long` |
| `bool` | Boolean | `bool` | `boolean` |
| `string` | UTF-8 string | `str` | `String` |
| `bytes` | Byte sequence | `bytes` | `ByteString` |

#### Правила полей

```protobuf
message Example {
    // Singular (по умолчанию в proto3)
    string name = 1;

    // Repeated (массив)
    repeated string tags = 2;

    // Optional (явно опциональное)
    optional string description = 3;

    // Map (словарь)
    map<string, int32> scores = 4;

    // Oneof (одно из)
    oneof contact {
        string email = 5;
        string phone = 6;
    }
}
```

### Примеры кода Protobuf

#### Генерация кода

```bash
# Установка компилятора protobuf
# macOS
brew install protobuf

# Ubuntu
sudo apt-get install protobuf-compiler

# Генерация Python кода
protoc --python_out=./generated --proto_path=./proto ./proto/order.proto

# Генерация Java кода
protoc --java_out=./generated --proto_path=./proto ./proto/order.proto

# Генерация с плагином для gRPC
protoc --python_out=./generated \
       --grpc_python_out=./generated \
       --proto_path=./proto ./proto/order.proto
```

#### Python: Работа с Protobuf

```python
from confluent_kafka import Producer, Consumer
from confluent_kafka.serialization import SerializationContext, MessageField
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.protobuf import ProtobufSerializer, ProtobufDeserializer

# Импорт сгенерированного кода
from generated import order_pb2

# Настройка Schema Registry
schema_registry_conf = {'url': 'http://localhost:8081'}
schema_registry_client = SchemaRegistryClient(schema_registry_conf)

# Создание сериализатора для Protobuf
protobuf_serializer = ProtobufSerializer(
    order_pb2.Order,
    schema_registry_client,
    {'use.deprecated.format': False}
)

# Producer
producer_conf = {'bootstrap.servers': 'localhost:9092'}
producer = Producer(producer_conf)

# Создание сообщения
order = order_pb2.Order()
order.order_id = "ORD-002"
order.customer_id = 12345
order.amount_cents = 9999  # $99.99
order.status = order_pb2.ORDER_STATUS_PENDING

# Добавление элементов заказа
item1 = order.items.add()
item1.product_id = "PROD-1"
item1.quantity = 2
item1.price = 29.99

item2 = order.items.add()
item2.product_id = "PROD-2"
item2.quantity = 1
item2.price = 39.99

# Адрес доставки
order.shipping_address.street = "ул. Пушкина, 10"
order.shipping_address.city = "Санкт-Петербург"
order.shipping_address.country = "Россия"
order.shipping_address.postal_code = "190000"

# Метаданные
order.metadata["source"] = "mobile_app"
order.metadata["version"] = "2.0"

# Отправка
producer.produce(
    topic='orders-proto',
    key=order.order_id,
    value=protobuf_serializer(order, SerializationContext('orders-proto', MessageField.VALUE)),
    on_delivery=lambda err, msg: print(f"Delivered: {msg.topic()}" if not err else f"Error: {err}")
)
producer.flush()

# Consumer
protobuf_deserializer = ProtobufDeserializer(
    order_pb2.Order,
    {'use.deprecated.format': False}
)

consumer_conf = {
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'order-consumer',
    'auto.offset.reset': 'earliest'
}
consumer = Consumer(consumer_conf)
consumer.subscribe(['orders-proto'])

try:
    while True:
        msg = consumer.poll(1.0)
        if msg is None:
            continue
        if msg.error():
            print(f"Error: {msg.error()}")
            continue

        order = protobuf_deserializer(
            msg.value(),
            SerializationContext(msg.topic(), MessageField.VALUE)
        )
        print(f"Получен заказ: {order.order_id}")
        print(f"  Клиент: {order.customer_id}")
        print(f"  Статус: {order_pb2.OrderStatus.Name(order.status)}")
        print(f"  Товаров: {len(order.items)}")
        for item in order.items:
            print(f"    - {item.product_id}: {item.quantity} x ${item.price}")

except KeyboardInterrupt:
    pass
finally:
    consumer.close()
```

#### Java: Работа с Protobuf

```java
import com.example.events.OrderProtos.Order;
import com.example.events.OrderProtos.OrderItem;
import com.example.events.OrderProtos.OrderStatus;
import com.example.events.OrderProtos.Address;
import io.confluent.kafka.serializers.protobuf.KafkaProtobufSerializer;
import io.confluent.kafka.serializers.protobuf.KafkaProtobufDeserializer;
import org.apache.kafka.clients.producer.*;

import java.util.Properties;

public class ProtobufKafkaExample {

    public static void main(String[] args) throws Exception {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", KafkaProtobufSerializer.class.getName());
        props.put("schema.registry.url", "http://localhost:8081");

        // Создание заказа с использованием Builder
        Order order = Order.newBuilder()
            .setOrderId("ORD-003")
            .setCustomerId(12345L)
            .setAmountCents(9999L)
            .setStatus(OrderStatus.ORDER_STATUS_PENDING)
            .addItems(OrderItem.newBuilder()
                .setProductId("PROD-1")
                .setQuantity(2)
                .setPrice(29.99)
                .build())
            .addItems(OrderItem.newBuilder()
                .setProductId("PROD-2")
                .setQuantity(1)
                .setPrice(39.99)
                .build())
            .setShippingAddress(Address.newBuilder()
                .setStreet("ул. Гагарина, 5")
                .setCity("Казань")
                .setCountry("Россия")
                .setPostalCode("420000")
                .build())
            .putMetadata("source", "api")
            .putMetadata("priority", "high")
            .build();

        try (Producer<String, Order> producer = new KafkaProducer<>(props)) {
            ProducerRecord<String, Order> record =
                new ProducerRecord<>("orders-proto", order.getOrderId(), order);

            producer.send(record, (metadata, exception) -> {
                if (exception != null) {
                    System.err.println("Ошибка отправки: " + exception.getMessage());
                } else {
                    System.out.printf("Отправлено в %s partition %d offset %d%n",
                        metadata.topic(), metadata.partition(), metadata.offset());
                }
            }).get();
        }
    }
}
```

---

## JSON Schema

### Описание

JSON Schema — это стандарт для описания структуры JSON документов. В отличие от Avro и Protobuf, JSON Schema использует текстовый формат JSON для хранения данных, что делает сообщения читаемыми человеком. Это удобно для отладки, но приводит к увеличению размера сообщений.

### Структура JSON Schema

```json
{
    "$schema": "http://json-schema.org/draft-07/schema#",
    "$id": "https://example.com/schemas/order.json",
    "title": "Order",
    "description": "Схема события заказа",
    "type": "object",
    "properties": {
        "order_id": {
            "type": "string",
            "description": "Уникальный идентификатор заказа",
            "pattern": "^ORD-[0-9]+$"
        },
        "customer_id": {
            "type": "integer",
            "minimum": 1
        },
        "amount": {
            "type": "number",
            "minimum": 0,
            "description": "Сумма заказа в рублях"
        },
        "status": {
            "type": "string",
            "enum": ["PENDING", "CONFIRMED", "SHIPPED", "DELIVERED", "CANCELLED"]
        },
        "items": {
            "type": "array",
            "minItems": 1,
            "items": {
                "$ref": "#/definitions/OrderItem"
            }
        },
        "shipping_address": {
            "oneOf": [
                {"type": "null"},
                {"$ref": "#/definitions/Address"}
            ]
        },
        "created_at": {
            "type": "string",
            "format": "date-time"
        },
        "metadata": {
            "type": "object",
            "additionalProperties": {
                "type": "string"
            }
        }
    },
    "required": ["order_id", "customer_id", "amount", "status", "items", "created_at"],
    "additionalProperties": false,
    "definitions": {
        "OrderItem": {
            "type": "object",
            "properties": {
                "product_id": {
                    "type": "string"
                },
                "quantity": {
                    "type": "integer",
                    "minimum": 1
                },
                "price": {
                    "type": "number",
                    "minimum": 0
                }
            },
            "required": ["product_id", "quantity", "price"]
        },
        "Address": {
            "type": "object",
            "properties": {
                "street": {"type": "string"},
                "city": {"type": "string"},
                "country": {"type": "string"},
                "postal_code": {"type": "string", "pattern": "^[0-9]{6}$"}
            },
            "required": ["street", "city", "country", "postal_code"]
        }
    }
}
```

### Ключевые элементы JSON Schema

#### Типы данных

| Тип | Описание | Пример |
|-----|----------|--------|
| `string` | Строка | `"hello"` |
| `number` | Число (float) | `3.14` |
| `integer` | Целое число | `42` |
| `boolean` | Булево | `true` |
| `array` | Массив | `[1, 2, 3]` |
| `object` | Объект | `{"key": "value"}` |
| `null` | Null | `null` |

#### Валидация строк

```json
{
    "type": "string",
    "minLength": 1,
    "maxLength": 100,
    "pattern": "^[A-Z]{2}-[0-9]+$",
    "format": "email"  // email, date-time, uri, uuid и др.
}
```

#### Валидация чисел

```json
{
    "type": "number",
    "minimum": 0,
    "maximum": 100,
    "exclusiveMinimum": 0,
    "multipleOf": 0.01
}
```

#### Валидация массивов

```json
{
    "type": "array",
    "items": {"type": "string"},
    "minItems": 1,
    "maxItems": 10,
    "uniqueItems": true
}
```

#### Комбинаторы

```json
// oneOf - ровно одна схема должна подходить
{
    "oneOf": [
        {"type": "string"},
        {"type": "integer"}
    ]
}

// anyOf - хотя бы одна схема должна подходить
{
    "anyOf": [
        {"type": "string", "maxLength": 5},
        {"type": "number", "minimum": 0}
    ]
}

// allOf - все схемы должны подходить
{
    "allOf": [
        {"type": "object", "properties": {"id": {"type": "integer"}}},
        {"required": ["id"]}
    ]
}
```

### Примеры кода JSON Schema

#### Python: Работа с JSON Schema

```python
from confluent_kafka import Producer, Consumer
from confluent_kafka.serialization import SerializationContext, MessageField
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.json_schema import JSONSerializer, JSONDeserializer
from dataclasses import dataclass, asdict
from typing import Optional, List, Dict
from datetime import datetime
import json

# Определение классов
@dataclass
class OrderItem:
    product_id: str
    quantity: int
    price: float

@dataclass
class Address:
    street: str
    city: str
    country: str
    postal_code: str

@dataclass
class Order:
    order_id: str
    customer_id: int
    amount: float
    status: str
    items: List[OrderItem]
    created_at: str
    shipping_address: Optional[Address] = None
    metadata: Optional[Dict[str, str]] = None

# JSON Schema
order_schema_str = """
{
    "$schema": "http://json-schema.org/draft-07/schema#",
    "title": "Order",
    "type": "object",
    "properties": {
        "order_id": {"type": "string"},
        "customer_id": {"type": "integer"},
        "amount": {"type": "number"},
        "status": {"type": "string", "enum": ["PENDING", "CONFIRMED", "SHIPPED", "DELIVERED", "CANCELLED"]},
        "items": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "product_id": {"type": "string"},
                    "quantity": {"type": "integer"},
                    "price": {"type": "number"}
                },
                "required": ["product_id", "quantity", "price"]
            }
        },
        "shipping_address": {
            "oneOf": [
                {"type": "null"},
                {
                    "type": "object",
                    "properties": {
                        "street": {"type": "string"},
                        "city": {"type": "string"},
                        "country": {"type": "string"},
                        "postal_code": {"type": "string"}
                    },
                    "required": ["street", "city", "country", "postal_code"]
                }
            ]
        },
        "created_at": {"type": "string", "format": "date-time"},
        "metadata": {
            "type": "object",
            "additionalProperties": {"type": "string"}
        }
    },
    "required": ["order_id", "customer_id", "amount", "status", "items", "created_at"]
}
"""

def order_to_dict(order: Order, ctx) -> dict:
    """Преобразование Order в словарь"""
    result = {
        "order_id": order.order_id,
        "customer_id": order.customer_id,
        "amount": order.amount,
        "status": order.status,
        "items": [asdict(item) for item in order.items],
        "created_at": order.created_at
    }
    if order.shipping_address:
        result["shipping_address"] = asdict(order.shipping_address)
    if order.metadata:
        result["metadata"] = order.metadata
    return result

def dict_to_order(obj: dict, ctx) -> Order:
    """Преобразование словаря в Order"""
    if obj is None:
        return None

    items = [OrderItem(**item) for item in obj["items"]]

    address = None
    if obj.get("shipping_address"):
        address = Address(**obj["shipping_address"])

    return Order(
        order_id=obj["order_id"],
        customer_id=obj["customer_id"],
        amount=obj["amount"],
        status=obj["status"],
        items=items,
        created_at=obj["created_at"],
        shipping_address=address,
        metadata=obj.get("metadata")
    )

# Настройка
schema_registry_client = SchemaRegistryClient({'url': 'http://localhost:8081'})

json_serializer = JSONSerializer(order_schema_str, schema_registry_client, order_to_dict)
json_deserializer = JSONDeserializer(order_schema_str, dict_to_order)

# Producer
producer = Producer({'bootstrap.servers': 'localhost:9092'})

order = Order(
    order_id="ORD-004",
    customer_id=12345,
    amount=99.99,
    status="PENDING",
    items=[
        OrderItem("PROD-1", 2, 29.99),
        OrderItem("PROD-2", 1, 39.99)
    ],
    created_at=datetime.now().isoformat(),
    shipping_address=Address("ул. Мира, 15", "Новосибирск", "Россия", "630000"),
    metadata={"channel": "web"}
)

producer.produce(
    topic='orders-json',
    key=order.order_id,
    value=json_serializer(order, SerializationContext('orders-json', MessageField.VALUE))
)
producer.flush()
print(f"Заказ {order.order_id} отправлен")

# Consumer
consumer = Consumer({
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'order-json-consumer',
    'auto.offset.reset': 'earliest'
})
consumer.subscribe(['orders-json'])

try:
    while True:
        msg = consumer.poll(1.0)
        if msg is None:
            continue
        if msg.error():
            print(f"Error: {msg.error()}")
            continue

        order = json_deserializer(
            msg.value(),
            SerializationContext(msg.topic(), MessageField.VALUE)
        )
        print(f"Получен заказ: {order.order_id}")
        print(f"  Сумма: {order.amount} руб.")
        print(f"  Статус: {order.status}")

except KeyboardInterrupt:
    pass
finally:
    consumer.close()
```

---

## Best Practices

### Выбор формата сериализации

```
┌────────────────────────────────────────────────────────────────────┐
│                    Дерево принятия решений                         │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Нужна ли максимальная производительность?                        │
│  ├── Да → Есть ли существующие .proto файлы?                      │
│  │        ├── Да → Protobuf                                       │
│  │        └── Нет → Работа с Big Data/Hadoop?                     │
│  │                  ├── Да → Avro                                  │
│  │                  └── Нет → Protobuf                             │
│  │                                                                │
│  └── Нет → Важна ли читаемость сообщений?                         │
│            ├── Да → JSON Schema                                    │
│            └── Нет → Avro                                          │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Рекомендации по форматам

#### Avro

1. **Используйте логические типы** для дат и времени
2. **Добавляйте namespace** для избежания конфликтов имён
3. **Документируйте поля** с помощью `doc`
4. **Используйте union с null** для опциональных полей: `["null", "string"]`

#### Protobuf

1. **Никогда не переиспользуйте номера полей** — используйте `reserved`
2. **Начинайте enum с UNSPECIFIED = 0**
3. **Используйте optional явно** для опциональных полей в proto3
4. **Группируйте связанные поля** в message

#### JSON Schema

1. **Указывайте `$schema`** для версии спецификации
2. **Используйте `$ref`** для переиспользования определений
3. **Устанавливайте `additionalProperties: false`** для строгой валидации
4. **Добавляйте `format`** для строк: email, date-time, uri

### Миграция между форматами

| Переход | Сложность | Рекомендация |
|---------|-----------|--------------|
| JSON → Avro | Средняя | Рекомендуется для оптимизации |
| JSON → Protobuf | Высокая | Требует кодогенерации |
| Avro → Protobuf | Высокая | Редко необходим |
| Protobuf → Avro | Средняя | При интеграции с Hadoop |

### Типичные ошибки

1. **Смешивание форматов в одном топике** — используйте один формат
2. **Отсутствие значений по умолчанию** — ломает обратную совместимость
3. **Слишком глубокая вложенность** — усложняет эволюцию схем
4. **Игнорирование логических типов** — теряется семантика данных
