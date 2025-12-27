# Messaging Queues

## Введение

**Message Queue (Очередь сообщений)** — это механизм асинхронной коммуникации между компонентами системы. Производители (producers) отправляют сообщения в очередь, а потребители (consumers) получают и обрабатывают их независимо от производителей.

## Основные концепции

### Архитектурная диаграмма

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Message Queue Architecture                       │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────┐      ┌─────────────────────────────┐      ┌──────────┐  │
│  │          │      │                             │      │          │  │
│  │ Producer │ ───> │      Message Broker         │ ───> │ Consumer │  │
│  │    1     │      │                             │      │    1     │  │
│  └──────────┘      │  ┌─────────────────────┐   │      └──────────┘  │
│                    │  │      Queue A        │   │                     │
│  ┌──────────┐      │  │ [msg1][msg2][msg3]  │   │      ┌──────────┐  │
│  │          │      │  └─────────────────────┘   │      │          │  │
│  │ Producer │ ───> │                             │ ───> │ Consumer │  │
│  │    2     │      │  ┌─────────────────────┐   │      │    2     │  │
│  └──────────┘      │  │      Queue B        │   │      └──────────┘  │
│                    │  │ [msg4][msg5]        │   │                     │
│  ┌──────────┐      │  └─────────────────────┘   │      ┌──────────┐  │
│  │          │      │                             │      │          │  │
│  │ Producer │ ───> │  ┌─────────────────────┐   │ ───> │ Consumer │  │
│  │    3     │      │  │      Queue C        │   │      │    3     │  │
│  └──────────┘      │  │ [msg6]              │   │      └──────────┘  │
│                    │  └─────────────────────┘   │                     │
│                    │                             │                     │
│                    └─────────────────────────────┘                     │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

### Ключевые термины

| Термин | Описание |
|--------|----------|
| **Producer** | Отправляет сообщения в очередь |
| **Consumer** | Получает и обрабатывает сообщения |
| **Broker** | Сервер, управляющий очередями |
| **Queue** | Буфер для хранения сообщений |
| **Topic** | Канал для pub/sub модели |
| **Exchange** | Маршрутизатор сообщений (RabbitMQ) |
| **Partition** | Раздел топика для параллелизма (Kafka) |

---

## Паттерны обмена сообщениями

### 1. Point-to-Point (Queue)

Одно сообщение обрабатывается одним потребителем.

```
Producer ──> [Queue] ──> Consumer
                 │
                 └───> (только один consumer получит сообщение)
```

```python
# RabbitMQ Point-to-Point
import pika
import json

# Producer
class TaskProducer:
    def __init__(self, connection_url: str):
        self.connection = pika.BlockingConnection(
            pika.URLParameters(connection_url)
        )
        self.channel = self.connection.channel()
        self.channel.queue_declare(queue='task_queue', durable=True)

    def send_task(self, task: dict):
        self.channel.basic_publish(
            exchange='',
            routing_key='task_queue',
            body=json.dumps(task),
            properties=pika.BasicProperties(
                delivery_mode=2,  # persistent
                content_type='application/json'
            )
        )

    def close(self):
        self.connection.close()

# Consumer
class TaskConsumer:
    def __init__(self, connection_url: str):
        self.connection = pika.BlockingConnection(
            pika.URLParameters(connection_url)
        )
        self.channel = self.connection.channel()
        self.channel.queue_declare(queue='task_queue', durable=True)
        # Prefetch — получаем только 1 сообщение за раз
        self.channel.basic_qos(prefetch_count=1)

    def process_task(self, ch, method, properties, body):
        task = json.loads(body)
        print(f'Processing task: {task}')

        # Обработка задачи...
        result = self.do_work(task)

        # Подтверждаем обработку
        ch.basic_ack(delivery_tag=method.delivery_tag)

    def start_consuming(self):
        self.channel.basic_consume(
            queue='task_queue',
            on_message_callback=self.process_task
        )
        self.channel.start_consuming()

# Использование
producer = TaskProducer('amqp://localhost')
producer.send_task({'type': 'send_email', 'to': 'user@example.com'})
producer.close()

consumer = TaskConsumer('amqp://localhost')
consumer.start_consuming()  # Блокирующий вызов
```

### 2. Publish/Subscribe (Topic)

Одно сообщение получают все подписчики.

