# Streams vs Queues: сравнение и выбор

## Введение

RabbitMQ предлагает два типа структур данных для обмена сообщениями: классические очереди (Queues) и потоки (Streams). Понимание различий между ними критично для правильного выбора архитектуры.

## Фундаментальные различия

### Концептуальная модель

```
QUEUE (Очередь)
┌─────────────────────────────────────────┐
│  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐        │
│  │ 1 │→│ 2 │→│ 3 │→│ 4 │→│ 5 │→ Consumer│
│  └───┘ └───┘ └───┘ └───┘ └───┘        │
│                                         │
│  После ACK сообщение УДАЛЯЕТСЯ          │
└─────────────────────────────────────────┘

STREAM (Поток)
┌─────────────────────────────────────────┐
│  ┌───┬───┬───┬───┬───┬───┬───┬───┐     │
│  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │...│ →   │
│  └───┴───┴───┴───┴───┴───┴───┴───┘     │
│    ↑       ↑           ↑               │
│   C1      C2          C3               │
│                                         │
│  Сообщения СОХРАНЯЮТСЯ до retention    │
└─────────────────────────────────────────┘
```

### Ключевые отличия

| Характеристика | Queue | Stream |
|---------------|-------|--------|
| Модель данных | FIFO очередь | Append-only log |
| Хранение | До acknowledgment | До retention policy |
| Удаление сообщений | После ACK | Только по retention |
| Повторное чтение | Нет (requeue = исключение) | Да, с любого offset |
| Множественные consumers | Конкурентное чтение | Независимое чтение |
| Протокол | AMQP 0.9.1 | Stream Protocol / AMQP |
| Порт по умолчанию | 5672 | 5552 (Stream), 5672 (AMQP) |

## Сравнение по функциональности

### 1. Работа с сообщениями

#### Queue

```python
import pika

def queue_consumer():
    """Consumer для классической очереди"""
    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    channel = connection.channel()

    channel.queue_declare(queue='tasks', durable=True)

    def callback(ch, method, properties, body):
        print(f"Получено: {body.decode()}")
        # После ACK сообщение удаляется навсегда
        ch.basic_ack(delivery_tag=method.delivery_tag)

    channel.basic_consume(queue='tasks', on_message_callback=callback)
    channel.start_consuming()
```

#### Stream

```python
import asyncio
from rstream import Consumer, OffsetSpecification

async def stream_consumer():
    """Consumer для stream - можно перечитывать"""
    consumer = Consumer(host='localhost', port=5552, username='guest', password='guest')
    await consumer.start()

    async def callback(message, context):
        print(f"Offset {context.offset}: {message.body.decode()}")
        # Сообщение остаётся в stream!

    # Можно начать с любого места
    await consumer.subscribe(
        stream='events',
        callback=callback,
        offset_specification=OffsetSpecification.first()  # С начала
    )

    await asyncio.sleep(30)
    await consumer.close()
```

### 2. Множественные потребители

#### Queue: Конкурентное потребление

```python
# Consumer 1 и Consumer 2 делят сообщения между собой
# Каждое сообщение обрабатывается ОДНИМ consumer

# Producer отправляет: M1, M2, M3, M4, M5, M6
# Consumer 1 получает: M1, M3, M5
# Consumer 2 получает: M2, M4, M6

def setup_competing_consumers():
    """Конкурентные consumers на одной очереди"""
    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    channel = connection.channel()

    channel.queue_declare(queue='tasks')

    # prefetch_count определяет сколько сообщений consumer получает за раз
    channel.basic_qos(prefetch_count=1)

    def callback(ch, method, properties, body):
        print(f"Worker processing: {body.decode()}")
        ch.basic_ack(delivery_tag=method.delivery_tag)

    channel.basic_consume(queue='tasks', on_message_callback=callback)
    channel.start_consuming()
```

#### Stream: Независимое потребление

