# Message Queues and Streams (Очереди сообщений и потоки)

## Что такое Message Queues и Streams?

Message Queues (очереди сообщений) и Streams (потоки) — это архитектурные паттерны для асинхронной коммуникации между компонентами системы. Они позволяют развязать производителей и потребителей сообщений, обеспечивая масштабируемость и отказоустойчивость.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Message Queue Architecture                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────┐      ┌─────────────────┐      ┌──────────┐       │
│   │Producer 1│─────►│                 │─────►│Consumer 1│       │
│   └──────────┘      │                 │      └──────────┘       │
│                     │  Message Queue  │                         │
│   ┌──────────┐      │   ┌─┬─┬─┬─┬─┐   │      ┌──────────┐       │
│   │Producer 2│─────►│   │ │ │ │ │ │   │─────►│Consumer 2│       │
│   └──────────┘      │   └─┴─┴─┴─┴─┘   │      └──────────┘       │
│                     │                 │                         │
│   ┌──────────┐      │   Messages      │      ┌──────────┐       │
│   │Producer 3│─────►│                 │─────►│Consumer 3│       │
│   └──────────┘      └─────────────────┘      └──────────┘       │
│                                                                  │
│   • Сообщения обрабатываются одним потребителем                 │
│   • Гарантия доставки                                           │
│   • FIFO порядок (обычно)                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      Stream Architecture                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────┐      ┌─────────────────┐      ┌──────────┐       │
│   │Producer 1│─────►│   Event Stream  │◄────►│Consumer 1│       │
│   └──────────┘      │                 │      └──────────┘       │
│                     │   ┌─┬─┬─┬─┬─┬─┐ │                         │
│   ┌──────────┐      │   │1│2│3│4│5│6│ │      ┌──────────┐       │
│   │Producer 2│─────►│   └─┴─┴─┴─┴─┴─┘ │◄────►│Consumer 2│       │
│   └──────────┘      │                 │      └──────────┘       │
│                     │   Ordered Log   │                         │
│                     └─────────────────┘      ┌──────────┐       │
│                              ▲               │Consumer 3│       │
│                              └───────────────┴──────────┘       │
│                                                                  │
│   • События сохраняются в упорядоченном логе                    │
│   • Множественные потребители                                   │
│   • Replay возможен                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Message Queue (Очередь сообщений)

### Основные концепции

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional, List, Callable, Any
from enum import Enum
import json
import uuid
import threading
import time


