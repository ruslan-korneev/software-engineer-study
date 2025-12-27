# Publish-Subscribe (Pub/Sub)

## Определение

**Publish-Subscribe (Pub/Sub)** — это паттерн обмена сообщениями, в котором отправители сообщений (publishers) не отправляют сообщения напрямую получателям (subscribers). Вместо этого сообщения классифицируются по категориям (topics/channels) и доставляются всем подписчикам, заинтересованным в этих категориях.

Ключевая идея: **издатели и подписчики не знают друг о друге**. Это обеспечивает слабую связанность и высокую масштабируемость системы.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           Message Broker                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                         Topic: "orders"                              │ │
│  │    ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐              │ │
│  │    │ msg 1   │  │ msg 2   │  │ msg 3   │  │ msg 4   │  ...         │ │
│  │    └─────────┘  └─────────┘  └─────────┘  └─────────┘              │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                         Topic: "payments"                            │ │
│  │    ┌─────────┐  ┌─────────┐                                         │ │
│  │    │ msg 1   │  │ msg 2   │  ...                                    │ │
│  │    └─────────┘  └─────────┘                                         │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
         ▲                                           │
         │ publish                                   │ deliver
         │                                           ▼
┌─────────────────┐                    ┌─────────────────────────────────┐
│   Publishers    │                    │          Subscribers            │
│                 │                    │                                 │
│ ┌─────────────┐ │                    │ ┌───────────┐ ┌───────────────┐│
│ │Order Service│ │                    │ │Inventory  │ │Notification   ││
│ └─────────────┘ │                    │ │Service    │ │Service        ││
│ ┌─────────────┐ │                    │ └───────────┘ └───────────────┘│
│ │Payment      │ │                    │ ┌───────────┐ ┌───────────────┐│
│ │Service      │ │                    │ │Analytics  │ │Audit          ││
│ └─────────────┘ │                    │ │Service    │ │Service        ││
└─────────────────┘                    └─────────────────────────────────┘
```

## Ключевые характеристики

### 1. Слабая связанность (Decoupling)

Издатели и подписчики полностью независимы:

```python
# Publisher не знает о подписчиках
class OrderService:
    def __init__(self, event_bus):
        self.event_bus = event_bus

    def create_order(self, order_data: dict):
        order = Order(**order_data)
        order.save()

        # Просто публикуем событие, не зная кто его получит
        self.event_bus.publish("order.created", {
            "order_id": order.id,
            "user_id": order.user_id,
            "total": order.total
        })
        return order

# Subscribers не знают об издателях
class NotificationService:
    def __init__(self, event_bus):
        event_bus.subscribe("order.created", self.on_order_created)

    def on_order_created(self, event_data: dict):
        # Обрабатываем событие независимо
        self.send_confirmation_email(event_data["user_id"], event_data["order_id"])

class InventoryService:
    def __init__(self, event_bus):
        event_bus.subscribe("order.created", self.on_order_created)

    def on_order_created(self, event_data: dict):
        # Свой обработчик того же события
        self.reserve_items(event_data["order_id"])
```

### 2. Асинхронная коммуникация

Сообщения обрабатываются независимо от времени публикации:

```python
import asyncio
from typing import Callable, Dict, List

class AsyncEventBus:
    def __init__(self):
        self._subscribers: Dict[str, List[Callable]] = {}
        self._queue: asyncio.Queue = asyncio.Queue()

    def subscribe(self, topic: str, handler: Callable):
        if topic not in self._subscribers:
            self._subscribers[topic] = []
        self._subscribers[topic].append(handler)

    async def publish(self, topic: str, message: dict):
        """Публикация неблокирующая - сообщение попадает в очередь"""
        await self._queue.put((topic, message))

    async def process_messages(self):
        """Фоновый процесс обработки сообщений"""
        while True:
            topic, message = await self._queue.get()
            handlers = self._subscribers.get(topic, [])

            # Параллельная обработка всеми подписчиками
            tasks = [asyncio.create_task(h(message)) for h in handlers]
            await asyncio.gather(*tasks, return_exceptions=True)

            self._queue.task_done()

# Использование
async def main():
    bus = AsyncEventBus()

    # Подписка
    async def log_event(msg):
        await asyncio.sleep(0.1)  # Симуляция I/O
        print(f"Logged: {msg}")

    bus.subscribe("user.created", log_event)

    # Публикация (не ждёт обработки)
    await bus.publish("user.created", {"user_id": 123})

    # Запуск обработчика в фоне
    asyncio.create_task(bus.process_messages())