```python
# Consumer 1 и Consumer 2 читают ОДНИ И ТЕ ЖЕ сообщения

# Producer отправляет: M1, M2, M3, M4, M5, M6
# Consumer 1 (Analytics) получает: M1, M2, M3, M4, M5, M6
# Consumer 2 (Archiver) получает: M1, M2, M3, M4, M5, M6

async def setup_independent_consumers():
    """Независимые consumers на одном stream"""

    async def create_consumer(name, reference):
        consumer = Consumer(host='localhost', port=5552, username='guest', password='guest')
        await consumer.start()

        async def callback(message, context):
            print(f"[{name}] offset={context.offset}: {message.body.decode()}")

        await consumer.subscribe(
            stream='events',
            callback=callback,
            offset_specification=OffsetSpecification.first()
        )
        return consumer

    # Оба consumer читают ВСЕ сообщения
    analytics = await create_consumer('Analytics', 'analytics-ref')
    archiver = await create_consumer('Archiver', 'archiver-ref')

    await asyncio.sleep(60)
```

### 3. Routing и Exchange

#### Queue: Полная поддержка

```python
import pika

def setup_queue_routing():
    """Очереди поддерживают все типы exchange"""
    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    channel = connection.channel()

    # Direct exchange
    channel.exchange_declare(exchange='orders', exchange_type='direct')
    channel.queue_declare(queue='order-processing')
    channel.queue_bind(queue='order-processing', exchange='orders', routing_key='new')

    # Topic exchange
    channel.exchange_declare(exchange='logs', exchange_type='topic')
    channel.queue_declare(queue='error-logs')
    channel.queue_bind(queue='error-logs', exchange='logs', routing_key='*.error')

    # Fanout exchange
    channel.exchange_declare(exchange='notifications', exchange_type='fanout')
    channel.queue_declare(queue='email-notifier')
    channel.queue_bind(queue='email-notifier', exchange='notifications')

    connection.close()
```

#### Stream: Ограниченная поддержка

```python
import pika

def setup_stream_routing():
    """Streams можно привязать к exchange, но с ограничениями"""
    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    channel = connection.channel()

    # Создание stream
    channel.queue_declare(
        queue='events-stream',
        arguments={'x-queue-type': 'stream'}
    )

    # Можно привязать к exchange
    channel.exchange_declare(exchange='events', exchange_type='topic')
    channel.queue_bind(
        queue='events-stream',
        exchange='events',
        routing_key='order.#'
    )

    # Но: сообщения через exchange публикуются по AMQP
    # Что менее эффективно чем прямая публикация через Stream Protocol

    connection.close()
```

### 4. Dead Letter и Retry

#### Queue: Полная поддержка

```python
def setup_dlx():
    """Dead Letter Exchange для очередей"""
    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    channel = connection.channel()

    # DLX и очередь для неудачных сообщений
    channel.exchange_declare(exchange='dlx', exchange_type='direct')
    channel.queue_declare(queue='failed-tasks')
    channel.queue_bind(queue='failed-tasks', exchange='dlx', routing_key='tasks')

    # Основная очередь с DLX
    channel.queue_declare(
        queue='tasks',
        arguments={
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'tasks',
            'x-message-ttl': 60000  # 1 минута TTL
        }
    )

    connection.close()
```

#### Stream: Нет поддержки DLX

```python
# Streams НЕ поддерживают Dead Letter Exchange
# Нужно реализовывать retry логику в приложении

async def stream_with_manual_retry():
    """Ручная обработка ошибок для stream"""
    consumer = Consumer(host='localhost', port=5552, username='guest', password='guest')
    await consumer.start()

    error_log = []  # Хранение неудачных offset'ов

    async def callback(message, context):
        try:
            process_message(message)
        except Exception as e:
            # Записываем ошибку, но продолжаем
            error_log.append({
                'offset': context.offset,
                'error': str(e),
                'message': message.body
            })

    await consumer.subscribe(
        stream='events',
        callback=callback,
        offset_specification=OffsetSpecification.first()
    )

    # Периодически обрабатываем ошибки
    async def process_errors():
        while True:
            if error_log:
                failed = error_log.pop(0)
                # Retry логика
                print(f"Retrying offset {failed['offset']}")
            await asyncio.sleep(10)

    await asyncio.gather(
        asyncio.sleep(300),
        process_errors()
    )
```

## Сравнение производительности

### Пропускная способность

```
Тест: 1 миллион сообщений по 1KB

QUEUE (Classic)
├── Publisher: ~20,000 msg/sec
├── Consumer: ~15,000 msg/sec
└── Latency: 1-5 ms

QUEUE (Quorum)
├── Publisher: ~15,000 msg/sec
├── Consumer: ~12,000 msg/sec
└── Latency: 2-10 ms

STREAM
├── Publisher: ~100,000+ msg/sec
├── Consumer: ~200,000+ msg/sec (batch read)
└── Latency: <1 ms (в пределах сегмента)
```

