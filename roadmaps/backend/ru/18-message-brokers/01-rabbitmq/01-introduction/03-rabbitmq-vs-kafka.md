# RabbitMQ vs Kafka

## Введение

RabbitMQ и Apache Kafka — два самых популярных решения для обмена сообщениями, но они решают разные задачи и имеют принципиально различные архитектуры. Понимание этих различий критически важно для выбора правильного инструмента.

---

## Фундаментальные различия

### Архитектурная философия

```
┌─────────────────────────────────────────────────────────────────────┐
│                         RabbitMQ                                    │
│                    "Smart Broker, Dumb Consumer"                    │
│                                                                     │
│  Producer ──▶ Exchange ──▶ Queue ──▶ Consumer                      │
│                   │                     ▲                           │
│                   └── Routing Logic ────┘                           │
│                                                                     │
│  • Брокер отвечает за маршрутизацию                                │
│  • Сообщение удаляется после подтверждения                         │
│  • Push-модель доставки                                            │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                          Kafka                                      │
│                    "Dumb Broker, Smart Consumer"                    │
│                                                                     │
│  Producer ──▶ Topic (Partition 0) ──▶ Consumer Group               │
│               Topic (Partition 1) ──▶ Consumer Group               │
│               Topic (Partition 2) ──▶ Consumer Group               │
│                       │                                             │
│                       └── Append-only Log                           │
│                                                                     │
│  • Брокер — это лог с append-only записью                          │
│  • Сообщения хранятся, consumer отслеживает offset                 │
│  • Pull-модель доставки                                            │
└─────────────────────────────────────────────────────────────────────┘
```

### Ключевые концепции

| Аспект | RabbitMQ | Kafka |
|--------|----------|-------|
| **Модель** | Message Queue | Distributed Log |
| **Хранение** | До подтверждения (ack) | Retention period (дни/недели) |
| **Порядок** | В пределах очереди | В пределах partition |
| **Доставка** | Push (брокер отправляет) | Pull (consumer запрашивает) |
| **Routing** | Exchange + Binding | Partition key |
| **Replay** | Нет (сообщение удаляется) | Да (смещение offset) |

---

## Сравнение производительности

### Пропускная способность

```
Throughput (сообщений/сек, примерные значения):

RabbitMQ:
├── Простые сообщения: 20,000 - 50,000 msg/s
├── С persistence: 10,000 - 30,000 msg/s
└── Кластер 3 ноды: 30,000 - 100,000 msg/s

Kafka:
├── Один брокер: 100,000 - 200,000 msg/s
├── Кластер 3 брокера: 500,000 - 1,000,000 msg/s
└── Оптимизированный: до 2,000,000+ msg/s
```

### Latency (задержка)

| Сценарий | RabbitMQ | Kafka |
|----------|----------|-------|
| P50 | ~1-2 ms | ~5-10 ms |
| P99 | ~5-10 ms | ~20-50 ms |
| P99.9 | ~10-50 ms | ~50-100 ms |

**Вывод:** RabbitMQ быстрее для низкой задержки, Kafka — для высокой пропускной способности.

---

## Сравнение архитектуры

### RabbitMQ Cluster

```python
# Подключение к кластеру RabbitMQ
import pika

# Список нод кластера
parameters = [
    pika.ConnectionParameters('rabbit-node1'),
    pika.ConnectionParameters('rabbit-node2'),
    pika.ConnectionParameters('rabbit-node3'),
]

# Подключение с failover
connection = pika.BlockingConnection(parameters)
channel = connection.channel()

# Создание зеркальной очереди (classic mirroring)
channel.queue_declare(
    queue='ha_queue',
    durable=True,
    arguments={
        'x-queue-type': 'quorum'  # Quorum queue для HA
    }
)
```

### Kafka Cluster

```python
# Подключение к кластеру Kafka
from kafka import KafkaProducer, KafkaConsumer

# Producer
producer = KafkaProducer(
    bootstrap_servers=['kafka1:9092', 'kafka2:9092', 'kafka3:9092'],
    acks='all',  # Подтверждение от всех реплик
    retries=3
)

# Consumer с consumer group
consumer = KafkaConsumer(
    'my-topic',
    bootstrap_servers=['kafka1:9092', 'kafka2:9092', 'kafka3:9092'],
    group_id='my-consumer-group',
    auto_offset_reset='earliest',
    enable_auto_commit=False
)
```

---

## Модели доставки сообщений

### RabbitMQ — Push Model

