# Messaging Architecture

## Определение

**Messaging Architecture** (архитектура обмена сообщениями) — это архитектурный стиль, в котором компоненты системы взаимодействуют через асинхронный обмен сообщениями с использованием брокера сообщений (Message Broker). Этот подход обеспечивает слабую связанность компонентов, надёжную доставку сообщений и масштабируемость системы.

## Схема архитектуры

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        MESSAGE BROKER                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                         EXCHANGES                                │   │
│  │   ┌──────────┐    ┌──────────┐    ┌──────────────┐              │   │
│  │   │  Direct  │    │  Topic   │    │   Fanout     │              │   │
│  │   │ Exchange │    │ Exchange │    │   Exchange   │              │   │
│  │   └────┬─────┘    └────┬─────┘    └──────┬───────┘              │   │
│  └────────┼───────────────┼─────────────────┼──────────────────────┘   │
│           │               │                 │                           │
│  ┌────────▼───────────────▼─────────────────▼──────────────────────┐   │
│  │                         QUEUES                                   │   │
│  │   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │   │
│  │   │ Orders   │    │ Payments │    │  Emails  │    │   Logs   │  │   │
│  │   │  Queue   │    │  Queue   │    │  Queue   │    │  Queue   │  │   │
│  │   └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘  │   │
│  └────────┼───────────────┼───────────────┼───────────────┼────────┘   │
└───────────┼───────────────┼───────────────┼───────────────┼────────────┘
            │               │               │               │
    ┌───────▼───────┐ ┌─────▼─────┐ ┌───────▼───────┐ ┌─────▼─────┐
    │    Order      │ │  Payment  │ │    Email      │ │    Log    │
    │   Consumer    │ │  Consumer │ │   Consumer    │ │  Consumer │
    └───────────────┘ └───────────┘ └───────────────┘ └───────────┘
```

## Основные компоненты

### 1. Сообщение (Message)

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Any, Dict, Optional
from enum import Enum
import json
import uuid


class MessagePriority(Enum):
    LOW = 1
    NORMAL = 5
    HIGH = 10
    CRITICAL = 100


@dataclass
class MessageHeaders:
    """Заголовки сообщения с метаданными."""
    message_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    correlation_id: Optional[str] = None
    reply_to: Optional[str] = None
    timestamp: datetime = field(default_factory=datetime.utcnow)
    content_type: str = "application/json"
    priority: MessagePriority = MessagePriority.NORMAL
    expiration: Optional[int] = None  # TTL в секундах
    retry_count: int = 0
    max_retries: int = 3
    custom_headers: Dict[str, str] = field(default_factory=dict)

    def to_dict(self) -> Dict[str, Any]:
        return {
            "message_id": self.message_id,
            "correlation_id": self.correlation_id,
            "reply_to": self.reply_to,
            "timestamp": self.timestamp.isoformat(),
            "content_type": self.content_type,
            "priority": self.priority.value,
            "expiration": self.expiration,
            "retry_count": self.retry_count,
            "max_retries": self.max_retries,
            **self.custom_headers
        }


@dataclass
class Message:
    """Базовое сообщение в системе."""
    body: Dict[str, Any]
    headers: MessageHeaders = field(default_factory=MessageHeaders)
    routing_key: Optional[str] = None

    def serialize(self) -> bytes:
        """Сериализация сообщения в байты."""
        data = {
            "headers": self.headers.to_dict(),
            "body": self.body,
            "routing_key": self.routing_key
        }
        return json.dumps(data).encode("utf-8")

    @classmethod
    def deserialize(cls, data: bytes) -> "Message":
        """Десериализация сообщения из байтов."""
        parsed = json.loads(data.decode("utf-8"))
        headers_data = parsed["headers"]

        headers = MessageHeaders(
            message_id=headers_data["message_id"],
            correlation_id=headers_data.get("correlation_id"),
            reply_to=headers_data.get("reply_to"),
            timestamp=datetime.fromisoformat(headers_data["timestamp"]),
            content_type=headers_data["content_type"],
            priority=MessagePriority(headers_data["priority"]),
            expiration=headers_data.get("expiration"),
            retry_count=headers_data.get("retry_count", 0),
            max_retries=headers_data.get("max_retries", 3)
        )

        return cls(
            body=parsed["body"],
            headers=headers,
            routing_key=parsed.get("routing_key")
        )

    def can_retry(self) -> bool:
        """Проверяет, можно ли повторить обработку сообщения."""
        return self.headers.retry_count < self.headers.max_retries

    def increment_retry(self) -> "Message":
        """Создаёт копию сообщения с увеличенным счётчиком попыток."""
        new_headers = MessageHeaders(
            message_id=self.headers.message_id,
            correlation_id=self.headers.correlation_id,
            reply_to=self.headers.reply_to,
            timestamp=self.headers.timestamp,
            content_type=self.headers.content_type,
            priority=self.headers.priority,
            expiration=self.headers.expiration,
            retry_count=self.headers.retry_count + 1,
            max_retries=self.headers.max_retries,
            custom_headers=self.headers.custom_headers
        )
        return Message(body=self.body, headers=new_headers, routing_key=self.routing_key)
```

