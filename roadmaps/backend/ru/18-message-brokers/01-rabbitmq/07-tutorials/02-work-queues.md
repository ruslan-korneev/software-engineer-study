# Work Queues

[prev: 01-hello-world](./01-hello-world.md) | [next: 03-publish-subscribe](./03-publish-subscribe.md)

---

## Введение

В этом туториале мы создадим **Work Queue** (рабочую очередь), которая распределяет ресурсоёмкие задачи между несколькими workers. Это основная идея паттерна **Task Queue**.

```
                              ┌──────────────┐
                         ┌───>│   Worker 1   │
┌──────────────┐         │    └──────────────┘
│   Producer   │────>[ Queue ]
│  (new_task)  │         │    ┌──────────────┐
└──────────────┘         └───>│   Worker 2   │
                              └──────────────┘
```

## Концепция Work Queues

Work Queues используются для:
- Распределения ресурсоёмких задач между workers
- Отложенной обработки (задача ставится в очередь и выполняется позже)
- Балансировки нагрузки между несколькими обработчиками

## Шаг 1: Создание Producer (new_task.py)

Producer отправляет задачи в очередь. Количество точек в сообщении определяет "сложность" задачи.

```python
#!/usr/bin/env python
import pika
import sys

# Устанавливаем соединение
connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost')
)
channel = connection.channel()

# Объявляем durable очередь
# durable=True означает, что очередь переживёт перезапуск RabbitMQ
channel.queue_declare(queue='task_queue', durable=True)

# Получаем сообщение из аргументов командной строки
# или используем значение по умолчанию
message = ' '.join(sys.argv[1:]) or "Hello World!"

# Публикуем сообщение с persistent delivery mode
channel.basic_publish(
    exchange='',
    routing_key='task_queue',
    body=message,
    properties=pika.BasicProperties(
        delivery_mode=pika.DeliveryMode.Persistent  # Сообщение сохраняется на диск
    )
)

print(f" [x] Sent '{message}'")
connection.close()
```

### Объяснение ключевых моментов

1. **durable=True**: Очередь сохраняется при перезапуске RabbitMQ
2. **delivery_mode=Persistent**: Сообщения записываются на диск
3. **sys.argv[1:]**: Позволяет передавать сообщение через командную строку

## Шаг 2: Создание Worker (worker.py)

Worker получает задачи и "обрабатывает" их. Каждая точка в сообщении означает 1 секунду работы.

```python
#!/usr/bin/env python
import pika
import time
import sys
import os

def main():
    # Устанавливаем соединение
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(host='localhost')
    )
    channel = connection.channel()

    # Объявляем очередь (должна совпадать с producer)
    channel.queue_declare(queue='task_queue', durable=True)

    # Callback-функция для обработки сообщений
    def callback(ch, method, properties, body):
        message = body.decode()
        print(f" [x] Received '{message}'")

        # Симулируем тяжёлую работу
        # Количество точек = количество секунд работы
        time.sleep(message.count('.'))

        print(" [x] Done")

        # ВАЖНО: Отправляем подтверждение (ack) ПОСЛЕ обработки
        ch.basic_ack(delivery_tag=method.delivery_tag)

    # Устанавливаем prefetch_count=1
    # Это означает: "не давай мне больше одной задачи за раз"
    channel.basic_qos(prefetch_count=1)

    # Подписываемся на очередь БЕЗ auto_ack
    # Теперь мы должны отправлять ack вручную
    channel.basic_consume(
        queue='task_queue',
        on_message_callback=callback,
        auto_ack=False  # Ручное подтверждение!
    )

    print(' [*] Waiting for messages. To exit press CTRL+C')
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

### Объяснение ключевых моментов

1. **auto_ack=False**: Отключаем автоподтверждение
2. **basic_ack()**: Отправляем подтверждение вручную после успешной обработки
3. **basic_qos(prefetch_count=1)**: Worker получает только одну задачу за раз

## Шаг 3: Запуск примера

### Терминал 1 — запускаем первый Worker

```bash
python worker.py
# [*] Waiting for messages. To exit press CTRL+C
```

### Терминал 2 — запускаем второй Worker

```bash
python worker.py
# [*] Waiting for messages. To exit press CTRL+C
```

### Терминал 3 — отправляем задачи

```bash
python new_task.py "First task."
python new_task.py "Second task.."
python new_task.py "Third task..."
python new_task.py "Fourth task...."
python new_task.py "Fifth task....."
```

### Результат

Задачи распределяются между workers:

**Worker 1:**
```
[x] Received 'First task.'
[x] Done
[x] Received 'Third task...'
[x] Done
[x] Received 'Fifth task.....'
[x] Done
```

**Worker 2:**
```
[x] Received 'Second task..'
[x] Done
[x] Received 'Fourth task....'
[x] Done
```

## Механизм подтверждений (Acknowledgements)

### Проблема без подтверждений

Если worker упадёт во время обработки сообщения, оно будет потеряно навсегда.

### Решение: Manual Acknowledgement

```python
def callback(ch, method, properties, body):
    # Обрабатываем сообщение
    process_message(body)

    # Отправляем подтверждение ПОСЛЕ успешной обработки
    ch.basic_ack(delivery_tag=method.delivery_tag)
