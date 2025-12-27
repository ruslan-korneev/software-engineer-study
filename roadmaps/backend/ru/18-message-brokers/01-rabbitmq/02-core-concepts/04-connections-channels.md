# Connections и Channels

## Обзор

В RabbitMQ **Connection** и **Channel** — это два уровня абстракции для взаимодействия с брокером. Понимание их различий критически важно для построения производительных и надёжных приложений.

## Connection (Соединение)

**Connection** — это TCP-соединение между клиентом и RabbitMQ сервером.

### Характеристики Connection

- Одно TCP-соединение на процесс
- Использует порт 5672 (AMQP) или 5671 (AMQPS)
- Ресурсоёмкий процесс установки (TLS handshake, аутентификация)
- Поддерживает heartbeat для проверки живости
- Требует аутентификации (username/password)

```
┌─────────────────────────────────────────────────────────┐
│                    APPLICATION                           │
├─────────────────────────────────────────────────────────┤
│                    CONNECTION                            │
│            (TCP Socket + AMQP Protocol)                  │
├────────────┬────────────┬────────────┬─────────────────┤
│  Channel 1 │  Channel 2 │  Channel 3 │    ...          │
│   (Pub)    │   (Pub)    │   (Sub)    │                 │
└────────────┴────────────┴────────────┴─────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                    RabbitMQ Server                       │
└─────────────────────────────────────────────────────────┘
```

### Создание Connection

```python
import pika

# Простое соединение
connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)

# Соединение с параметрами
connection = pika.BlockingConnection(
    pika.ConnectionParameters(
        host='localhost',
        port=5672,
        virtual_host='/',
        credentials=pika.PlainCredentials('guest', 'guest'),
        heartbeat=600,                    # Интервал heartbeat (сек)
        blocked_connection_timeout=300,   # Таймаут блокировки
        connection_attempts=3,            # Попытки подключения
        retry_delay=5,                    # Задержка между попытками
        socket_timeout=10                 # Таймаут сокета
    )
)

# Соединение через URL
connection = pika.BlockingConnection(
    pika.URLParameters('amqp://guest:guest@localhost:5672/%2F')
)

print(f"Connected: {connection.is_open}")
connection.close()
```

## Channel (Канал)

**Channel** — это виртуальное соединение внутри Connection. Все AMQP-операции выполняются через каналы.

### Характеристики Channel

- Лёгкий процесс создания
- Мультиплексирование внутри одного Connection
- Каждый channel имеет уникальный ID
- Операции на channel — атомарные
- Рекомендуется один channel на поток

### Создание Channel

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)

# Создание канала
channel = connection.channel()

# Операции через канал
channel.queue_declare(queue='test')
channel.basic_publish(exchange='', routing_key='test', body='Hello')

print(f"Channel number: {channel.channel_number}")
print(f"Channel is open: {channel.is_open}")

channel.close()
connection.close()
```

## Разница между Connection и Channel

| Аспект | Connection | Channel |
|--------|------------|---------|
| Тип | TCP-соединение | Виртуальное соединение |
| Создание | Дорогое (TLS, auth) | Дешёвое |
| Количество | 1 на приложение | Много на Connection |
| Thread-safety | Да | Нет (1 на поток) |
| Ресурсы | Высокие | Низкие |
| Закрытие | Закрывает все каналы | Не влияет на другие |

## Управление Connection

### Connection с автоматическим переподключением

```python
import pika
import time
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


