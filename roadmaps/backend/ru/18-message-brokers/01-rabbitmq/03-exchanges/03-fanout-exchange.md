# Fanout Exchange

## Введение

Fanout Exchange - это тип обменника в RabbitMQ, который рассылает копию каждого сообщения **во все привязанные очереди** без учёта routing key. Это классическая реализация паттерна Publish/Subscribe (Pub/Sub).

## Принцип работы

```
                              ┌──────────────┐
                         ┌───▶│   Queue 1    │
                         │    └──────────────┘
┌──────────┐    ┌────────┴───────┐
│ Producer │───▶│ Fanout Exchange│ ┌──────────────┐
└──────────┘    │                │─▶│   Queue 2    │
                └────────┬───────┘ └──────────────┘
                         │    ┌──────────────┐
                         └───▶│   Queue 3    │
                              └──────────────┘

Сообщение доставляется во ВСЕ привязанные очереди
routing_key ИГНОРИРУЕТСЯ
```

### Ключевые особенности:

1. **Broadcast** - каждое сообщение копируется во все привязанные очереди
2. **Routing key игнорируется** - можно указать любой или пустой
3. **Самый быстрый тип** - нет логики маршрутизации
4. **Идеален для Pub/Sub** - один publisher, много subscribers

## Схема Fanout Exchange

```
┌─────────────┐
│  Publisher  │
└──────┬──────┘
       │ message
       ▼
┌──────────────────┐
│  Fanout Exchange │  routing_key = "" (игнорируется)
│   "broadcast"    │
└────────┬─────────┘
         │
    ┌────┼────┬────────┐
    │    │    │        │
    ▼    ▼    ▼        ▼
┌─────┐┌─────┐┌─────┐┌─────┐
│ Q1  ││ Q2  ││ Q3  ││ Q4  │
└──┬──┘└──┬──┘└──┬──┘└──┬──┘
   │      │      │      │
   ▼      ▼      ▼      ▼
┌─────┐┌─────┐┌─────┐┌─────┐
│ C1  ││ C2  ││ C3  ││ C4  │
└─────┘└─────┘└─────┘└─────┘
(Каждый consumer получает копию)
```

## Базовый пример

### Publisher (издатель)

```python
import pika

# Подключение к RabbitMQ
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Объявление Fanout Exchange
channel.exchange_declare(
    exchange='notifications',
    exchange_type='fanout',
    durable=True
)

# Отправка сообщения (routing_key игнорируется)
message = "Важное уведомление для всех!"

channel.basic_publish(
    exchange='notifications',
    routing_key='',  # Можно указать что угодно - будет игнорироваться
    body=message,
    properties=pika.BasicProperties(
        delivery_mode=2  # Persistent
    )
)

print(f" [x] Отправлено: {message}")
connection.close()
```

### Subscriber (подписчик)

```python
import pika

# Подключение к RabbitMQ
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Объявление Exchange (идемпотентно)
channel.exchange_declare(
    exchange='notifications',
    exchange_type='fanout',
    durable=True
)

# Создание эксклюзивной очереди для этого подписчика
result = channel.queue_declare(queue='', exclusive=True)
queue_name = result.method.queue

# Привязка к exchange (routing_key не нужен)
channel.queue_bind(
    exchange='notifications',
    queue=queue_name
    # routing_key не указываем - он игнорируется
)

def callback(ch, method, properties, body):
    print(f" [x] Получено: {body.decode()}")

channel.basic_consume(
    queue=queue_name,
    on_message_callback=callback,
    auto_ack=True
)

print(' [*] Ожидание уведомлений. Для выхода нажмите CTRL+C')
channel.start_consuming()
```

## Use Case 1: Система уведомлений

### Архитектура

```
┌─────────────────┐
│  Order Service  │
└────────┬────────┘
         │ "order.placed"
         ▼
┌─────────────────────┐
│   Fanout Exchange   │
│   "order_events"    │
└──────────┬──────────┘
           │
    ┌──────┼──────┬──────────────┐
    │      │      │              │
    ▼      ▼      ▼              ▼
┌───────┐┌───────┐┌───────┐┌──────────┐
│ Email ││  SMS  ││ Push  ││ Analytics│
│Service││Service││Service││  Service │
└───────┘└───────┘└───────┘└──────────┘
```

