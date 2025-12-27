# Message Queues

## Введение

**Message Queue (очередь сообщений)** — это механизм асинхронной коммуникации между сервисами, где сообщения сохраняются в очереди до тех пор, пока получатель не будет готов их обработать. Это ключевой компонент распределённых систем, обеспечивающий развязку (decoupling), масштабируемость и отказоустойчивость.

### Зачем нужны очереди сообщений?

1. **Развязка сервисов** — отправитель не знает о получателе
2. **Буферизация нагрузки** — сглаживание пиков трафика
3. **Гарантия доставки** — сообщения не теряются при сбоях
4. **Масштабирование** — добавление consumers без изменения producers
5. **Асинхронная обработка** — не блокировать основной поток

---

## Основные концепции

### Базовые компоненты

```
┌──────────┐    ┌─────────────────┐    ┌──────────┐
│ Producer │───▶│  Message Queue  │───▶│ Consumer │
└──────────┘    └─────────────────┘    └──────────┘
     │                  │                   │
 Отправитель        Брокер            Получатель
```

**Producer (издатель)** — сервис, отправляющий сообщения в очередь.

**Consumer (потребитель)** — сервис, читающий и обрабатывающий сообщения.

**Broker (брокер)** — промежуточное ПО, управляющее очередями (RabbitMQ, Kafka, SQS).

**Message (сообщение)** — единица данных, передаваемая через очередь.

---

## Паттерны обмена сообщениями

### 1. Point-to-Point (Очередь)

Каждое сообщение обрабатывается **ровно одним** consumer'ом.

```
                    ┌──────────┐
                    │Consumer 1│
┌────────┐         ┌┴──────────┤
│Producer│──▶Queue─┤Consumer 2 │  ← только один получит сообщение
└────────┘         └┬──────────┤
                    │Consumer 3│
                    └──────────┘
```

**Применение:**
- Обработка заказов
- Выполнение задач (task queue)
- Распределение работы между воркерами

**Пример с RabbitMQ (Python):**

```python
import pika
import json

# Producer
def send_task(task_data):
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Создаём durable очередь (переживёт перезапуск брокера)
    channel.queue_declare(queue='task_queue', durable=True)

    channel.basic_publish(
        exchange='',
        routing_key='task_queue',
        body=json.dumps(task_data),
        properties=pika.BasicProperties(
            delivery_mode=2,  # Persistent message
            content_type='application/json'
        )
    )
    connection.close()

# Consumer
def process_tasks():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()
    channel.queue_declare(queue='task_queue', durable=True)

    # Prefetch = 1: не брать новое сообщение, пока не обработано текущее
    channel.basic_qos(prefetch_count=1)

    def callback(ch, method, properties, body):
        task = json.loads(body)
        print(f"Processing: {task}")

        # Обработка задачи...

        # Подтверждение обработки
        ch.basic_ack(delivery_tag=method.delivery_tag)

    channel.basic_consume(
        queue='task_queue',
        on_message_callback=callback
    )

    print("Waiting for tasks...")
    channel.start_consuming()
```

### 2. Publish/Subscribe (Pub/Sub)

Сообщение доставляется **всем** подписчикам.

```
                         ┌──────────┐
                    ┌───▶│Consumer 1│
┌────────┐         │    └──────────┘
│Producer│──▶Topic─┼───▶│Consumer 2│  ← все получат копию сообщения
└────────┘         │    └──────────┘
                    └───▶│Consumer 3│
                         └──────────┘
```

**Применение:**
- Уведомления
- Event broadcasting
- Логирование
- Обновление кэшей

**Пример с RabbitMQ (fanout exchange):**

```python
# Publisher
def publish_event(event):
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Fanout exchange рассылает всем привязанным очередям
    channel.exchange_declare(
        exchange='events',
        exchange_type='fanout'
    )

    channel.basic_publish(
        exchange='events',
        routing_key='',  # Игнорируется для fanout
        body=json.dumps(event)
    )
    connection.close()

# Subscriber
def subscribe_to_events(handler_name):
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    channel.exchange_declare(
        exchange='events',
        exchange_type='fanout'
    )

    # Создаём уникальную очередь для этого подписчика
    result = channel.queue_declare(queue='', exclusive=True)
    queue_name = result.method.queue

    # Привязываем очередь к exchange
    channel.queue_bind(exchange='events', queue=queue_name)

    def callback(ch, method, properties, body):
        event = json.loads(body)
        print(f"[{handler_name}] Received: {event}")

    channel.basic_consume(
        queue=queue_name,
        on_message_callback=callback,
        auto_ack=True
    )

    channel.start_consuming()
```

