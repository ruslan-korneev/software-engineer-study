# Персистентность сообщений

[prev: 02-publisher-confirms](./02-publisher-confirms.md) | [next: 04-message-ttl](./04-message-ttl.md)

---

## Введение

Персистентность (persistence) в RabbitMQ - это механизм сохранения сообщений на диск для защиты от потери данных при перезапуске брокера или сбое системы. Без персистентности все сообщения хранятся только в оперативной памяти и теряются при остановке RabbitMQ.

## Компоненты персистентности

Для полной персистентности необходимо настроить три компонента:
1. **Durable Queue** - очередь, которая переживает перезапуск
2. **Persistent Messages** - сообщения, сохраняемые на диск
3. **Durable Exchange** - обменник, сохраняющийся при перезапуске

```
Publisher → Durable Exchange → Durable Queue → Disk Storage
                                     ↓
                               Persistent
                               Messages
```

## Durable Queues (Устойчивые очереди)

### Создание durable очереди

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Создаем durable очередь
channel.queue_declare(
    queue='persistent_queue',
    durable=True  # Очередь переживет перезапуск
)

print("Durable очередь создана")
```

### Сравнение durable и non-durable очередей

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Non-durable очередь (по умолчанию)
channel.queue_declare(
    queue='temporary_queue',
    durable=False  # Будет удалена при перезапуске RabbitMQ
)

# Durable очередь
channel.queue_declare(
    queue='permanent_queue',
    durable=True  # Сохранится при перезапуске
)

# ВНИМАНИЕ: Нельзя изменить durable флаг существующей очереди!
# Это вызовет ошибку:
# channel.queue_declare(queue='permanent_queue', durable=False)  # Ошибка!
```

### Дополнительные параметры очереди

```python
channel.queue_declare(
    queue='advanced_queue',
    durable=True,
    arguments={
        'x-message-ttl': 86400000,  # TTL 24 часа
        'x-max-length': 100000,     # Максимум 100k сообщений
        'x-overflow': 'reject-publish',  # Отклонять при переполнении
        'x-queue-type': 'quorum'    # Quorum очередь для высокой надежности
    }
)
```

## Persistent Messages (Персистентные сообщения)

### Публикация персистентных сообщений

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(queue='persistent_queue', durable=True)

# Персистентное сообщение
channel.basic_publish(
    exchange='',
    routing_key='persistent_queue',
    body='Важные данные',
    properties=pika.BasicProperties(
        delivery_mode=2  # 2 = persistent, 1 = transient
    )
)

print("Персистентное сообщение отправлено")
```

### Режимы доставки (delivery_mode)

```python
import pika

# Transient (непостоянное) сообщение - delivery_mode=1
transient_props = pika.BasicProperties(
    delivery_mode=1  # Хранится только в памяти
)

# Persistent (постоянное) сообщение - delivery_mode=2
persistent_props = pika.BasicProperties(
    delivery_mode=2  # Сохраняется на диск
)

# Используем pika константы для читаемости
from pika.spec import PERSISTENT_DELIVERY_MODE, TRANSIENT_DELIVERY_MODE

persistent_props = pika.BasicProperties(
    delivery_mode=PERSISTENT_DELIVERY_MODE
)
```

### Полный пример персистентной публикации

```python
import pika
import json
from datetime import datetime

def create_persistent_publisher(host='localhost'):
    connection = pika.BlockingConnection(pika.ConnectionParameters(host))
    channel = connection.channel()

    # Durable очередь
    channel.queue_declare(queue='orders', durable=True)

    # Включаем подтверждения для надежности
    channel.confirm_delivery()

    return connection, channel


def publish_order(channel, order_data):
    """Публикация заказа с гарантией сохранения"""
    message = json.dumps({
        'order': order_data,
        'timestamp': datetime.utcnow().isoformat(),
        'version': '1.0'
    })

    try:
        channel.basic_publish(
            exchange='',
            routing_key='orders',
            body=message,
            properties=pika.BasicProperties(
                delivery_mode=2,  # Persistent
                content_type='application/json',
                content_encoding='utf-8'
            ),
            mandatory=True
        )
        print(f"Заказ сохранен: {order_data['id']}")
        return True
    except pika.exceptions.UnroutableError:
        print("Ошибка маршрутизации")
        return False


# Использование
connection, channel = create_persistent_publisher()

order = {'id': 12345, 'product': 'Laptop', 'quantity': 1}
publish_order(channel, order)

