# Publisher Confirms

## Введение

Publisher Confirms - это механизм RabbitMQ, который позволяет издателю (publisher) получать подтверждение о том, что сообщение было успешно принято брокером. Это дополняет механизм consumer acknowledgments и обеспечивает сквозную надежность доставки сообщений.

## Зачем нужны Publisher Confirms?

Без подтверждений издатель не знает, дошло ли сообщение до брокера:
- Сетевой сбой между издателем и брокером
- Брокер переполнен и отклоняет сообщения
- Очередь не существует или routing key неверный

```
Publisher  ------>  RabbitMQ  ------>  Consumer
    |                  |                  |
    |   confirm/nack   |                  |
    |<-----------------|                  |
```

## Включение Publisher Confirms

### Активация режима подтверждений

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Включаем режим подтверждений для канала
channel.confirm_delivery()

print("Publisher confirms активированы")
```

## Синхронные подтверждения

### Простой пример с ожиданием подтверждения

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(queue='confirmed_queue', durable=True)
channel.confirm_delivery()

def publish_with_confirm(message):
    """Публикация с синхронным ожиданием подтверждения"""
    try:
        channel.basic_publish(
            exchange='',
            routing_key='confirmed_queue',
            body=message,
            properties=pika.BasicProperties(
                delivery_mode=2  # Persistent
            ),
            mandatory=True  # Вернет ошибку если очередь не найдена
        )
        print(f"Сообщение '{message}' подтверждено брокером")
        return True
    except pika.exceptions.UnroutableError:
        print(f"Сообщение '{message}' не удалось доставить")
        return False
    except pika.exceptions.NackError:
        print(f"Сообщение '{message}' отклонено брокером")
        return False

# Публикация сообщений
for i in range(5):
    publish_with_confirm(f"Message {i}")
```

### Batch подтверждения

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(queue='batch_queue', durable=True)
channel.confirm_delivery()

BATCH_SIZE = 100
messages = [f"Message {i}" for i in range(500)]

def publish_batch(messages, batch_size):
    """Публикация пакетами с подтверждением каждого пакета"""
    confirmed = 0
    failed = 0

    for i in range(0, len(messages), batch_size):
        batch = messages[i:i + batch_size]

        try:
            for message in batch:
                channel.basic_publish(
                    exchange='',
                    routing_key='batch_queue',
                    body=message,
                    properties=pika.BasicProperties(delivery_mode=2)
                )

            # Ждем подтверждения всего пакета
            channel.wait_for_pending_acks(timeout=10)
            confirmed += len(batch)
            print(f"Пакет {i // batch_size + 1} подтвержден")

        except Exception as e:
            failed += len(batch)
            print(f"Пакет {i // batch_size + 1} не подтвержден: {e}")

    return confirmed, failed

confirmed, failed = publish_batch(messages, BATCH_SIZE)
print(f"Подтверждено: {confirmed}, Не удалось: {failed}")
```

## Асинхронные подтверждения

### Использование callback для подтверждений

```python
import pika
from pika.adapters.blocking_connection import BlockingChannel
from collections import defaultdict
import threading

class AsyncPublisher:
    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()
        self.channel.queue_declare(queue='async_queue', durable=True)

        # Хранилище неподтвержденных сообщений
        self.unconfirmed = {}
        self.sequence_number = 0
        self.lock = threading.Lock()

        # Включаем подтверждения
        self.channel.confirm_delivery()

    def publish(self, message):
        """Асинхронная публикация с отслеживанием"""
        with self.lock:
            self.sequence_number += 1
            seq = self.sequence_number
            self.unconfirmed[seq] = message

        try:
            self.channel.basic_publish(
                exchange='',
                routing_key='async_queue',
                body=message,
                properties=pika.BasicProperties(delivery_mode=2)
            )
            return seq
        except Exception as e:
            with self.lock:
                del self.unconfirmed[seq]
            raise e

    def wait_for_confirms(self, timeout=30):
        """Ожидание подтверждения всех сообщений"""
        try:
            self.channel.wait_for_pending_acks(timeout=timeout)
            with self.lock:
                confirmed = list(self.unconfirmed.keys())
                self.unconfirmed.clear()
            return confirmed, []
        except pika.exceptions.NackError as e:
            with self.lock:
                failed = list(self.unconfirmed.keys())
                self.unconfirmed.clear()
            return [], failed

    def close(self):
        self.connection.close()


# Использование
publisher = AsyncPublisher()

# Публикуем много сообщений быстро
for i in range(100):
    publisher.publish(f"Async message {i}")

# Ждем подтверждения всех
confirmed, failed = publisher.wait_for_confirms()
print(f"Подтверждено: {len(confirmed)}, Отклонено: {len(failed)}")

