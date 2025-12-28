# Priority Queues (Приоритетные очереди)

[prev: 04-message-ttl](./04-message-ttl.md) | [next: 06-prefetch](./06-prefetch.md)

---

## Введение

Priority Queues - это специальный тип очередей в RabbitMQ, который позволяет обрабатывать сообщения в порядке их приоритета, а не в порядке поступления (FIFO). Сообщения с более высоким приоритетом доставляются потребителям раньше, чем сообщения с низким приоритетом.

## Принцип работы

```
Обычная очередь (FIFO):
[Msg1] -> [Msg2] -> [Msg3] -> Consumer
(порядок поступления)

Приоритетная очередь:
[Msg2(P:10)] -> [Msg1(P:5)] -> [Msg3(P:1)] -> Consumer
(порядок по приоритету)
```

## Создание приоритетной очереди

### Базовая настройка

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Создаем очередь с поддержкой приоритетов
channel.queue_declare(
    queue='priority_queue',
    durable=True,
    arguments={
        'x-max-priority': 10  # Максимальный приоритет (1-255)
    }
)

print("Приоритетная очередь создана (max priority: 10)")
```

### Выбор максимального приоритета

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Низкий диапазон - меньше overhead
channel.queue_declare(
    queue='low_priority_range',
    durable=True,
    arguments={
        'x-max-priority': 5  # 0-5, 6 уровней
    }
)

# Стандартный диапазон
channel.queue_declare(
    queue='standard_priority_range',
    durable=True,
    arguments={
        'x-max-priority': 10  # 0-10, 11 уровней
    }
)

# Высокий диапазон (использовать с осторожностью)
channel.queue_declare(
    queue='high_priority_range',
    durable=True,
    arguments={
        'x-max-priority': 255  # Максимум, большой overhead
    }
)

"""
Рекомендации по x-max-priority:
- 1-5: Для простых сценариев (low/normal/high)
- 10: Стандартный выбор
- 255: Редко нужен, большое потребление памяти

Каждый уровень приоритета создает отдельную подочередь в памяти!
"""
```

## Публикация сообщений с приоритетом

### Базовая публикация

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(
    queue='tasks',
    durable=True,
    arguments={'x-max-priority': 10}
)

# Сообщение с высоким приоритетом
channel.basic_publish(
    exchange='',
    routing_key='tasks',
    body='Срочная задача!',
    properties=pika.BasicProperties(
        priority=10,  # Высший приоритет
        delivery_mode=2
    )
)

# Сообщение с нормальным приоритетом
channel.basic_publish(
    exchange='',
    routing_key='tasks',
    body='Обычная задача',
    properties=pika.BasicProperties(
        priority=5,  # Средний приоритет
        delivery_mode=2
    )
)

# Сообщение с низким приоритетом
channel.basic_publish(
    exchange='',
    routing_key='tasks',
    body='Фоновая задача',
    properties=pika.BasicProperties(
        priority=1,  # Низкий приоритет
        delivery_mode=2
    )
)

print("Сообщения с разными приоритетами отправлены")
```

### Система именованных приоритетов

```python
import pika
from enum import IntEnum

class TaskPriority(IntEnum):
    CRITICAL = 10
    HIGH = 7
    NORMAL = 5
    LOW = 3
    BACKGROUND = 1

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(
    queue='tasks',
    durable=True,
    arguments={'x-max-priority': 10}
)

def publish_task(task_data, priority=TaskPriority.NORMAL):
    """Публикация задачи с именованным приоритетом"""
    channel.basic_publish(
        exchange='',
        routing_key='tasks',
        body=str(task_data),
        properties=pika.BasicProperties(
            priority=int(priority),
            delivery_mode=2
        )
    )
    print(f"Задача отправлена: {task_data} (приоритет: {priority.name})")

# Использование
publish_task("Сервер упал!", TaskPriority.CRITICAL)
publish_task("Обработать заказ", TaskPriority.HIGH)
publish_task("Отправить отчет", TaskPriority.NORMAL)
publish_task("Очистить логи", TaskPriority.BACKGROUND)
```

## Потребление приоритетных сообщений

### Базовый consumer

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(
    queue='priority_queue',
    durable=True,
    arguments={'x-max-priority': 10}
)

def callback(ch, method, properties, body):
    priority = properties.priority or 0
    print(f"Получено [P:{priority}]: {body.decode()}")

    # Обработка сообщения
    process_message(body, priority)

    ch.basic_ack(delivery_tag=method.delivery_tag)

def process_message(body, priority):
    """Обработка с учетом приоритета"""
    if priority >= 8:
        print("  -> Критическая обработка")
    elif priority >= 5:
        print("  -> Стандартная обработка")
    else:
        print("  -> Фоновая обработка")

# Важно: установить prefetch_count для правильной работы приоритетов
channel.basic_qos(prefetch_count=1)

channel.basic_consume(
    queue='priority_queue',
    on_message_callback=callback,
    auto_ack=False
)

print("Ожидание сообщений...")
channel.start_consuming()
```