### 2. Типы Exchange

```python
from abc import ABC, abstractmethod
from typing import List, Dict, Set
import re


class Exchange(ABC):
    """Абстрактный exchange для маршрутизации сообщений."""

    def __init__(self, name: str):
        self.name = name
        self.bindings: Dict[str, Set[str]] = {}  # routing_key -> queue_names

    def bind(self, queue_name: str, routing_key: str) -> None:
        """Привязывает очередь к exchange с routing key."""
        if routing_key not in self.bindings:
            self.bindings[routing_key] = set()
        self.bindings[routing_key].add(queue_name)

    def unbind(self, queue_name: str, routing_key: str) -> None:
        """Отвязывает очередь от exchange."""
        if routing_key in self.bindings:
            self.bindings[routing_key].discard(queue_name)

    @abstractmethod
    def route(self, routing_key: str) -> List[str]:
        """Определяет очереди для маршрутизации сообщения."""
        pass


class DirectExchange(Exchange):
    """Direct exchange — точное совпадение routing key."""

    def route(self, routing_key: str) -> List[str]:
        return list(self.bindings.get(routing_key, set()))


class FanoutExchange(Exchange):
    """Fanout exchange — отправка во все привязанные очереди."""

    def route(self, routing_key: str) -> List[str]:
        all_queues: Set[str] = set()
        for queues in self.bindings.values():
            all_queues.update(queues)
        return list(all_queues)


class TopicExchange(Exchange):
    """Topic exchange — маршрутизация по шаблону."""

    def route(self, routing_key: str) -> List[str]:
        matched_queues: Set[str] = set()

        for pattern, queues in self.bindings.items():
            if self._matches_pattern(routing_key, pattern):
                matched_queues.update(queues)

        return list(matched_queues)

    def _matches_pattern(self, routing_key: str, pattern: str) -> bool:
        """
        Проверяет соответствие routing key шаблону.
        * — одно слово
        # — ноль или более слов
        """
        # Преобразуем шаблон в регулярное выражение
        regex_pattern = pattern.replace(".", r"\.")
        regex_pattern = regex_pattern.replace("*", r"[^.]+")
        regex_pattern = regex_pattern.replace("#", r".*")
        regex_pattern = f"^{regex_pattern}$"

        return bool(re.match(regex_pattern, routing_key))


class HeadersExchange(Exchange):
    """Headers exchange — маршрутизация по заголовкам."""

    def __init__(self, name: str):
        super().__init__(name)
        self.header_bindings: Dict[str, Dict[str, str]] = {}  # queue_name -> required_headers
        self.match_all: Dict[str, bool] = {}  # queue_name -> x-match (all/any)

    def bind_with_headers(
        self,
        queue_name: str,
        headers: Dict[str, str],
        match_all: bool = True
    ) -> None:
        """Привязывает очередь с требуемыми заголовками."""
        self.header_bindings[queue_name] = headers
        self.match_all[queue_name] = match_all

    def route_with_headers(self, headers: Dict[str, str]) -> List[str]:
        """Маршрутизация на основе заголовков сообщения."""
        matched_queues: List[str] = []

        for queue_name, required_headers in self.header_bindings.items():
            if self.match_all.get(queue_name, True):
                # Все заголовки должны совпадать
                if all(
                    headers.get(key) == value
                    for key, value in required_headers.items()
                ):
                    matched_queues.append(queue_name)
            else:
                # Любой заголовок должен совпадать
                if any(
                    headers.get(key) == value
                    for key, value in required_headers.items()
                ):
                    matched_queues.append(queue_name)

        return matched_queues

    def route(self, routing_key: str) -> List[str]:
        # Headers exchange игнорирует routing key
        return []
```

