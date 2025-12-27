# Свойства очередей

## Введение

При создании очереди в RabbitMQ можно задать различные свойства, которые определяют её поведение, жизненный цикл и дополнительные функции. Понимание этих свойств критически важно для правильного проектирования системы обмена сообщениями.

## Основные свойства очередей

### Обзор параметров queue_declare

```python
import pika

def queue_declare_overview():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    channel.queue_declare(
        queue='example_queue',      # Имя очереди
        durable=True,               # Переживает перезапуск брокера
        exclusive=False,            # Доступна только этому соединению
        auto_delete=False,          # Удаляется при отключении последнего потребителя
        arguments={}                # Дополнительные аргументы
    )

    connection.close()
```

## Свойство: durable

### Что такое durable

`durable=True` означает, что очередь переживёт перезапуск RabbitMQ. Метаданные очереди будут сохранены на диске.

```python
import pika

def demonstrate_durable():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Durable очередь — сохраняется после перезапуска
    channel.queue_declare(
        queue='persistent_queue',
        durable=True
    )

    # Non-durable очередь — исчезает после перезапуска
    channel.queue_declare(
        queue='temporary_queue',
        durable=False
    )

    connection.close()

demonstrate_durable()
```

### Важно: durable очередь и персистентные сообщения

```python
import pika

def durable_with_persistent_messages():
    """
    ВАЖНО: durable очередь НЕ гарантирует сохранность сообщений!
    Для полной персистентности нужно также делать сообщения персистентными.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Durable очередь
    channel.queue_declare(queue='reliable_queue', durable=True)

    # Персистентное сообщение (delivery_mode=2)
    channel.basic_publish(
        exchange='',
        routing_key='reliable_queue',
        body='This message will survive restart',
        properties=pika.BasicProperties(
            delivery_mode=2  # Делает сообщение персистентным
        )
    )

    print("Сообщение отправлено с персистентностью")
    connection.close()

durable_with_persistent_messages()
```

### Когда использовать durable

```python
"""
Используйте durable=True когда:
✓ Очередь содержит критические данные
✓ Нужна надёжность при перезапусках
✓ Это production-очередь

Используйте durable=False когда:
✓ Данные временные и некритичные
✓ Нужна максимальная производительность
✓ Это очередь для разработки/тестирования
"""
```

## Свойство: exclusive

### Что такое exclusive

`exclusive=True` означает, что очередь доступна только текущему соединению и будет удалена при его закрытии.

```python
import pika

def demonstrate_exclusive():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Exclusive очередь — только для этого соединения
    result = channel.queue_declare(
        queue='',  # Автогенерируемое имя
        exclusive=True
    )

    exclusive_queue = result.method.queue
    print(f"Exclusive очередь: {exclusive_queue}")

    # Эта очередь:
    # 1. Доступна только этому соединению
    # 2. Будет удалена при закрытии соединения
    # 3. Другие соединения НЕ могут её использовать

    connection.close()
    # После close() очередь автоматически удаляется

demonstrate_exclusive()
```

### Типичное использование exclusive

```python
import pika
import uuid

def rpc_client_with_exclusive_queue():
    """
    Exclusive очереди идеальны для RPC callbacks.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Создаём exclusive callback queue
    result = channel.queue_declare(queue='', exclusive=True)
    callback_queue = result.method.queue

    correlation_id = str(uuid.uuid4())

    # Отправляем RPC запрос
    channel.basic_publish(
        exchange='',
        routing_key='rpc_queue',
        body='request data',
        properties=pika.BasicProperties(
            reply_to=callback_queue,
            correlation_id=correlation_id
        )
    )

    # Ожидаем ответ
    def on_response(ch, method, props, body):
        if props.correlation_id == correlation_id:
            print(f"Получен ответ: {body.decode()}")

    channel.basic_consume(
        queue=callback_queue,
        on_message_callback=on_response,
        auto_ack=True
    )

    # При закрытии соединения callback_queue удалится автоматически
    connection.close()
```

### Ограничения exclusive очередей

```python
"""
Ограничения exclusive=True:
1. Не может использоваться в кластере несколькими узлами
2. Не совместимо с Quorum Queues
3. Привязано к конкретному соединению
4. Нельзя объявить durable (не имеет смысла)
"""

import pika

def exclusive_limitations():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Exclusive всегда используется с durable=False
    channel.queue_declare(
        queue='exclusive_temp',
        exclusive=True,
        durable=False  # Рекомендуется явно указать
    )

    connection.close()
```

## Свойство: auto_delete

### Что такое auto_delete

`auto_delete=True` означает, что очередь будет удалена, когда от неё отключится последний потребитель.