### Consumer с разной обработкой по приоритету

```python
import pika
import time
from concurrent.futures import ThreadPoolExecutor

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(
    queue='priority_tasks',
    durable=True,
    arguments={'x-max-priority': 10}
)

# Пулы потоков для разных приоритетов
critical_executor = ThreadPoolExecutor(max_workers=4)  # Больше ресурсов
normal_executor = ThreadPoolExecutor(max_workers=2)
background_executor = ThreadPoolExecutor(max_workers=1)  # Минимум ресурсов

def process_critical(body):
    print(f"[CRITICAL] Обработка: {body.decode()}")
    time.sleep(0.1)  # Быстрая обработка

def process_normal(body):
    print(f"[NORMAL] Обработка: {body.decode()}")
    time.sleep(0.5)

def process_background(body):
    print(f"[BACKGROUND] Обработка: {body.decode()}")
    time.sleep(1)  # Может быть медленной

def callback(ch, method, properties, body):
    priority = properties.priority or 0

    if priority >= 8:
        future = critical_executor.submit(process_critical, body)
    elif priority >= 4:
        future = normal_executor.submit(process_normal, body)
    else:
        future = background_executor.submit(process_background, body)

    # Ждем завершения перед ack
    future.result()
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_qos(prefetch_count=1)
channel.basic_consume(
    queue='priority_tasks',
    on_message_callback=callback,
    auto_ack=False
)

channel.start_consuming()
```

## Важность prefetch_count

### Проблема без prefetch

```python
"""
БЕЗ prefetch_count=1:

Очередь: [P:10] [P:5] [P:1] [P:10] [P:10]

Consumer подключается и получает ВСЕ сообщения сразу:
Consumer buffer: [P:10] [P:5] [P:1] [P:10] [P:10]

Теперь новое сообщение [P:10] поступает в очередь,
но consumer обрабатывает их в порядке получения, не по приоритету!

С prefetch_count=1:
- Consumer получает по одному сообщению
- Брокер выбирает следующее с наивысшим приоритетом
- Приоритеты работают правильно
"""

import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# КРИТИЧЕСКИ ВАЖНО для приоритетных очередей
channel.basic_qos(prefetch_count=1)

channel.basic_consume(
    queue='priority_queue',
    on_message_callback=callback,
    auto_ack=False
)
```

### Баланс между производительностью и приоритетами

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

"""
prefetch_count влияет на работу приоритетов:

prefetch_count=1:
  + Идеальное соблюдение приоритетов
  - Низкая производительность (round-trip на каждое сообщение)

prefetch_count=10:
  + Хорошая производительность
  - Приоритеты работают только между батчами

prefetch_count=100:
  + Максимальная производительность
  - Приоритеты практически не работают

Рекомендация: начните с 1-5 и увеличивайте при необходимости
"""

# Компромиссный вариант
channel.basic_qos(prefetch_count=5)
```

## Практические сценарии

### Система обработки заказов

```python
import pika
import json
from enum import IntEnum
from datetime import datetime

class OrderPriority(IntEnum):
    VIP_CUSTOMER = 10
    EXPRESS_DELIVERY = 8
    STANDARD = 5
    BULK_ORDER = 2

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(
    queue='orders',
    durable=True,
    arguments={'x-max-priority': 10}
)

def publish_order(order):
    """Публикация заказа с автоматическим определением приоритета"""
    priority = OrderPriority.STANDARD

    if order.get('customer_type') == 'vip':
        priority = OrderPriority.VIP_CUSTOMER
    elif order.get('delivery') == 'express':
        priority = OrderPriority.EXPRESS_DELIVERY
    elif order.get('items_count', 0) > 100:
        priority = OrderPriority.BULK_ORDER

    channel.basic_publish(
        exchange='',
        routing_key='orders',
        body=json.dumps({
            **order,
            'priority': int(priority),
            'created_at': datetime.utcnow().isoformat()
        }),
        properties=pika.BasicProperties(
            priority=int(priority),
            delivery_mode=2
        )
    )

    print(f"Заказ #{order['id']} отправлен (приоритет: {priority.name})")


