# Consumers (Потребители)

## Что такое Consumer?

**Consumer (Потребитель)** — это приложение или компонент, который получает и обрабатывает сообщения из очередей RabbitMQ. Consumer подписывается на одну или несколько очередей и получает сообщения по мере их поступления.

## Архитектура Consumer

```
┌─────────────┐     ┌──────────┐     ┌─────────┐     ┌───────────┐
│  Producer   │────▶│ Exchange │────▶│  Queue  │────▶│  Consumer │
└─────────────┘     └──────────┘     └─────────┘     └───────────┘
                                           │
                                           ▼
                                    ┌───────────┐
                                    │  Consumer │
                                    └───────────┘
```

Consumer отвечает за:
- Подключение к RabbitMQ
- Подписку на очереди
- Получение сообщений
- Обработку сообщений
- Отправку подтверждений (acknowledgements)

## Базовый пример Consumer

```python
import pika

# Установка соединения
connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Объявление очереди (должна существовать)
channel.queue_declare(queue='hello')

# Callback-функция для обработки сообщений
def callback(ch, method, properties, body):
    print(f" [x] Received {body.decode()}")

# Подписка на очередь
channel.basic_consume(
    queue='hello',
    on_message_callback=callback,
    auto_ack=True  # Автоматическое подтверждение
)

print(' [*] Waiting for messages. To exit press CTRL+C')
channel.start_consuming()
```

## Acknowledgements (Подтверждения)

Подтверждения гарантируют, что сообщение было успешно обработано.

### Ручные подтверждения (Manual Ack)

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

channel.queue_declare(queue='tasks', durable=True)

def callback(ch, method, properties, body):
    try:
        print(f" [x] Processing: {body.decode()}")
        # Обработка сообщения...

        # Подтверждение успешной обработки
        ch.basic_ack(delivery_tag=method.delivery_tag)
        print(" [x] Done")

    except Exception as e:
        print(f" [!] Error: {e}")
        # Отклонение с возвратом в очередь
        ch.basic_nack(
            delivery_tag=method.delivery_tag,
            requeue=True
        )

channel.basic_consume(
    queue='tasks',
    on_message_callback=callback,
    auto_ack=False  # Ручные подтверждения
)

channel.start_consuming()
```

### Типы подтверждений

```python
# Положительное подтверждение (сообщение обработано)
ch.basic_ack(delivery_tag=method.delivery_tag)

# Множественное подтверждение (все сообщения до этого)
ch.basic_ack(delivery_tag=method.delivery_tag, multiple=True)

# Отрицательное подтверждение (возврат в очередь)
ch.basic_nack(
    delivery_tag=method.delivery_tag,
    requeue=True  # Вернуть в очередь для повторной обработки
)

# Отклонение сообщения (удаление или dead letter)
ch.basic_reject(
    delivery_tag=method.delivery_tag,
    requeue=False  # Не возвращать в очередь
)
```

## Prefetch (Предварительная выборка)

Prefetch контролирует количество неподтверждённых сообщений, которые consumer может получить.

### Настройка QoS (Quality of Service)

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Prefetch = 1: consumer получает только одно сообщение за раз
channel.basic_qos(prefetch_count=1)

# Prefetch = 10: consumer может иметь до 10 необработанных сообщений
channel.basic_qos(prefetch_count=10)

# Global prefetch: применяется ко всем consumers на канале
channel.basic_qos(prefetch_count=10, global_qos=True)

def callback(ch, method, properties, body):
    # Долгая обработка
    process_message(body)
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(
    queue='tasks',
    on_message_callback=callback
)

channel.start_consuming()
```

### Выбор значения Prefetch

```python
# Для CPU-интенсивных задач
channel.basic_qos(prefetch_count=1)  # Fair dispatch

# Для I/O-интенсивных задач
channel.basic_qos(prefetch_count=10)  # Буферизация

# Для очень быстрой обработки
channel.basic_qos(prefetch_count=50)  # Высокая пропускная способность
```

## Конкурентные Consumers

Несколько consumers могут обрабатывать сообщения из одной очереди параллельно.

### Round-Robin Distribution

```python
# Consumer 1 (worker1.py)
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

channel.queue_declare(queue='tasks', durable=True)
channel.basic_qos(prefetch_count=1)

def callback(ch, method, properties, body):
    print(f" [Worker 1] Processing: {body.decode()}")
    # Обработка...
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(queue='tasks', on_message_callback=callback)
print(' [Worker 1] Waiting for messages...')
channel.start_consuming()
```

```python
# Consumer 2 (worker2.py) - аналогичный код
```