### 3. Topic-based Routing

Маршрутизация по паттерну в routing key.

```python
# RabbitMQ Topic Exchange
channel.exchange_declare(exchange='logs', exchange_type='topic')

# Publisher
channel.basic_publish(
    exchange='logs',
    routing_key='order.created.europe',  # Формат: category.action.region
    body=message
)

# Subscriber 1: все события заказов
channel.queue_bind(
    exchange='logs',
    queue=queue_name,
    routing_key='order.*.*'  # * - одно слово
)

# Subscriber 2: все события из Europe
channel.queue_bind(
    exchange='logs',
    queue=queue_name,
    routing_key='*.*.europe'
)

# Subscriber 3: все события
channel.queue_bind(
    exchange='logs',
    queue=queue_name,
    routing_key='#'  # # - ноль или более слов
)
```

---

## Популярные Message Brokers

### RabbitMQ

**Тип:** Traditional message broker (AMQP)

**Характеристики:**
- Push-based (брокер отправляет сообщения consumers)
- Сложная маршрутизация (exchanges, bindings)
- Подтверждение сообщений (acknowledgments)
- Приоритеты сообщений
- TTL для сообщений

```python
# RabbitMQ с retry логикой
import pika
from tenacity import retry, stop_after_attempt, wait_exponential

class RabbitMQClient:
    def __init__(self, host='localhost'):
        self.host = host
        self.connection = None
        self.channel = None

    def connect(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(
                host=self.host,
                heartbeat=600,
                blocked_connection_timeout=300
            )
        )
        self.channel = self.connection.channel()

    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=1, max=10)
    )
    def publish(self, exchange, routing_key, message):
        if not self.connection or self.connection.is_closed:
            self.connect()

        self.channel.basic_publish(
            exchange=exchange,
            routing_key=routing_key,
            body=json.dumps(message),
            properties=pika.BasicProperties(
                delivery_mode=2,
                content_type='application/json'
            )
        )

    def close(self):
        if self.connection:
            self.connection.close()
```

### Apache Kafka

**Тип:** Distributed event streaming platform

**Характеристики:**
- Pull-based (consumers запрашивают сообщения)
- Log-based storage (сообщения хранятся в логе)
- Высокая пропускная способность
- Партиционирование для масштабирования
- Consumer groups для параллельной обработки

```python
from kafka import KafkaProducer, KafkaConsumer
import json

# Producer
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    key_serializer=lambda k: k.encode('utf-8') if k else None,
    acks='all',  # Ждать подтверждения от всех реплик
    retries=3,
    max_in_flight_requests_per_connection=1  # Гарантия порядка
)

def send_event(topic, key, event):
    future = producer.send(
        topic=topic,
        key=key,  # Сообщения с одинаковым ключом попадут в одну партицию
        value=event
    )
    # Синхронное ожидание (опционально)
    record_metadata = future.get(timeout=10)
    return record_metadata

# Consumer
consumer = KafkaConsumer(
    'orders',
    bootstrap_servers=['localhost:9092'],
    group_id='order-processor',  # Consumer group
    auto_offset_reset='earliest',  # Читать с начала, если нет offset
    enable_auto_commit=False,  # Ручное подтверждение
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

for message in consumer:
    print(f"Partition: {message.partition}, Offset: {message.offset}")
    print(f"Key: {message.key}, Value: {message.value}")

    # Обработка сообщения...

    # Ручной commit offset
    consumer.commit()
```

**Партиционирование в Kafka:**

```
Topic: orders
┌─────────────────────────────────────────────────────┐
│ Partition 0: [msg1, msg4, msg7, msg10, ...]        │ ← Consumer 1
│ Partition 1: [msg2, msg5, msg8, msg11, ...]        │ ← Consumer 2
│ Partition 2: [msg3, msg6, msg9, msg12, ...]        │ ← Consumer 3
└─────────────────────────────────────────────────────┘

Consumer Group: order-processors
- Каждая партиция обрабатывается одним consumer'ом в группе
- Добавление consumers увеличивает параллелизм
- Больше consumers чем партиций = idle consumers
```