publisher.close()
```

### Асинхронный publisher с SelectConnection

```python
import pika
from pika import SelectConnection
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class AsyncConfirmPublisher:
    def __init__(self, amqp_url):
        self.amqp_url = amqp_url
        self.connection = None
        self.channel = None
        self.deliveries = {}
        self.acked = 0
        self.nacked = 0
        self.message_number = 0

    def connect(self):
        """Создание асинхронного соединения"""
        return SelectConnection(
            pika.URLParameters(self.amqp_url),
            on_open_callback=self.on_connection_open,
            on_open_error_callback=self.on_connection_error,
            on_close_callback=self.on_connection_closed
        )

    def on_connection_open(self, connection):
        logger.info("Соединение установлено")
        self.connection = connection
        self.connection.channel(on_open_callback=self.on_channel_open)

    def on_connection_error(self, connection, error):
        logger.error(f"Ошибка соединения: {error}")

    def on_connection_closed(self, connection, reason):
        logger.info(f"Соединение закрыто: {reason}")

    def on_channel_open(self, channel):
        logger.info("Канал открыт")
        self.channel = channel

        # Включаем publisher confirms
        self.channel.confirm_delivery(
            ack_nack_callback=self.on_delivery_confirmation
        )

        # Объявляем очередь и начинаем публикацию
        self.channel.queue_declare(
            queue='async_confirmed',
            durable=True,
            callback=self.on_queue_declared
        )

    def on_queue_declared(self, frame):
        logger.info("Очередь объявлена, начинаем публикацию")
        self.publish_messages()

    def on_delivery_confirmation(self, frame):
        """Обработка подтверждений от брокера"""
        confirmation_type = frame.method.NAME.split('.')[1].lower()
        delivery_tag = frame.method.delivery_tag
        multiple = frame.method.multiple

        if confirmation_type == 'ack':
            self.acked += 1
            logger.info(f"ACK для сообщения {delivery_tag}")
        else:
            self.nacked += 1
            logger.warning(f"NACK для сообщения {delivery_tag}")

        # Удаляем подтвержденные сообщения
        if multiple:
            # Удаляем все до delivery_tag включительно
            to_remove = [k for k in self.deliveries if k <= delivery_tag]
            for key in to_remove:
                del self.deliveries[key]
        elif delivery_tag in self.deliveries:
            del self.deliveries[delivery_tag]

        logger.info(f"ACKed: {self.acked}, NACKed: {self.nacked}, "
                   f"Ожидают: {len(self.deliveries)}")

    def publish_messages(self):
        """Публикация сообщений"""
        for i in range(100):
            self.message_number += 1
            message = f"Message #{self.message_number}"

            self.channel.basic_publish(
                exchange='',
                routing_key='async_confirmed',
                body=message,
                properties=pika.BasicProperties(
                    delivery_mode=2,
                    content_type='text/plain'
                )
            )

            self.deliveries[self.message_number] = message
            logger.debug(f"Опубликовано: {message}")

    def run(self):
        """Запуск event loop"""
        connection = self.connect()
        try:
            connection.ioloop.start()
        except KeyboardInterrupt:
            connection.close()
            connection.ioloop.start()


# Запуск
# publisher = AsyncConfirmPublisher('amqp://guest:guest@localhost:5672/%2F')
# publisher.run()
```

## Обработка ошибок и повторные попытки

### Стратегия retry с экспоненциальной задержкой

```python
import pika
import time
import random

class ReliablePublisher:
    def __init__(self, host='localhost'):
        self.host = host
        self.max_retries = 5
        self.base_delay = 1
        self.max_delay = 60
        self.connection = None
        self.channel = None
        self._connect()

    def _connect(self):
        """Установка соединения с повторными попытками"""
        for attempt in range(self.max_retries):
            try:
                self.connection = pika.BlockingConnection(
                    pika.ConnectionParameters(self.host)
                )
                self.channel = self.connection.channel()
                self.channel.confirm_delivery()
                return
            except pika.exceptions.AMQPConnectionError as e:
                delay = min(
                    self.base_delay * (2 ** attempt) + random.uniform(0, 1),
                    self.max_delay
                )
                print(f"Попытка {attempt + 1}/{self.max_retries} не удалась. "
                      f"Повтор через {delay:.1f}с")
                time.sleep(delay)
        raise Exception("Не удалось подключиться к RabbitMQ")

    def publish(self, queue, message, persistent=True):
        """Надежная публикация с повторными попытками"""
        for attempt in range(self.max_retries):
            try:
                if self.connection.is_closed:
                    self._connect()

                self.channel.queue_declare(queue=queue, durable=True)

                self.channel.basic_publish(
                    exchange='',
                    routing_key=queue,
                    body=message,
                    properties=pika.BasicProperties(
                        delivery_mode=2 if persistent else 1
                    ),
                    mandatory=True
                )

                return True

            except pika.exceptions.UnroutableError:
                print(f"Сообщение не маршрутизируемо")
                return False

            except pika.exceptions.NackError:
                delay = self.base_delay * (2 ** attempt)
                print(f"NACK, повтор через {delay}с")
                time.sleep(delay)

            except pika.exceptions.AMQPError as e:
                print(f"Ошибка AMQP: {e}")
                self._connect()

        return False

    def close(self):
        if self.connection and not self.connection.is_closed:
            self.connection.close()


# Использование
publisher = ReliablePublisher()

messages = ["важное сообщение 1", "важное сообщение 2", "важное сообщение 3"]