```python
import pika
import threading
import time

def demonstrate_auto_delete():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Auto-delete очередь
    channel.queue_declare(
        queue='auto_delete_queue',
        auto_delete=True
    )

    print("Очередь создана")

    # Подключаем потребителя
    def callback(ch, method, properties, body):
        print(f"Получено: {body.decode()}")
        ch.basic_ack(delivery_tag=method.delivery_tag)

    consumer_tag = channel.basic_consume(
        queue='auto_delete_queue',
        on_message_callback=callback
    )

    print(f"Consumer подключен: {consumer_tag}")

    # Отменяем потребителя
    channel.basic_cancel(consumer_tag)
    print("Consumer отключен")

    # Проверяем существование очереди
    try:
        channel.queue_declare(queue='auto_delete_queue', passive=True)
        print("Очередь всё ещё существует")
    except pika.exceptions.ChannelClosedByBroker:
        print("Очередь удалена (auto_delete)")

    connection.close()
```

### Разница между exclusive и auto_delete

```python
"""
┌─────────────────────┬──────────────────┬──────────────────┐
│     Свойство        │    exclusive     │   auto_delete    │
├─────────────────────┼──────────────────┼──────────────────┤
│ Доступность         │ Только одно      │ Любые            │
│                     │ соединение       │ соединения       │
├─────────────────────┼──────────────────┼──────────────────┤
│ Удаление            │ При закрытии     │ При отключении   │
│                     │ соединения       │ последнего       │
│                     │                  │ consumer         │
├─────────────────────┼──────────────────┼──────────────────┤
│ Сообщения без       │ Накапливаются    │ Накапливаются    │
│ consumers           │ до закрытия      │ (очередь живёт)  │
│                     │ соединения       │                  │
└─────────────────────┴──────────────────┴──────────────────┘
"""

import pika

def compare_exclusive_auto_delete():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Exclusive: удаляется при закрытии соединения
    channel.queue_declare(
        queue='',
        exclusive=True
    )

    # Auto-delete: удаляется при отключении последнего consumer
    channel.queue_declare(
        queue='shared_temp',
        auto_delete=True,
        exclusive=False
    )

    # Комбинация: удаляется при первом из событий
    channel.queue_declare(
        queue='',
        exclusive=True,
        auto_delete=True
    )

    connection.close()
```

## Свойство: arguments

### Обзор дополнительных аргументов

Аргументы позволяют настроить расширенные функции очереди.

```python
import pika

def queue_with_arguments():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    channel.queue_declare(
        queue='configured_queue',
        durable=True,
        arguments={
            # Тип очереди
            'x-queue-type': 'classic',  # или 'quorum', 'stream'

            # TTL сообщений
            'x-message-ttl': 60000,  # 60 секунд

            # TTL самой очереди
            'x-expires': 1800000,  # 30 минут без использования

            # Ограничения размера
            'x-max-length': 10000,
            'x-max-length-bytes': 104857600,  # 100MB

            # Поведение при переполнении
            'x-overflow': 'reject-publish',

            # Dead letter настройки
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'dead',

            # Single active consumer
            'x-single-active-consumer': False
        }
    )

    connection.close()
```

### x-message-ttl

```python
import pika

def message_ttl_example():
    """
    TTL (Time To Live) определяет, как долго сообщение
    живёт в очереди до автоматического удаления.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Все сообщения в этой очереди живут максимум 30 секунд
    channel.queue_declare(
        queue='ttl_queue',
        arguments={
            'x-message-ttl': 30000  # миллисекунды
        }
    )

    # Отправляем сообщение
    channel.basic_publish(
        exchange='',
        routing_key='ttl_queue',
        body='This message expires in 30 seconds'
    )

    # Альтернатива: TTL для конкретного сообщения
    channel.basic_publish(
        exchange='',
        routing_key='other_queue',
        body='This specific message expires in 10 seconds',
        properties=pika.BasicProperties(
            expiration='10000'  # строка в миллисекундах
        )
    )

    connection.close()
```

### x-expires

```python
import pika

def queue_expiration_example():
    """
    x-expires удаляет саму очередь, если она не используется
    в течение указанного времени.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Очередь удалится через 10 минут неиспользования
    channel.queue_declare(
        queue='expiring_queue',
        arguments={
            'x-expires': 600000  # 10 минут в миллисекундах
        }
    )

    # "Неиспользование" означает:
    # - Нет потребителей
    # - Нет вызовов basic.get

    connection.close()
```

### x-max-length и x-max-length-bytes

```python
import pika

def queue_length_limits():
    """
    Ограничение размера очереди по количеству сообщений
    или по объёму данных.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    channel.queue_declare(
        queue='bounded_queue',
        arguments={
            'x-max-length': 1000,           # Максимум сообщений
            'x-max-length-bytes': 52428800, # Максимум 50MB

            # Что делать при достижении лимита
            'x-overflow': 'drop-head'  # Удалять старые сообщения
            # Альтернативы:
            # 'x-overflow': 'reject-publish'  # Отклонять новые
            # 'x-overflow': 'reject-publish-dlx'  # Отклонять в DLX
        }
    )

    connection.close()
```

### x-dead-letter-exchange и x-dead-letter-routing-key