class ResilientConnection:
    """Connection с автоматическим переподключением"""

    def __init__(self, host='localhost', max_retries=5):
        self.host = host
        self.max_retries = max_retries
        self.connection = None
        self.channel = None

    def connect(self):
        """Установка соединения с retry логикой"""
        for attempt in range(self.max_retries):
            try:
                self.connection = pika.BlockingConnection(
                    pika.ConnectionParameters(
                        host=self.host,
                        heartbeat=600,
                        blocked_connection_timeout=300
                    )
                )
                self.channel = self.connection.channel()
                logger.info(f"Connected to RabbitMQ (attempt {attempt + 1})")
                return True

            except pika.exceptions.AMQPConnectionError as e:
                logger.warning(f"Connection failed (attempt {attempt + 1}): {e}")
                if attempt < self.max_retries - 1:
                    time.sleep(2 ** attempt)  # Exponential backoff
                else:
                    raise

        return False

    def ensure_connection(self):
        """Проверка и восстановление соединения"""
        if not self.connection or self.connection.is_closed:
            self.connect()
        if not self.channel or self.channel.is_closed:
            self.channel = self.connection.channel()

    def close(self):
        """Закрытие соединения"""
        if self.connection and self.connection.is_open:
            self.connection.close()
            logger.info("Connection closed")


# Использование
conn = ResilientConnection()
conn.connect()

# Работа с каналом
conn.channel.queue_declare(queue='test')

conn.close()
```

### Heartbeat и обнаружение разрывов

```python
import pika

# Настройка heartbeat
connection = pika.BlockingConnection(
    pika.ConnectionParameters(
        host='localhost',
        heartbeat=60,  # Heartbeat каждые 60 секунд
        # Если нет ответа за 2*heartbeat — соединение закрывается
    )
)

# Обработка закрытия соединения
def on_connection_closed(connection, reason):
    print(f"Connection closed: {reason}")

# Для SelectConnection/AsyncioConnection
# connection.add_on_close_callback(on_connection_closed)
```

## Управление Channels

### Несколько каналов в одном соединении

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)

# Канал для публикации
publish_channel = connection.channel()
publish_channel.queue_declare(queue='tasks')

# Канал для потребления
consume_channel = connection.channel()
consume_channel.queue_declare(queue='results')

# Публикация через один канал
publish_channel.basic_publish(
    exchange='',
    routing_key='tasks',
    body='Task data'
)

# Потребление через другой канал
def callback(ch, method, props, body):
    print(f"Received: {body}")
    ch.basic_ack(delivery_tag=method.delivery_tag)

consume_channel.basic_consume(
    queue='results',
    on_message_callback=callback
)

# Очистка
publish_channel.close()
consume_channel.close()
connection.close()
```

### Channel Pool (Пул каналов)

```python
import pika
from queue import Queue
from threading import Lock
from contextlib import contextmanager


class ChannelPool:
    """Пул каналов для многопоточных приложений"""

    def __init__(self, connection, pool_size=10):
        self.connection = connection
        self.pool_size = pool_size
        self.pool = Queue(maxsize=pool_size)
        self.lock = Lock()

        # Предварительное создание каналов
        for _ in range(pool_size):
            channel = connection.channel()
            self.pool.put(channel)

    @contextmanager
    def acquire(self):
        """Получение канала из пула"""
        channel = self.pool.get()
        try:
            # Проверка живости канала
            if channel.is_closed:
                channel = self.connection.channel()
            yield channel
        finally:
            self.pool.put(channel)

    def close_all(self):
        """Закрытие всех каналов"""
        while not self.pool.empty():
            channel = self.pool.get_nowait()
            if channel.is_open:
                channel.close()


# Использование
connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)

pool = ChannelPool(connection, pool_size=5)

# В разных потоках
with pool.acquire() as channel:
    channel.queue_declare(queue='test')
    channel.basic_publish(
        exchange='',
        routing_key='test',
        body='Hello'
    )

pool.close_all()
connection.close()
```

## Многопоточность

### Один канал на поток

