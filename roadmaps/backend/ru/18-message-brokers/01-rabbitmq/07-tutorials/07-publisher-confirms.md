# Publisher Confirms

[prev: 06-rpc](./06-rpc.md) | [next: 01-streams-overview](../08-streams/01-streams-overview.md)

---

## Введение

В предыдущих туториалах мы использовали `durable=True` и `delivery_mode=Persistent` для сохранения сообщений. Но этого недостаточно для 100% гарантии доставки. **Publisher Confirms** — механизм, при котором RabbitMQ подтверждает получение каждого сообщения.

```
┌──────────────┐                    ┌──────────────┐
│   Producer   │ ─── publish ─────> │   RabbitMQ   │
│              │ <── confirm/nack ─ │              │
└──────────────┘                    └──────────────┘
```

## Проблема без подтверждений

При обычной публикации `basic_publish()` возвращается сразу, не дожидаясь подтверждения от брокера. Сообщение может быть потеряно, если:
- Сеть разорвалась после отправки
- RabbitMQ упал до записи на диск
- Очередь переполнена

## Концепция Publisher Confirms

### Как это работает

1. Producer включает режим подтверждений на канале
2. Producer отправляет сообщение
3. RabbitMQ подтверждает получение (ack) или отклоняет (nack)
4. Producer получает подтверждение и может действовать соответственно

### Типы подтверждений

| Тип | Описание |
|-----|----------|
| `ack` | Сообщение успешно принято брокером |
| `nack` | Сообщение отклонено (ошибка на стороне брокера) |

## Стратегии Publisher Confirms

### Стратегия 1: Синхронные подтверждения (по одному)

Самая простая, но медленная стратегия.

```python
#!/usr/bin/env python
import pika
import time

def publish_individually():
    """Публикация с синхронным подтверждением каждого сообщения."""
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(host='localhost')
    )
    channel = connection.channel()

    # Объявляем очередь
    channel.queue_declare(queue='confirms_queue', durable=True)

    # Включаем режим подтверждений
    channel.confirm_delivery()

    message_count = 100
    start_time = time.time()

    for i in range(message_count):
        body = f"Message {i}"
        try:
            # basic_publish с confirm_delivery вызывает исключение при nack
            channel.basic_publish(
                exchange='',
                routing_key='confirms_queue',
                body=body,
                properties=pika.BasicProperties(
                    delivery_mode=pika.DeliveryMode.Persistent
                ),
                mandatory=True  # Возвращать, если нет подходящей очереди
            )
            # Если мы здесь, значит сообщение подтверждено
            print(f" [x] Sent and confirmed: {body}")
        except pika.exceptions.UnroutableError:
            print(f" [!] Message was returned: {body}")
        except pika.exceptions.NackError:
            print(f" [!] Message was nacked: {body}")

    elapsed = time.time() - start_time
    print(f"\nSent {message_count} messages in {elapsed:.2f}s")
    print(f"Rate: {message_count/elapsed:.0f} msg/s")

    connection.close()


if __name__ == '__main__':
    publish_individually()
```

### Объяснение

1. **confirm_delivery()**: Включает режим подтверждений на канале
2. **mandatory=True**: Сообщение возвращается, если нет подходящей очереди
3. **UnroutableError**: Исключение при возврате сообщения
4. **NackError**: Исключение при отклонении сообщения

## Стратегия 2: Пакетные подтверждения

Отправляем пакет сообщений, затем ждём подтверждения всех.

```python
#!/usr/bin/env python
import pika
import time

def publish_in_batches():
    """Публикация пакетами с подтверждением."""
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(host='localhost')
    )
    channel = connection.channel()

    channel.queue_declare(queue='confirms_queue', durable=True)
    channel.confirm_delivery()

    message_count = 1000
    batch_size = 100
    start_time = time.time()

    outstanding_confirms = []

    for i in range(message_count):
        body = f"Message {i}"

        channel.basic_publish(
            exchange='',
            routing_key='confirms_queue',
            body=body,
            properties=pika.BasicProperties(
                delivery_mode=pika.DeliveryMode.Persistent
            )
        )
        outstanding_confirms.append(body)

        # Каждые batch_size сообщений ждём подтверждения
        if len(outstanding_confirms) >= batch_size:
            try:
                # wait_for_confirms ждёт подтверждения всех отправленных сообщений
                channel.confirm_delivery()
                print(f" [x] Batch of {len(outstanding_confirms)} confirmed")
                outstanding_confirms.clear()
            except Exception as e:
                print(f" [!] Batch confirmation failed: {e}")
                # Здесь нужно повторить отправку

    # Подтверждаем оставшиеся
    if outstanding_confirms:
        channel.confirm_delivery()
        print(f" [x] Final batch of {len(outstanding_confirms)} confirmed")

    elapsed = time.time() - start_time
    print(f"\nSent {message_count} messages in {elapsed:.2f}s")
    print(f"Rate: {message_count/elapsed:.0f} msg/s")

    connection.close()


if __name__ == '__main__':
    publish_in_batches()
```