```python
import pika

def dead_letter_configuration():
    """
    Настройка перенаправления "мёртвых" сообщений.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Создаём DLX и DLQ
    channel.exchange_declare(exchange='dlx', exchange_type='direct')
    channel.queue_declare(queue='dead_letters', durable=True)
    channel.queue_bind(queue='dead_letters', exchange='dlx', routing_key='dead')

    # Основная очередь с DLX
    channel.queue_declare(
        queue='main_queue',
        arguments={
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'dead',

            # Сообщения попадут в DLX если:
            # 1. Истёк TTL
            'x-message-ttl': 60000,

            # 2. Очередь переполнена
            'x-max-length': 1000,
            'x-overflow': 'reject-publish-dlx',

            # 3. Сообщение отклонено (nack/reject с requeue=false)
        }
    )

    connection.close()
```

### x-max-priority

```python
import pika

def priority_queue_example():
    """
    Создание очереди с поддержкой приоритетов.
    Только для Classic Queues!
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Очередь с 10 уровнями приоритета (0-9)
    channel.queue_declare(
        queue='priority_tasks',
        arguments={
            'x-max-priority': 10
        }
    )

    # Высокоприоритетное сообщение
    channel.basic_publish(
        exchange='',
        routing_key='priority_tasks',
        body='Urgent task',
        properties=pika.BasicProperties(priority=9)
    )

    # Низкоприоритетное сообщение
    channel.basic_publish(
        exchange='',
        routing_key='priority_tasks',
        body='Normal task',
        properties=pika.BasicProperties(priority=1)
    )

    connection.close()
```

### x-queue-mode

```python
import pika

def lazy_queue_example():
    """
    Lazy режим хранит сообщения на диске вместо RAM.
    Экономит память для очередей с большим количеством сообщений.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Lazy очередь
    channel.queue_declare(
        queue='large_queue',
        durable=True,
        arguments={
            'x-queue-mode': 'lazy'
            # Альтернатива: 'default' (хранить в RAM когда возможно)
        }
    )

    connection.close()
```

### x-single-active-consumer

```python
import pika

def single_active_consumer_example():
    """
    Гарантирует, что в любой момент времени активен только
    один потребитель. Полезно для упорядоченной обработки.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    channel.queue_declare(
        queue='ordered_processing',
        arguments={
            'x-single-active-consumer': True
        }
    )

    # Если запущено несколько consumers:
    # - Только один будет получать сообщения
    # - При его отключении следующий станет активным
    # - Гарантирует порядок обработки

    connection.close()
```

## Комбинации свойств

### Production-ready durable очередь

```python
import pika

def production_queue():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # DLX инфраструктура
    channel.exchange_declare(exchange='dlx', exchange_type='direct')
    channel.queue_declare(queue='failed_messages', durable=True)
    channel.queue_bind(queue='failed_messages', exchange='dlx', routing_key='failed')

    # Production очередь
    channel.queue_declare(
        queue='orders',
        durable=True,
        exclusive=False,
        auto_delete=False,
        arguments={
            'x-queue-type': 'quorum',  # Высокая доступность
            'x-max-length': 100000,
            'x-max-length-bytes': 1073741824,  # 1GB
            'x-overflow': 'reject-publish',
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'failed',
            'x-delivery-limit': 5  # Для quorum queues
        }
    )

    connection.close()
```

### Временная очередь для RPC

```python
import pika

def rpc_callback_queue():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Временная callback очередь
    result = channel.queue_declare(
        queue='',  # Автоматическое имя
        exclusive=True,  # Только для этого соединения
        auto_delete=True,  # Удалить при отключении
        arguments={
            'x-expires': 60000,  # Удалить через минуту на всякий случай
            'x-message-ttl': 30000  # Ответы живут 30 секунд
        }
    )

    callback_queue = result.method.queue
    print(f"RPC callback queue: {callback_queue}")

    connection.close()
```

## Best Practices

### 1. Выбор свойств по сценарию

```python
"""
Сценарий                          | durable | exclusive | auto_delete
----------------------------------|---------|-----------|------------
Production критичные данные       | True    | False     | False
RPC callback очередь              | False   | True      | True
Временные уведомления             | False   | False     | True
Обработка событий (pub/sub)       | False   | False     | True
Очередь задач (worker)            | True    | False     | False
"""
```

### 2. Не меняйте свойства существующей очереди

```python
import pika

def handle_property_mismatch():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Первое объявление
    channel.queue_declare(queue='my_queue', durable=True)

    # Повторное с другими свойствами — ОШИБКА!
    try:
        channel.queue_declare(queue='my_queue', durable=False)
    except pika.exceptions.ChannelClosedByBroker as e:
        print(f"Ошибка: {e}")
        # PRECONDITION_FAILED - inequivalent arg 'durable'

    connection.close()
```

### 3. Используйте policies для изменения аргументов

```python
"""
Свойства durable, exclusive, auto_delete нельзя изменить.
Но некоторые arguments можно переопределить через policies.

Пример через rabbitmqctl:
rabbitmqctl set_policy my-policy "^my_queue$" \
    '{"max-length": 5000}' --apply-to queues
"""
```

## Заключение

Правильный выбор свойств очереди определяет её поведение, надёжность и производительность. Основные рекомендации:

- **durable=True** для критических данных
- **exclusive=True** для временных приватных очередей
- **auto_delete=True** для динамически создаваемых очередей
- **arguments** для тонкой настройки TTL, лимитов и DLX
