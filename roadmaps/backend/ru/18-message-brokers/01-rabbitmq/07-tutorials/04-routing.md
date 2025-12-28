# Routing

[prev: 03-publish-subscribe](./03-publish-subscribe.md) | [next: 05-topics](./05-topics.md)

---

## Введение

В предыдущем туториале мы использовали fanout exchange, который отправляет сообщения во все очереди. Теперь мы научимся **выборочно получать сообщения** с помощью **direct exchange**.

```
                              routing_key="error"
                            ┌───────────────────────>┌─────────┐    ┌──────────────┐
┌──────────────┐   ┌──────┐ │                        │ Queue 1 │───>│ Subscriber 1 │
│   Producer   │──>│direct│─┤                        └─────────┘    │ (all logs)   │
│  (emit_log)  │   │  X   │ │  routing_key="info"                   └──────────────┘
└──────────────┘   └──────┘ │  routing_key="warning"
                            │  routing_key="error"   ┌─────────┐    ┌──────────────┐
                            └───────────────────────>│ Queue 2 │───>│ Subscriber 2 │
                                                     └─────────┘    │ (errors only)│
                                                                    └──────────────┘
```

## Концепция Direct Exchange

### Как работает Direct Exchange

**Direct exchange** направляет сообщения в очереди по точному совпадению **routing key**:
1. Producer отправляет сообщение с определённым routing key
2. Exchange сравнивает routing key с binding key каждой очереди
3. Сообщение попадает только в очереди с совпадающим binding key

### Binding Key

При привязке очереди к exchange можно указать **binding key**:

```python
channel.queue_bind(
    exchange='direct_logs',
    queue=queue_name,
    routing_key='error'  # Это binding key
)
```

## Пример: Система логирования с фильтрацией

Создадим систему, где можно подписаться только на определённые уровни логов.

## Шаг 1: Создание Producer (emit_log_direct.py)

```python
#!/usr/bin/env python
import pika
import sys

# Устанавливаем соединение
connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost')
)
channel = connection.channel()

# Объявляем direct exchange
channel.exchange_declare(
    exchange='direct_logs',
    exchange_type='direct'
)

# Получаем severity из аргументов (info, warning, error)
# По умолчанию - info
severity = sys.argv[1] if len(sys.argv) > 1 else 'info'

# Получаем сообщение
message = ' '.join(sys.argv[2:]) or 'Hello World!'

# Публикуем с routing_key = severity
channel.basic_publish(
    exchange='direct_logs',
    routing_key=severity,    # routing key определяет, куда пойдёт сообщение
    body=message
)

print(f" [x] Sent '{severity}':'{message}'")
connection.close()
```

### Объяснение

1. **exchange_type='direct'**: Используем direct exchange
2. **routing_key=severity**: Уровень лога (info, warning, error) определяет маршрут

## Шаг 2: Создание Subscriber (receive_logs_direct.py)

```python
#!/usr/bin/env python
import pika
import sys
import os

def main():
    # Проверяем, что указаны severities
    if len(sys.argv) < 2:
        print("Usage: python receive_logs_direct.py [info] [warning] [error]")
        sys.exit(1)

    # Получаем список severities для подписки
    severities = sys.argv[1:]

    connection = pika.BlockingConnection(
        pika.ConnectionParameters(host='localhost')
    )
    channel = connection.channel()

    # Объявляем тот же direct exchange
    channel.exchange_declare(
        exchange='direct_logs',
        exchange_type='direct'
    )

    # Создаём временную эксклюзивную очередь
    result = channel.queue_declare(queue='', exclusive=True)
    queue_name = result.method.queue

    # Привязываем очередь к exchange для КАЖДОГО severity
    for severity in severities:
        channel.queue_bind(
            exchange='direct_logs',
            queue=queue_name,
            routing_key=severity  # binding key = severity
        )
        print(f" [*] Subscribed to '{severity}'")

    def callback(ch, method, properties, body):
        # method.routing_key содержит routing key сообщения
        print(f" [x] {method.routing_key}:{body.decode()}")

    channel.basic_consume(
        queue=queue_name,
        on_message_callback=callback,
        auto_ack=True
    )

    print(f' [*] Waiting for logs. To exit press CTRL+C')
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

### Объяснение

1. **sys.argv[1:]**: Получаем список severities из командной строки
2. **Цикл привязок**: Создаём отдельный binding для каждого severity
3. **method.routing_key**: Получаем routing key полученного сообщения

## Шаг 3: Запуск примера

### Терминал 1 — подписываемся на все уровни

```bash
python receive_logs_direct.py info warning error
# [*] Subscribed to 'info'
# [*] Subscribed to 'warning'
# [*] Subscribed to 'error'
# [*] Waiting for logs. To exit press CTRL+C
```

### Терминал 2 — подписываемся только на ошибки

```bash
python receive_logs_direct.py error
# [*] Subscribed to 'error'
# [*] Waiting for logs. To exit press CTRL+C
```

### Терминал 3 — отправляем логи разных уровней

```bash
python emit_log_direct.py info "This is info message"
python emit_log_direct.py warning "This is warning message"
python emit_log_direct.py error "This is error message"
```

### Результат

**Терминал 1 (все уровни):**
```
[x] info:This is info message
[x] warning:This is warning message
[x] error:This is error message
```

**Терминал 2 (только ошибки):**
```
[x] error:This is error message
```

## Multiple Bindings

### Одна очередь — несколько bindings

Одна очередь может быть привязана к exchange с несколькими binding keys:

```python
# Подписываемся на info И warning
channel.queue_bind(exchange='direct_logs', queue=q, routing_key='info')
channel.queue_bind(exchange='direct_logs', queue=q, routing_key='warning')
```

### Несколько очередей — один binding key

Несколько очередей могут иметь одинаковый binding key. В этом случае direct exchange работает как fanout для этого ключа:

```
Queue1 ─┬─ binding_key="error" ─┐
        │                       │
        └───────────────────────┼─> Exchange
                                │
