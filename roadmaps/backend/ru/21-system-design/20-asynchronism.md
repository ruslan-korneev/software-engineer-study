# Асинхронность в системном дизайне

## Что такое асинхронная обработка

**Асинхронная обработка** — это модель выполнения задач, при которой инициатор запроса не ожидает немедленного результата. Вместо блокирования на время выполнения операции, система принимает запрос, подтверждает его получение и обрабатывает в фоновом режиме.

### Зачем нужна асинхронность

```
Проблема синхронной обработки:

User Request → [Web Server] → [Database] → [External API] → Response
                   ↓              ↓              ↓
              Блокировка     Блокировка     Блокировка
                   ↓              ↓              ↓
              Время ожидания = 50ms + 200ms + 500ms = 750ms
```

**Основные причины использования асинхронности:**

1. **Ресурсоёмкие операции** — обработка видео, генерация отчётов, ML-инференс
2. **Внешние зависимости** — интеграции с медленными API, отправка email/SMS
3. **Пиковые нагрузки** — сглаживание всплесков трафика
4. **Отложенные задачи** — scheduled jobs, напоминания, retry-механизмы
5. **Событийные системы** — реакция на изменения в реальном времени

---

## Синхронная vs Асинхронная коммуникация

### Синхронная коммуникация

```
┌──────────┐    Request     ┌──────────┐
│  Client  │ ─────────────► │  Server  │
│          │                │          │
│  Waiting │ ◄───────────── │ Process  │
│    ...   │    Response    │          │
└──────────┘                └──────────┘

Характеристики:
- Клиент блокируется до получения ответа
- Тесная связь между компонентами
- Простая модель программирования
- Сложно масштабировать
```

**Примеры:** REST API, gRPC, SQL-запросы

### Асинхронная коммуникация

```
┌──────────┐   Publish      ┌─────────────┐   Consume     ┌──────────┐
│ Producer │ ──────────────►│   Message   │──────────────►│ Consumer │
│          │                │    Queue    │               │          │
│ Continue │   ACK (fast)   │             │               │ Process  │
│  work    │◄───────────────│             │               │          │
└──────────┘                └─────────────┘               └──────────┘

Характеристики:
- Продюсер не ждёт обработки
- Слабая связь (loose coupling)
- Буферизация сообщений
- Независимое масштабирование
```

**Примеры:** RabbitMQ, Kafka, SQS, Redis Streams

### Сравнительная таблица

| Аспект | Синхронная | Асинхронная |
|--------|------------|-------------|
| Latency для клиента | Высокий | Низкий (только ACK) |
| Coupling | Tight | Loose |
| Error handling | Прямой | Через DLQ, retry |
| Debugging | Простой | Сложнее (distributed tracing) |
| Гарантия доставки | Немедленная | Зависит от настроек |
| Масштабирование | Вертикальное | Горизонтальное |

---

## Преимущества асинхронности

### 1. Decoupling (Разделение)

```
Синхронная система (сильная связь):
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Service │────►│ Service │────►│ Service │
│    A    │     │    B    │     │    C    │
└─────────┘     └─────────┘     └─────────┘
    │               │               │
    └───────────────┴───────────────┘
    Если B падает — всё ломается

Асинхронная система (слабая связь):
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Service │────►│  Queue  │────►│ Service │
│    A    │     │         │     │    B    │
└─────────┘     └─────────┘     └─────────┘
    │               │
    │    Сообщения сохраняются в очереди
    │    пока B недоступен
    ▼
  Продолжает работать
```

**Преимущества:**
- Независимое развёртывание сервисов
- Разные технологические стеки
- Изолированные failure domains

### 2. Масштабируемость

```python
# Пример: масштабирование обработки заказов

# Без очередей — все workers должны быть ready
def process_order_sync(order):
    validate(order)           # 10ms
    charge_payment(order)     # 500ms (external API)
    update_inventory(order)   # 50ms
    send_notification(order)  # 200ms
    return "OK"               # Total: 760ms, 1.3 RPS per worker

# С очередями — каждый этап масштабируется отдельно
def process_order_async(order):
    queue.publish("order.created", order)
    return "Accepted"         # 5ms, 200 RPS per worker

# Consumers масштабируются независимо:
# - 5 workers для payment (bottleneck)
# - 2 workers для inventory
# - 10 workers для notifications (высокий объём)
```

**Формула throughput:**
```
Синхронный: Throughput = Workers / Latency
Асинхронный: Throughput = min(Producer_Rate, Consumer_Rate)
             где Consumer_Rate = Consumers × Consumer_Speed
```

### 3. Устойчивость к сбоям (Resilience)

```
Сценарий: Payment Service недоступен

Синхронная обработка:
┌───────┐      ┌─────────┐
│ Order │─────►│ Payment │ ✗ 503 Error
│  API  │      │ Service │
└───────┘      └─────────┘
    │
    ▼
  HTTP 500 для пользователя

Асинхронная обработка:
┌───────┐      ┌─────────┐      ┌─────────┐
│ Order │─────►│  Queue  │─────►│ Payment │ ✗
│  API  │      │         │      │ Service │
└───────┘      └─────────┘      └─────────┘
    │              │
    ▼              ▼
  HTTP 202       Сообщение остаётся в очереди
  Accepted       → Retry через 5 сек
                 → Success на 3-й попытке
```

### 4. Сглаживание пиковых нагрузок (Load Leveling)

```
Трафик без очередей:

Requests  │    ╭───╮
  500 ────│────│ ▲ │──────── Capacity: 100 RPS
          │   ╭┘   ╰╮        → 400 requests lost!
  100 ────│───┤     ├───
          │   │     │
        ──┴───┴─────┴─────► Time
           Peak Hour

Трафик с очередями:

Requests  │    ╭───╮
  500 ────│────│   │──────── Incoming
          │   ╭┘   ╰╮
          │   │     │
  100 ────│───┼─────┼─────── Processing (constant)
          │   │░░░░░│        ░ = Queued messages
        ──┴───┴─────┴─────► Time
           Peak Hour

Queue absorbs the spike, consumers process at steady rate
```

**Пример: Black Friday**
```python
# Конфигурация для пиковых нагрузок
queue_config = {
    "max_length": 1_000_000,        # Буфер на миллион сообщений
    "consumer_prefetch": 100,        # Batch processing
    "message_ttl": 3600,            # 1 час на обработку
}

# В обычное время: 10 consumers
# В Black Friday: автоскейлинг до 100 consumers
# Очередь сглаживает разницу между входящей и исходящей скоростью
```

---

## Message Queues

### Основные концепции

