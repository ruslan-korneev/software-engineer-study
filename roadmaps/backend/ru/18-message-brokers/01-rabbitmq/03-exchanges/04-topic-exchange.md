# Topic Exchange

## Введение

Topic Exchange - это самый гибкий тип обменника в RabbitMQ, который маршрутизирует сообщения на основе **паттерн-матчинга** routing key. Это позволяет подписчикам получать только те сообщения, которые соответствуют определённым категориям или паттернам.

## Принцип работы

```
┌──────────┐                           ┌───────────────┐
│ Producer │──routing_key="order.us.created"──▶│ Topic Exchange│
└──────────┘                           └───────┬───────┘
                                               │
                    ┌──────────────────────────┼──────────────────────────┐
                    │                          │                          │
                    ▼                          ▼                          ▼
            ┌──────────────┐          ┌──────────────┐          ┌──────────────┐
            │ order.*.created │         │  order.us.*  │          │      #       │
            │   Queue A    │          │   Queue B    │          │   Queue C    │
            └──────────────┘          └──────────────┘          └──────────────┘
                  ✓                         ✓                         ✓
```

### Формат routing key:

- Routing key состоит из слов, разделённых точками: `word1.word2.word3`
- Примеры: `order.created`, `user.europe.signup`, `log.error.database`

### Wildcards (специальные символы):

| Символ | Описание | Пример |
|--------|----------|--------|
| `*` (звёздочка) | Заменяет **ровно одно** слово | `order.*.created` |
| `#` (решётка) | Заменяет **ноль или более** слов | `order.#` |

## Примеры паттернов

```
Routing Key сообщения: "order.europe.created"

Паттерны привязок:
┌─────────────────────┬─────────┬─────────────────────────────┐
│ Паттерн             │ Матч?   │ Объяснение                  │
├─────────────────────┼─────────┼─────────────────────────────┤
│ order.europe.created│   ✓     │ Точное совпадение           │
│ order.*.created     │   ✓     │ * = "europe" (одно слово)   │
│ order.#             │   ✓     │ # = "europe.created"        │
│ #.created           │   ✓     │ # = "order.europe"          │
│ order.*             │   ✗     │ * ожидает 1 слово, есть 2   │
│ *.europe.*          │   ✓     │ * = "order", * = "created"  │
│ #                   │   ✓     │ # матчит всё                │
│ order.asia.created  │   ✗     │ "asia" ≠ "europe"           │
└─────────────────────┴─────────┴─────────────────────────────┘
```

## Схема Topic Exchange

```
                                    Routing Keys:
Producer 1 ─── "stock.usd.nyse" ───┐
                                   │
Producer 2 ─── "stock.eur.lse" ────┤    ┌─────────────────────┐
                                   ├───▶│   Topic Exchange    │
Producer 3 ─── "forex.usd.eur" ────┤    │    "markets"        │
                                   │    └──────────┬──────────┘
Producer 4 ─── "crypto.btc.sell" ──┘               │
                                                   │
                    ┌──────────────────────────────┼──────────────┐
                    │              │               │              │
              "stock.usd.*"  "stock.#"       "*.*.nyse"      "#"
                    │              │               │              │
                    ▼              ▼               ▼              ▼
              ┌─────────┐   ┌─────────┐     ┌─────────┐    ┌─────────┐
              │ NYSE USD│   │All Stock│     │  NYSE   │    │   All   │
              │ Tracker │   │ Tracker │     │ Monitor │    │ Logger  │
              └─────────┘   └─────────┘     └─────────┘    └─────────┘
```

## Базовый пример

### Publisher

```python
import pika
import sys

# Подключение к RabbitMQ
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Объявление Topic Exchange
channel.exchange_declare(
    exchange='topic_logs',
    exchange_type='topic',
    durable=True
)

# Формат routing_key: <facility>.<severity>
# Например: "auth.error", "order.info", "payment.critical"

routing_key = sys.argv[1] if len(sys.argv) > 1 else 'anonymous.info'
message = ' '.join(sys.argv[2:]) or 'Hello World!'

channel.basic_publish(
    exchange='topic_logs',
    routing_key=routing_key,
    body=message,
    properties=pika.BasicProperties(
        delivery_mode=2
    )
)

print(f" [x] Отправлено {routing_key}: {message}")
connection.close()
```

