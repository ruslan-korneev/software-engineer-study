# Producers (Издатели)

## Что такое Producer?

**Producer (Издатель, Производитель)** — это приложение или компонент, который создаёт и отправляет сообщения в RabbitMQ. Producer не отправляет сообщения напрямую в очередь — он публикует их в **Exchange**, который затем маршрутизирует сообщения в соответствующие очереди.

## Роль Producer в архитектуре RabbitMQ

```
┌─────────────┐     ┌──────────┐     ┌─────────┐     ┌───────────┐
│  Producer   │────▶│ Exchange │────▶│  Queue  │────▶│  Consumer │
└─────────────┘     └──────────┘     └─────────┘     └───────────┘
```

Producer отвечает за:
- Установку соединения с RabbitMQ
- Создание канала для публикации
- Формирование сообщений с необходимыми свойствами
- Публикацию сообщений в Exchange
- Обработку подтверждений (confirmations)

## Базовый пример Producer

```python
import pika

# Установка соединения
connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Объявление очереди (idempotent операция)
channel.queue_declare(queue='hello')

# Публикация сообщения
channel.basic_publish(
    exchange='',           # Default exchange
    routing_key='hello',   # Имя очереди для default exchange
    body='Hello World!'
)

print(" [x] Sent 'Hello World!'")

# Закрытие соединения
connection.close()
```

## Публикация с различными Exchange типами

### Direct Exchange

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Объявление direct exchange
channel.exchange_declare(
    exchange='direct_logs',
    exchange_type='direct'
)

# Публикация с routing key
severity = 'error'
message = 'Critical error occurred!'

channel.basic_publish(
    exchange='direct_logs',
    routing_key=severity,
    body=message
)

print(f" [x] Sent {severity}:{message}")
connection.close()
```

### Fanout Exchange

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Объявление fanout exchange
channel.exchange_declare(
    exchange='notifications',
    exchange_type='fanout'
)

# Публикация (routing_key игнорируется)
channel.basic_publish(
    exchange='notifications',
    routing_key='',  # Игнорируется для fanout
    body='Broadcast message to all!'
)

connection.close()
```

### Topic Exchange

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

channel.exchange_declare(
    exchange='topic_logs',
    exchange_type='topic'
)

# Публикация с иерархическим routing key
routing_key = 'user.order.created'
message = '{"order_id": 12345, "user_id": 1}'

channel.basic_publish(
    exchange='topic_logs',
    routing_key=routing_key,
    body=message
)

print(f" [x] Sent {routing_key}:{message}")
connection.close()
```

## Publisher Confirms (Подтверждения публикации)

Publisher Confirms гарантируют, что сообщение было принято брокером.

### Синхронные подтверждения

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Включение режима подтверждений
channel.confirm_delivery()

channel.queue_declare(queue='confirmed_queue', durable=True)

try:
    channel.basic_publish(
        exchange='',
        routing_key='confirmed_queue',
        body='Important message',
        properties=pika.BasicProperties(
            delivery_mode=2,  # Persistent message
        ),
        mandatory=True  # Вернёт ошибку, если нет очереди
    )
    print(" [x] Message was confirmed")
except pika.exceptions.UnroutableError:
    print(" [!] Message could not be routed")

connection.close()
```

### Асинхронные подтверждения (Batch)

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()
channel.confirm_delivery()

channel.queue_declare(queue='batch_queue', durable=True)

messages = ['msg1', 'msg2', 'msg3', 'msg4', 'msg5']

for msg in messages:
    channel.basic_publish(
        exchange='',
        routing_key='batch_queue',
        body=msg,
        properties=pika.BasicProperties(delivery_mode=2)
    )

print(f" [x] Published {len(messages)} messages with confirms")
connection.close()
```

## Установка свойств сообщений

```python
import pika
import json
import uuid
from datetime import datetime

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

channel.queue_declare(queue='tasks', durable=True)

# Формирование сообщения с полными свойствами
message = json.dumps({
    'task': 'send_email',
    'recipient': 'user@example.com',
    'subject': 'Hello!'
})

properties = pika.BasicProperties(
    delivery_mode=2,              # Persistent (сохранение на диск)
    content_type='application/json',
    content_encoding='utf-8',
    priority=5,                   # Приоритет (0-9)
    correlation_id=str(uuid.uuid4()),  # ID для корреляции
    reply_to='response_queue',    # Очередь для ответа (RPC)
    expiration='60000',           # TTL в миллисекундах
    message_id=str(uuid.uuid4()), # Уникальный ID сообщения
    timestamp=int(datetime.now().timestamp()),
    type='email_task',            # Тип сообщения
    user_id='guest',              # ID пользователя
    app_id='email_service',       # ID приложения
    headers={                     # Дополнительные заголовки
        'x-retry-count': 0,
        'x-source': 'api',
        'x-trace-id': str(uuid.uuid4())
    }
)

channel.basic_publish(
    exchange='',
    routing_key='tasks',
    body=message,
    properties=properties
)

print(" [x] Sent task with full properties")
connection.close()
```

## Реализация надёжного Producer

```python
import pika
import json
import logging
from typing import Optional, Dict, Any
from functools import wraps
import time

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