```
                    ┌─────────────────────────────────────┐
                    │           Message Broker            │
                    │                                     │
┌──────────┐        │  ┌─────────┐    ┌─────────────┐    │        ┌──────────┐
│ Producer │───────►│  │Exchange │───►│    Queue    │────│───────►│ Consumer │
│   (P1)   │  msg   │  │         │    │  (orders)   │    │  msg   │   (C1)   │
└──────────┘        │  └─────────┘    └─────────────┘    │        └──────────┘
                    │       │                            │
┌──────────┐        │       │         ┌─────────────┐    │        ┌──────────┐
│ Producer │───────►│       └────────►│    Queue    │────│───────►│ Consumer │
│   (P2)   │        │                 │  (emails)   │    │        │   (C2)   │
└──────────┘        │                 └─────────────┘    │        └──────────┘
                    │                                     │
                    └─────────────────────────────────────┘

Компоненты:
- Producer: генерирует сообщения
- Consumer: обрабатывает сообщения
- Exchange: маршрутизация (routing)
- Queue: хранилище сообщений (buffer)
- Topic: категория/тема сообщений
- Binding: связь exchange → queue
```

### Структура сообщения

```json
{
  "id": "msg-uuid-12345",
  "timestamp": "2024-01-15T10:30:00Z",
  "type": "order.created",
  "version": "1.0",
  "source": "order-service",
  "correlation_id": "request-uuid-67890",
  "headers": {
    "content-type": "application/json",
    "retry-count": 0,
    "trace-id": "abc123"
  },
  "payload": {
    "order_id": "ORD-001",
    "customer_id": "CUST-123",
    "total": 99.99
  }
}
```

---

### RabbitMQ

**AMQP-брокер** с богатыми возможностями маршрутизации.

```
Архитектура RabbitMQ:

Producer ─── Exchange ─── Binding ─── Queue ─── Consumer

Exchange Types:
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  Direct Exchange:     routing_key = "order"               │
│  ┌─────────┐          ┌────────────┐                      │
│  │ Direct  │─────────►│ order_queue│  (exact match)       │
│  └─────────┘          └────────────┘                      │
│                                                            │
│  Fanout Exchange:     no routing_key needed               │
│  ┌─────────┐──────────►│ queue_1   │                      │
│  │ Fanout  │──────────►│ queue_2   │  (broadcast)         │
│  └─────────┘──────────►│ queue_3   │                      │
│                                                            │
│  Topic Exchange:      pattern matching                     │
│  ┌─────────┐          "order.*" ──► │ all_orders │        │
│  │  Topic  │          "order.created" ──► │ new_orders │  │
│  └─────────┘          "*.payment" ──► │ payments │        │
│                                                            │
│  Headers Exchange:    match by headers                     │
│  ┌─────────┐          {"format": "pdf"} ──► │ pdf_queue │ │
│  │ Headers │                                               │
│  └─────────┘                                               │
└────────────────────────────────────────────────────────────┘
```

**Пример на Python (pika):**

```python
import pika
import json

# Connection
connection = pika.BlockingConnection(
    pika.ConnectionParameters(
        host='localhost',
        credentials=pika.PlainCredentials('guest', 'guest')
    )
)
channel = connection.channel()

# Declare exchange and queue
channel.exchange_declare(
    exchange='orders',
    exchange_type='topic',
    durable=True
)

channel.queue_declare(
    queue='order_processing',
    durable=True,
    arguments={
        'x-message-ttl': 86400000,      # 24h TTL
        'x-dead-letter-exchange': 'dlx',
        'x-max-length': 100000
    }
)

channel.queue_bind(
    exchange='orders',
    queue='order_processing',
    routing_key='order.created'
)

# Producer
def publish_order(order):
    channel.basic_publish(
        exchange='orders',
        routing_key='order.created',
        body=json.dumps(order),
        properties=pika.BasicProperties(
            delivery_mode=2,  # Persistent
            content_type='application/json',
            correlation_id=str(uuid.uuid4()),
            headers={'version': '1.0'}
        )
    )

# Consumer
def callback(ch, method, properties, body):
    try:
        order = json.loads(body)
        process_order(order)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception as e:
        # Reject with requeue=False → goes to DLQ
        ch.basic_nack(
            delivery_tag=method.delivery_tag,
            requeue=False
        )

channel.basic_qos(prefetch_count=10)
channel.basic_consume(
    queue='order_processing',
    on_message_callback=callback
)
channel.start_consuming()
```

**Преимущества RabbitMQ:**
- Гибкая маршрутизация (exchanges)
- Подтверждения (acks/nacks)
- Priority queues
- Delayed messages (plugin)
- Management UI

**Недостатки:**
- Не предназначен для хранения (retention)
- Сложнее масштабировать горизонтально
- Ниже throughput чем у Kafka

---

### Apache Kafka

**Распределённый лог событий** для потоковой обработки.

```
Архитектура Kafka:

┌──────────────────────────────────────────────────────────────────┐
│                        Kafka Cluster                              │
│                                                                   │
│  Topic: orders (partitions=3, replication-factor=2)              │
│                                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│  │ Partition 0 │  │ Partition 1 │  │ Partition 2 │               │
│  │ [0][1][2][3]│  │ [0][1][2]   │  │ [0][1][2][3]│               │
│  │  Leader:B1  │  │  Leader:B2  │  │  Leader:B3  │               │
│  │  Replica:B2 │  │  Replica:B3 │  │  Replica:B1 │               │
│  └─────────────┘  └─────────────┘  └─────────────┘               │
│        ▲                ▲                ▲                        │
│        │                │                │                        │
│        │     Producers distribute by key (hash)                  │
│        │                                                          │
└────────┼──────────────────────────────────────────────────────────┘
         │
    ┌────┴────┐
    │Producer │  msg {key: "user-123", value: {...}}
    └─────────┘  → hash("user-123") % 3 = partition 1


Consumer Group:

┌───────────────────────────────────────────────────────┐
│              Consumer Group: order-processors         │
│                                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │Consumer-1│  │Consumer-2│  │Consumer-3│           │
│  │  P0, P1  │  │    P2    │  │  (idle)  │           │
│  └──────────┘  └──────────┘  └──────────┘           │
│                                                       │
│  Каждая partition — только один consumer в группе    │
└───────────────────────────────────────────────────────┘
```

**Ключевые концепции:**

```python
# Offset — позиция в partition
Partition: [msg0][msg1][msg2][msg3][msg4][msg5]
                              ↑
                         Consumer offset = 3
                         (следующее сообщение для чтения)

# Consumer Groups — параллелизм
Group A: читает все партиции → real-time processing
Group B: читает все партиции → analytics (независимо)

# Retention — хранение
retention.ms = 604800000 (7 дней)
retention.bytes = 10737418240 (10 GB)
```

**Пример на Python (kafka-python):**