### Subscriber

```python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.exchange_declare(
    exchange='topic_logs',
    exchange_type='topic',
    durable=True
)

# Создаём эксклюзивную очередь
result = channel.queue_declare(queue='', exclusive=True)
queue_name = result.method.queue

# Паттерны для подписки
binding_keys = sys.argv[1:] if len(sys.argv) > 1 else ['#']

for binding_key in binding_keys:
    channel.queue_bind(
        exchange='topic_logs',
        queue=queue_name,
        routing_key=binding_key
    )
    print(f" [*] Подписка на: {binding_key}")

def callback(ch, method, properties, body):
    print(f" [x] {method.routing_key}: {body.decode()}")

channel.basic_consume(
    queue=queue_name,
    on_message_callback=callback,
    auto_ack=True
)

print(' [*] Ожидание логов. Для выхода нажмите CTRL+C')
channel.start_consuming()
```

### Запуск

```bash
# Терминал 1: Все ошибки
python subscriber.py "*.error"

# Терминал 2: Все логи от auth
python subscriber.py "auth.#"

# Терминал 3: Критические ошибки auth
python subscriber.py "auth.critical"

# Терминал 4: Отправка
python publisher.py auth.error "Ошибка авторизации"
python publisher.py auth.critical "Критическая ошибка!"
python publisher.py order.info "Заказ создан"
```

## Use Case 1: Микросервисная архитектура

### Архитектура событий

```
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│  Order Service  │   │  User Service   │   │Payment Service  │
└────────┬────────┘   └────────┬────────┘   └────────┬────────┘
         │                     │                     │
"order.created"        "user.registered"     "payment.completed"
"order.updated"        "user.deleted"        "payment.failed"
"order.cancelled"      "user.verified"       "payment.refunded"
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               ▼
                    ┌─────────────────────┐
                    │   Topic Exchange    │
                    │     "events"        │
                    └──────────┬──────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
    "order.#"            "*.created"            "#"
         │                     │                     │
         ▼                     ▼                     ▼
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│ Order Analytics │   │   Notification  │   │   Audit Log     │
│    Service      │   │    Service      │   │    Service      │
└─────────────────┘   └─────────────────┘   └─────────────────┘
```

### Реализация

