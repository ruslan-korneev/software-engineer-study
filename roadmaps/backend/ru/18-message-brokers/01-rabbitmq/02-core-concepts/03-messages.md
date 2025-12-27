# Сообщения (Messages)

## Что такое Message в RabbitMQ?

**Message (Сообщение)** — это единица данных, передаваемая между producer и consumer через RabbitMQ. Сообщение состоит из двух основных частей: **свойств (properties)** и **тела (body)**.

## Структура сообщения

```
┌─────────────────────────────────────────────────────────────┐
│                        MESSAGE                               │
├─────────────────────────────────────────────────────────────┤
│  PROPERTIES (Metadata)                                       │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ delivery_mode, content_type, content_encoding,         ││
│  │ priority, correlation_id, reply_to, expiration,        ││
│  │ message_id, timestamp, type, user_id, app_id, headers  ││
│  └─────────────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────────────┤
│  BODY (Payload)                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ Binary data (bytes): JSON, XML, Protobuf, plain text   ││
│  └─────────────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────────────┤
│  ROUTING KEY (для маршрутизации)                             │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ String: "user.created", "order.payment.completed"      ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

## Свойства сообщения (Properties)

### Основные свойства AMQP

```python
import pika
from datetime import datetime
import uuid

properties = pika.BasicProperties(
    # Режим доставки: 1 = transient, 2 = persistent
    delivery_mode=2,

    # MIME-тип содержимого
    content_type='application/json',

    # Кодировка содержимого
    content_encoding='utf-8',

    # Приоритет сообщения (0-9, где 9 - высший)
    priority=5,

    # ID для корреляции запрос-ответ
    correlation_id=str(uuid.uuid4()),

    # Очередь для ответа (RPC паттерн)
    reply_to='response_queue',

    # Время жизни сообщения в миллисекундах
    expiration='60000',

    # Уникальный идентификатор сообщения
    message_id=str(uuid.uuid4()),

    # Временная метка создания
    timestamp=int(datetime.now().timestamp()),

    # Тип сообщения (для десериализации)
    type='order.created',

    # ID пользователя (проверяется RabbitMQ)
    user_id='guest',

    # ID приложения-отправителя
    app_id='order_service',

    # Кластерный ID (устарел)
    cluster_id=None
)
```

### Подробное описание свойств

| Свойство | Тип | Описание |
|----------|-----|----------|
| `delivery_mode` | int | 1 = не сохранять на диск, 2 = persistent |
| `content_type` | str | MIME-тип: application/json, text/plain и т.д. |
| `content_encoding` | str | Кодировка: utf-8, gzip и т.д. |
| `priority` | int | Приоритет 0-9 (требует priority queue) |
| `correlation_id` | str | ID для связи запроса и ответа |
| `reply_to` | str | Имя очереди для ответа |
| `expiration` | str | TTL в миллисекундах (строка!) |
| `message_id` | str | Уникальный ID сообщения |
| `timestamp` | int | Unix timestamp создания |
| `type` | str | Тип сообщения для роутинга/десериализации |
| `user_id` | str | ID пользователя (валидируется брокером) |
| `app_id` | str | ID приложения-отправителя |

## Headers (Заголовки)

Headers позволяют добавлять произвольные метаданные.

```python
import pika

properties = pika.BasicProperties(
    headers={
        # Строки
        'x-trace-id': 'abc123',
        'x-request-id': 'req-456',

        # Числа
        'x-retry-count': 0,
        'x-priority-level': 5,

        # Булевы значения
        'x-is-test': False,

        # Списки
        'x-tags': ['important', 'urgent'],

        # Вложенные структуры (таблицы)
        'x-metadata': {
            'source': 'api',
            'version': '1.0'
        }
    }
)
```

### Использование Headers для маршрутизации

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Объявление headers exchange
channel.exchange_declare(
    exchange='headers_exchange',
    exchange_type='headers'
)

# Сообщение с headers для маршрутизации
channel.basic_publish(
    exchange='headers_exchange',
    routing_key='',  # Игнорируется
    body='Message for VIP users',
    properties=pika.BasicProperties(
        headers={
            'x-match': 'all',  # all = AND, any = OR
            'user-type': 'vip',
            'region': 'eu'
        }
    )
)

connection.close()
```

## Body (Тело сообщения)

Body — это бинарные данные (bytes). Формат определяется приложением.

### JSON (самый распространённый)

