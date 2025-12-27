# Quorum Queues

## Введение

Quorum Queues — это современный тип очередей в RabbitMQ, разработанный для обеспечения высокой доступности и надёжности данных. Они используют алгоритм консенсуса Raft для репликации сообщений между узлами кластера.

## Что такое Raft консенсус

### Основы алгоритма Raft

Raft — это алгоритм распределённого консенсуса, обеспечивающий согласованность данных между узлами кластера.

```
┌─────────────────────────────────────────────────────────────┐
│                    Raft Consensus                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐          │
│  │  Leader  │─────▶│ Follower │      │ Follower │          │
│  │  Node 1  │      │  Node 2  │      │  Node 3  │          │
│  └──────────┘      └──────────┘      └──────────┘          │
│       │                 ▲                 ▲                 │
│       │                 │                 │                 │
│       └─────────────────┴─────────────────┘                 │
│              Репликация записей                             │
│                                                             │
│  Кворум = (N/2) + 1 узлов должны подтвердить запись        │
│  Для 3 узлов: кворум = 2                                   │
│  Для 5 узлов: кворум = 3                                   │
└─────────────────────────────────────────────────────────────┘
```

### Роли узлов в Raft

1. **Leader** — принимает все записи и реплицирует их
2. **Follower** — получает записи от лидера
3. **Candidate** — узел, претендующий на роль лидера при выборах

```python
"""
Жизненный цикл Raft:

1. Выбор лидера (Leader Election)
   - При старте или потере лидера узлы голосуют за нового

2. Репликация записей (Log Replication)
   - Лидер принимает запись
   - Отправляет её follower'ам
   - Ждёт подтверждения от кворума
   - Фиксирует запись

3. Безопасность (Safety)
   - Только лидер с самым актуальным логом может быть избран
   - Гарантирует отсутствие потери подтверждённых записей
"""
```

## Создание Quorum Queue

### Базовое создание

```python
import pika

def create_quorum_queue():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Создание Quorum Queue
    channel.queue_declare(
        queue='quorum_tasks',
        durable=True,  # Quorum Queues всегда durable
        arguments={
            'x-queue-type': 'quorum'  # Указываем тип очереди
        }
    )

    print("Quorum Queue создана успешно")
    connection.close()

create_quorum_queue()
```

### Создание с настройками репликации

```python
import pika

def create_quorum_queue_with_replicas():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Quorum Queue с настройками
    channel.queue_declare(
        queue='critical_orders',
        durable=True,
        arguments={
            'x-queue-type': 'quorum',

            # Начальный размер группы (количество реплик)
            'x-quorum-initial-group-size': 5,

            # Лимиты доставки для poison messages
            'x-delivery-limit': 5,

            # Dead letter exchange для недоставленных
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'failed.orders'
        }
    )

    connection.close()

create_quorum_queue_with_replicas()
```

## Репликация данных

### Как работает репликация

```python
"""
Процесс репликации сообщения в Quorum Queue:

1. Publisher отправляет сообщение
2. Сообщение попадает на Leader
3. Leader записывает в свой лог
4. Leader реплицирует на Followers
5. Followers подтверждают получение
6. При достижении кворума — сообщение считается принятым
7. Publisher получает подтверждение (если включены confirms)
"""

import pika

def publish_with_confirms():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Включаем подтверждения публикации
    channel.confirm_delivery()

    # Создаём Quorum Queue
    channel.queue_declare(
        queue='confirmed_queue',
        durable=True,
        arguments={'x-queue-type': 'quorum'}
    )

    try:
        # Публикация с ожиданием подтверждения
        channel.basic_publish(
            exchange='',
            routing_key='confirmed_queue',
            body='Important message',
            properties=pika.BasicProperties(
                delivery_mode=2  # Персистентное сообщение
            ),
            mandatory=True  # Требуем маршрутизацию
        )
        print("Сообщение подтверждено кворумом!")
    except pika.exceptions.UnroutableError:
        print("Сообщение не было маршрутизировано")

    connection.close()

publish_with_confirms()
```

### Визуализация процесса репликации

```
┌───────────────────────────────────────────────────────────────┐
│                    Репликация сообщения                       │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  Publisher ──── Сообщение ────▶ Leader (Node 1)               │
│                                     │                         │
│                     ┌───────────────┼───────────────┐         │
│                     ▼               ▼               ▼         │
│               Follower 2      Follower 3      Follower 4      │
│                     │               │               │         │
│                     ▼               ▼               ▼         │
│                   ACK ──────────▶ Leader ◀────── ACK          │
│                                     │                         │
│                          Кворум достигнут                     │
│                                     │                         │
│                                     ▼                         │
│                          Подтверждение ────▶ Publisher        │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

## Высокая доступность

### Автоматическое восстановление

```python
import pika
import time