```python
import pika
import json
from datetime import datetime
from typing import Callable, List

class EventPublisher:
    """Издатель событий для микросервисной архитектуры"""

    def __init__(self, service_name: str, host='localhost'):
        self.service_name = service_name
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()

        self.channel.exchange_declare(
            exchange='events',
            exchange_type='topic',
            durable=True
        )

    def publish(self, event_type: str, data: dict):
        """
        Публикация события.
        event_type: строка вида "entity.action", например "order.created"
        """
        routing_key = event_type

        event = {
            'type': event_type,
            'source': self.service_name,
            'timestamp': datetime.utcnow().isoformat(),
            'data': data
        }

        self.channel.basic_publish(
            exchange='events',
            routing_key=routing_key,
            body=json.dumps(event),
            properties=pika.BasicProperties(
                delivery_mode=2,
                content_type='application/json'
            )
        )

        print(f"[{self.service_name}] Опубликовано: {routing_key}")

    def close(self):
        self.connection.close()


class EventSubscriber:
    """Подписчик на события с поддержкой паттернов"""

    def __init__(self, service_name: str, patterns: List[str], host='localhost'):
        self.service_name = service_name
        self.patterns = patterns
        self.handlers: dict[str, Callable] = {}

        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()

        self.channel.exchange_declare(
            exchange='events',
            exchange_type='topic',
            durable=True
        )

        # Создаём durable очередь для сервиса
        self.queue_name = f'{service_name}_events'
        self.channel.queue_declare(queue=self.queue_name, durable=True)

        # Привязываем ко всем паттернам
        for pattern in patterns:
            self.channel.queue_bind(
                exchange='events',
                queue=self.queue_name,
                routing_key=pattern
            )
            print(f"[{service_name}] Подписка на: {pattern}")

        self.channel.basic_qos(prefetch_count=1)

    def on(self, event_pattern: str, handler: Callable):
        """Регистрация обработчика для определённого типа событий"""
        self.handlers[event_pattern] = handler

    def _match_pattern(self, pattern: str, routing_key: str) -> bool:
        """Проверка соответствия routing_key паттерну"""
        pattern_parts = pattern.split('.')
        key_parts = routing_key.split('.')

        pi, ki = 0, 0
        while pi < len(pattern_parts) and ki < len(key_parts):
            if pattern_parts[pi] == '#':
                if pi == len(pattern_parts) - 1:
                    return True
                pi += 1
                while ki < len(key_parts):
                    if self._match_pattern('.'.join(pattern_parts[pi:]),
                                          '.'.join(key_parts[ki:])):
                        return True
                    ki += 1
                return False
            elif pattern_parts[pi] == '*' or pattern_parts[pi] == key_parts[ki]:
                pi += 1
                ki += 1
            else:
                return False

        if pi == len(pattern_parts) and ki == len(key_parts):
            return True
        if pi < len(pattern_parts) and pattern_parts[pi] == '#':
            return True
        return False

    def _on_message(self, ch, method, properties, body):
        event = json.loads(body)
        routing_key = method.routing_key

        print(f"[{self.service_name}] Получено: {routing_key}")

        # Находим подходящий обработчик
        for pattern, handler in self.handlers.items():
            if self._match_pattern(pattern, routing_key):
                try:
                    handler(event)
                except Exception as e:
                    print(f"[{self.service_name}] Ошибка обработки: {e}")

        ch.basic_ack(delivery_tag=method.delivery_tag)

    def start(self):
        self.channel.basic_consume(
            queue=self.queue_name,
            on_message_callback=self._on_message
        )
        print(f"[{self.service_name}] Запущен")
        self.channel.start_consuming()


# Пример использования
if __name__ == "__main__":
    # Издатель (Order Service)
    order_publisher = EventPublisher('order-service')
    order_publisher.publish('order.created', {
        'order_id': 'ORD-123',
        'user_id': 'USR-456',
        'amount': 99.99
    })
    order_publisher.publish('order.shipped', {
        'order_id': 'ORD-123',
        'tracking': 'TRACK-789'
    })
    order_publisher.close()

    # Подписчик (Notification Service)
    notification_subscriber = EventSubscriber(
        'notification-service',
        patterns=['order.created', 'order.shipped', 'payment.failed']
    )

    notification_subscriber.on('order.created', lambda e:
        print(f"Отправляем email: Заказ {e['data']['order_id']} создан"))

    notification_subscriber.on('order.shipped', lambda e:
        print(f"Отправляем SMS: Заказ {e['data']['order_id']} отправлен"))

    # notification_subscriber.start()
```

## Use Case 2: Система логирования

