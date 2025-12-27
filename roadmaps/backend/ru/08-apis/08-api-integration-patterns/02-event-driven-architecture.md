# Event-Driven Architecture (EDA)

## Введение

Event-Driven Architecture (Событийно-ориентированная архитектура) — это архитектурный паттерн, в котором компоненты системы взаимодействуют через **события**. Вместо прямых вызовов между сервисами, компоненты публикуют и подписываются на события.

## Ключевые концепции

### Что такое событие (Event)?

**Событие** — это неизменяемая (immutable) запись о том, что что-то произошло в системе.

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Any
import uuid

@dataclass(frozen=True)  # immutable
class Event:
    """Базовый класс события."""
    event_id: str
    event_type: str
    timestamp: datetime
    source: str
    data: dict

    @classmethod
    def create(cls, event_type: str, source: str, data: dict):
        return cls(
            event_id=str(uuid.uuid4()),
            event_type=event_type,
            timestamp=datetime.utcnow(),
            source=source,
            data=data
        )

# Примеры событий
@dataclass(frozen=True)
class OrderCreatedEvent(Event):
    """Событие создания заказа."""
    pass

@dataclass(frozen=True)
class PaymentProcessedEvent(Event):
    """Событие обработки платежа."""
    pass

@dataclass(frozen=True)
class InventoryReservedEvent(Event):
    """Событие резервирования товара."""
    pass
```

### Компоненты EDA

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Event-Driven Architecture                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌──────────┐      ┌─────────────────┐      ┌──────────┐          │
│   │ Producer │ ───> │   Event Bus     │ ───> │ Consumer │          │
│   │ (Source) │      │   (Broker)      │      │ (Sink)   │          │
│   └──────────┘      └─────────────────┘      └──────────┘          │
│                              │                                      │
│                              ▼                                      │
│                     ┌─────────────────┐                            │
│                     │  Event Store    │                            │
│                     │  (опционально)  │                            │
│                     └─────────────────┘                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

Producer (Издатель)  — генерирует и публикует события
Event Bus (Шина)     — маршрутизирует события потребителям
Consumer (Подписчик) — получает и обрабатывает события
Event Store          — хранит историю всех событий
```

---

## Паттерны Event-Driven Architecture

### 1. Event Notification

Простейший паттерн — уведомление об изменении без передачи полных данных.

```python
# Producer публикует уведомление
class OrderService:
    def __init__(self, event_bus):
        self.event_bus = event_bus

    async def create_order(self, order_data: dict) -> Order:
        order = await self.repository.save(order_data)

        # Публикуем только ID — минимум данных
        await self.event_bus.publish(Event.create(
            event_type='order.created',
            source='order-service',
            data={'order_id': order.id}  # Только ID!
        ))

        return order

# Consumer должен сам запросить данные
class NotificationService:
    def __init__(self, order_client):
        self.order_client = order_client

    async def handle_order_created(self, event: Event):
        # Нужно делать дополнительный запрос
        order = await self.order_client.get_order(event.data['order_id'])
        await self.send_notification(order)
```

**Плюсы:** Минимальный размер событий, слабая связность
**Минусы:** Дополнительные запросы, возможны race conditions

### 2. Event-Carried State Transfer

Событие содержит все необходимые данные.

```python
# Producer включает полные данные в событие
class OrderService:
    async def create_order(self, order_data: dict) -> Order:
        order = await self.repository.save(order_data)

        # Публикуем полную информацию о заказе
        await self.event_bus.publish(Event.create(
            event_type='order.created',
            source='order-service',
            data={
                'order_id': order.id,
                'customer_id': order.customer_id,
                'customer_email': order.customer_email,
                'items': [
                    {'product_id': item.product_id,
                     'quantity': item.quantity,
                     'price': item.price}
                    for item in order.items
                ],
                'total': order.total,
                'created_at': order.created_at.isoformat()
            }
        ))

        return order

# Consumer имеет все нужные данные
class NotificationService:
    async def handle_order_created(self, event: Event):
        # Не нужны дополнительные запросы!
        order_data = event.data
        await self.send_email(
            to=order_data['customer_email'],
            subject='Order Confirmation',
            body=f"Your order #{order_data['order_id']} is confirmed"
        )
```

**Плюсы:** Автономность consumers, нет дополнительных запросов
**Минусы:** Большой размер событий, дублирование данных

### 3. Event Sourcing

Все изменения состояния сохраняются как последовательность событий.

