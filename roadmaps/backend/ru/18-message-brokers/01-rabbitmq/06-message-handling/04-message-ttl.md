# TTL сообщений (Time-To-Live)

## Введение

TTL (Time-To-Live) - это механизм RabbitMQ, определяющий время жизни сообщений или очередей. После истечения TTL сообщение автоматически удаляется из очереди или перемещается в Dead Letter Exchange. TTL помогает управлять устаревшими данными и предотвращать переполнение очередей.

## Типы TTL

В RabbitMQ существует два способа установки TTL:
1. **Per-Queue TTL** - устанавливается для всей очереди
2. **Per-Message TTL** - устанавливается для каждого сообщения индивидуально

```
Per-Queue TTL: Все сообщения в очереди имеют одинаковый TTL
Per-Message TTL: Каждое сообщение имеет свой TTL
```

## Per-Queue TTL

### Установка TTL для очереди

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# TTL 60 секунд (60000 миллисекунд) для всех сообщений в очереди
channel.queue_declare(
    queue='ttl_queue',
    durable=True,
    arguments={
        'x-message-ttl': 60000  # TTL в миллисекундах
    }
)

# Публикуем сообщение
channel.basic_publish(
    exchange='',
    routing_key='ttl_queue',
    body='Это сообщение исчезнет через 60 секунд'
)

print("Сообщение отправлено, TTL = 60 секунд")
```

### Примеры различных TTL

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Очередь для кратковременных уведомлений (5 минут)
channel.queue_declare(
    queue='notifications',
    durable=True,
    arguments={
        'x-message-ttl': 300000  # 5 минут
    }
)

# Очередь для задач (24 часа)
channel.queue_declare(
    queue='daily_tasks',
    durable=True,
    arguments={
        'x-message-ttl': 86400000  # 24 часа
    }
)

# Очередь для временных данных (30 секунд)
channel.queue_declare(
    queue='temp_data',
    durable=True,
    arguments={
        'x-message-ttl': 30000  # 30 секунд
    }
)

# Таблица преобразования
TTL_VALUES = {
    '30 секунд': 30000,
    '1 минута': 60000,
    '5 минут': 300000,
    '1 час': 3600000,
    '24 часа': 86400000,
    '7 дней': 604800000,
}
```

## Per-Message TTL

### Установка TTL для отдельного сообщения

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(queue='mixed_ttl_queue', durable=True)

# Сообщение с TTL 10 секунд
channel.basic_publish(
    exchange='',
    routing_key='mixed_ttl_queue',
    body='Срочное сообщение',
    properties=pika.BasicProperties(
        expiration='10000'  # TTL в миллисекундах (строка!)
    )
)

# Сообщение с TTL 1 час
channel.basic_publish(
    exchange='',
    routing_key='mixed_ttl_queue',
    body='Обычное сообщение',
    properties=pika.BasicProperties(
        expiration='3600000'  # 1 час
    )
)

# Сообщение без TTL (живет вечно)
channel.basic_publish(
    exchange='',
    routing_key='mixed_ttl_queue',
    body='Важное сообщение без срока годности'
)

print("Сообщения с разным TTL отправлены")
```

### Комбинирование Queue и Message TTL

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Очередь с TTL 1 час
channel.queue_declare(
    queue='combo_queue',
    durable=True,
    arguments={
        'x-message-ttl': 3600000  # 1 час
    }
)

# Сообщение с TTL 5 минут (победит меньший TTL)
channel.basic_publish(
    exchange='',
    routing_key='combo_queue',
    body='Это сообщение исчезнет через 5 минут',
    properties=pika.BasicProperties(
        expiration='300000'  # 5 минут < 1 час
    )
)

# Сообщение с TTL 2 часа (победит Queue TTL - 1 час)
channel.basic_publish(
    exchange='',
    routing_key='combo_queue',
    body='Это сообщение исчезнет через 1 час (Queue TTL)',
    properties=pika.BasicProperties(
        expiration='7200000'  # 2 часа > 1 час
    )
)

"""
Правило: Применяется МЕНЬШИЙ из двух TTL
- Queue TTL: 1 час
- Message TTL: 5 минут → Результат: 5 минут
- Message TTL: 2 часа → Результат: 1 час (Queue TTL)
"""
```

## Queue Expiry (TTL для очередей)

### Автоматическое удаление неиспользуемых очередей

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Очередь будет удалена через 10 минут неактивности
channel.queue_declare(
    queue='auto_delete_queue',
    durable=False,
    arguments={
        'x-expires': 600000  # 10 минут
    }
)

"""
x-expires:
- Очередь удаляется если нет consumers И нет операций
- Таймер сбрасывается при:
  - Подключении consumer
  - basic.get
  - queue.declare с теми же параметрами
"""
```

### Комбинация message-ttl и expires

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Очередь для временных сессий
channel.queue_declare(
    queue='session_queue',
    durable=False,
    arguments={
        'x-message-ttl': 300000,  # Сообщения живут 5 минут
        'x-expires': 600000       # Очередь удаляется через 10 минут
    }
)
```