### Многопоточный Consumer

```python
import pika
import threading
from concurrent.futures import ThreadPoolExecutor

class ThreadedConsumer:
    def __init__(self, queue_name: str, num_workers: int = 4):
        self.queue_name = queue_name
        self.num_workers = num_workers
        self.executor = ThreadPoolExecutor(max_workers=num_workers)

        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()
        self.channel.queue_declare(queue=queue_name, durable=True)
        self.channel.basic_qos(prefetch_count=num_workers)

    def process_message(self, body: bytes) -> bool:
        """Обработка сообщения в отдельном потоке"""
        try:
            message = body.decode()
            print(f" [x] Processing in thread: {message}")
            # Симуляция работы
            import time
            time.sleep(1)
            return True
        except Exception as e:
            print(f" [!] Error: {e}")
            return False

    def callback(self, ch, method, properties, body):
        """Callback с делегированием в thread pool"""
        def task():
            success = self.process_message(body)
            if success:
                self.connection.add_callback_threadsafe(
                    lambda: ch.basic_ack(delivery_tag=method.delivery_tag)
                )
            else:
                self.connection.add_callback_threadsafe(
                    lambda: ch.basic_nack(
                        delivery_tag=method.delivery_tag,
                        requeue=True
                    )
                )

        self.executor.submit(task)

    def start(self):
        """Запуск consumer"""
        self.channel.basic_consume(
            queue=self.queue_name,
            on_message_callback=self.callback
        )
        print(f' [*] Starting consumer with {self.num_workers} workers')
        self.channel.start_consuming()

    def stop(self):
        """Остановка consumer"""
        self.channel.stop_consuming()
        self.executor.shutdown(wait=True)
        self.connection.close()


if __name__ == '__main__':
    consumer = ThreadedConsumer('tasks', num_workers=4)
    try:
        consumer.start()
    except KeyboardInterrupt:
        consumer.stop()
```

## Обработка ошибок и Retry

### Retry с экспоненциальной задержкой

```python
import pika
import json
import time

def callback(ch, method, properties, body):
    message = json.loads(body)
    retry_count = properties.headers.get('x-retry-count', 0) if properties.headers else 0
    max_retries = 3

    try:
        # Обработка сообщения
        process_message(message)
        ch.basic_ack(delivery_tag=method.delivery_tag)

    except Exception as e:
        print(f" [!] Error processing message: {e}")

        if retry_count < max_retries:
            # Повторная публикация с увеличенным счётчиком
            delay = 2 ** retry_count  # Экспоненциальная задержка
            time.sleep(delay)

            new_properties = pika.BasicProperties(
                delivery_mode=2,
                headers={'x-retry-count': retry_count + 1}
            )

            ch.basic_publish(
                exchange='',
                routing_key=method.routing_key,
                body=body,
                properties=new_properties
            )

            ch.basic_ack(delivery_tag=method.delivery_tag)
            print(f" [x] Requeued message (attempt {retry_count + 1})")
        else:
            # Отправка в Dead Letter Queue
            ch.basic_reject(
                delivery_tag=method.delivery_tag,
                requeue=False
            )
            print(" [!] Message moved to DLQ")
```

### Dead Letter Queue (DLQ)

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Объявление DLQ
channel.queue_declare(queue='tasks_dlq', durable=True)

# Объявление основной очереди с DLX
channel.queue_declare(
    queue='tasks',
    durable=True,
    arguments={
        'x-dead-letter-exchange': '',
        'x-dead-letter-routing-key': 'tasks_dlq'
    }
)

# Consumer для обработки "мёртвых" сообщений
def dlq_callback(ch, method, properties, body):
    print(f" [DLQ] Processing failed message: {body.decode()}")
    # Логирование, уведомление, ручная обработка
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(
    queue='tasks_dlq',
    on_message_callback=dlq_callback
)

channel.start_consuming()
```

## Graceful Shutdown

```python
import pika
import signal
import sys

class GracefulConsumer:
    def __init__(self, queue_name: str):
        self.queue_name = queue_name
        self.should_stop = False
        self.connection = None
        self.channel = None

        # Регистрация обработчиков сигналов
        signal.signal(signal.SIGINT, self.signal_handler)
        signal.signal(signal.SIGTERM, self.signal_handler)

    def signal_handler(self, signum, frame):
        print("\n [*] Received shutdown signal, stopping gracefully...")
        self.should_stop = True
        if self.channel:
            self.channel.stop_consuming()

    def connect(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()
        self.channel.queue_declare(queue=self.queue_name, durable=True)
        self.channel.basic_qos(prefetch_count=1)

    def callback(self, ch, method, properties, body):
        if self.should_stop:
            # Не принимаем новые сообщения
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)
            return

        try:
            print(f" [x] Processing: {body.decode()}")
            # Обработка...
            ch.basic_ack(delivery_tag=method.delivery_tag)
        except Exception as e:
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)

    def start(self):
        self.connect()
        self.channel.basic_consume(
            queue=self.queue_name,
            on_message_callback=self.callback
        )
        print(' [*] Waiting for messages...')

        try:
            self.channel.start_consuming()
        finally:
            if self.connection and self.connection.is_open:
                self.connection.close()
            print(' [*] Shutdown complete')