connection.close()
```

## Durable Exchanges (Устойчивые обменники)

### Создание durable exchange

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Durable direct exchange
channel.exchange_declare(
    exchange='orders_exchange',
    exchange_type='direct',
    durable=True  # Сохранится при перезапуске
)

# Durable topic exchange
channel.exchange_declare(
    exchange='events_exchange',
    exchange_type='topic',
    durable=True
)

# Durable fanout exchange
channel.exchange_declare(
    exchange='broadcast_exchange',
    exchange_type='fanout',
    durable=True
)
```

### Полная цепочка персистентности

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# 1. Durable Exchange
channel.exchange_declare(
    exchange='persistent_exchange',
    exchange_type='direct',
    durable=True
)

# 2. Durable Queue
channel.queue_declare(
    queue='persistent_queue',
    durable=True
)

# 3. Binding (привязка)
channel.queue_bind(
    queue='persistent_queue',
    exchange='persistent_exchange',
    routing_key='important'
)

# 4. Persistent Message
channel.basic_publish(
    exchange='persistent_exchange',
    routing_key='important',
    body='Критически важные данные',
    properties=pika.BasicProperties(
        delivery_mode=2
    )
)

print("Полная цепочка персистентности настроена")
```

## Как работает персистентность на диске

### Структура хранения

RabbitMQ использует два механизма хранения:
1. **Message Store** - хранит тела сообщений
2. **Queue Index** - хранит метаданные и позиции сообщений

```python
# Расположение данных (по умолчанию)
# Linux: /var/lib/rabbitmq/mnesia/
# macOS: /usr/local/var/lib/rabbitmq/mnesia/
# Windows: %APPDATA%\RabbitMQ\db\

# Структура директорий:
# mnesia/
#   rabbit@hostname/
#     msg_stores/
#       vhosts/
#         628WB79CIFDYO9LJI6DKMI09L/  # ID виртуального хоста
#           msg_store_persistent/     # Персистентные сообщения
#           msg_store_transient/      # Временные сообщения
#     queues/
#       <queue_id>/
#         journal.jif                 # Журнал очереди
```

### Процесс сохранения

```python
"""
Процесс персистентности:

1. Сообщение приходит в RabbitMQ
2. Записывается в write-ahead log (WAL)
3. Сообщение помещается в очередь в памяти
4. Асинхронно сбрасывается на диск (fsync)
5. Подтверждение отправляется издателю (если confirm mode)

Важно: fsync может вызывать задержки!
"""

import pika
import time

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(queue='bench_queue', durable=True)
channel.confirm_delivery()

# Замер времени с персистентностью
start = time.time()
for i in range(1000):
    channel.basic_publish(
        exchange='',
        routing_key='bench_queue',
        body=f'Message {i}',
        properties=pika.BasicProperties(delivery_mode=2)
    )
persistent_time = time.time() - start

# Замер без персистентности
channel.queue_declare(queue='transient_queue', durable=False)
start = time.time()
for i in range(1000):
    channel.basic_publish(
        exchange='',
        routing_key='transient_queue',
        body=f'Message {i}',
        properties=pika.BasicProperties(delivery_mode=1)
    )
transient_time = time.time() - start

print(f"Персистентные: {persistent_time:.2f}с")
print(f"Транзиентные: {transient_time:.2f}с")
print(f"Разница: {persistent_time/transient_time:.1f}x")
```

## Quorum Queues - современный подход

### Создание quorum очереди

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Quorum queue - рекомендуется для продакшена
channel.queue_declare(
    queue='quorum_queue',
    durable=True,  # Обязательно для quorum
    arguments={
        'x-queue-type': 'quorum',
        'x-quorum-initial-group-size': 3  # Минимум 3 реплики
    }
)

# Публикация в quorum queue
channel.basic_publish(
    exchange='',
    routing_key='quorum_queue',
    body='Высоконадежное сообщение',
    properties=pika.BasicProperties(
        delivery_mode=2
    )
)
```

### Преимущества Quorum Queues

```python
"""
Quorum Queues vs Classic Durable Queues:

1. Репликация на основе Raft консенсуса
2. Автоматическое восстановление при сбоях
3. Более предсказуемая производительность
4. Лучшая защита от потери данных

Ограничения:
- Только durable (нельзя сделать transient)
- Не поддерживают некоторые фичи classic очередей
- Требуют больше ресурсов (память, сеть)
"""

import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Проверка типа очереди
result = channel.queue_declare(
    queue='quorum_queue',
    durable=True,
    passive=True  # Только проверка, не создание
)

# Получение информации через Management API
import requests

response = requests.get(
    'http://localhost:15672/api/queues/%2F/quorum_queue',
    auth=('guest', 'guest')
)

queue_info = response.json()
print(f"Тип очереди: {queue_info.get('type', 'classic')}")
print(f"Сообщений: {queue_info['messages']}")
```

## Lazy Queues - для больших объемов