```
                    ┌──> Consumer 1
Producer ──> Topic ─┼──> Consumer 2
                    └──> Consumer 3
```

```python
# RabbitMQ Pub/Sub с fanout exchange
import pika

# Publisher
class EventPublisher:
    def __init__(self, connection_url: str):
        self.connection = pika.BlockingConnection(
            pika.URLParameters(connection_url)
        )
        self.channel = self.connection.channel()
        # fanout exchange — рассылает всем подписчикам
        self.channel.exchange_declare(
            exchange='events',
            exchange_type='fanout'
        )

    def publish(self, event: dict):
        self.channel.basic_publish(
            exchange='events',
            routing_key='',  # игнорируется для fanout
            body=json.dumps(event)
        )

# Subscriber
class EventSubscriber:
    def __init__(self, connection_url: str, handler):
        self.connection = pika.BlockingConnection(
            pika.URLParameters(connection_url)
        )
        self.channel = self.connection.channel()
        self.channel.exchange_declare(
            exchange='events',
            exchange_type='fanout'
        )

        # Создаём временную очередь для этого подписчика
        result = self.channel.queue_declare(queue='', exclusive=True)
        self.queue_name = result.method.queue

        # Привязываем очередь к exchange
        self.channel.queue_bind(
            exchange='events',
            queue=self.queue_name
        )

        self.handler = handler

    def start(self):
        self.channel.basic_consume(
            queue=self.queue_name,
            on_message_callback=lambda ch, m, p, b: self.handler(json.loads(b)),
            auto_ack=True
        )
        self.channel.start_consuming()

# Разные подписчики на одни события
notification_subscriber = EventSubscriber(url, send_notification)
analytics_subscriber = EventSubscriber(url, track_analytics)
audit_subscriber = EventSubscriber(url, log_to_audit)
```

### 3. Routing (Topic с фильтрацией)

Сообщения маршрутизируются по ключам.

```
                          ┌──> Consumer A (orders.*)
Producer ──> Exchange ────┼──> Consumer B (orders.created)
     (orders.created)     └──> Consumer C (*.created)
```

```python
# RabbitMQ Topic Exchange
class RoutingPublisher:
    def __init__(self, connection_url: str):
        self.connection = pika.BlockingConnection(
            pika.URLParameters(connection_url)
        )
        self.channel = self.connection.channel()
        self.channel.exchange_declare(
            exchange='business_events',
            exchange_type='topic'
        )

    def publish(self, routing_key: str, event: dict):
        """
        Routing keys примеры:
        - orders.created
        - orders.shipped
        - payments.processed
        - users.registered
        """
        self.channel.basic_publish(
            exchange='business_events',
            routing_key=routing_key,
            body=json.dumps(event)
        )

class RoutingSubscriber:
    def __init__(self, connection_url: str, binding_keys: list, handler):
        self.connection = pika.BlockingConnection(
            pika.URLParameters(connection_url)
        )
        self.channel = self.connection.channel()
        self.channel.exchange_declare(
            exchange='business_events',
            exchange_type='topic'
        )

        result = self.channel.queue_declare(queue='', exclusive=True)
        self.queue_name = result.method.queue

        # Подписываемся на нужные паттерны
        for key in binding_keys:
            self.channel.queue_bind(
                exchange='business_events',
                queue=self.queue_name,
                routing_key=key
            )

        self.handler = handler

# Подписчики с разными фильтрами
# Получает все события orders.*
order_subscriber = RoutingSubscriber(url, ['orders.*'], handle_orders)

# Получает только orders.created и payments.processed
notification_sub = RoutingSubscriber(
    url,
    ['orders.created', 'payments.processed'],
    send_notification
)

# Получает все события *.created
audit_sub = RoutingSubscriber(url, ['*.created'], log_creation)
```

---

## Популярные Message Brokers

### 1. RabbitMQ

**Особенности:**
- AMQP протокол
- Гибкая маршрутизация (exchanges)
- Подтверждения доставки
- Кластеризация