### 3. Очередь сообщений (Message Queue)

```python
from collections import deque
from threading import Lock
from typing import Optional, Callable
import heapq


@dataclass
class QueueConfig:
    """Конфигурация очереди."""
    name: str
    durable: bool = True  # Сохранять на диск
    exclusive: bool = False  # Только одно подключение
    auto_delete: bool = False  # Удалять при отключении всех
    max_length: Optional[int] = None  # Максимальное количество сообщений
    max_length_bytes: Optional[int] = None  # Максимальный размер в байтах
    message_ttl: Optional[int] = None  # TTL для сообщений в мс
    dead_letter_exchange: Optional[str] = None  # Exchange для недоставленных
    dead_letter_routing_key: Optional[str] = None


class MessageQueue:
    """Очередь сообщений с поддержкой приоритетов."""

    def __init__(self, config: QueueConfig):
        self.config = config
        self._messages: List[tuple] = []  # heap: (priority, timestamp, message)
        self._lock = Lock()
        self._unacked: Dict[str, Message] = {}  # message_id -> message
        self._current_size_bytes = 0

    def enqueue(self, message: Message) -> bool:
        """Добавляет сообщение в очередь."""
        with self._lock:
            # Проверяем лимиты
            if self.config.max_length and len(self._messages) >= self.config.max_length:
                return False

            message_size = len(message.serialize())
            if (self.config.max_length_bytes and
                self._current_size_bytes + message_size > self.config.max_length_bytes):
                return False

            # Добавляем в heap с приоритетом (меньший приоритет = выше в очереди)
            priority = -message.headers.priority.value  # Инвертируем для heap
            timestamp = message.headers.timestamp.timestamp()
            heapq.heappush(self._messages, (priority, timestamp, message))
            self._current_size_bytes += message_size

            return True

    def dequeue(self, auto_ack: bool = False) -> Optional[Message]:
        """Извлекает сообщение из очереди."""
        with self._lock:
            if not self._messages:
                return None

            _, _, message = heapq.heappop(self._messages)
            self._current_size_bytes -= len(message.serialize())

            if not auto_ack:
                self._unacked[message.headers.message_id] = message

            return message

    def ack(self, message_id: str) -> bool:
        """Подтверждает обработку сообщения."""
        with self._lock:
            if message_id in self._unacked:
                del self._unacked[message_id]
                return True
            return False

    def nack(self, message_id: str, requeue: bool = True) -> Optional[Message]:
        """Отклоняет сообщение с возможностью возврата в очередь."""
        with self._lock:
            if message_id not in self._unacked:
                return None

            message = self._unacked.pop(message_id)

            if requeue:
                self.enqueue(message)

            return message

    def reject(self, message_id: str) -> Optional[Message]:
        """Отклоняет сообщение без возврата в очередь."""
        return self.nack(message_id, requeue=False)

    def peek(self) -> Optional[Message]:
        """Просмотр сообщения без извлечения."""
        with self._lock:
            if self._messages:
                return self._messages[0][2]
            return None

    def __len__(self) -> int:
        return len(self._messages)

    @property
    def size_bytes(self) -> int:
        return self._current_size_bytes
```

### 4. Message Broker