```python
from abc import ABC, abstractmethod
from typing import List

class EventSourcedAggregate(ABC):
    """Базовый класс для Event Sourcing."""

    def __init__(self):
        self._uncommitted_events: List[Event] = []
        self._version = 0

    def apply_event(self, event: Event):
        """Применяет событие к состоянию."""
        handler = getattr(self, f'_apply_{event.event_type.replace(".", "_")}', None)
        if handler:
            handler(event)
        self._version += 1

    def load_from_history(self, events: List[Event]):
        """Восстанавливает состояние из истории событий."""
        for event in events:
            self.apply_event(event)

    @property
    def uncommitted_events(self) -> List[Event]:
        return self._uncommitted_events.copy()

    def clear_uncommitted_events(self):
        self._uncommitted_events.clear()

class Order(EventSourcedAggregate):
    """Заказ с Event Sourcing."""

    def __init__(self):
        super().__init__()
        self.id = None
        self.status = None
        self.items = []
        self.total = 0

    # Команды (изменяют состояние)
    def create(self, order_id: str, items: list):
        event = Event.create(
            event_type='order.created',
            source='order-aggregate',
            data={'order_id': order_id, 'items': items}
        )
        self._uncommitted_events.append(event)
        self.apply_event(event)

    def confirm(self):
        if self.status != 'pending':
            raise ValueError("Order must be pending to confirm")

        event = Event.create(
            event_type='order.confirmed',
            source='order-aggregate',
            data={'order_id': self.id}
        )
        self._uncommitted_events.append(event)
        self.apply_event(event)

    def cancel(self, reason: str):
        event = Event.create(
            event_type='order.cancelled',
            source='order-aggregate',
            data={'order_id': self.id, 'reason': reason}
        )
        self._uncommitted_events.append(event)
        self.apply_event(event)

    # Event handlers (применяют события к состоянию)
    def _apply_order_created(self, event: Event):
        self.id = event.data['order_id']
        self.items = event.data['items']
        self.status = 'pending'
        self.total = sum(item['price'] * item['quantity'] for item in self.items)

    def _apply_order_confirmed(self, event: Event):
        self.status = 'confirmed'

    def _apply_order_cancelled(self, event: Event):
        self.status = 'cancelled'

# Event Store
class EventStore:
    """Хранилище событий."""

    def __init__(self):
        self._events = {}  # stream_id -> List[Event]

    async def append(self, stream_id: str, events: List[Event], expected_version: int):
        """Добавляет события в поток с оптимистичной блокировкой."""
        current_events = self._events.get(stream_id, [])

        if len(current_events) != expected_version:
            raise ConcurrencyError(
                f"Expected version {expected_version}, but found {len(current_events)}"
            )

        self._events.setdefault(stream_id, []).extend(events)

    async def get_events(self, stream_id: str) -> List[Event]:
        """Получает все события потока."""
        return self._events.get(stream_id, [])

# Использование
async def create_order_example():
    event_store = EventStore()

    # Создаём заказ
    order = Order()
    order.create(
        order_id='order-123',
        items=[{'product_id': 'prod-1', 'quantity': 2, 'price': 100}]
    )

    # Сохраняем события
    await event_store.append(
        stream_id=f'order-{order.id}',
        events=order.uncommitted_events,
        expected_version=0
    )
    order.clear_uncommitted_events()

    # Восстанавливаем заказ из событий
    events = await event_store.get_events(f'order-{order.id}')
    restored_order = Order()
    restored_order.load_from_history(events)

    print(restored_order.status)  # 'pending'
    print(restored_order.total)   # 200
```

### 4. CQRS (Command Query Responsibility Segregation)

Разделение операций чтения и записи с Event Sourcing.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              CQRS + Event Sourcing                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────┐     Command      ┌─────────────┐     Events                │
│  │         │ ───────────────> │   Command   │ ──────────┐               │
│  │  User   │                  │   Handler   │           │               │
│  │         │                  └─────────────┘           ▼               │
│  └────┬────┘                                     ┌─────────────┐        │
│       │                                          │ Event Store │        │
│       │         Query         ┌─────────────┐    └──────┬──────┘        │
│       └─────────────────────> │   Query     │           │               │
│                               │   Handler   │           │               │
│                               └──────┬──────┘           │               │
│                                      │                  │               │
│                                      ▼                  ▼               │
│                               ┌─────────────┐    ┌─────────────┐        │
│                               │  Read Model │ <──│  Projector  │        │
│                               │  (Optimized)│    │             │        │
│                               └─────────────┘    └─────────────┘        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

