# Publish/Subscribe

[prev: 02-work-queues](./02-work-queues.md) | [next: 04-routing](./04-routing.md)

---

## Введение

В предыдущих туториалах мы отправляли сообщения в конкретную очередь. Теперь мы изучим паттерн **Publish/Subscribe** — отправка одного сообщения нескольким получателям одновременно.

```
                                    ┌─────────┐    ┌──────────────┐
                               ┌───>│ Queue 1 │───>│ Subscriber 1 │
┌──────────────┐   ┌────────┐  │    └─────────┘    └──────────────┘
│   Producer   │──>│ fanout │──┤
│ (emit_log)   │   │exchange│  │    ┌─────────┐    ┌──────────────┐
└──────────────┘   └────────┘  └───>│ Queue 2 │───>│ Subscriber 2 │
                                    └─────────┘    └──────────────┘
```

## Концепция Exchanges

### Что такое Exchange?

**Exchange** — это маршрутизатор сообщений. Producer никогда не отправляет сообщения напрямую в очередь. Вместо этого он отправляет их в exchange, который решает, куда направить сообщение.

### Типы Exchanges

| Тип | Описание |
|-----|----------|
| **direct** | Точное совпадение routing key |
| **topic** | Шаблонное совпадение routing key |
| **fanout** | Отправка во все привязанные очереди |
| **headers** | Маршрутизация по заголовкам |

В этом туториале мы используем **fanout** — самый простой тип, который broadcast'ит сообщения во все очереди.

## Пример: Система логирования

Создадим систему, где один producer отправляет логи, а несколько consumers их получают.

## Шаг 1: Создание Producer (emit_log.py)

```python
#!/usr/bin/env python
import pika
import sys

# Устанавливаем соединение
connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost')
)
channel = connection.channel()

# Объявляем fanout exchange
# fanout отправляет сообщения во ВСЕ привязанные очереди
channel.exchange_declare(
    exchange='logs',
    exchange_type='fanout'
)

# Получаем сообщение из аргументов
message = ' '.join(sys.argv[1:]) or "info: Hello World!"

# Публикуем в exchange (не в очередь!)
# routing_key игнорируется для fanout
channel.basic_publish(
    exchange='logs',       # Имя exchange
    routing_key='',        # Для fanout не важен
    body=message
)

print(f" [x] Sent '{message}'")
connection.close()
```

### Объяснение кода

1. **exchange_declare()**: Создаём exchange с типом `fanout`
2. **exchange='logs'**: Указываем имя exchange вместо пустой строки
3. **routing_key=''**: Для fanout routing key игнорируется

## Шаг 2: Создание Subscriber (receive_logs.py)

```python
#!/usr/bin/env python
import pika
import sys
import os

def main():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(host='localhost')
    )
    channel = connection.channel()

    # Объявляем тот же exchange
    channel.exchange_declare(
        exchange='logs',
        exchange_type='fanout'
    )

    # Создаём временную очередь с уникальным именем
    # exclusive=True означает:
    # 1. Очередь удаляется при закрытии соединения
    # 2. Только это соединение может использовать очередь
    result = channel.queue_declare(
        queue='',           # Пустое имя = RabbitMQ генерирует уникальное
        exclusive=True      # Эксклюзивная временная очередь
    )

    # Получаем сгенерированное имя очереди
    queue_name = result.method.queue
    print(f" [*] Queue name: {queue_name}")

    # Привязываем очередь к exchange (binding)
    # Теперь exchange будет отправлять сообщения в эту очередь
    channel.queue_bind(
        exchange='logs',
        queue=queue_name
    )

    def callback(ch, method, properties, body):
        print(f" [x] {body.decode()}")

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

### Объяснение кода

1. **queue=''**: RabbitMQ генерирует уникальное имя (например, `amq.gen-JzTY20BRgKO-HjmUJj0wLg`)
2. **exclusive=True**: Очередь удаляется при отключении consumer
3. **queue_bind()**: Создаёт связь между exchange и очередью

## Шаг 3: Запуск примера

### Терминал 1 — запускаем первый Subscriber

```bash
python receive_logs.py
# [*] Queue name: amq.gen-JzTY20BRgKO-HjmUJj0wLg
# [*] Waiting for logs. To exit press CTRL+C
```

### Терминал 2 — запускаем второй Subscriber

```bash
python receive_logs.py
# [*] Queue name: amq.gen-DIFFERENT-RANDOM-NAME
# [*] Waiting for logs. To exit press CTRL+C
```

### Терминал 3 — отправляем логи

```bash
python emit_log.py "First log message"
python emit_log.py "Second log message"
python emit_log.py "Third log message"
```

### Результат

Оба subscriber'а получат ВСЕ сообщения:

**Subscriber 1:**
```
[x] First log message
[x] Second log message
[x] Third log message
```

**Subscriber 2:**
```
[x] First log message
[x] Second log message
[x] Third log message
```

## Bindings (Привязки)

### Что такое Binding?

**Binding** — это связь между exchange и очередью. Оно говорит exchange: "отправляй сообщения в эту очередь".

```python
channel.queue_bind(
    exchange='logs',
    queue=queue_name,
    routing_key=''  # Для fanout не используется
)
```

### Просмотр bindings

```bash
rabbitmqctl list_bindings
```

## Temporary Queues (Временные очереди)

### Зачем нужны?

Для паттерна pub/sub важно:
1. Каждый subscriber должен иметь свою очередь
2. Очередь должна удаляться при отключении subscriber'а
3. Нам не важно имя очереди

### Как создать

```python
result = channel.queue_declare(queue='', exclusive=True)
queue_name = result.method.queue  # amq.gen-xxxxx
```

### Параметры временной очереди

| Параметр | Значение | Эффект |
|----------|----------|--------|
| queue='' | пустая строка | RabbitMQ генерирует имя |
| exclusive=True | True | Только одно соединение, удаляется при отключении |

## Практический пример: Логирование в файл и на экран

### Subscriber, сохраняющий логи в файл (receive_logs_to_file.py)

```python
#!/usr/bin/env python
import pika
import sys
import os
from datetime import datetime

