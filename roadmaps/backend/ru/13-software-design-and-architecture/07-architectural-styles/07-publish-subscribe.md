# Publish-Subscribe Pattern

## Что такое паттерн Publish-Subscribe?

Publish-Subscribe (Pub/Sub) — это паттерн обмена сообщениями, в котором отправители сообщений (publishers) не знают о конкретных получателях (subscribers). Вместо этого сообщения публикуются в топики (темы), а подписчики получают только те сообщения, на которые они подписаны.

> "Pub/Sub обеспечивает полную развязку между производителями и потребителями сообщений."

## Основные концепции

```
┌───────────────────────────────────────────────────────────────┐
│                    Publish-Subscribe System                    │
│                                                                │
│  ┌────────────┐                           ┌────────────────┐  │
│  │ Publisher  │───────┐                   │  Subscriber A  │  │
│  │     1      │       │                   │   (Topic: X)   │  │
│  └────────────┘       │                   └────────────────┘  │
│                       ▼                           ▲           │
│  ┌────────────┐  ┌─────────┐  ┌─────────┐        │           │
│  │ Publisher  │─►│ Topic X │──┤ Message │────────┘           │
│  │     2      │  └─────────┘  │  Router │                    │
│  └────────────┘               └─────────┘                    │
│                                   │                          │
│  ┌────────────┐  ┌─────────┐     │      ┌────────────────┐  │
│  │ Publisher  │─►│ Topic Y │─────┴─────►│  Subscriber B  │  │
│  │     3      │  └─────────┘            │  (Topics: X,Y) │  │
│  └────────────┘                         └────────────────┘  │
│                                                              │
└───────────────────────────────────────────────────────────────┘
```

### Ключевые компоненты

1. **Publisher (Издатель)** — отправляет сообщения в топик
2. **Subscriber (Подписчик)** — получает сообщения из подписанных топиков
3. **Topic (Топик)** — именованный канал для сообщений определённого типа
4. **Message Broker** — посредник, управляющий топиками и доставкой

## Реализация простого Pub/Sub

```python
from typing import Callable, Dict, List, Set, Any
from dataclasses import dataclass, field
from datetime import datetime
from abc import ABC, abstractmethod
import asyncio
import uuid
import logging

@dataclass
class Message:
    """Сообщение в системе Pub/Sub."""
    id: str = field(default_factory=lambda: str(uuid.uuid4()))
    topic: str = ""
    payload: Any = None
    timestamp: datetime = field(default_factory=datetime.utcnow)
    headers: Dict[str, str] = field(default_factory=dict)

    def to_dict(self) -> dict:
        return {
            "id": self.id,
            "topic": self.topic,
            "payload": self.payload,
            "timestamp": self.timestamp.isoformat(),
            "headers": self.headers
        }


# Тип callback для подписчика
MessageHandler = Callable[[Message], None]
AsyncMessageHandler = Callable[[Message], asyncio.coroutine]


class PubSubBroker:
    """Простой in-memory Pub/Sub брокер."""

    def __init__(self):
        self._subscriptions: Dict[str, Set[MessageHandler]] = {}
        self._logger = logging.getLogger("pubsub")

    def subscribe(self, topic: str, handler: MessageHandler) -> None:
        """Подписка на топик."""
        if topic not in self._subscriptions:
            self._subscriptions[topic] = set()

        self._subscriptions[topic].add(handler)
        self._logger.info(f"Subscribed to topic: {topic}")

    def unsubscribe(self, topic: str, handler: MessageHandler) -> None:
        """Отписка от топика."""
        if topic in self._subscriptions:
            self._subscriptions[topic].discard(handler)

    def publish(self, topic: str, payload: Any, headers: Dict[str, str] = None) -> str:
        """Публикация сообщения в топик."""
        message = Message(
            topic=topic,
            payload=payload,
            headers=headers or {}
        )

        self._logger.info(f"Publishing to {topic}: {message.id}")

        handlers = self._subscriptions.get(topic, set())

        for handler in handlers:
            try:
                handler(message)
            except Exception as e:
                self._logger.error(f"Handler error: {e}")

        return message.id

    def get_topics(self) -> List[str]:
        """Получение списка активных топиков."""
        return list(self._subscriptions.keys())

    def get_subscriber_count(self, topic: str) -> int:
        """Количество подписчиков топика."""
        return len(self._subscriptions.get(topic, set()))


# Пример использования
def main():
    broker = PubSubBroker()

    # Подписчики
    def order_handler(msg: Message):
        print(f"Order service received: {msg.payload}")

    def notification_handler(msg: Message):
        print(f"Notification service received: {msg.payload}")

    def analytics_handler(msg: Message):
        print(f"Analytics service received: {msg.payload}")

    # Подписки
    broker.subscribe("orders.created", order_handler)
    broker.subscribe("orders.created", notification_handler)
    broker.subscribe("orders.created", analytics_handler)

    broker.subscribe("orders.shipped", notification_handler)

    # Публикация
    broker.publish("orders.created", {"order_id": "123", "user_id": "456"})
    broker.publish("orders.shipped", {"order_id": "123", "tracking": "TR123"})


if __name__ == "__main__":
    main()
```