```python
# Command Side
class CreateOrderCommand:
    def __init__(self, customer_id: str, items: list):
        self.customer_id = customer_id
        self.items = items

class OrderCommandHandler:
    def __init__(self, event_store: EventStore, event_bus):
        self.event_store = event_store
        self.event_bus = event_bus

    async def handle_create_order(self, command: CreateOrderCommand):
        order = Order()
        order.create(
            order_id=str(uuid.uuid4()),
            customer_id=command.customer_id,
            items=command.items
        )

        await self.event_store.append(
            stream_id=f'order-{order.id}',
            events=order.uncommitted_events,
            expected_version=0
        )

        # Публикуем события для построения read models
        for event in order.uncommitted_events:
            await self.event_bus.publish(event)

# Query Side
class OrderReadModel:
    """Оптимизированная модель для чтения."""
    def __init__(self):
        self.id: str
        self.customer_id: str
        self.customer_name: str  # Денормализовано!
        self.items_count: int    # Предвычислено!
        self.total: float
        self.status: str
        self.created_at: datetime

class OrderProjector:
    """Проектор — строит read model из событий."""

    def __init__(self, db):
        self.db = db

    async def handle_order_created(self, event: Event):
        read_model = OrderReadModel(
            id=event.data['order_id'],
            customer_id=event.data['customer_id'],
            customer_name=await self._get_customer_name(event.data['customer_id']),
            items_count=len(event.data['items']),
            total=event.data['total'],
            status='pending',
            created_at=event.timestamp
        )
        await self.db.orders_read.insert_one(read_model.__dict__)

    async def handle_order_confirmed(self, event: Event):
        await self.db.orders_read.update_one(
            {'id': event.data['order_id']},
            {'$set': {'status': 'confirmed'}}
        )

class OrderQueryHandler:
    """Обработчик запросов — читает из read model."""

    def __init__(self, db):
        self.db = db

    async def get_order(self, order_id: str) -> OrderReadModel:
        return await self.db.orders_read.find_one({'id': order_id})

    async def get_customer_orders(self, customer_id: str) -> List[OrderReadModel]:
        return await self.db.orders_read.find({'customer_id': customer_id}).to_list()
```

---

## Архитектурная диаграмма E-commerce системы

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         E-commerce Event-Driven System                      │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐                                                           │
│  │   Client    │                                                           │
│  └──────┬──────┘                                                           │
│         │ HTTP                                                              │
│         ▼                                                                   │
│  ┌─────────────┐                     ┌─────────────────────────────────┐  │
│  │ API Gateway │ ─────────────────── │           Kafka                  │  │
│  └──────┬──────┘                     │                                  │  │
│         │                            │  Topics:                         │  │
│         ▼                            │  ├── orders.created              │  │
│  ┌─────────────┐  publish ──────────>│  ├── orders.confirmed            │  │
│  │   Order     │                     │  ├── payments.processed          │  │
│  │   Service   │                     │  ├── inventory.reserved          │  │
│  └─────────────┘                     │  ├── inventory.shipped           │  │
│                                      │  └── notifications.send          │  │
│                                      │                                  │  │
│  ┌─────────────┐  subscribe ─────────│                                  │  │
│  │  Payment    │ <───────────────────│  (orders.created)                │  │
│  │  Service    │  publish ──────────>│  (payments.processed)            │  │
│  └─────────────┘                     │                                  │  │
│                                      │                                  │  │
│  ┌─────────────┐  subscribe ─────────│                                  │  │
│  │ Inventory   │ <───────────────────│  (payments.processed)            │  │
│  │  Service    │  publish ──────────>│  (inventory.reserved)            │  │
│  └─────────────┘                     │                                  │  │
│                                      │                                  │  │
│  ┌─────────────┐  subscribe ─────────│                                  │  │
│  │ Notification│ <───────────────────│  (orders.*, inventory.*)         │  │
│  │  Service    │                     │                                  │  │
│  └─────────────┘                     └─────────────────────────────────┘  │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Практический пример с Kafka