### Реализация

```python
import pika
import json
from datetime import datetime

class NotificationPublisher:
    """Издатель событий для системы уведомлений"""

    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()

        # Exchange для уведомлений о заказах
        self.channel.exchange_declare(
            exchange='order_events',
            exchange_type='fanout',
            durable=True
        )

    def publish_order_event(self, event_type: str, order_data: dict):
        """Публикация события заказа"""
        event = {
            'type': event_type,
            'timestamp': datetime.utcnow().isoformat(),
            'data': order_data
        }

        self.channel.basic_publish(
            exchange='order_events',
            routing_key='',  # Игнорируется
            body=json.dumps(event),
            properties=pika.BasicProperties(
                delivery_mode=2,
                content_type='application/json'
            )
        )
        print(f"Событие {event_type} опубликовано")

    def order_placed(self, order_id: str, customer_email: str, amount: float):
        self.publish_order_event('order.placed', {
            'order_id': order_id,
            'customer_email': customer_email,
            'amount': amount
        })

    def order_shipped(self, order_id: str, tracking_number: str):
        self.publish_order_event('order.shipped', {
            'order_id': order_id,
            'tracking_number': tracking_number
        })

    def close(self):
        self.connection.close()


class NotificationSubscriber:
    """Базовый подписчик на уведомления"""

    def __init__(self, service_name: str, handler_func, host='localhost'):
        self.service_name = service_name
        self.handler = handler_func

        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()

        # Объявляем exchange
        self.channel.exchange_declare(
            exchange='order_events',
            exchange_type='fanout',
            durable=True
        )

        # Создаём именованную durable очередь для сервиса
        self.queue_name = f'{service_name}_queue'
        self.channel.queue_declare(queue=self.queue_name, durable=True)

        # Привязываем к exchange
        self.channel.queue_bind(
            exchange='order_events',
            queue=self.queue_name
        )

        # Настраиваем consumer
        self.channel.basic_consume(
            queue=self.queue_name,
            on_message_callback=self._on_message
        )

    def _on_message(self, ch, method, properties, body):
        event = json.loads(body)
        print(f"[{self.service_name}] Получено: {event['type']}")

        try:
            self.handler(event)
            ch.basic_ack(delivery_tag=method.delivery_tag)
        except Exception as e:
            print(f"[{self.service_name}] Ошибка: {e}")
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)

    def start(self):
        print(f"[{self.service_name}] Подписчик запущен")
        self.channel.start_consuming()


# Пример использования
def email_handler(event):
    if event['type'] == 'order.placed':
        print(f"Отправляем email на {event['data']['customer_email']}")

def sms_handler(event):
    if event['type'] == 'order.shipped':
        print(f"Отправляем SMS о доставке заказа {event['data']['order_id']}")

def analytics_handler(event):
    print(f"Записываем в аналитику: {event['type']}")


# Запуск
if __name__ == "__main__":
    # Publisher
    publisher = NotificationPublisher()
    publisher.order_placed('ORD-123', 'user@example.com', 99.99)
    publisher.order_shipped('ORD-123', 'TRACK-456')
    publisher.close()
```

## Use Case 2: Инвалидация кэша

### Архитектура

```
┌──────────────────┐
│   Data Service   │  (изменил данные)
└────────┬─────────┘
         │ "cache.invalidate"
         ▼
┌────────────────────────┐
│    Fanout Exchange     │
│   "cache_invalidation" │
└───────────┬────────────┘
            │
     ┌──────┼──────┬──────────┐
     │      │      │          │
     ▼      ▼      ▼          ▼
┌────────┐┌────────┐┌────────┐┌────────┐
│Server 1││Server 2││Server 3││Server 4│
│ Cache  ││ Cache  ││ Cache  ││ Cache  │
└────────┘└────────┘└────────┘└────────┘
   │          │          │          │
   ▼          ▼          ▼          ▼
 clear      clear      clear      clear
```

### Реализация