```python
import pika
import json
from datetime import datetime

class StructuredLogger:
    """
    Структурированный логгер с topic-based маршрутизацией.
    Формат routing_key: <app>.<module>.<severity>
    Примеры: "api.auth.error", "worker.payment.info", "web.cart.warning"
    """

    def __init__(self, app_name: str, host='localhost'):
        self.app_name = app_name

        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()

        self.channel.exchange_declare(
            exchange='logs',
            exchange_type='topic',
            durable=True
        )

    def log(self, module: str, severity: str, message: str, **extra):
        routing_key = f"{self.app_name}.{module}.{severity}"

        log_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'app': self.app_name,
            'module': module,
            'severity': severity,
            'message': message,
            **extra
        }

        self.channel.basic_publish(
            exchange='logs',
            routing_key=routing_key,
            body=json.dumps(log_entry),
            properties=pika.BasicProperties(
                delivery_mode=2,
                content_type='application/json'
            )
        )

    def debug(self, module: str, message: str, **extra):
        self.log(module, 'debug', message, **extra)

    def info(self, module: str, message: str, **extra):
        self.log(module, 'info', message, **extra)

    def warning(self, module: str, message: str, **extra):
        self.log(module, 'warning', message, **extra)

    def error(self, module: str, message: str, **extra):
        self.log(module, 'error', message, **extra)

    def critical(self, module: str, message: str, **extra):
        self.log(module, 'critical', message, **extra)


class LogSubscriber:
    """
    Подписчик на логи с гибкой фильтрацией.

    Примеры паттернов:
    - "#" - все логи
    - "*.*.error" - все ошибки
    - "api.#" - все логи от api
    - "api.auth.*" - все логи auth модуля api
    """

    def __init__(self, name: str, patterns: list, handler, host='localhost'):
        self.name = name
        self.handler = handler

        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()

        self.channel.exchange_declare(
            exchange='logs',
            exchange_type='topic',
            durable=True
        )

        result = self.channel.queue_declare(queue=f'logs_{name}', durable=True)
        self.queue_name = result.method.queue

        for pattern in patterns:
            self.channel.queue_bind(
                exchange='logs',
                queue=self.queue_name,
                routing_key=pattern
            )

        self.channel.basic_consume(
            queue=self.queue_name,
            on_message_callback=self._on_message
        )

    def _on_message(self, ch, method, properties, body):
        log_entry = json.loads(body)
        self.handler(log_entry)
        ch.basic_ack(delivery_tag=method.delivery_tag)

    def start(self):
        self.channel.start_consuming()


# Пример использования
logger = StructuredLogger('api')
logger.info('auth', 'Пользователь вошёл в систему', user_id='123')
logger.error('payment', 'Ошибка оплаты', order_id='456', error='timeout')
logger.critical('database', 'Потеряно соединение с БД', host='db1')

# Подписчики:
# 1. Elasticsearch - все логи
# LogSubscriber('elasticsearch', ['#'], lambda l: send_to_elastic(l))

# 2. PagerDuty - только critical
# LogSubscriber('pagerduty', ['*.*.critical'], lambda l: send_alert(l))

# 3. Error tracker - все ошибки
# LogSubscriber('errors', ['*.*.error', '*.*.critical'], lambda l: track_error(l))
```

## Use Case 3: Мультирегиональная система

```
Routing Key формат: <entity>.<region>.<action>

Примеры:
- order.us.created
- order.eu.shipped
- user.asia.registered
- payment.us.failed
```

```python
import pika
import json

class RegionalEventBus:
    """Event bus с поддержкой регионов"""

    REGIONS = ['us', 'eu', 'asia']

    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()

        self.channel.exchange_declare(
            exchange='regional_events',
            exchange_type='topic',
            durable=True
        )

    def emit(self, entity: str, region: str, action: str, data: dict):
        """Emit события с региональной маршрутизацией"""
        routing_key = f"{entity}.{region}.{action}"

        self.channel.basic_publish(
            exchange='regional_events',
            routing_key=routing_key,
            body=json.dumps(data),
            properties=pika.BasicProperties(delivery_mode=2)
        )


class RegionalSubscriber:
    """
    Подписчик с региональной фильтрацией.

    Примеры:
    - Глобальный аналитик: ["#"]
    - US-only сервис: ["*.us.*"]
    - Все заказы EU: ["order.eu.#"]
    - Все созданные заказы: ["order.*.created"]
    """

    def __init__(self, service_name: str, patterns: list):
        self.service_name = service_name

        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()

        self.channel.exchange_declare(
            exchange='regional_events',
            exchange_type='topic',
            durable=True
        )

        self.queue_name = f'{service_name}_queue'
        self.channel.queue_declare(queue=self.queue_name, durable=True)

        for pattern in patterns:
            self.channel.queue_bind(
                exchange='regional_events',
                queue=self.queue_name,
                routing_key=pattern
            )
            print(f"[{service_name}] Подписка: {pattern}")


# Примеры подписчиков:

# 1. US Order Processor - обрабатывает только US заказы
# RegionalSubscriber('us-order-processor', ['order.us.*'])

# 2. Global Analytics - все события
# RegionalSubscriber('global-analytics', ['#'])

# 3. EU Compliance - все EU события для аудита
# RegionalSubscriber('eu-compliance', ['*.eu.*'])

# 4. Order Created Notifier - уведомления о всех созданных заказах
# RegionalSubscriber('order-notifier', ['order.*.created'])
```

