# Основы работы со Streams

## Введение

В этом разделе рассмотрим практические аспекты работы с RabbitMQ Streams: создание streams, публикацию сообщений и их чтение с использованием различных клиентских библиотек.

## Установка клиента

### Python библиотека rstream

```bash
# Установка официального Python клиента для Streams
pip install rstream

# Для работы через AMQP также понадобится
pip install pika
```

## Создание Stream

### Через AMQP (pika)

```python
import pika

def create_stream_via_amqp():
    """Создание stream через стандартный AMQP протокол"""

    connection = pika.BlockingConnection(
        pika.ConnectionParameters(
            host='localhost',
            port=5672,
            credentials=pika.PlainCredentials('guest', 'guest')
        )
    )
    channel = connection.channel()

    # Объявление stream с параметрами
    channel.queue_declare(
        queue='events-stream',
        durable=True,  # Streams всегда durable
        arguments={
            'x-queue-type': 'stream',  # Обязательный параметр
            'x-max-length-bytes': 1073741824,  # 1GB максимум
            'x-max-age': '7D',  # Хранить 7 дней
            'x-stream-max-segment-size-bytes': 104857600  # 100MB сегменты
        }
    )

    print("Stream 'events-stream' создан успешно")
    connection.close()

create_stream_via_amqp()
```

### Через Stream Protocol (rstream)

```python
import asyncio
from rstream import Producer, AMQPMessage

async def create_stream_via_protocol():
    """Создание stream через Stream Protocol"""

    producer = Producer(
        host='localhost',
        port=5552,  # Stream Protocol порт
        username='guest',
        password='guest'
    )

    await producer.start()

    # Создание stream
    await producer.create_stream(
        stream='orders-stream',
        arguments={
            'max-length-bytes': 5368709120,  # 5GB
            'max-age': '30D'  # 30 дней
        }
    )

    print("Stream 'orders-stream' создан")
    await producer.close()

asyncio.run(create_stream_via_protocol())
```

## Публикация сообщений

### Простая публикация

```python
import asyncio
from rstream import Producer, AMQPMessage

async def publish_messages():
    """Базовая публикация сообщений в stream"""

    producer = Producer(
        host='localhost',
        port=5552,
        username='guest',
        password='guest'
    )

    await producer.start()

    # Публикация одного сообщения
    await producer.send(
        stream='events-stream',
        message=AMQPMessage(
            body=b'Hello, Streams!'
        )
    )

    # Публикация нескольких сообщений
    for i in range(10):
        message = AMQPMessage(
            body=f'Event #{i}'.encode()
        )
        await producer.send(
            stream='events-stream',
            message=message
        )

    print("Опубликовано 11 сообщений")
    await producer.close()

asyncio.run(publish_messages())
```

### Публикация с подтверждениями

```python
import asyncio
from rstream import Producer, AMQPMessage, ConfirmationStatus

async def publish_with_confirmations():
    """Публикация с отслеживанием подтверждений"""

    confirmed_count = 0
    failed_count = 0

    async def confirmation_handler(confirmation):
        """Обработчик подтверждений"""
        nonlocal confirmed_count, failed_count

        if confirmation.status == ConfirmationStatus.OK:
            confirmed_count += 1
        else:
            failed_count += 1
            print(f"Ошибка публикации: {confirmation.status}")

    producer = Producer(
        host='localhost',
        port=5552,
        username='guest',
        password='guest'
    )

    await producer.start()

    # Публикация с confirmation callback
    for i in range(100):
        await producer.send(
            stream='events-stream',
            message=AMQPMessage(body=f'Event {i}'.encode()),
            on_publish_confirm=confirmation_handler
        )

    # Ждём подтверждений
    await asyncio.sleep(1)

    print(f"Подтверждено: {confirmed_count}, Ошибок: {failed_count}")
    await producer.close()

asyncio.run(publish_with_confirmations())
```

### Batch публикация

```python
import asyncio
from rstream import Producer, AMQPMessage

async def batch_publish():
    """Эффективная batch публикация"""

    producer = Producer(
        host='localhost',
        port=5552,
        username='guest',
        password='guest'
    )

    await producer.start()

    # Подготовка batch сообщений
    messages = []
    for i in range(1000):
        messages.append(
            AMQPMessage(body=f'Batch message {i}'.encode())
        )

    # Отправка batch
    await producer.send_batch(
        stream='events-stream',
        messages=messages
    )

    print("Отправлено 1000 сообщений в batch")
    await producer.close()

asyncio.run(batch_publish())
```

