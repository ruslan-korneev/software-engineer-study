# Classic Queues

## Введение

Classic Queues — это традиционный тип очередей в RabbitMQ, который использовался по умолчанию до версии 3.8. Они обеспечивают высокую производительность для одноузловых развёртываний и поддерживают все базовые функции очередей RabbitMQ.

## Архитектура Classic Queues

### Хранение данных

Classic Queues хранят сообщения в двух местах:

1. **В памяти (RAM)** — для быстрого доступа
2. **На диске** — для персистентных сообщений

```
┌─────────────────────────────────────┐
│         Classic Queue               │
├─────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐   │
│  │   Memory    │  │    Disk     │   │
│  │   Buffer    │  │   Storage   │   │
│  └─────────────┘  └─────────────┘   │
│         ↓                ↓          │
│    Быстрый доступ   Персистентность │
└─────────────────────────────────────┘
```

### Процесс Master-Slave (для миррored queues)

До появления Quorum Queues для репликации использовались Mirrored Queues:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Master    │────▶│   Slave 1   │────▶│   Slave 2   │
│   Node 1    │     │   Node 2    │     │   Node 3    │
└─────────────┘     └─────────────┘     └─────────────┘
       ↑                   ↑                   ↑
   Все записи        Только чтение       Только чтение
```

## Создание Classic Queue

### Базовое создание

```python
import pika

def create_classic_queue():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Создание Classic Queue (по умолчанию)
    channel.queue_declare(
        queue='classic_queue',
        durable=True
    )

    print("Classic Queue создана успешно")
    connection.close()

create_classic_queue()
```

### Явное указание типа очереди

```python
import pika

def create_explicit_classic_queue():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Явно указываем тип очереди через arguments
    channel.queue_declare(
        queue='explicit_classic_queue',
        durable=True,
        arguments={
            'x-queue-type': 'classic'  # Явно указываем тип
        }
    )

    connection.close()

create_explicit_classic_queue()
```

### Настройка Mirrored Queue (устаревший подход)

```python
import pika

def create_mirrored_queue():
    """
    ВНИМАНИЕ: Mirrored Queues устарели!
    Используйте Quorum Queues для репликации в новых проектах.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Создаём очередь с политикой зеркалирования
    # Политика должна быть настроена через rabbitmqctl или HTTP API
    channel.queue_declare(
        queue='mirrored_queue',
        durable=True,
        arguments={
            'x-ha-policy': 'all'  # Зеркалировать на все узлы
        }
    )

    connection.close()
```

## Особенности Classic Queues

### Преимущества

```python
import pika
import time

def demonstrate_classic_performance():
    """
    Classic Queues оптимизированы для высокой пропускной способности
    на одном узле.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    channel.queue_declare(queue='perf_test', durable=False)

    # Высокая скорость публикации
    start = time.time()
    for i in range(10000):
        channel.basic_publish(
            exchange='',
            routing_key='perf_test',
            body=f'Message {i}'
        )
    elapsed = time.time() - start

    print(f"Отправлено 10000 сообщений за {elapsed:.2f} сек")
    print(f"Скорость: {10000/elapsed:.0f} сообщений/сек")

    connection.close()

demonstrate_classic_performance()
```

### Поддержка приоритетов

```python
import pika

def create_priority_queue():
    """
    Classic Queues поддерживают приоритеты сообщений.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Создаём очередь с поддержкой 10 уровней приоритета
    channel.queue_declare(
        queue='priority_queue',
        durable=True,
        arguments={
            'x-max-priority': 10  # Приоритеты от 0 до 10
        }
    )

    # Отправляем сообщения с разными приоритетами
    channel.basic_publish(
        exchange='',
        routing_key='priority_queue',
        body='Низкий приоритет',
        properties=pika.BasicProperties(priority=1)
    )

    channel.basic_publish(
        exchange='',
        routing_key='priority_queue',
        body='Высокий приоритет',
        properties=pika.BasicProperties(priority=9)
    )

    connection.close()

create_priority_queue()
```

### Ленивые очереди (Lazy Queues)

```python
import pika

def create_lazy_queue():
    """
    Lazy Queues хранят сообщения на диске, экономя RAM.
    Полезно для очередей с большим количеством сообщений.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    channel.queue_declare(
        queue='lazy_classic_queue',
        durable=True,
        arguments={
            'x-queue-mode': 'lazy'  # Хранить сообщения на диске
        }
    )

    print("Lazy Queue создана")
    connection.close()

create_lazy_queue()
```

## Когда использовать Classic Queues

### Подходящие сценарии

```python
"""
Classic Queues рекомендуются для:

1. Одноузловых развёртываний
2. Временных очередей (exclusive, auto-delete)
3. Очередей с приоритетами
4. Когда нужна максимальная производительность и репликация не критична
"""

import pika

def setup_single_node_queue():
    """
    Пример для одноузлового развёртывания.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Идеально для одного узла — нет накладных расходов на репликацию
    channel.queue_declare(
        queue='single_node_tasks',
        durable=True
    )

    connection.close()
```

### Сценарии, где лучше выбрать Quorum Queues

```python
"""
Когда НЕ стоит использовать Classic Queues:

