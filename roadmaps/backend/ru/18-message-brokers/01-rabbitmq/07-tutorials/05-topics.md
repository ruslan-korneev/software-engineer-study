# Topics

## Введение

В предыдущем туториале мы использовали direct exchange для выборочной маршрутизации. Но что если нам нужна более гибкая фильтрация? **Topic exchange** позволяет использовать шаблоны (patterns) в routing keys.

```
                                          ┌─────────┐
                 routing_key              │ Queue 1 │  binding: *.orange.*
                 "quick.orange.rabbit"    └────┬────┘
                         │                     │
┌──────────────┐         ▼                     │
│   Producer   │──>┌───────────┐               │
└──────────────┘   │  topic    │───────────────┤
                   │ exchange  │               │
                   └───────────┘               │    ┌─────────┐
                         │                     └───>│ Queue 2 │  binding: *.*.rabbit
                         │                          └─────────┘  binding: lazy.#
                         ▼
                   "lazy.brown.fox"
```

## Синтаксис Topic Exchange

### Формат Routing Key

Routing key в topic exchange — это список слов, разделённых точками:
- `stock.usd.nyse`
- `kern.critical`
- `quick.orange.rabbit`

Максимальная длина: 255 байт.

### Wildcards (Подстановочные символы)

| Символ | Значение | Пример |
|--------|----------|--------|
| `*` | Ровно одно слово | `*.orange.*` совпадает с `a.orange.b`, но НЕ с `a.b.orange.c` |
| `#` | Ноль или более слов | `lazy.#` совпадает с `lazy`, `lazy.a`, `lazy.a.b.c` |

### Примеры совпадений

| Binding Key | Routing Key | Совпадение? |
|-------------|-------------|-------------|
| `*.orange.*` | `quick.orange.rabbit` | Да |
| `*.orange.*` | `lazy.orange.elephant` | Да |
| `*.orange.*` | `quick.orange.male.rabbit` | Нет (4 слова) |
| `lazy.#` | `lazy.brown.fox` | Да |
| `lazy.#` | `lazy` | Да |
| `lazy.#` | `lazy.pink.rabbit` | Да |
| `#` | любой ключ | Да (как fanout) |
| `*` | `orange` | Да |
| `*` | `quick.orange` | Нет |

## Пример: Система логирования с иерархией

Создадим систему, где routing key имеет формат: `<facility>.<severity>`

Например: `kern.critical`, `auth.info`, `cron.warning`

## Шаг 1: Создание Producer (emit_log_topic.py)

```python
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost')
)
channel = connection.channel()

# Объявляем topic exchange
channel.exchange_declare(
    exchange='topic_logs',
    exchange_type='topic'
)

# Формат: <facility>.<severity>
# Например: kern.critical, auth.info
routing_key = sys.argv[1] if len(sys.argv) > 1 else 'anonymous.info'
message = ' '.join(sys.argv[2:]) or 'Hello World!'

channel.basic_publish(
    exchange='topic_logs',
    routing_key=routing_key,
    body=message
)

print(f" [x] Sent '{routing_key}':'{message}'")
connection.close()
```

## Шаг 2: Создание Subscriber (receive_logs_topic.py)

```python
#!/usr/bin/env python
import pika
import sys
import os

def main():
    if len(sys.argv) < 2:
        print("Usage: python receive_logs_topic.py <binding_key> [binding_key2] ...")
        print("Examples:")
        print("  python receive_logs_topic.py '#'                 # все сообщения")
        print("  python receive_logs_topic.py 'kern.*'            # все kernel логи")
        print("  python receive_logs_topic.py '*.critical'        # все critical")
        print("  python receive_logs_topic.py 'kern.*' '*.critical'")
        sys.exit(1)

    binding_keys = sys.argv[1:]

    connection = pika.BlockingConnection(
        pika.ConnectionParameters(host='localhost')
    )
    channel = connection.channel()

    channel.exchange_declare(
        exchange='topic_logs',
        exchange_type='topic'
    )

    result = channel.queue_declare(queue='', exclusive=True)
    queue_name = result.method.queue

    # Привязываем очередь к каждому binding key
    for binding_key in binding_keys:
        channel.queue_bind(
            exchange='topic_logs',
            queue=queue_name,
            routing_key=binding_key
        )
        print(f" [*] Subscribed to pattern: '{binding_key}'")

    def callback(ch, method, properties, body):
        print(f" [x] {method.routing_key}: {body.decode()}")

    channel.basic_consume(
        queue=queue_name,
        on_message_callback=callback,
        auto_ack=True
    )

    print(' [*] Waiting for logs. To exit press CTRL+C')
    channel.start_consuming()

if __name__ == '__main__':
    try:
        main()
    except KeyboardInterrupt:
        print('Interrupted')
        try:
            sys.exit(0)
        except SystemExit:
            os._exit(0)
```

