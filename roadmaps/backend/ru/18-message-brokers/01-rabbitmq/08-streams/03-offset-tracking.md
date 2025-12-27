# Offset Tracking в RabbitMQ Streams

## Введение

Offset (смещение) — это позиция сообщения в stream. В отличие от классических очередей, где сообщение удаляется после подтверждения, в streams offset позволяет:

- Читать сообщения с любой позиции
- Возобновлять чтение после перезапуска
- Иметь несколько независимых consumers на одном stream

## Концепция Offset

### Что такое Offset?

```
Stream "events"
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│ M0  │ M1  │ M2  │ M3  │ M4  │ M5  │ M6  │ M7  │
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
  ↑                       ↑                   ↑
  offset=0            offset=4            offset=7
  (first)                                  (last)
                                               ↑
                                           offset=8 (next - будущее сообщение)
```

- **Offset** — это уникальный последовательный номер сообщения в stream
- Начинается с 0
- Монотонно увеличивается
- Никогда не переиспользуется

## Типы Offset Specification

### 1. First — с начала stream

```python
import asyncio
from rstream import Consumer, OffsetSpecification

async def consume_from_first():
    """Чтение всех сообщений с самого начала"""

    consumer = Consumer(
        host='localhost',
        port=5552,
        username='guest',
        password='guest'
    )

    await consumer.start()

    async def handler(message, context):
        print(f"[{context.offset}] {message.body.decode()}")

    # Начинаем с первого сообщения (offset=0)
    await consumer.subscribe(
        stream='events',
        callback=handler,
        offset_specification=OffsetSpecification.first()
    )

    await asyncio.sleep(30)
    await consumer.close()

asyncio.run(consume_from_first())
```

### 2. Last — последнее сообщение

```python
async def consume_from_last():
    """Начинаем с последнего сообщения в stream"""

    consumer = Consumer(host='localhost', port=5552, username='guest', password='guest')
    await consumer.start()

    async def handler(message, context):
        print(f"Последнее и новые: [{context.offset}] {message.body.decode()}")

    # Начинаем с последнего существующего сообщения
    await consumer.subscribe(
        stream='events',
        callback=handler,
        offset_specification=OffsetSpecification.last()
    )

    await asyncio.sleep(60)
    await consumer.close()

asyncio.run(consume_from_last())
```

### 3. Next — только новые сообщения

```python
async def consume_only_new():
    """Получаем только сообщения, опубликованные после подписки"""

    consumer = Consumer(host='localhost', port=5552, username='guest', password='guest')
    await consumer.start()

    async def handler(message, context):
        print(f"Новое сообщение: [{context.offset}] {message.body.decode()}")

    # Не получаем существующие сообщения, только новые
    await consumer.subscribe(
        stream='events',
        callback=handler,
        offset_specification=OffsetSpecification.next()
    )

    print("Ожидаем новые сообщения...")
    await asyncio.sleep(60)
    await consumer.close()

asyncio.run(consume_only_new())
```

### 4. Offset — конкретная позиция

```python
async def consume_from_offset():
    """Чтение с конкретного offset"""

    consumer = Consumer(host='localhost', port=5552, username='guest', password='guest')
    await consumer.start()

    async def handler(message, context):
        print(f"[{context.offset}] {message.body.decode()}")

    # Начинаем с offset 1000
    await consumer.subscribe(
        stream='events',
        callback=handler,
        offset_specification=OffsetSpecification.offset(1000)
    )

    await asyncio.sleep(30)
    await consumer.close()

asyncio.run(consume_from_offset())
```

### 5. Timestamp — с определённого времени

```python
from datetime import datetime, timedelta

async def consume_from_timestamp():
    """Чтение сообщений с определённого момента времени"""

    consumer = Consumer(host='localhost', port=5552, username='guest', password='guest')
    await consumer.start()

    async def handler(message, context):
        msg_time = datetime.fromtimestamp(context.timestamp / 1000)
        print(f"[{context.offset}] {msg_time}: {message.body.decode()}")

    # Сообщения за последний час
    one_hour_ago = datetime.now() - timedelta(hours=1)
    timestamp_ms = int(one_hour_ago.timestamp() * 1000)

    await consumer.subscribe(
        stream='events',
        callback=handler,
        offset_specification=OffsetSpecification.timestamp(timestamp_ms)
    )

    await asyncio.sleep(30)
    await consumer.close()

asyncio.run(consume_from_timestamp())
```