### Amazon SQS

**Тип:** Fully managed message queue (AWS)

**Два типа очередей:**
1. **Standard Queue** — высокая пропускная способность, at-least-once, возможен неправильный порядок
2. **FIFO Queue** — строгий порядок, exactly-once, до 300 сообщений/сек

```python
import boto3
import json

# Клиент SQS
sqs = boto3.client('sqs', region_name='us-east-1')

queue_url = 'https://sqs.us-east-1.amazonaws.com/123456789/my-queue'

# Отправка сообщения
def send_message(message_body, attributes=None):
    response = sqs.send_message(
        QueueUrl=queue_url,
        MessageBody=json.dumps(message_body),
        MessageAttributes=attributes or {},
        # Для FIFO очередей:
        # MessageGroupId='order-123',  # Группировка для FIFO
        # MessageDeduplicationId='unique-id'  # Дедупликация
    )
    return response['MessageId']

# Получение сообщений
def receive_messages(max_messages=10, wait_time=20):
    response = sqs.receive_message(
        QueueUrl=queue_url,
        MaxNumberOfMessages=max_messages,
        WaitTimeSeconds=wait_time,  # Long polling
        MessageAttributeNames=['All'],
        VisibilityTimeout=30  # Время на обработку
    )
    return response.get('Messages', [])

# Удаление сообщения после обработки
def delete_message(receipt_handle):
    sqs.delete_message(
        QueueUrl=queue_url,
        ReceiptHandle=receipt_handle
    )

# Обработка сообщений
def process_queue():
    while True:
        messages = receive_messages()

        for message in messages:
            try:
                body = json.loads(message['Body'])
                print(f"Processing: {body}")

                # Обработка...

                delete_message(message['ReceiptHandle'])
            except Exception as e:
                print(f"Error processing message: {e}")
                # Сообщение вернётся в очередь после visibility timeout
```

### Сравнительная таблица

| Характеристика | RabbitMQ | Kafka | SQS |
|---------------|----------|-------|-----|
| Модель | Push | Pull | Pull |
| Порядок | FIFO (per queue) | FIFO (per partition) | FIFO (optional) |
| Хранение | До ACK | Retention period | 14 дней |
| Throughput | ~10K msg/sec | ~100K+ msg/sec | ~3K msg/sec |
| Replay | Нет | Да | Нет |
| Managed | Нет | Confluent Cloud | AWS |
| Use case | Task queues, routing | Event streaming, logs | Simple queuing |

---

## Гарантии доставки

### At-Most-Once (Не более одного раза)

Сообщение может быть потеряно, но никогда не будет доставлено дважды.

```python
# Kafka: acks=0 (fire and forget)
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    acks=0  # Не ждать подтверждения
)

# RabbitMQ: auto_ack=True
channel.basic_consume(
    queue='my_queue',
    on_message_callback=callback,
    auto_ack=True  # ACK отправляется сразу при получении
)
```

**Когда использовать:** Метрики, логи, где потеря части данных допустима.

### At-Least-Once (Не менее одного раза)

Сообщение гарантированно доставлено, но может быть доставлено несколько раз.

```python
# Kafka: acks=all + ручной commit
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    acks='all',
    retries=3
)

consumer = KafkaConsumer(
    'topic',
    enable_auto_commit=False
)

for message in consumer:
    process(message)  # Обработка
    consumer.commit()  # Commit после обработки

# Проблема: если процесс упадёт после process(), но до commit(),
# сообщение будет обработано повторно
```

**ВАЖНО:** При at-least-once обработчик должен быть **идемпотентным**.

```python
# Идемпотентная обработка с дедупликацией
processed_ids = set()  # В реальности — Redis или БД

def process_message(message):
    message_id = message.get('id')

    # Проверяем, обрабатывали ли уже
    if message_id in processed_ids:
        print(f"Duplicate message {message_id}, skipping")
        return

    # Обработка
    do_work(message)

    # Помечаем как обработанное
    processed_ids.add(message_id)
```

### Exactly-Once (Ровно один раз)

Сообщение гарантированно доставлено и обработано ровно один раз.

**Kafka Transactions (Producer):**