## Шаг 3: Запуск примера

### Терминал 1 — все сообщения

```bash
python receive_logs_topic.py "#"
# [*] Subscribed to pattern: '#'
# [*] Waiting for logs. To exit press CTRL+C
```

### Терминал 2 — только kernel логи

```bash
python receive_logs_topic.py "kern.*"
# [*] Subscribed to pattern: 'kern.*'
# [*] Waiting for logs. To exit press CTRL+C
```

### Терминал 3 — только critical логи

```bash
python receive_logs_topic.py "*.critical"
# [*] Subscribed to pattern: '*.critical'
# [*] Waiting for logs. To exit press CTRL+C
```

### Терминал 4 — отправляем логи

```bash
python emit_log_topic.py "kern.critical" "A critical kernel error"
python emit_log_topic.py "kern.info" "Kernel info message"
python emit_log_topic.py "auth.critical" "Authentication failed"
python emit_log_topic.py "cron.warning" "Cron job warning"
```

### Результат

**Терминал 1 (#) — все:**
```
[x] kern.critical: A critical kernel error
[x] kern.info: Kernel info message
[x] auth.critical: Authentication failed
[x] cron.warning: Cron job warning
```

**Терминал 2 (kern.*) — только kernel:**
```
[x] kern.critical: A critical kernel error
[x] kern.info: Kernel info message
```

**Терминал 3 (*.critical) — только critical:**
```
[x] kern.critical: A critical kernel error
[x] auth.critical: Authentication failed
```

## Практический пример: Мониторинг микросервисов

### Формат routing key: `<service>.<env>.<level>`

Например: `payment.prod.error`, `auth.staging.info`

### Producer мониторинга (emit_monitoring.py)

```python
#!/usr/bin/env python
import pika
import sys
import json
from datetime import datetime

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost')
)
channel = connection.channel()

channel.exchange_declare(exchange='monitoring', exchange_type='topic')

# Формат: <service>.<env>.<level>
service = sys.argv[1] if len(sys.argv) > 1 else 'unknown'
env = sys.argv[2] if len(sys.argv) > 2 else 'dev'
level = sys.argv[3] if len(sys.argv) > 3 else 'info'
message = ' '.join(sys.argv[4:]) or 'No message'

routing_key = f"{service}.{env}.{level}"

payload = {
    'service': service,
    'environment': env,
    'level': level,
    'message': message,
    'timestamp': datetime.now().isoformat()
}

channel.basic_publish(
    exchange='monitoring',
    routing_key=routing_key,
    body=json.dumps(payload)
)

print(f" [x] Sent {routing_key}: {message}")
connection.close()
```

### Consumer для production ошибок (receive_prod_errors.py)

```python
#!/usr/bin/env python
import pika
import json
import sys
import os

def main():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(host='localhost')
    )
    channel = connection.channel()

    channel.exchange_declare(exchange='monitoring', exchange_type='topic')

    result = channel.queue_declare(queue='', exclusive=True)
    queue_name = result.method.queue

    # Подписываемся на все production ошибки
    # *.prod.error - любой сервис, prod окружение, error уровень
    channel.queue_bind(
        exchange='monitoring',
        queue=queue_name,
        routing_key='*.prod.error'
    )

    # Также подписываемся на критические ошибки
    channel.queue_bind(
        exchange='monitoring',
        queue=queue_name,
        routing_key='*.prod.critical'
    )

    def callback(ch, method, properties, body):
        data = json.loads(body)
        print("=" * 60)
        print(f"PRODUCTION {data['level'].upper()}!")
        print(f"Service: {data['service']}")
        print(f"Time: {data['timestamp']}")
        print(f"Message: {data['message']}")
        print("=" * 60)

    channel.basic_consume(
        queue=queue_name,
        on_message_callback=callback,
        auto_ack=True
    )

    print(' [*] Waiting for PRODUCTION errors. To exit press CTRL+C')
    channel.start_consuming()

if __name__ == '__main__':
    try:
        main()
    except KeyboardInterrupt:
        print('Interrupted')
        sys.exit(0)
```

### Consumer для конкретного сервиса (receive_service_logs.py)

```python
#!/usr/bin/env python
import pika
import json
import sys
import os

def main():
    if len(sys.argv) < 2:
        print("Usage: python receive_service_logs.py <service_name>")
        print("Example: python receive_service_logs.py payment")
        sys.exit(1)

    service = sys.argv[1]

    connection = pika.BlockingConnection(
        pika.ConnectionParameters(host='localhost')
    )
    channel = connection.channel()

    channel.exchange_declare(exchange='monitoring', exchange_type='topic')

    result = channel.queue_declare(queue='', exclusive=True)
    queue_name = result.method.queue

    # Подписываемся на ВСЕ логи конкретного сервиса
    # payment.# - все сообщения от payment сервиса
    binding_key = f"{service}.#"
    channel.queue_bind(
        exchange='monitoring',
        queue=queue_name,
        routing_key=binding_key
    )

    print(f" [*] Subscribed to all '{service}' logs ({binding_key})")

    def callback(ch, method, properties, body):
        data = json.loads(body)

        # Цветной вывод
        colors = {
            'info': '\033[94m',
            'warning': '\033[93m',
            'error': '\033[91m',
            'critical': '\033[95m'
        }
        reset = '\033[0m'

        color = colors.get(data['level'], '')
        print(f"{color}[{data['environment']}][{data['level'].upper()}]{reset} "
              f"{data['timestamp']}: {data['message']}")

    channel.basic_consume(
        queue=queue_name,
        on_message_callback=callback,
        auto_ack=True
    )

    print(' [*] Waiting for logs. To exit press CTRL+C')
    channel.start_consuming()

if __name__ == '__main__':
    try:
        main()
    except KeyboardInterrupt:
        print('Interrupted')
        sys.exit(0)
```

### Тестирование

```bash
# Терминал 1: все production ошибки
python receive_prod_errors.py

# Терминал 2: все логи payment сервиса
python receive_service_logs.py payment

# Терминал 3: отправляем тестовые сообщения
python emit_monitoring.py payment prod error "Payment failed for order 12345"
python emit_monitoring.py payment dev info "Testing payment flow"
python emit_monitoring.py auth prod error "Invalid token"
python emit_monitoring.py payment prod info "Order processed"
```

## Особые случаи Topic Exchange

### Topic как Fanout

Если binding key = `#`, очередь получит ВСЕ сообщения (как fanout):

```python
channel.queue_bind(exchange='topic_logs', queue=q, routing_key='#')
```

### Topic как Direct

Если binding key не содержит `*` или `#`, topic работает как direct:

```python
# Точное совпадение с "kern.error"
channel.queue_bind(exchange='topic_logs', queue=q, routing_key='kern.error')
```

## Сравнение типов Exchange

| Тип | Routing | Пример Binding |
|-----|---------|----------------|
| fanout | Все очереди | (не нужен) |
| direct | Точное совпадение | `error` |
| topic | Pattern matching | `*.error`, `kern.#` |

## Правила Pattern Matching

### Правило 1: `*` заменяет ровно одно слово

```
Binding: stock.*.nyse
Matches: stock.usd.nyse, stock.eur.nyse
Not matches: stock.nyse, stock.usd.eur.nyse
```

### Правило 2: `#` заменяет ноль или более слов

```
Binding: stock.#
Matches: stock, stock.usd, stock.usd.nyse, stock.a.b.c.d
```

### Правило 3: Комбинации

```
Binding: *.stock.#
Matches: usd.stock, usd.stock.nyse, usd.stock.nyse.2024
Not matches: stock, stock.nyse
```

## CLI команды

```bash
# Список exchanges
rabbitmqctl list_exchanges

# Список bindings
rabbitmqctl list_bindings source_name destination_name routing_key

# Тестовая публикация через CLI
rabbitmqadmin publish exchange=topic_logs routing_key=kern.error payload="Test"
```

## Частые ошибки

### Неправильный формат routing key

```
# Неправильно (пробелы)
routing_key = "kern .error"

# Правильно
routing_key = "kern.error"
```

### Путаница между * и #

```
# *.*.rabbit НЕ совпадёт с lazy.rabbit (нужно ровно 3 слова)
# Используйте #.rabbit для "любое количество слов, заканчивающееся на rabbit"
```

## Резюме

В этом туториале мы изучили:
1. **Topic exchange** — маршрутизация по шаблонам
2. **Wildcards**: `*` (одно слово), `#` (ноль или более)
3. **Иерархические routing keys** — для сложной категоризации
4. **Гибкая подписка** — подписка на группы сообщений

Следующий туториал: **RPC** — реализация запрос-ответ паттерна через очереди.