## Сложные паттерны

### Комбинация * и #

```python
# Примеры сложных паттернов

# "stock.*.*.buy" - покупка акций на любой бирже любой валюты
# Матчит: stock.usd.nyse.buy, stock.eur.lse.buy
# Не матчит: stock.buy, stock.usd.buy

# "log.#.error" - ошибки из любого источника
# Матчит: log.error, log.api.error, log.api.auth.error
# Не матчит: log.api.auth.warning

# "#.critical.#" - всё со словом critical
# Матчит: critical, app.critical, app.critical.db, a.b.critical.c.d
```

### Подписка на несколько паттернов

```python
# Одна очередь может иметь несколько привязок с разными паттернами

channel.queue_bind(exchange='events', queue='alerts',
                   routing_key='*.*.error')
channel.queue_bind(exchange='events', queue='alerts',
                   routing_key='*.*.critical')
channel.queue_bind(exchange='events', queue='alerts',
                   routing_key='security.#')

# Очередь 'alerts' получит:
# - Все ошибки: app.module.error
# - Все критические: app.module.critical
# - Все security события: security.breach, security.login.failed
```

## Производительность

Topic Exchange медленнее Direct из-за паттерн-матчинга:

```
Сравнение производительности:
- Direct:  ~50,000 msg/sec
- Topic:   ~25,000-35,000 msg/sec (зависит от сложности паттернов)
- Headers: ~15,000-20,000 msg/sec
```

Рекомендации:
- Избегайте слишком длинных routing keys (более 5-7 слов)
- Минимизируйте количество привязок с `#` в начале
- Для простых случаев рассмотрите Direct exchange

## Best Practices

### 1. Продуманная структура routing key

```python
# Хорошо - понятная иерархия
"order.created"           # entity.action
"order.us.created"        # entity.region.action
"log.api.auth.error"      # category.app.module.severity

# Плохо - непоследовательно
"created_order"
"us-order-new"
"error-auth-api"
```

### 2. Документируйте схему routing keys

```python
"""
Routing Key Schema:

Format: <domain>.<entity>.<action>

Domains:
- order: заказы
- user: пользователи
- payment: платежи

Actions:
- created, updated, deleted
- completed, failed, cancelled

Examples:
- order.order.created
- user.profile.updated
- payment.transaction.failed
"""
```

### 3. Используйте константы

```python
class RoutingKeys:
    """Константы для routing keys"""

    # Orders
    ORDER_CREATED = 'order.order.created'
    ORDER_UPDATED = 'order.order.updated'
    ORDER_CANCELLED = 'order.order.cancelled'

    # Payments
    PAYMENT_COMPLETED = 'payment.transaction.completed'
    PAYMENT_FAILED = 'payment.transaction.failed'

    # Patterns
    ALL_ORDERS = 'order.#'
    ALL_ERRORS = '*.*.failed'
```

## Частые ошибки

### 1. Путаница между * и #

```python
# * = ровно одно слово
# # = ноль или более слов

# Ошибка: ожидание что * матчит несколько слов
channel.queue_bind(exchange='e', queue='q', routing_key='order.*')
# НЕ матчит: order.us.created (три слова)
# Матчит: order.created (два слова)

# Правильно:
channel.queue_bind(exchange='e', queue='q', routing_key='order.#')
```

### 2. # в середине паттерна

```python
# Технически работает, но может быть неожиданным

# "a.#.b" матчит:
# - a.b
# - a.x.b
# - a.x.y.b
# - a.x.y.z.b

# Лучше избегать такие паттерны - они сложны для понимания
```

## Заключение

Topic Exchange - самый мощный инструмент маршрутизации в RabbitMQ:

**Используйте когда:**
- Нужна гибкая фильтрация по категориям
- Микросервисная архитектура с доменными событиями
- Сложная система логирования
- Мультирегиональные/мультитенантные системы

**Не используйте когда:**
- Достаточно точной маршрутизации (Direct)
- Нужен broadcast всем (Fanout)
- Критична максимальная производительность