```python
from kafka import KafkaProducer, KafkaConsumer
from kafka.errors import KafkaError
import json

# Producer с идемпотентностью
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    key_serializer=lambda k: k.encode('utf-8') if k else None,
    acks='all',              # Wait for all replicas
    retries=3,
    enable_idempotence=True, # Exactly-once semantics
    compression_type='gzip'
)

def send_order(order):
    future = producer.send(
        topic='orders',
        key=order['customer_id'],  # Partition by customer
        value=order,
        headers=[
            ('event_type', b'order.created'),
            ('version', b'1')
        ]
    )

    try:
        record_metadata = future.get(timeout=10)
        print(f"Sent to partition {record_metadata.partition} "
              f"at offset {record_metadata.offset}")
    except KafkaError as e:
        print(f"Failed to send: {e}")

# Consumer
consumer = KafkaConsumer(
    'orders',
    bootstrap_servers=['localhost:9092'],
    group_id='order-processors',
    auto_offset_reset='earliest',  # Start from beginning if no offset
    enable_auto_commit=False,      # Manual commit for at-least-once
    value_deserializer=lambda v: json.loads(v.decode('utf-8'))
)

for message in consumer:
    try:
        order = message.value
        print(f"Processing order from partition {message.partition}, "
              f"offset {message.offset}")

        process_order(order)

        # Commit after successful processing
        consumer.commit()

    except Exception as e:
        print(f"Error processing: {e}")
        # Don't commit — message will be reprocessed
```

**Kafka Streams (обработка потоков):**

```python
# Концептуальный пример stream processing
from kafka import KafkaConsumer, KafkaProducer

# Stream: orders → enriched_orders → notifications

# Step 1: Read orders
orders = consumer.subscribe('orders')

# Step 2: Enrich (join with customer data)
for order in orders:
    customer = get_customer(order['customer_id'])
    enriched = {**order, 'customer': customer}
    producer.send('enriched_orders', value=enriched)

# Step 3: Generate notifications
for enriched in enriched_consumer:
    notification = create_notification(enriched)
    producer.send('notifications', value=notification)
```

**Kafka vs RabbitMQ:**

| Аспект | Kafka | RabbitMQ |
|--------|-------|----------|
| Модель | Log (append-only) | Queue (FIFO, delete on ack) |
| Retention | Хранит (дни/недели) | Удаляет после ack |
| Replay | Можно перечитать | Нельзя |
| Throughput | Миллионы msg/sec | Тысячи msg/sec |
| Ordering | В пределах partition | В пределах queue |
| Routing | Примитивная (topics) | Гибкая (exchanges) |
| Use case | Event sourcing, logs | Task queues, RPC |

---

### Amazon SQS

**Полностью управляемый** сервис очередей AWS.

```
SQS Types:

Standard Queue:
┌────────────────────────────────────────────┐
│  • At-least-once delivery                  │
│  • Best-effort ordering                    │
│  • Nearly unlimited throughput             │
│  • May have duplicates                     │
│                                            │
│  Use: High-volume, order not critical      │
└────────────────────────────────────────────┘

FIFO Queue:
┌────────────────────────────────────────────┐
│  • Exactly-once processing                 │
│  • Strict ordering (within message group)  │
│  • 300 msg/sec (batch: 3000)              │
│  • Deduplication (5 min window)           │
│                                            │
│  Use: Financial transactions, ordering     │
└────────────────────────────────────────────┘
```

**Пример с boto3:**

```python
import boto3
import json

sqs = boto3.client('sqs', region_name='us-east-1')
queue_url = 'https://sqs.us-east-1.amazonaws.com/123456789/my-queue'

# Send message
def send_message(order):
    response = sqs.send_message(
        QueueUrl=queue_url,
        MessageBody=json.dumps(order),
        MessageAttributes={
            'EventType': {
                'DataType': 'String',
                'StringValue': 'order.created'
            }
        },
        # For FIFO queues:
        # MessageGroupId='customer-123',
        # MessageDeduplicationId='order-uuid'
    )
    return response['MessageId']

# Receive and process
def process_messages():
    while True:
        response = sqs.receive_message(
            QueueUrl=queue_url,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=20,  # Long polling
            VisibilityTimeout=300,  # 5 min to process
            MessageAttributeNames=['All']
        )

        for message in response.get('Messages', []):
            try:
                order = json.loads(message['Body'])
                process_order(order)

                # Delete on success
                sqs.delete_message(
                    QueueUrl=queue_url,
                    ReceiptHandle=message['ReceiptHandle']
                )
            except Exception as e:
                # Message returns to queue after visibility timeout
                print(f"Error: {e}")

# Dead Letter Queue setup (via AWS Console or IaC)
dlq_config = {
    "RedrivePolicy": {
        "deadLetterTargetArn": "arn:aws:sqs:...:my-dlq",
        "maxReceiveCount": 3
    }
}
```

**SQS + Lambda:**

```python
# Lambda function triggered by SQS
def lambda_handler(event, context):
    for record in event['Records']:
        body = json.loads(record['body'])

        try:
            result = process_order(body)
            # Success → Lambda automatically deletes message
        except Exception as e:
            # Raise → message returns to queue for retry
            raise e

    return {'statusCode': 200}
```

---

### Redis Streams

**Lightweight log structure** в Redis для потоковой обработки.

```
Redis Stream Structure:

STREAM: orders
┌────────────────────────────────────────────────────────────┐
│ Entry ID          │ Fields                                 │
├───────────────────┼────────────────────────────────────────┤
│ 1699999999000-0   │ {order_id: "001", total: "99.99"}     │
│ 1699999999001-0   │ {order_id: "002", total: "149.99"}    │
│ 1699999999002-0   │ {order_id: "003", total: "29.99"}     │
└───────────────────┴────────────────────────────────────────┘
         ↑
    Timestamp-based ID (milliseconds-sequence)

Consumer Groups:
┌────────────────────────────────────────────────────────────┐
│ Group: order-processors                                    │
│ Last Delivered ID: 1699999999001-0                        │
│                                                            │
│ Consumer: worker-1                                         │
│   Pending: [1699999999000-0] (processing)                 │
│                                                            │
│ Consumer: worker-2                                         │
│   Pending: [1699999999001-0] (processing)                 │
└────────────────────────────────────────────────────────────┘
```

**Пример:**

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Producer: add to stream
def publish_order(order):
    entry_id = r.xadd(
        'orders',
        order,
        maxlen=100000,  # Trim to last 100k entries
        approximate=True
    )
    return entry_id

# Create consumer group
try:
    r.xgroup_create('orders', 'order-processors', id='0', mkstream=True)
except redis.exceptions.ResponseError:
    pass  # Group already exists

# Consumer
def consume_orders(consumer_name):
    while True:
        # Read new messages
        entries = r.xreadgroup(
            groupname='order-processors',
            consumername=consumer_name,
            streams={'orders': '>'},  # '>' = new messages only
            count=10,
            block=5000  # Block for 5 sec
        )

        for stream, messages in entries:
            for msg_id, fields in messages:
                try:
                    process_order(fields)
                    # Acknowledge
                    r.xack('orders', 'order-processors', msg_id)
                except Exception as e:
                    print(f"Error: {e}")
                    # Message remains pending

