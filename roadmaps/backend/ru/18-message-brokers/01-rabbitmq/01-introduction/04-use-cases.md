# Сценарии использования RabbitMQ

## Введение

RabbitMQ — универсальный инструмент, который находит применение во множестве архитектурных паттернов. В этом разделе мы рассмотрим основные сценарии использования с практическими примерами кода.

---

## 1. Очереди задач (Task Queues / Work Queues)

### Описание

Task Queue — это паттерн распределения ресурсоёмких задач между несколькими воркерами. Сообщения не обрабатываются сразу, а ставятся в очередь для последующей обработки.

```
                    ┌──────────────┐
                    │   Worker 1   │
Producer ──────────▶│              │
                    └──────────────┘
    │                      ▲
    │   ┌──────────┐       │
    └──▶│  Queue   │───────┼───────▶ Worker 2
        └──────────┘       │
                           │
                    ┌──────────────┐
                    │   Worker 3   │
                    └──────────────┘
```

### Типичные задачи

- Отправка email/SMS уведомлений
- Генерация отчётов
- Обработка изображений и видео
- Импорт/экспорт данных
- Индексация для поиска

### Пример реализации

```python
# producer.py - Отправитель задач
import pika
import json

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Создаём durable очередь (переживёт перезапуск RabbitMQ)
channel.queue_declare(queue='email_tasks', durable=True)

def send_email_task(to, subject, body):
    """Отправляет задачу на отправку email в очередь"""
    task = {
        'to': to,
        'subject': subject,
        'body': body
    }

    channel.basic_publish(
        exchange='',
        routing_key='email_tasks',
        body=json.dumps(task),
        properties=pika.BasicProperties(
            delivery_mode=2,  # Персистентное сообщение
            content_type='application/json'
        )
    )
    print(f"[x] Задача отправлена: {to}")

# Отправляем несколько задач
send_email_task('user1@example.com', 'Welcome!', 'Hello, User 1!')
send_email_task('user2@example.com', 'Welcome!', 'Hello, User 2!')
send_email_task('user3@example.com', 'Welcome!', 'Hello, User 3!')

connection.close()
```

```python
# worker.py - Обработчик задач
import pika
import json
import time

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(queue='email_tasks', durable=True)

# Fair dispatch: получаем новую задачу только после завершения текущей
channel.basic_qos(prefetch_count=1)

def send_email(to, subject, body):
    """Имитация отправки email"""
    print(f"  Отправка email на {to}...")
    time.sleep(2)  # Имитация работы
    print(f"  Email отправлен на {to}")

def callback(ch, method, properties, body):
    task = json.loads(body)
    print(f"[x] Получена задача: {task['to']}")

    try:
        send_email(task['to'], task['subject'], task['body'])
        ch.basic_ack(delivery_tag=method.delivery_tag)
        print(f"[✓] Задача выполнена: {task['to']}")
    except Exception as e:
        print(f"[!] Ошибка: {e}")
        # Возвращаем в очередь для повторной попытки
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)

channel.basic_consume(queue='email_tasks', on_message_callback=callback)

print('[*] Ожидание задач. Для выхода нажмите CTRL+C')
channel.start_consuming()
```

---

## 2. Микросервисная архитектура

### Описание

RabbitMQ обеспечивает слабосвязанную коммуникацию между микросервисами, позволяя им работать независимо и масштабироваться отдельно.

```
┌─────────────┐         ┌─────────────────────┐         ┌─────────────┐
│   Order     │────────▶│      RabbitMQ       │────────▶│  Inventory  │
│   Service   │         │                     │         │   Service   │
└─────────────┘         │  order.created ───▶ │         └─────────────┘
                        │                     │
                        │  payment.confirmed ─┼────────▶ ┌─────────────┐
                        │                     │         │  Shipping   │
┌─────────────┐         │  notification.* ────┼────────▶│   Service   │
│   Payment   │────────▶│                     │         └─────────────┘
│   Service   │         │                     │
└─────────────┘         └─────────────────────┘
                                  │
                                  ▼
                        ┌─────────────────────┐
                        │   Notification      │
                        │      Service        │
                        └─────────────────────┘
```

### Пример: Система обработки заказов