## Стратегия 3: Асинхронные подтверждения (самая эффективная)

Используем callback для обработки подтверждений.

```python
#!/usr/bin/env python
import pika
import time
import threading
from collections import OrderedDict

class AsyncPublisher:
    def __init__(self):
        self.connection = None
        self.channel = None
        # Словарь для отслеживания неподтверждённых сообщений
        # {delivery_tag: message_body}
        self.outstanding_confirms = OrderedDict()
        self.confirmed_count = 0
        self.nacked_count = 0
        self.lock = threading.Lock()

    def connect(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host='localhost')
        )
        self.channel = self.connection.channel()

        # Объявляем очередь
        self.channel.queue_declare(queue='async_confirms_queue', durable=True)

        # Включаем подтверждения с callback'ами
        self.channel.confirm_delivery()

        # Регистрируем callback'и для подтверждений
        self.channel.add_on_return_callback(self.on_return)

    def on_return(self, channel, method, properties, body):
        """Callback при возврате сообщения (unroutable)."""
        print(f" [!] Message returned: {body.decode()}")

    def publish_message(self, body):
        """Публикация одного сообщения."""
        try:
            self.channel.basic_publish(
                exchange='',
                routing_key='async_confirms_queue',
                body=body,
                properties=pika.BasicProperties(
                    delivery_mode=pika.DeliveryMode.Persistent
                ),
                mandatory=True
            )
            with self.lock:
                self.confirmed_count += 1
            return True
        except pika.exceptions.NackError:
            with self.lock:
                self.nacked_count += 1
            print(f" [!] Message nacked: {body}")
            return False
        except pika.exceptions.UnroutableError:
            print(f" [!] Message unroutable: {body}")
            return False

    def publish_messages(self, count):
        """Публикация множества сообщений."""
        start_time = time.time()

        for i in range(count):
            body = f"Async message {i}"
            self.publish_message(body)

            if (i + 1) % 100 == 0:
                print(f" [x] Published {i + 1} messages")

        elapsed = time.time() - start_time

        print(f"\n=== Results ===")
        print(f"Total sent: {count}")
        print(f"Confirmed: {self.confirmed_count}")
        print(f"Nacked: {self.nacked_count}")
        print(f"Time: {elapsed:.2f}s")
        print(f"Rate: {count/elapsed:.0f} msg/s")

    def close(self):
        if self.connection:
            self.connection.close()


def main():
    publisher = AsyncPublisher()
    publisher.connect()

    try:
        publisher.publish_messages(1000)
    finally:
        publisher.close()


if __name__ == '__main__':
    main()
```

## Продвинутый пример: Publisher с повторными попытками

```python
#!/usr/bin/env python
import pika
import time
import json
from collections import deque

class ReliablePublisher:
    """Publisher с гарантированной доставкой и повторными попытками."""

    def __init__(self, max_retries=3, retry_delay=1.0):
        self.max_retries = max_retries
        self.retry_delay = retry_delay
        self.connection = None
        self.channel = None
        self.failed_messages = deque()
        self.stats = {
            'sent': 0,
            'confirmed': 0,
            'retried': 0,
            'failed': 0
        }

    def connect(self):
        """Установка соединения."""
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(
                host='localhost',
                heartbeat=600,
                blocked_connection_timeout=300
            )
        )
        self.channel = self.connection.channel()
        self.channel.queue_declare(queue='reliable_queue', durable=True)
        self.channel.confirm_delivery()

    def publish(self, message, routing_key='reliable_queue'):
        """Публикация с повторными попытками."""
        body = json.dumps(message) if isinstance(message, dict) else str(message)

        for attempt in range(self.max_retries):
            try:
                self.channel.basic_publish(
                    exchange='',
                    routing_key=routing_key,
                    body=body,
                    properties=pika.BasicProperties(
                        delivery_mode=pika.DeliveryMode.Persistent,
                        content_type='application/json'
                    ),
                    mandatory=True
                )
                self.stats['sent'] += 1
                self.stats['confirmed'] += 1
                return True

            except pika.exceptions.NackError:
                self.stats['retried'] += 1
                print(f" [!] Nack, attempt {attempt + 1}/{self.max_retries}")
                time.sleep(self.retry_delay)

            except pika.exceptions.UnroutableError:
                print(f" [!] Unroutable: {body[:50]}...")
                self.stats['failed'] += 1
                return False

            except pika.exceptions.AMQPConnectionError:
                print(" [!] Connection lost, reconnecting...")
                self.reconnect()
                self.stats['retried'] += 1

        # Все попытки исчерпаны
        self.stats['failed'] += 1
        self.failed_messages.append(message)
        print(f" [X] Failed after {self.max_retries} attempts: {body[:50]}...")
        return False

    def reconnect(self):
        """Переподключение при потере соединения."""
        try:
            self.connection.close()
        except:
            pass

        time.sleep(self.retry_delay)
        self.connect()

    def publish_batch(self, messages):
        """Публикация пакета сообщений."""
        results = []
        for msg in messages:
            success = self.publish(msg)
            results.append(success)
        return results

    def get_failed_messages(self):
        """Получение списка неотправленных сообщений."""
        return list(self.failed_messages)

    def retry_failed(self):
        """Повторная отправка неудачных сообщений."""
        failed = list(self.failed_messages)
        self.failed_messages.clear()

        print(f" [*] Retrying {len(failed)} failed messages...")
        for msg in failed:
            self.publish(msg)

    def get_stats(self):
        """Получение статистики."""
        return self.stats.copy()

    def close(self):
        if self.connection:
            self.connection.close()


def main():
    publisher = ReliablePublisher(max_retries=3, retry_delay=0.5)
    publisher.connect()

    try:
        # Отправляем тестовые сообщения
        messages = [
            {'id': i, 'data': f'Test message {i}'}
            for i in range(100)
        ]

        print("Publishing messages...")
        start = time.time()

        for msg in messages:
            publisher.publish(msg)

        elapsed = time.time() - start

        # Выводим статистику
        stats = publisher.get_stats()
        print(f"\n=== Statistics ===")
        print(f"Sent: {stats['sent']}")
        print(f"Confirmed: {stats['confirmed']}")
        print(f"Retried: {stats['retried']}")
        print(f"Failed: {stats['failed']}")
        print(f"Time: {elapsed:.2f}s")
        print(f"Rate: {stats['sent']/elapsed:.0f} msg/s")

        # Проверяем неудачные сообщения
        failed = publisher.get_failed_messages()
        if failed:
            print(f"\nFailed messages: {len(failed)}")
            publisher.retry_failed()

    finally:
        publisher.close()


if __name__ == '__main__':
    main()
```