```

### 3. Многоадресная доставка (Multicast)

Одно сообщение доставляется всем заинтересованным подписчикам:

```python
class EventBus:
    def __init__(self):
        self._subscribers = {}

    def subscribe(self, pattern: str, handler):
        """Подписка с поддержкой wildcard"""
        if pattern not in self._subscribers:
            self._subscribers[pattern] = []
        self._subscribers[pattern].append(handler)

    def publish(self, topic: str, message: dict):
        """Доставка всем подходящим подписчикам"""
        for pattern, handlers in self._subscribers.items():
            if self._matches(pattern, topic):
                for handler in handlers:
                    handler(message)

    def _matches(self, pattern: str, topic: str) -> bool:
        """Проверка соответствия топика паттерну"""
        if pattern == topic:
            return True
        if pattern.endswith('.*'):
            prefix = pattern[:-2]
            return topic.startswith(prefix + '.')
        if pattern.endswith('.#'):
            prefix = pattern[:-2]
            return topic.startswith(prefix)
        return False

# Использование
bus = EventBus()

# Подписка на конкретный топик
bus.subscribe("order.created", lambda m: print(f"Order created: {m}"))

# Подписка на все события заказов
bus.subscribe("order.*", lambda m: print(f"Order event: {m}"))

# Подписка на все события
bus.subscribe("#", lambda m: print(f"Any event: {m}"))

# Публикация
bus.publish("order.created", {"id": 1})
# Выведет все три сообщения
```

### 4. Масштабируемость

Легко добавлять новых издателей и подписчиков:

```
Горизонтальное масштабирование:

                    ┌─────────────────┐
                    │  Message Broker │
                    │  (Redis/Kafka)  │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  Subscriber   │   │  Subscriber   │   │  Subscriber   │
│  Instance 1   │   │  Instance 2   │   │  Instance 3   │
└───────────────┘   └───────────────┘   └───────────────┘

Каждый instance обрабатывает часть сообщений (Consumer Groups)
```

## Когда использовать

### Идеальные случаи применения

1. **Микросервисная архитектура** — взаимодействие между сервисами
2. **Event-driven системы** — реакция на события в реальном времени
3. **Уведомления и оповещения** — рассылка множеству получателей
4. **Логирование и мониторинг** — сбор данных из разных источников
5. **Интеграция систем** — слабосвязанное взаимодействие

### Примеры use cases

```python
# 1. E-commerce: событие заказа обрабатывается многими сервисами
class OrderCreatedEvent:
    topic = "orders.created"

    def __init__(self, order_id: str, items: list, total: float):
        self.order_id = order_id
        self.items = items
        self.total = total

# Подписчики
class InventoryHandler:
    """Резервирует товары"""
    def handle(self, event: OrderCreatedEvent):
        for item in event.items:
            self.inventory.reserve(item['product_id'], item['quantity'])

class PaymentHandler:
    """Инициирует оплату"""
    def handle(self, event: OrderCreatedEvent):
        self.payment_gateway.charge(event.order_id, event.total)

class NotificationHandler:
    """Отправляет уведомление"""
    def handle(self, event: OrderCreatedEvent):
        self.email_service.send_order_confirmation(event.order_id)

class AnalyticsHandler:
    """Записывает в аналитику"""
    def handle(self, event: OrderCreatedEvent):
        self.analytics.track("order_created", {"total": event.total})


# 2. Real-time система: чат
class ChatRoom:
    def __init__(self, room_id: str, pubsub):
        self.room_id = room_id
        self.pubsub = pubsub

    def send_message(self, user_id: str, text: str):
        self.pubsub.publish(f"chat.{self.room_id}", {
            "type": "message",
            "user_id": user_id,
            "text": text,
            "timestamp": time.time()
        })

    def subscribe_user(self, user_id: str, websocket):
        def handler(message):
            websocket.send_json(message)

        self.pubsub.subscribe(f"chat.{self.room_id}", handler)