```python
# order_service.py - Сервис заказов
import pika
import json
from datetime import datetime

class OrderService:
    def __init__(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()

        # Создаём Topic Exchange для событий
        self.channel.exchange_declare(
            exchange='orders',
            exchange_type='topic',
            durable=True
        )

    def create_order(self, user_id, items, total):
        """Создаёт заказ и публикует событие"""
        order = {
            'order_id': f'ORD-{datetime.now().timestamp()}',
            'user_id': user_id,
            'items': items,
            'total': total,
            'status': 'created',
            'created_at': datetime.now().isoformat()
        }

        # Сохраняем в БД (имитация)
        print(f"[Order Service] Заказ создан: {order['order_id']}")

        # Публикуем событие
        self.channel.basic_publish(
            exchange='orders',
            routing_key='order.created',
            body=json.dumps(order),
            properties=pika.BasicProperties(
                delivery_mode=2,
                content_type='application/json'
            )
        )

        return order

# Использование
service = OrderService()
order = service.create_order(
    user_id='user-123',
    items=[{'sku': 'LAPTOP-001', 'qty': 1}],
    total=999.99
)
```

```python
# inventory_service.py - Сервис инвентаря
import pika
import json

class InventoryService:
    def __init__(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()

        # Подписываемся на события заказов
        self.channel.exchange_declare(
            exchange='orders',
            exchange_type='topic',
            durable=True
        )

        result = self.channel.queue_declare(
            queue='inventory.order.created',
            durable=True
        )

        self.channel.queue_bind(
            exchange='orders',
            queue='inventory.order.created',
            routing_key='order.created'
        )

    def process_order_created(self, ch, method, properties, body):
        """Резервирует товары при создании заказа"""
        order = json.loads(body)
        print(f"[Inventory] Резервирование товаров для {order['order_id']}")

        for item in order['items']:
            # Резервируем товар (имитация)
            print(f"  - Зарезервирован: {item['sku']} x {item['qty']}")

        ch.basic_ack(delivery_tag=method.delivery_tag)

        # Публикуем событие о резервировании
        self.channel.basic_publish(
            exchange='orders',
            routing_key='inventory.reserved',
            body=json.dumps({
                'order_id': order['order_id'],
                'status': 'reserved'
            })
        )

    def start(self):
        self.channel.basic_qos(prefetch_count=1)
        self.channel.basic_consume(
            queue='inventory.order.created',
            on_message_callback=self.process_order_created
        )
        print('[Inventory Service] Ожидание событий...')
        self.channel.start_consuming()

# Запуск
service = InventoryService()
service.start()
```

---

## 3. Event-Driven Architecture (EDA)

### Описание

Event-Driven Architecture — архитектурный паттерн, где компоненты общаются через события. RabbitMQ выступает как Event Bus.

### Паттерны событий

| Паттерн | Описание | RabbitMQ реализация |
|---------|----------|---------------------|
| **Publish/Subscribe** | Один ко многим | Fanout Exchange |
| **Event Notification** | Уведомление о событии | Topic Exchange |
| **Event-Carried State Transfer** | Событие с данными | Any Exchange + payload |
| **Event Sourcing** | История событий | Persistent queues |

### Пример: Pub/Sub с Fanout Exchange

```python
# event_publisher.py - Издатель событий
import pika
import json

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Fanout: отправка всем подписчикам
channel.exchange_declare(exchange='events.broadcast', exchange_type='fanout')

def publish_event(event_type, data):
    """Публикует событие всем подписчикам"""
    event = {
        'type': event_type,
        'data': data,
        'timestamp': '2024-01-15T10:30:00Z'
    }

    channel.basic_publish(
        exchange='events.broadcast',
        routing_key='',  # Игнорируется для fanout
        body=json.dumps(event)
    )
    print(f"[Event] Опубликовано: {event_type}")

# Публикуем события
publish_event('user.registered', {'user_id': '123', 'email': 'user@example.com'})
publish_event('user.verified', {'user_id': '123'})
```

```python
# event_subscriber.py - Подписчик на события
import pika
import json

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='events.broadcast', exchange_type='fanout')

# Каждый подписчик создаёт свою очередь
result = channel.queue_declare(queue='', exclusive=True)
queue_name = result.method.queue

channel.queue_bind(exchange='events.broadcast', queue=queue_name)

def on_event(ch, method, properties, body):
    event = json.loads(body)
    print(f"[Subscriber] Получено событие: {event['type']}")
    print(f"  Данные: {event['data']}")

channel.basic_consume(queue=queue_name, on_message_callback=on_event, auto_ack=True)

print('[*] Ожидание событий...')
channel.start_consuming()
```

---

## 4. RPC (Remote Procedure Call)

### Описание

RabbitMQ позволяет реализовать синхронный RPC поверх асинхронного протокола, что полезно для запросов, требующих немедленного ответа.

```
┌────────────┐      Request       ┌─────────────┐      Request       ┌────────────┐
│   Client   │───────────────────▶│   RabbitMQ  │───────────────────▶│   Server   │
│            │                    │             │                    │            │
│            │◀───────────────────│             │◀───────────────────│            │
└────────────┘      Response      └─────────────┘      Response      └────────────┘
     │                                                                     │
     │  correlation_id='abc123'                    correlation_id='abc123' │
     │  reply_to='amq.gen-xxx'                                             │
     └─────────────────────────────────────────────────────────────────────┘
```