```python
# Асинхронный клиент с aio-pika
import aio_pika
import asyncio

class AsyncRabbitMQClient:
    def __init__(self, url: str):
        self.url = url
        self.connection = None
        self.channel = None

    async def connect(self):
        self.connection = await aio_pika.connect_robust(self.url)
        self.channel = await self.connection.channel()
        await self.channel.set_qos(prefetch_count=10)

    async def declare_queue(self, name: str, durable: bool = True):
        return await self.channel.declare_queue(name, durable=durable)

    async def publish(self, queue_name: str, message: dict):
        await self.channel.default_exchange.publish(
            aio_pika.Message(
                body=json.dumps(message).encode(),
                delivery_mode=aio_pika.DeliveryMode.PERSISTENT
            ),
            routing_key=queue_name
        )

    async def consume(self, queue_name: str, handler):
        queue = await self.declare_queue(queue_name)

        async with queue.iterator() as queue_iter:
            async for message in queue_iter:
                async with message.process():
                    data = json.loads(message.body.decode())
                    await handler(data)

    async def close(self):
        await self.connection.close()

# Использование
async def main():
    client = AsyncRabbitMQClient('amqp://localhost')
    await client.connect()

    # Отправка
    await client.publish('tasks', {'action': 'process', 'data': {...}})

    # Потребление
    async def handler(message):
        print(f'Received: {message}')

    await client.consume('tasks', handler)

asyncio.run(main())
```

### 2. Apache Kafka

**Особенности:**
- Высокая пропускная способность
- Хранение сообщений (log)
- Партиционирование
- Consumer Groups

```
┌────────────────────────────────────────────────────────────────────────┐
│                           Kafka Architecture                            │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Topic: orders                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │                                                                  │  │
│   │  Partition 0:  [msg0][msg3][msg6][msg9]...   ──> Consumer A     │  │
│   │                                                                  │  │
│   │  Partition 1:  [msg1][msg4][msg7][msg10]...  ──> Consumer B     │  │
│   │                                                                  │  │
│   │  Partition 2:  [msg2][msg5][msg8][msg11]...  ──> Consumer C     │  │
│   │                                                                  │  │
│   └─────────────────────────────────────────────────────────────────┘  │
│                           │                                             │
│                    Consumer Group                                       │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

```python
# Kafka с aiokafka
from aiokafka import AIOKafkaProducer, AIOKafkaConsumer
import asyncio
import json

class KafkaClient:
    def __init__(self, bootstrap_servers: str):
        self.bootstrap_servers = bootstrap_servers

    async def create_producer(self) -> AIOKafkaProducer:
        producer = AIOKafkaProducer(
            bootstrap_servers=self.bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode(),
            key_serializer=lambda k: k.encode() if k else None,
            acks='all',  # Ждём подтверждения от всех реплик
            enable_idempotence=True,  # Exactly-once семантика
            compression_type='gzip'
        )
        await producer.start()
        return producer

    async def create_consumer(
        self,
        topic: str,
        group_id: str
    ) -> AIOKafkaConsumer:
        consumer = AIOKafkaConsumer(
            topic,
            bootstrap_servers=self.bootstrap_servers,
            group_id=group_id,
            value_deserializer=lambda v: json.loads(v.decode()),
            auto_offset_reset='earliest',
            enable_auto_commit=False  # Ручной commit
        )
        await consumer.start()
        return consumer

# Producer
async def produce_orders():
    client = KafkaClient('localhost:9092')
    producer = await client.create_producer()

    try:
        for i in range(100):
            order = {
                'order_id': f'order-{i}',
                'customer_id': f'customer-{i % 10}',
                'total': 100.0 + i
            }

            # Ключ определяет партицию
            await producer.send_and_wait(
                topic='orders',
                key=order['customer_id'],  # Заказы одного клиента в одной партиции
                value=order
            )
    finally:
        await producer.stop()

# Consumer
async def consume_orders():
    client = KafkaClient('localhost:9092')
    consumer = await client.create_consumer('orders', 'order-processor')

    try:
        async for message in consumer:
            order = message.value
            print(f'Processing order: {order}')

            # Обрабатываем заказ...

            # Commit offset после успешной обработки
            await consumer.commit()
    finally:
        await consumer.stop()

# Consumer Group — несколько инстансов обрабатывают партиции параллельно
async def run_consumer_group():
    tasks = [
        consume_orders(),  # Получит партицию 0
        consume_orders(),  # Получит партицию 1
        consume_orders()   # Получит партицию 2
    ]
    await asyncio.gather(*tasks)
```

### 3. Redis Streams

**Особенности:**
- Простота использования
- Низкая латентность
- Consumer Groups (с Redis 5.0+)
- Идеально для real-time данных

```python
import redis.asyncio as redis
import asyncio
import json