```

### Что происходит при сбое worker

1. Worker получает сообщение
2. Worker падает ДО отправки ack
3. RabbitMQ понимает, что ack не получен
4. Сообщение возвращается в очередь
5. Другой worker получает это сообщение

### Проверка неподтверждённых сообщений

```bash
rabbitmqctl list_queues name messages_ready messages_unacknowledged
```

## Fair Dispatch (Честное распределение)

### Проблема Round-Robin

По умолчанию RabbitMQ использует round-robin: каждый worker получает одинаковое количество сообщений. Но если одни задачи тяжёлые, а другие лёгкие, один worker может быть перегружен.

### Решение: prefetch_count

```python
# Говорим RabbitMQ: "Не давай мне новую задачу,
# пока я не закончу текущую"
channel.basic_qos(prefetch_count=1)
```

Теперь RabbitMQ отправляет задачу свободному worker'у, а не просто по очереди.

## Message Durability (Долговечность сообщений)

### Проблема

При перезапуске RabbitMQ очереди и сообщения могут быть потеряны.

### Решение

1. **Durable Queue**: Очередь сохраняется при перезапуске

```python
channel.queue_declare(queue='task_queue', durable=True)
```

2. **Persistent Messages**: Сообщения записываются на диск

```python
channel.basic_publish(
    exchange='',
    routing_key='task_queue',
    body=message,
    properties=pika.BasicProperties(
        delivery_mode=pika.DeliveryMode.Persistent
    )
)
```

### Важное замечание

Persistent messages не дают 100% гарантии. Есть короткое окно, когда сообщение принято, но ещё не записано на диск. Для полной надёжности используйте **Publisher Confirms** (туториал 7).

## Полный пример с улучшениями

### Улучшенный Worker с обработкой ошибок

```python
#!/usr/bin/env python
import pika
import time
import sys
import os
import traceback

def process_task(body):
    """Обработка задачи. Может выбросить исключение."""
    message = body.decode()
    print(f" [x] Processing '{message}'")

    # Симуляция работы
    time.sleep(message.count('.'))

    # Симуляция возможной ошибки
    if 'error' in message.lower():
        raise ValueError("Simulated error in task")

    return True

def main():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(host='localhost')
    )
    channel = connection.channel()
    channel.queue_declare(queue='task_queue', durable=True)

    def callback(ch, method, properties, body):
        try:
            process_task(body)
            print(" [x] Done")
            # Подтверждаем успешную обработку
            ch.basic_ack(delivery_tag=method.delivery_tag)
        except Exception as e:
            print(f" [!] Error: {e}")
            traceback.print_exc()
            # При ошибке можно:
            # 1. Отклонить и вернуть в очередь (requeue=True)
            # 2. Отклонить и удалить (requeue=False)
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)

    channel.basic_qos(prefetch_count=1)
    channel.basic_consume(queue='task_queue', on_message_callback=callback)

    print(' [*] Waiting for messages. To exit press CTRL+C')
    channel.start_consuming()

if __name__ == '__main__':
    try:
        main()
    except KeyboardInterrupt:
        print('Interrupted')
        sys.exit(0)
```

## Сравнение режимов подтверждения

| Режим | Преимущества | Недостатки |
|-------|-------------|------------|
| auto_ack=True | Простота, высокая производительность | Потеря сообщений при сбое |
| auto_ack=False | Надёжность, повторная обработка при сбое | Ниже производительность |

## Частые ошибки

### Забыли отправить ack

Сообщения накапливаются как unacknowledged и не удаляются:

```bash
rabbitmqctl list_queues name messages_ready messages_unacknowledged
# Видим растущее число unacknowledged
```

### Разные параметры очереди

```
pika.exceptions.ChannelClosedByBroker: (406, "PRECONDITION_FAILED -
inequivalent arg 'durable' for queue 'task_queue'")
```

**Решение**: Удалите очередь и создайте заново или используйте те же параметры.

## Резюме

В этом туториале мы изучили:
1. **Work Queues** — распределение задач между workers
2. **Message Acknowledgement** — подтверждение обработки
3. **Fair Dispatch** — честное распределение через prefetch_count
4. **Message Durability** — сохранение сообщений на диск

Следующий туториал: **Publish/Subscribe** — отправка сообщений нескольким получателям одновременно.

---

[prev: 01-hello-world](./01-hello-world.md) | [next: 03-publish-subscribe](./03-publish-subscribe.md)