### Пример реализации RPC

```python
# rpc_server.py - RPC Сервер
import pika
import json

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(queue='rpc_queue')

def calculate_fibonacci(n):
    """Вычисляет число Фибоначчи"""
    if n <= 1:
        return n
    return calculate_fibonacci(n - 1) + calculate_fibonacci(n - 2)

def on_request(ch, method, props, body):
    request = json.loads(body)
    n = request.get('n', 0)

    print(f"[RPC Server] Вычисление fib({n})")
    result = calculate_fibonacci(n)

    response = {'result': result}

    # Отправляем ответ в очередь reply_to
    ch.basic_publish(
        exchange='',
        routing_key=props.reply_to,
        body=json.dumps(response),
        properties=pika.BasicProperties(
            correlation_id=props.correlation_id  # Связываем с запросом
        )
    )

    ch.basic_ack(delivery_tag=method.delivery_tag)
    print(f"[RPC Server] Ответ отправлен: fib({n}) = {result}")

channel.basic_qos(prefetch_count=1)
channel.basic_consume(queue='rpc_queue', on_message_callback=on_request)

print('[RPC Server] Ожидание запросов...')
channel.start_consuming()
```

```python
# rpc_client.py - RPC Клиент
import pika
import json
import uuid

class FibonacciRpcClient:
    def __init__(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()

        # Создаём эксклюзивную очередь для ответов
        result = self.channel.queue_declare(queue='', exclusive=True)
        self.callback_queue = result.method.queue

        self.channel.basic_consume(
            queue=self.callback_queue,
            on_message_callback=self.on_response,
            auto_ack=True
        )

        self.response = None
        self.corr_id = None

    def on_response(self, ch, method, props, body):
        if self.corr_id == props.correlation_id:
            self.response = json.loads(body)

    def call(self, n):
        """Вызывает удалённую функцию fib(n)"""
        self.response = None
        self.corr_id = str(uuid.uuid4())

        request = {'n': n}

        self.channel.basic_publish(
            exchange='',
            routing_key='rpc_queue',
            body=json.dumps(request),
            properties=pika.BasicProperties(
                reply_to=self.callback_queue,
                correlation_id=self.corr_id
            )
        )

        # Ждём ответ
        while self.response is None:
            self.connection.process_data_events(time_limit=None)

        return self.response['result']

# Использование
client = FibonacciRpcClient()
print(f"[RPC Client] Запрос: fib(30)")
result = client.call(30)
print(f"[RPC Client] Результат: {result}")
```

---

## 5. Отложенные сообщения (Delayed Messages)

### Описание

Сценарий, когда сообщение должно быть обработано не сразу, а через определённое время.

### Реализация через Dead Letter Exchange

```python
# delayed_messages.py
import pika
import json
from datetime import datetime

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Очередь для обработки (целевая)
channel.queue_declare(queue='scheduled_tasks', durable=True)

# Exchange для отложенных сообщений
channel.exchange_declare(exchange='delayed', exchange_type='direct')

# Очередь-ожидание с TTL и DLX
def create_delay_queue(delay_ms):
    """Создаёт очередь с заданной задержкой"""
    queue_name = f'delay.{delay_ms}ms'

    channel.queue_declare(
        queue=queue_name,
        durable=True,
        arguments={
            'x-message-ttl': delay_ms,  # Время жизни сообщения
            'x-dead-letter-exchange': '',  # Default exchange
            'x-dead-letter-routing-key': 'scheduled_tasks'  # Целевая очередь
        }
    )

    channel.queue_bind(
        exchange='delayed',
        queue=queue_name,
        routing_key=str(delay_ms)
    )

    return queue_name

# Создаём очереди для разных задержек
create_delay_queue(5000)   # 5 секунд
create_delay_queue(60000)  # 1 минута
create_delay_queue(300000) # 5 минут

def schedule_task(task, delay_ms):
    """Планирует задачу с задержкой"""
    message = {
        'task': task,
        'scheduled_at': datetime.now().isoformat(),
        'delay_ms': delay_ms
    }

    channel.basic_publish(
        exchange='delayed',
        routing_key=str(delay_ms),
        body=json.dumps(message),
        properties=pika.BasicProperties(delivery_mode=2)
    )

    print(f"[Scheduler] Задача запланирована через {delay_ms}ms: {task}")

# Планируем задачи
schedule_task('send_reminder_email', 5000)
schedule_task('cleanup_temp_files', 60000)
schedule_task('generate_daily_report', 300000)
```

---

## 6. Приоритетные очереди