# Claim stale messages (from dead consumers)
def recover_pending():
    # Get pending messages older than 60 seconds
    pending = r.xpending_range(
        'orders',
        'order-processors',
        min='-',
        max='+',
        count=100
    )

    stale_ids = [
        entry['message_id']
        for entry in pending
        if entry['time_since_delivered'] > 60000
    ]

    if stale_ids:
        # Claim for this consumer
        r.xclaim(
            'orders',
            'order-processors',
            'recovery-worker',
            min_idle_time=60000,
            message_ids=stale_ids
        )
```

**Redis Streams vs Kafka:**

| Аспект | Redis Streams | Kafka |
|--------|--------------|-------|
| Persistence | RDB/AOF (optional) | Always persisted |
| Throughput | 100k+ msg/sec | Millions msg/sec |
| Complexity | Simple setup | Complex cluster |
| Use case | Small-medium scale | Large-scale streaming |
| Memory | In-memory | Disk-based |

---

## Паттерны обмена сообщениями

### 1. Point-to-Point (Queue)

```
Один consumer обрабатывает каждое сообщение

Producer ─────► Queue ─────► Consumer
              [m1][m2][m3]

Пример: Task queues (обработка заказов)

┌────────┐     ┌───────────┐     ┌────────┐
│  Web   │────►│  Orders   │────►│Worker 1│ → m1
│ Server │     │   Queue   │────►│Worker 2│ → m2
└────────┘     └───────────┘────►│Worker 3│ → m3
                                 └────────┘

Каждое сообщение обрабатывается ТОЛЬКО ОДНИМ worker
```

### 2. Publish-Subscribe (Pub/Sub)

```
Все subscribers получают копию сообщения

                    ┌──────────┐
                ───►│ Service A│ → m1
Publisher ────►    ┌┴──────────┘
           Topic   ├──────────┐
                ───►│ Service B│ → m1 (copy)
                   ├┴──────────┘
                ───►│ Service C│ → m1 (copy)
                    └──────────┘

Пример: Event broadcasting

# Order created event
event = {"type": "order.created", "order_id": "123"}

# Subscribers:
# - Inventory Service: резервирует товар
# - Payment Service: инициирует оплату
# - Notification Service: отправляет email
# - Analytics Service: логирует событие
```

### 3. Request-Reply

```
Асинхронный RPC через очереди

┌────────┐  request  ┌─────────────┐  request  ┌────────┐
│ Client │──────────►│  Request Q  │──────────►│ Server │
│        │           └─────────────┘           │        │
│        │  reply    ┌─────────────┐  reply    │        │
│        │◄──────────│  Reply Q    │◄──────────│        │
└────────┘           └─────────────┘           └────────┘

correlation_id связывает request с reply
```

```python
# Request-Reply pattern with RabbitMQ
import uuid

class RpcClient:
    def __init__(self):
        self.response = None
        self.corr_id = None

        # Create exclusive reply queue
        result = channel.queue_declare(queue='', exclusive=True)
        self.callback_queue = result.method.queue

        channel.basic_consume(
            queue=self.callback_queue,
            on_message_callback=self.on_response,
            auto_ack=True
        )

    def on_response(self, ch, method, props, body):
        if self.corr_id == props.correlation_id:
            self.response = body

    def call(self, request):
        self.response = None
        self.corr_id = str(uuid.uuid4())

        channel.basic_publish(
            exchange='',
            routing_key='rpc_queue',
            properties=pika.BasicProperties(
                reply_to=self.callback_queue,
                correlation_id=self.corr_id,
            ),
            body=json.dumps(request)
        )

        # Wait for response
        while self.response is None:
            connection.process_data_events()

        return json.loads(self.response)
```

### 4. Fan-out / Fan-in

```
Fan-out: Split work across workers
Fan-in: Aggregate results

Fan-out (распределение):
                    ┌────────────┐
               ────►│  Worker 1  │────┐
┌─────────┐        ├────────────┤    │
│ Splitter│───────►│  Worker 2  │────┼────► Results
│         │        ├────────────┤    │
└─────────┘   ────►│  Worker 3  │────┘
                    └────────────┘

Пример: Image processing pipeline

def process_image_batch(images):
    # Fan-out: distribute to workers
    tasks = []
    for image in images:
        task_id = queue.publish('image.process', image)
        tasks.append(task_id)

    # Fan-in: aggregate results
    results = []
    for task_id in tasks:
        result = wait_for_result(task_id)
        results.append(result)

    return combine_results(results)
```

### 5. Dead Letter Queue (DLQ)

```
Обработка неуспешных сообщений

┌──────────┐     ┌─────────────┐     ┌──────────┐
│ Producer │────►│ Main Queue  │────►│ Consumer │
└──────────┘     └─────────────┘     └──────────┘
                        │                  │
                        │            Fail 3x
                        │                  │
                        ▼                  ▼
                 ┌─────────────┐     ┌──────────┐
                 │ Dead Letter │◄────│  Reject  │
                 │   Queue     │     └──────────┘
                 └─────────────┘
                        │
                        ▼
                 ┌─────────────┐
                 │  DLQ Worker │  → Alert, manual review
                 └─────────────┘
```

```python
# DLQ handling example
def process_dlq():
    for message in dlq_consumer:
        error_info = {
            'original_message': message.body,
            'failed_at': message.headers.get('x-death', [{}])[0],
            'error_count': message.headers.get('x-death', [{}])[0].get('count', 0),
            'original_queue': message.headers.get('x-death', [{}])[0].get('queue')
        }

        # Log for investigation
        logger.error(f"DLQ message: {error_info}")

        # Alert operations team
        send_alert(error_info)

        # Options:
        # 1. Republish with fix
        # 2. Save to error database
        # 3. Manual intervention required

        message.ack()
```

---

## Event-Driven Architecture (EDA)

### Концепция

```
Event-Driven vs Request-Driven:

Request-Driven (команда):
┌─────────────┐    "Create Order"    ┌─────────────┐
│ Order       │─────────────────────►│  Inventory  │
│ Service     │◄─────────────────────│  Service    │
└─────────────┘    "OK / Error"      └─────────────┘
    Прямая зависимость, знает про Inventory

Event-Driven (событие):
┌─────────────┐  "OrderCreated" event  ┌───────────────┐
│ Order       │───────────────────────►│  Event Bus    │
│ Service     │                        │               │
└─────────────┘                        └───────┬───────┘
    Не знает про subscribers                   │
                           ┌───────────────────┼───────────────────┐
                           ▼                   ▼                   ▼
                    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
                    │  Inventory  │     │  Payment    │     │  Analytics  │
                    │  Service    │     │  Service    │     │  Service    │
                    └─────────────┘     └─────────────┘     └─────────────┘
                    Независимые subscribers