```python
import asyncio
from typing import Callable, Awaitable


MessageHandler = Callable[[Message], Awaitable[None]]


class MessageBroker:
    """Центральный брокер сообщений."""

    def __init__(self):
        self.exchanges: Dict[str, Exchange] = {}
        self.queues: Dict[str, MessageQueue] = {}
        self.consumers: Dict[str, List[MessageHandler]] = {}  # queue_name -> handlers
        self._running = False

    def declare_exchange(
        self,
        name: str,
        exchange_type: str = "direct"
    ) -> Exchange:
        """Объявляет exchange."""
        if name in self.exchanges:
            return self.exchanges[name]

        exchange: Exchange
        if exchange_type == "direct":
            exchange = DirectExchange(name)
        elif exchange_type == "fanout":
            exchange = FanoutExchange(name)
        elif exchange_type == "topic":
            exchange = TopicExchange(name)
        elif exchange_type == "headers":
            exchange = HeadersExchange(name)
        else:
            raise ValueError(f"Unknown exchange type: {exchange_type}")

        self.exchanges[name] = exchange
        return exchange

    def declare_queue(self, config: QueueConfig) -> MessageQueue:
        """Объявляет очередь."""
        if config.name in self.queues:
            return self.queues[config.name]

        queue = MessageQueue(config)
        self.queues[config.name] = queue
        return queue

    def bind_queue(
        self,
        queue_name: str,
        exchange_name: str,
        routing_key: str
    ) -> None:
        """Привязывает очередь к exchange."""
        if exchange_name not in self.exchanges:
            raise ValueError(f"Exchange {exchange_name} not found")
        if queue_name not in self.queues:
            raise ValueError(f"Queue {queue_name} not found")

        self.exchanges[exchange_name].bind(queue_name, routing_key)

    async def publish(
        self,
        exchange_name: str,
        message: Message,
        routing_key: Optional[str] = None
    ) -> int:
        """Публикует сообщение через exchange."""
        if exchange_name not in self.exchanges:
            raise ValueError(f"Exchange {exchange_name} not found")

        exchange = self.exchanges[exchange_name]
        key = routing_key or message.routing_key or ""

        target_queues = exchange.route(key)
        delivered = 0

        for queue_name in target_queues:
            if queue_name in self.queues:
                if self.queues[queue_name].enqueue(message):
                    delivered += 1

        return delivered

    def subscribe(
        self,
        queue_name: str,
        handler: MessageHandler
    ) -> None:
        """Подписывает обработчик на очередь."""
        if queue_name not in self.consumers:
            self.consumers[queue_name] = []
        self.consumers[queue_name].append(handler)

    async def start_consuming(self) -> None:
        """Запускает обработку сообщений."""
        self._running = True

        async def consume_queue(queue_name: str) -> None:
            queue = self.queues[queue_name]
            handlers = self.consumers.get(queue_name, [])

            while self._running:
                message = queue.dequeue()
                if message is None:
                    await asyncio.sleep(0.01)
                    continue

                for handler in handlers:
                    try:
                        await handler(message)
                        queue.ack(message.headers.message_id)
                    except Exception as e:
                        print(f"Error processing message: {e}")
                        if message.can_retry():
                            queue.nack(message.headers.message_id, requeue=True)
                        else:
                            queue.reject(message.headers.message_id)

        tasks = [
            asyncio.create_task(consume_queue(queue_name))
            for queue_name in self.consumers.keys()
        ]

        await asyncio.gather(*tasks)

    def stop_consuming(self) -> None:
        """Останавливает обработку сообщений."""
        self._running = False
```

## Паттерны обмена сообщениями

### 1. Request-Reply Pattern

```
┌──────────┐         ┌──────────────┐         ┌──────────┐
│  Client  │────────▶│    Broker    │────────▶│  Server  │
│          │         │              │         │          │
│          │◀────────│              │◀────────│          │
└──────────┘         └──────────────┘         └──────────┘
     │                      │                      │
     │  1. Send Request     │                      │
     │  (reply_to=temp_q)   │                      │
     │─────────────────────▶│  2. Route to server  │
     │                      │─────────────────────▶│
     │                      │                      │ 3. Process
     │                      │  4. Send Response    │
     │                      │◀─────────────────────│
     │  5. Receive Response │                      │
     │◀─────────────────────│                      │
```

