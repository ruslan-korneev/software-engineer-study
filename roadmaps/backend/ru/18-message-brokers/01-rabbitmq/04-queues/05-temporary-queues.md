# Временные очереди

## Введение

Временные очереди в RabbitMQ — это очереди, которые существуют ограниченное время и автоматически удаляются при определённых условиях. Они используются для сценариев, где не требуется долговременное хранение данных: RPC-паттерны, временные подписки, сессионные данные.

## Типы временных очередей

### Обзор

```
┌─────────────────────────────────────────────────────────────────┐
│                    Временные очереди                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   Exclusive     │  │  Auto-delete    │  │ TTL-based       │ │
│  │                 │  │                 │  │                 │ │
│  │ Удаляется при   │  │ Удаляется при   │  │ Удаляется по    │ │
│  │ закрытии        │  │ отключении      │  │ истечении       │ │
│  │ соединения      │  │ последнего      │  │ времени         │ │
│  │                 │  │ consumer        │  │                 │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Exclusive очереди

### Создание exclusive очереди

```python
import pika

def create_exclusive_queue():
    """
    Exclusive очередь:
    - Доступна только создавшему соединению
    - Автоматически удаляется при закрытии соединения
    - Идеальна для приватных callback очередей
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Exclusive очередь с автогенерируемым именем
    result = channel.queue_declare(
        queue='',  # Пустое имя = автогенерация
        exclusive=True
    )

    queue_name = result.method.queue
    print(f"Создана exclusive очередь: {queue_name}")
    # Пример: amq.gen-XYZ123abc

    # Очередь доступна только через этот channel
    channel.basic_publish(
        exchange='',
        routing_key=queue_name,
        body='Private message'
    )

    connection.close()
    # Очередь автоматически удалена!

create_exclusive_queue()
```

### Exclusive с явным именем

```python
import pika

def named_exclusive_queue():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Exclusive очередь с явным именем
    channel.queue_declare(
        queue='session-user-12345',
        exclusive=True
    )

    # Попытка доступа из другого соединения вызовет ошибку
    # RESOURCE_LOCKED - cannot obtain exclusive access

    connection.close()
```

### Ограничения exclusive очередей

```python
"""
Ограничения:
1. Нельзя использовать с Quorum Queues
2. Нельзя сделать durable (не имеет смысла)
3. Недоступна другим соединениям
4. Не работает в кластере (привязана к узлу)
"""

import pika

def exclusive_with_quorum_error():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    try:
        # Это вызовет ошибку!
        channel.queue_declare(
            queue='exclusive_quorum',
            exclusive=True,
            arguments={'x-queue-type': 'quorum'}
        )
    except Exception as e:
        print(f"Ошибка: {e}")
        # Quorum queues не поддерживают exclusive

    connection.close()
```

## Auto-delete очереди

### Создание auto-delete очереди

```python
import pika

def create_auto_delete_queue():
    """
    Auto-delete очередь:
    - Удаляется когда отключается последний consumer
    - Может использоваться несколькими соединениями
    - Остаётся жить пока есть хотя бы один consumer
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Auto-delete очередь
    channel.queue_declare(
        queue='notifications_temp',
        auto_delete=True
    )

    print("Auto-delete очередь создана")

    # Подключаем consumer
    def callback(ch, method, properties, body):
        print(f"Получено: {body.decode()}")
        ch.basic_ack(delivery_tag=method.delivery_tag)

    channel.basic_consume(
        queue='notifications_temp',
        on_message_callback=callback
    )

    # Пока consumer подключен — очередь существует
    # При отключении всех consumers — очередь удалится

    connection.close()

create_auto_delete_queue()
```

### Auto-delete vs Exclusive

```python
import pika
import threading
import time

def demonstrate_auto_delete_behavior():
    """
    Показываем, что auto-delete удаляется только
    при отключении ПОСЛЕДНЕГО consumer.
    """
    # Создаём очередь
    conn1 = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    ch1 = conn1.channel()
    ch1.queue_declare(queue='shared_temp', auto_delete=True)

    # Consumer 1
    def consumer1_callback(ch, method, props, body):
        print(f"Consumer 1: {body.decode()}")
        ch.basic_ack(delivery_tag=method.delivery_tag)

    ch1.basic_consume(queue='shared_temp', on_message_callback=consumer1_callback)

    # Consumer 2 из другого соединения
    conn2 = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    ch2 = conn2.channel()

    def consumer2_callback(ch, method, props, body):
        print(f"Consumer 2: {body.decode()}")
        ch.basic_ack(delivery_tag=method.delivery_tag)

    ch2.basic_consume(queue='shared_temp', on_message_callback=consumer2_callback)

    # Закрываем первое соединение
    conn1.close()
    print("Consumer 1 отключен — очередь ещё существует")

    # Проверяем
    result = ch2.queue_declare(queue='shared_temp', passive=True)
    print(f"Очередь существует: {result.method.queue}")

    # Закрываем второе соединение
    conn2.close()
    print("Consumer 2 отключен — очередь удалена")