def demonstrate_failover():
    """
    Демонстрация отказоустойчивости Quorum Queue.

    При падении лидера:
    1. Followers обнаруживают потерю heartbeat
    2. Начинаются выборы нового лидера
    3. Follower с наиболее актуальным логом становится лидером
    4. Очередь продолжает работать
    """
    # Подключение к кластеру (несколько узлов)
    connection_params = [
        pika.ConnectionParameters('node1.rabbitmq.local'),
        pika.ConnectionParameters('node2.rabbitmq.local'),
        pika.ConnectionParameters('node3.rabbitmq.local'),
    ]

    connection = None
    for params in connection_params:
        try:
            connection = pika.BlockingConnection(params)
            print(f"Подключено к {params.host}")
            break
        except pika.exceptions.AMQPConnectionError:
            print(f"Узел {params.host} недоступен, пробуем следующий...")

    if connection is None:
        raise Exception("Не удалось подключиться к кластеру")

    channel = connection.channel()

    # Quorum Queue автоматически переживёт отказ узла
    channel.queue_declare(
        queue='ha_queue',
        durable=True,
        arguments={'x-queue-type': 'quorum'}
    )

    connection.close()
```

### Мониторинг состояния реплик

```python
import requests

def check_quorum_queue_health():
    """
    Проверка здоровья Quorum Queue через Management API.
    """
    response = requests.get(
        'http://localhost:15672/api/queues/%2F/quorum_tasks',
        auth=('guest', 'guest')
    )

    if response.status_code == 200:
        data = response.json()

        print(f"Имя очереди: {data['name']}")
        print(f"Тип: {data['type']}")
        print(f"Сообщений: {data['messages']}")

        # Информация о членах кворума
        if 'members' in data:
            print(f"\nЧлены кворума:")
            for member in data['members']:
                print(f"  - {member}")

        # Лидер очереди
        if 'leader' in data:
            print(f"\nЛидер: {data['leader']}")

        # Online реплики
        if 'online' in data:
            print(f"Online реплик: {len(data['online'])}")

check_quorum_queue_health()
```

## Настройки Quorum Queues

### Размер группы (Initial Group Size)

```python
import pika

def configure_group_size():
    """
    Настройка количества реплик.

    Рекомендации:
    - 3 реплики: переживёт отказ 1 узла
    - 5 реплик: переживёт отказ 2 узлов
    - 7 реплик: переживёт отказ 3 узлов

    Формула: кворум = (N/2) + 1
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # 5 реплик для высокой надёжности
    channel.queue_declare(
        queue='highly_available_queue',
        durable=True,
        arguments={
            'x-queue-type': 'quorum',
            'x-quorum-initial-group-size': 5
        }
    )

    connection.close()
```

### Delivery Limit (защита от poison messages)

```python
import pika

def configure_delivery_limit():
    """
    Ограничение количества повторных доставок.

    Предотвращает бесконечный цикл requeue для
    проблемных сообщений (poison messages).
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # DLX для отклонённых сообщений
    channel.exchange_declare(
        exchange='dlx',
        exchange_type='direct'
    )

    channel.queue_declare(
        queue='dlq',
        durable=True
    )

    channel.queue_bind(
        queue='dlq',
        exchange='dlx',
        routing_key='failed'
    )

    # Основная очередь с лимитом доставки
    channel.queue_declare(
        queue='limited_delivery_queue',
        durable=True,
        arguments={
            'x-queue-type': 'quorum',
            'x-delivery-limit': 3,  # Максимум 3 попытки
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'failed'
        }
    )

    connection.close()

configure_delivery_limit()
```

### Memory Limit (ограничение памяти)

```python
import pika