```python
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    transactional_id='my-transactional-id',  # Включает транзакции
    acks='all'
)

producer.init_transactions()

try:
    producer.begin_transaction()

    producer.send('topic1', value={'data': 1})
    producer.send('topic2', value={'data': 2})

    producer.commit_transaction()
except Exception as e:
    producer.abort_transaction()
    raise
```

**SQS FIFO с дедупликацией:**

```python
# Дедупликация на основе content
sqs.send_message(
    QueueUrl=fifo_queue_url,
    MessageBody=json.dumps(message),
    MessageGroupId='order-group',
    MessageDeduplicationId=hashlib.md5(
        json.dumps(message).encode()
    ).hexdigest()
)
# SQS отклонит дубликаты в течение 5 минут
```

---

## Dead Letter Queues (DLQ)

**Dead Letter Queue** — очередь для сообщений, которые не удалось обработать после нескольких попыток.

```
┌────────┐    ┌───────────┐    ┌──────────┐
│Producer│───▶│Main Queue │───▶│ Consumer │
└────────┘    └─────┬─────┘    └────┬─────┘
                    │               │
              (max retries)    (processing fails)
                    │               │
                    ▼               │
              ┌───────────┐         │
              │    DLQ    │◀────────┘
              └───────────┘
                    │
                    ▼
              Анализ ошибок / Ручная обработка
```

### RabbitMQ DLQ

```python
# Создание DLQ
channel.queue_declare(queue='task_queue_dlq', durable=True)

# Основная очередь с DLQ
channel.queue_declare(
    queue='task_queue',
    durable=True,
    arguments={
        'x-dead-letter-exchange': '',
        'x-dead-letter-routing-key': 'task_queue_dlq',
        'x-message-ttl': 60000,  # TTL 60 сек (опционально)
        'x-max-length': 10000    # Макс. размер очереди
    }
)

# Consumer с retry логикой
def callback(ch, method, properties, body):
    retry_count = (properties.headers or {}).get('x-retry-count', 0)
    max_retries = 3

    try:
        process_message(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception as e:
        if retry_count < max_retries:
            # Переотправка с увеличенным счётчиком
            ch.basic_publish(
                exchange='',
                routing_key='task_queue',
                body=body,
                properties=pika.BasicProperties(
                    headers={'x-retry-count': retry_count + 1},
                    delivery_mode=2
                )
            )
            ch.basic_ack(delivery_tag=method.delivery_tag)
        else:
            # Reject — сообщение попадёт в DLQ
            ch.basic_nack(
                delivery_tag=method.delivery_tag,
                requeue=False
            )
```

### SQS DLQ

```python
# Конфигурация через AWS Console или Terraform:
# RedrivePolicy: {"deadLetterTargetArn": "arn:aws:sqs:...:my-dlq", "maxReceiveCount": 3}

# Обработка DLQ
def process_dlq():
    dlq_url = 'https://sqs.../my-queue-dlq'

    while True:
        messages = sqs.receive_message(
            QueueUrl=dlq_url,
            MaxNumberOfMessages=10
        ).get('Messages', [])

        for message in messages:
            # Анализ ошибки
            body = json.loads(message['Body'])
            print(f"Failed message: {body}")

            # Варианты:
            # 1. Исправить и переотправить в основную очередь
            # 2. Сохранить для ручного анализа
            # 3. Уведомить команду

            # После обработки — удалить
            sqs.delete_message(
                QueueUrl=dlq_url,
                ReceiptHandle=message['ReceiptHandle']
            )
```

---

## Паттерны использования

### 1. Work Queue (Распределение задач)

```python
# Celery-подобная реализация с RabbitMQ
class TaskQueue:
    def __init__(self):
        self.connection = pika.BlockingConnection()
        self.channel = self.connection.channel()
        self.channel.queue_declare(queue='tasks', durable=True)
        self.channel.basic_qos(prefetch_count=1)

    def enqueue(self, task_name, args):
        task = {
            'id': str(uuid.uuid4()),
            'task': task_name,
            'args': args,
            'created_at': datetime.utcnow().isoformat()
        }
        self.channel.basic_publish(
            exchange='',
            routing_key='tasks',
            body=json.dumps(task),
            properties=pika.BasicProperties(delivery_mode=2)
        )
        return task['id']

    def worker(self, handlers):
        def callback(ch, method, properties, body):
            task = json.loads(body)
            handler = handlers.get(task['task'])

            if handler:
                try:
                    result = handler(*task['args'])
                    print(f"Task {task['id']} completed: {result}")
                except Exception as e:
                    print(f"Task {task['id']} failed: {e}")

            ch.basic_ack(delivery_tag=method.delivery_tag)

        self.channel.basic_consume(
            queue='tasks',
            on_message_callback=callback
        )
        self.channel.start_consuming()

# Использование
queue = TaskQueue()

# Producer
queue.enqueue('send_email', ['user@example.com', 'Hello!'])
queue.enqueue('process_image', ['/path/to/image.jpg'])

# Worker
def send_email(to, message):
    print(f"Sending email to {to}")
    return True

def process_image(path):
    print(f"Processing image {path}")
    return True

queue.worker({
    'send_email': send_email,
    'process_image': process_image
})
```