## Чтение сообщений

### Базовое чтение

```python
import asyncio
from rstream import Consumer, AMQPMessage, OffsetSpecification

async def consume_messages():
    """Базовое чтение сообщений из stream"""

    async def message_handler(message: AMQPMessage, message_context):
        """Обработчик сообщений"""
        print(f"Получено: {message.body.decode()}")
        print(f"  Offset: {message_context.offset}")
        print(f"  Timestamp: {message_context.timestamp}")

    consumer = Consumer(
        host='localhost',
        port=5552,
        username='guest',
        password='guest'
    )

    await consumer.start()

    # Подписка на stream
    await consumer.subscribe(
        stream='events-stream',
        callback=message_handler,
        offset_specification=OffsetSpecification.first()  # С начала
    )

    # Читаем 10 секунд
    await asyncio.sleep(10)

    await consumer.close()

asyncio.run(consume_messages())
```

### Чтение с разных позиций (Offset)

```python
import asyncio
from rstream import Consumer, OffsetSpecification
from datetime import datetime, timedelta

async def consume_from_different_offsets():
    """Демонстрация различных способов указания начальной позиции"""

    async def handler(message, context):
        print(f"[{context.offset}] {message.body.decode()}")

    consumer = Consumer(
        host='localhost',
        port=5552,
        username='guest',
        password='guest'
    )

    await consumer.start()

    # Вариант 1: С самого начала stream
    await consumer.subscribe(
        stream='events-stream',
        callback=handler,
        offset_specification=OffsetSpecification.first()
    )

    # Вариант 2: С конца (только новые сообщения)
    await consumer.subscribe(
        stream='events-stream',
        callback=handler,
        offset_specification=OffsetSpecification.last()
    )

    # Вариант 3: Следующее после последнего
    await consumer.subscribe(
        stream='events-stream',
        callback=handler,
        offset_specification=OffsetSpecification.next()
    )

    # Вариант 4: С конкретного offset
    await consumer.subscribe(
        stream='events-stream',
        callback=handler,
        offset_specification=OffsetSpecification.offset(1000)
    )

    # Вариант 5: С определённого времени
    one_hour_ago = datetime.now() - timedelta(hours=1)
    await consumer.subscribe(
        stream='events-stream',
        callback=handler,
        offset_specification=OffsetSpecification.timestamp(
            int(one_hour_ago.timestamp() * 1000)  # Миллисекунды
        )
    )

    await consumer.close()
```

### Чтение через AMQP

```python
import pika

def consume_via_amqp():
    """Чтение stream через стандартный AMQP (с ограничениями)"""

    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Важно: указываем x-stream-offset
    channel.basic_qos(prefetch_count=100)

    def callback(ch, method, properties, body):
        print(f"Получено: {body.decode()}")
        # В streams acknowledgement работает иначе
        ch.basic_ack(delivery_tag=method.delivery_tag)

    # Подписка с указанием offset
    channel.basic_consume(
        queue='events-stream',
        on_message_callback=callback,
        arguments={
            'x-stream-offset': 'first'  # first, last, next, offset, timestamp
        }
    )

    print("Ожидание сообщений... (Ctrl+C для выхода)")
    try:
        channel.start_consuming()
    except KeyboardInterrupt:
        channel.stop_consuming()

    connection.close()

consume_via_amqp()
```

## Работа с сообщениями

### Структура сообщения

```python
from rstream import AMQPMessage

# Полная структура AMQP сообщения для stream
message = AMQPMessage(
    # Тело сообщения
    body=b'{"event": "user_created", "user_id": 123}',

    # Свойства приложения (произвольные)
    application_properties={
        'event_type': 'user_created',
        'version': '1.0',
        'source': 'auth-service'
    },

    # Стандартные свойства AMQP
    properties={
        'message_id': 'msg-12345',
        'correlation_id': 'req-67890',
        'content_type': 'application/json',
        'user_id': 'auth-service'
    }
)
```

### Фильтрация сообщений

