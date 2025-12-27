# Default Exchange

## Введение

Default Exchange (обменник по умолчанию) - это особый предопределённый exchange в RabbitMQ, который позволяет отправлять сообщения **напрямую в очередь по её имени**. Это самый простой способ работы с RabbitMQ, не требующий явного создания exchange и привязок.

## Ключевые особенности

```
┌──────────┐                              ┌─────────────────┐
│ Producer │──routing_key="my_queue"────▶│ Default Exchange│
└──────────┘                              │     (AMQP "")   │
                                          └────────┬────────┘
                                                   │
                                                   │ routing_key = queue_name
                                                   ▼
                                          ┌─────────────────┐
                                          │    my_queue     │
                                          └─────────────────┘
```

| Характеристика | Значение |
|----------------|----------|
| Имя | `""` (пустая строка) |
| Тип | Direct |
| Durable | Да |
| Предопределённый | Да (создаётся автоматически) |
| Удаляемый | Нет |

## Принцип работы

Default Exchange автоматически привязывает **каждую созданную очередь** к себе, используя **имя очереди как routing key**:

```
┌─────────────────────────────────────────────────────────────┐
│                    Default Exchange ""                      │
│                                                             │
│  Автоматические привязки (implicit bindings):              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ routing_key="orders"    ──▶  Queue "orders"         │   │
│  │ routing_key="emails"    ──▶  Queue "emails"         │   │
│  │ routing_key="tasks"     ──▶  Queue "tasks"          │   │
│  │ routing_key="logs"      ──▶  Queue "logs"           │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

При создании очереди "my_queue" автоматически создаётся привязка:
Default Exchange ──routing_key="my_queue"──▶ Queue "my_queue"
```

## Базовый пример

### Producer

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Создаём очередь
channel.queue_declare(queue='hello', durable=True)

# Отправляем напрямую в очередь через default exchange
channel.basic_publish(
    exchange='',              # Пустая строка = default exchange
    routing_key='hello',      # Имя очереди!
    body='Hello World!',
    properties=pika.BasicProperties(
        delivery_mode=2  # Persistent
    )
)

print(" [x] Отправлено 'Hello World!'")
connection.close()
```

### Consumer

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Объявляем очередь (идемпотентно)
channel.queue_declare(queue='hello', durable=True)

def callback(ch, method, properties, body):
    print(f" [x] Получено: {body.decode()}")
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(
    queue='hello',
    on_message_callback=callback
)

print(' [*] Ожидание сообщений. Для выхода нажмите CTRL+C')
channel.start_consuming()
```

## Сравнение с явным Direct Exchange

```python
# ========== Default Exchange (простой способ) ==========
# Не нужно создавать exchange и binding

channel.queue_declare(queue='tasks')

channel.basic_publish(
    exchange='',           # Default exchange
    routing_key='tasks',   # = имя очереди
    body='task data'
)

# ========== Явный Direct Exchange (гибкий способ) ==========
# Нужно создать exchange и binding

channel.exchange_declare(exchange='my_exchange', exchange_type='direct')
channel.queue_declare(queue='tasks')
channel.queue_bind(exchange='my_exchange', queue='tasks', routing_key='task_key')

channel.basic_publish(
    exchange='my_exchange',
    routing_key='task_key',  # Может отличаться от имени очереди
    body='task data'
)
```

| Аспект | Default Exchange | Явный Exchange |
|--------|------------------|----------------|
| Настройка | Минимальная | Требует конфигурации |
| Гибкость | Низкая | Высокая |
| routing_key | = имя очереди | Любой |
| Несколько очередей на ключ | Нет | Да |
| Подходит для | Простых случаев | Production систем |

## Use Case 1: Простая очередь задач

```python
import pika
import json
import uuid

class SimpleTaskQueue:
    """Простая очередь задач через default exchange"""

    def __init__(self, queue_name: str, host='localhost'):
        self.queue_name = queue_name

        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()

        # Создаём durable очередь
        self.channel.queue_declare(queue=queue_name, durable=True)

    def enqueue(self, task_data: dict) -> str:
        """Добавить задачу в очередь"""
        task_id = str(uuid.uuid4())

        task = {
            'id': task_id,
            'data': task_data
        }

        self.channel.basic_publish(
            exchange='',                # Default exchange
            routing_key=self.queue_name, # = имя очереди
            body=json.dumps(task),
            properties=pika.BasicProperties(
                delivery_mode=2,  # Persistent
                content_type='application/json'
            )
        )

        return task_id

    def close(self):
        self.connection.close()


class SimpleTaskWorker:
    """Воркер для обработки задач"""

    def __init__(self, queue_name: str, handler, host='localhost'):
        self.queue_name = queue_name
        self.handler = handler

        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()

        self.channel.queue_declare(queue=queue_name, durable=True)
        self.channel.basic_qos(prefetch_count=1)

    def start(self):
        def callback(ch, method, properties, body):
            task = json.loads(body)
            print(f"Обработка задачи {task['id']}")

            try:
                self.handler(task['data'])
                ch.basic_ack(delivery_tag=method.delivery_tag)
                print(f"Задача {task['id']} выполнена")
            except Exception as e:
                print(f"Ошибка: {e}")
                ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)

        self.channel.basic_consume(
            queue=self.queue_name,
            on_message_callback=callback
        )

        print(f"Воркер запущен для очереди '{self.queue_name}'")
        self.channel.start_consuming()


# Использование
queue = SimpleTaskQueue('email_tasks')
task_id = queue.enqueue({'to': 'user@example.com', 'subject': 'Hello'})
print(f"Задача добавлена: {task_id}")
queue.close()

# Воркер
# def send_email(data):
#     print(f"Отправка email на {data['to']}")
# worker = SimpleTaskWorker('email_tasks', send_email)
# worker.start()
```