for msg in messages:
    if publisher.publish('reliable_queue', msg):
        print(f"Доставлено: {msg}")
    else:
        print(f"Не удалось доставить: {msg}")

publisher.close()
```

## Мониторинг подтверждений

### Сбор метрик

```python
import pika
import time
from dataclasses import dataclass, field
from typing import Dict, List

@dataclass
class PublishMetrics:
    total_published: int = 0
    total_confirmed: int = 0
    total_nacked: int = 0
    publish_times: List[float] = field(default_factory=list)
    confirm_times: List[float] = field(default_factory=list)

    @property
    def confirmation_rate(self):
        if self.total_published == 0:
            return 0
        return self.total_confirmed / self.total_published * 100

    @property
    def avg_publish_time(self):
        if not self.publish_times:
            return 0
        return sum(self.publish_times) / len(self.publish_times)

    @property
    def avg_confirm_time(self):
        if not self.confirm_times:
            return 0
        return sum(self.confirm_times) / len(self.confirm_times)


class MeteredPublisher:
    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()
        self.channel.confirm_delivery()
        self.metrics = PublishMetrics()
        self.pending: Dict[int, float] = {}
        self.seq = 0

    def publish(self, queue, message):
        self.seq += 1
        start_time = time.time()

        try:
            self.channel.basic_publish(
                exchange='',
                routing_key=queue,
                body=message,
                properties=pika.BasicProperties(delivery_mode=2)
            )

            publish_time = time.time() - start_time
            self.metrics.publish_times.append(publish_time)
            self.metrics.total_published += 1

            # Ждем подтверждения
            confirm_start = time.time()
            # В BlockingConnection подтверждение происходит автоматически
            confirm_time = time.time() - confirm_start

            self.metrics.total_confirmed += 1
            self.metrics.confirm_times.append(confirm_time)

            return True

        except pika.exceptions.NackError:
            self.metrics.total_nacked += 1
            return False

    def print_metrics(self):
        print("\n=== Publisher Metrics ===")
        print(f"Опубликовано: {self.metrics.total_published}")
        print(f"Подтверждено: {self.metrics.total_confirmed}")
        print(f"Отклонено: {self.metrics.total_nacked}")
        print(f"Процент подтверждений: {self.metrics.confirmation_rate:.1f}%")
        print(f"Среднее время публикации: {self.metrics.avg_publish_time*1000:.2f}мс")
        print(f"Среднее время подтверждения: {self.metrics.avg_confirm_time*1000:.2f}мс")


# Тестирование
publisher = MeteredPublisher()
publisher.channel.queue_declare(queue='metered_queue', durable=True)

for i in range(100):
    publisher.publish('metered_queue', f"Message {i}")

publisher.print_metrics()
```

## Best Practices

### 1. Всегда включайте confirms для важных сообщений

```python
# Правильно
channel.confirm_delivery()
channel.basic_publish(...)

# Неправильно для критичных данных
channel.basic_publish(...)  # Нет гарантии доставки
```

### 2. Используйте батчинг для производительности

```python
# Для большого количества сообщений
BATCH_SIZE = 100

for i, message in enumerate(messages):
    channel.basic_publish(exchange='', routing_key='queue', body=message)

    if (i + 1) % BATCH_SIZE == 0:
        channel.wait_for_pending_acks(timeout=10)
```

### 3. Обрабатывайте nack правильно

```python
try:
    channel.basic_publish(...)
except pika.exceptions.NackError as e:
    # Сохраните сообщение для повторной отправки
    failed_messages.append(message)
    # Логируйте ошибку
    logger.error(f"Сообщение отклонено: {message}")
```

### 4. Комбинируйте с mandatory флагом

```python
# Mandatory гарантирует, что сообщение попадет в очередь
channel.basic_publish(
    exchange='',
    routing_key='queue',
    body=message,
    mandatory=True  # Вернет ошибку если очередь не найдена
)
```

## Частые ошибки

### 1. Игнорирование nack

```python
# НЕПРАВИЛЬНО
channel.basic_publish(...)  # Игнорируем результат

# ПРАВИЛЬНО
try:
    channel.basic_publish(...)
except pika.exceptions.NackError:
    handle_failed_message(message)
```

### 2. Слишком большие батчи

```python
# НЕПРАВИЛЬНО - слишком большой батч
for message in million_messages:
    channel.basic_publish(...)
channel.wait_for_pending_acks()  # Долгое ожидание

# ПРАВИЛЬНО - разумный размер батча
for i, message in enumerate(million_messages):
    channel.basic_publish(...)
    if i % 1000 == 0:
        channel.wait_for_pending_acks(timeout=30)
```

### 3. Отсутствие таймаута

```python
# НЕПРАВИЛЬНО - может зависнуть навсегда
channel.wait_for_pending_acks()

# ПРАВИЛЬНО
try:
    channel.wait_for_pending_acks(timeout=30)
except pika.exceptions.ChannelError:
    handle_timeout()
```

## Заключение

Publisher Confirms - важный механизм для построения надежных систем обмена сообщениями. В сочетании с consumer acknowledgments и persistent messages он обеспечивает сквозную гарантию доставки сообщений от издателя до потребителя.