## Автоматическое отслеживание Offset

### Server-side Offset Tracking

RabbitMQ может хранить offset на стороне сервера:

```python
import asyncio
from rstream import Consumer, OffsetSpecification

async def consume_with_auto_tracking():
    """Consumer с автоматическим сохранением offset на сервере"""

    consumer = Consumer(
        host='localhost',
        port=5552,
        username='guest',
        password='guest'
    )

    await consumer.start()

    message_count = 0

    async def handler(message, context):
        nonlocal message_count
        message_count += 1

        print(f"[{context.offset}] {message.body.decode()}")

        # Сохраняем offset каждые 100 сообщений
        if message_count % 100 == 0:
            await consumer.store_offset(
                stream='events',
                offset=context.offset,
                reference='my-consumer-group'  # Уникальное имя consumer
            )
            print(f"Offset {context.offset} сохранён")

    # Пытаемся получить сохранённый offset
    try:
        stored_offset = await consumer.query_offset(
            stream='events',
            reference='my-consumer-group'
        )
        print(f"Найден сохранённый offset: {stored_offset}")
        offset_spec = OffsetSpecification.offset(stored_offset + 1)
    except Exception:
        print("Сохранённый offset не найден, начинаем с начала")
        offset_spec = OffsetSpecification.first()

    await consumer.subscribe(
        stream='events',
        callback=handler,
        offset_specification=offset_spec
    )

    await asyncio.sleep(60)

    # Сохраняем финальный offset перед выходом
    await consumer.close()

asyncio.run(consume_with_auto_tracking())
```

### Класс для управления Offset

```python
import asyncio
from rstream import Consumer, OffsetSpecification
from dataclasses import dataclass
from typing import Optional

@dataclass
class OffsetManager:
    """Менеджер offset для надёжного чтения stream"""

    consumer: Consumer
    stream: str
    reference: str  # Идентификатор consumer group
    auto_commit_interval: int = 100  # Авто-сохранение каждые N сообщений

    _message_count: int = 0
    _last_offset: Optional[int] = None

    async def get_start_offset(self) -> OffsetSpecification:
        """Получить offset для начала чтения"""
        try:
            stored = await self.consumer.query_offset(
                stream=self.stream,
                reference=self.reference
            )
            print(f"Восстановлен offset: {stored}")
            # Начинаем со следующего сообщения
            return OffsetSpecification.offset(stored + 1)
        except Exception as e:
            print(f"Offset не найден: {e}, начинаем с first")
            return OffsetSpecification.first()

    async def track(self, offset: int, force: bool = False):
        """Отслеживать offset и периодически сохранять"""
        self._last_offset = offset
        self._message_count += 1

        if force or self._message_count % self.auto_commit_interval == 0:
            await self.commit()

    async def commit(self):
        """Принудительно сохранить текущий offset"""
        if self._last_offset is not None:
            await self.consumer.store_offset(
                stream=self.stream,
                offset=self._last_offset,
                reference=self.reference
            )
            print(f"Committed offset: {self._last_offset}")


async def reliable_consumer():
    """Пример надёжного consumer с offset tracking"""

    consumer = Consumer(host='localhost', port=5552, username='guest', password='guest')
    await consumer.start()

    offset_manager = OffsetManager(
        consumer=consumer,
        stream='events',
        reference='analytics-consumer',
        auto_commit_interval=50
    )

    async def handler(message, context):
        # Обработка сообщения
        print(f"Processing: {message.body.decode()}")

        # Отслеживание offset
        await offset_manager.track(context.offset)

    # Получаем начальный offset
    start_offset = await offset_manager.get_start_offset()

    await consumer.subscribe(
        stream='events',
        callback=handler,
        offset_specification=start_offset
    )

    try:
        await asyncio.sleep(300)
    except KeyboardInterrupt:
        # Сохраняем offset перед выходом
        await offset_manager.commit()
    finally:
        await consumer.close()

asyncio.run(reliable_consumer())
```