## Use Case 2: Request-Reply паттерн (RPC)

Default exchange идеально подходит для RPC, так как позволяет отвечать напрямую в очередь клиента:

```python
import pika
import json
import uuid

class RPCClient:
    """RPC клиент с использованием default exchange"""

    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()

        # Создаём эксклюзивную очередь для ответов
        result = self.channel.queue_declare(queue='', exclusive=True)
        self.callback_queue = result.method.queue

        self.pending_requests = {}

        self.channel.basic_consume(
            queue=self.callback_queue,
            on_message_callback=self._on_response,
            auto_ack=True
        )

    def _on_response(self, ch, method, properties, body):
        correlation_id = properties.correlation_id
        if correlation_id in self.pending_requests:
            self.pending_requests[correlation_id] = json.loads(body)

    def call(self, queue_name: str, request: dict, timeout: int = 30) -> dict:
        """Синхронный RPC вызов"""
        correlation_id = str(uuid.uuid4())
        self.pending_requests[correlation_id] = None

        # Отправляем запрос через default exchange
        self.channel.basic_publish(
            exchange='',              # Default exchange
            routing_key=queue_name,   # Очередь сервера
            body=json.dumps(request),
            properties=pika.BasicProperties(
                reply_to=self.callback_queue,   # Куда отвечать
                correlation_id=correlation_id,
                content_type='application/json'
            )
        )

        # Ждём ответ
        self.connection.process_data_events(time_limit=timeout)

        response = self.pending_requests.pop(correlation_id)
        if response is None:
            raise TimeoutError("RPC call timed out")

        return response


class RPCServer:
    """RPC сервер с использованием default exchange"""

    def __init__(self, queue_name: str, handler, host='localhost'):
        self.queue_name = queue_name
        self.handler = handler

        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()

        self.channel.queue_declare(queue=queue_name, durable=True)
        self.channel.basic_qos(prefetch_count=1)

    def start(self):
        def on_request(ch, method, properties, body):
            request = json.loads(body)
            print(f"Получен запрос: {request}")

            try:
                result = self.handler(request)
                response = {'success': True, 'result': result}
            except Exception as e:
                response = {'success': False, 'error': str(e)}

            # Отвечаем через default exchange в очередь клиента
            ch.basic_publish(
                exchange='',                  # Default exchange
                routing_key=properties.reply_to,  # Очередь клиента
                body=json.dumps(response),
                properties=pika.BasicProperties(
                    correlation_id=properties.correlation_id,
                    content_type='application/json'
                )
            )

            ch.basic_ack(delivery_tag=method.delivery_tag)

        self.channel.basic_consume(
            queue=self.queue_name,
            on_message_callback=on_request
        )

        print(f"RPC сервер запущен на '{self.queue_name}'")
        self.channel.start_consuming()


# Использование
# Сервер
def calculate(request):
    a, b = request['a'], request['b']
    op = request['operation']
    if op == 'add':
        return a + b
    elif op == 'multiply':
        return a * b

# server = RPCServer('calc_queue', calculate)
# server.start()

# Клиент
# client = RPCClient()
# result = client.call('calc_queue', {'a': 5, 'b': 3, 'operation': 'add'})
# print(f"Результат: {result}")  # {'success': True, 'result': 8}
```

## Use Case 3: Несколько очередей

```python
import pika
import json

class MultiQueuePublisher:
    """Публикация в несколько очередей через default exchange"""

    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()

        # Создаём несколько очередей
        self.queues = ['high_priority', 'normal_priority', 'low_priority']
        for queue in self.queues:
            self.channel.queue_declare(queue=queue, durable=True)

    def publish(self, message: str, priority: str = 'normal'):
        """Публикация в очередь по приоритету"""
        queue_map = {
            'high': 'high_priority',
            'normal': 'normal_priority',
            'low': 'low_priority'
        }

        queue_name = queue_map.get(priority, 'normal_priority')

        self.channel.basic_publish(
            exchange='',            # Default exchange
            routing_key=queue_name, # Прямо в нужную очередь
            body=message,
            properties=pika.BasicProperties(delivery_mode=2)
        )

        print(f"Сообщение отправлено в {queue_name}")

    def close(self):
        self.connection.close()


# Использование
publisher = MultiQueuePublisher()
publisher.publish("Критическая задача!", priority='high')
publisher.publish("Обычная задача", priority='normal')
publisher.publish("Фоновая задача", priority='low')
publisher.close()
```