```

## Преимущества и недостатки

### Преимущества

| Преимущество | Описание |
|-------------|----------|
| **Слабая связанность** | Компоненты независимы друг от друга |
| **Масштабируемость** | Легко добавлять новых издателей/подписчиков |
| **Асинхронность** | Publisher не ждёт обработки сообщения |
| **Гибкость** | Динамическая подписка/отписка |
| **Отказоустойчивость** | Сбой подписчика не влияет на других |
| **Расширяемость** | Добавление функций без изменения существующего кода |

### Недостатки

| Недостаток | Описание |
|-----------|----------|
| **Сложность отладки** | Трудно отследить путь сообщения |
| **Гарантии доставки** | Требуется дополнительная работа для exactly-once |
| **Порядок сообщений** | Не гарантируется без дополнительных мер |
| **Латентность** | Добавляется задержка на обработку очередью |
| **Сложность тестирования** | Асинхронные сценарии сложнее тестировать |
| **Overhead** | Дополнительная инфраструктура (message broker) |

## Примеры реализации

### In-Memory Pub/Sub (для небольших приложений)

```python
from typing import Callable, Dict, List, Any
from dataclasses import dataclass
from datetime import datetime
import threading
import uuid

@dataclass
class Message:
    id: str
    topic: str
    data: Any
    timestamp: datetime

    @classmethod
    def create(cls, topic: str, data: Any) -> 'Message':
        return cls(
            id=str(uuid.uuid4()),
            topic=topic,
            data=data,
            timestamp=datetime.utcnow()
        )

class InMemoryPubSub:
    def __init__(self):
        self._subscribers: Dict[str, List[Callable[[Message], None]]] = {}
        self._lock = threading.Lock()

    def subscribe(self, topic: str, handler: Callable[[Message], None]) -> Callable:
        """Подписка на топик. Возвращает функцию отписки."""
        with self._lock:
            if topic not in self._subscribers:
                self._subscribers[topic] = []
            self._subscribers[topic].append(handler)

        def unsubscribe():
            with self._lock:
                self._subscribers[topic].remove(handler)

        return unsubscribe

    def publish(self, topic: str, data: Any) -> Message:
        """Публикация сообщения в топик."""
        message = Message.create(topic, data)

        with self._lock:
            handlers = self._subscribers.get(topic, []).copy()

        for handler in handlers:
            try:
                handler(message)
            except Exception as e:
                print(f"Handler error: {e}")

        return message

# Использование
pubsub = InMemoryPubSub()

def order_logger(message: Message):
    print(f"[LOG] Order event: {message.data}")

def order_notifier(message: Message):
    print(f"[NOTIFY] Sending notification for order {message.data['order_id']}")

# Подписка
unsubscribe_logger = pubsub.subscribe("orders", order_logger)
unsubscribe_notifier = pubsub.subscribe("orders", order_notifier)

# Публикация
pubsub.publish("orders", {"order_id": "123", "action": "created"})

# Отписка
unsubscribe_logger()
```

### Redis Pub/Sub

```python
import redis
import json
import threading
from typing import Callable, Dict

class RedisPubSub:
    def __init__(self, host: str = 'localhost', port: int = 6379):
        self.redis = redis.Redis(host=host, port=port, decode_responses=True)
        self.pubsub = self.redis.pubsub()
        self._handlers: Dict[str, Callable] = {}
        self._listener_thread = None

    def subscribe(self, channel: str, handler: Callable[[dict], None]):
        """Подписка на канал Redis."""
        self._handlers[channel] = handler
        self.pubsub.subscribe(**{channel: self._message_handler})

        if self._listener_thread is None:
            self._listener_thread = self.pubsub.run_in_thread(sleep_time=0.001)

    def _message_handler(self, message):
        """Внутренний обработчик сообщений Redis."""
        if message['type'] == 'message':
            channel = message['channel']
            handler = self._handlers.get(channel)
            if handler:
                try:
                    data = json.loads(message['data'])
                    handler(data)
                except json.JSONDecodeError:
                    handler(message['data'])

    def publish(self, channel: str, data: dict):
        """Публикация сообщения в канал."""
        self.redis.publish(channel, json.dumps(data))

    def close(self):
        """Закрытие соединения."""
        if self._listener_thread:
            self._listener_thread.stop()
        self.pubsub.close()
        self.redis.close()

# Использование
pubsub = RedisPubSub()

def handle_user_event(data: dict):
    print(f"User event received: {data}")

pubsub.subscribe("users", handle_user_event)
pubsub.publish("users", {"action": "created", "user_id": 42})
```

### Kafka Producer/Consumer

```python
from kafka import KafkaProducer, KafkaConsumer
import json
from typing import Callable
import threading