```python
# RabbitMQ: брокер отправляет сообщения consumer'у
def callback(ch, method, properties, body):
    print(f"Получено: {body}")
    # Обработка
    ch.basic_ack(delivery_tag=method.delivery_tag)

# Контроль потока через prefetch
channel.basic_qos(prefetch_count=10)  # Не более 10 сообщений за раз

# Подписка — брокер будет отправлять сообщения
channel.basic_consume(queue='tasks', on_message_callback=callback)
channel.start_consuming()
```

### Kafka — Pull Model

```python
# Kafka: consumer запрашивает сообщения
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    'orders',
    bootstrap_servers=['localhost:9092'],
    group_id='order-processor'
)

# Consumer сам запрашивает данные
for message in consumer:  # Polling loop
    print(f"Partition: {message.partition}, Offset: {message.offset}")
    print(f"Value: {message.value}")
    # Обработка
    consumer.commit()  # Фиксация offset
```

---

## Гарантии доставки

### Семантика доставки

| Гарантия | RabbitMQ | Kafka |
|----------|----------|-------|
| **At-most-once** | auto_ack=True | No commit |
| **At-least-once** | Manual ack | Auto/Manual commit |
| **Exactly-once** | Сложно реализовать | Transactional API |

### RabbitMQ: At-least-once

```python
# RabbitMQ: гарантия at-least-once
channel.confirm_delivery()

def callback(ch, method, properties, body):
    try:
        process_message(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception:
        # Вернуть в очередь для повторной обработки
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)
```

### Kafka: Exactly-once

```python
# Kafka: exactly-once semantics (EOS)
from kafka import KafkaProducer

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    enable_idempotence=True,  # Идемпотентность
    transactional_id='my-transactional-producer'
)

producer.init_transactions()

try:
    producer.begin_transaction()
    producer.send('topic1', value=b'message1')
    producer.send('topic2', value=b'message2')
    producer.commit_transaction()
except Exception:
    producer.abort_transaction()
```

---

## Порядок сообщений

### RabbitMQ

```python
# RabbitMQ: порядок гарантирован только в одной очереди
# При нескольких consumer'ах порядок НЕ гарантирован

# Для сохранения порядка:
channel.basic_qos(prefetch_count=1)  # Один consumer обрабатывает по одному

# Или использовать Single Active Consumer
channel.queue_declare(
    queue='ordered_queue',
    arguments={'x-single-active-consumer': True}
)
```

### Kafka

```python
# Kafka: порядок гарантирован в пределах partition
from kafka import KafkaProducer

producer = KafkaProducer(bootstrap_servers=['localhost:9092'])

# Сообщения с одинаковым ключом попадут в одну partition
order_id = "order-123"
producer.send('orders', key=order_id.encode(), value=b'created')
producer.send('orders', key=order_id.encode(), value=b'paid')
producer.send('orders', key=order_id.encode(), value=b'shipped')
# Порядок гарантирован для order-123
```

---

## Replay и хранение сообщений

### RabbitMQ: нет replay

```python
# RabbitMQ: после ack сообщение удаляется
# Для "replay" нужно хранить сообщения отдельно или не подтверждать

# Workaround: Dead Letter Exchange для сохранения
channel.exchange_declare(exchange='dlx', exchange_type='fanout')
channel.queue_declare(
    queue='main_queue',
    arguments={
        'x-dead-letter-exchange': 'dlx',
        'x-message-ttl': 86400000  # 24 часа
    }
)
```

### Kafka: встроенный replay

```python
# Kafka: сообщения хранятся, можно перечитать
from kafka import KafkaConsumer, TopicPartition

consumer = KafkaConsumer(bootstrap_servers=['localhost:9092'])
tp = TopicPartition('orders', 0)
consumer.assign([tp])

# Перемотка к началу
consumer.seek_to_beginning(tp)

# Или к конкретному offset
consumer.seek(tp, 1000)

# Или по времени
offsets = consumer.offsets_for_times({tp: 1703673600000})  # Unix ms
consumer.seek(tp, offsets[tp].offset)
```

---

## Сценарии использования

### Когда выбрать RabbitMQ

| Сценарий | Почему RabbitMQ |
|----------|-----------------|
| **Task Queues** | Сложная маршрутизация, приоритеты |
| **RPC** | Встроенная поддержка request-reply |
| **Маршрутизация** | Exchange types, headers routing |
| **Низкая latency** | Миллисекундные задержки |
| **Legacy интеграция** | AMQP, STOMP, MQTT |
| **Простые очереди** | Быстрый старт, меньше зависимостей |