def configure_memory_limit():
    """
    Настройка использования памяти.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    channel.queue_declare(
        queue='memory_limited_queue',
        durable=True,
        arguments={
            'x-queue-type': 'quorum',
            # Максимальная длина очереди
            'x-max-length': 100000,
            # Максимальный размер в байтах
            'x-max-length-bytes': 1073741824,  # 1GB
            # Политика при достижении лимита
            'x-overflow': 'reject-publish'
        }
    )

    connection.close()
```

## Потребление сообщений

### Базовое потребление

```python
import pika

def consume_from_quorum_queue():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Объявляем очередь (идемпотентно)
    channel.queue_declare(
        queue='quorum_tasks',
        durable=True,
        arguments={'x-queue-type': 'quorum'}
    )

    # Настраиваем prefetch
    channel.basic_qos(prefetch_count=10)

    def callback(ch, method, properties, body):
        print(f"Получено: {body.decode()}")

        # Обработка сообщения
        try:
            process_message(body)
            # Подтверждаем успешную обработку
            ch.basic_ack(delivery_tag=method.delivery_tag)
        except Exception as e:
            print(f"Ошибка обработки: {e}")
            # Отклоняем с возвратом в очередь
            ch.basic_nack(
                delivery_tag=method.delivery_tag,
                requeue=True
            )

    def process_message(body):
        # Логика обработки
        pass

    channel.basic_consume(
        queue='quorum_tasks',
        on_message_callback=callback,
        auto_ack=False
    )

    print("Ожидание сообщений...")
    channel.start_consuming()
```

### Single Active Consumer

```python
import pika

def single_active_consumer():
    """
    Гарантирует, что только один consumer активен в любой момент.
    Полезно для упорядоченной обработки.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    channel.queue_declare(
        queue='ordered_queue',
        durable=True,
        arguments={
            'x-queue-type': 'quorum',
            'x-single-active-consumer': True  # Только один активный consumer
        }
    )

    def callback(ch, method, properties, body):
        print(f"Обработка: {body.decode()}")
        ch.basic_ack(delivery_tag=method.delivery_tag)

    channel.basic_consume(
        queue='ordered_queue',
        on_message_callback=callback,
        auto_ack=False
    )

    channel.start_consuming()
```

## Сравнение с Classic Queues

```python
"""
┌─────────────────────┬──────────────────┬──────────────────┐
│     Характеристика  │  Classic Queue   │  Quorum Queue    │
├─────────────────────┼──────────────────┼──────────────────┤
│ Репликация          │ Mirrored (устар.)│ Raft консенсус   │
│ Надёжность          │ Низкая           │ Высокая          │
│ Производительность  │ Выше             │ Ниже             │
│ Приоритеты          │ Да               │ Нет              │
│ TTL сообщений       │ Да               │ Да               │
│ Lazy mode           │ Да               │ Не нужен         │
│ Exclusive           │ Да               │ Нет              │
│ Non-durable         │ Да               │ Нет              │
│ Delivery limit      │ Нет              │ Да               │
│ Poison message      │ Ручная обработка │ Автоматическая   │
└─────────────────────┴──────────────────┴──────────────────┘
"""
```

## Best Practices

### 1. Правильный размер кластера

```python
"""
Рекомендации по размеру кластера:

- Минимум 3 узла (переживает отказ 1)
- Оптимально 5 узлов (переживает отказ 2)
- Нечётное количество (избегает split-brain)
"""
```

### 2. Настройка подтверждений

```python
import pika

def reliable_publishing():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Обязательно включаем confirms
    channel.confirm_delivery()

    channel.queue_declare(
        queue='reliable_queue',
        durable=True,
        arguments={'x-queue-type': 'quorum'}
    )

    # Публикуем с обязательной маршрутизацией
    channel.basic_publish(
        exchange='',
        routing_key='reliable_queue',
        body='Critical data',
        properties=pika.BasicProperties(
            delivery_mode=2
        ),
        mandatory=True
    )

    connection.close()
```

### 3. Обработка poison messages

```python
import pika

def handle_poison_messages():
    """
    Используйте delivery_limit и DLQ для обработки
    проблемных сообщений.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Настройка DLX
    channel.exchange_declare(exchange='dlx', exchange_type='direct')
    channel.queue_declare(queue='poison_messages', durable=True)
    channel.queue_bind(queue='poison_messages', exchange='dlx', routing_key='poison')

    # Основная очередь
    channel.queue_declare(
        queue='main_queue',
        durable=True,
        arguments={
            'x-queue-type': 'quorum',
            'x-delivery-limit': 5,
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'poison'
        }
    )

    connection.close()
```

### 4. Мониторинг

```python
import requests

def monitor_quorum_queues():
    """
    Регулярный мониторинг состояния Quorum Queues.
    """
    response = requests.get(
        'http://localhost:15672/api/queues',
        auth=('guest', 'guest')
    )

    queues = response.json()

    for queue in queues:
        if queue.get('type') == 'quorum':
            name = queue['name']
            messages = queue['messages']
            online = len(queue.get('online', []))
            members = len(queue.get('members', []))

            print(f"Queue: {name}")
            print(f"  Messages: {messages}")
            print(f"  Online replicas: {online}/{members}")

            if online < members:
                print(f"  WARNING: Not all replicas online!")
```

## Ограничения Quorum Queues

1. **Не поддерживают приоритеты** — используйте отдельные очереди
2. **Только durable** — не могут быть временными
3. **Не поддерживают exclusive** — доступны всем соединениям
4. **Более высокая latency** — из-за репликации
5. **Больше ресурсов** — требуют больше памяти и CPU

## Заключение

Quorum Queues — рекомендуемый выбор для production-систем, где важна надёжность и высокая доступность данных. Алгоритм Raft обеспечивает консистентную репликацию и автоматическое восстановление при отказах узлов.
