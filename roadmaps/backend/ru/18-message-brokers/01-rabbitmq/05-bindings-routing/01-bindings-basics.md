# Основы привязок (Bindings)

## Что такое Binding?

**Binding (привязка)** — это связь между exchange и queue в RabbitMQ. Привязка определяет правила, по которым сообщения из exchange попадают в определённую очередь.

```
┌─────────────────┐     Binding      ┌─────────────────┐
│                 │ ═══════════════> │                 │
│    Exchange     │   routing_key    │     Queue       │
│                 │ <═══════════════ │                 │
└─────────────────┘                  └─────────────────┘
```

Без привязок сообщения, отправленные в exchange, никогда не достигнут очередей — они просто будут отброшены (если не настроен alternate exchange).

## Структура привязки

Каждая привязка включает:

| Компонент | Описание |
|-----------|----------|
| **Exchange** | Источник сообщений |
| **Queue** | Получатель сообщений |
| **Routing Key** | Ключ маршрутизации (для direct и topic exchanges) |
| **Arguments** | Дополнительные аргументы (для headers exchange) |

## Создание привязок

### Базовое создание привязки

```python
import pika

# Подключение к RabbitMQ
connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Объявляем exchange
channel.exchange_declare(
    exchange='orders_exchange',
    exchange_type='direct',
    durable=True
)

# Объявляем очередь
channel.queue_declare(
    queue='new_orders',
    durable=True
)

# Создаём привязку
channel.queue_bind(
    queue='new_orders',
    exchange='orders_exchange',
    routing_key='order.new'
)

print("Привязка создана успешно!")
connection.close()
```

### Привязка с аргументами (для headers exchange)

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Headers exchange
channel.exchange_declare(
    exchange='notifications_headers',
    exchange_type='headers',
    durable=True
)

channel.queue_declare(queue='urgent_emails', durable=True)

# Привязка с аргументами
channel.queue_bind(
    queue='urgent_emails',
    exchange='notifications_headers',
    arguments={
        'x-match': 'all',        # все заголовки должны совпадать
        'type': 'email',
        'priority': 'urgent'
    }
)
```

## Множественные привязки

### Одна очередь — несколько привязок

Очередь может быть привязана к нескольким exchange или к одному exchange с разными routing keys:

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Создаём exchange и очередь
channel.exchange_declare(exchange='logs', exchange_type='direct')
channel.queue_declare(queue='all_errors')

# Множественные привязки к одной очереди
routing_keys = ['error', 'critical', 'fatal']

for key in routing_keys:
    channel.queue_bind(
        queue='all_errors',
        exchange='logs',
        routing_key=key
    )
    print(f"Привязка с ключом '{key}' создана")

connection.close()
```

Визуализация:

```
                        ┌─── routing_key: "error" ────┐
                        │                             │
┌─────────────┐         ├─── routing_key: "critical" ─┼───> ┌─────────────┐
│    logs     │ ────────┤                             │     │ all_errors  │
│  (direct)   │         └─── routing_key: "fatal" ────┘     └─────────────┘
└─────────────┘
```

### Один exchange — несколько очередей

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

channel.exchange_declare(exchange='events', exchange_type='topic')

# Разные очереди для разных типов событий
queues_config = [
    ('user_events', 'user.*'),
    ('order_events', 'order.*'),
    ('all_events', '#')
]

for queue_name, routing_key in queues_config:
    channel.queue_declare(queue=queue_name, durable=True)
    channel.queue_bind(
        queue=queue_name,
        exchange='events',
        routing_key=routing_key
    )
    print(f"Очередь '{queue_name}' привязана с ключом '{routing_key}'")

connection.close()
```

## Удаление привязок

### Метод queue_unbind

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Удаляем конкретную привязку
channel.queue_unbind(
    queue='new_orders',
    exchange='orders_exchange',
    routing_key='order.new'
)

print("Привязка удалена")
connection.close()
```

### Удаление с аргументами

```python
# Для headers exchange нужно указать те же аргументы
channel.queue_unbind(
    queue='urgent_emails',
    exchange='notifications_headers',
    arguments={
        'x-match': 'all',
        'type': 'email',
        'priority': 'urgent'
    }
)
```