```

### Event Types

```
1. Domain Events (бизнес-события):
   - OrderCreated
   - PaymentReceived
   - ItemShipped

2. Integration Events (интеграционные):
   - UserRegistered → sync to CRM
   - InventoryLow → notify supplier

3. System Events (инфраструктурные):
   - ServiceHealthChanged
   - DeploymentCompleted
```

### Event Sourcing

```
Традиционный подход (state):
┌──────────────────────────────────────┐
│ Orders Table                         │
├──────────────────────────────────────┤
│ id: 1, status: "shipped", total: 100 │  ← Только текущее состояние
└──────────────────────────────────────┘

Event Sourcing (события):
┌────────────────────────────────────────────────────────┐
│ Order Events Log                                       │
├────────────────────────────────────────────────────────┤
│ 1. OrderCreated    {id: 1, items: [...], total: 100}  │
│ 2. PaymentReceived {id: 1, payment_id: "abc"}         │
│ 3. OrderShipped    {id: 1, tracking: "xyz"}           │
└────────────────────────────────────────────────────────┘
    ↓
Текущее состояние = replay всех событий
```

```python
# Event Sourcing example
class Order:
    def __init__(self, id):
        self.id = id
        self.status = None
        self.items = []
        self.events = []

    def apply_event(self, event):
        if event['type'] == 'OrderCreated':
            self.status = 'created'
            self.items = event['items']
        elif event['type'] == 'PaymentReceived':
            self.status = 'paid'
        elif event['type'] == 'OrderShipped':
            self.status = 'shipped'

        self.events.append(event)

    @classmethod
    def from_events(cls, id, events):
        order = cls(id)
        for event in events:
            order.apply_event(event)
        return order

# CQRS (Command Query Responsibility Segregation)
# Write side: events
# Read side: materialized views (projections)

class OrderProjection:
    def __init__(self, event_store):
        self.orders = {}
        self.event_store = event_store

    def rebuild(self):
        for event in self.event_store.get_all_events():
            self.handle(event)

    def handle(self, event):
        order_id = event['order_id']
        if event['type'] == 'OrderCreated':
            self.orders[order_id] = {
                'id': order_id,
                'status': 'created',
                'items': event['items']
            }
        elif event['type'] == 'OrderShipped':
            self.orders[order_id]['status'] = 'shipped'
```

---

## Task Queues

### Celery (Python)

```python
# celery_app.py
from celery import Celery

app = Celery(
    'tasks',
    broker='redis://localhost:6379/0',
    backend='redis://localhost:6379/1'
)

app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='UTC',
    enable_utc=True,
    task_acks_late=True,  # Ack after completion
    task_reject_on_worker_lost=True,
    worker_prefetch_multiplier=1,
)

# tasks.py
from celery_app import app

@app.task(bind=True, max_retries=3, default_retry_delay=60)
def process_order(self, order_id):
    try:
        order = get_order(order_id)
        charge_payment(order)
        update_inventory(order)
        send_confirmation(order)
        return {"status": "success", "order_id": order_id}
    except PaymentError as e:
        raise self.retry(exc=e, countdown=30)  # Retry in 30 sec
    except Exception as e:
        # Will go to failure handler after max_retries
        raise

@app.task
def send_email(to, subject, body):
    # Long-running task
    email_service.send(to, subject, body)

# Chains and groups
from celery import chain, group, chord

# Sequential execution
workflow = chain(
    validate_order.s(order_id),
    process_payment.s(),
    update_inventory.s(),
    send_notification.s()
)
result = workflow.apply_async()

# Parallel execution
parallel = group(
    resize_image.s(image, 'thumbnail'),
    resize_image.s(image, 'medium'),
    resize_image.s(image, 'large')
)
results = parallel.apply_async()

# Parallel with callback
chord_workflow = chord(
    [process_item.s(item) for item in items],
    aggregate_results.s()
)
```

```bash
# Start Celery worker
celery -A celery_app worker --loglevel=info --concurrency=4

# Start Celery beat (scheduler)
celery -A celery_app beat --loglevel=info
```

**Periodic tasks:**

```python
from celery.schedules import crontab

app.conf.beat_schedule = {
    'cleanup-every-hour': {
        'task': 'tasks.cleanup_old_records',
        'schedule': crontab(minute=0),  # Every hour
    },
    'send-daily-report': {
        'task': 'tasks.send_daily_report',
        'schedule': crontab(hour=9, minute=0),  # 9 AM daily
    },
    'check-inventory': {
        'task': 'tasks.check_inventory_levels',
        'schedule': 300.0,  # Every 5 minutes
    },
}
```

### Sidekiq (Ruby)

```ruby
# worker.rb
class OrderProcessor
  include Sidekiq::Worker

  sidekiq_options queue: 'critical', retry: 5

  def perform(order_id)
    order = Order.find(order_id)
    PaymentService.charge(order)
    InventoryService.update(order)
    NotificationService.send_confirmation(order)
  end
end

# Enqueue
OrderProcessor.perform_async(order_id)

# Scheduled job
OrderProcessor.perform_in(5.minutes, order_id)
OrderProcessor.perform_at(Time.now + 1.day, order_id)
```

---

## Webhooks и Callbacks

### Webhook Pattern

```
┌───────────────┐      Event occurs      ┌───────────────┐
│   Provider    │ ─────────────────────► │   Consumer    │
│  (Stripe,     │   HTTP POST callback   │   (Your App)  │
│   GitHub)     │                        │               │
└───────────────┘                        └───────────────┘

Webhook = Provider initiated HTTP request to your endpoint
```

**Реализация webhook-приёмника:**

```python
from flask import Flask, request
import hmac
import hashlib

app = Flask(__name__)

@app.route('/webhooks/stripe', methods=['POST'])
def stripe_webhook():
    payload = request.get_data()
    sig_header = request.headers.get('Stripe-Signature')

    # Verify signature
    if not verify_stripe_signature(payload, sig_header):
        return 'Invalid signature', 401

    event = json.loads(payload)

    # Idempotency: check if already processed
    if is_processed(event['id']):
        return 'Already processed', 200

    # Handle event types
    if event['type'] == 'payment_intent.succeeded':
        handle_payment_success(event['data']['object'])
    elif event['type'] == 'payment_intent.failed':
        handle_payment_failure(event['data']['object'])

    # Mark as processed
    mark_processed(event['id'])

    return 'OK', 200

def verify_stripe_signature(payload, sig_header):
    secret = os.environ['STRIPE_WEBHOOK_SECRET']

    expected_sig = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(expected_sig, sig_header)
```

### Callback Pattern

```python
# Async processing with callback URL