```python
import pika
import json
import uuid

class CacheInvalidator:
    """Публикация событий инвалидации кэша"""

    def __init__(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()

        self.channel.exchange_declare(
            exchange='cache_invalidation',
            exchange_type='fanout',
            durable=True
        )

    def invalidate(self, cache_key: str):
        """Инвалидировать ключ кэша на всех серверах"""
        message = {
            'action': 'invalidate',
            'key': cache_key,
            'timestamp': datetime.utcnow().isoformat()
        }

        self.channel.basic_publish(
            exchange='cache_invalidation',
            routing_key='',
            body=json.dumps(message)
        )
        print(f"Инвалидация {cache_key} отправлена")

    def invalidate_pattern(self, pattern: str):
        """Инвалидировать ключи по паттерну"""
        message = {
            'action': 'invalidate_pattern',
            'pattern': pattern
        }

        self.channel.basic_publish(
            exchange='cache_invalidation',
            routing_key='',
            body=json.dumps(message)
        )

    def clear_all(self):
        """Очистить весь кэш"""
        message = {'action': 'clear_all'}

        self.channel.basic_publish(
            exchange='cache_invalidation',
            routing_key='',
            body=json.dumps(message)
        )


class CacheSubscriber:
    """Подписчик на события инвалидации кэша"""

    def __init__(self, server_id: str, cache_client):
        self.server_id = server_id
        self.cache = cache_client

        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()

        self.channel.exchange_declare(
            exchange='cache_invalidation',
            exchange_type='fanout',
            durable=True
        )

        # Эксклюзивная очередь - удаляется при отключении
        result = self.channel.queue_declare(queue='', exclusive=True)
        self.queue_name = result.method.queue

        self.channel.queue_bind(
            exchange='cache_invalidation',
            queue=self.queue_name
        )

        self.channel.basic_consume(
            queue=self.queue_name,
            on_message_callback=self._handle_message,
            auto_ack=True
        )

    def _handle_message(self, ch, method, properties, body):
        message = json.loads(body)
        action = message['action']

        if action == 'invalidate':
            key = message['key']
            self.cache.delete(key)
            print(f"[{self.server_id}] Удалён ключ: {key}")

        elif action == 'invalidate_pattern':
            pattern = message['pattern']
            # Удаление по паттерну (зависит от реализации кэша)
            self.cache.delete_pattern(pattern)
            print(f"[{self.server_id}] Удалены ключи по паттерну: {pattern}")

        elif action == 'clear_all':
            self.cache.clear()
            print(f"[{self.server_id}] Кэш очищен")

    def start(self):
        print(f"[{self.server_id}] Подписка на инвалидацию кэша")
        self.channel.start_consuming()
```

## Use Case 3: Real-time обновления (WebSocket broadcast)

```python
import pika
import json

class RealtimeBroadcaster:
    """Broadcast real-time обновлений на все WebSocket серверы"""

    def __init__(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()

        self.channel.exchange_declare(
            exchange='realtime_updates',
            exchange_type='fanout',
            durable=False  # Не нужно сохранять при перезапуске
        )

    def broadcast(self, event_type: str, data: dict):
        """Отправить обновление на все WebSocket серверы"""
        message = {
            'event': event_type,
            'data': data
        }

        self.channel.basic_publish(
            exchange='realtime_updates',
            routing_key='',
            body=json.dumps(message)
        )

    def user_online(self, user_id: str):
        self.broadcast('user.online', {'user_id': user_id})

    def new_message(self, chat_id: str, message: str, sender_id: str):
        self.broadcast('chat.message', {
            'chat_id': chat_id,
            'message': message,
            'sender_id': sender_id
        })

    def typing_indicator(self, chat_id: str, user_id: str):
        self.broadcast('chat.typing', {
            'chat_id': chat_id,
            'user_id': user_id
        })
```

## Сравнение с другими Exchange

| Характеристика | Fanout | Direct | Topic |
|----------------|--------|--------|-------|
| Routing key | Игнорируется | Точное совпадение | Паттерн |
| Производительность | Самая высокая | Высокая | Средняя |
| Гибкость | Низкая | Средняя | Высокая |
| Use case | Broadcast | Точная маршрутизация | Категории |

## Fanout vs множественные Direct bindings

```
# Fanout (правильно для broadcast):
exchange_declare(exchange='events', exchange_type='fanout')
queue_bind(exchange='events', queue='q1')  # routing_key не нужен
queue_bind(exchange='events', queue='q2')
queue_bind(exchange='events', queue='q3')

# Direct (неправильно для broadcast):
exchange_declare(exchange='events', exchange_type='direct')
queue_bind(exchange='events', queue='q1', routing_key='all')
queue_bind(exchange='events', queue='q2', routing_key='all')
queue_bind(exchange='events', queue='q3', routing_key='all')
# Работает, но менее эффективно!
```