## Resuming (Возобновление чтения)

### Паттерн безопасного возобновления

```python
import asyncio
import signal
from rstream import Consumer, OffsetSpecification

class ResumableConsumer:
    """Consumer с возможностью безопасного возобновления"""

    def __init__(self, stream: str, consumer_name: str):
        self.stream = stream
        self.consumer_name = consumer_name
        self.consumer: Optional[Consumer] = None
        self.running = False
        self.current_offset: Optional[int] = None

    async def start(self):
        """Запуск consumer"""
        self.consumer = Consumer(
            host='localhost',
            port=5552,
            username='guest',
            password='guest'
        )
        await self.consumer.start()
        self.running = True

        # Получаем сохранённый offset или начинаем сначала
        offset_spec = await self._get_resume_offset()

        await self.consumer.subscribe(
            stream=self.stream,
            callback=self._handle_message,
            offset_specification=offset_spec
        )

        print(f"Consumer '{self.consumer_name}' запущен")

    async def _get_resume_offset(self) -> OffsetSpecification:
        """Определить offset для возобновления"""
        try:
            offset = await self.consumer.query_offset(
                stream=self.stream,
                reference=self.consumer_name
            )
            print(f"Возобновление с offset {offset + 1}")
            return OffsetSpecification.offset(offset + 1)
        except Exception:
            print("Начинаем с начала stream")
            return OffsetSpecification.first()

    async def _handle_message(self, message, context):
        """Обработчик сообщений"""
        if not self.running:
            return

        # Бизнес-логика
        print(f"[{context.offset}] {message.body.decode()}")

        # Запоминаем текущий offset
        self.current_offset = context.offset

        # Периодически сохраняем (каждые 100 сообщений)
        if context.offset % 100 == 0:
            await self._commit_offset()

    async def _commit_offset(self):
        """Сохранить текущий offset"""
        if self.current_offset is not None:
            await self.consumer.store_offset(
                stream=self.stream,
                offset=self.current_offset,
                reference=self.consumer_name
            )

    async def stop(self):
        """Graceful shutdown"""
        print("Остановка consumer...")
        self.running = False

        # Сохраняем последний offset
        await self._commit_offset()

        if self.consumer:
            await self.consumer.close()

        print(f"Consumer остановлен на offset {self.current_offset}")


async def main():
    """Пример использования ResumableConsumer"""

    consumer = ResumableConsumer(
        stream='events',
        consumer_name='order-processor'
    )

    # Обработка сигналов для graceful shutdown
    loop = asyncio.get_event_loop()

    async def shutdown():
        await consumer.stop()
        loop.stop()

    for sig in (signal.SIGINT, signal.SIGTERM):
        loop.add_signal_handler(
            sig,
            lambda: asyncio.create_task(shutdown())
        )

    await consumer.start()

    # Работаем бесконечно
    while consumer.running:
        await asyncio.sleep(1)

asyncio.run(main())
```

## Offset в AMQP (через pika)

### Использование x-stream-offset

```python
import pika

def consume_with_amqp_offset():
    """Чтение stream через AMQP с указанием offset"""

    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Настройка prefetch
    channel.basic_qos(prefetch_count=100)

    def callback(ch, method, properties, body):
        # В AMQP offset доступен через headers
        offset = properties.headers.get('x-stream-offset', 'N/A')
        print(f"[offset={offset}] {body.decode()}")
        ch.basic_ack(delivery_tag=method.delivery_tag)

    # Варианты x-stream-offset:
    # - 'first' - с начала
    # - 'last' - с последнего
    # - 'next' - только новые
    # - offset (int) - конкретный offset
    # - timestamp (int) - UNIX timestamp в секундах

    channel.basic_consume(
        queue='events',
        on_message_callback=callback,
        arguments={
            'x-stream-offset': 'first'  # или число для конкретного offset
        }
    )

    try:
        channel.start_consuming()
    except KeyboardInterrupt:
        channel.stop_consuming()

    connection.close()

consume_with_amqp_offset()
```

