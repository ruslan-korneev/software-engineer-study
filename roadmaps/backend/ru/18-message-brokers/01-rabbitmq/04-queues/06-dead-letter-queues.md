# Dead Letter Queues

## Введение

Dead Letter Queue (DLQ) — это механизм обработки сообщений, которые не могут быть успешно доставлены или обработаны. Вместо потери таких сообщений, RabbitMQ перенаправляет их в специальный Dead Letter Exchange (DLX), откуда они попадают в очередь для последующего анализа или повторной обработки.

## Что такое Dead Letter

### Определение

"Мёртвое письмо" (Dead Letter) — это сообщение, которое не может быть обработано по одной из причин:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Причины Dead Letter                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. REJECTED/NACKED                                             │
│     Сообщение явно отклонено consumer'ом                        │
│     с requeue=false                                             │
│                                                                 │
│  2. TTL EXPIRED                                                 │
│     Истекло время жизни сообщения                               │
│     (x-message-ttl или per-message expiration)                  │
│                                                                 │
│  3. QUEUE OVERFLOW                                              │
│     Очередь переполнена, новые сообщения                        │
│     отбрасываются (x-overflow: reject-publish-dlx)              │
│                                                                 │
│  4. DELIVERY LIMIT (Quorum Queues)                              │
│     Превышено количество попыток доставки                       │
│     (x-delivery-limit)                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Настройка Dead Letter Exchange

### Базовая настройка

```python
import pika

def setup_dead_letter_infrastructure():
    """
    Создание базовой инфраструктуры DLX.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # 1. Создаём Dead Letter Exchange
    channel.exchange_declare(
        exchange='dlx',
        exchange_type='direct'
    )

    # 2. Создаём Dead Letter Queue
    channel.queue_declare(
        queue='dead_letters',
        durable=True
    )

    # 3. Привязываем DLQ к DLX
    channel.queue_bind(
        queue='dead_letters',
        exchange='dlx',
        routing_key='dead'
    )

    # 4. Создаём основную очередь с DLX
    channel.queue_declare(
        queue='main_queue',
        durable=True,
        arguments={
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'dead'
        }
    )

    print("DLX инфраструктура создана")
    connection.close()

setup_dead_letter_infrastructure()
```

### Полная настройка с TTL

```python
import pika

def complete_dlx_setup():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # DLX для разных типов ошибок
    channel.exchange_declare(
        exchange='dlx.errors',
        exchange_type='topic'
    )

    # DLQ для истёкших сообщений
    channel.queue_declare(queue='dlq.expired', durable=True)
    channel.queue_bind(
        queue='dlq.expired',
        exchange='dlx.errors',
        routing_key='expired.*'
    )

    # DLQ для отклонённых сообщений
    channel.queue_declare(queue='dlq.rejected', durable=True)
    channel.queue_bind(
        queue='dlq.rejected',
        exchange='dlx.errors',
        routing_key='rejected.*'
    )

    # Основная очередь с полной настройкой
    channel.queue_declare(
        queue='orders',
        durable=True,
        arguments={
            'x-dead-letter-exchange': 'dlx.errors',
            'x-dead-letter-routing-key': 'rejected.orders',
            'x-message-ttl': 300000,  # 5 минут
            'x-max-length': 10000,
            'x-overflow': 'reject-publish-dlx'
        }
    )

    connection.close()
```

## Обработка отклонённых сообщений

### Явное отклонение

```python
import pika
import json

def consumer_with_rejection():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    def callback(ch, method, properties, body):
        try:
            message = json.loads(body)
            process_order(message)

            # Успешная обработка
            ch.basic_ack(delivery_tag=method.delivery_tag)

        except ValidationError as e:
            # Ошибка валидации — сообщение невозможно обработать
            # Отправляем в DLQ без повторной попытки
            print(f"Ошибка валидации: {e}")
            ch.basic_reject(
                delivery_tag=method.delivery_tag,
                requeue=False  # НЕ возвращаем в очередь = DLX
            )

        except TemporaryError as e:
            # Временная ошибка — можно попробовать снова
            print(f"Временная ошибка: {e}")
            ch.basic_nack(
                delivery_tag=method.delivery_tag,
                requeue=True  # Возвращаем в очередь для повтора
            )

    def process_order(order):
        # Логика обработки заказа
        pass

    channel.basic_qos(prefetch_count=1)
    channel.basic_consume(
        queue='orders',
        on_message_callback=callback,
        auto_ack=False
    )

    channel.start_consuming()
```

