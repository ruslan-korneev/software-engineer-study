# Direct Exchange

## Введение

Direct Exchange - это тип обменника в RabbitMQ, который маршрутизирует сообщения в очереди на основе **точного совпадения** routing key. Это самый простой и понятный тип маршрутизации после Default Exchange.

## Принцип работы

```
                              routing_key привязки
                              ┌─────────────────┐
┌──────────┐                  │                 ▼
│ Producer │──routing_key──▶ [Direct Exchange]──▶ [Queue]
└──────────┘
                              Совпадение routing_key
                              сообщения и привязки
```

### Алгоритм маршрутизации:

1. Producer отправляет сообщение с `routing_key`
2. Direct Exchange сравнивает `routing_key` сообщения с `routing_key` каждой привязки
3. Если есть **точное совпадение** - сообщение отправляется в соответствующую очередь
4. Одно сообщение может попасть в несколько очередей (если они привязаны с одинаковым ключом)

## Схема Direct Exchange

```
                              ┌──────────────────┐
                         ┌───▶│  errors_queue    │ (routing_key="error")
                         │    └──────────────────┘
┌──────────┐    ┌────────┴───────┐
│ Producer │───▶│ Direct Exchange│ ┌──────────────────┐
└──────────┘    │   "logs"       │─▶│  warnings_queue  │ (routing_key="warning")
                └────────┬───────┘ └──────────────────┘
                         │    ┌──────────────────┐
                         └───▶│   info_queue     │ (routing_key="info")
                              └──────────────────┘

Сообщение с routing_key="error" попадёт ТОЛЬКО в errors_queue
```

## Базовый пример

### Producer (отправитель)

```python
import pika

# Подключение к RabbitMQ
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Объявление Direct Exchange
channel.exchange_declare(
    exchange='direct_logs',
    exchange_type='direct',
    durable=True
)

# Отправка сообщений с разными routing keys
severities = ['info', 'warning', 'error']

for severity in severities:
    message = f"Это сообщение уровня {severity}"

    channel.basic_publish(
        exchange='direct_logs',
        routing_key=severity,  # Ключ маршрутизации
        body=message,
        properties=pika.BasicProperties(
            delivery_mode=2  # Persistent
        )
    )
    print(f" [x] Отправлено {severity}: {message}")

connection.close()
```

### Consumer (получатель)

```python
import pika
import sys

# Подключение к RabbitMQ
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Объявление Exchange
channel.exchange_declare(
    exchange='direct_logs',
    exchange_type='direct',
    durable=True
)

# Создание очереди
result = channel.queue_declare(queue='', exclusive=True)
queue_name = result.method.queue

# Получаем уровни логирования из аргументов
severities = sys.argv[1:] if len(sys.argv) > 1 else ['info']

# Привязываем очередь к exchange для каждого уровня
for severity in severities:
    channel.queue_bind(
        exchange='direct_logs',
        queue=queue_name,
        routing_key=severity
    )
    print(f" [*] Подписка на: {severity}")

def callback(ch, method, properties, body):
    print(f" [x] Получено [{method.routing_key}]: {body.decode()}")

channel.basic_consume(
    queue=queue_name,
    on_message_callback=callback,
    auto_ack=True
)

print(' [*] Ожидание сообщений. Для выхода нажмите CTRL+C')
channel.start_consuming()
```

## Use Case: Система логирования

### Архитектура

```
                              ┌─────────────────┐
                         ┌───▶│ error_processor │──▶ Алерты, PagerDuty
                         │    └─────────────────┘
┌────────────┐    ┌──────┴──────┐
│ Application│───▶│ logs_direct │ ┌─────────────────┐
└────────────┘    │   Exchange  │─▶│ warning_handler │──▶ Мониторинг
                  └──────┬──────┘ └─────────────────┘
                         │    ┌─────────────────┐
                         └───▶│   all_logs      │──▶ Elasticsearch
                              └─────────────────┘
                              (привязан к error, warning, info)
```

### Реализация