1. Требуется высокая доступность (HA)
2. Критически важные данные
3. Кластерные развёртывания
4. Необходима надёжная репликация
"""

# Пример: для критических данных используйте Quorum Queues
# channel.queue_declare(
#     queue='critical_data',
#     durable=True,
#     arguments={'x-queue-type': 'quorum'}
# )
```

## Миграция с Classic на Quorum Queues

### Проверка текущего типа очереди

```python
import pika

def check_queue_type():
    """
    Проверяем тип существующей очереди.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    try:
        # Используем passive=True для проверки
        result = channel.queue_declare(
            queue='existing_queue',
            passive=True
        )
        print(f"Очередь существует: {result.method.queue}")
        # Для определения типа нужно использовать Management API
    except pika.exceptions.ChannelClosedByBroker:
        print("Очередь не существует")

    connection.close()
```

### Стратегия миграции

```python
import pika
import json

def migrate_classic_to_quorum():
    """
    Пошаговая миграция с Classic на Quorum Queue.

    ВАЖНО: Прямое преобразование невозможно!
    Требуется создание новой очереди и перенос данных.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    old_queue = 'classic_orders'
    new_queue = 'quorum_orders'

    # Шаг 1: Создаём новую Quorum Queue
    channel.queue_declare(
        queue=new_queue,
        durable=True,
        arguments={'x-queue-type': 'quorum'}
    )

    # Шаг 2: Перенаправляем продюсеров на новую очередь
    # (изменить конфигурацию приложений)

    # Шаг 3: Дождаться обработки всех сообщений в старой очереди
    while True:
        result = channel.queue_declare(queue=old_queue, passive=True)
        if result.method.message_count == 0:
            break
        print(f"Осталось сообщений: {result.method.message_count}")
        import time
        time.sleep(1)

    # Шаг 4: Удаляем старую очередь
    channel.queue_delete(queue=old_queue)

    print("Миграция завершена")
    connection.close()
```

## Мониторинг Classic Queues

```python
import pika
import requests

def monitor_classic_queue():
    """
    Мониторинг состояния Classic Queue через Management API.
    """
    # Через pika
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    result = channel.queue_declare(queue='monitored_queue', passive=True)
    print(f"Сообщений в очереди: {result.method.message_count}")
    print(f"Потребителей: {result.method.consumer_count}")

    connection.close()

    # Через Management HTTP API
    response = requests.get(
        'http://localhost:15672/api/queues/%2F/monitored_queue',
        auth=('guest', 'guest')
    )

    if response.status_code == 200:
        data = response.json()
        print(f"Тип очереди: {data.get('type', 'classic')}")
        print(f"Память: {data.get('memory', 0)} байт")
        print(f"Состояние: {data.get('state', 'unknown')}")
```

## Настройка производительности

```python
import pika

def optimize_classic_queue():
    """
    Оптимизация производительности Classic Queue.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Включаем подтверждения публикации
    channel.confirm_delivery()

    # Создаём оптимизированную очередь
    channel.queue_declare(
        queue='optimized_classic',
        durable=True,
        arguments={
            # Ограничение размера очереди
            'x-max-length': 100000,
            'x-max-length-bytes': 1073741824,  # 1GB

            # Политика при переполнении
            'x-overflow': 'drop-head',  # Удалять старые сообщения

            # TTL для сообщений
            'x-message-ttl': 3600000,  # 1 час
        }
    )

    # Prefetch для равномерной нагрузки
    channel.basic_qos(prefetch_count=10)

    connection.close()

optimize_classic_queue()
```

## Best Practices для Classic Queues

### 1. Используйте для временных очередей

```python
import pika

def temporary_queue_example():
    """
    Classic Queues идеальны для временных очередей.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Временная очередь для RPC callback
    result = channel.queue_declare(
        queue='',
        exclusive=True,
        auto_delete=True
    )
    callback_queue = result.method.queue

    print(f"Временная очередь: {callback_queue}")
    connection.close()
```

### 2. Ограничивайте размер

```python
channel.queue_declare(
    queue='bounded_classic',
    arguments={
        'x-max-length': 10000,
        'x-overflow': 'reject-publish'
    }
)
```

### 3. Используйте lazy mode для больших очередей

```python
channel.queue_declare(
    queue='large_queue',
    arguments={
        'x-queue-mode': 'lazy'
    }
)
```

## Ограничения Classic Queues

1. **Репликация ненадёжна** — Mirrored Queues могут терять данные при сбоях
2. **Single-master** — все операции записи идут через один узел
3. **Split-brain** — возможны проблемы при сетевых разделениях
4. **Устаревший подход** — RabbitMQ рекомендует Quorum Queues для HA

## Заключение

Classic Queues остаются хорошим выбором для простых сценариев, одноузловых развёртываний и временных очередей. Однако для production-систем, требующих высокой доступности, рекомендуется использовать Quorum Queues.