# 1. Client submits job with callback
@app.route('/api/reports', methods=['POST'])
def create_report():
    data = request.json
    callback_url = data['callback_url']

    # Enqueue async job
    job_id = report_queue.submit(
        data['params'],
        callback_url=callback_url
    )

    return {'job_id': job_id, 'status': 'processing'}, 202

# 2. Worker processes and calls back
def process_report(params, callback_url):
    try:
        result = generate_report(params)

        # Callback with result
        requests.post(callback_url, json={
            'status': 'success',
            'result_url': result.url
        }, timeout=10)

    except Exception as e:
        requests.post(callback_url, json={
            'status': 'failed',
            'error': str(e)
        }, timeout=10)

# 3. Client receives callback
@app.route('/callbacks/report', methods=['POST'])
def report_callback():
    data = request.json
    if data['status'] == 'success':
        process_completed_report(data['result_url'])
    else:
        handle_report_error(data['error'])
    return 'OK', 200
```

---

## Гарантии доставки

### At-Most-Once

```
Сообщение доставляется максимум один раз (может потеряться)

Producer ───send──► Broker ───deliver──► Consumer
              │                              │
              │  No ack required             │  May lose if crash
              ▼                              ▼
         "Fire and forget"              "Best effort"

Use case: Metrics, logs (потеря некритична)

# Example: UDP-style messaging
producer.send(message, acks=0)  # Don't wait for acknowledgment
```

### At-Least-Once

```
Сообщение доставляется минимум один раз (могут быть дубликаты)

Producer ───send──► Broker ───deliver──► Consumer
    │         │                    │         │
    │         ▼                    ▼         │
    │      Store                 Process     │
    │         │                    │         │
    └──wait───┤                    ├───ack───┘
              ▼                    ▼
         Retry if            Delete message
         no ack              (or crash → redeliver)

Use case: Most business operations (с idempotency)

# Example
while True:
    message = consumer.receive()
    try:
        process(message)
        consumer.ack(message)  # Only ack after success
    except:
        # Message will be redelivered
        pass
```

### Exactly-Once

```
Сообщение обрабатывается ровно один раз (сложно достичь)

Producer ───send──► Broker ───deliver──► Consumer
    │         │         │          │         │
    │    Transaction    │    Transaction     │
    │    + Idempotent   │    + Idempotent    │
    │         │         │          │         │
    ▼         ▼         ▼          ▼         ▼
 Dedup ID   Commit    Replay     Dedup    Process

Use case: Financial transactions, critical operations

Strategies:
1. Idempotency keys (client-generated)
2. Transactional outbox pattern
3. Kafka transactions + idempotent producer
```

```python
# Idempotent processing
def process_payment(payment_id, amount):
    # Check if already processed
    if redis.exists(f"processed:{payment_id}"):
        return {"status": "already_processed"}

    # Process in transaction
    with db.transaction():
        # Double-check in DB
        if Payment.exists(payment_id):
            return {"status": "already_processed"}

        # Process
        result = charge_card(amount)

        # Record
        Payment.create(id=payment_id, result=result)

    # Mark as processed (for fast lookup)
    redis.setex(f"processed:{payment_id}", 86400, "1")

    return {"status": "success"}
```

**Transactional Outbox Pattern:**

```
┌──────────────────────────────────────────────────┐
│                   Database                        │
│                                                   │
│  ┌─────────────┐      ┌─────────────────────┐   │
│  │   Orders    │      │    Outbox Table     │   │
│  ├─────────────┤      ├─────────────────────┤   │
│  │ id: 1       │      │ id: 1               │   │
│  │ status: new │      │ event: OrderCreated │   │
│  │ ...         │      │ payload: {...}      │   │
│  └─────────────┘      │ published: false    │   │
│                       └─────────────────────┘   │
│            ▲               │                     │
│            │               │                     │
│        TRANSACTION         │                     │
│            │               ▼                     │
└────────────┼──────────────────────────────────────┘
             │
             │  Outbox Publisher (separate process)
             │
             ▼
     ┌───────────────┐
     │ Message Broker│
     └───────────────┘
```

```python
# Transactional outbox implementation
from sqlalchemy import Column, Integer, String, Boolean, JSON

class OutboxEvent(Base):
    __tablename__ = 'outbox_events'

    id = Column(Integer, primary_key=True)
    aggregate_type = Column(String)
    aggregate_id = Column(String)
    event_type = Column(String)
    payload = Column(JSON)
    published = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow)

def create_order(order_data):
    with db.transaction():
        # Create order
        order = Order(**order_data)
        db.add(order)

        # Create outbox event (same transaction!)
        event = OutboxEvent(
            aggregate_type='Order',
            aggregate_id=str(order.id),
            event_type='OrderCreated',
            payload={'order_id': order.id, **order_data}
        )
        db.add(event)

        db.commit()

    return order

# Outbox publisher (separate process)
def publish_outbox_events():
    while True:
        events = OutboxEvent.query.filter_by(published=False).limit(100)

        for event in events:
            try:
                message_broker.publish(
                    topic=f"{event.aggregate_type}.{event.event_type}",
                    message=event.payload,
                    headers={'event_id': str(event.id)}
                )
                event.published = True
                db.commit()
            except Exception as e:
                logger.error(f"Failed to publish: {e}")

        time.sleep(1)
```

---

## Проблемы и решения

### 1. Ordering (Упорядоченность)

```
Проблема: Сообщения могут обрабатываться не по порядку

Producer: Order1 → Order2 → Order3
Consumer1: Order1, Order3
Consumer2: Order2
Результат: Order3 обработан раньше Order2

Решения:

1. Single partition/queue per entity:
   └── Все сообщения для user-123 → partition-X
   └── Один consumer на partition

2. Sequence numbers:
   └── {seq: 1, ...} → {seq: 2, ...} → {seq: 3, ...}
   └── Consumer buffers out-of-order messages

3. Accept eventual consistency:
   └── Система eventually consistent
   └── Conflict resolution (last-write-wins, merge)
```

```python
# Ordering with partition key
def send_order_event(order):
    # All events for same customer go to same partition
    producer.send(
        topic='orders',
        key=order['customer_id'],  # Partition key
        value=order
    )

# Kafka: messages with same key → same partition → ordered
```

### 2. Idempotency (Идемпотентность)

```
Проблема: Одно сообщение может обработаться несколько раз

Network issue → Retry → Duplicate processing

Решения:

1. Idempotency key (client-generated):
   request: {idempotency_key: "abc123", ...}
   server: if processed(abc123) → return cached result

2. Deduplication window:
   Store processed message IDs for N minutes

3. Idempotent operations:
   SET balance = 100 (idempotent)
   vs
   ADD 50 to balance (NOT idempotent)
```

```python
import hashlib

