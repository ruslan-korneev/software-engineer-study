# Routing Keys

[prev: 01-bindings-basics](./01-bindings-basics.md) | [next: 03-routing-patterns](./03-routing-patterns.md)

---

## Что такое Routing Key?

**Routing Key** — это строковый идентификатор, который используется для определения маршрута сообщения от exchange к очередям. Это ключевой механизм маршрутизации в RabbitMQ.

```
┌──────────────┐                                    ┌──────────────┐
│   Producer   │                                    │    Queue 1   │
│              │                                    │ routing_key: │
│ routing_key: │     ┌──────────────┐               │  "order.new" │
│ "order.new"  │────>│   Exchange   │──────────────>└──────────────┘
│              │     │   (direct)   │
└──────────────┘     └──────────────┘               ┌──────────────┐
                              │                     │    Queue 2   │
                              │                     │ routing_key: │
                              └────────────────────>│"order.cancel"│
                                                    └──────────────┘
```

## Routing Key в разных типах Exchange

| Тип Exchange | Использование Routing Key |
|--------------|--------------------------|
| **Direct** | Точное совпадение ключа |
| **Topic** | Pattern matching с wildcards |
| **Fanout** | Игнорируется полностью |
| **Headers** | Игнорируется (используются headers) |

## Структура Routing Key

### Простая структура

```
routing_key = "orders"
```

### Иерархическая структура

Точка (`.`) используется как разделитель уровней:

```
routing_key = "domain.action.entity"

# Примеры:
"order.created"
"order.updated.status"
"user.profile.updated"
"payment.completed.success"
"notification.email.sent"
```

### Схема формирования

```
┌─────────────────────────────────────────────────────────┐
│                    Routing Key                          │
├─────────────┬─────────────┬─────────────┬──────────────┤
│   Domain    │   Entity    │   Action    │   Details    │
│  (домен)    │  (сущность) │  (действие) │   (детали)   │
├─────────────┼─────────────┼─────────────┼──────────────┤
│   order     │   item      │   added     │   premium    │
│   user      │   profile   │   updated   │   verified   │
│   payment   │   invoice   │   created   │   monthly    │
└─────────────┴─────────────┴─────────────┴──────────────┘

Результат: "order.item.added.premium"
```

## Конвенции именования

### Рекомендуемые практики

```python
# 1. Используйте lowercase
GOOD = "order.created"
BAD  = "Order.Created"

# 2. Используйте точки как разделители
GOOD = "user.profile.updated"
BAD  = "user_profile_updated"
BAD  = "user-profile-updated"

# 3. Порядок: общее -> частное
GOOD = "europe.germany.berlin"
BAD  = "berlin.germany.europe"

# 4. Глаголы в прошедшем времени для событий
GOOD = "order.created"
GOOD = "payment.processed"
BAD  = "order.create"
BAD  = "create.order"

# 5. Краткие, но понятные имена
GOOD = "user.auth.failed"
BAD  = "user.authentication.failure.occurred"
```

### Примеры для разных доменов

```python
# E-commerce
ECOMMERCE_KEYS = [
    "order.created",
    "order.paid",
    "order.shipped",
    "order.delivered",
    "order.cancelled",
    "order.refunded",
    "product.created",
    "product.updated",
    "product.deleted",
    "inventory.updated",
    "inventory.low",
]

# Пользователи
USER_KEYS = [
    "user.registered",
    "user.verified",
    "user.login.success",
    "user.login.failed",
    "user.password.changed",
    "user.profile.updated",
    "user.deleted",
]

# Уведомления
NOTIFICATION_KEYS = [
    "notification.email.sent",
    "notification.sms.sent",
    "notification.push.sent",
    "notification.email.failed",
]
```

## Практические примеры

### Direct Exchange с Routing Keys

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Объявляем direct exchange
channel.exchange_declare(
    exchange='orders',
    exchange_type='direct',
    durable=True
)

# Создаём очереди для разных статусов заказов
order_statuses = ['new', 'processing', 'shipped', 'delivered']