## Асинхронный Pub/Sub

```python
import asyncio
from typing import Dict, Set, List
from dataclasses import dataclass, field
from datetime import datetime
import uuid

@dataclass
class AsyncMessage:
    """Асинхронное сообщение."""
    id: str = field(default_factory=lambda: str(uuid.uuid4()))
    topic: str = ""
    payload: Any = None
    timestamp: datetime = field(default_factory=datetime.utcnow)


class AsyncPubSubBroker:
    """Асинхронный Pub/Sub брокер."""

    def __init__(self, max_queue_size: int = 1000):
        self._subscriptions: Dict[str, Set[AsyncMessageHandler]] = {}
        self._queues: Dict[str, asyncio.Queue] = {}
        self._max_queue_size = max_queue_size
        self._running = False
        self._workers: List[asyncio.Task] = []

    async def start(self) -> None:
        """Запуск брокера."""
        self._running = True
        # Запускаем воркеры для каждого топика
        for topic in self._subscriptions:
            worker = asyncio.create_task(self._process_topic(topic))
            self._workers.append(worker)

    async def stop(self) -> None:
        """Остановка брокера."""
        self._running = False
        for worker in self._workers:
            worker.cancel()
        await asyncio.gather(*self._workers, return_exceptions=True)

    def subscribe(self, topic: str, handler: AsyncMessageHandler) -> None:
        """Подписка на топик."""
        if topic not in self._subscriptions:
            self._subscriptions[topic] = set()
            self._queues[topic] = asyncio.Queue(maxsize=self._max_queue_size)

            # Запускаем воркер для нового топика если брокер работает
            if self._running:
                worker = asyncio.create_task(self._process_topic(topic))
                self._workers.append(worker)

        self._subscriptions[topic].add(handler)

    async def publish(self, topic: str, payload: Any) -> str:
        """Публикация сообщения."""
        message = AsyncMessage(topic=topic, payload=payload)

        if topic in self._queues:
            await self._queues[topic].put(message)

        return message.id

    async def _process_topic(self, topic: str) -> None:
        """Обработка сообщений топика."""
        queue = self._queues[topic]

        while self._running:
            try:
                message = await asyncio.wait_for(queue.get(), timeout=1.0)
                handlers = self._subscriptions.get(topic, set())

                # Параллельная обработка всеми подписчиками
                tasks = [handler(message) for handler in handlers]
                await asyncio.gather(*tasks, return_exceptions=True)

            except asyncio.TimeoutError:
                continue
            except asyncio.CancelledError:
                break


# Пример с фильтрацией сообщений
class FilteredPubSubBroker(AsyncPubSubBroker):
    """Pub/Sub с поддержкой фильтров."""

    def __init__(self):
        super().__init__()
        self._filters: Dict[str, Dict[AsyncMessageHandler, Callable]] = {}

    def subscribe_with_filter(
        self,
        topic: str,
        handler: AsyncMessageHandler,
        filter_fn: Callable[[AsyncMessage], bool]
    ) -> None:
        """Подписка с фильтром сообщений."""
        self.subscribe(topic, handler)

        if topic not in self._filters:
            self._filters[topic] = {}
        self._filters[topic][handler] = filter_fn

    async def _process_topic(self, topic: str) -> None:
        """Обработка с учётом фильтров."""
        queue = self._queues[topic]

        while self._running:
            try:
                message = await asyncio.wait_for(queue.get(), timeout=1.0)
                handlers = self._subscriptions.get(topic, set())

                for handler in handlers:
                    # Проверяем фильтр
                    filter_fn = self._filters.get(topic, {}).get(handler)
                    if filter_fn and not filter_fn(message):
                        continue

                    try:
                        await handler(message)
                    except Exception as e:
                        print(f"Handler error: {e}")

            except asyncio.TimeoutError:
                continue
            except asyncio.CancelledError:
                break
```