```python
import asyncio
from typing import TypeVar, Generic

T = TypeVar("T")


class AsyncReplyFuture(Generic[T]):
    """Future для ожидания ответа на запрос."""

    def __init__(self, correlation_id: str, timeout: float = 30.0):
        self.correlation_id = correlation_id
        self.timeout = timeout
        self._event = asyncio.Event()
        self._result: Optional[T] = None
        self._error: Optional[Exception] = None

    def set_result(self, result: T) -> None:
        self._result = result
        self._event.set()

    def set_error(self, error: Exception) -> None:
        self._error = error
        self._event.set()

    async def get_result(self) -> T:
        try:
            await asyncio.wait_for(self._event.wait(), timeout=self.timeout)
        except asyncio.TimeoutError:
            raise TimeoutError(f"Request {self.correlation_id} timed out")

        if self._error:
            raise self._error

        return self._result  # type: ignore


class RequestReplyClient:
    """Клиент для паттерна Request-Reply."""

    def __init__(self, broker: MessageBroker, reply_queue: str):
        self.broker = broker
        self.reply_queue = reply_queue
        self.pending_requests: Dict[str, AsyncReplyFuture] = {}

        # Подписываемся на очередь ответов
        broker.subscribe(reply_queue, self._handle_reply)

    async def _handle_reply(self, message: Message) -> None:
        """Обрабатывает входящий ответ."""
        correlation_id = message.headers.correlation_id
        if correlation_id and correlation_id in self.pending_requests:
            future = self.pending_requests.pop(correlation_id)

            if "error" in message.body:
                future.set_error(Exception(message.body["error"]))
            else:
                future.set_result(message.body)

    async def request(
        self,
        exchange: str,
        routing_key: str,
        body: Dict[str, Any],
        timeout: float = 30.0
    ) -> Dict[str, Any]:
        """Отправляет запрос и ожидает ответ."""
        correlation_id = str(uuid.uuid4())

        headers = MessageHeaders(
            correlation_id=correlation_id,
            reply_to=self.reply_queue
        )

        message = Message(body=body, headers=headers, routing_key=routing_key)

        future: AsyncReplyFuture[Dict[str, Any]] = AsyncReplyFuture(
            correlation_id,
            timeout
        )
        self.pending_requests[correlation_id] = future

        await self.broker.publish(exchange, message, routing_key)

        return await future.get_result()


class RequestReplyServer:
    """Сервер для паттерна Request-Reply."""

    def __init__(self, broker: MessageBroker):
        self.broker = broker
        self.handlers: Dict[str, MessageHandler] = {}

    def register_handler(
        self,
        request_type: str,
        handler: Callable[[Dict[str, Any]], Awaitable[Dict[str, Any]]]
    ) -> None:
        """Регистрирует обработчик для типа запроса."""
        async def wrapped_handler(message: Message) -> None:
            reply_to = message.headers.reply_to
            correlation_id = message.headers.correlation_id

            if not reply_to:
                return

            try:
                result = await handler(message.body)
                response_body = result
            except Exception as e:
                response_body = {"error": str(e)}

            response_headers = MessageHeaders(correlation_id=correlation_id)
            response = Message(body=response_body, headers=response_headers)

            # Отправляем ответ напрямую в очередь
            if reply_to in self.broker.queues:
                self.broker.queues[reply_to].enqueue(response)

        self.handlers[request_type] = wrapped_handler

    def subscribe_to_queue(self, queue_name: str) -> None:
        """Подписывается на очередь запросов."""
        async def dispatch(message: Message) -> None:
            request_type = message.body.get("type", "default")
            handler = self.handlers.get(request_type)

            if handler:
                await handler(message)

        self.broker.subscribe(queue_name, dispatch)
```

### 2. Competing Consumers Pattern

```
                         ┌─────────────┐
                    ┌───▶│  Consumer 1 │
                    │    └─────────────┘
┌──────────┐    ┌───┴───┐
│ Producer │───▶│ Queue │───┐
└──────────┘    └───────┘   │ ┌─────────────┐
                    │       ├▶│  Consumer 2 │
                    │       │ └─────────────┘
                    │       │ ┌─────────────┐
                    └───────┴▶│  Consumer 3 │
                              └─────────────┘
```