class KafkaPubSub:
    def __init__(self, bootstrap_servers: list = ['localhost:9092']):
        self.bootstrap_servers = bootstrap_servers
        self._producer = None
        self._consumers: Dict[str, KafkaConsumer] = {}
        self._consumer_threads: Dict[str, threading.Thread] = {}

    @property
    def producer(self):
        if self._producer is None:
            self._producer = KafkaProducer(
                bootstrap_servers=self.bootstrap_servers,
                value_serializer=lambda v: json.dumps(v).encode('utf-8'),
                key_serializer=lambda k: k.encode('utf-8') if k else None
            )
        return self._producer

    def publish(self, topic: str, message: dict, key: str = None):
        """Публикация сообщения в Kafka topic."""
        future = self.producer.send(topic, value=message, key=key)
        # Синхронное ожидание (можно сделать асинхронным)
        return future.get(timeout=10)

    def subscribe(
        self,
        topic: str,
        handler: Callable[[dict], None],
        group_id: str = None
    ):
        """Подписка на Kafka topic с автоматическим запуском consumer."""
        consumer = KafkaConsumer(
            topic,
            bootstrap_servers=self.bootstrap_servers,
            group_id=group_id or f"{topic}-consumer-group",
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            auto_offset_reset='earliest',
            enable_auto_commit=True
        )

        def consume_loop():
            for message in consumer:
                try:
                    handler(message.value)
                except Exception as e:
                    print(f"Handler error: {e}")

        thread = threading.Thread(target=consume_loop, daemon=True)
        thread.start()

        self._consumers[topic] = consumer
        self._consumer_threads[topic] = thread

    def close(self):
        """Закрытие всех соединений."""
        if self._producer:
            self._producer.close()
        for consumer in self._consumers.values():
            consumer.close()

# Использование
kafka = KafkaPubSub(['kafka:9092'])

# Publisher
kafka.publish("orders", {
    "order_id": "ord-123",
    "user_id": "user-456",
    "items": [{"product": "Widget", "qty": 2}]
}, key="ord-123")

# Subscriber
def process_order(order_data: dict):
    print(f"Processing order: {order_data['order_id']}")

kafka.subscribe("orders", process_order, group_id="order-processor")
```

## Best Practices и антипаттерны

### Best Practices

```python
# 1. Идемпотентные обработчики (можно безопасно повторять)
class OrderHandler:
    def __init__(self, db):
        self.db = db

    def handle(self, message: dict):
        order_id = message['order_id']

        # Проверяем, не обработали ли уже
        if self.db.is_processed(order_id):
            return  # Идемпотентность!

        # Обрабатываем
        self.process_order(message)
        self.db.mark_processed(order_id)


# 2. Структурированные события с версионированием
from dataclasses import dataclass, asdict
from typing import Optional
from datetime import datetime

@dataclass
class Event:
    version: str = "1.0"
    timestamp: str = None
    correlation_id: Optional[str] = None

    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = datetime.utcnow().isoformat()

    def to_dict(self) -> dict:
        return asdict(self)

@dataclass
class OrderCreatedEvent(Event):
    event_type: str = "order.created"
    order_id: str = None
    user_id: str = None
    total: float = 0.0


# 3. Dead Letter Queue для необработанных сообщений
class RobustSubscriber:
    def __init__(self, pubsub, dlq_topic: str):
        self.pubsub = pubsub
        self.dlq_topic = dlq_topic
        self.max_retries = 3

    def subscribe(self, topic: str, handler: Callable):
        def wrapper(message: dict):
            retries = message.get('_retries', 0)

            try:
                handler(message)
            except Exception as e:
                if retries < self.max_retries:
                    message['_retries'] = retries + 1
                    message['_last_error'] = str(e)
                    self.pubsub.publish(topic, message)  # Retry
                else:
                    # Отправляем в Dead Letter Queue
                    self.pubsub.publish(self.dlq_topic, {
                        'original_topic': topic,
                        'message': message,
                        'error': str(e)
                    })

        self.pubsub.subscribe(topic, wrapper)


# 4. Graceful shutdown
import signal
import sys

class GracefulSubscriber:
    def __init__(self):
        self.running = True
        signal.signal(signal.SIGTERM, self._handle_shutdown)
        signal.signal(signal.SIGINT, self._handle_shutdown)

    def _handle_shutdown(self, signum, frame):
        print("Shutting down gracefully...")
        self.running = False

    def consume(self, consumer):
        while self.running:
            messages = consumer.poll(timeout_ms=1000)
            for message in messages:
                if not self.running:
                    break
                self.process(message)

        consumer.close()