### Различие reject и nack

```python
import pika

def demonstrate_reject_vs_nack():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    def callback(ch, method, props, body):
        # basic_reject — отклоняет одно сообщение
        ch.basic_reject(
            delivery_tag=method.delivery_tag,
            requeue=False  # False = в DLX, True = обратно в очередь
        )

        # basic_nack — может отклонить несколько сообщений
        ch.basic_nack(
            delivery_tag=method.delivery_tag,
            multiple=False,  # True = все до этого delivery_tag
            requeue=False
        )

    channel.basic_consume(
        queue='test_queue',
        on_message_callback=callback,
        auto_ack=False
    )
```

## TTL и Dead Letter

### Message TTL

```python
import pika
import time

def ttl_to_dead_letter():
    """
    Сообщения с истёкшим TTL автоматически
    перенаправляются в DLX.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # DLX инфраструктура
    channel.exchange_declare(exchange='dlx', exchange_type='direct')
    channel.queue_declare(queue='dlq', durable=True)
    channel.queue_bind(queue='dlq', exchange='dlx', routing_key='expired')

    # Очередь с TTL
    channel.queue_declare(
        queue='short_ttl_queue',
        arguments={
            'x-message-ttl': 5000,  # 5 секунд
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'expired'
        }
    )

    # Отправляем сообщение
    channel.basic_publish(
        exchange='',
        routing_key='short_ttl_queue',
        body='This will expire'
    )

    print("Сообщение отправлено")
    print("Ждём 6 секунд...")
    time.sleep(6)

    # Проверяем DLQ
    method, props, body = channel.basic_get(queue='dlq', auto_ack=True)
    if body:
        print(f"Сообщение в DLQ: {body.decode()}")

    connection.close()
```

### Per-message TTL

```python
import pika

def per_message_ttl():
    """
    TTL можно задать для конкретного сообщения.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Очередь без общего TTL, но с DLX
    channel.queue_declare(
        queue='variable_ttl',
        arguments={
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'expired'
        }
    )

    # Сообщение с коротким TTL
    channel.basic_publish(
        exchange='',
        routing_key='variable_ttl',
        body='Quick expiry',
        properties=pika.BasicProperties(
            expiration='5000'  # 5 секунд (строка!)
        )
    )

    # Сообщение с длинным TTL
    channel.basic_publish(
        exchange='',
        routing_key='variable_ttl',
        body='Long expiry',
        properties=pika.BasicProperties(
            expiration='3600000'  # 1 час
        )
    )

    connection.close()
```

## Queue Overflow и Dead Letter

### Настройка переполнения

```python
import pika

def overflow_to_dead_letter():
    """
    При переполнении очереди лишние сообщения
    отправляются в DLX.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # DLX инфраструктура
    channel.exchange_declare(exchange='dlx', exchange_type='direct')
    channel.queue_declare(queue='dlq.overflow', durable=True)
    channel.queue_bind(queue='dlq.overflow', exchange='dlx', routing_key='overflow')

    # Очередь с лимитом
    channel.queue_declare(
        queue='limited_queue',
        arguments={
            'x-max-length': 100,  # Максимум 100 сообщений
            'x-overflow': 'reject-publish-dlx',  # Переполнение → DLX
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'overflow'
        }
    )

    # Заполняем очередь
    for i in range(150):
        try:
            channel.basic_publish(
                exchange='',
                routing_key='limited_queue',
                body=f'Message {i}'
            )
        except Exception as e:
            print(f"Ошибка публикации {i}: {e}")

    connection.close()
```

### Типы overflow

```python
"""
x-overflow может быть:

1. 'drop-head' (default)
   - Удаляет старые сообщения (с начала очереди)
   - Новые сообщения принимаются
   - Старые НЕ попадают в DLX!

2. 'reject-publish'
   - Отклоняет новые сообщения
   - Publisher получает basic.nack
   - Сообщения НЕ попадают в DLX

3. 'reject-publish-dlx'
   - Отклоняет новые сообщения
   - Отклонённые попадают в DLX
   - Publisher получает basic.nack
"""
```

## Delivery Limit (Quorum Queues)

### Настройка лимита доставок