## Динамические подписчики

Fanout идеален для динамически добавляемых/удаляемых подписчиков:

```python
class DynamicSubscriber:
    """Подписчик, который может динамически подключаться/отключаться"""

    def __init__(self, subscriber_id: str):
        self.subscriber_id = subscriber_id
        self.connection = None
        self.channel = None
        self.queue_name = None

    def subscribe(self, exchange: str):
        """Подписаться на exchange"""
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()

        # Эксклюзивная очередь - автоудаление при отключении
        result = self.channel.queue_declare(
            queue='',
            exclusive=True  # Удаляется при закрытии соединения
        )
        self.queue_name = result.method.queue

        self.channel.queue_bind(
            exchange=exchange,
            queue=self.queue_name
        )

        print(f"[{self.subscriber_id}] Подписан на {exchange}")

    def unsubscribe(self):
        """Отписаться (очередь удалится автоматически)"""
        if self.connection:
            self.connection.close()
            self.connection = None
            print(f"[{self.subscriber_id}] Отписан")

    def consume(self, callback):
        """Начать получать сообщения"""
        self.channel.basic_consume(
            queue=self.queue_name,
            on_message_callback=callback,
            auto_ack=True
        )
        self.channel.start_consuming()
```

## Best Practices

### 1. Используйте exclusive очереди для временных подписчиков

```python
# Временный подписчик (очередь удалится при отключении)
result = channel.queue_declare(queue='', exclusive=True)

# Постоянный подписчик (очередь сохраняется)
channel.queue_declare(queue='email_notifications', durable=True)
```

### 2. Не злоупотребляйте fanout

```python
# Плохо - fanout для разных типов сообщений
channel.exchange_declare(exchange='all_events', exchange_type='fanout')
# Все получают ВСЁ, даже что не нужно

# Хорошо - разделяйте по exchange или используйте topic
channel.exchange_declare(exchange='order_events', exchange_type='fanout')
channel.exchange_declare(exchange='user_events', exchange_type='fanout')
```

### 3. Учитывайте нагрузку

```python
# Fanout дублирует сообщения
# 1 сообщение * 100 подписчиков = 100 сообщений в очередях

# Для больших payload-ов используйте ссылки
message = {
    'type': 'large_file_ready',
    'file_url': 'https://storage.example.com/file.zip'  # Ссылка вместо данных
}
```

## Производительность

Fanout Exchange - самый быстрый тип:

```
Сравнение (относительно fanout):
- Fanout:  1.0x (базовая линия)
- Direct:  1.1x
- Topic:   1.5-2x
- Headers: 2-3x
```

Причина: нет логики маршрутизации, просто копирование во все привязки.

## Частые ошибки

### 1. Создание очереди без привязки

```python
# Ошибка: очередь создана, но не привязана
channel.exchange_declare(exchange='events', exchange_type='fanout')
channel.queue_declare(queue='my_queue')
# Забыли: channel.queue_bind(...)
# Сообщения не доходят!
```

### 2. Использование routing_key как фильтра

```python
# Ошибка: routing_key игнорируется в fanout
channel.basic_publish(
    exchange='events',
    routing_key='important',  # Это НЕ фильтр!
    body='message'
)
# Сообщение придёт ВСЕМ подписчикам
```

### 3. Durable exchange с exclusive queues

```python
# При перезапуске сервера:
# - Exchange сохранится (durable=True)
# - Очереди удалятся (exclusive=True)
# - Сообщения потеряются!

# Решение: durable queues для важных сообщений
channel.queue_declare(queue='important_notifications', durable=True)
```

## Заключение

Fanout Exchange - идеальный выбор для:
- Broadcast уведомлений
- Pub/Sub паттерна
- Инвалидации кэша
- Real-time обновлений
- Логирования на несколько систем

Не используйте Fanout, когда:
- Нужна фильтрация сообщений (Topic)
- Нужна точная маршрутизация (Direct)
- Разные подписчики должны получать разные сообщения