class RedisStreamClient:
    def __init__(self, url: str):
        self.redis = redis.from_url(url)

    async def produce(self, stream: str, data: dict):
        """Добавляет сообщение в stream."""
        await self.redis.xadd(
            stream,
            {'data': json.dumps(data)},
            maxlen=10000  # Ограничение размера
        )

    async def consume(
        self,
        stream: str,
        group: str,
        consumer: str,
        handler
    ):
        """Потребляет сообщения из stream."""
        # Создаём consumer group если не существует
        try:
            await self.redis.xgroup_create(stream, group, id='0', mkstream=True)
        except redis.ResponseError:
            pass  # Группа уже существует

        while True:
            # Читаем новые сообщения
            messages = await self.redis.xreadgroup(
                groupname=group,
                consumername=consumer,
                streams={stream: '>'},  # Только новые
                count=10,
                block=5000  # Ждём 5 секунд
            )

            for stream_name, stream_messages in messages:
                for message_id, fields in stream_messages:
                    data = json.loads(fields[b'data'])

                    try:
                        await handler(data)
                        # Подтверждаем обработку
                        await self.redis.xack(stream, group, message_id)
                    except Exception as e:
                        print(f'Error processing message: {e}')
                        # Сообщение останется в pending

    async def get_pending(self, stream: str, group: str) -> list:
        """Получает необработанные сообщения."""
        return await self.redis.xpending(stream, group)

    async def claim_pending(
        self,
        stream: str,
        group: str,
        consumer: str,
        min_idle_time: int = 60000
    ):
        """Забирает 'застрявшие' сообщения от других consumers."""
        pending = await self.redis.xpending_range(
            stream, group,
            min='-', max='+',
            count=10
        )

        for msg in pending:
            if msg['time_since_delivered'] > min_idle_time:
                await self.redis.xclaim(
                    stream, group, consumer,
                    min_idle_time=min_idle_time,
                    message_ids=[msg['message_id']]
                )

# Использование
async def main():
    client = RedisStreamClient('redis://localhost')

    # Producer
    await client.produce('events', {
        'type': 'user.created',
        'user_id': '123'
    })

    # Consumer
    async def handler(data):
        print(f'Handling: {data}')

    await client.consume(
        stream='events',
        group='event-handlers',
        consumer='handler-1',
        handler=handler
    )
```

### 4. Amazon SQS

```python
import boto3
import json
from typing import Callable
import asyncio

class SQSClient:
    def __init__(self, queue_url: str):
        self.sqs = boto3.client('sqs')
        self.queue_url = queue_url

    def send_message(self, message: dict, delay_seconds: int = 0):
        """Отправляет сообщение в очередь."""
        response = self.sqs.send_message(
            QueueUrl=self.queue_url,
            MessageBody=json.dumps(message),
            DelaySeconds=delay_seconds,
            MessageAttributes={
                'MessageType': {
                    'DataType': 'String',
                    'StringValue': message.get('type', 'default')
                }
            }
        )
        return response['MessageId']

    def send_batch(self, messages: list):
        """Отправляет пакет сообщений (до 10)."""
        entries = [
            {
                'Id': str(i),
                'MessageBody': json.dumps(msg)
            }
            for i, msg in enumerate(messages)
        ]

        return self.sqs.send_message_batch(
            QueueUrl=self.queue_url,
            Entries=entries
        )

    def receive_messages(
        self,
        max_messages: int = 10,
        wait_time: int = 20
    ) -> list:
        """Получает сообщения из очереди (long polling)."""
        response = self.sqs.receive_message(
            QueueUrl=self.queue_url,
            MaxNumberOfMessages=max_messages,
            WaitTimeSeconds=wait_time,
            MessageAttributeNames=['All']
        )

        return response.get('Messages', [])

    def delete_message(self, receipt_handle: str):
        """Удаляет обработанное сообщение."""
        self.sqs.delete_message(
            QueueUrl=self.queue_url,
            ReceiptHandle=receipt_handle
        )

    def process_messages(self, handler: Callable):
        """Обрабатывает сообщения в цикле."""
        while True:
            messages = self.receive_messages()

            for message in messages:
                try:
                    body = json.loads(message['Body'])
                    handler(body)
                    self.delete_message(message['ReceiptHandle'])
                except Exception as e:
                    print(f'Error: {e}')
                    # Сообщение вернётся в очередь после visibility timeout