```python
import pika

def delivery_limit_example():
    """
    Для Quorum Queues можно ограничить количество
    повторных доставок. При превышении → DLX.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # DLX инфраструктура
    channel.exchange_declare(exchange='dlx', exchange_type='direct')
    channel.queue_declare(queue='dlq.poison', durable=True)
    channel.queue_bind(queue='dlq.poison', exchange='dlx', routing_key='poison')

    # Quorum Queue с лимитом доставок
    channel.queue_declare(
        queue='resilient_queue',
        durable=True,
        arguments={
            'x-queue-type': 'quorum',
            'x-delivery-limit': 3,  # Максимум 3 попытки
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'poison'
        }
    )

    connection.close()
```

### Consumer с delivery count

```python
import pika

def consumer_with_delivery_tracking():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    def callback(ch, method, properties, body):
        # Получаем количество доставок из headers
        headers = properties.headers or {}
        delivery_count = headers.get('x-delivery-count', 1)

        print(f"Попытка доставки #{delivery_count}")

        try:
            process_message(body)
            ch.basic_ack(delivery_tag=method.delivery_tag)
        except Exception as e:
            print(f"Ошибка: {e}")
            # Возвращаем в очередь для повторной попытки
            # При превышении delivery_limit → автоматически в DLX
            ch.basic_nack(
                delivery_tag=method.delivery_tag,
                requeue=True
            )

    def process_message(body):
        # Симуляция случайной ошибки
        import random
        if random.random() < 0.5:
            raise Exception("Random failure")

    channel.basic_consume(
        queue='resilient_queue',
        on_message_callback=callback,
        auto_ack=False
    )

    channel.start_consuming()
```

## Обработка Dead Letters

### Мониторинг DLQ

```python
import pika
import json

def monitor_dead_letters():
    """
    Мониторинг сообщений в Dead Letter Queue.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    def analyze_dead_letter(ch, method, properties, body):
        headers = properties.headers or {}

        # Информация о dead letter
        death_info = headers.get('x-death', [])

        print("=" * 50)
        print(f"Dead Letter: {body.decode()}")

        for death in death_info:
            print(f"  Причина: {death.get('reason')}")
            print(f"  Исходная очередь: {death.get('queue')}")
            print(f"  Время: {death.get('time')}")
            print(f"  Количество: {death.get('count')}")

        # Подтверждаем обработку
        ch.basic_ack(delivery_tag=method.delivery_tag)

    channel.basic_consume(
        queue='dead_letters',
        on_message_callback=analyze_dead_letter,
        auto_ack=False
    )

    print("Мониторинг DLQ...")
    channel.start_consuming()
```

### Повторная обработка Dead Letters

```python
import pika
import json
import time

class DeadLetterProcessor:
    """
    Обработчик Dead Letters с возможностью
    повторной отправки в исходную очередь.
    """

    def __init__(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()

    def process_dead_letters(self, dlq_name, max_retries=3):
        """
        Обрабатываем dead letters с логикой повторов.
        """
        def callback(ch, method, properties, body):
            headers = properties.headers or {}
            death_info = headers.get('x-death', [{}])[0]

            original_queue = death_info.get('queue', 'unknown')
            reason = death_info.get('reason', 'unknown')
            retry_count = headers.get('x-retry-count', 0)

            print(f"Обработка dead letter из {original_queue}")
            print(f"Причина: {reason}, Попытка: {retry_count + 1}")

            if retry_count >= max_retries:
                # Слишком много попыток — архивируем
                self.archive_message(body, headers)
                ch.basic_ack(delivery_tag=method.delivery_tag)
                return

            if self.can_retry(reason):
                # Можем попробовать снова
                self.requeue_message(
                    original_queue,
                    body,
                    properties,
                    retry_count + 1
                )
            else:
                # Невосстановимая ошибка — архивируем
                self.archive_message(body, headers)

            ch.basic_ack(delivery_tag=method.delivery_tag)

        self.channel.basic_qos(prefetch_count=1)
        self.channel.basic_consume(
            queue=dlq_name,
            on_message_callback=callback,
            auto_ack=False
        )

        self.channel.start_consuming()

    def can_retry(self, reason):
        """Определяем, можно ли повторить."""
        retryable = ['expired', 'maxlen']
        return reason in retryable

    def requeue_message(self, queue, body, original_props, retry_count):
        """Отправляем сообщение обратно в очередь."""
        new_headers = dict(original_props.headers or {})
        new_headers['x-retry-count'] = retry_count
        new_headers['x-requeued-at'] = time.time()

        self.channel.basic_publish(
            exchange='',
            routing_key=queue,
            body=body,
            properties=pika.BasicProperties(
                headers=new_headers,
                delivery_mode=2
            )
        )
        print(f"Сообщение возвращено в {queue}")

    def archive_message(self, body, headers):
        """Архивируем неисправимое сообщение."""
        # Сохраняем в файл или базу данных
        print(f"Архивация сообщения: {body.decode()[:50]}...")


# Использование
processor = DeadLetterProcessor()
processor.process_dead_letters('dead_letters')
```

