# Паттерны маршрутизации

## Введение в паттерны маршрутизации

Паттерны маршрутизации позволяют гибко направлять сообщения в очереди на основе routing keys. Topic exchange в RabbitMQ предоставляет мощный механизм pattern matching с использованием wildcards.

```
┌─────────────────────────────────────────────────────────────────┐
│                         Topic Exchange                          │
│                                                                 │
│   Routing Key: "order.europe.germany.created"                   │
│                     │       │       │       │                   │
│   Pattern Match:    ▼       ▼       ▼       ▼                   │
│                   order  europe  germany  created               │
│                     │       │       │       │                   │
│   "order.#"    ─────┴───────┴───────┴───────┘  ✓ Match          │
│   "*.europe.*.*" ────────────┘       │          ✓ Match          │
│   "order.*.*.created" ───────────────┘          ✓ Match          │
│   "order.asia.#" ──────────────────────────     ✗ No Match       │
└─────────────────────────────────────────────────────────────────┘
```

## Wildcards в RabbitMQ

### Символ * (звёздочка)

Заменяет **ровно одно слово** в routing key:

```python
# Паттерн: "order.*"

"order.created"     # ✓ Match
"order.updated"     # ✓ Match
"order.cancelled"   # ✓ Match
"order.item.added"  # ✗ No Match (2 слова после order)
"order"             # ✗ No Match (0 слов после order)
```

### Символ # (решётка)

Заменяет **ноль или более слов**:

```python
# Паттерн: "order.#"

"order"                    # ✓ Match (0 слов)
"order.created"            # ✓ Match (1 слово)
"order.item.created"       # ✓ Match (2 слова)
"order.item.price.updated" # ✓ Match (3 слова)
"user.created"             # ✗ No Match (не начинается с order)
```

### Комбинации wildcards

```python
# Паттерн: "*.*.created"
"order.new.created"      # ✓ Match
"user.profile.created"   # ✓ Match
"order.created"          # ✗ No Match (только 2 слова)

# Паттерн: "#.created"
"order.created"          # ✓ Match
"user.profile.created"   # ✓ Match
"created"                # ✓ Match
"order.item.new.created" # ✓ Match

# Паттерн: "order.*.#"
"order.new"              # ✓ Match
"order.new.created"      # ✓ Match
"order.new.item.added"   # ✓ Match
"order"                  # ✗ No Match (нужно минимум 1 слово после order.)
```

## Практические сценарии

### Сценарий 1: Географическая маршрутизация

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

channel.exchange_declare(exchange='geo_events', exchange_type='topic')

# Структура routing key: region.country.city.event
geo_bindings = [
    # Все события из Европы
    ('europe_all', 'europe.#'),

    # Только события из Германии
    ('germany_only', 'europe.germany.*.*'),

    # Все события из столиц (любой регион)
    ('capitals', '*.*.capital.*'),

    # Все события типа "alert" из любого места
    ('all_alerts', '#.alert'),

    # Глобальный мониторинг
    ('global_monitor', '#'),
]

for queue_name, pattern in geo_bindings:
    channel.queue_declare(queue=queue_name, durable=True)
    channel.queue_bind(
        queue=queue_name,
        exchange='geo_events',
        routing_key=pattern
    )

# Публикация событий
events = [
    ('europe.germany.berlin.alert', 'Системная ошибка в Берлине'),
    ('europe.france.capital.update', 'Обновление в Париже'),
    ('asia.japan.tokyo.alert', 'Тревога в Токио'),
    ('europe.germany.munich.info', 'Информация из Мюнхена'),
]

for routing_key, message in events:
    channel.basic_publish(
        exchange='geo_events',
        routing_key=routing_key,
        body=message
    )
    print(f"Sent to '{routing_key}': {message}")

connection.close()
```

Матрица маршрутизации:

```
Routing Key                    │europe.#│ europe.  │*.*.     │  #.   │  #  │
                               │        │germany.*.*│capital.*│alert │     │