## Паттерны маршрутизации

### Topic-based Routing

```python
class TopicRouter:
    """Маршрутизация на основе топиков."""

    def __init__(self):
        self._exact_subscriptions: Dict[str, Set[MessageHandler]] = {}
        self._wildcard_subscriptions: Dict[str, Set[MessageHandler]] = {}

    def subscribe(self, pattern: str, handler: MessageHandler) -> None:
        """
        Подписка с поддержкой wildcard:
        - orders.* — все подтопики orders
        - orders.# — все уровни вложенности под orders
        """
        if '*' in pattern or '#' in pattern:
            self._wildcard_subscriptions[pattern] = \
                self._wildcard_subscriptions.get(pattern, set())
            self._wildcard_subscriptions[pattern].add(handler)
        else:
            self._exact_subscriptions[pattern] = \
                self._exact_subscriptions.get(pattern, set())
            self._exact_subscriptions[pattern].add(handler)

    def get_handlers(self, topic: str) -> Set[MessageHandler]:
        """Получение обработчиков для топика."""
        handlers = set()

        # Точное совпадение
        handlers.update(self._exact_subscriptions.get(topic, set()))

        # Проверка wildcard паттернов
        for pattern, pattern_handlers in self._wildcard_subscriptions.items():
            if self._matches(pattern, topic):
                handlers.update(pattern_handlers)

        return handlers

    def _matches(self, pattern: str, topic: str) -> bool:
        """Проверка соответствия топика паттерну."""
        pattern_parts = pattern.split('.')
        topic_parts = topic.split('.')

        p_idx = 0
        t_idx = 0

        while p_idx < len(pattern_parts) and t_idx < len(topic_parts):
            if pattern_parts[p_idx] == '#':
                return True  # # соответствует всему остальному
            elif pattern_parts[p_idx] == '*':
                p_idx += 1
                t_idx += 1
            elif pattern_parts[p_idx] == topic_parts[t_idx]:
                p_idx += 1
                t_idx += 1
            else:
                return False

        return p_idx == len(pattern_parts) and t_idx == len(topic_parts)


# Использование
router = TopicRouter()

# Подписки
router.subscribe("orders.created", lambda m: print("Exact: orders.created"))
router.subscribe("orders.*", lambda m: print("Wildcard: orders.*"))
router.subscribe("orders.#", lambda m: print("Multi-level: orders.#"))

# Получение обработчиков
handlers = router.get_handlers("orders.created")  # Все три
handlers = router.get_handlers("orders.shipped")  # orders.* и orders.#
handlers = router.get_handlers("orders.us.shipped")  # Только orders.#
```

### Content-based Routing

```python
from typing import Callable, Dict, List

class ContentRouter:
    """Маршрутизация на основе содержимого сообщения."""

    def __init__(self):
        self._rules: List[tuple] = []  # (condition, handler)

    def add_rule(
        self,
        condition: Callable[[Message], bool],
        handler: MessageHandler
    ) -> None:
        """Добавление правила маршрутизации."""
        self._rules.append((condition, handler))

    def route(self, message: Message) -> None:
        """Маршрутизация сообщения."""
        for condition, handler in self._rules:
            if condition(message):
                try:
                    handler(message)
                except Exception as e:
                    print(f"Handler error: {e}")


# Пример использования
router = ContentRouter()

# Правила маршрутизации
router.add_rule(
    condition=lambda m: m.payload.get("priority") == "high",
    handler=lambda m: print(f"High priority: {m.payload}")
)

router.add_rule(
    condition=lambda m: m.payload.get("amount", 0) > 1000,
    handler=lambda m: print(f"Large order: {m.payload}")
)

router.add_rule(
    condition=lambda m: "user_id" in m.payload,
    handler=lambda m: print(f"User event: {m.payload}")
)
```

## Consumer Groups