class IdempotentProcessor:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.ttl = 86400  # 24 hours

    def get_idempotency_key(self, message):
        # Use message ID or create hash
        if 'idempotency_key' in message:
            return message['idempotency_key']

        content = json.dumps(message, sort_keys=True)
        return hashlib.sha256(content.encode()).hexdigest()

    def process(self, message, handler):
        key = f"processed:{self.get_idempotency_key(message)}"

        # Check if already processed
        cached = self.redis.get(key)
        if cached:
            return json.loads(cached)

        # Process
        result = handler(message)

        # Cache result
        self.redis.setex(key, self.ttl, json.dumps(result))

        return result
```

### 3. Backpressure

```
Проблема: Consumer не успевает обрабатывать

Producer rate: 1000 msg/sec
Consumer rate: 100 msg/sec
Result: Queue grows infinitely → OOM

Решения:

1. Rate limiting на producer:
   if queue.size > threshold:
       slow_down() or reject()

2. Consumer prefetch limit:
   consumer.prefetch(10)  # Max 10 unacked messages

3. Queue limits:
   max_length: 100000
   overflow: reject / drop-oldest

4. Auto-scaling consumers:
   if queue.size > X:
       scale_up_consumers()
```

```python
# Backpressure with Redis Streams
def producer_with_backpressure(stream, data, max_pending=10000):
    # Check pending messages
    info = redis.xinfo_stream(stream)
    pending = info['length']

    if pending > max_pending:
        # Option 1: Wait
        time.sleep(1)
        return producer_with_backpressure(stream, data, max_pending)

        # Option 2: Reject
        # raise BackpressureError("Queue full")

        # Option 3: Drop oldest
        # redis.xtrim(stream, maxlen=max_pending)

    return redis.xadd(stream, data)
```

### 4. Poison Messages

```
Проблема: Сообщение вызывает crash consumer

Message → Consumer → Crash → Redelivery → Crash → ∞

Решения:

1. Retry limits:
   After N failures → move to DLQ

2. Message validation:
   Validate before processing

3. Error isolation:
   Try-catch, log, continue

4. Circuit breaker:
   If error rate > threshold → stop consuming
```

```python
class PoisonMessageHandler:
    def __init__(self, max_retries=3):
        self.max_retries = max_retries
        self.retry_counts = {}

    def process(self, message, handler, dlq):
        message_id = message['id']
        retry_count = self.retry_counts.get(message_id, 0)

        try:
            # Validate first
            if not self.validate(message):
                raise InvalidMessageError("Invalid message format")

            # Process
            result = handler(message)

            # Success - clean up
            self.retry_counts.pop(message_id, None)
            return result

        except Exception as e:
            retry_count += 1
            self.retry_counts[message_id] = retry_count

            if retry_count >= self.max_retries:
                # Move to DLQ
                dlq.send({
                    'original_message': message,
                    'error': str(e),
                    'retry_count': retry_count
                })
                self.retry_counts.pop(message_id, None)
                logger.error(f"Message {message_id} moved to DLQ")
            else:
                # Will be retried
                logger.warning(f"Message {message_id} failed, retry {retry_count}")
                raise
```

---

## Best Practices

### 1. Message Design

```python
# Good message structure
message = {
    # Metadata
    "id": "msg-uuid-12345",            # Unique ID for idempotency
    "type": "order.created",           # Event type
    "version": "1.0",                  # Schema version
    "timestamp": "2024-01-15T10:30:00Z",
    "correlation_id": "req-uuid-67890", # For tracing
    "source": "order-service",

    # Headers (for routing, not business logic)
    "headers": {
        "content-type": "application/json",
        "priority": "high"
    },

    # Payload (business data)
    "data": {
        "order_id": "ORD-001",
        "customer_id": "CUST-123",
        "items": [...]
    }
}
```

### 2. Error Handling

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=60)
)
def process_with_retry(message):
    """Retry with exponential backoff"""
    return external_api.call(message)

def safe_process(message):
    try:
        result = process_with_retry(message)
        return {"status": "success", "result": result}
    except Exception as e:
        # Log context for debugging
        logger.error(
            "Processing failed",
            extra={
                "message_id": message['id'],
                "error": str(e),
                "trace_id": message.get('correlation_id')
            }
        )
        # Move to DLQ for investigation
        send_to_dlq(message, e)
        return {"status": "failed", "error": str(e)}
```

### 3. Monitoring и Alerting

```python
# Key metrics to monitor
metrics = {
    # Queue health
    "queue_length": Gauge,           # Current messages in queue
    "queue_age_seconds": Gauge,      # Age of oldest message
    "messages_in_flight": Gauge,     # Processing now

    # Throughput
    "messages_published_total": Counter,
    "messages_consumed_total": Counter,
    "messages_failed_total": Counter,

    # Latency
    "message_processing_seconds": Histogram,
    "end_to_end_latency_seconds": Histogram,

    # DLQ
    "dlq_length": Gauge,
}

# Alerts
alerts = [
    "queue_length > 10000 for 5m",           # Queue growing
    "queue_age_seconds > 300",               # Messages stuck
    "messages_failed_total increase > 10/m", # Failure spike
    "dlq_length > 0",                        # Any DLQ messages
]
```

### 4. Testing

```python
import pytest
from unittest.mock import Mock, patch

class TestMessageProcessing:

    def test_successful_processing(self):
        message = create_test_message()
        result = process_message(message)
        assert result['status'] == 'success'

    def test_idempotency(self):
        message = create_test_message()

        result1 = process_message(message)
        result2 = process_message(message)

        assert result1 == result2
        # Verify side effects happened only once
        assert db.count(Order) == 1

    def test_retry_on_transient_error(self):
        message = create_test_message()

        with patch('external_api.call') as mock:
            mock.side_effect = [
                ConnectionError(),  # 1st call fails
                ConnectionError(),  # 2nd call fails
                {'status': 'ok'}    # 3rd call succeeds
            ]

            result = process_with_retry(message)
            assert mock.call_count == 3
            assert result['status'] == 'ok'

    def test_dlq_on_permanent_error(self):
        message = create_test_message(invalid=True)

        with patch('dlq.send') as mock_dlq:
            process_message(message)
            mock_dlq.assert_called_once()
```

### 5. Deployment Strategies

```yaml
# Blue-Green deployment for consumers
# 1. Deploy new version (green)
# 2. Verify green processes correctly
# 3. Shift traffic gradually
# 4. Stop old version (blue)

# Consumer deployment with graceful shutdown
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-consumer
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: consumer
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 30"]
        terminationGracePeriodSeconds: 60
```

---

## Примеры архитектур