## Ограничения Default Exchange

### 1. Нельзя удалить или переконфигурировать

```python
# Это НЕ сработает!
channel.exchange_delete(exchange='')  # Ошибка!

# Нельзя изменить тип
channel.exchange_declare(exchange='', exchange_type='fanout')  # Ошибка!
```

### 2. Нельзя отправить в несколько очередей одним сообщением

```python
# С default exchange нельзя:
# - Отправить одно сообщение в несколько очередей
# - Использовать broadcast

# Для этого нужен Fanout или Topic exchange
```

### 3. routing_key должен точно совпадать с именем очереди

```python
# Это НЕ сработает - очереди 'my-queue' не существует
channel.basic_publish(
    exchange='',
    routing_key='my-queue',  # Если очередь называется 'my_queue'
    body='test'
)
# Сообщение будет потеряно!
```

## Когда использовать Default Exchange

| Сценарий | Рекомендация |
|----------|--------------|
| Простой producer-consumer | Да |
| Прототипирование | Да |
| RPC паттерн (reply_to) | Да |
| Обучение RabbitMQ | Да |
| Production с простой логикой | Возможно |
| Сложная маршрутизация | Нет |
| Broadcast | Нет |
| Микросервисы | Нет |

## Сравнение с другими Exchange

```
┌──────────────────────────────────────────────────────────────┐
│                  Сравнение Exchange                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Default ("")                                                │
│  ├── Тип: Direct (предопределённый)                         │
│  ├── routing_key = queue_name                                │
│  └── Простой, но ограниченный                               │
│                                                              │
│  Direct (named)                                              │
│  ├── Тип: Direct (настраиваемый)                            │
│  ├── routing_key = любой                                     │
│  └── Гибкий, точная маршрутизация                           │
│                                                              │
│  Fanout                                                      │
│  ├── routing_key = игнорируется                             │
│  └── Broadcast всем                                          │
│                                                              │
│  Topic                                                       │
│  ├── routing_key = паттерн с wildcards                       │
│  └── Гибкая категоризация                                   │
│                                                              │
│  Headers                                                     │
│  ├── routing_key = игнорируется                             │
│  └── Маршрутизация по headers                               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## Best Practices

### 1. Явно объявляйте очереди

```python
# Хорошо - явное объявление
channel.queue_declare(queue='my_queue', durable=True)
channel.basic_publish(exchange='', routing_key='my_queue', body='test')

# Плохо - надежда что очередь уже существует
channel.basic_publish(exchange='', routing_key='my_queue', body='test')
# Если очереди нет - сообщение потеряно!
```

### 2. Используйте durable для production

```python
# Очередь переживёт перезапуск RabbitMQ
channel.queue_declare(queue='important_queue', durable=True)

# Сообщение переживёт перезапуск
channel.basic_publish(
    exchange='',
    routing_key='important_queue',
    body='important data',
    properties=pika.BasicProperties(delivery_mode=2)  # Persistent
)
```

### 3. Документируйте имена очередей

```python
class QueueNames:
    """Централизованные имена очередей"""
    EMAIL_TASKS = 'email_tasks'
    SMS_TASKS = 'sms_tasks'
    NOTIFICATIONS = 'notifications'
    ANALYTICS = 'analytics'

# Использование
channel.basic_publish(
    exchange='',
    routing_key=QueueNames.EMAIL_TASKS,
    body=message
)
```

## Переход с Default на Named Exchange

Когда проект растёт, может понадобиться перейти на именованный exchange:

```python
# До: Default exchange
channel.basic_publish(
    exchange='',
    routing_key='orders',
    body=order_data
)

# После: Named exchange с большей гибкостью
channel.exchange_declare(exchange='orders_exchange', exchange_type='topic')
channel.queue_bind(exchange='orders_exchange', queue='orders', routing_key='order.*')

channel.basic_publish(
    exchange='orders_exchange',
    routing_key='order.created',  # Более информативный routing key
    body=order_data
)
```

## Заключение

Default Exchange - это:
- **Самый простой способ** работы с RabbitMQ
- **Идеален для обучения** и прототипирования
- **Хорош для RPC** (reply_to через default exchange)
- **Ограничен** для сложных сценариев

Начинайте с Default Exchange для простых случаев, переходите на именованные exchange когда нужна:
- Маршрутизация одного сообщения в несколько очередей
- Broadcast (fanout)
- Паттерн-матчинг (topic)
- Сложная логика доставки