───────────────────────────────┼────────┼──────────┼─────────┼──────┼─────┤
europe.germany.berlin.alert    │   ✓    │    ✓     │    ✗    │  ✓   │  ✓  │
europe.france.capital.update   │   ✓    │    ✗     │    ✓    │  ✗   │  ✓  │
asia.japan.tokyo.alert         │   ✗    │    ✗     │    ✗    │  ✓   │  ✓  │
europe.germany.munich.info     │   ✓    │    ✓     │    ✗    │  ✗   │  ✓  │
```

### Сценарий 2: Логирование по уровням

```python
import pika
from enum import Enum
from datetime import datetime

class LogLevel(Enum):
    DEBUG = 'debug'
    INFO = 'info'
    WARNING = 'warning'
    ERROR = 'error'
    CRITICAL = 'critical'

def setup_logging_exchange():
    """Настройка exchange для логирования."""

    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    channel.exchange_declare(exchange='logs', exchange_type='topic')

    # Структура: service.module.level
    log_queues = [
        # Все логи от auth сервиса
        ('auth_all_logs', 'auth.#'),

        # Только ошибки и критические от любого сервиса
        ('all_errors', '*.*.error'),
        ('all_critical', '*.*.critical'),

        # Все важные логи (warning и выше)
        ('important_logs', ['#.warning', '#.error', '#.critical']),

        # Debug логи для разработки
        ('debug_logs', '*.*.debug'),

        # Все логи для архива
        ('archive_all', '#'),
    ]

    for queue_name, patterns in log_queues:
        channel.queue_declare(queue=queue_name, durable=True)

        # patterns может быть строкой или списком
        if isinstance(patterns, str):
            patterns = [patterns]

        for pattern in patterns:
            channel.queue_bind(
                queue=queue_name,
                exchange='logs',
                routing_key=pattern
            )
            print(f"Bound '{queue_name}' to '{pattern}'")

    connection.close()

def log_message(service: str, module: str, level: LogLevel, message: str):
    """Отправка лог-сообщения."""

    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    routing_key = f'{service}.{module}.{level.value}'

    log_entry = {
        'timestamp': datetime.utcnow().isoformat(),
        'service': service,
        'module': module,
        'level': level.value,
        'message': message
    }

    import json
    channel.basic_publish(
        exchange='logs',
        routing_key=routing_key,
        body=json.dumps(log_entry)
    )

    print(f"[{level.value.upper()}] {service}.{module}: {message}")
    connection.close()

# Использование
setup_logging_exchange()

log_message('auth', 'login', LogLevel.INFO, 'User logged in')
log_message('auth', 'login', LogLevel.ERROR, 'Invalid password')
log_message('orders', 'payment', LogLevel.CRITICAL, 'Payment gateway down')
log_message('api', 'request', LogLevel.DEBUG, 'Request received')
```

### Сценарий 3: Микросервисная архитектура

```python
import pika
import json
from typing import List, Callable