# Dead Letter Queue
class SQSWithDLQ:
    def __init__(self, main_queue_url: str, dlq_url: str):
        self.main_queue = SQSClient(main_queue_url)
        self.dlq = SQSClient(dlq_url)
        self.max_retries = 3

    def process_with_retry(self, handler: Callable):
        while True:
            messages = self.main_queue.receive_messages()

            for message in messages:
                retry_count = int(
                    message.get('Attributes', {}).get('ApproximateReceiveCount', 0)
                )

                try:
                    body = json.loads(message['Body'])
                    handler(body)
                    self.main_queue.delete_message(message['ReceiptHandle'])

                except Exception as e:
                    if retry_count >= self.max_retries:
                        # Отправляем в DLQ
                        self.dlq.send_message({
                            'original_message': body,
                            'error': str(e),
                            'retry_count': retry_count
                        })
                        self.main_queue.delete_message(message['ReceiptHandle'])
                    # Иначе сообщение вернётся автоматически
```

---

## Гарантии доставки

### 1. At-Most-Once (Не более одного раза)

Сообщение может быть потеряно, но не будет дублироваться.

```python
# Auto-ack — сообщение удаляется сразу после получения
async def at_most_once_consumer(queue):
    async for message in queue.iterator():
        # Сообщение уже удалено из очереди
        # Если обработка упадёт — данные потеряны
        process(message)
```

### 2. At-Least-Once (Как минимум один раз)

Сообщение будет доставлено, но возможны дубликаты.

```python
# Manual ack — подтверждаем после обработки
async def at_least_once_consumer(queue):
    async for message in queue.iterator():
        async with message.process():  # ack после успеха
            try:
                process(message)
                # Если упадёт здесь — сообщение обработается повторно
            except Exception:
                # Сообщение вернётся в очередь
                raise
```

### 3. Exactly-Once (Ровно один раз)

Сообщение обрабатывается ровно один раз (требует идемпотентности).

```python
class ExactlyOnceProcessor:
    def __init__(self, db, queue):
        self.db = db
        self.queue = queue

    async def process(self, message):
        message_id = message.id

        # Проверяем, обрабатывали ли мы это сообщение
        async with self.db.transaction():
            if await self.db.processed_messages.exists(message_id):
                # Уже обработано — пропускаем
                await message.ack()
                return

            # Обрабатываем
            result = await self.handle(message.body)

            # Записываем, что обработали
            await self.db.processed_messages.insert({
                'message_id': message_id,
                'processed_at': datetime.utcnow()
            })

            await message.ack()

# Kafka Transactions для exactly-once
async def kafka_exactly_once():
    producer = AIOKafkaProducer(
        bootstrap_servers='localhost:9092',
        transactional_id='my-transactional-producer'
    )
    await producer.start()

    async with producer.transaction():
        await producer.send('topic1', b'message1')
        await producer.send('topic2', b'message2')
        # Либо все сообщения доставлены, либо ни одного
```

---

## Паттерны обработки сообщений

### 1. Competing Consumers

Несколько consumers обрабатывают одну очередь параллельно.

```python
# Несколько workers обрабатывают одну очередь
async def run_workers(num_workers: int):
    tasks = []
    for i in range(num_workers):
        worker = Worker(f'worker-{i}')
        tasks.append(worker.run())
    await asyncio.gather(*tasks)

class Worker:
    def __init__(self, name: str):
        self.name = name
        self.consumer = await create_consumer()

    async def run(self):
        async for message in self.consumer:
            print(f'{self.name} processing: {message}')
            await self.process(message)
            await message.ack()
```

### 2. Request-Reply

Запрос через очередь с ожиданием ответа.

```python
import uuid