```python
# Пример: RPC с RabbitMQ
import pika
import uuid

class RpcClient:
    def __init__(self):
        self.connection = pika.BlockingConnection()
        self.channel = self.connection.channel()
        result = self.channel.queue_declare(queue='', exclusive=True)
        self.callback_queue = result.method.queue
        self.channel.basic_consume(
            queue=self.callback_queue,
            on_message_callback=self.on_response,
            auto_ack=True
        )
        self.response = None
        self.corr_id = None

    def on_response(self, ch, method, props, body):
        if self.corr_id == props.correlation_id:
            self.response = body

    def call(self, message):
        self.corr_id = str(uuid.uuid4())
        self.channel.basic_publish(
            exchange='',
            routing_key='rpc_queue',
            body=message,
            properties=pika.BasicProperties(
                reply_to=self.callback_queue,
                correlation_id=self.corr_id,
            )
        )
        while self.response is None:
            self.connection.process_data_events()
        return self.response
```

### Когда выбрать Kafka

| Сценарий | Почему Kafka |
|----------|--------------|
| **Event Streaming** | Retention, replay, ordering |
| **Big Data** | Высокая пропускная способность |
| **Event Sourcing** | Лог событий как источник истины |
| **Log Aggregation** | Централизованный сбор логов |
| **Metrics Pipeline** | Большие объёмы метрик |
| **CDC (Change Data Capture)** | Kafka Connect, Debezium |

```python
# Пример: Event Sourcing с Kafka
from kafka import KafkaProducer, KafkaConsumer
import json

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode()
)

# Публикация событий
events = [
    {'type': 'OrderCreated', 'order_id': 1, 'items': ['A', 'B']},
    {'type': 'PaymentReceived', 'order_id': 1, 'amount': 100},
    {'type': 'OrderShipped', 'order_id': 1, 'tracking': 'XYZ'}
]

for event in events:
    producer.send('order-events', key=b'order-1', value=event)
producer.flush()

# Восстановление состояния из событий
consumer = KafkaConsumer(
    'order-events',
    bootstrap_servers=['localhost:9092'],
    auto_offset_reset='earliest',
    value_deserializer=lambda v: json.loads(v.decode())
)

order_state = {}
for message in consumer:
    event = message.value
    if event['type'] == 'OrderCreated':
        order_state = {'items': event['items'], 'status': 'created'}
    elif event['type'] == 'PaymentReceived':
        order_state['paid'] = event['amount']
    elif event['type'] == 'OrderShipped':
        order_state['status'] = 'shipped'
```

---

## Сравнительная таблица

| Критерий | RabbitMQ | Kafka |
|----------|----------|-------|
| **Модель** | Message Queue | Event Log |
| **Протокол** | AMQP, MQTT, STOMP | Kafka Protocol |
| **Язык** | Erlang | Scala/Java |
| **Порядок** | Per queue | Per partition |
| **Retention** | До ack | Configurable |
| **Replay** | Нет | Да |
| **Throughput** | ~50K msg/s | ~500K+ msg/s |
| **Latency** | ~1-2ms | ~5-10ms |
| **Routing** | Flexible (4 types) | Partition key only |
| **Consumer Groups** | Через плагин | Встроено |
| **Transactions** | Базовые | Exactly-once |
| **Управление** | Management UI | Многие инструменты |
| **Операционная сложность** | Средняя | Высокая |

---

## Best Practices по выбору

### Выбирайте RabbitMQ если:
1. Нужна сложная маршрутизация сообщений
2. Важна низкая задержка (latency)
3. Реализуете паттерн RPC
4. Работаете с приоритетными очередями
5. Команда знакома с AMQP
6. Проект небольшой/средний по масштабу

### Выбирайте Kafka если:
1. Высокие требования к пропускной способности
2. Нужен replay событий
3. Реализуете Event Sourcing / CQRS
4. Строите Data Pipeline
5. Важно долгосрочное хранение событий
6. Масштаб — миллионы событий в секунду

---

## Заключение

RabbitMQ и Kafka — не конкуренты, а инструменты для разных задач:

- **RabbitMQ** — классический брокер сообщений с богатой маршрутизацией
- **Kafka** — распределённый лог событий с высокой производительностью

Во многих современных архитектурах они используются вместе:
- Kafka для event streaming и долгосрочного хранения
- RabbitMQ для task queues и RPC между сервисами