```python
import pika
import json
from datetime import datetime

class LogPublisher:
    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()

        # Объявляем exchange
        self.channel.exchange_declare(
            exchange='logs_direct',
            exchange_type='direct',
            durable=True
        )

    def log(self, level: str, message: str, context: dict = None):
        """Отправка лог-сообщения"""
        log_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'level': level,
            'message': message,
            'context': context or {}
        }

        self.channel.basic_publish(
            exchange='logs_direct',
            routing_key=level,
            body=json.dumps(log_entry),
            properties=pika.BasicProperties(
                delivery_mode=2,
                content_type='application/json'
            )
        )

    def info(self, message: str, **context):
        self.log('info', message, context)

    def warning(self, message: str, **context):
        self.log('warning', message, context)

    def error(self, message: str, **context):
        self.log('error', message, context)

    def close(self):
        self.connection.close()


# Использование
logger = LogPublisher()
logger.info("Приложение запущено", version="1.0.0")
logger.warning("Высокая нагрузка CPU", cpu_percent=85)
logger.error("Ошибка подключения к БД", error_code=500)
logger.close()
```

## Use Case: Распределение задач по типам

### Архитектура

```
                              ┌──────────────────┐
                         ┌───▶│ image_processor  │ (routing_key="image")
                         │    │   Worker 1       │
                         │    │   Worker 2       │
                         │    └──────────────────┘
┌────────────┐    ┌──────┴──────┐
│   API      │───▶│   tasks     │ ┌──────────────────┐
│  Server    │    │   Exchange  │─▶│  pdf_processor   │ (routing_key="pdf")
└────────────┘    └──────┬──────┘ │   Worker 1       │
                         │        └──────────────────┘
                         │    ┌──────────────────┐
                         └───▶│ video_processor  │ (routing_key="video")
                              │   Worker 1       │
                              │   Worker 2       │
                              │   Worker 3       │
                              └──────────────────┘
```

### Код

```python
import pika
import json

class TaskDispatcher:
    """Диспетчер задач с маршрутизацией по типу"""

    def __init__(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()

        # Exchange для задач
        self.channel.exchange_declare(
            exchange='tasks',
            exchange_type='direct',
            durable=True
        )

        # Очереди для разных типов задач
        task_types = ['image', 'pdf', 'video']
        for task_type in task_types:
            queue_name = f'{task_type}_tasks'

            self.channel.queue_declare(queue=queue_name, durable=True)
            self.channel.queue_bind(
                exchange='tasks',
                queue=queue_name,
                routing_key=task_type
            )

    def dispatch(self, task_type: str, task_data: dict):
        """Отправка задачи в соответствующую очередь"""
        self.channel.basic_publish(
            exchange='tasks',
            routing_key=task_type,
            body=json.dumps(task_data),
            properties=pika.BasicProperties(
                delivery_mode=2,
                content_type='application/json'
            )
        )
        print(f"Задача {task_type} отправлена: {task_data}")


class TaskWorker:
    """Воркер для обработки определённого типа задач"""

    def __init__(self, task_type: str, processor_func):
        self.task_type = task_type
        self.processor = processor_func

        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()

        queue_name = f'{task_type}_tasks'
        self.channel.queue_declare(queue=queue_name, durable=True)

        # Prefetch - берём по одной задаче
        self.channel.basic_qos(prefetch_count=1)

        self.channel.basic_consume(
            queue=queue_name,
            on_message_callback=self._handle_task
        )

    def _handle_task(self, ch, method, properties, body):
        task_data = json.loads(body)
        print(f"[{self.task_type}] Получена задача: {task_data}")

        try:
            self.processor(task_data)
            ch.basic_ack(delivery_tag=method.delivery_tag)
            print(f"[{self.task_type}] Задача выполнена")
        except Exception as e:
            print(f"[{self.task_type}] Ошибка: {e}")
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)

    def start(self):
        print(f"[{self.task_type}] Воркер запущен")
        self.channel.start_consuming()


# Пример использования
if __name__ == "__main__":
    # Отправка задач
    dispatcher = TaskDispatcher()
    dispatcher.dispatch('image', {'file': 'photo.jpg', 'action': 'resize'})
    dispatcher.dispatch('pdf', {'file': 'doc.pdf', 'action': 'convert'})
    dispatcher.dispatch('video', {'file': 'movie.mp4', 'action': 'transcode'})
```