for status in order_statuses:
    queue_name = f'orders_{status}'
    routing_key = f'order.{status}'

    # Создаём очередь
    channel.queue_declare(queue=queue_name, durable=True)

    # Привязываем с routing key
    channel.queue_bind(
        queue=queue_name,
        exchange='orders',
        routing_key=routing_key
    )

    print(f"Queue '{queue_name}' bound with key '{routing_key}'")

connection.close()
```

### Публикация с Routing Key

```python
import pika
import json

def publish_order_event(order_id: int, status: str, data: dict):
    """Публикация события заказа с соответствующим routing key."""

    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Формируем routing key
    routing_key = f'order.{status}'

    # Подготавливаем сообщение
    message = {
        'order_id': order_id,
        'status': status,
        'data': data
    }

    # Публикуем
    channel.basic_publish(
        exchange='orders',
        routing_key=routing_key,
        body=json.dumps(message),
        properties=pika.BasicProperties(
            delivery_mode=2,  # persistent
            content_type='application/json'
        )
    )

    print(f"Published to '{routing_key}': order {order_id}")
    connection.close()

# Использование
publish_order_event(12345, 'new', {'items': ['item1', 'item2']})
publish_order_event(12345, 'processing', {'warehouse': 'WH-01'})
publish_order_event(12345, 'shipped', {'tracking': 'TRK123456'})
```

### Consumer с фильтрацией по Routing Key

```python
import pika
import json

def create_order_consumer(status: str):
    """Создание consumer для определённого статуса заказа."""

    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    queue_name = f'orders_{status}'

    def callback(ch, method, properties, body):
        data = json.loads(body)
        print(f"[{status.upper()}] Order {data['order_id']}: {data['data']}")
        ch.basic_ack(delivery_tag=method.delivery_tag)

    channel.basic_consume(
        queue=queue_name,
        on_message_callback=callback
    )

    print(f"Waiting for '{status}' orders...")
    channel.start_consuming()

# Запускаем consumer для новых заказов
create_order_consumer('new')
```

## Routing Key в Topic Exchange

Topic exchange поддерживает wildcards в routing keys:

| Wildcard | Описание | Пример |
|----------|----------|--------|
| `*` | Заменяет ровно одно слово | `order.*` совпадает с `order.created` |
| `#` | Заменяет ноль или более слов | `order.#` совпадает с `order.created.success` |

### Пример Topic Exchange

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

channel.exchange_declare(exchange='events', exchange_type='topic')

# Разные привязки с разными паттернами
bindings = [
    ('all_orders', 'order.#'),           # Все события заказов
    ('new_only', 'order.created'),        # Только новые заказы
    ('all_created', '*.created'),         # Все события создания
    ('everything', '#'),                  # Абсолютно всё
]

for queue_name, pattern in bindings:
    channel.queue_declare(queue=queue_name)
    channel.queue_bind(
        queue=queue_name,
        exchange='events',
        routing_key=pattern
    )
    print(f"'{queue_name}' <- '{pattern}'")

connection.close()
```

Таблица совпадений:

```
Routing Key           │ order.# │ order.created │ *.created │   #   │
──────────────────────┼─────────┼───────────────┼───────────┼───────┤
order.created         │    ✓    │       ✓       │     ✓     │   ✓   │
order.updated         │    ✓    │       ✗       │     ✗     │   ✓   │
order.item.added      │    ✓    │       ✗       │     ✗     │   ✓   │
user.created          │    ✗    │       ✗       │     ✓     │   ✓   │
payment.processed     │    ✗    │       ✗       │     ✗     │   ✓   │
```

## Routing Key Generators

### Класс для генерации ключей

```python
from enum import Enum
from typing import Optional

class Domain(Enum):
    ORDER = "order"
    USER = "user"
    PAYMENT = "payment"
    NOTIFICATION = "notification"

class Action(Enum):
    CREATED = "created"
    UPDATED = "updated"
    DELETED = "deleted"
    PROCESSED = "processed"
    FAILED = "failed"