```python
import pika
import threading
from concurrent.futures import ThreadPoolExecutor


class ThreadSafePublisher:
    """Потокобезопасный publisher с отдельным каналом на поток"""

    def __init__(self, host='localhost'):
        self.host = host
        self.local = threading.local()

    def _get_connection(self):
        """Получение connection для текущего потока"""
        if not hasattr(self.local, 'connection') or \
           self.local.connection.is_closed:
            self.local.connection = pika.BlockingConnection(
                pika.ConnectionParameters(self.host)
            )
        return self.local.connection

    def _get_channel(self):
        """Получение channel для текущего потока"""
        if not hasattr(self.local, 'channel') or \
           self.local.channel.is_closed:
            self.local.channel = self._get_connection().channel()
        return self.local.channel

    def publish(self, queue: str, message: str):
        """Публикация сообщения"""
        channel = self._get_channel()
        channel.queue_declare(queue=queue)
        channel.basic_publish(
            exchange='',
            routing_key=queue,
            body=message
        )
        print(f"[{threading.current_thread().name}] Published: {message}")


# Использование
publisher = ThreadSafePublisher()

def worker(task_id):
    publisher.publish('tasks', f'Task {task_id}')

with ThreadPoolExecutor(max_workers=4) as executor:
    for i in range(10):
        executor.submit(worker, i)
```

### Потокобезопасные callback-ы

```python
import pika
import threading

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()
channel.queue_declare(queue='tasks')

def process_in_thread(body):
    """Обработка в отдельном потоке"""
    result = f"Processed: {body.decode()}"
    print(result)
    return result

def callback(ch, method, properties, body):
    # Запуск обработки в отдельном потоке
    def task():
        result = process_in_thread(body)
        # Потокобезопасное подтверждение
        connection.add_callback_threadsafe(
            lambda: ch.basic_ack(delivery_tag=method.delivery_tag)
        )

    thread = threading.Thread(target=task)
    thread.start()

channel.basic_consume(
    queue='tasks',
    on_message_callback=callback,
    auto_ack=False
)

print('Waiting for messages...')
channel.start_consuming()
```

## Connection Pooling

### Пул соединений для высоконагруженных приложений

```python
import pika
from queue import Queue
from threading import Lock
from contextlib import contextmanager
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


class ConnectionPool:
    """Пул соединений для RabbitMQ"""

    def __init__(
        self,
        host='localhost',
        port=5672,
        username='guest',
        password='guest',
        virtual_host='/',
        pool_size=5
    ):
        self.connection_params = pika.ConnectionParameters(
            host=host,
            port=port,
            virtual_host=virtual_host,
            credentials=pika.PlainCredentials(username, password),
            heartbeat=600,
            blocked_connection_timeout=300
        )
        self.pool_size = pool_size
        self.pool = Queue(maxsize=pool_size)
        self.lock = Lock()
        self._initialize_pool()

    def _initialize_pool(self):
        """Инициализация пула соединений"""
        for i in range(self.pool_size):
            conn = self._create_connection()
            self.pool.put(conn)
            logger.info(f"Created connection {i + 1}/{self.pool_size}")

    def _create_connection(self):
        """Создание нового соединения"""
        return pika.BlockingConnection(self.connection_params)

    @contextmanager
    def acquire(self):
        """Получение соединения из пула"""
        connection = self.pool.get()
        try:
            # Проверка живости
            if connection.is_closed:
                logger.warning("Connection was closed, creating new one")
                connection = self._create_connection()
            yield connection
        finally:
            if connection.is_open:
                self.pool.put(connection)
            else:
                # Заменяем закрытое соединение новым
                new_conn = self._create_connection()
                self.pool.put(new_conn)

    def close_all(self):
        """Закрытие всех соединений"""
        while not self.pool.empty():
            conn = self.pool.get_nowait()
            if conn.is_open:
                conn.close()
        logger.info("All connections closed")


# Использование
pool = ConnectionPool(pool_size=3)

# Поток 1
with pool.acquire() as conn:
    channel = conn.channel()
    channel.queue_declare(queue='test')
    channel.basic_publish(exchange='', routing_key='test', body='msg1')

# Поток 2 (параллельно)
with pool.acquire() as conn:
    channel = conn.channel()
    channel.basic_publish(exchange='', routing_key='test', body='msg2')

pool.close_all()
```

## Обработка ошибок

### Ошибки Connection