## Множественные привязки с одним ключом

Одна очередь может быть привязана к нескольким ключам, и несколько очередей могут быть привязаны к одному ключу:

```python
# Очередь критических алертов получает и error, и critical
channel.queue_bind(exchange='logs', queue='critical_alerts', routing_key='error')
channel.queue_bind(exchange='logs', queue='critical_alerts', routing_key='critical')

# Обе очереди получают error
channel.queue_bind(exchange='logs', queue='file_log', routing_key='error')
channel.queue_bind(exchange='logs', queue='email_alert', routing_key='error')
```

```
                              ┌────────────────┐
                         ┌───▶│ critical_alerts│◀──┐
                         │    └────────────────┘   │
"error"──────────────────┤                         │
                         │    ┌────────────────┐   │
                         └───▶│   file_log     │   │
                              └────────────────┘   │
                                                   │
"critical"─────────────────────────────────────────┘
```

## Best Practices

### 1. Именование routing keys

```python
# Хорошо - понятные, консистентные имена
routing_keys = ['error', 'warning', 'info', 'debug']
routing_keys = ['order.create', 'order.update', 'order.delete']

# Плохо - непонятные сокращения
routing_keys = ['e', 'w', 'i', 'd']
```

### 2. Используйте durable для production

```python
# Exchange
channel.exchange_declare(
    exchange='production_logs',
    exchange_type='direct',
    durable=True  # Переживёт перезапуск
)

# Queue
channel.queue_declare(
    queue='error_logs',
    durable=True
)

# Message
channel.basic_publish(
    exchange='production_logs',
    routing_key='error',
    body=message,
    properties=pika.BasicProperties(delivery_mode=2)  # Persistent
)
```

### 3. Обработка недоставленных сообщений

```python
# Alternate exchange для сообщений без маршрута
channel.exchange_declare(
    exchange='main_direct',
    exchange_type='direct',
    arguments={'alternate-exchange': 'unrouted'}
)

channel.exchange_declare(
    exchange='unrouted',
    exchange_type='fanout'
)

channel.queue_declare(queue='unrouted_messages')
channel.queue_bind(exchange='unrouted', queue='unrouted_messages')
```

## Когда использовать Direct Exchange

| Сценарий | Подходит |
|----------|----------|
| Логирование по уровням | Да |
| Распределение задач по типам | Да |
| RPC вызовы | Да |
| Broadcast всем подписчикам | Нет (Fanout) |
| Гибкие паттерны routing | Нет (Topic) |
| Маршрутизация по заголовкам | Нет (Headers) |

## Производительность

Direct Exchange - один из самых быстрых типов:
- Простое сравнение строк (O(1) с хеш-таблицей)
- Минимальные накладные расходы
- Подходит для высоконагруженных систем

## Частые ошибки

### 1. Забыли привязать очередь

```python
# Ошибка: сообщения пропадают
channel.exchange_declare(exchange='logs', exchange_type='direct')
channel.queue_declare(queue='my_queue')
# Забыли: channel.queue_bind(...)
channel.basic_publish(exchange='logs', routing_key='error', body='test')
# Сообщение потеряно!
```

### 2. Опечатка в routing key

```python
# Producer отправляет с 'error'
channel.basic_publish(exchange='logs', routing_key='error', body='test')

# Consumer подписан на 'errors' (с s на конце)
channel.queue_bind(exchange='logs', queue='q', routing_key='errors')
# Сообщение не доставлено!
```

## Заключение

Direct Exchange - надёжный выбор для:
- Простой точной маршрутизации
- Систем логирования
- Распределения задач
- RPC паттернов

Когда нужна более гибкая маршрутизация - используйте Topic Exchange.