```python
from dataclasses import dataclass
from typing import List
import random


@dataclass
class ConsumerInfo:
    """Информация о потребителе."""
    consumer_id: str
    queue_name: str
    prefetch_count: int = 1
    is_active: bool = True
    messages_processed: int = 0


class CompetingConsumersManager:
    """Менеджер конкурирующих потребителей."""

    def __init__(self, broker: MessageBroker):
        self.broker = broker
        self.consumers: Dict[str, ConsumerInfo] = {}
        self.queue_consumers: Dict[str, List[str]] = {}  # queue -> consumer_ids

    def register_consumer(
        self,
        consumer_id: str,
        queue_name: str,
        prefetch_count: int = 1
    ) -> ConsumerInfo:
        """Регистрирует нового потребителя."""
        consumer = ConsumerInfo(
            consumer_id=consumer_id,
            queue_name=queue_name,
            prefetch_count=prefetch_count
        )

        self.consumers[consumer_id] = consumer

        if queue_name not in self.queue_consumers:
            self.queue_consumers[queue_name] = []
        self.queue_consumers[queue_name].append(consumer_id)

        return consumer

    def unregister_consumer(self, consumer_id: str) -> None:
        """Отменяет регистрацию потребителя."""
        if consumer_id in self.consumers:
            consumer = self.consumers.pop(consumer_id)
            if consumer.queue_name in self.queue_consumers:
                self.queue_consumers[consumer.queue_name].remove(consumer_id)

    def get_next_consumer(self, queue_name: str) -> Optional[str]:
        """Выбирает следующего потребителя (round-robin)."""
        if queue_name not in self.queue_consumers:
            return None

        active_consumers = [
            cid for cid in self.queue_consumers[queue_name]
            if self.consumers[cid].is_active
        ]

        if not active_consumers:
            return None

        # Round-robin выбор
        min_processed = min(
            self.consumers[cid].messages_processed
            for cid in active_consumers
        )

        candidates = [
            cid for cid in active_consumers
            if self.consumers[cid].messages_processed == min_processed
        ]

        return random.choice(candidates)

    def mark_processed(self, consumer_id: str) -> None:
        """Отмечает обработку сообщения потребителем."""
        if consumer_id in self.consumers:
            self.consumers[consumer_id].messages_processed += 1
```

### 3. Dead Letter Queue Pattern

```
┌──────────┐    ┌─────────────┐    ┌──────────────┐
│ Producer │───▶│ Main Queue  │───▶│   Consumer   │
└──────────┘    └──────┬──────┘    └──────────────┘
                       │
                       │ Rejected / Expired / Max retries
                       ▼
               ┌───────────────┐    ┌────────────────┐
               │   DLX         │───▶│  Dead Letter   │
               │  Exchange     │    │    Queue       │
               └───────────────┘    └───────┬────────┘
                                            │
                                            ▼
                                    ┌───────────────┐
                                    │  DLQ Handler  │
                                    │  (Alert/Log)  │
                                    └───────────────┘
```

```python
@dataclass
class DeadLetterConfig:
    """Конфигурация Dead Letter Queue."""
    dlx_exchange: str
    dlq_queue: str
    routing_key: str = "dead-letter"
    max_retries: int = 3
    retry_delay_ms: int = 5000


class DeadLetterHandler:
    """Обработчик недоставленных сообщений."""

    def __init__(self, broker: MessageBroker, config: DeadLetterConfig):
        self.broker = broker
        self.config = config
        self._setup_dlq()

    def _setup_dlq(self) -> None:
        """Настраивает DLX и DLQ."""
        # Создаём DLX
        self.broker.declare_exchange(self.config.dlx_exchange, "direct")

        # Создаём DLQ
        dlq_config = QueueConfig(
            name=self.config.dlq_queue,
            durable=True
        )
        self.broker.declare_queue(dlq_config)

        # Привязываем DLQ к DLX
        self.broker.bind_queue(
            self.config.dlq_queue,
            self.config.dlx_exchange,
            self.config.routing_key
        )

        # Подписываемся на DLQ
        self.broker.subscribe(self.config.dlq_queue, self._handle_dead_letter)

    async def send_to_dlq(
        self,
        message: Message,
        reason: str,
        original_queue: str
    ) -> None:
        """Отправляет сообщение в DLQ."""
        # Добавляем информацию о причине
        dlq_body = {
            "original_message": message.body,
            "death_reason": reason,
            "original_queue": original_queue,
            "original_routing_key": message.routing_key,
            "retry_count": message.headers.retry_count,
            "death_time": datetime.utcnow().isoformat()
        }

        dlq_headers = MessageHeaders(
            correlation_id=message.headers.message_id,
            priority=MessagePriority.HIGH
        )

        dlq_message = Message(
            body=dlq_body,
            headers=dlq_headers,
            routing_key=self.config.routing_key
        )

        await self.broker.publish(self.config.dlx_exchange, dlq_message)

    async def _handle_dead_letter(self, message: Message) -> None:
        """Обрабатывает сообщение из DLQ."""
        print(f"Dead letter received: {message.body}")

        # Здесь можно добавить логику:
        # - Отправка алерта
        # - Логирование
        # - Попытка исправить и переотправить
        # - Сохранение для ручной обработки
```