class MessageStatus(Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"
    DEAD_LETTER = "dead_letter"


@dataclass
class Message:
    """Сообщение в очереди"""
    id: str = field(default_factory=lambda: str(uuid.uuid4()))
    body: Any = None
    headers: dict = field(default_factory=dict)
    created_at: datetime = field(default_factory=datetime.now)
    attempts: int = 0
    max_attempts: int = 3
    status: MessageStatus = MessageStatus.PENDING

    def to_json(self) -> str:
        return json.dumps({
            "id": self.id,
            "body": self.body,
            "headers": self.headers,
            "created_at": self.created_at.isoformat(),
            "attempts": self.attempts
        })

    @classmethod
    def from_json(cls, data: str) -> 'Message':
        parsed = json.loads(data)
        return cls(
            id=parsed["id"],
            body=parsed["body"],
            headers=parsed["headers"],
            created_at=datetime.fromisoformat(parsed["created_at"]),
            attempts=parsed["attempts"]
        )


class MessageQueue(ABC):
    """Абстрактный интерфейс очереди сообщений"""

    @abstractmethod
    def send(self, message: Message) -> None:
        """Отправить сообщение"""
        pass

    @abstractmethod
    def receive(self, timeout: float = None) -> Optional[Message]:
        """Получить сообщение"""
        pass

    @abstractmethod
    def ack(self, message_id: str) -> None:
        """Подтвердить обработку"""
        pass

    @abstractmethod
    def nack(self, message_id: str, requeue: bool = True) -> None:
        """Отклонить сообщение"""
        pass


class InMemoryQueue(MessageQueue):
    """In-memory реализация очереди"""

    def __init__(self):
        self._queue: List[Message] = []
        self._processing: dict[str, Message] = {}
        self._dead_letter: List[Message] = []
        self._lock = threading.Lock()
        self._not_empty = threading.Condition(self._lock)

    def send(self, message: Message) -> None:
        with self._lock:
            self._queue.append(message)
            self._not_empty.notify()

    def receive(self, timeout: float = None) -> Optional[Message]:
        with self._not_empty:
            while not self._queue:
                if not self._not_empty.wait(timeout):
                    return None

            message = self._queue.pop(0)
            message.status = MessageStatus.PROCESSING
            message.attempts += 1
            self._processing[message.id] = message
            return message

    def ack(self, message_id: str) -> None:
        with self._lock:
            if message_id in self._processing:
                message = self._processing.pop(message_id)
                message.status = MessageStatus.COMPLETED

    def nack(self, message_id: str, requeue: bool = True) -> None:
        with self._lock:
            if message_id not in self._processing:
                return

            message = self._processing.pop(message_id)

            if requeue and message.attempts < message.max_attempts:
                message.status = MessageStatus.PENDING
                self._queue.append(message)
            else:
                message.status = MessageStatus.DEAD_LETTER
                self._dead_letter.append(message)
```

### Producer и Consumer

```python
class Producer:
    """Производитель сообщений"""

    def __init__(self, queue: MessageQueue):
        self._queue = queue

    def send(self, body: Any, headers: dict = None) -> str:
        """Отправить сообщение"""
        message = Message(
            body=body,
            headers=headers or {}
        )
        self._queue.send(message)
        return message.id

    def send_batch(self, messages: List[Any]) -> List[str]:
        """Отправить пакет сообщений"""
        ids = []
        for body in messages:
            message = Message(body=body)
            self._queue.send(message)
            ids.append(message.id)
        return ids


class Consumer:
    """Потребитель сообщений"""

    def __init__(self, queue: MessageQueue):
        self._queue = queue
        self._handlers: dict[str, Callable] = {}
        self._running = False

    def register_handler(self, message_type: str, handler: Callable) -> None:
        """Зарегистрировать обработчик"""
        self._handlers[message_type] = handler

    def start(self) -> None:
        """Запустить потребление сообщений"""
        self._running = True

        while self._running:
            message = self._queue.receive(timeout=1.0)

            if message is None:
                continue

            try:
                self._process_message(message)
                self._queue.ack(message.id)
            except Exception as e:
                print(f"Error processing message {message.id}: {e}")
                self._queue.nack(message.id)

    def stop(self) -> None:
        """Остановить потребление"""
        self._running = False

    def _process_message(self, message: Message) -> None:
        """Обработать сообщение"""
        message_type = message.headers.get("type", "default")

        if message_type in self._handlers:
            self._handlers[message_type](message.body)
        else:
            print(f"No handler for message type: {message_type}")


# Пример использования
queue = InMemoryQueue()

# Producer
producer = Producer(queue)
producer.send(
    body={"order_id": "123", "amount": 99.99},
    headers={"type": "order.created"}
)

# Consumer
consumer = Consumer(queue)

@consumer.register_handler("order.created")
def handle_order_created(body):
    print(f"Processing order: {body['order_id']}")
    # Бизнес-логика обработки заказа

# В отдельном потоке
import threading
consumer_thread = threading.Thread(target=consumer.start)
consumer_thread.start()
```

### Работа с RabbitMQ

```python
import pika
import json
from typing import Callable


class RabbitMQProducer:
    """Producer для RabbitMQ"""

    def __init__(self, host: str = 'localhost', queue_name: str = 'default'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()
        self.queue_name = queue_name

        # Объявляем очередь
        self.channel.queue_declare(queue=queue_name, durable=True)

    def send(self, message: dict, routing_key: str = None) -> None:
        """Отправить сообщение"""
        self.channel.basic_publish(
            exchange='',
            routing_key=routing_key or self.queue_name,
            body=json.dumps(message),
            properties=pika.BasicProperties(
                delivery_mode=2,  # Персистентное сообщение
                content_type='application/json'
            )
        )

    def close(self) -> None:
        self.connection.close()


class RabbitMQConsumer:
    """Consumer для RabbitMQ"""

    def __init__(self, host: str = 'localhost', queue_name: str = 'default'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()
        self.queue_name = queue_name

        self.channel.queue_declare(queue=queue_name, durable=True)
        # Prefetch — получаем по одному сообщению
        self.channel.basic_qos(prefetch_count=1)

    def consume(self, handler: Callable[[dict], None]) -> None:
        """Начать потребление"""

        def callback(ch, method, properties, body):
            try:
                message = json.loads(body)
                handler(message)
                ch.basic_ack(delivery_tag=method.delivery_tag)
            except Exception as e:
                print(f"Error: {e}")
                ch.basic_nack(
                    delivery_tag=method.delivery_tag,
                    requeue=True
                )

        self.channel.basic_consume(
            queue=self.queue_name,
            on_message_callback=callback
        )

        print(f"Consuming from {self.queue_name}")
        self.channel.start_consuming()

    def close(self) -> None:
        self.connection.close()


# Пример с RabbitMQ
producer = RabbitMQProducer(queue_name='orders')
producer.send({
    "type": "order.created",
    "data": {"order_id": "123", "total": 99.99}
})

consumer = RabbitMQConsumer(queue_name='orders')

def process_order(message):
    print(f"Processing: {message}")

consumer.consume(process_order)
```

## Event Streams (Потоки событий)

### Apache Kafka пример

```python
from kafka import KafkaProducer, KafkaConsumer
from kafka.admin import KafkaAdminClient, NewTopic
import json
from typing import Callable, List
from dataclasses import dataclass
from datetime import datetime


@dataclass
class Event:
    """Событие в потоке"""
    event_type: str
    data: dict
    timestamp: datetime = field(default_factory=datetime.now)
    key: str = None

    def to_json(self) -> bytes:
        return json.dumps({
            "event_type": self.event_type,
            "data": self.data,
            "timestamp": self.timestamp.isoformat()
        }).encode('utf-8')

    @classmethod
    def from_json(cls, data: bytes) -> 'Event':
        parsed = json.loads(data.decode('utf-8'))
        return cls(
            event_type=parsed["event_type"],
            data=parsed["data"],
            timestamp=datetime.fromisoformat(parsed["timestamp"])
        )


class KafkaEventProducer:
    """Продюсер событий для Kafka"""

    def __init__(self, bootstrap_servers: List[str]):
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            key_serializer=lambda k: k.encode('utf-8') if k else None
        )

    def publish(self, topic: str, event: Event) -> None:
        """Опубликовать событие"""
        self.producer.send(
            topic=topic,
            key=event.key,
            value={
                "event_type": event.event_type,
                "data": event.data,
                "timestamp": event.timestamp.isoformat()
            }
        )
        self.producer.flush()

    def publish_batch(self, topic: str, events: List[Event]) -> None:
        """Опубликовать пакет событий"""
        for event in events:
            self.producer.send(
                topic=topic,
                key=event.key,
                value={
                    "event_type": event.event_type,
                    "data": event.data,
                    "timestamp": event.timestamp.isoformat()
                }
            )
        self.producer.flush()

    def close(self) -> None:
        self.producer.close()


class KafkaEventConsumer:
    """Консьюмер событий для Kafka"""

    def __init__(self, bootstrap_servers: List[str],
                 group_id: str, topics: List[str]):
        self.consumer = KafkaConsumer(
            *topics,
            bootstrap_servers=bootstrap_servers,
            group_id=group_id,
            value_deserializer=lambda v: json.loads(v.decode('utf-8')),
            auto_offset_reset='earliest',
            enable_auto_commit=False
        )
        self._handlers: dict[str, Callable] = {}

    def register_handler(self, event_type: str, handler: Callable) -> None:
        """Зарегистрировать обработчик события"""
        self._handlers[event_type] = handler

    def start(self) -> None:
        """Начать потребление"""
        for message in self.consumer:
            try:
                event_data = message.value
                event_type = event_data.get("event_type")

                if event_type in self._handlers:
                    self._handlers[event_type](event_data["data"])

                # Коммитим offset после успешной обработки
                self.consumer.commit()

            except Exception as e:
                print(f"Error processing event: {e}")

    def seek_to_beginning(self) -> None:
        """Перемотать на начало (replay)"""
        self.consumer.poll(0)  # Trigger partition assignment
        self.consumer.seek_to_beginning()

    def close(self) -> None:
        self.consumer.close()


# Пример: Event-driven система заказов
class OrderEventService:
    """Сервис событий заказов"""

    def __init__(self, producer: KafkaEventProducer):
        self.producer = producer
        self.topic = "orders"

    def order_created(self, order_id: str, customer_id: str, total: float):
        """Событие: заказ создан"""
        event = Event(
            event_type="order.created",
            data={
                "order_id": order_id,
                "customer_id": customer_id,
                "total": total
            },
            key=order_id  # Партиционирование по order_id
        )
        self.producer.publish(self.topic, event)

    def order_confirmed(self, order_id: str):
        """Событие: заказ подтверждён"""
        event = Event(
            event_type="order.confirmed",
            data={"order_id": order_id},
            key=order_id
        )
        self.producer.publish(self.topic, event)

    def order_shipped(self, order_id: str, tracking_number: str):
        """Событие: заказ отправлен"""
        event = Event(
            event_type="order.shipped",
            data={
                "order_id": order_id,
                "tracking_number": tracking_number
            },
            key=order_id
        )
        self.producer.publish(self.topic, event)


class InventoryEventHandler:
    """Обработчик событий для инвентаря"""

    def __init__(self, consumer: KafkaEventConsumer):
        self.consumer = consumer
        self._setup_handlers()

    def _setup_handlers(self):
        self.consumer.register_handler("order.created", self._on_order_created)
        self.consumer.register_handler("order.cancelled", self._on_order_cancelled)

    def _on_order_created(self, data: dict):
        """Резервирование товаров"""
        order_id = data["order_id"]
        print(f"Reserving inventory for order {order_id}")
        # Логика резервирования

    def _on_order_cancelled(self, data: dict):
        """Освобождение резерва"""
        order_id = data["order_id"]
        print(f"Releasing inventory for order {order_id}")
        # Логика освобождения

    def start(self):
        self.consumer.start()
```

## Паттерны обмена сообщениями

### Pub/Sub (Публикация/Подписка)

```python
class PubSubBroker:
    """Простой Pub/Sub брокер"""

    def __init__(self):
        self._topics: dict[str, List[Callable]] = {}
        self._lock = threading.Lock()

    def subscribe(self, topic: str, handler: Callable) -> None:
        """Подписаться на топик"""
        with self._lock:
            if topic not in self._topics:
                self._topics[topic] = []
            self._topics[topic].append(handler)

    def unsubscribe(self, topic: str, handler: Callable) -> None:
        """Отписаться от топика"""
        with self._lock:
            if topic in self._topics:
                self._topics[topic].remove(handler)

    def publish(self, topic: str, message: Any) -> None:
        """Опубликовать сообщение"""
        with self._lock:
            handlers = self._topics.get(topic, []).copy()

        for handler in handlers:
            try:
                handler(message)
            except Exception as e:
                print(f"Handler error: {e}")


# Пример использования
broker = PubSubBroker()

# Подписчики
def email_handler(order):
    print(f"Sending email for order {order['id']}")

def analytics_handler(order):
    print(f"Tracking order {order['id']} in analytics")

broker.subscribe("order.created", email_handler)
broker.subscribe("order.created", analytics_handler)

# Публикация
broker.publish("order.created", {"id": "123", "total": 99.99})
```

### Request/Reply

```python
class RequestReplyQueue:
    """Очередь с паттерном Request/Reply"""

    def __init__(self):
        self._requests = InMemoryQueue()
        self._responses: dict[str, Any] = {}
        self._conditions: dict[str, threading.Condition] = {}

    def request(self, message: Any, timeout: float = 30) -> Any:
        """Отправить запрос и ждать ответ"""
        correlation_id = str(uuid.uuid4())

        condition = threading.Condition()
        self._conditions[correlation_id] = condition

        # Отправляем запрос
        request_msg = Message(
            body=message,
            headers={"correlation_id": correlation_id}
        )
        self._requests.send(request_msg)

        # Ждём ответ
        with condition:
            if not condition.wait(timeout):
                raise TimeoutError("Request timed out")

        response = self._responses.pop(correlation_id)
        del self._conditions[correlation_id]

        return response

    def reply(self, correlation_id: str, response: Any) -> None:
        """Отправить ответ"""
        self._responses[correlation_id] = response

        if correlation_id in self._conditions:
            with self._conditions[correlation_id]:
                self._conditions[correlation_id].notify()

    def get_request(self, timeout: float = None) -> Optional[Message]:
        """Получить запрос для обработки"""
        return self._requests.receive(timeout)
```

### Dead Letter Queue

```python
class DeadLetterQueue:
    """Очередь недоставленных сообщений"""

    def __init__(self, main_queue: MessageQueue):
        self._main_queue = main_queue
        self._dlq: List[Message] = []
        self._max_retries = 3

    def send(self, message: Message) -> None:
        """Отправить в основную очередь"""
        message.attempts = 0
        self._main_queue.send(message)

    def receive_and_process(self, handler: Callable) -> None:
        """Получить и обработать сообщение"""
        message = self._main_queue.receive()
        if not message:
            return

        try:
            handler(message.body)
            self._main_queue.ack(message.id)
        except Exception as e:
            message.attempts += 1

            if message.attempts >= self._max_retries:
                # Перемещаем в DLQ
                message.status = MessageStatus.DEAD_LETTER
                message.headers["error"] = str(e)
                message.headers["failed_at"] = datetime.now().isoformat()
                self._dlq.append(message)
                self._main_queue.ack(message.id)
                print(f"Message {message.id} moved to DLQ")
            else:
                # Повторяем
                self._main_queue.nack(message.id, requeue=True)

    def get_dead_letters(self) -> List[Message]:
        """Получить недоставленные сообщения"""
        return self._dlq.copy()

    def retry_dead_letter(self, message_id: str) -> bool:
        """Повторить обработку из DLQ"""
        for i, msg in enumerate(self._dlq):
            if msg.id == message_id:
                msg.attempts = 0
                msg.status = MessageStatus.PENDING
                self._main_queue.send(msg)
                self._dlq.pop(i)
                return True
        return False
```

## Сравнение очередей и стримов

```
┌────────────────────────────────────────────────────────────────────────┐
│               Message Queue vs Event Stream                            │
├──────────────────────────────┬─────────────────────────────────────────┤
│       Message Queue          │           Event Stream                  │
├──────────────────────────────┼─────────────────────────────────────────┤
│                              │                                         │
│  Сообщение удаляется после   │  События сохраняются                   │
│  обработки                   │  (log-based storage)                   │
│                              │                                         │
│  Один consumer на сообщение  │  Множество consumers                   │
│                              │                                         │
│  Point-to-point              │  Publish-subscribe                     │
│                              │                                         │
│  Нет replay                  │  Replay возможен                       │
│                              │                                         │
│  RabbitMQ, Amazon SQS       │  Apache Kafka, Amazon Kinesis          │
│                              │                                         │
│  Подходит для:               │  Подходит для:                         │
│  - Task processing           │  - Event sourcing                      │
│  - Work queues               │  - Analytics                           │
│  - Request/Reply             │  - Real-time processing                │
│                              │                                         │
└──────────────────────────────┴─────────────────────────────────────────┘
```

## Плюсы и минусы

### Плюсы

1. **Развязка** — производители и потребители независимы
2. **Масштабируемость** — легко добавлять потребителей
3. **Отказоустойчивость** — сообщения сохраняются при сбоях
4. **Асинхронность** — неблокирующая обработка
5. **Пиковые нагрузки** — буферизация сообщений

### Минусы

1. **Сложность** — дополнительная инфраструктура
2. **Eventual consistency** — данные не сразу консистентны
3. **Отладка** — сложнее отслеживать поток сообщений
4. **Порядок** — не всегда гарантируется
5. **Дублирование** — возможна повторная обработка

## Когда использовать

### Используйте Message Queues когда:

- Нужна гарантированная доставка
- Обработка задач в фоне
- Распределение нагрузки между воркерами
- Асинхронные операции

### Используйте Streams когда:

- Event sourcing
- Аналитика в реальном времени
- Множественные потребители одних событий
- Нужен replay событий

## Заключение

Message Queues и Streams — фундаментальные паттерны для построения масштабируемых распределённых систем. Выбор между ними зависит от требований: очереди для обработки задач, стримы для event-driven архитектуры и аналитики.