```python
import pika
import json

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()
channel.queue_declare(queue='orders')

# Формирование JSON сообщения
order = {
    'order_id': 12345,
    'user_id': 100,
    'items': [
        {'product_id': 1, 'quantity': 2, 'price': 29.99},
        {'product_id': 5, 'quantity': 1, 'price': 49.99}
    ],
    'total': 109.97,
    'created_at': '2024-01-15T10:30:00Z'
}

channel.basic_publish(
    exchange='',
    routing_key='orders',
    body=json.dumps(order),
    properties=pika.BasicProperties(
        content_type='application/json',
        content_encoding='utf-8'
    )
)

connection.close()
```

### Чтение JSON сообщения

```python
import pika
import json

def callback(ch, method, properties, body):
    # Проверка content_type
    if properties.content_type == 'application/json':
        message = json.loads(body.decode('utf-8'))
        print(f"Order ID: {message['order_id']}")
        print(f"Total: ${message['total']}")
    else:
        print(f"Unknown format: {properties.content_type}")

    ch.basic_ack(delivery_tag=method.delivery_tag)

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()
channel.basic_consume(queue='orders', on_message_callback=callback)
channel.start_consuming()
```

### Protocol Buffers (для высокой производительности)

```python
# Определение proto файла: order.proto
# syntax = "proto3";
# message Order {
#     int64 order_id = 1;
#     int64 user_id = 2;
#     repeated OrderItem items = 3;
#     double total = 4;
# }

import pika
# from order_pb2 import Order  # Сгенерированный класс

# order = Order()
# order.order_id = 12345
# order.user_id = 100
# order.total = 109.97

# channel.basic_publish(
#     exchange='',
#     routing_key='orders',
#     body=order.SerializeToString(),
#     properties=pika.BasicProperties(
#         content_type='application/x-protobuf'
#     )
# )
```

### Сжатие сообщений

```python
import pika
import json
import gzip

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()
channel.queue_declare(queue='compressed')

# Большие данные
large_data = {'items': [{'id': i, 'data': 'x' * 1000} for i in range(100)]}

# Сжатие
json_bytes = json.dumps(large_data).encode('utf-8')
compressed = gzip.compress(json_bytes)

print(f"Original: {len(json_bytes)} bytes")
print(f"Compressed: {len(compressed)} bytes")

channel.basic_publish(
    exchange='',
    routing_key='compressed',
    body=compressed,
    properties=pika.BasicProperties(
        content_type='application/json',
        content_encoding='gzip'
    )
)

connection.close()
```

### Чтение сжатых сообщений

```python
import pika
import json
import gzip

def callback(ch, method, properties, body):
    # Проверка кодировки
    if properties.content_encoding == 'gzip':
        body = gzip.decompress(body)

    if properties.content_type == 'application/json':
        data = json.loads(body.decode('utf-8'))
        print(f"Received {len(data['items'])} items")

    ch.basic_ack(delivery_tag=method.delivery_tag)

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()
channel.basic_consume(queue='compressed', on_message_callback=callback)
channel.start_consuming()
```

## Routing Key (Ключ маршрутизации)

Routing Key определяет, в какую очередь попадёт сообщение.

### Для Direct Exchange

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

channel.exchange_declare(exchange='logs', exchange_type='direct')

# Разные routing keys для разных уровней логов
for level, message in [
    ('info', 'User logged in'),
    ('warning', 'High memory usage'),
    ('error', 'Database connection failed'),
    ('critical', 'System crash')
]:
    channel.basic_publish(
        exchange='logs',
        routing_key=level,  # Точное совпадение
        body=message
    )
    print(f"Sent [{level}]: {message}")

connection.close()
```

### Для Topic Exchange

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

channel.exchange_declare(exchange='events', exchange_type='topic')

# Иерархические routing keys
events = [
    ('user.created', {'user_id': 1}),
    ('user.updated', {'user_id': 1, 'field': 'email'}),
    ('user.deleted', {'user_id': 1}),
    ('order.created', {'order_id': 100}),
    ('order.payment.completed', {'order_id': 100}),
    ('order.payment.failed', {'order_id': 101}),
    ('order.shipped', {'order_id': 100}),
]

for routing_key, payload in events:
    channel.basic_publish(
        exchange='events',
        routing_key=routing_key,
        body=json.dumps(payload),
        properties=pika.BasicProperties(
            content_type='application/json'
        )
    )
    print(f"Published: {routing_key}")

# Consumers могут подписаться на:
# - "user.*" - все события пользователей
# - "order.payment.*" - все события оплаты
# - "#" - все события
# - "*.created" - все события создания

connection.close()
```

## Message TTL (Time-To-Live)

### TTL на уровне сообщения

```python
import pika

channel.basic_publish(
    exchange='',
    routing_key='tasks',
    body='Time-sensitive task',
    properties=pika.BasicProperties(
        expiration='30000'  # 30 секунд в миллисекундах
    )
)
```