### 4. Message Deduplication

```python
from typing import Set
from datetime import datetime, timedelta


class MessageDeduplicator:
    """Дедупликатор сообщений."""

    def __init__(self, ttl_seconds: int = 3600):
        self.ttl_seconds = ttl_seconds
        self._seen_messages: Dict[str, datetime] = {}
        self._lock = Lock()

    def is_duplicate(self, message_id: str) -> bool:
        """Проверяет, является ли сообщение дубликатом."""
        with self._lock:
            self._cleanup_expired()

            if message_id in self._seen_messages:
                return True

            self._seen_messages[message_id] = datetime.utcnow()
            return False

    def _cleanup_expired(self) -> None:
        """Удаляет устаревшие записи."""
        cutoff = datetime.utcnow() - timedelta(seconds=self.ttl_seconds)
        expired = [
            msg_id for msg_id, timestamp in self._seen_messages.items()
            if timestamp < cutoff
        ]
        for msg_id in expired:
            del self._seen_messages[msg_id]

    def mark_seen(self, message_id: str) -> None:
        """Помечает сообщение как обработанное."""
        with self._lock:
            self._seen_messages[message_id] = datetime.utcnow()

    def remove(self, message_id: str) -> None:
        """Удаляет сообщение из списка обработанных."""
        with self._lock:
            self._seen_messages.pop(message_id, None)


class IdempotentConsumer:
    """Потребитель с идемпотентной обработкой."""

    def __init__(
        self,
        broker: MessageBroker,
        deduplicator: MessageDeduplicator,
        handler: MessageHandler
    ):
        self.broker = broker
        self.deduplicator = deduplicator
        self.handler = handler

    async def process(self, message: Message) -> None:
        """Обрабатывает сообщение с дедупликацией."""
        message_id = message.headers.message_id

        if self.deduplicator.is_duplicate(message_id):
            print(f"Duplicate message detected: {message_id}")
            return

        try:
            await self.handler(message)
            self.deduplicator.mark_seen(message_id)
        except Exception as e:
            self.deduplicator.remove(message_id)
            raise
```

## Полный пример системы