```python
# producer.py
from aiokafka import AIOKafkaProducer
import json

class KafkaEventPublisher:
    def __init__(self, bootstrap_servers: str):
        self.bootstrap_servers = bootstrap_servers
        self.producer = None

    async def start(self):
        self.producer = AIOKafkaProducer(
            bootstrap_servers=self.bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )
        await self.producer.start()

    async def stop(self):
        await self.producer.stop()

    async def publish(self, topic: str, event: Event):
        await self.producer.send_and_wait(
            topic=topic,
            value={
                'event_id': event.event_id,
                'event_type': event.event_type,
                'timestamp': event.timestamp.isoformat(),
                'source': event.source,
                'data': event.data
            }
        )

# consumer.py
from aiokafka import AIOKafkaConsumer
import json

class KafkaEventConsumer:
    def __init__(self, bootstrap_servers: str, group_id: str, topics: list):
        self.bootstrap_servers = bootstrap_servers
        self.group_id = group_id
        self.topics = topics
        self.handlers = {}

    def register_handler(self, event_type: str, handler):
        self.handlers[event_type] = handler

    async def start(self):
        consumer = AIOKafkaConsumer(
            *self.topics,
            bootstrap_servers=self.bootstrap_servers,
            group_id=self.group_id,
            value_deserializer=lambda v: json.loads(v.decode('utf-8')),
            auto_offset_reset='earliest',
            enable_auto_commit=False
        )

        await consumer.start()

        try:
            async for msg in consumer:
                event_data = msg.value
                event_type = event_data['event_type']

                handler = self.handlers.get(event_type)
                if handler:
                    try:
                        await handler(event_data)
                        await consumer.commit()
                    except Exception as e:
                        # Dead Letter Queue или retry
                        await self.handle_error(event_data, e)
        finally:
            await consumer.stop()

# order_service.py
class OrderService:
    def __init__(self, publisher: KafkaEventPublisher, repository):
        self.publisher = publisher
        self.repository = repository

    async def create_order(self, order_data: dict) -> Order:
        order = await self.repository.save(order_data)

        event = Event.create(
            event_type='order.created',
            source='order-service',
            data={
                'order_id': order.id,
                'customer_id': order.customer_id,
                'items': order.items,
                'total': order.total
            }
        )

        await self.publisher.publish('orders', event)

        return order

# payment_service.py
class PaymentService:
    def __init__(self, publisher: KafkaEventPublisher, payment_gateway):
        self.publisher = publisher
        self.payment_gateway = payment_gateway

    async def handle_order_created(self, event_data: dict):
        """Обработчик события создания заказа."""
        order_id = event_data['data']['order_id']
        total = event_data['data']['total']

        result = await self.payment_gateway.process(order_id, total)

        await self.publisher.publish('payments', Event.create(
            event_type='payment.processed',
            source='payment-service',
            data={
                'order_id': order_id,
                'payment_id': result.payment_id,
                'status': result.status
            }
        ))

# main.py
async def main():
    publisher = KafkaEventPublisher('localhost:9092')
    await publisher.start()

    consumer = KafkaEventConsumer(
        bootstrap_servers='localhost:9092',
        group_id='payment-service',
        topics=['orders']
    )

    payment_service = PaymentService(publisher, payment_gateway)
    consumer.register_handler('order.created', payment_service.handle_order_created)

    await consumer.start()
```

---

## Saga Pattern

Для распределённых транзакций в EDA используется Saga Pattern.

```python
# Choreography-based Saga (через события)

class OrderSaga:
    """
    Saga для создания заказа:
    1. Order Created → Payment Service
    2. Payment Processed → Inventory Service
    3. Inventory Reserved → Shipping Service
    4. Если что-то не так — компенсирующие действия
    """

    async def handle_payment_failed(self, event_data: dict):
        """Компенсирующее действие при ошибке оплаты."""
        order_id = event_data['data']['order_id']

        # Отменяем заказ
        await self.publisher.publish('orders', Event.create(
            event_type='order.cancelled',
            source='order-saga',
            data={
                'order_id': order_id,
                'reason': 'Payment failed'
            }
        ))

    async def handle_inventory_failed(self, event_data: dict):
        """Компенсирующее действие при нехватке товара."""
        order_id = event_data['data']['order_id']
        payment_id = event_data['data']['payment_id']

        # Возвращаем деньги
        await self.publisher.publish('payments', Event.create(
            event_type='payment.refund_requested',
            source='order-saga',
            data={
                'payment_id': payment_id,
                'reason': 'Inventory not available'
            }
        ))

        # Отменяем заказ
        await self.publisher.publish('orders', Event.create(
            event_type='order.cancelled',
            source='order-saga',
            data={
                'order_id': order_id,
                'reason': 'Inventory not available'
            }
        ))
```

```
Saga Flow:

SUCCESS PATH:
Order Created ──> Payment Processed ──> Inventory Reserved ──> Order Shipped
     │                    │                     │                    │
     ▼                    ▼                     ▼                    ▼
  pending             payment_ok           inventory_ok          shipped

COMPENSATION PATH (Payment Failed):
Order Created ──> Payment Failed
     │                  │
     ▼                  ▼
  pending ────────> cancelled

COMPENSATION PATH (Inventory Failed):
Order Created ──> Payment Processed ──> Inventory Failed
     │                    │                    │
     ▼                    ▼                    ▼
  pending             payment_ok          refund + cancel
```