class EventRouter:
    """Маршрутизатор событий для микросервисов."""

    def __init__(self, host: str = 'localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()
        self.exchange = 'microservices'

        self.channel.exchange_declare(
            exchange=self.exchange,
            exchange_type='topic',
            durable=True
        )

    def subscribe(self, service_name: str, patterns: List[str],
                  callback: Callable):
        """Подписка сервиса на определённые паттерны событий."""

        queue_name = f'{service_name}_events'
        self.channel.queue_declare(queue=queue_name, durable=True)

        for pattern in patterns:
            self.channel.queue_bind(
                queue=queue_name,
                exchange=self.exchange,
                routing_key=pattern
            )
            print(f"[{service_name}] Subscribed to: {pattern}")

        def wrapper(ch, method, properties, body):
            event = json.loads(body)
            callback(event, method.routing_key)
            ch.basic_ack(delivery_tag=method.delivery_tag)

        self.channel.basic_consume(
            queue=queue_name,
            on_message_callback=wrapper
        )

    def publish(self, routing_key: str, event: dict):
        """Публикация события."""

        self.channel.basic_publish(
            exchange=self.exchange,
            routing_key=routing_key,
            body=json.dumps(event),
            properties=pika.BasicProperties(
                delivery_mode=2,
                content_type='application/json'
            )
        )

    def start_consuming(self):
        self.channel.start_consuming()

    def close(self):
        self.connection.close()


# Пример: Notification Service
def notification_handler(event: dict, routing_key: str):
    print(f"[Notification] Got event '{routing_key}': {event}")

router = EventRouter()

# Notification service подписывается на события, требующие уведомлений
router.subscribe(
    service_name='notification',
    patterns=[
        'user.*.registered',      # Новые пользователи
        'order.*.completed',      # Завершённые заказы
        '#.error',                # Все ошибки
        '#.alert',                # Все алерты
    ],
    callback=notification_handler
)
```

## Сложные паттерны маршрутизации

### Паттерн: Content-Based Router

```
┌─────────────────────────────────────────────────────────────┐
│                    Content-Based Router                      │
│                                                              │
│   ┌──────────┐     ┌─────────────┐     ┌──────────────┐     │
│   │ Producer │────>│   Router    │────>│ High Priority│     │
│   │          │     │  Exchange   │     │    Queue     │     │
│   └──────────┘     │  (topic)    │     └──────────────┘     │
│                    │             │                          │
│                    │             │────>┌──────────────┐     │
│                    │             │     │ Medium Prio. │     │
│                    │             │     │    Queue     │     │
│                    │             │     └──────────────┘     │
│                    │             │                          │
│                    │             │────>┌──────────────┐     │
│                    └─────────────┘     │  Low Priority│     │
│                                        │    Queue     │     │
│                                        └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

```python
import pika

def setup_priority_router():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    channel.exchange_declare(exchange='priority_router', exchange_type='topic')

    # Маршрутизация по приоритету в routing key
    priority_config = [
        ('high_priority', ['*.high.#', '*.urgent.#', '*.critical.#']),
        ('medium_priority', ['*.medium.#', '*.normal.#']),
        ('low_priority', ['*.low.#', '*.background.#']),
    ]

    for queue_name, patterns in priority_config:
        # Создаём очередь с разными параметрами
        args = {}
        if queue_name == 'high_priority':
            args['x-max-priority'] = 10

        channel.queue_declare(queue=queue_name, durable=True, arguments=args)

        for pattern in patterns:
            channel.queue_bind(
                queue=queue_name,
                exchange='priority_router',
                routing_key=pattern
            )

    connection.close()
```

### Паттерн: Scatter-Gather

```
┌─────────────────────────────────────────────────────────────────┐
│                      Scatter-Gather Pattern                      │
│                                                                  │
│   ┌──────────┐     ┌─────────────┐     ┌──────────┐             │
│   │ Request  │────>│   Scatter   │────>│ Worker 1 │──┐          │
│   │          │     │  Exchange   │     └──────────┘  │          │
│   └──────────┘     │  (fanout)   │                   │          │
│                    │             │────>┌──────────┐  │          │
│                    │             │     │ Worker 2 │──┤          │
│                    │             │     └──────────┘  │          │
│                    │             │                   │          │
│                    │             │────>┌──────────┐  │          │
│                    └─────────────┘     │ Worker 3 │──┤          │
│                                        └──────────┘  │          │
│                                                      ▼          │
│   ┌──────────┐     ┌─────────────┐     ┌──────────────┐         │
│   │ Response │<────│   Gather    │<────│ Reply Queue  │         │
│   │          │     │  (direct)   │     │              │         │
│   └──────────┘     └─────────────┘     └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

```python
import pika
import json
import uuid
from concurrent.futures import ThreadPoolExecutor

class ScatterGather:
    def __init__(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()

        # Scatter exchange (fanout для рассылки всем)
        self.channel.exchange_declare(
            exchange='scatter',
            exchange_type='fanout'
        )

        # Gather exchange (direct для сбора ответов)
        self.channel.exchange_declare(
            exchange='gather',
            exchange_type='direct'
        )

    def scatter_request(self, request: dict, timeout: int = 5):
        """Отправляет запрос всем workers и собирает ответы."""

        correlation_id = str(uuid.uuid4())

        # Создаём временную очередь для ответов
        result = self.channel.queue_declare(queue='', exclusive=True)
        reply_queue = result.method.queue

        self.channel.queue_bind(
            queue=reply_queue,
            exchange='gather',
            routing_key=correlation_id
        )

        # Отправляем запрос
        self.channel.basic_publish(
            exchange='scatter',
            routing_key='',
            body=json.dumps(request),
            properties=pika.BasicProperties(
                reply_to=reply_queue,
                correlation_id=correlation_id
            )
        )

        # Собираем ответы
        responses = []
        # ... сбор ответов с таймаутом

        return responses
```

### Паттерн: Dead Letter с маршрутизацией

```python
import pika

def setup_dlx_routing():
    """Настройка Dead Letter Exchange с умной маршрутизацией."""

    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Основной exchange
    channel.exchange_declare(
        exchange='main',
        exchange_type='topic'
    )

    # Dead Letter Exchange с маршрутизацией по причине
    channel.exchange_declare(
        exchange='dlx',
        exchange_type='topic'
    )

    # Очереди для разных типов DLX сообщений
    dlx_queues = [
        ('dlx_expired', 'dlx.expired.#'),     # TTL истёк
        ('dlx_rejected', 'dlx.rejected.#'),   # Отклонённые
        ('dlx_max_length', 'dlx.overflow.#'), # Переполнение
        ('dlx_all', 'dlx.#'),                 # Все DLX сообщения
    ]

    for queue_name, pattern in dlx_queues:
        channel.queue_declare(queue=queue_name, durable=True)
        channel.queue_bind(
            queue=queue_name,
            exchange='dlx',
            routing_key=pattern
        )

    # Основные очереди с DLX
    channel.queue_declare(
        queue='orders',
        durable=True,
        arguments={
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'dlx.rejected.orders',
            'x-message-ttl': 60000
        }
    )

    connection.close()
```

## Оптимизация паттернов

### Производительность wildcards

```
Производительность (от быстрой к медленной):
┌────────────────────────────────────────────────────┐
│  1. Точное совпадение:  "order.created"       ★★★★★│
│  2. Один * в конце:     "order.*"             ★★★★ │
│  3. Несколько *:        "*.order.*"           ★★★  │
│  4. # в конце:          "order.#"             ★★   │
│  5. # в середине:       "order.#.created"     ★    │
│  6. Только #:           "#"                   ★    │
└────────────────────────────────────────────────────┘
```

### Рекомендации по оптимизации

```python
# ХОРОШО: Специфичные паттерны
OPTIMIZED_PATTERNS = [
    'order.created',           # Точное совпадение
    'order.*',                 # Один wildcard в конце
    'user.profile.*',          # Специфичный путь
]

# ПЛОХО: Слишком общие паттерны
SLOW_PATTERNS = [
    '#',                       # Всё подряд
    '*.*.*.created',           # Много wildcards
    '#.created.#',             # # в середине
]
```

## Тестирование паттернов

```python
def test_routing_pattern(pattern: str, routing_keys: list) -> dict:
    """Тестирование паттерна против списка routing keys."""

    import re

    # Преобразуем RabbitMQ паттерн в regex
    regex_pattern = pattern
    regex_pattern = regex_pattern.replace('.', r'\.')
    regex_pattern = regex_pattern.replace('*', r'[^.]+')
    regex_pattern = regex_pattern.replace('#', r'.*')
    regex_pattern = f'^{regex_pattern}$'

    regex = re.compile(regex_pattern)

    results = {}
    for key in routing_keys:
        results[key] = bool(regex.match(key))

    return results

# Тестирование
pattern = "order.*.created"
keys = [
    "order.new.created",
    "order.created",
    "order.item.created.success",
    "user.new.created"
]

results = test_routing_pattern(pattern, keys)
for key, matches in results.items():
    status = "✓" if matches else "✗"
    print(f"{status} '{key}'")
```

## Best Practices

1. **Используйте специфичные паттерны** когда возможно
2. **Избегайте `#` в середине** routing key
3. **Документируйте все паттерны** в проекте
4. **Тестируйте паттерны** перед деплоем
5. **Мониторьте routing** через management plugin
6. **Используйте иерархию** в routing keys для гибкости

## Резюме

- `*` заменяет ровно одно слово
- `#` заменяет ноль или более слов
- Topic exchange обеспечивает гибкую маршрутизацию
- Производительность зависит от специфичности паттерна
- Используйте иерархические routing keys для сложных сценариев