```python
from typing import Dict, List, Set
import random
import asyncio

class ConsumerGroup:
    """Группа потребителей для распределённой обработки."""

    def __init__(self, group_id: str):
        self.group_id = group_id
        self.consumers: List[AsyncMessageHandler] = []
        self._current_index = 0

    def add_consumer(self, handler: AsyncMessageHandler) -> None:
        """Добавление потребителя в группу."""
        self.consumers.append(handler)

    def remove_consumer(self, handler: AsyncMessageHandler) -> None:
        """Удаление потребителя из группы."""
        self.consumers.remove(handler)

    async def deliver(self, message: AsyncMessage) -> None:
        """Доставка сообщения одному потребителю (round-robin)."""
        if not self.consumers:
            return

        handler = self.consumers[self._current_index]
        self._current_index = (self._current_index + 1) % len(self.consumers)

        await handler(message)


class GroupPubSubBroker:
    """Pub/Sub брокер с поддержкой consumer groups."""

    def __init__(self):
        self._direct_subscriptions: Dict[str, Set[AsyncMessageHandler]] = {}
        self._groups: Dict[str, Dict[str, ConsumerGroup]] = {}  # topic -> group_id -> group
        self._queues: Dict[str, asyncio.Queue] = {}
        self._running = False

    def subscribe(self, topic: str, handler: AsyncMessageHandler) -> None:
        """Прямая подписка (все получают все сообщения)."""
        if topic not in self._direct_subscriptions:
            self._direct_subscriptions[topic] = set()
            self._queues[topic] = asyncio.Queue()

        self._direct_subscriptions[topic].add(handler)

    def subscribe_group(
        self,
        topic: str,
        group_id: str,
        handler: AsyncMessageHandler
    ) -> None:
        """Подписка в составе группы (только один получает сообщение)."""
        if topic not in self._groups:
            self._groups[topic] = {}
            if topic not in self._queues:
                self._queues[topic] = asyncio.Queue()

        if group_id not in self._groups[topic]:
            self._groups[topic][group_id] = ConsumerGroup(group_id)

        self._groups[topic][group_id].add_consumer(handler)

    async def publish(self, topic: str, payload: Any) -> None:
        """Публикация сообщения."""
        message = AsyncMessage(topic=topic, payload=payload)

        if topic in self._queues:
            await self._queues[topic].put(message)

    async def start(self) -> None:
        """Запуск обработки."""
        self._running = True
        topics = set(self._direct_subscriptions.keys()) | set(self._groups.keys())

        for topic in topics:
            asyncio.create_task(self._process_topic(topic))

    async def _process_topic(self, topic: str) -> None:
        """Обработка топика."""
        queue = self._queues.get(topic)
        if not queue:
            return

        while self._running:
            try:
                message = await asyncio.wait_for(queue.get(), timeout=1.0)

                # Доставка всем прямым подписчикам
                direct_handlers = self._direct_subscriptions.get(topic, set())
                for handler in direct_handlers:
                    asyncio.create_task(handler(message))

                # Доставка группам (по одному из группы)
                groups = self._groups.get(topic, {})
                for group in groups.values():
                    asyncio.create_task(group.deliver(message))

            except asyncio.TimeoutError:
                continue
```

## Гарантии доставки