```python
import asyncio
from dataclasses import dataclass


@dataclass
class OrderCreatedEvent:
    order_id: str
    customer_id: str
    items: List[Dict[str, Any]]
    total_amount: float


async def main():
    # Создаём брокер
    broker = MessageBroker()

    # Настраиваем exchanges
    orders_exchange = broker.declare_exchange("orders", "topic")
    notifications_exchange = broker.declare_exchange("notifications", "fanout")

    # Настраиваем очереди
    order_processing_queue = broker.declare_queue(QueueConfig(
        name="order_processing",
        durable=True,
        dead_letter_exchange="dlx.orders",
        dead_letter_routing_key="dead.orders"
    ))

    payment_queue = broker.declare_queue(QueueConfig(
        name="payments",
        durable=True,
        max_length=10000
    ))

    email_queue = broker.declare_queue(QueueConfig(
        name="emails",
        durable=True
    ))

    sms_queue = broker.declare_queue(QueueConfig(
        name="sms",
        durable=True
    ))

    # Привязываем очереди к exchanges
    broker.bind_queue("order_processing", "orders", "order.created")
    broker.bind_queue("order_processing", "orders", "order.updated")
    broker.bind_queue("payments", "orders", "order.created")
    broker.bind_queue("emails", "notifications", "")
    broker.bind_queue("sms", "notifications", "")

    # Настраиваем Dead Letter Queue
    dlq_config = DeadLetterConfig(
        dlx_exchange="dlx.orders",
        dlq_queue="orders.dead-letter"
    )
    dlq_handler = DeadLetterHandler(broker, dlq_config)

    # Дедупликатор
    deduplicator = MessageDeduplicator(ttl_seconds=3600)

    # Обработчики
    async def process_order(message: Message) -> None:
        print(f"Processing order: {message.body}")
        # Бизнес-логика обработки заказа

    async def process_payment(message: Message) -> None:
        print(f"Processing payment for order: {message.body.get('order_id')}")
        # Логика оплаты

    async def send_email(message: Message) -> None:
        print(f"Sending email notification: {message.body}")

    async def send_sms(message: Message) -> None:
        print(f"Sending SMS notification: {message.body}")

    # Создаём идемпотентных потребителей
    order_consumer = IdempotentConsumer(broker, deduplicator, process_order)
    payment_consumer = IdempotentConsumer(broker, deduplicator, process_payment)

    # Подписываемся на очереди
    broker.subscribe("order_processing", order_consumer.process)
    broker.subscribe("payments", payment_consumer.process)
    broker.subscribe("emails", send_email)
    broker.subscribe("sms", send_sms)

    # Публикуем событие создания заказа
    order_event = Message(
        body={
            "order_id": "ORD-001",
            "customer_id": "CUST-123",
            "items": [
                {"product": "Laptop", "quantity": 1, "price": 1500.00}
            ],
            "total_amount": 1500.00
        },
        routing_key="order.created"
    )

    delivered = await broker.publish("orders", order_event)
    print(f"Order event delivered to {delivered} queues")

    # Публикуем уведомление
    notification = Message(
        body={
            "type": "order_confirmation",
            "order_id": "ORD-001",
            "message": "Your order has been confirmed!"
        }
    )

    delivered = await broker.publish("notifications", notification)
    print(f"Notification delivered to {delivered} queues")

    # Запускаем обработку (в реальном приложении — в фоне)
    # await broker.start_consuming()


if __name__ == "__main__":
    asyncio.run(main())
```

## Гарантии доставки

| Гарантия | Описание | Реализация |
|----------|----------|------------|
| **At-Most-Once** | Максимум один раз | Нет подтверждений, может потерять |
| **At-Least-Once** | Минимум один раз | ACK после обработки, могут быть дубликаты |
| **Exactly-Once** | Ровно один раз | Транзакции + дедупликация |

## Преимущества

1. **Слабая связанность** — компоненты не знают друг о друге
2. **Надёжность** — персистентность сообщений и повторная доставка
3. **Масштабируемость** — горизонтальное масштабирование потребителей
4. **Асинхронность** — отправитель не ждёт обработки
5. **Буферизация** — сглаживание пиков нагрузки
6. **Отказоустойчивость** — продолжение работы при сбоях компонентов

## Недостатки

1. **Сложность** — дополнительная инфраструктура (брокер)
2. **Eventual Consistency** — данные синхронизируются не мгновенно
3. **Сложность отладки** — асинхронные потоки труднее отслеживать
4. **Дополнительная задержка** — overhead на маршрутизацию
5. **Порядок сообщений** — не всегда гарантирован

## Когда использовать

**Подходит для:**
- Микросервисной архитектуры
- Интеграции разнородных систем
- Обработки событий и уведомлений
- Задач с неравномерной нагрузкой
- Систем с высокими требованиями к отказоустойчивости

**Не подходит для:**
- Синхронных операций с немедленным ответом
- Простых монолитных приложений
- Систем с жёсткими требованиями к задержке
- Сценариев, требующих строгой консистентности

## Популярные Message Brokers

- **RabbitMQ** — классический AMQP брокер с гибкой маршрутизацией
- **Apache Kafka** — распределённый лог для streaming данных
- **Amazon SQS** — управляемый сервис очередей AWS
- **Redis Streams** — легковесные очереди на базе Redis
- **NATS** — высокопроизводительный облачный messaging

## Лучшие практики

1. **Используйте идемпотентные обработчики** — сообщения могут приходить повторно
2. **Настраивайте Dead Letter Queue** — для обработки проблемных сообщений
3. **Мониторьте глубину очередей** — для предотвращения переполнения
4. **Устанавливайте TTL для сообщений** — избегайте накопления устаревших
5. **Используйте correlation ID** — для отслеживания запросов
6. **Версионируйте схему сообщений** — для совместимости при обновлениях
7. **Тестируйте сценарии сбоев** — retry, timeout, dead letter