Queue2 ─── binding_key="error" ─┘
```

Обе очереди получат сообщения с routing_key="error".

## Практический пример: Система оповещений

### Producer оповещений (emit_alert.py)

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

channel.exchange_declare(exchange='alerts', exchange_type='direct')

# Уровни: critical, warning, info
level = sys.argv[1] if len(sys.argv) > 1 else 'info'
message = ' '.join(sys.argv[2:]) or 'Test alert'

# Формируем JSON payload
alert = {
    'level': level,
    'message': message,
    'timestamp': datetime.now().isoformat(),
    'source': 'monitoring-system'
}

channel.basic_publish(
    exchange='alerts',
    routing_key=level,
    body=json.dumps(alert)
)

print(f" [x] Sent {level} alert: {message}")
connection.close()
```

### Consumer для критических оповещений (receive_critical_alerts.py)

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

    channel.exchange_declare(exchange='alerts', exchange_type='direct')

    result = channel.queue_declare(queue='', exclusive=True)
    queue_name = result.method.queue

    # Подписываемся ТОЛЬКО на critical
    channel.queue_bind(
        exchange='alerts',
        queue=queue_name,
        routing_key='critical'
    )

    def callback(ch, method, properties, body):
        alert = json.loads(body)
        print("=" * 50)
        print(f"CRITICAL ALERT!")
        print(f"Time: {alert['timestamp']}")
        print(f"Source: {alert['source']}")
        print(f"Message: {alert['message']}")
        print("=" * 50)

        # Здесь можно добавить отправку SMS, email и т.д.

    channel.basic_consume(
        queue=queue_name,
        on_message_callback=callback,
        auto_ack=True
    )

    print(' [*] Waiting for CRITICAL alerts. To exit press CTRL+C')
    channel.start_consuming()

if __name__ == '__main__':
    try:
        main()
    except KeyboardInterrupt:
        print('Interrupted')
        sys.exit(0)
```

### Consumer для всех оповещений (receive_all_alerts.py)

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

    channel.exchange_declare(exchange='alerts', exchange_type='direct')

    result = channel.queue_declare(queue='', exclusive=True)
    queue_name = result.method.queue

    # Подписываемся на ВСЕ уровни
    for level in ['info', 'warning', 'critical']:
        channel.queue_bind(
            exchange='alerts',
            queue=queue_name,
            routing_key=level
        )

    def callback(ch, method, properties, body):
        alert = json.loads(body)
        level = alert['level'].upper()

        # Цветной вывод в терминал
        colors = {
            'INFO': '\033[94m',      # Blue
            'WARNING': '\033[93m',   # Yellow
            'CRITICAL': '\033[91m',  # Red
        }
        reset = '\033[0m'

        color = colors.get(level, '')
        print(f"{color}[{level}]{reset} {alert['timestamp']}: {alert['message']}")

    channel.basic_consume(
        queue=queue_name,
        on_message_callback=callback,
        auto_ack=True
    )

    print(' [*] Logging all alerts. To exit press CTRL+C')
    channel.start_consuming()

if __name__ == '__main__':
    try:
        main()
    except KeyboardInterrupt:
        print('Interrupted')
        sys.exit(0)
```

## Сравнение типов Exchange

| Тип | Routing Key | Описание |
|-----|-------------|----------|
| fanout | Игнорируется | Broadcast во все очереди |
| direct | Точное совпадение | Только в очереди с matching binding key |
| topic | Pattern matching | Шаблоны с * и # (следующий туториал) |

## Диаграмма маршрутизации

```
Producer                    Exchange                    Queues
────────                    ────────                    ──────

emit("error", msg) ──┐
                     │
                     ▼
                 ┌───────┐
                 │direct │──── binding_key="error" ───> Queue1
                 │ logs  │
                 │       │──── binding_key="info" ────> Queue2
                 │       │
                 │       │──── binding_key="error" ───> Queue2
                 └───────┘

emit("info", msg) идёт ТОЛЬКО в Queue2
emit("error", msg) идёт в Queue1 И Queue2
```

## CLI команды для проверки

```bash
# Список exchanges
rabbitmqctl list_exchanges

# Список bindings с routing keys
rabbitmqctl list_bindings

# Подробная информация
rabbitmqctl list_bindings source_name destination_name routing_key
```

## Частые ошибки

### Сообщения не доходят

**Проблема**: Сообщения отправляются, но subscriber их не получает.

**Причины**:
1. Routing key не совпадает с binding key
2. Subscriber не подписан на нужный routing key

**Решение**: Проверьте bindings через CLI:
```bash
rabbitmqctl list_bindings
```

### Несоответствие типа exchange

```
pika.exceptions.ChannelClosedByBroker: (406, "PRECONDITION_FAILED -
inequivalent arg 'type' for exchange 'logs'")
```

**Решение**: Удалите exchange и создайте с правильным типом.

## Резюме

В этом туториале мы изучили:
1. **Direct exchange** — маршрутизация по точному совпадению routing key
2. **Binding key** — ключ привязки очереди к exchange
3. **Multiple bindings** — одна очередь может иметь несколько binding keys
4. **Selective receiving** — получение только нужных сообщений

Следующий туториал: **Topics** — шаблонная маршрутизация с wildcards.

---

[prev: 03-publish-subscribe](./03-publish-subscribe.md) | [next: 05-topics](./05-topics.md)