### Настройка lazy queue

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Lazy queue - сообщения сразу на диск
channel.queue_declare(
    queue='lazy_queue',
    durable=True,
    arguments={
        'x-queue-mode': 'lazy'
    }
)

"""
Lazy Queues:
- Сообщения сразу сохраняются на диск
- Минимальное использование RAM
- Медленнее при чтении (нужно читать с диска)
- Подходит для очередей с миллионами сообщений
"""
```

## Гарантии и ограничения

### Что НЕ гарантирует персистентность

```python
"""
Персистентность НЕ защищает от:

1. Потери сообщения до записи на диск
   - Используйте publisher confirms!

2. Потери при сбое диска
   - Используйте RAID, репликацию

3. Потери в transit
   - Используйте TLS, надежную сеть

4. Потери при полном заполнении диска
   - Мониторьте место на диске

5. Потери при OOM killer
   - Настройте лимиты памяти
"""

import pika

def reliable_publish(channel, queue, message):
    """Максимально надежная публикация"""

    # 1. Durable queue
    channel.queue_declare(queue=queue, durable=True)

    # 2. Enable confirms
    channel.confirm_delivery()

    try:
        # 3. Persistent message + mandatory
        channel.basic_publish(
            exchange='',
            routing_key=queue,
            body=message,
            properties=pika.BasicProperties(delivery_mode=2),
            mandatory=True
        )

        # 4. Wait for disk sync (implicit with confirms)
        return True

    except pika.exceptions.UnroutableError:
        return False
    except pika.exceptions.NackError:
        return False
```

## Best Practices

### 1. Всегда используйте полную цепочку персистентности

```python
# ПРАВИЛЬНО - полная персистентность
channel.exchange_declare(exchange='ex', exchange_type='direct', durable=True)
channel.queue_declare(queue='q', durable=True)
channel.queue_bind(queue='q', exchange='ex', routing_key='key')
channel.basic_publish(
    exchange='ex',
    routing_key='key',
    body='data',
    properties=pika.BasicProperties(delivery_mode=2)
)

# НЕПРАВИЛЬНО - частичная персистентность
channel.queue_declare(queue='q', durable=True)
channel.basic_publish(
    exchange='',
    routing_key='q',
    body='data',
    properties=pika.BasicProperties(delivery_mode=1)  # Не персистентное!
)
```

### 2. Используйте publisher confirms

```python
channel.confirm_delivery()

try:
    channel.basic_publish(...)
except pika.exceptions.NackError:
    # Сообщение НЕ записано на диск
    handle_failure()
```

### 3. Мониторьте дисковую активность

```python
# Проверка состояния через API
import requests

response = requests.get(
    'http://localhost:15672/api/nodes',
    auth=('guest', 'guest')
)

for node in response.json():
    print(f"Node: {node['name']}")
    print(f"  Disk free: {node['disk_free'] / 1024**3:.1f} GB")
    print(f"  Disk alarm: {node['disk_free_alarm']}")
    print(f"  Memory used: {node['mem_used'] / 1024**3:.1f} GB")
```

### 4. Настройте алармы

```ini
# rabbitmq.conf
disk_free_limit.absolute = 5GB
disk_free_limit.relative = 1.5

vm_memory_high_watermark.relative = 0.6
```

## Частые ошибки

### 1. Персистентное сообщение в non-durable очередь

```python
# НЕПРАВИЛЬНО - сообщение потеряется при перезапуске
channel.queue_declare(queue='temp', durable=False)
channel.basic_publish(
    exchange='',
    routing_key='temp',
    body='data',
    properties=pika.BasicProperties(delivery_mode=2)
)
```

### 2. Игнорирование producer confirms

```python
# НЕПРАВИЛЬНО - нет гарантии записи на диск
channel.basic_publish(exchange='', routing_key='q', body='data')

# ПРАВИЛЬНО
channel.confirm_delivery()
channel.basic_publish(exchange='', routing_key='q', body='data')
```

### 3. Недостаточно места на диске

```python
"""
При заполнении диска RabbitMQ:
1. Включает disk alarm
2. Блокирует всех publishers
3. Не принимает новые сообщения

Решение:
- Мониторинг свободного места
- Настройка TTL для старых сообщений
- Настройка max-length для очередей
"""
```

## Заключение

Персистентность - критически важная функция для надежных систем обмена сообщениями. Помните:

1. Используйте полную цепочку: durable exchange + durable queue + persistent message
2. Всегда включайте publisher confirms
3. Рассмотрите quorum queues для продакшена
4. Мониторьте дисковое пространство и производительность
5. Персистентность снижает производительность - это компромисс

---

[prev: 02-publisher-confirms](./02-publisher-confirms.md) | [next: 04-message-ttl](./04-message-ttl.md)