### Использование ресурсов

```
QUEUE
├── Память: Пропорциональна количеству сообщений
├── Диск: Зависит от persistency
└── CPU: Умеренная нагрузка

STREAM
├── Память: Фиксированный буфер
├── Диск: Основное хранение
└── CPU: Низкая нагрузка на чтение
```

## Когда использовать Queue

### Идеальные сценарии

```python
# 1. Task Queue - распределение задач между workers
# Каждая задача должна быть обработана ОДИН раз

def task_queue_example():
    """Очередь задач для обработки изображений"""
    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    channel = connection.channel()

    channel.queue_declare(queue='image-processing', durable=True)

    def callback(ch, method, properties, body):
        task = json.loads(body)
        process_image(task['image_url'])
        ch.basic_ack(delivery_tag=method.delivery_tag)

    channel.basic_qos(prefetch_count=1)  # Fair dispatch
    channel.basic_consume(queue='image-processing', on_message_callback=callback)
    channel.start_consuming()
```

```python
# 2. RPC паттерн - запрос-ответ
# Нужна корреляция между запросом и ответом

def rpc_client():
    """RPC клиент"""
    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    channel = connection.channel()

    result = channel.queue_declare(queue='', exclusive=True)
    callback_queue = result.method.queue

    correlation_id = str(uuid.uuid4())

    channel.basic_publish(
        exchange='',
        routing_key='rpc_queue',
        properties=pika.BasicProperties(
            reply_to=callback_queue,
            correlation_id=correlation_id,
        ),
        body='request data'
    )
```

```python
# 3. Priority Queue - приоритетная обработка
# Важные сообщения должны обрабатываться первыми

def priority_queue():
    """Очередь с приоритетами"""
    channel.queue_declare(
        queue='priority-tasks',
        arguments={'x-max-priority': 10}
    )

    # Высокий приоритет
    channel.basic_publish(
        exchange='',
        routing_key='priority-tasks',
        body='urgent task',
        properties=pika.BasicProperties(priority=9)
    )

    # Низкий приоритет
    channel.basic_publish(
        exchange='',
        routing_key='priority-tasks',
        body='normal task',
        properties=pika.BasicProperties(priority=1)
    )
```

```python
# 4. Delayed Messages - отложенные сообщения
# Сообщение должно быть доставлено через определённое время

def delayed_message():
    """Отложенная доставка через DLX"""
    # Delay queue
    channel.queue_declare(
        queue='delay-5min',
        arguments={
            'x-dead-letter-exchange': '',
            'x-dead-letter-routing-key': 'destination',
            'x-message-ttl': 300000  # 5 минут
        }
    )

    channel.basic_publish(
        exchange='',
        routing_key='delay-5min',
        body='delayed message'
    )
```

## Когда использовать Stream

### Идеальные сценарии

```python
# 1. Event Sourcing - хранение всех событий
# Нужна полная история изменений

async def event_sourcing_example():
    """Event sourcing для агрегата Order"""
    producer = Producer(host='localhost', port=5552, username='guest', password='guest')
    await producer.start()

    # Все события сохраняются и могут быть воспроизведены
    events = [
        {'type': 'OrderCreated', 'order_id': '123', 'timestamp': '...'},
        {'type': 'ItemAdded', 'order_id': '123', 'item': 'Product A'},
        {'type': 'ItemAdded', 'order_id': '123', 'item': 'Product B'},
        {'type': 'OrderPaid', 'order_id': '123', 'amount': 100.00},
        {'type': 'OrderShipped', 'order_id': '123', 'tracking': 'ABC123'}
    ]

    for event in events:
        await producer.send(
            stream='orders-events',
            message=AMQPMessage(body=json.dumps(event).encode())
        )

    await producer.close()
```

```python
# 2. Множественные consumers - один поток, много читателей
# Каждый сервис читает все события независимо

async def multiple_consumers():
    """Несколько сервисов читают один stream"""

    # Сервис аналитики - читает всё для построения метрик
    analytics_consumer = Consumer(...)
    await analytics_consumer.subscribe(
        stream='user-activity',
        callback=analytics_handler,
        offset_specification=OffsetSpecification.first()
    )

    # Сервис уведомлений - читает только новые
    notifications_consumer = Consumer(...)
    await notifications_consumer.subscribe(
        stream='user-activity',
        callback=notifications_handler,
        offset_specification=OffsetSpecification.next()
    )

    # Сервис ML - читает данные за последние 24 часа
    ml_consumer = Consumer(...)
    await ml_consumer.subscribe(
        stream='user-activity',
        callback=ml_handler,
        offset_specification=OffsetSpecification.timestamp(yesterday)
    )
```