### Описание

RabbitMQ поддерживает очереди с приоритетами, позволяя обрабатывать важные сообщения первыми.

```python
# priority_queue.py
import pika
import json

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Создаём очередь с поддержкой приоритетов
channel.queue_declare(
    queue='priority_tasks',
    durable=True,
    arguments={
        'x-max-priority': 10  # Максимальный приоритет
    }
)

def publish_task(task, priority):
    """Публикует задачу с указанным приоритетом"""
    channel.basic_publish(
        exchange='',
        routing_key='priority_tasks',
        body=json.dumps(task),
        properties=pika.BasicProperties(
            delivery_mode=2,
            priority=priority  # Приоритет от 0 до 10
        )
    )
    print(f"[Publisher] Задача отправлена (priority={priority}): {task['name']}")

# Отправляем задачи с разными приоритетами
publish_task({'name': 'low_priority_task', 'data': '...'}, priority=1)
publish_task({'name': 'normal_task', 'data': '...'}, priority=5)
publish_task({'name': 'high_priority_task', 'data': '...'}, priority=9)
publish_task({'name': 'urgent_task', 'data': '...'}, priority=10)

# Consumer получит: urgent -> high -> normal -> low
```

---

## 7. Dead Letter Queue (DLQ)

### Описание

Dead Letter Queue — очередь для сообщений, которые не удалось обработать. Это важный паттерн для обработки ошибок.

```python
# dlq_setup.py
import pika
import json

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# 1. Создаём DLX (Dead Letter Exchange)
channel.exchange_declare(exchange='dlx', exchange_type='direct')

# 2. Создаём DLQ (Dead Letter Queue)
channel.queue_declare(queue='dead_letters', durable=True)
channel.queue_bind(exchange='dlx', queue='dead_letters', routing_key='failed')

# 3. Создаём основную очередь с DLX
channel.queue_declare(
    queue='main_queue',
    durable=True,
    arguments={
        'x-dead-letter-exchange': 'dlx',
        'x-dead-letter-routing-key': 'failed',
        'x-message-ttl': 60000  # Опционально: TTL
    }
)

def process_message(ch, method, properties, body):
    """Обработчик с логикой retry"""
    message = json.loads(body)
    headers = properties.headers or {}
    retry_count = headers.get('x-retry-count', 0)

    try:
        # Имитация ошибки
        if message.get('fail', False):
            raise ValueError("Simulated failure")

        print(f"[Worker] Успешно обработано: {message}")
        ch.basic_ack(delivery_tag=method.delivery_tag)

    except Exception as e:
        print(f"[Worker] Ошибка: {e}")

        if retry_count < 3:
            # Повторная попытка
            print(f"[Worker] Повторная попытка #{retry_count + 1}")
            ch.basic_publish(
                exchange='',
                routing_key='main_queue',
                body=body,
                properties=pika.BasicProperties(
                    headers={'x-retry-count': retry_count + 1}
                )
            )
            ch.basic_ack(delivery_tag=method.delivery_tag)
        else:
            # Отклоняем в DLQ
            print(f"[Worker] Превышен лимит попыток, отправка в DLQ")
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)

channel.basic_qos(prefetch_count=1)
channel.basic_consume(queue='main_queue', on_message_callback=process_message)
channel.start_consuming()
```

---

## Best Practices

### Общие рекомендации

1. **Делайте сообщения идемпотентными** — повторная обработка не должна вызывать проблем
2. **Используйте correlation_id** для отслеживания сообщений
3. **Логируйте delivery_tag** для отладки
4. **Устанавливайте TTL** для предотвращения накопления
5. **Мониторьте длину очередей** через Management API

### Паттерны обработки ошибок

```python
# Retry с экспоненциальной задержкой
def calculate_retry_delay(retry_count, base_delay=1000, max_delay=60000):
    delay = min(base_delay * (2 ** retry_count), max_delay)
    return delay

# Использование в DLQ handler
retry_delay = calculate_retry_delay(retry_count)
# Отправить в delay queue с рассчитанной задержкой
```

---

## Заключение

RabbitMQ поддерживает множество паттернов использования:

| Паттерн | Основной Exchange | Особенности |
|---------|-------------------|-------------|
| Task Queues | Default | prefetch, ack |
| Pub/Sub | Fanout | Broadcast |
| Routing | Direct/Topic | Гибкая маршрутизация |
| RPC | Default | correlation_id, reply_to |
| Delayed | DLX | TTL + Dead Letter |
| Priority | Default | x-max-priority |
| DLQ | DLX | Обработка ошибок |

Выбор паттерна зависит от конкретных требований вашего приложения, но гибкость RabbitMQ позволяет комбинировать их для создания сложных и надёжных систем.