---

## Best Practices

### 1. Идемпотентность обработчиков

```python
class IdempotentEventHandler:
    def __init__(self, redis, handler):
        self.redis = redis
        self.handler = handler

    async def handle(self, event_data: dict):
        event_id = event_data['event_id']

        # Проверяем, обрабатывали ли мы это событие
        if await self.redis.exists(f'processed:{event_id}'):
            return  # Уже обработано

        await self.handler(event_data)

        # Помечаем как обработанное (с TTL)
        await self.redis.setex(f'processed:{event_id}', 86400, '1')
```

### 2. Версионирование событий

```python
class EventV1:
    event_type = 'order.created'
    version = 1

    def __init__(self, order_id: str, total: float):
        self.order_id = order_id
        self.total = total

class EventV2:
    event_type = 'order.created'
    version = 2

    def __init__(self, order_id: str, total: float, currency: str):
        self.order_id = order_id
        self.total = total
        self.currency = currency  # Новое поле

class EventUpgrader:
    def upgrade_to_v2(self, event_v1: EventV1) -> EventV2:
        return EventV2(
            order_id=event_v1.order_id,
            total=event_v1.total,
            currency='USD'  # Значение по умолчанию
        )
```

### 3. Dead Letter Queue

```python
class DeadLetterHandler:
    def __init__(self, dlq_publisher):
        self.dlq_publisher = dlq_publisher

    async def handle_failed_event(self, event_data: dict, error: Exception, retry_count: int):
        if retry_count >= 3:
            # Отправляем в DLQ
            await self.dlq_publisher.publish('dead-letter-queue', {
                'original_event': event_data,
                'error': str(error),
                'retry_count': retry_count,
                'failed_at': datetime.utcnow().isoformat()
            })
        else:
            # Retry с exponential backoff
            delay = 2 ** retry_count
            await asyncio.sleep(delay)
            raise  # Повторная попытка
```

---

## Типичные ошибки

### 1. Потеря событий

```python
# ❌ Плохо: публикация после коммита
async def create_order(self, data):
    order = await self.repository.save(data)
    await self.db.commit()  # Если упадёт после этого...
    await self.publisher.publish(event)  # ...событие потеряется!

# ✅ Хорошо: Transactional Outbox Pattern
async def create_order(self, data):
    async with self.db.transaction():
        order = await self.repository.save(data)
        # Сохраняем событие в ту же транзакцию
        await self.outbox.save(event)

    # Отдельный процесс публикует события из outbox
```

### 2. Отсутствие идемпотентности

```python
# ❌ Плохо: дублирование при повторной обработке
async def handle_payment(self, event):
    await self.gateway.charge(event.data['amount'])  # Двойное списание!

# ✅ Хорошо: идемпотентная обработка
async def handle_payment(self, event):
    payment_id = event.data['payment_id']
    existing = await self.repository.find_by_id(payment_id)
    if existing:
        return existing  # Уже обработано

    result = await self.gateway.charge(event.data['amount'])
    await self.repository.save(result)
    return result
```

### 3. Сильная связность через события

```python
# ❌ Плохо: Consumer зависит от внутренней структуры Producer
event.data = {
    'internal_order_id': order._internal_id,  # Внутренняя деталь!
    'db_status_code': 15  # Что это значит?
}

# ✅ Хорошо: Публичный контракт события
event.data = {
    'order_id': 'uuid-публичный',
    'status': 'confirmed'  # Понятное значение
}
```

---

## Сравнение с другими подходами

| Критерий | Request/Response | Event-Driven |
|----------|-----------------|--------------|
| Связность | Сильная | Слабая |
| Масштабируемость | Ограниченная | Высокая |
| Сложность | Низкая | Высокая |
| Отладка | Простая | Сложная |
| Отказоустойчивость | Низкая | Высокая |
| Consistency | Strong | Eventual |

---

## Заключение

Event-Driven Architecture — мощный подход для построения масштабируемых, слабосвязанных систем. Ключевые принципы:

1. **События неизменяемы** — записывайте факты, а не намерения
2. **Consumers должны быть идемпотентными** — обрабатывайте повторы
3. **Версионируйте события** — контракт может меняться
4. **Используйте Saga для транзакций** — компенсируйте ошибки
5. **Мониторьте потоки событий** — observability критически важна