def main():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(host='localhost')
    )
    channel = connection.channel()

    channel.exchange_declare(exchange='logs', exchange_type='fanout')

    result = channel.queue_declare(queue='', exclusive=True)
    queue_name = result.method.queue

    channel.queue_bind(exchange='logs', queue=queue_name)

    # Открываем файл для записи логов
    log_file = open('rabbitmq_logs.txt', 'a')

    def callback(ch, method, properties, body):
        timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        log_entry = f"[{timestamp}] {body.decode()}\n"

        # Записываем в файл
        log_file.write(log_entry)
        log_file.flush()

        print(f" [x] Saved to file: {body.decode()}")

    channel.basic_consume(
        queue=queue_name,
        on_message_callback=callback,
        auto_ack=True
    )

    print(' [*] Saving logs to file. To exit press CTRL+C')
    try:
        channel.start_consuming()
    finally:
        log_file.close()

if __name__ == '__main__':
    try:
        main()
    except KeyboardInterrupt:
        print('Interrupted')
        sys.exit(0)
```

### Subscriber, выводящий логи на экран (receive_logs_to_screen.py)

```python
#!/usr/bin/env python
import pika
import sys
import os

def main():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(host='localhost')
    )
    channel = connection.channel()

    channel.exchange_declare(exchange='logs', exchange_type='fanout')

    result = channel.queue_declare(queue='', exclusive=True)
    queue_name = result.method.queue

    channel.queue_bind(exchange='logs', queue=queue_name)

    def callback(ch, method, properties, body):
        # Красивый вывод на экран
        message = body.decode()
        if 'error' in message.lower():
            print(f" [ERROR] {message}")
        elif 'warning' in message.lower():
            print(f" [WARN]  {message}")
        else:
            print(f" [INFO]  {message}")

    channel.basic_consume(
        queue=queue_name,
        on_message_callback=callback,
        auto_ack=True
    )

    print(' [*] Displaying logs on screen. To exit press CTRL+C')
    channel.start_consuming()

if __name__ == '__main__':
    try:
        main()
    except KeyboardInterrupt:
        print('Interrupted')
        sys.exit(0)
```

## Проверка через CLI

### Список exchanges

```bash
rabbitmqctl list_exchanges
```

### Список bindings

```bash
rabbitmqctl list_bindings
```

### Список очередей

```bash
rabbitmqctl list_queues
```

## Сравнение с Work Queues

| Аспект | Work Queues | Publish/Subscribe |
|--------|-------------|-------------------|
| Exchange | Default ('') | Fanout |
| Очереди | Одна общая | Своя для каждого subscriber |
| Доставка | Одному из workers | Всем subscribers |
| Удаление очереди | Вручную | Автоматически (exclusive) |

## Важные концепции

### Nameless Exchange vs Named Exchange

```python
# До (Hello World, Work Queues):
channel.basic_publish(exchange='', routing_key='queue_name', body=msg)

# Теперь (Publish/Subscribe):
channel.basic_publish(exchange='logs', routing_key='', body=msg)
```

### Порядок объявления

1. Сначала объявляем exchange
2. Затем объявляем очередь
3. Затем связываем их (bind)

## Частые ошибки

### Exchange не существует

```
pika.exceptions.ChannelClosedByBroker: (404, "NOT_FOUND - no exchange 'logs'")
```

**Решение**: Объявите exchange перед использованием.

### Сообщения теряются

Если отправить сообщение до того, как subscriber создаст очередь и привяжет её — сообщение будет потеряно. Fanout exchange не хранит сообщения, если нет привязанных очередей.

**Решение**: Запускайте subscribers ДО producer'а.

## Резюме

В этом туториале мы изучили:
1. **Exchanges** — маршрутизаторы сообщений
2. **Fanout exchange** — broadcast во все привязанные очереди
3. **Temporary queues** — временные эксклюзивные очереди
4. **Bindings** — связи между exchange и очередями

Следующий туториал: **Routing** — выборочное получение сообщений с помощью direct exchange.

---

[prev: 02-work-queues](./02-work-queues.md) | [next: 04-routing](./04-routing.md)