### 2. Request-Reply

```python
import uuid

class RPCClient:
    def __init__(self):
        self.connection = pika.BlockingConnection()
        self.channel = self.connection.channel()

        # Создаём callback queue для ответов
        result = self.channel.queue_declare(queue='', exclusive=True)
        self.callback_queue = result.method.queue

        self.responses = {}

        self.channel.basic_consume(
            queue=self.callback_queue,
            on_message_callback=self._on_response,
            auto_ack=True
        )

    def _on_response(self, ch, method, props, body):
        self.responses[props.correlation_id] = json.loads(body)

    def call(self, method_name, params, timeout=30):
        correlation_id = str(uuid.uuid4())

        self.channel.basic_publish(
            exchange='',
            routing_key='rpc_queue',
            body=json.dumps({'method': method_name, 'params': params}),
            properties=pika.BasicProperties(
                reply_to=self.callback_queue,
                correlation_id=correlation_id
            )
        )

        # Ожидание ответа
        start = time.time()
        while correlation_id not in self.responses:
            self.connection.process_data_events()
            if time.time() - start > timeout:
                raise TimeoutError("RPC call timed out")

        return self.responses.pop(correlation_id)

# RPC Server
def rpc_server():
    connection = pika.BlockingConnection()
    channel = connection.channel()
    channel.queue_declare(queue='rpc_queue')

    def on_request(ch, method, props, body):
        request = json.loads(body)

        # Обработка запроса
        result = handle_rpc(request['method'], request['params'])

        # Отправка ответа
        ch.basic_publish(
            exchange='',
            routing_key=props.reply_to,
            body=json.dumps(result),
            properties=pika.BasicProperties(
                correlation_id=props.correlation_id
            )
        )
        ch.basic_ack(delivery_tag=method.delivery_tag)

    channel.basic_consume(queue='rpc_queue', on_message_callback=on_request)
    channel.start_consuming()
```

### 3. Competing Consumers

```python
# Несколько consumers обрабатывают одну очередь
# Каждое сообщение обрабатывается только одним consumer'ом

# Worker 1, 2, 3... запускаются как отдельные процессы
def start_worker(worker_id):
    connection = pika.BlockingConnection()
    channel = connection.channel()
    channel.queue_declare(queue='shared_queue', durable=True)
    channel.basic_qos(prefetch_count=1)  # Fair dispatch

    def callback(ch, method, properties, body):
        print(f"Worker {worker_id} processing: {body}")
        time.sleep(random.uniform(0.5, 2))  # Симуляция работы
        ch.basic_ack(delivery_tag=method.delivery_tag)

    channel.basic_consume(queue='shared_queue', on_message_callback=callback)
    print(f"Worker {worker_id} started")
    channel.start_consuming()

# Запуск нескольких workers
# python worker.py 1
# python worker.py 2
# python worker.py 3
```

---

## Best Practices

### 1. Идемпотентность

```python
# Всегда делайте обработчики идемпотентными
class IdempotentProcessor:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.processed_key = "processed_messages"

    def process(self, message):
        message_id = message['id']

        # Атомарная проверка и установка
        if not self.redis.setnx(f"{self.processed_key}:{message_id}", "1"):
            print(f"Already processed: {message_id}")
            return

        # Устанавливаем TTL для очистки
        self.redis.expire(f"{self.processed_key}:{message_id}", 86400)

        # Обработка
        self._do_process(message)
```

### 2. Monitoring и Alerting