class RoutingKeyBuilder:
    """Builder для создания routing keys."""

    def __init__(self):
        self._parts = []

    def domain(self, domain: Domain) -> 'RoutingKeyBuilder':
        self._parts.append(domain.value)
        return self

    def entity(self, entity: str) -> 'RoutingKeyBuilder':
        self._parts.append(entity.lower())
        return self

    def action(self, action: Action) -> 'RoutingKeyBuilder':
        self._parts.append(action.value)
        return self

    def detail(self, detail: str) -> 'RoutingKeyBuilder':
        self._parts.append(detail.lower())
        return self

    def build(self) -> str:
        return '.'.join(self._parts)

# Использование
key = (RoutingKeyBuilder()
    .domain(Domain.ORDER)
    .entity('item')
    .action(Action.CREATED)
    .detail('premium')
    .build())

print(key)  # "order.item.created.premium"
```

## Валидация Routing Keys

```python
import re
from typing import Tuple

def validate_routing_key(key: str) -> Tuple[bool, str]:
    """
    Валидация routing key.

    Returns:
        Tuple[bool, str]: (is_valid, error_message)
    """

    # Максимальная длина
    if len(key) > 255:
        return False, "Routing key слишком длинный (max 255)"

    # Пустой ключ
    if not key:
        return False, "Routing key не может быть пустым"

    # Только допустимые символы
    pattern = r'^[a-zA-Z0-9._#*-]+$'
    if not re.match(pattern, key):
        return False, "Недопустимые символы в routing key"

    # Проверка wildcard использования
    if '*' in key or '#' in key:
        # Wildcards только в binding, не в публикации
        return False, "Wildcards можно использовать только в bindings"

    return True, ""

# Тесты
test_keys = [
    "order.created",
    "order.item.updated.premium",
    "Order.Created",  # Не ошибка, но не рекомендуется
    "a" * 300,        # Слишком длинный
    "",               # Пустой
    "order created",  # Пробел
]

for key in test_keys:
    is_valid, error = validate_routing_key(key)
    status = "✓" if is_valid else "✗"
    print(f"{status} '{key[:30]}...' - {error if error else 'OK'}")
```

## Best Practices

### 1. Используйте константы

```python
class RoutingKeys:
    """Централизованное хранение routing keys."""

    # Orders
    ORDER_CREATED = "order.created"
    ORDER_UPDATED = "order.updated"
    ORDER_CANCELLED = "order.cancelled"

    # Users
    USER_REGISTERED = "user.registered"
    USER_DELETED = "user.deleted"

    # Patterns для bindings
    ALL_ORDERS = "order.#"
    ALL_USER_EVENTS = "user.*"
```

### 2. Документируйте структуру

```python
"""
Routing Key Structure:
    {domain}.{entity}.{action}[.{detail}]

Domains:
    - order: заказы и связанные операции
    - user: пользователи и аутентификация
    - payment: платежи
    - notification: уведомления

Actions:
    - created: создание
    - updated: обновление
    - deleted: удаление
    - processed: обработка
    - failed: ошибка
"""
```

### 3. Версионирование (при необходимости)

```python
# Если нужна обратная совместимость
routing_key = "v1.order.created"
routing_key = "v2.order.created"
```

## Типичные ошибки

| Ошибка | Проблема | Решение |
|--------|----------|---------|
| Wildcards в publish | Сообщение не доставится | Используйте конкретные ключи |
| Слишком общие ключи | Сложно фильтровать | Добавьте иерархию |
| Слишком длинные ключи | Снижение производительности | Сократите до сути |
| Несогласованные имена | Путаница в routing | Используйте константы |

## Резюме

- **Routing Key** — строка для маршрутизации сообщений
- Используйте точечную нотацию для иерархии: `domain.entity.action`
- В **Direct** exchange требуется точное совпадение
- В **Topic** exchange можно использовать wildcards (`*`, `#`)
- Используйте константы и builders для консистентности
- Валидируйте ключи перед использованием

---

[prev: 01-bindings-basics](./01-bindings-basics.md) | [next: 03-routing-patterns](./03-routing-patterns.md)