## Сравнение стратегий

| Стратегия | Throughput | Сложность | Надёжность |
|-----------|------------|-----------|------------|
| Синхронная (по одному) | Низкий (~100 msg/s) | Простая | Высокая |
| Пакетная | Средний (~1000 msg/s) | Средняя | Высокая |
| Асинхронная | Высокий (~10000 msg/s) | Сложная | Высокая |

## Когда что использовать

### Синхронные подтверждения
- Критически важные сообщения (финансовые транзакции)
- Низкий объём сообщений
- Простота важнее производительности

### Пакетные подтверждения
- Баланс между надёжностью и производительностью
- Обработка ошибок на уровне пакета

### Асинхронные подтверждения
- Высокий throughput
- Сложная логика обработки ошибок
- Production-системы с большой нагрузкой

## Consumer Acknowledgements vs Publisher Confirms

| Аспект | Consumer Ack | Publisher Confirm |
|--------|--------------|-------------------|
| Кто подтверждает | Consumer -> Broker | Broker -> Publisher |
| Что подтверждает | Обработка сообщения | Получение сообщения |
| Направление | Downstream | Upstream |

## Полная цепочка надёжности

Для 100% гарантии доставки нужно:

1. **Publisher Confirms** — подтверждение от брокера
2. **Durable Queue** — очередь сохраняется при перезапуске
3. **Persistent Messages** — сообщения записываются на диск
4. **Consumer Acknowledgements** — подтверждение обработки

```python
# Полный пример надёжной публикации
channel.queue_declare(queue='reliable', durable=True)  # Durable queue
channel.confirm_delivery()  # Publisher confirms

channel.basic_publish(
    exchange='',
    routing_key='reliable',
    body='Important message',
    properties=pika.BasicProperties(
        delivery_mode=pika.DeliveryMode.Persistent  # Persistent message
    )
)
```

## CLI команды для отладки

```bash
# Статистика подтверждений
rabbitmqctl list_queues name messages_ready messages_unacknowledged

# Информация о соединениях
rabbitmqctl list_connections name channels confirm

# Детальная информация о каналах
rabbitmqctl list_channels name confirm
```

## Частые ошибки

### Не включены подтверждения

```python
# НЕПРАВИЛЬНО - подтверждения не включены
channel.basic_publish(exchange='', routing_key='q', body='msg')

# ПРАВИЛЬНО
channel.confirm_delivery()
channel.basic_publish(exchange='', routing_key='q', body='msg')
```

### Игнорирование исключений

```python
# НЕПРАВИЛЬНО
try:
    channel.basic_publish(...)
except:
    pass  # Игнорируем ошибку

# ПРАВИЛЬНО
try:
    channel.basic_publish(...)
except pika.exceptions.NackError:
    # Повторить или сохранить для повторной отправки
    retry_queue.append(message)
```

## Резюме

В этом туториале мы изучили:
1. **Publisher Confirms** — механизм подтверждения публикации
2. **Три стратегии**: синхронная, пакетная, асинхронная
3. **Обработка ошибок** и повторные попытки
4. **Полная цепочка надёжности** для гарантированной доставки

Это завершает серию туториалов RabbitMQ. Теперь вы знаете все основные паттерны работы с очередями сообщений.

---

[prev: 06-rpc](./06-rpc.md) | [next: 01-streams-overview](../08-streams/01-streams-overview.md)