```python
from enum import Enum
from dataclasses import dataclass, field
from typing import Dict, Set, Optional
import asyncio

class DeliverySemantics(Enum):
    AT_MOST_ONCE = "at_most_once"    # Максимум один раз
    AT_LEAST_ONCE = "at_least_once"  # Минимум один раз
    EXACTLY_ONCE = "exactly_once"     # Ровно один раз


@dataclass
class AcknowledgeableMessage:
    """Сообщение с подтверждением."""
    id: str
    topic: str
    payload: Any
    delivery_count: int = 0
    acked: bool = False
    nacked: bool = False


class ReliablePubSub:
    """Pub/Sub с гарантиями доставки."""

    def __init__(
        self,
        semantics: DeliverySemantics = DeliverySemantics.AT_LEAST_ONCE,
        max_retries: int = 3,
        ack_timeout: float = 30.0
    ):
        self.semantics = semantics
        self.max_retries = max_retries
        self.ack_timeout = ack_timeout

        self._subscriptions: Dict[str, Set[AsyncMessageHandler]] = {}
        self._pending_messages: Dict[str, AcknowledgeableMessage] = {}
        self._processed_ids: Set[str] = set()  # Для exactly-once

    def subscribe(self, topic: str, handler: AsyncMessageHandler) -> None:
        if topic not in self._subscriptions:
            self._subscriptions[topic] = set()
        self._subscriptions[topic].add(handler)

    async def publish(self, topic: str, payload: Any) -> str:
        """Публикация с гарантией доставки."""
        message = AcknowledgeableMessage(
            id=str(uuid.uuid4()),
            topic=topic,
            payload=payload
        )

        if self.semantics == DeliverySemantics.AT_MOST_ONCE:
            await self._deliver_once(message)

        elif self.semantics == DeliverySemantics.AT_LEAST_ONCE:
            await self._deliver_with_retry(message)

        elif self.semantics == DeliverySemantics.EXACTLY_ONCE:
            await self._deliver_exactly_once(message)

        return message.id

    async def _deliver_once(self, message: AcknowledgeableMessage) -> None:
        """At-most-once: отправить и забыть."""
        handlers = self._subscriptions.get(message.topic, set())
        for handler in handlers:
            try:
                await handler(message)
            except Exception:
                pass

    async def _deliver_with_retry(self, message: AcknowledgeableMessage) -> None:
        """At-least-once: повторять до успеха."""
        self._pending_messages[message.id] = message

        while message.delivery_count < self.max_retries and not message.acked:
            message.delivery_count += 1

            handlers = self._subscriptions.get(message.topic, set())
            for handler in handlers:
                try:
                    await asyncio.wait_for(
                        handler(message),
                        timeout=self.ack_timeout
                    )
                    message.acked = True
                except (asyncio.TimeoutError, Exception):
                    continue

            if not message.acked:
                await asyncio.sleep(1)

        if message.acked:
            del self._pending_messages[message.id]
        else:
            await self._move_to_dlq(message)

    async def _deliver_exactly_once(self, message: AcknowledgeableMessage) -> None:
        """Exactly-once: дедупликация + at-least-once."""
        if message.id in self._processed_ids:
            return

        await self._deliver_with_retry(message)

        if message.acked:
            self._processed_ids.add(message.id)

    async def _move_to_dlq(self, message: AcknowledgeableMessage) -> None:
        """Перемещение в Dead Letter Queue."""
        print(f"Message {message.id} moved to DLQ")

    def acknowledge(self, message_id: str) -> None:
        """Подтверждение обработки."""
        if message_id in self._pending_messages:
            self._pending_messages[message_id].acked = True
```

## Интеграция с брокерами сообщений

### Redis Pub/Sub

```python
import redis
import json
from typing import Callable, Dict

class RedisPubSub:
    """Pub/Sub на базе Redis."""

    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis = redis.from_url(redis_url)
        self.pubsub = self.redis.pubsub()
        self._handlers: Dict[str, Callable] = {}
        self._running = False

    def subscribe(self, channel: str, handler: Callable) -> None:
        """Подписка на канал."""
        self._handlers[channel] = handler
        self.pubsub.subscribe(channel)

    def psubscribe(self, pattern: str, handler: Callable) -> None:
        """Подписка по паттерну."""
        self._handlers[pattern] = handler
        self.pubsub.psubscribe(pattern)

    def publish(self, channel: str, message: dict) -> None:
        """Публикация сообщения."""
        self.redis.publish(channel, json.dumps(message))

    def start_listening(self) -> None:
        """Запуск обработки сообщений."""
        self._running = True

        for message in self.pubsub.listen():
            if not self._running:
                break

            if message["type"] in ["message", "pmessage"]:
                channel = message.get("channel", b"").decode()
                pattern = message.get("pattern", b"")
                if pattern:
                    pattern = pattern.decode()

                data = json.loads(message["data"])

                handler = self._handlers.get(channel) or self._handlers.get(pattern)
                if handler:
                    try:
                        handler(data)
                    except Exception as e:
                        print(f"Handler error: {e}")

    def stop(self) -> None:
        """Остановка."""
        self._running = False
        self.pubsub.close()
```

## Паттерны использования

### Fan-out

```python
class FanOutPublisher:
    """Публикация одного сообщения множеству получателей."""

    def __init__(self, broker: PubSubBroker):
        self.broker = broker

    def broadcast(self, payload: Any, topics: List[str]) -> None:
        """Рассылка сообщения в несколько топиков."""
        for topic in topics:
            self.broker.publish(topic, payload)


# Использование
publisher = FanOutPublisher(broker)
publisher.broadcast(
    {"event": "user_registered", "user_id": "123"},
    ["notifications", "analytics", "email-service", "crm"]
)
```

### Fan-in