```python
import pika
import pika.exceptions
import logging

logger = logging.getLogger(__name__)


def safe_connect(host='localhost', max_attempts=3):
    """Безопасное подключение с обработкой ошибок"""
    for attempt in range(max_attempts):
        try:
            connection = pika.BlockingConnection(
                pika.ConnectionParameters(host)
            )
            logger.info("Connected successfully")
            return connection

        except pika.exceptions.AMQPConnectionError as e:
            logger.error(f"Connection error: {e}")
            if attempt == max_attempts - 1:
                raise

        except pika.exceptions.AuthenticationError as e:
            logger.error(f"Authentication failed: {e}")
            raise

        except pika.exceptions.ProbableAccessDeniedError as e:
            logger.error(f"Access denied: {e}")
            raise

    return None
```

### Ошибки Channel

```python
import pika
import pika.exceptions


def safe_publish(channel, queue, body):
    """Безопасная публикация с обработкой ошибок канала"""
    try:
        channel.basic_publish(
            exchange='',
            routing_key=queue,
            body=body
        )
        return True

    except pika.exceptions.ChannelClosedByBroker as e:
        # Канал закрыт брокером (например, очередь не существует)
        logger.error(f"Channel closed by broker: {e}")
        return False

    except pika.exceptions.ChannelWrongStateError as e:
        # Канал в неправильном состоянии
        logger.error(f"Channel in wrong state: {e}")
        return False

    except pika.exceptions.StreamLostError as e:
        # Потеря соединения
        logger.error(f"Stream lost: {e}")
        return False
```

## Best Practices

### 1. Один Connection на приложение

```python
# Плохо: много соединений
for msg in messages:
    conn = pika.BlockingConnection(...)
    channel = conn.channel()
    channel.basic_publish(...)
    conn.close()

# Хорошо: одно соединение
conn = pika.BlockingConnection(...)
channel = conn.channel()
for msg in messages:
    channel.basic_publish(...)
conn.close()
```

### 2. Один Channel на поток

```python
# В многопоточном приложении
local = threading.local()

def get_channel():
    if not hasattr(local, 'channel'):
        local.channel = connection.channel()
    return local.channel
```

### 3. Настраивайте heartbeat

```python
connection = pika.BlockingConnection(
    pika.ConnectionParameters(
        heartbeat=60,  # Каждые 60 секунд
        blocked_connection_timeout=300  # Таймаут блокировки
    )
)
```

### 4. Обрабатывайте закрытие соединений

```python
try:
    # Работа с RabbitMQ
    pass
except pika.exceptions.AMQPConnectionError:
    # Переподключение
    pass
finally:
    if connection.is_open:
        connection.close()
```

### 5. Используйте connection pooling для высокой нагрузки

```python
pool = ConnectionPool(pool_size=10)

with pool.acquire() as conn:
    # Работа с соединением
    pass
```

### 6. Закрывайте ресурсы правильно

```python
try:
    channel.close()
finally:
    connection.close()
```

## Мониторинг

### Проверка состояния

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Проверка Connection
print(f"Connection open: {connection.is_open}")
print(f"Connection closed: {connection.is_closed}")

# Проверка Channel
print(f"Channel open: {channel.is_open}")
print(f"Channel closed: {channel.is_closed}")
print(f"Channel number: {channel.channel_number}")

connection.close()
```

### Метрики через Management API

```python
import requests

# Получение информации о соединениях
response = requests.get(
    'http://localhost:15672/api/connections',
    auth=('guest', 'guest')
)

for conn in response.json():
    print(f"Connection: {conn['name']}")
    print(f"  Channels: {conn['channels']}")
    print(f"  State: {conn['state']}")
```

## Заключение

Правильное управление Connection и Channel — ключ к производительности:

1. **Connection** — дорогой ресурс, создавайте один на приложение
2. **Channel** — лёгкий ресурс, создавайте по одному на поток
3. Используйте **пулы** для высоконагруженных приложений
4. Настраивайте **heartbeat** для обнаружения разрывов
5. Всегда **обрабатывайте ошибки** соединения
6. **Закрывайте** ресурсы при завершении работы

Следуя этим практикам, вы обеспечите стабильную работу вашего приложения с RabbitMQ.