```python
# 3. Replay/Reprocessing - перечитывание данных
# Нужно переобработать данные после исправления бага

async def replay_processing():
    """Перечитывание данных с начала"""
    consumer = Consumer(host='localhost', port=5552, username='guest', password='guest')
    await consumer.start()

    async def reprocess_handler(message, context):
        # Применяем исправленную логику ко всем данным
        data = json.loads(message.body)
        process_with_fixed_logic(data)

        if context.offset % 10000 == 0:
            print(f"Reprocessed {context.offset} messages")

    # Начинаем с начала stream
    await consumer.subscribe(
        stream='transactions',
        callback=reprocess_handler,
        offset_specification=OffsetSpecification.first()
    )
```

```python
# 4. Audit Log - журнал аудита
# Все действия должны храниться долгосрочно

async def audit_log():
    """Журнал аудита с долгосрочным хранением"""
    producer = Producer(host='localhost', port=5552, username='guest', password='guest')
    await producer.start()

    # Stream с 1 годом хранения
    await producer.create_stream(
        stream='audit-log',
        arguments={
            'max-age': '365D',  # Хранить 1 год
            'max-length-bytes': 107374182400  # 100GB максимум
        }
    )

    # Все действия логируются
    audit_event = {
        'timestamp': datetime.utcnow().isoformat(),
        'user_id': 'user-123',
        'action': 'DELETE_DOCUMENT',
        'resource_id': 'doc-456',
        'ip_address': '192.168.1.100'
    }

    await producer.send(
        stream='audit-log',
        message=AMQPMessage(body=json.dumps(audit_event).encode())
    )
```

## Гибридный подход

### Комбинирование Queue и Stream

```python
# Архитектура: Stream для хранения + Queue для обработки

async def hybrid_architecture():
    """Stream как источник + Queue для распределения задач"""

    # 1. События публикуются в Stream (хранение)
    stream_producer = Producer(...)
    await stream_producer.send(
        stream='events',
        message=AMQPMessage(body=event_data)
    )

    # 2. Bridge читает Stream и публикует в Queue для обработки
    async def bridge_handler(message, context):
        # Публикуем в Queue через AMQP
        amqp_channel.basic_publish(
            exchange='',
            routing_key='processing-queue',
            body=message.body
        )

    stream_consumer = Consumer(...)
    await stream_consumer.subscribe(
        stream='events',
        callback=bridge_handler,
        offset_specification=OffsetSpecification.next()
    )

    # 3. Workers обрабатывают из Queue
    # (конкурентное потребление, retry, DLX)
```

## Таблица выбора

| Требование | Queue | Stream |
|-----------|-------|--------|
| Одноразовая обработка | Да | Нет* |
| Повторное чтение | Нет | Да |
| Множественные consumers (один и тот же message) | Нет** | Да |
| Priority | Да | Нет |
| Dead Letter | Да | Нет |
| TTL на сообщение | Да | Нет |
| Высокая пропускная способность | Средняя | Высокая |
| Event Sourcing | Нет | Да |
| Долгосрочное хранение | Нет | Да |
| RPC паттерн | Да | Нет |

*В streams можно реализовать семантику одноразовой обработки через offset tracking
**Для broadcast в queues используют fanout exchange с несколькими очередями

## Заключение

### Выбирайте Queue когда:
- Нужна семантика "сообщение обработано и удалено"
- Требуется конкурентная обработка workers
- Нужны приоритеты, TTL, Dead Letter Exchange
- Реализуете RPC или request-reply паттерны

### Выбирайте Stream когда:
- Нужно хранить историю событий
- Несколько consumers должны читать одни и те же данные
- Требуется replay/reprocessing
- Высокая пропускная способность критична
- Реализуете event sourcing или audit log

### Комбинируйте оба типа когда:
- Нужно и хранение истории, и конкурентная обработка
- Разные части системы имеют разные требования
- Требуется максимальная гибкость архитектуры