# Пример заказов
publish_order({'id': 1, 'customer_type': 'vip', 'total': 5000})
publish_order({'id': 2, 'customer_type': 'regular', 'total': 100})
publish_order({'id': 3, 'delivery': 'express', 'total': 200})
publish_order({'id': 4, 'items_count': 150, 'total': 10000})
```

### Система уведомлений

```python
import pika
import json
from enum import IntEnum

class NotificationPriority(IntEnum):
    EMERGENCY = 10      # Критические алерты
    SECURITY = 9        # Безопасность
    TRANSACTION = 7     # Финансовые операции
    SYSTEM = 5          # Системные уведомления
    MARKETING = 2       # Маркетинг
    DIGEST = 1          # Дайджесты

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(
    queue='notifications',
    durable=True,
    arguments={'x-max-priority': 10}
)

def send_notification(user_id, message, notification_type):
    """Отправка уведомления с приоритетом по типу"""
    priority_map = {
        'emergency': NotificationPriority.EMERGENCY,
        'security': NotificationPriority.SECURITY,
        'transaction': NotificationPriority.TRANSACTION,
        'system': NotificationPriority.SYSTEM,
        'marketing': NotificationPriority.MARKETING,
        'digest': NotificationPriority.DIGEST
    }

    priority = priority_map.get(notification_type, NotificationPriority.SYSTEM)

    channel.basic_publish(
        exchange='',
        routing_key='notifications',
        body=json.dumps({
            'user_id': user_id,
            'message': message,
            'type': notification_type
        }),
        properties=pika.BasicProperties(
            priority=int(priority),
            delivery_mode=2
        )
    )


# Примеры
send_notification(1, "Вход с нового устройства!", "security")
send_notification(2, "Новая акция 50%", "marketing")
send_notification(1, "Сервер недоступен!", "emergency")
```

### Система задач с deadline

```python
import pika
import json
from datetime import datetime, timedelta

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(
    queue='deadline_tasks',
    durable=True,
    arguments={'x-max-priority': 10}
)

def calculate_priority_from_deadline(deadline):
    """Расчет приоритета на основе времени до дедлайна"""
    now = datetime.utcnow()

    if isinstance(deadline, str):
        deadline = datetime.fromisoformat(deadline)

    time_left = deadline - now

    if time_left <= timedelta(minutes=5):
        return 10  # Критический
    elif time_left <= timedelta(minutes=30):
        return 8
    elif time_left <= timedelta(hours=1):
        return 6
    elif time_left <= timedelta(hours=4):
        return 4
    else:
        return 2  # Есть время


def publish_deadline_task(task_data, deadline):
    """Публикация задачи с дедлайном"""
    priority = calculate_priority_from_deadline(deadline)

    channel.basic_publish(
        exchange='',
        routing_key='deadline_tasks',
        body=json.dumps({
            'task': task_data,
            'deadline': deadline.isoformat() if isinstance(deadline, datetime) else deadline
        }),
        properties=pika.BasicProperties(
            priority=priority,
            delivery_mode=2
        )
    )

    print(f"Задача отправлена: {task_data} (дедлайн: {deadline}, приоритет: {priority})")


# Использование
now = datetime.utcnow()

publish_deadline_task("Срочный отчет", now + timedelta(minutes=3))
publish_deadline_task("Еженедельный отчет", now + timedelta(hours=5))
publish_deadline_task("Ежемесячный отчет", now + timedelta(days=2))
```

## Мониторинг приоритетных очередей

### Статистика по приоритетам

```python
import pika
import requests

def get_priority_queue_stats(queue_name, vhost='%2F'):
    """Получение статистики приоритетной очереди"""
    response = requests.get(
        f'http://localhost:15672/api/queues/{vhost}/{queue_name}',
        auth=('guest', 'guest')
    )

    if response.status_code == 200:
        info = response.json()

        print(f"Очередь: {queue_name}")
        print(f"  Всего сообщений: {info['messages']}")
        print(f"  Готовых: {info['messages_ready']}")
        print(f"  Unacked: {info['messages_unacknowledged']}")

        arguments = info.get('arguments', {})
        max_priority = arguments.get('x-max-priority')

        if max_priority:
            print(f"  Max priority: {max_priority}")
            print(f"  Тип: Priority Queue")
        else:
            print(f"  Тип: Standard Queue")

        return info

    return None


# Мониторинг
def monitor_priority_distribution(channel, queue_name, sample_size=100):
    """Анализ распределения приоритетов (destructive - читает сообщения!)"""
    priorities = {}

    for _ in range(sample_size):
        method, properties, body = channel.basic_get(queue_name, auto_ack=False)

        if method:
            priority = properties.priority or 0
            priorities[priority] = priorities.get(priority, 0) + 1
            channel.basic_nack(method.delivery_tag, requeue=True)
        else:
            break

    print("Распределение приоритетов:")
    for p in sorted(priorities.keys(), reverse=True):
        count = priorities[p]
        bar = "#" * (count * 2)
        print(f"  P{p:2d}: {bar} ({count})")

    return priorities