## Просмотр привязок

### Через Management API

```python
import requests

# Получаем список привязок
response = requests.get(
    'http://localhost:15672/api/bindings',
    auth=('guest', 'guest')
)

bindings = response.json()
for binding in bindings:
    print(f"Exchange: {binding['source']} -> Queue: {binding['destination']}")
    print(f"  Routing Key: {binding['routing_key']}")
    print(f"  Arguments: {binding['arguments']}")
    print()
```

### Через CLI

```bash
# Список всех привязок
rabbitmqctl list_bindings

# С подробной информацией
rabbitmqctl list_bindings source_name destination_name routing_key
```

## Привязка Exchange к Exchange

RabbitMQ позволяет привязывать exchange к другим exchange:

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Создаём два exchange
channel.exchange_declare(exchange='source_exchange', exchange_type='topic')
channel.exchange_declare(exchange='destination_exchange', exchange_type='fanout')

# Привязываем exchange к exchange
channel.exchange_bind(
    destination='destination_exchange',
    source='source_exchange',
    routing_key='important.*'
)

print("Exchange привязан к exchange")
connection.close()
```

Схема:

```
┌─────────────────────┐     Binding      ┌─────────────────────┐
│   source_exchange   │ ═══════════════> │ destination_exchange│
│      (topic)        │  important.*     │      (fanout)       │
└─────────────────────┘                  └─────────────────────┘
                                                   │
                                    ┌──────────────┼──────────────┐
                                    ▼              ▼              ▼
                              ┌─────────┐    ┌─────────┐    ┌─────────┐
                              │ Queue 1 │    │ Queue 2 │    │ Queue 3 │
                              └─────────┘    └─────────┘    └─────────┘
```

## Best Practices

### 1. Именование привязок

Используйте понятную структуру routing keys:

```python
# Хорошо: иерархическая структура
routing_keys = [
    'order.created',
    'order.updated',
    'order.cancelled',
    'user.registered',
    'user.deleted'
]

# Плохо: неструктурированные ключи
routing_keys = [
    'neworder',
    'orderupd',
    'cancel',
    'reg',
    'del'
]
```

### 2. Документирование привязок

```python
BINDINGS_CONFIG = {
    'orders_exchange': {
        'type': 'topic',
        'bindings': [
            {
                'queue': 'order_processing',
                'routing_key': 'order.*',
                'description': 'Все события заказов'
            },
            {
                'queue': 'notifications',
                'routing_key': '*.created',
                'description': 'Уведомления о создании'
            }
        ]
    }
}
```

### 3. Идемпотентность

Привязки в RabbitMQ идемпотентны — повторный вызов `queue_bind` с теми же параметрами не создаст дубликат:

```python
# Безопасно вызывать несколько раз
for _ in range(3):
    channel.queue_bind(
        queue='my_queue',
        exchange='my_exchange',
        routing_key='my.key'
    )
# Будет создана только одна привязка
```

### 4. Обработка ошибок

```python
import pika
from pika.exceptions import ChannelClosedByBroker

try:
    channel.queue_bind(
        queue='nonexistent_queue',
        exchange='my_exchange',
        routing_key='test'
    )
except ChannelClosedByBroker as e:
    print(f"Ошибка привязки: {e}")
    # Очередь или exchange не существует
```

## Типичные ошибки

| Ошибка | Причина | Решение |
|--------|---------|---------|
| NOT_FOUND | Очередь или exchange не существует | Создайте их перед привязкой |
| PRECONDITION_FAILED | Неверные аргументы для headers binding | Проверьте x-match и ключи |
| ACCESS_REFUSED | Нет прав на привязку | Проверьте permissions пользователя |

## Резюме

- **Binding** связывает exchange с queue и определяет правила доставки сообщений
- Одна очередь может иметь множество привязок
- Привязки создаются через `queue_bind()` и удаляются через `queue_unbind()`
- Exchange можно привязывать к другим exchange через `exchange_bind()`
- Привязки идемпотентны — повторное создание безопасно