class RequestReplyClient:
    def __init__(self, channel):
        self.channel = channel
        self.callback_queue = channel.queue_declare(
            queue='',
            exclusive=True
        ).method.queue
        self.pending = {}

        channel.basic_consume(
            queue=self.callback_queue,
            on_message_callback=self._on_response,
            auto_ack=True
        )

    def _on_response(self, ch, method, props, body):
        correlation_id = props.correlation_id
        if correlation_id in self.pending:
            future = self.pending.pop(correlation_id)
            future.set_result(json.loads(body))

    async def call(self, routing_key: str, request: dict) -> dict:
        correlation_id = str(uuid.uuid4())
        future = asyncio.Future()
        self.pending[correlation_id] = future

        self.channel.basic_publish(
            exchange='',
            routing_key=routing_key,
            properties=pika.BasicProperties(
                reply_to=self.callback_queue,
                correlation_id=correlation_id
            ),
            body=json.dumps(request)
        )

        # Ждём ответ
        return await asyncio.wait_for(future, timeout=30.0)

# RPC Server
class RPCServer:
    def __init__(self, channel, queue_name: str):
        self.channel = channel
        channel.queue_declare(queue=queue_name)
        channel.basic_consume(
            queue=queue_name,
            on_message_callback=self._on_request
        )

    def _on_request(self, ch, method, props, body):
        request = json.loads(body)

        # Обрабатываем запрос
        response = self.handle(request)

        # Отправляем ответ
        ch.basic_publish(
            exchange='',
            routing_key=props.reply_to,
            properties=pika.BasicProperties(
                correlation_id=props.correlation_id
            ),
            body=json.dumps(response)
        )
        ch.basic_ack(delivery_tag=method.delivery_tag)
```

### 3. Dead Letter Queue (DLQ)

Обработка неудачных сообщений.

```python
# RabbitMQ с DLQ
class QueueWithDLQ:
    def __init__(self, channel, queue_name: str):
        self.channel = channel

        # Dead Letter Exchange
        channel.exchange_declare(
            exchange=f'{queue_name}.dlx',
            exchange_type='direct'
        )

        # Dead Letter Queue
        channel.queue_declare(
            queue=f'{queue_name}.dlq',
            durable=True
        )
        channel.queue_bind(
            queue=f'{queue_name}.dlq',
            exchange=f'{queue_name}.dlx',
            routing_key='dead-letter'
        )

        # Main Queue с DLQ
        channel.queue_declare(
            queue=queue_name,
            durable=True,
            arguments={
                'x-dead-letter-exchange': f'{queue_name}.dlx',
                'x-dead-letter-routing-key': 'dead-letter',
                'x-message-ttl': 60000  # 1 минута
            }
        )

class DLQProcessor:
    """Обрабатывает сообщения из DLQ."""

    def __init__(self, dlq_name: str):
        self.dlq_name = dlq_name

    async def process_failed_messages(self):
        async for message in self.consume_dlq():
            original_error = message.headers.get('x-first-death-reason')
            original_queue = message.headers.get('x-first-death-queue')

            # Анализируем ошибку
            if self.is_retryable(original_error):
                # Повторная попытка
                await self.requeue(message, original_queue)
            else:
                # Логируем и уведомляем
                await self.alert_and_log(message, original_error)
```

### 4. Priority Queue

```python
# RabbitMQ с приоритетами
channel.queue_declare(
    queue='priority_queue',
    durable=True,
    arguments={'x-max-priority': 10}  # Приоритеты 0-10
)

# Отправка с приоритетом
def send_with_priority(message: dict, priority: int):
    channel.basic_publish(
        exchange='',
        routing_key='priority_queue',
        body=json.dumps(message),
        properties=pika.BasicProperties(
            priority=priority,  # 0 = низкий, 10 = высокий
            delivery_mode=2
        )
    )

# Важные сообщения обрабатываются первыми
send_with_priority({'type': 'critical_alert'}, priority=10)
send_with_priority({'type': 'normal_task'}, priority=1)
```

---

## Best Practices

### 1. Idempotency (Идемпотентность)

```python
class IdempotentConsumer:
    def __init__(self, redis):
        self.redis = redis

    async def process(self, message):
        message_id = message.headers['message_id']

        # Проверяем, обрабатывали ли
        if await self.redis.sismember('processed_messages', message_id):
            return  # Пропускаем дубликат

        # Обрабатываем
        result = await self.handle(message)

        # Помечаем как обработанное (с TTL)
        await self.redis.sadd('processed_messages', message_id)
        await self.redis.expire('processed_messages', 86400)  # 24 часа

        return result
```

### 2. Message Schema Evolution

```python
from dataclasses import dataclass
from typing import Optional

# Версионирование сообщений
@dataclass
class MessageV1:
    version: int = 1
    order_id: str
    amount: float