```

## Best Practices

### 1. Используйте разумный диапазон приоритетов

```python
# ПРАВИЛЬНО - достаточно для большинства случаев
channel.queue_declare(
    queue='tasks',
    arguments={'x-max-priority': 10}
)

# ИЗБЕГАЙТЕ - слишком много уровней
channel.queue_declare(
    queue='tasks',
    arguments={'x-max-priority': 255}  # Большой overhead!
)
```

### 2. Всегда устанавливайте prefetch_count

```python
# ОБЯЗАТЕЛЬНО для приоритетных очередей
channel.basic_qos(prefetch_count=1)  # Или небольшое значение
```

### 3. Избегайте priority starvation

```python
"""
Priority Starvation: низкоприоритетные сообщения никогда не обрабатываются
если постоянно поступают высокоприоритетные.

Решения:
1. Время жизни (TTL) для всех сообщений
2. Aging - повышение приоритета старых сообщений
3. Separate queues с гарантированным processing
"""

import pika
from datetime import datetime, timedelta

def publish_with_aging(channel, queue, body, initial_priority):
    """Публикация с поддержкой aging"""
    channel.basic_publish(
        exchange='',
        routing_key=queue,
        body=body,
        properties=pika.BasicProperties(
            priority=initial_priority,
            headers={
                'created_at': datetime.utcnow().isoformat(),
                'initial_priority': initial_priority
            },
            delivery_mode=2
        )
    )


def apply_aging(properties):
    """Расчет aged приоритета"""
    headers = properties.headers or {}
    created_at = headers.get('created_at')
    initial_priority = headers.get('initial_priority', properties.priority or 0)

    if created_at:
        age = datetime.utcnow() - datetime.fromisoformat(created_at)
        # Повышаем приоритет на 1 каждые 5 минут
        age_boost = min(age.seconds // 300, 5)
        return min(initial_priority + age_boost, 10)

    return initial_priority
```

### 4. Документируйте приоритеты

```python
"""
Система приоритетов для проекта XYZ:

Priority | Name       | SLA        | Description
---------|------------|------------|-------------
10       | CRITICAL   | < 1 min    | Системные сбои, безопасность
8-9      | HIGH       | < 5 min    | Важные бизнес-операции
5-7      | NORMAL     | < 30 min   | Стандартные задачи
3-4      | LOW        | < 2 hours  | Отложенные операции
1-2      | BACKGROUND | Best effort| Фоновые задачи, отчеты
"""

from enum import IntEnum

class Priority(IntEnum):
    """Централизованное определение приоритетов"""
    CRITICAL = 10
    HIGH = 8
    NORMAL = 5
    LOW = 3
    BACKGROUND = 1

    @property
    def sla_seconds(self):
        sla_map = {10: 60, 8: 300, 5: 1800, 3: 7200, 1: None}
        return sla_map.get(self.value)
```

## Частые ошибки

### 1. Отсутствие x-max-priority при создании очереди

```python
# НЕПРАВИЛЬНО - обычная очередь, приоритеты игнорируются
channel.queue_declare(queue='tasks')
channel.basic_publish(
    ...,
    properties=pika.BasicProperties(priority=10)  # Игнорируется!
)

# ПРАВИЛЬНО
channel.queue_declare(queue='tasks', arguments={'x-max-priority': 10})
```

### 2. Большой prefetch_count

```python
# НЕПРАВИЛЬНО - приоритеты не работают
channel.basic_qos(prefetch_count=100)

# ПРАВИЛЬНО
channel.basic_qos(prefetch_count=1)
```

### 3. Приоритет выше x-max-priority

```python
channel.queue_declare(queue='q', arguments={'x-max-priority': 5})

# Приоритет 10 будет приведен к 5!
channel.basic_publish(
    ...,
    properties=pika.BasicProperties(priority=10)  # Станет 5
)
```

## Заключение

Priority Queues - мощный инструмент для управления порядком обработки сообщений:

1. Используйте x-max-priority от 5 до 10 для большинства случаев
2. Всегда устанавливайте prefetch_count=1 или небольшое значение
3. Документируйте систему приоритетов
4. Защищайтесь от priority starvation
5. Мониторьте распределение приоритетов в очередях

---

[prev: 04-message-ttl](./04-message-ttl.md) | [next: 06-prefetch](./06-prefetch.md)