```python
from prometheus_client import Counter, Histogram, Gauge

# Метрики
messages_processed = Counter(
    'messages_processed_total',
    'Total messages processed',
    ['queue', 'status']
)

message_processing_time = Histogram(
    'message_processing_seconds',
    'Time spent processing message',
    ['queue']
)

queue_depth = Gauge(
    'queue_depth',
    'Current queue depth',
    ['queue']
)

def process_with_metrics(queue_name, message, handler):
    with message_processing_time.labels(queue=queue_name).time():
        try:
            handler(message)
            messages_processed.labels(queue=queue_name, status='success').inc()
        except Exception as e:
            messages_processed.labels(queue=queue_name, status='error').inc()
            raise
```

### 3. Graceful Shutdown

```python
import signal
import sys

class GracefulWorker:
    def __init__(self):
        self.should_stop = False
        signal.signal(signal.SIGTERM, self._signal_handler)
        signal.signal(signal.SIGINT, self._signal_handler)

    def _signal_handler(self, signum, frame):
        print("Received shutdown signal")
        self.should_stop = True

    def run(self):
        connection = pika.BlockingConnection()
        channel = connection.channel()
        channel.queue_declare(queue='my_queue')

        for method, properties, body in channel.consume('my_queue'):
            if self.should_stop:
                # Возвращаем сообщение в очередь
                channel.basic_nack(delivery_tag=method.delivery_tag, requeue=True)
                break

            try:
                process(body)
                channel.basic_ack(delivery_tag=method.delivery_tag)
            except Exception:
                channel.basic_nack(delivery_tag=method.delivery_tag, requeue=True)

        connection.close()
        print("Worker stopped gracefully")
```

### 4. Backpressure

```python
# Контроль нагрузки через prefetch
channel.basic_qos(prefetch_count=10)  # Не более 10 сообщений одновременно

# Rate limiting
from ratelimit import limits, sleep_and_retry

@sleep_and_retry
@limits(calls=100, period=60)  # 100 сообщений в минуту
def process_message(message):
    # Обработка
    pass
```

---

## Типичные ошибки

### 1. Потеря сообщений при ошибках

```python
# ❌ Неправильно: ACK до обработки
def callback(ch, method, properties, body):
    ch.basic_ack(delivery_tag=method.delivery_tag)  # ACK сразу
    process(body)  # Если упадёт — сообщение потеряно

# ✅ Правильно: ACK после обработки
def callback(ch, method, properties, body):
    try:
        process(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)
```

### 2. Бесконечный retry loop

```python
# ❌ Неправильно: requeue без ограничений
def callback(ch, method, properties, body):
    try:
        process(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)
        # Сообщение будет крутиться бесконечно!

# ✅ Правильно: ограничение retry + DLQ
def callback(ch, method, properties, body):
    retry_count = get_retry_count(properties)

    try:
        process(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception:
        if retry_count < MAX_RETRIES:
            republish_with_delay(ch, body, retry_count + 1)
            ch.basic_ack(delivery_tag=method.delivery_tag)
        else:
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
            # Сообщение уйдёт в DLQ
```

### 3. Отсутствие таймаутов

```python
# ❌ Неправильно: обработка может зависнуть
def process(message):
    response = requests.get(message['url'])  # Может зависнуть навсегда
    return response.text

# ✅ Правильно: всегда используйте таймауты
def process(message):
    try:
        response = requests.get(message['url'], timeout=30)
        return response.text
    except requests.Timeout:
        raise ProcessingError("External service timeout")
```

### 4. Огромные сообщения

```python
# ❌ Неправильно: большие данные в сообщении
message = {
    'type': 'process_file',
    'content': base64.b64encode(large_file_bytes).decode()  # 100MB!
}

# ✅ Правильно: ссылка на хранилище
message = {
    'type': 'process_file',
    's3_key': 'uploads/file-123.pdf'  # Только ссылка
}
```

---

## Заключение

Message Queues — фундаментальный компонент распределённых систем. Ключевые моменты:

1. **Выбор брокера** зависит от требований: RabbitMQ для сложной маршрутизации, Kafka для event streaming, SQS для простых задач
2. **Гарантии доставки** должны соответствовать бизнес-требованиям
3. **Идемпотентность** обязательна при at-least-once
4. **DLQ** необходим для обработки ошибок
5. **Мониторинг** критически важен для production

Правильно спроектированная система очередей обеспечивает надёжность, масштабируемость и отказоустойчивость всего приложения.