@dataclass
class MessageV2:
    version: int = 2
    order_id: str
    amount: float
    currency: str  # Новое поле

class MessageHandler:
    def handle(self, raw_message: dict):
        version = raw_message.get('version', 1)

        if version == 1:
            message = MessageV1(**raw_message)
            # Конвертируем к новой версии
            message = self.upgrade_v1_to_v2(message)
        else:
            message = MessageV2(**raw_message)

        return self.process(message)

    def upgrade_v1_to_v2(self, v1: MessageV1) -> MessageV2:
        return MessageV2(
            version=2,
            order_id=v1.order_id,
            amount=v1.amount,
            currency='USD'  # Default
        )
```

### 3. Monitoring & Alerting

```python
from prometheus_client import Counter, Histogram, Gauge

# Метрики
MESSAGES_PROCESSED = Counter(
    'messages_processed_total',
    'Total messages processed',
    ['queue', 'status']
)

PROCESSING_TIME = Histogram(
    'message_processing_seconds',
    'Time spent processing messages',
    ['queue']
)

QUEUE_SIZE = Gauge(
    'queue_size',
    'Current queue size',
    ['queue']
)

class MonitoredConsumer:
    async def process(self, message):
        queue = message.queue_name

        with PROCESSING_TIME.labels(queue=queue).time():
            try:
                result = await self.handle(message)
                MESSAGES_PROCESSED.labels(queue=queue, status='success').inc()
                return result
            except Exception as e:
                MESSAGES_PROCESSED.labels(queue=queue, status='error').inc()
                raise

    async def update_queue_metrics(self):
        while True:
            for queue_name in self.queues:
                size = await self.get_queue_size(queue_name)
                QUEUE_SIZE.labels(queue=queue_name).set(size)
            await asyncio.sleep(10)
```

---

## Типичные ошибки

### 1. Poison Message

```python
# ❌ Плохо: бесконечный retry
async def bad_consumer(message):
    try:
        process(message)  # Всегда падает
    except:
        message.nack()  # Сообщение возвращается в очередь
        # И так бесконечно...

# ✅ Хорошо: ограничение retry + DLQ
async def good_consumer(message):
    retry_count = message.headers.get('x-retry-count', 0)

    try:
        process(message)
    except Exception as e:
        if retry_count >= 3:
            await send_to_dlq(message, error=str(e))
        else:
            await requeue_with_delay(message, retry_count + 1)
```

### 2. Message Ordering

```python
# ❌ Плохо: потеря порядка при параллельной обработке
# Consumer 1: обрабатывает "update balance"
# Consumer 2: обрабатывает "check balance" (опережает)

# ✅ Хорошо: партиционирование по ключу (Kafka)
producer.send(
    topic='account_events',
    key=account_id,  # Все события аккаунта в одной партиции
    value=event
)
```

### 3. Lost Acknowledgement

```python
# ❌ Плохо: ack до обработки
async def bad_process(message):
    await message.ack()  # Сообщение удалено
    await do_work(message)  # Если упадёт — данные потеряны

# ✅ Хорошо: ack после успешной обработки
async def good_process(message):
    try:
        await do_work(message)
        await message.ack()
    except Exception:
        await message.nack()  # Вернётся в очередь
```

---

## Сравнение решений

| Критерий | RabbitMQ | Kafka | Redis Streams | SQS |
|----------|----------|-------|---------------|-----|
| Модель | Queue/Topic | Log | Stream | Queue |
| Throughput | Средний | Очень высокий | Высокий | Средний |
| Latency | Низкая | Средняя | Очень низкая | Средняя |
| Persistence | Опционально | Да | Опционально | Да |
| Ordering | Per-queue | Per-partition | Per-stream | Нет |
| Scaling | Кластер | Партиции | Кластер | Авто |
| Use case | Tasks, RPC | Event streaming | Real-time | Serverless |

---

## Заключение

Message Queues — фундаментальный инструмент для построения распределённых систем. Ключевые принципы:

1. **Выбирайте правильную гарантию доставки** для вашего use case
2. **Делайте обработчики идемпотентными** — дубликаты неизбежны
3. **Используйте DLQ** для обработки проблемных сообщений
4. **Мониторьте очереди** — размер, latency, error rate
5. **Версионируйте сообщения** — схема будет меняться