```

## Reply Queues для RPC

### Классический RPC паттерн

```python
import pika
import uuid

class RpcClient:
    """
    RPC клиент использует временную exclusive очередь
    для получения ответов.
    """

    def __init__(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()

        # Создаём temporary reply queue
        result = self.channel.queue_declare(
            queue='',
            exclusive=True  # Временная, приватная
        )
        self.callback_queue = result.method.queue

        # Словарь для хранения ответов
        self.responses = {}

        # Начинаем слушать ответы
        self.channel.basic_consume(
            queue=self.callback_queue,
            on_message_callback=self.on_response,
            auto_ack=True
        )

    def on_response(self, ch, method, props, body):
        """Обработчик входящих ответов."""
        self.responses[props.correlation_id] = body

    def call(self, message):
        """Отправляет RPC запрос и ждёт ответ."""
        corr_id = str(uuid.uuid4())
        self.responses[corr_id] = None

        # Отправляем запрос
        self.channel.basic_publish(
            exchange='',
            routing_key='rpc_queue',
            properties=pika.BasicProperties(
                reply_to=self.callback_queue,
                correlation_id=corr_id,
            ),
            body=message
        )

        # Ждём ответ
        while self.responses[corr_id] is None:
            self.connection.process_data_events()

        return self.responses.pop(corr_id)

    def close(self):
        self.connection.close()
        # callback_queue автоматически удалится


# Использование
client = RpcClient()
response = client.call("Compute something")
print(f"Ответ: {response}")
client.close()
```

### RPC сервер

```python
import pika

def rpc_server():
    """
    RPC сервер обрабатывает запросы и отправляет
    ответы в reply_to очередь клиента.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Очередь для входящих запросов
    channel.queue_declare(queue='rpc_queue')

    def process_request(data):
        """Обработка запроса."""
        return f"Processed: {data}"

    def on_request(ch, method, props, body):
        request = body.decode()
        print(f"Получен запрос: {request}")

        # Обрабатываем
        response = process_request(request)

        # Отправляем ответ в reply_to очередь
        ch.basic_publish(
            exchange='',
            routing_key=props.reply_to,  # Временная очередь клиента
            properties=pika.BasicProperties(
                correlation_id=props.correlation_id
            ),
            body=response
        )

        ch.basic_ack(delivery_tag=method.delivery_tag)

    channel.basic_qos(prefetch_count=1)
    channel.basic_consume(
        queue='rpc_queue',
        on_message_callback=on_request
    )

    print("RPC сервер запущен...")
    channel.start_consuming()

rpc_server()
```

### Direct Reply-to (оптимизация)

```python
import pika
import uuid

class OptimizedRpcClient:
    """
    Использует Direct Reply-to для избежания
    создания временной очереди.

    RabbitMQ предоставляет специальную псевдо-очередь
    'amq.rabbitmq.reply-to' для быстрых RPC.
    """

    def __init__(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()

        self.response = None
        self.corr_id = None

        # Используем Direct Reply-to
        self.channel.basic_consume(
            queue='amq.rabbitmq.reply-to',
            on_message_callback=self.on_response,
            auto_ack=True
        )

    def on_response(self, ch, method, props, body):
        if props.correlation_id == self.corr_id:
            self.response = body

    def call(self, message):
        self.response = None
        self.corr_id = str(uuid.uuid4())

        self.channel.basic_publish(
            exchange='',
            routing_key='rpc_queue',
            properties=pika.BasicProperties(
                reply_to='amq.rabbitmq.reply-to',  # Специальная очередь
                correlation_id=self.corr_id,
            ),
            body=message
        )

        while self.response is None:
            self.connection.process_data_events()

        return self.response

    def close(self):
        self.connection.close()
```

## TTL-based временные очереди

### Очередь с временем жизни

```python
import pika

def create_expiring_queue():
    """
    Очередь с x-expires удаляется если не используется
    в течение указанного времени.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Очередь удалится через 5 минут неиспользования
    channel.queue_declare(
        queue='session_queue',
        arguments={
            'x-expires': 300000  # 5 минут в миллисекундах
        }
    )

    # "Неиспользование" означает:
    # - Нет consumers
    # - Нет вызовов basic.get
    # - Нет переобъявлений

    connection.close()

create_expiring_queue()
```

### Очередь с TTL сообщений

```python
import pika

def create_queue_with_message_ttl():
    """
    Сообщения автоматически удаляются после истечения TTL.
    Очередь при этом остаётся.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Сообщения живут максимум 60 секунд
    channel.queue_declare(
        queue='short_lived_messages',
        arguments={
            'x-message-ttl': 60000
        }
    )

    # Отправляем сообщение
    channel.basic_publish(
        exchange='',
        routing_key='short_lived_messages',
        body='This will expire in 60 seconds'
    )

    connection.close()
```

## Комбинированные паттерны

### Exclusive + Auto-delete

```python
import pika

def fully_temporary_queue():
    """
    Максимально временная очередь:
    - Удаляется при закрытии соединения (exclusive)
    - Удаляется при отключении consumers (auto_delete)
    - Первое событие вызывает удаление
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    result = channel.queue_declare(
        queue='',
        exclusive=True,
        auto_delete=True
    )

    print(f"Полностью временная очередь: {result.method.queue}")

    connection.close()
```

### Временная очередь с DLX

```python
import pika

def temporary_with_dlx():
    """
    Временная очередь, которая отправляет
    необработанные сообщения в DLQ перед удалением.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # DLX для сохранения потерянных сообщений
    channel.exchange_declare(exchange='dlx', exchange_type='direct')
    channel.queue_declare(queue='lost_messages', durable=True)
    channel.queue_bind(queue='lost_messages', exchange='dlx', routing_key='lost')

    # Временная очередь с DLX
    result = channel.queue_declare(
        queue='',
        exclusive=True,
        arguments={
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'lost',
            'x-message-ttl': 30000  # 30 секунд
        }
    )

    print(f"Временная очередь с DLX: {result.method.queue}")
    connection.close()
```

## Pub/Sub с временными очередями

### Подписка на события

```python
import pika

def event_subscriber(subscriber_id):
    """
    Каждый подписчик создаёт временную очередь
    для получения событий.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Exchange для событий
    channel.exchange_declare(
        exchange='events',
        exchange_type='fanout'
    )

    # Временная очередь для этого подписчика
    result = channel.queue_declare(
        queue='',
        exclusive=True  # Удалится при отключении
    )
    queue_name = result.method.queue

    # Привязываем к exchange
    channel.queue_bind(
        exchange='events',
        queue=queue_name
    )

    print(f"Subscriber {subscriber_id} подключен к {queue_name}")

    def callback(ch, method, properties, body):
        print(f"[{subscriber_id}] Событие: {body.decode()}")

    channel.basic_consume(
        queue=queue_name,
        on_message_callback=callback,
        auto_ack=True
    )

    channel.start_consuming()
    # При остановке — очередь удалится


def event_publisher():
    """
    Публикует события во все временные очереди подписчиков.
    """
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    channel.exchange_declare(
        exchange='events',
        exchange_type='fanout'
    )

    # Публикуем событие
    channel.basic_publish(
        exchange='events',
        routing_key='',  # Fanout игнорирует routing_key
        body='New event occurred!'
    )

    connection.close()
```

## Best Practices

### 1. Выбор правильного типа

```python
"""
┌──────────────────────┬─────────────────────────────────────────┐
│ Сценарий             │ Рекомендуемый тип                       │
├──────────────────────┼─────────────────────────────────────────┤
│ RPC callback         │ exclusive=True                          │
│ Pub/Sub подписка     │ exclusive=True, auto_delete=True        │
│ Сессионные данные    │ x-expires=N                             │
│ Shared temp queue    │ auto_delete=True                        │
│ Короткоживущие msg   │ x-message-ttl=N                         │
└──────────────────────┴─────────────────────────────────────────┘
"""
```

### 2. Обработка edge cases

```python
import pika

def handle_temporary_queue_edge_cases():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # 1. Auto-delete без consumers сразу не удалится
    #    Нужен хотя бы один consumer, потом отключение
    channel.queue_declare(queue='auto_del', auto_delete=True)
    # Очередь существует даже без consumers!

    # 2. Exclusive блокирует доступ другим соединениям
    result = channel.queue_declare(queue='', exclusive=True)
    # Другое соединение получит RESOURCE_LOCKED

    # 3. x-expires сбрасывается при активности
    channel.queue_declare(
        queue='expiring',
        arguments={'x-expires': 60000}
    )
    # Каждый basic.get сбрасывает таймер

    connection.close()
```

### 3. Мониторинг временных очередей

```python
import requests

def monitor_temporary_queues():
    """
    Мониторинг временных очередей через Management API.
    """
    response = requests.get(
        'http://localhost:15672/api/queues',
        auth=('guest', 'guest')
    )

    queues = response.json()

    print("Временные очереди:")
    for q in queues:
        is_temp = (
            q.get('exclusive', False) or
            q.get('auto_delete', False) or
            q['name'].startswith('amq.gen-')
        )

        if is_temp:
            print(f"  {q['name']}")
            print(f"    Exclusive: {q.get('exclusive', False)}")
            print(f"    Auto-delete: {q.get('auto_delete', False)}")
            print(f"    Messages: {q['messages']}")
            print(f"    Consumers: {q['consumers']}")
```

## Заключение

Временные очереди — мощный инструмент для сценариев, где не требуется долговременное хранение:

- **Exclusive** — для приватных, привязанных к соединению очередей
- **Auto-delete** — для очередей, которые должны жить пока есть consumers
- **x-expires** — для очередей с фиксированным временем жизни
- **Reply queues** — классический паттерн для RPC

Правильный выбор типа временной очереди помогает автоматически управлять ресурсами и избегать утечек.