## Паттерны обработки ошибок

### Retry с экспоненциальной задержкой

```python
import pika
import time
import math

def setup_delayed_retry():
    """
    Реализация retry с задержкой через TTL.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Очереди ожидания с разным TTL
    delays = [5000, 30000, 60000]  # 5с, 30с, 60с

    for i, delay in enumerate(delays):
        wait_queue = f'wait.{i}'

        # Очередь ожидания → основная очередь
        channel.queue_declare(
            queue=wait_queue,
            arguments={
                'x-message-ttl': delay,
                'x-dead-letter-exchange': '',
                'x-dead-letter-routing-key': 'main_queue'
            }
        )

    # Основная очередь с DLX
    channel.queue_declare(
        queue='main_queue',
        arguments={
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'failed'
        }
    )

    connection.close()


def retry_with_delay(channel, body, retry_count):
    """
    Отправляем сообщение в очередь ожидания.
    """
    delays = [5000, 30000, 60000]

    if retry_count < len(delays):
        wait_queue = f'wait.{retry_count}'
        channel.basic_publish(
            exchange='',
            routing_key=wait_queue,
            body=body,
            properties=pika.BasicProperties(
                headers={'x-retry-count': retry_count + 1}
            )
        )
    else:
        # Превышен лимит — в DLQ
        channel.basic_publish(
            exchange='dlx',
            routing_key='failed',
            body=body
        )
```

### Категоризация ошибок

```python
import pika

def setup_categorized_dlq():
    """
    Разные DLQ для разных типов ошибок.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Topic exchange для категоризации
    channel.exchange_declare(
        exchange='dlx.categorized',
        exchange_type='topic'
    )

    # DLQ для ошибок валидации
    channel.queue_declare(queue='dlq.validation', durable=True)
    channel.queue_bind(
        queue='dlq.validation',
        exchange='dlx.categorized',
        routing_key='error.validation.*'
    )

    # DLQ для ошибок таймаута
    channel.queue_declare(queue='dlq.timeout', durable=True)
    channel.queue_bind(
        queue='dlq.timeout',
        exchange='dlx.categorized',
        routing_key='error.timeout.*'
    )

    # DLQ для неизвестных ошибок
    channel.queue_declare(queue='dlq.unknown', durable=True)
    channel.queue_bind(
        queue='dlq.unknown',
        exchange='dlx.categorized',
        routing_key='error.#'
    )

    connection.close()
```

## Best Practices

### 1. Всегда настраивайте DLX для production очередей

```python
def production_queue_with_dlx():
    channel.queue_declare(
        queue='orders',
        durable=True,
        arguments={
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'orders.failed'
        }
    )
```

### 2. Логируйте и мониторьте DLQ

```python
def monitor_dlq_size():
    """Следите за размером DLQ."""
    result = channel.queue_declare(queue='dead_letters', passive=True)
    if result.method.message_count > 1000:
        alert("DLQ содержит много сообщений!")
```

### 3. Ограничивайте размер DLQ

```python
channel.queue_declare(
    queue='dead_letters',
    durable=True,
    arguments={
        'x-max-length': 100000,
        'x-overflow': 'drop-head'
    }
)
```

### 4. Используйте осмысленные routing keys

```python
# Плохо
'x-dead-letter-routing-key': 'dead'

# Хорошо
'x-dead-letter-routing-key': 'failed.orders.payment'
```

## Заключение

Dead Letter Queues — критически важный механизм для надёжной обработки сообщений:

- **Предотвращают потерю данных** — проблемные сообщения сохраняются
- **Помогают отладке** — можно анализировать причины ошибок
- **Позволяют повторную обработку** — можно реализовать retry логику
- **Улучшают мониторинг** — рост DLQ сигнализирует о проблемах

Правильно настроенный DLX — обязательный элемент production-системы.