## Consumer Groups и координация

### Координация нескольких consumers

```python
import asyncio
from rstream import Consumer, OffsetSpecification

async def coordinated_consumers():
    """
    Пример координации нескольких consumers на одном stream.
    Каждый consumer использует уникальный reference для своего offset.
    """

    async def create_consumer(name: str, reference: str):
        consumer = Consumer(
            host='localhost',
            port=5552,
            username='guest',
            password='guest'
        )
        await consumer.start()

        async def handler(message, context):
            print(f"[{name}] offset={context.offset}: {message.body.decode()}")

            # Каждый consumer сохраняет свой offset
            if context.offset % 50 == 0:
                await consumer.store_offset(
                    stream='events',
                    offset=context.offset,
                    reference=reference
                )

        # Получаем offset для этого consumer
        try:
            offset = await consumer.query_offset(
                stream='events',
                reference=reference
            )
            offset_spec = OffsetSpecification.offset(offset + 1)
        except Exception:
            offset_spec = OffsetSpecification.first()

        await consumer.subscribe(
            stream='events',
            callback=handler,
            offset_specification=offset_spec
        )

        return consumer

    # Создаём несколько независимых consumers
    consumers = await asyncio.gather(
        create_consumer('Analytics', 'analytics-consumer'),
        create_consumer('Monitoring', 'monitoring-consumer'),
        create_consumer('Archiver', 'archiver-consumer')
    )

    # Все читают один stream независимо
    await asyncio.sleep(60)

    for c in consumers:
        await c.close()

asyncio.run(coordinated_consumers())
```

## Best Practices

### 1. Выбор правильного offset type

```python
# Сценарий: Real-time processing (обработка новых событий)
offset_specification=OffsetSpecification.next()

# Сценарий: Replay/Reprocessing (перечитывание всех данных)
offset_specification=OffsetSpecification.first()

# Сценарий: Recovery (восстановление после сбоя)
stored_offset = await consumer.query_offset(stream, reference)
offset_specification=OffsetSpecification.offset(stored_offset + 1)

# Сценарий: Time-based replay (данные за период)
offset_specification=OffsetSpecification.timestamp(timestamp_ms)
```

### 2. Надёжное сохранение offset

```python
# Плохо: сохранять после каждого сообщения
async def handler_bad(message, context):
    process(message)
    await consumer.store_offset(stream, context.offset, ref)  # Медленно!

# Хорошо: batch сохранение
async def handler_good(message, context):
    process(message)
    if context.offset % 100 == 0:  # Каждые 100 сообщений
        await consumer.store_offset(stream, context.offset, ref)
```

### 3. Graceful shutdown

```python
# Всегда сохраняйте offset при остановке
async def graceful_shutdown(consumer, last_offset, reference):
    if last_offset is not None:
        await consumer.store_offset('stream', last_offset, reference)
    await consumer.close()
```

### 4. Уникальные reference для разных consumers

```python
# Каждый consumer должен иметь уникальный reference
# для независимого отслеживания offset

# Consumer 1: обработка заказов
reference='order-processor-v1'

# Consumer 2: аналитика
reference='analytics-pipeline-v1'

# Consumer 3: аудит
reference='audit-log-consumer-v1'
```

## Заключение

Offset tracking в RabbitMQ Streams:

1. **Offset types**: first, last, next, offset(N), timestamp(T)
2. **Server-side storage**: сохранение offset на сервере через store_offset
3. **Query offset**: получение сохранённого offset через query_offset
4. **Resuming**: возобновление чтения с последнего сохранённого offset
5. **Consumer groups**: независимые consumers с разными reference

Правильное использование offset tracking обеспечивает:
- Надёжную доставку сообщений (at-least-once)
- Возможность replay данных
- Независимую работу нескольких consumers
- Восстановление после сбоев