```python
import asyncio
from rstream import Consumer, OffsetSpecification, ConsumerFilter

async def consume_with_filter():
    """Чтение с серверной фильтрацией (RabbitMQ 3.13+)"""

    async def handler(message, context):
        print(f"Filtered: {message.body.decode()}")

    consumer = Consumer(
        host='localhost',
        port=5552,
        username='guest',
        password='guest'
    )

    await consumer.start()

    # Подписка с фильтром
    await consumer.subscribe(
        stream='events-stream',
        callback=handler,
        offset_specification=OffsetSpecification.first(),
        filter=ConsumerFilter(
            values=['user_created', 'user_updated'],  # Фильтр по значениям
            match_unfiltered=False  # Не получать сообщения без filter value
        )
    )

    await asyncio.sleep(30)
    await consumer.close()

asyncio.run(consume_with_filter())
```

## Управление Streams

### Информация о stream

```python
import asyncio
from rstream import Producer

async def get_stream_info():
    """Получение информации о stream"""

    producer = Producer(
        host='localhost',
        port=5552,
        username='guest',
        password='guest'
    )

    await producer.start()

    # Получение метаданных
    metadata = await producer.query_metadata(
        streams=['events-stream', 'orders-stream']
    )

    for stream_name, stream_metadata in metadata.items():
        print(f"\nStream: {stream_name}")
        print(f"  Leader: {stream_metadata.leader}")
        print(f"  Replicas: {stream_metadata.replicas}")

    await producer.close()

asyncio.run(get_stream_info())
```

### Удаление stream

```python
import asyncio
from rstream import Producer

async def delete_stream():
    """Удаление stream"""

    producer = Producer(
        host='localhost',
        port=5552,
        username='guest',
        password='guest'
    )

    await producer.start()

    # Удаление stream
    await producer.delete_stream('events-stream')
    print("Stream удалён")

    await producer.close()

asyncio.run(delete_stream())
```

## Super Streams (Partitioned Streams)

Super Streams позволяют партиционировать данные для масштабирования:

```python
import asyncio
from rstream import Producer, SuperStreamProducer, AMQPMessage

async def work_with_super_stream():
    """Работа с партиционированным stream"""

    producer = Producer(
        host='localhost',
        port=5552,
        username='guest',
        password='guest'
    )

    await producer.start()

    # Создание super stream с 3 партициями
    await producer.create_super_stream(
        super_stream='orders-super',
        partitions=3
    )

    # Публикация с routing
    super_producer = SuperStreamProducer(
        producer=producer,
        super_stream='orders-super',
        routing_extractor=lambda msg: msg.application_properties.get('order_id')
    )

    # Сообщения с одинаковым order_id попадут в одну партицию
    for i in range(100):
        await super_producer.send(
            AMQPMessage(
                body=f'Order {i}'.encode(),
                application_properties={'order_id': str(i % 10)}
            )
        )

    await producer.close()

asyncio.run(work_with_super_stream())
```

## Обработка ошибок

```python
import asyncio
from rstream import Producer, Consumer, AMQPMessage
from rstream.exceptions import StreamDoesNotExist, StreamAlreadyExists

async def error_handling():
    """Обработка типичных ошибок"""

    producer = Producer(
        host='localhost',
        port=5552,
        username='guest',
        password='guest'
    )

    try:
        await producer.start()

        # Попытка создать существующий stream
        try:
            await producer.create_stream('existing-stream')
        except StreamAlreadyExists:
            print("Stream уже существует - продолжаем работу")

        # Попытка публикации в несуществующий stream
        try:
            await producer.send(
                stream='non-existent-stream',
                message=AMQPMessage(body=b'test')
            )
        except StreamDoesNotExist:
            print("Stream не существует - создаём")
            await producer.create_stream('non-existent-stream')

    except ConnectionError as e:
        print(f"Ошибка подключения: {e}")
    finally:
        await producer.close()

asyncio.run(error_handling())
```

## Best Practices

### Производительность

1. **Используйте batch публикацию** для высокой пропускной способности
2. **Настройте prefetch** на стороне consumer
3. **Используйте Stream Protocol** вместо AMQP для максимальной производительности

### Надёжность

1. **Всегда проверяйте подтверждения** при критичных данных
2. **Настройте репликацию** для production
3. **Обрабатывайте disconnect/reconnect** в коде

### Масштабирование

1. **Используйте Super Streams** для партиционирования
2. **Распределяйте consumers** по партициям
3. **Мониторьте lag** между publisher и consumer

## Заключение

Работа с RabbitMQ Streams включает:
- Создание streams с нужными параметрами retention
- Публикацию сообщений (одиночную и batch)
- Чтение с различных offset позиций
- Использование Super Streams для масштабирования

Для production рекомендуется использовать Stream Protocol (порт 5552) с библиотекой rstream для максимальной производительности.