class RobustProducer:
    """Надёжный producer с переподключением и retry логикой"""

    def __init__(
        self,
        host: str = 'localhost',
        port: int = 5672,
        username: str = 'guest',
        password: str = 'guest',
        virtual_host: str = '/',
        max_retries: int = 3,
        retry_delay: float = 1.0
    ):
        self.connection_params = pika.ConnectionParameters(
            host=host,
            port=port,
            virtual_host=virtual_host,
            credentials=pika.PlainCredentials(username, password),
            heartbeat=600,
            blocked_connection_timeout=300,
            connection_attempts=3,
            retry_delay=5
        )
        self.max_retries = max_retries
        self.retry_delay = retry_delay
        self.connection: Optional[pika.BlockingConnection] = None
        self.channel: Optional[pika.channel.Channel] = None

    def connect(self) -> None:
        """Установка соединения с RabbitMQ"""
        if self.connection and self.connection.is_open:
            return

        self.connection = pika.BlockingConnection(self.connection_params)
        self.channel = self.connection.channel()
        self.channel.confirm_delivery()
        logger.info("Connected to RabbitMQ")

    def close(self) -> None:
        """Закрытие соединения"""
        if self.connection and self.connection.is_open:
            self.connection.close()
            logger.info("Connection closed")

    def _ensure_connection(self) -> None:
        """Проверка и восстановление соединения"""
        if not self.connection or self.connection.is_closed:
            self.connect()
        if not self.channel or self.channel.is_closed:
            self.channel = self.connection.channel()
            self.channel.confirm_delivery()

    def publish(
        self,
        exchange: str,
        routing_key: str,
        message: Dict[str, Any],
        properties: Optional[pika.BasicProperties] = None,
        mandatory: bool = True
    ) -> bool:
        """
        Публикация сообщения с retry логикой

        Args:
            exchange: Имя exchange
            routing_key: Routing key
            message: Словарь с данными сообщения
            properties: Свойства сообщения
            mandatory: Требовать наличие очереди

        Returns:
            True если сообщение успешно опубликовано
        """
        if properties is None:
            properties = pika.BasicProperties(
                delivery_mode=2,
                content_type='application/json'
            )

        body = json.dumps(message)

        for attempt in range(self.max_retries):
            try:
                self._ensure_connection()

                self.channel.basic_publish(
                    exchange=exchange,
                    routing_key=routing_key,
                    body=body,
                    properties=properties,
                    mandatory=mandatory
                )

                logger.info(
                    f"Published message to {exchange}/{routing_key}"
                )
                return True

            except pika.exceptions.UnroutableError:
                logger.error(
                    f"Message unroutable: {exchange}/{routing_key}"
                )
                return False

            except pika.exceptions.AMQPConnectionError as e:
                logger.warning(
                    f"Connection error (attempt {attempt + 1}): {e}"
                )
                self.connection = None
                self.channel = None

                if attempt < self.max_retries - 1:
                    time.sleep(self.retry_delay * (attempt + 1))
                else:
                    raise

        return False

    def __enter__(self):
        self.connect()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.close()


# Использование
if __name__ == '__main__':
    with RobustProducer(host='localhost') as producer:
        # Публикация одного сообщения
        producer.publish(
            exchange='',
            routing_key='tasks',
            message={'task': 'process_order', 'order_id': 123}
        )

        # Публикация пакета сообщений
        for i in range(10):
            producer.publish(
                exchange='',
                routing_key='tasks',
                message={'task': 'batch_item', 'index': i}
            )
```

## Best Practices для Producers

### 1. Используйте Publisher Confirms

```python
# Всегда включайте confirms для важных сообщений
channel.confirm_delivery()
```

### 2. Переиспользуйте соединения и каналы

```python
# Плохо: новое соединение для каждого сообщения
for msg in messages:
    conn = pika.BlockingConnection(...)
    channel = conn.channel()
    channel.basic_publish(...)
    conn.close()

# Хорошо: одно соединение для всех сообщений
conn = pika.BlockingConnection(...)
channel = conn.channel()
for msg in messages:
    channel.basic_publish(...)
conn.close()
```

### 3. Устанавливайте delivery_mode=2 для персистентных сообщений

```python
properties = pika.BasicProperties(
    delivery_mode=2  # Сообщение сохраняется на диск
)
```

### 4. Используйте mandatory=True для критичных сообщений

```python
channel.basic_publish(
    exchange='orders',
    routing_key='new',
    body=message,
    mandatory=True  # Гарантирует маршрутизацию
)
```

### 5. Добавляйте идентификаторы для трассировки

```python
properties = pika.BasicProperties(
    message_id=str(uuid.uuid4()),
    correlation_id=str(uuid.uuid4()),
    headers={'x-trace-id': trace_id}
)
```

### 6. Обрабатывайте ошибки подключения

```python
import pika.exceptions

try:
    channel.basic_publish(...)
except pika.exceptions.AMQPConnectionError:
    # Переподключение
    pass
except pika.exceptions.AMQPChannelError:
    # Пересоздание канала
    pass
```

### 7. Устанавливайте TTL для временных сообщений

```python
properties = pika.BasicProperties(
    expiration='300000'  # 5 минут в миллисекундах
)
```

## Типичные ошибки

1. **Не закрывать соединения** — приводит к утечке ресурсов
2. **Создавать соединение на каждое сообщение** — огромные накладные расходы
3. **Игнорировать Publisher Confirms** — потеря сообщений без уведомления
4. **Не использовать persistent delivery** — потеря сообщений при перезапуске
5. **Не обрабатывать ошибки соединения** — падение приложения при проблемах с сетью

## Заключение

Producer — ключевой компонент системы обмена сообщениями. Правильная реализация включает:
- Надёжное управление соединениями
- Использование Publisher Confirms
- Корректную обработку ошибок
- Установку необходимых свойств сообщений
- Выбор подходящего Exchange типа

Следуя этим практикам, вы обеспечите надёжную и эффективную публикацию сообщений в RabbitMQ.