## Dead Letter Exchange с TTL

### Настройка DLX для истекших сообщений

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Dead Letter Exchange
channel.exchange_declare(
    exchange='dlx',
    exchange_type='direct',
    durable=True
)

# Очередь для "мертвых" сообщений
channel.queue_declare(
    queue='dead_letters',
    durable=True
)

channel.queue_bind(
    queue='dead_letters',
    exchange='dlx',
    routing_key='expired'
)

# Основная очередь с TTL и DLX
channel.queue_declare(
    queue='main_queue',
    durable=True,
    arguments={
        'x-message-ttl': 30000,          # 30 секунд
        'x-dead-letter-exchange': 'dlx',
        'x-dead-letter-routing-key': 'expired'
    }
)

# Публикация сообщения
channel.basic_publish(
    exchange='',
    routing_key='main_queue',
    body='Сообщение попадет в DLQ через 30 секунд'
)

print("Сообщение отправлено, через 30с попадет в dead_letters")
```

### Реализация отложенных сообщений через TTL + DLX

```python
import pika
import json
from datetime import datetime

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

def setup_delayed_messaging():
    """Настройка системы отложенных сообщений"""

    # Exchange для обработки
    channel.exchange_declare(
        exchange='processing',
        exchange_type='direct',
        durable=True
    )

    # Очередь обработки
    channel.queue_declare(queue='process_queue', durable=True)
    channel.queue_bind(
        queue='process_queue',
        exchange='processing',
        routing_key='ready'
    )

    # Delay queues для разных интервалов
    delays = [5000, 30000, 60000, 300000]  # 5с, 30с, 1мин, 5мин

    for delay in delays:
        queue_name = f'delay_{delay}ms'

        channel.queue_declare(
            queue=queue_name,
            durable=True,
            arguments={
                'x-message-ttl': delay,
                'x-dead-letter-exchange': 'processing',
                'x-dead-letter-routing-key': 'ready'
            }
        )

        print(f"Создана очередь задержки: {queue_name}")


def publish_delayed(message, delay_ms):
    """Публикация сообщения с задержкой"""
    queue_name = f'delay_{delay_ms}ms'

    channel.basic_publish(
        exchange='',
        routing_key=queue_name,
        body=json.dumps({
            'message': message,
            'scheduled_at': datetime.utcnow().isoformat(),
            'delay_ms': delay_ms
        }),
        properties=pika.BasicProperties(
            delivery_mode=2
        )
    )

    print(f"Сообщение запланировано на выполнение через {delay_ms}мс")


# Использование
setup_delayed_messaging()
publish_delayed("Отправить напоминание", 60000)  # Через 1 минуту
publish_delayed("Проверить статус", 300000)      # Через 5 минут
```

## Мониторинг TTL

### Проверка настроек очереди

```python
import pika
import requests

# Через Management API
def get_queue_ttl_info(queue_name, vhost='%2F'):
    response = requests.get(
        f'http://localhost:15672/api/queues/{vhost}/{queue_name}',
        auth=('guest', 'guest')
    )

    if response.status_code == 200:
        info = response.json()
        arguments = info.get('arguments', {})

        print(f"Очередь: {queue_name}")
        print(f"  Message TTL: {arguments.get('x-message-ttl', 'не установлен')}")
        print(f"  Queue Expiry: {arguments.get('x-expires', 'не установлен')}")
        print(f"  Сообщений: {info['messages']}")
        print(f"  Готовых: {info['messages_ready']}")

        return info
    else:
        print(f"Ошибка: {response.status_code}")
        return None


# Через AMQP (passive declare)
def check_queue_exists(channel, queue_name):
    try:
        result = channel.queue_declare(queue=queue_name, passive=True)
        print(f"Очередь {queue_name} существует, сообщений: {result.method.message_count}")
        return True
    except pika.exceptions.ChannelClosedByBroker:
        print(f"Очередь {queue_name} не существует")
        return False


# Использование
get_queue_ttl_info('ttl_queue')
```

## Практические сценарии

### Кеш с автоматической инвалидацией

```python
import pika
import json

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

def create_cache_queue(cache_name, ttl_seconds):
    """Создание кеширующей очереди"""
    channel.queue_declare(
        queue=f'cache_{cache_name}',
        durable=True,
        arguments={
            'x-message-ttl': ttl_seconds * 1000,
            'x-max-length': 1,  # Только последнее значение
            'x-overflow': 'drop-head'
        }
    )


def cache_set(cache_name, value, ttl_seconds=60):
    """Сохранение значения в кеш"""
    create_cache_queue(cache_name, ttl_seconds)

    channel.basic_publish(
        exchange='',
        routing_key=f'cache_{cache_name}',
        body=json.dumps(value),
        properties=pika.BasicProperties(
            delivery_mode=2,
            expiration=str(ttl_seconds * 1000)
        )
    )


def cache_get(cache_name):
    """Получение значения из кеша"""
    method, properties, body = channel.basic_get(
        queue=f'cache_{cache_name}',
        auto_ack=False
    )

    if method:
        # Возвращаем сообщение обратно
        channel.basic_nack(method.delivery_tag, requeue=True)
        return json.loads(body)

    return None