```

### Антипаттерны

```python
# ❌ АНТИПАТТЕРН: Синхронная обработка блокирует publisher
class BadPublisher:
    def publish(self, topic: str, message: dict):
        for subscriber in self.subscribers[topic]:
            subscriber.handle(message)  # Блокирующий вызов!
        # Publisher ждёт завершения всех обработчиков


# ❌ АНТИПАТТЕРН: Большие сообщения
class BadEventPublisher:
    def publish_order(self, order):
        # Отправляем весь объект с вложенными данными - ПЛОХО!
        self.pubsub.publish("orders", {
            "order": order,
            "user": order.user,  # Полный объект пользователя
            "items": [item.to_full_dict() for item in order.items],  # Много данных
            "attachments": [a.binary_data for a in order.attachments]  # Бинарные данные!
        })

# ✅ ПРАВИЛЬНО: Только необходимые данные
class GoodEventPublisher:
    def publish_order(self, order):
        self.pubsub.publish("orders", {
            "order_id": order.id,
            "user_id": order.user_id,
            "total": order.total
            # Подписчик сам загрузит детали при необходимости
        })


# ❌ АНТИПАТТЕРН: Игнорирование ошибок
class BadSubscriber:
    def handle(self, message):
        # Ошибка потеряется!
        try:
            self.process(message)
        except:
            pass  # Сообщение потеряно навсегда


# ❌ АНТИПАТТЕРН: Порядок зависимых событий
class BadOrderProcessor:
    def __init__(self, pubsub):
        # Эти события могут прийти в неправильном порядке!
        pubsub.subscribe("order.created", self.on_created)
        pubsub.subscribe("order.paid", self.on_paid)

    def on_paid(self, msg):
        order = self.get_order(msg['order_id'])  # Может ещё не существовать!
        order.mark_paid()
```

## Связанные паттерны

### Observer vs Pub/Sub

```
Observer (локальный):
┌────────────────┐      notify()      ┌──────────────┐
│    Subject     │──────────────────► │   Observer   │
│  (Observable)  │◄──────────────────│              │
└────────────────┘    getState()      └──────────────┘

Pub/Sub (распределённый):
┌──────────────┐   publish   ┌──────────┐   deliver   ┌────────────┐
│  Publisher   │────────────►│  Broker  │────────────►│ Subscriber │
└──────────────┘             └──────────┘             └────────────┘
     Не знает                                              Не знает
     о подписчиках                                         об издателе
```

### Event Sourcing

Pub/Sub часто используется вместе с Event Sourcing:

```python
class EventStore:
    def __init__(self, pubsub):
        self.events = []
        self.pubsub = pubsub

    def append(self, event: Event):
        # Сохраняем событие
        self.events.append(event)

        # Публикуем для подписчиков
        self.pubsub.publish(event.event_type, event.to_dict())
```

### CQRS (Command Query Responsibility Segregation)

Pub/Sub соединяет команды и запросы:

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Command    │───►│   Command    │───►│    Event     │
│   Handler    │    │    Store     │    │    Bus       │
└──────────────┘    └──────────────┘    └──────┬───────┘
                                               │
                                               ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│    Query     │◄───│    Read      │◄───│   Projector  │
│   Handler    │    │    Store     │    │              │
└──────────────┘    └──────────────┘    └──────────────┘
```

## Ресурсы для изучения

### Технологии
- **Apache Kafka** — распределённая платформа стриминга
- **RabbitMQ** — message broker с поддержкой Pub/Sub
- **Redis Pub/Sub** — простой in-memory pub/sub
- **AWS SNS/SQS** — облачный pub/sub от Amazon
- **Google Cloud Pub/Sub** — managed pub/sub сервис

### Книги
- **"Enterprise Integration Patterns"** — Gregor Hohpe, Bobby Woolf
- **"Designing Data-Intensive Applications"** — Martin Kleppmann
- **"Building Event-Driven Microservices"** — Adam Bellemare

### Онлайн-ресурсы
- [Kafka Documentation](https://kafka.apache.org/documentation/)
- [RabbitMQ Tutorials](https://www.rabbitmq.com/getstarted.html)
- [Martin Fowler - Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)

---

**Следующая тема**: [Controller-Responder](./03-controller-responder.md)