if __name__ == '__main__':
    consumer = GracefulConsumer('tasks')
    consumer.start()
```

## Consumer с SelectConnection (Асинхронный)

```python
import pika
from pika.adapters.select_connection import SelectConnection

class AsyncConsumer:
    def __init__(self, queue_name: str):
        self.queue_name = queue_name
        self.connection = None
        self.channel = None

    def connect(self):
        parameters = pika.ConnectionParameters('localhost')
        self.connection = SelectConnection(
            parameters,
            on_open_callback=self.on_connection_open,
            on_open_error_callback=self.on_connection_error,
            on_close_callback=self.on_connection_closed
        )

    def on_connection_open(self, connection):
        print(' [*] Connection opened')
        connection.channel(on_open_callback=self.on_channel_open)

    def on_connection_error(self, connection, error):
        print(f' [!] Connection error: {error}')

    def on_connection_closed(self, connection, reason):
        print(f' [*] Connection closed: {reason}')

    def on_channel_open(self, channel):
        print(' [*] Channel opened')
        self.channel = channel
        channel.queue_declare(
            queue=self.queue_name,
            durable=True,
            callback=self.on_queue_declared
        )

    def on_queue_declared(self, frame):
        print(f' [*] Queue {self.queue_name} declared')
        self.channel.basic_qos(prefetch_count=1)
        self.channel.basic_consume(
            queue=self.queue_name,
            on_message_callback=self.on_message
        )

    def on_message(self, channel, method, properties, body):
        print(f' [x] Received: {body.decode()}')
        channel.basic_ack(delivery_tag=method.delivery_tag)

    def run(self):
        self.connect()
        try:
            self.connection.ioloop.start()
        except KeyboardInterrupt:
            self.connection.close()
            self.connection.ioloop.start()


if __name__ == '__main__':
    consumer = AsyncConsumer('tasks')
    consumer.run()
```

## Best Practices для Consumers

### 1. Всегда используйте ручные acknowledgements

```python
channel.basic_consume(
    queue='tasks',
    on_message_callback=callback,
    auto_ack=False  # ВАЖНО!
)
```

### 2. Подтверждайте после успешной обработки

```python
def callback(ch, method, properties, body):
    try:
        process(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)
```

### 3. Настраивайте prefetch под нагрузку

```python
# Для долгих задач
channel.basic_qos(prefetch_count=1)

# Для быстрых задач
channel.basic_qos(prefetch_count=10)
```

### 4. Реализуйте graceful shutdown

```python
signal.signal(signal.SIGTERM, shutdown_handler)
```

### 5. Логируйте всё

```python
import logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)
```

### 6. Используйте Dead Letter Queue

```python
arguments={
    'x-dead-letter-exchange': 'dlx',
    'x-dead-letter-routing-key': 'dlq'
}
```

### 7. Обрабатывайте дубликаты (идемпотентность)

```python
def callback(ch, method, properties, body):
    message_id = properties.message_id
    if is_already_processed(message_id):
        ch.basic_ack(delivery_tag=method.delivery_tag)
        return

    process(body)
    mark_as_processed(message_id)
    ch.basic_ack(delivery_tag=method.delivery_tag)
```

## Типичные ошибки

1. **Использование auto_ack=True** — потеря сообщений при сбое
2. **Слишком большой prefetch** — неравномерное распределение нагрузки
3. **Отсутствие обработки ошибок** — бесконечные циклы или потеря данных
4. **Блокирующие операции в callback** — снижение производительности
5. **Игнорирование graceful shutdown** — потеря обрабатываемых сообщений

## Заключение

Consumer — критически важный компонент в архитектуре RabbitMQ. Правильная реализация включает:
- Ручные подтверждения (acknowledgements)
- Настройку prefetch для оптимальной производительности
- Обработку ошибок и retry логику
- Graceful shutdown
- Масштабирование через конкурентных consumers

Следуя этим практикам, вы обеспечите надёжную и эффективную обработку сообщений.