# Использование
cache_set('user_123', {'name': 'John', 'email': 'john@example.com'}, ttl_seconds=300)
user = cache_get('user_123')
print(f"Из кеша: {user}")
```

### Очередь с приоритетной обработкой по времени

```python
import pika
import json
from datetime import datetime, timedelta

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

def setup_priority_ttl_system():
    """Система очередей с разным TTL по приоритету"""

    priorities = {
        'critical': 5000,     # 5 секунд - обработать немедленно
        'high': 30000,        # 30 секунд
        'normal': 300000,     # 5 минут
        'low': 3600000        # 1 час
    }

    # DLX для необработанных сообщений
    channel.exchange_declare(
        exchange='unprocessed_dlx',
        exchange_type='fanout',
        durable=True
    )

    channel.queue_declare(queue='unprocessed_tasks', durable=True)
    channel.queue_bind(queue='unprocessed_tasks', exchange='unprocessed_dlx')

    # Создаем очереди для каждого приоритета
    for priority, ttl in priorities.items():
        channel.queue_declare(
            queue=f'tasks_{priority}',
            durable=True,
            arguments={
                'x-message-ttl': ttl,
                'x-dead-letter-exchange': 'unprocessed_dlx'
            }
        )
        print(f"Очередь tasks_{priority}: TTL = {ttl}мс")


def publish_task(task_data, priority='normal'):
    """Публикация задачи с указанным приоритетом"""
    channel.basic_publish(
        exchange='',
        routing_key=f'tasks_{priority}',
        body=json.dumps({
            'task': task_data,
            'priority': priority,
            'created_at': datetime.utcnow().isoformat()
        }),
        properties=pika.BasicProperties(delivery_mode=2)
    )


# Использование
setup_priority_ttl_system()

publish_task({'type': 'alert', 'message': 'Server down!'}, priority='critical')
publish_task({'type': 'report', 'id': 123}, priority='normal')
publish_task({'type': 'cleanup', 'days': 30}, priority='low')
```

## Best Practices

### 1. Выбирайте правильный тип TTL

```python
# Per-Queue TTL - когда все сообщения имеют одинаковый срок жизни
channel.queue_declare(
    queue='logs',
    arguments={'x-message-ttl': 86400000}  # Все логи хранятся 24 часа
)

# Per-Message TTL - когда сообщения имеют разный срок
channel.basic_publish(
    exchange='',
    routing_key='notifications',
    body='Urgent!',
    properties=pika.BasicProperties(expiration='60000')  # Только это - 1 минута
)
```

### 2. Всегда используйте DLX с TTL

```python
# Без DLX сообщения просто удаляются - нет возможности их отследить
channel.queue_declare(
    queue='important_queue',
    arguments={
        'x-message-ttl': 300000,
        'x-dead-letter-exchange': 'dlx',  # Сохраняем истекшие
        'x-dead-letter-routing-key': 'expired'
    }
)
```

### 3. Мониторьте истекающие сообщения

```python
import requests

def monitor_expired_messages():
    """Мониторинг DLQ для истекших сообщений"""
    response = requests.get(
        'http://localhost:15672/api/queues/%2F/dead_letters',
        auth=('guest', 'guest')
    )

    if response.status_code == 200:
        queue = response.json()
        messages = queue.get('messages', 0)

        if messages > 100:
            print(f"ПРЕДУПРЕЖДЕНИЕ: {messages} истекших сообщений в DLQ!")

        return messages

    return 0
```

## Частые ошибки

### 1. Неправильный тип expiration

```python
# НЕПРАВИЛЬНО - expiration должен быть строкой!
properties = pika.BasicProperties(expiration=60000)  # Ошибка!

# ПРАВИЛЬНО
properties = pika.BasicProperties(expiration='60000')  # Строка
```

### 2. Ожидание немедленного удаления

```python
"""
ВАЖНО: Сообщения с Per-Message TTL удаляются только при попытке
доставки или когда они оказываются в начале очереди.

Если в очереди:
- Msg1 (TTL: 1 час)
- Msg2 (TTL: 1 минута) <- Не будет удалено пока Msg1 не обработано!

Решение: используйте Per-Queue TTL или разные очереди для разных TTL
"""
```

### 3. Забытая настройка x-expires

```python
# Очередь может накапливать "зомби" очереди
# Если consumers отключились и никто не читает

# ПРАВИЛЬНО - добавляем x-expires для cleanup
channel.queue_declare(
    queue='temp_queue',
    durable=False,
    arguments={
        'x-message-ttl': 60000,
        'x-expires': 120000  # Удалить очередь через 2 минуты неактивности
    }
)
```

## Заключение

TTL - мощный инструмент для управления жизненным циклом сообщений:

1. Используйте Per-Queue TTL для однородных очередей
2. Используйте Per-Message TTL для гибкости
3. Всегда настраивайте DLX для отслеживания истекших сообщений
4. Помните о порядке удаления в Per-Message TTL
5. Комбинируйте с x-expires для автоматической очистки очередей