### TTL на уровне очереди

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Все сообщения в очереди живут максимум 60 секунд
channel.queue_declare(
    queue='short_lived',
    arguments={
        'x-message-ttl': 60000  # 60 секунд
    }
)

connection.close()
```

## Priority Queue (Приоритетные очереди)

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Объявление очереди с поддержкой приоритетов
channel.queue_declare(
    queue='priority_tasks',
    arguments={
        'x-max-priority': 10  # Максимальный приоритет
    }
)

# Публикация сообщений с разными приоритетами
tasks = [
    ('Low priority task', 1),
    ('Normal task', 5),
    ('High priority task', 9),
    ('Critical task', 10),
]

for task, priority in tasks:
    channel.basic_publish(
        exchange='',
        routing_key='priority_tasks',
        body=task,
        properties=pika.BasicProperties(
            priority=priority
        )
    )
    print(f"Sent (priority={priority}): {task}")

connection.close()
```

## Создание типизированных сообщений

```python
import pika
import json
from dataclasses import dataclass, asdict
from datetime import datetime
from typing import Optional, List
import uuid


@dataclass
class OrderItem:
    product_id: int
    quantity: int
    price: float


@dataclass
class OrderMessage:
    order_id: int
    user_id: int
    items: List[OrderItem]
    total: float
    status: str = 'pending'
    created_at: str = None

    def __post_init__(self):
        if self.created_at is None:
            self.created_at = datetime.utcnow().isoformat()


class MessagePublisher:
    def __init__(self, connection):
        self.channel = connection.channel()

    def publish_order(
        self,
        order: OrderMessage,
        routing_key: str = 'orders'
    ) -> str:
        """Публикация сообщения заказа"""
        message_id = str(uuid.uuid4())

        # Преобразование dataclass в dict
        body = json.dumps(asdict(order))

        properties = pika.BasicProperties(
            delivery_mode=2,
            content_type='application/json',
            message_id=message_id,
            type='order.created',
            timestamp=int(datetime.now().timestamp()),
            app_id='order_service'
        )

        self.channel.basic_publish(
            exchange='',
            routing_key=routing_key,
            body=body,
            properties=properties
        )

        return message_id


# Использование
connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
connection.channel().queue_declare(queue='orders')

publisher = MessagePublisher(connection)

order = OrderMessage(
    order_id=12345,
    user_id=100,
    items=[
        OrderItem(product_id=1, quantity=2, price=29.99),
        OrderItem(product_id=5, quantity=1, price=49.99)
    ],
    total=109.97
)

message_id = publisher.publish_order(order)
print(f"Published order with message_id: {message_id}")

connection.close()
```

## Best Practices для сообщений

### 1. Всегда указывайте content_type

```python
properties = pika.BasicProperties(
    content_type='application/json'  # Явно указываем формат
)
```

### 2. Используйте message_id для трассировки

```python
properties = pika.BasicProperties(
    message_id=str(uuid.uuid4())
)
```

### 3. Устанавливайте delivery_mode=2 для важных данных

```python
properties = pika.BasicProperties(
    delivery_mode=2  # Persistent
)
```

### 4. Добавляйте timestamp

```python
properties = pika.BasicProperties(
    timestamp=int(datetime.now().timestamp())
)
```

### 5. Используйте headers для метаданных

```python
properties = pika.BasicProperties(
    headers={
        'x-trace-id': trace_id,
        'x-source': 'api'
    }
)
```

### 6. Ограничивайте размер сообщений

```python
MAX_MESSAGE_SIZE = 1024 * 1024  # 1 MB

if len(body) > MAX_MESSAGE_SIZE:
    # Сохранить в S3/Redis и передать ссылку
    pass
```

### 7. Версионируйте формат сообщений

```python
message = {
    'version': '1.0',
    'type': 'order.created',
    'data': {...}
}
```

## Типичные ошибки

1. **Не указывать content_type** — consumer не знает как парсить
2. **Передавать слишком большие сообщения** — снижение производительности
3. **Не использовать persistent для важных данных** — потеря при перезапуске
4. **Хранить чувствительные данные без шифрования** — проблемы безопасности
5. **Не добавлять идентификаторы** — невозможность отладки

## Заключение

Сообщение в RabbitMQ — это больше, чем просто данные. Правильное использование свойств, заголовков и routing key позволяет:
- Маршрутизировать сообщения по нужным очередям
- Обеспечивать надёжную доставку
- Реализовывать паттерны RPC и корреляции
- Управлять приоритетами и временем жизни
- Отслеживать и отлаживать обработку

Понимание структуры сообщений — ключ к построению эффективных систем на базе RabbitMQ.