```python
class FanInAggregator:
    """Агрегация сообщений из нескольких источников."""

    def __init__(self, broker: PubSubBroker, output_topic: str):
        self.broker = broker
        self.output_topic = output_topic
        self.buffer: List[Message] = []

    def subscribe_to_sources(self, topics: List[str]) -> None:
        """Подписка на несколько топиков-источников."""
        for topic in topics:
            self.broker.subscribe(topic, self._handle_message)

    def _handle_message(self, message: Message) -> None:
        """Обработка входящего сообщения."""
        self.buffer.append(message)

        if len(self.buffer) >= 10:
            self._flush()

    def _flush(self) -> None:
        """Отправка агрегированных данных."""
        if not self.buffer:
            return

        aggregated = {
            "messages": [m.payload for m in self.buffer],
            "count": len(self.buffer)
        }

        self.broker.publish(self.output_topic, aggregated)
        self.buffer.clear()
```

### Request-Reply

```python
import asyncio
from typing import Dict
import uuid

class RequestReplyBroker:
    """Pub/Sub с паттерном request-reply."""

    def __init__(self, broker: AsyncPubSubBroker):
        self.broker = broker
        self._pending_requests: Dict[str, asyncio.Future] = {}

    async def request(
        self,
        topic: str,
        payload: Any,
        timeout: float = 30.0
    ) -> Any:
        """Отправка запроса и ожидание ответа."""
        correlation_id = str(uuid.uuid4())
        reply_topic = f"reply.{correlation_id}"

        future = asyncio.Future()
        self._pending_requests[correlation_id] = future

        async def reply_handler(msg):
            if msg.payload.get("correlation_id") == correlation_id:
                future.set_result(msg.payload.get("result"))

        self.broker.subscribe(reply_topic, reply_handler)

        await self.broker.publish(topic, {
            "correlation_id": correlation_id,
            "reply_to": reply_topic,
            "payload": payload
        })

        try:
            result = await asyncio.wait_for(future, timeout=timeout)
            return result
        finally:
            del self._pending_requests[correlation_id]

    async def reply(self, reply_to: str, correlation_id: str, result: Any) -> None:
        """Отправка ответа на запрос."""
        await self.broker.publish(reply_to, {
            "correlation_id": correlation_id,
            "result": result
        })
```

## Преимущества Pub/Sub

### 1. Развязка компонентов

```python
# Publisher не знает о subscribers

class OrderService:
    def __init__(self, broker: PubSubBroker):
        self.broker = broker

    def create_order(self, data: dict) -> str:
        order_id = str(uuid.uuid4())
        self.broker.publish("orders.created", {
            "order_id": order_id,
            **data
        })
        return order_id
```

### 2. Масштабируемость

```python
# Легко добавлять новых подписчиков

broker.subscribe("orders.created", email_handler)
broker.subscribe("orders.created", sms_handler)
broker.subscribe("orders.created", analytics_handler)
```

## Недостатки и решения

### 1. Порядок сообщений

```python
# Решение: партиционирование по ключу
class OrderedPubSub:
    def __init__(self, partition_count: int = 10):
        self.partitions = [asyncio.Queue() for _ in range(partition_count)]

    def publish(self, key: str, payload: Any) -> None:
        partition = hash(key) % len(self.partitions)
        self.partitions[partition].put_nowait((key, payload))
```

### 2. Потеря сообщений

```python
# Решение: персистентное хранение
class PersistentPubSub:
    def __init__(self, storage):
        self.storage = storage

    async def publish(self, topic: str, payload: Any) -> str:
        message_id = str(uuid.uuid4())
        await self.storage.save_message(message_id, topic, payload)
        await self._deliver(topic, payload)
        return message_id
```

## Когда использовать?

### Подходит для:

1. **Микросервисных архитектур** с асинхронным взаимодействием
2. **Систем уведомлений** и оповещений
3. **Real-time обновлений** (чаты, новости)
4. **Логирования и мониторинга**
5. **Event-driven архитектур**

### Не подходит для:

1. **Синхронных операций** требующих немедленного ответа
2. **Строго упорядоченных** операций
3. **Простых систем** без требований к масштабируемости

## Заключение

Паттерн Publish-Subscribe — это фундамент для построения масштабируемых, слабосвязанных систем. Он обеспечивает гибкость при добавлении новых компонентов и упрощает развитие системы. При выборе Pub/Sub важно учитывать требования к гарантиям доставки, порядку сообщений и задержкам.