### 1. E-commerce Order Processing

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Order Processing Pipeline                          │
│                                                                               │
│  ┌─────────┐     ┌─────────────────────────────────────────────────────┐    │
│  │   Web   │────►│                    Kafka                            │    │
│  │   API   │     │                                                     │    │
│  └─────────┘     │  orders topic (partitioned by customer_id)         │    │
│                  └─────────────────────────────────────────────────────┘    │
│                           │              │              │                    │
│                           ▼              ▼              ▼                    │
│                  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│                  │ Validation │  │  Payment   │  │ Inventory  │            │
│                  │  Service   │  │  Service   │  │  Service   │            │
│                  └────────────┘  └────────────┘  └────────────┘            │
│                           │              │              │                    │
│                           ▼              ▼              ▼                    │
│                  ┌─────────────────────────────────────────────────────┐    │
│                  │              order-events topic                     │    │
│                  │  (OrderValidated, PaymentProcessed, InventoryReserved) │ │
│                  └─────────────────────────────────────────────────────┘    │
│                           │              │              │                    │
│                           ▼              ▼              ▼                    │
│                  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│                  │ Shipping   │  │Notification│  │ Analytics  │            │
│                  │  Service   │  │  Service   │  │  Service   │            │
│                  └────────────┘  └────────────┘  └────────────┘            │
│                                                                               │
│  Saga Pattern: координация через события                                    │
│  Каждый сервис публикует результат, следующий реагирует                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2. Real-time Notifications

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Notification System                                   │
│                                                                               │
│  Event Sources:               Fan-out:                  Delivery:            │
│                                                                               │
│  ┌──────────┐                                                                │
│  │ Order    │────┐                                                           │
│  │ Service  │    │         ┌───────────────┐                                │
│  └──────────┘    │         │               │        ┌─────────────┐         │
│                  ├────────►│   RabbitMQ    │───────►│ Email Worker│         │
│  ┌──────────┐    │         │   (fanout)    │        └─────────────┘         │
│  │ Payment  │────┤         │               │        ┌─────────────┐         │
│  │ Service  │    │         │               │───────►│ SMS Worker  │         │
│  └──────────┘    │         │               │        └─────────────┘         │
│                  │         │               │        ┌─────────────┐         │
│  ┌──────────┐    │         │               │───────►│ Push Worker │         │
│  │ Shipping │────┘         │               │        └─────────────┘         │
│  │ Service  │              └───────────────┘        ┌─────────────┐         │
│  └──────────┘                      │                │ WebSocket   │         │
│                                    └───────────────►│ Gateway     │         │
│                                                     └─────────────┘         │
│                                                            │                 │
│  User preferences:                                         ▼                 │
│  {user_id: 123, channels: ["email", "push"]}     ┌─────────────────┐       │
│                                                   │   Web Client    │       │
│  Delivery rules:                                  │ (real-time)     │       │
│  - Critical → all channels                        └─────────────────┘       │
│  - Marketing → respect preferences                                           │
│  - Transactional → email required                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3. Data Pipeline (ETL)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Data Processing Pipeline                            │
│                                                                               │
│  Sources:          Stream Processing:           Sinks:                       │
│                                                                               │
│  ┌──────────┐      ┌───────────────┐                                        │
│  │ App Logs │─────►│ Kafka Topic:  │                                        │
│  └──────────┘      │ raw_events    │      ┌─────────────────┐               │
│                    └───────────────┘      │                 │               │
│  ┌──────────┐             │               │  Elasticsearch  │───► Kibana    │
│  │ Metrics  │─────────────┤               │   (real-time)   │               │
│  └──────────┘             │               └─────────────────┘               │
│                           ▼                                                  │
│  ┌──────────┐      ┌───────────────┐      ┌─────────────────┐               │
│  │ User     │─────►│ Kafka Streams │─────►│    BigQuery     │───► Looker    │
│  │ Actions  │      │  Processing   │      │   (analytics)   │               │
│  └──────────┘      └───────────────┘      └─────────────────┘               │
│                           │                                                  │
│                           │               ┌─────────────────┐               │
│                           └──────────────►│  Redis Cache    │───► API       │
│                                           │ (aggregations)  │               │
│                                           └─────────────────┘               │
│                                                                               │
│  Processing steps:                                                           │
│  1. Parse & validate                                                         │
│  2. Enrich (user data, geo)                                                 │
│  3. Aggregate (counts, sums)                                                │
│  4. Route to appropriate sink                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4. Microservices Saga Pattern

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Order Saga (Choreography-based)                           │
│                                                                               │
│  Happy Path:                                                                 │
│                                                                               │
│  ┌────────┐   OrderCreated   ┌─────────┐   PaymentCompleted   ┌──────────┐ │
│  │ Order  │─────────────────►│ Payment │─────────────────────►│Inventory │ │
│  │Service │                  │ Service │                      │ Service  │ │
│  └────────┘                  └─────────┘                      └──────────┘ │
│                                                                      │       │
│                              InventoryReserved                       │       │
│  ┌────────┐◄─────────────────────────────────────────────────────────┘       │
│  │Shipping│                                                                  │
│  │Service │                                                                  │
│  └────────┘                                                                  │
│                                                                               │
│  Compensation (rollback):                                                    │
│                                                                               │
│  Payment fails:                                                              │
│  ┌────────┐   PaymentFailed   ┌────────┐                                    │
│  │ Order  │◄──────────────────│Payment │                                    │
│  │Service │  (cancel order)   │Service │                                    │
│  └────────┘                   └────────┘                                    │
│                                                                               │
│  Inventory fails:                                                            │
│  ┌────────┐   InventoryFailed  ┌─────────┐   RefundPayment   ┌─────────┐   │
│  │ Order  │◄───────────────────│Inventory│──────────────────►│ Payment │   │
│  │Service │   (cancel order)   │ Service │                   │ Service │   │
│  └────────┘                    └─────────┘                   └─────────┘   │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Заключение

**Когда использовать асинхронность:**

| Сценарий | Подход |
|----------|--------|
| Быстрый ответ пользователю | Async (очередь задач) |
| Длительные операции | Async (background jobs) |
| Интеграция с внешними API | Async (retry, fault tolerance) |
| Высокая нагрузка | Async (load leveling) |
| Event-driven системы | Async (pub/sub, event sourcing) |
| Простой CRUD | Sync (проще отлаживать) |
| Требуется немедленный результат | Sync или Request-Reply |

**Выбор технологии:**

| Требование | Технология |
|------------|------------|
| Простые задачи, Python | Celery + Redis |
| Enterprise, сложный routing | RabbitMQ |
| High-throughput, event sourcing | Kafka |
| AWS-native, managed | SQS/SNS |
| Lightweight, Redis already in stack | Redis Streams |

**Ключевые принципы:**
1. **Idempotency** — всегда предполагай дубликаты
2. **Monitoring** — отслеживай queue depth и latency
3. **DLQ** — обрабатывай failed messages
4. **Testing** — тестируй failure scenarios
5. **Graceful shutdown** — завершай обработку перед остановкой
