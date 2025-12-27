# Hello World

## Введение

Это первый туториал из серии RabbitMQ. В нём мы создадим простейшее приложение: producer (отправитель) будет отправлять сообщение, а consumer (получатель) будет его принимать и выводить на экран.

```
┌──────────────┐       ┌─────────┐       ┌──────────────┐
│   Producer   │──────>│  Queue  │──────>│   Consumer   │
│  (send.py)   │       │ "hello" │       │ (receive.py) │
└──────────────┘       └─────────┘       └──────────────┘
```

## Предварительные требования

### Установка RabbitMQ

Убедитесь, что RabbitMQ установлен и запущен:

```bash
# macOS
brew install rabbitmq
brew services start rabbitmq

# Ubuntu/Debian
sudo apt-get install rabbitmq-server
sudo systemctl start rabbitmq-server

# Docker (рекомендуется для обучения)
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:management
```

### Установка библиотеки pika

```bash
pip install pika
```

## Шаг 1: Создание Producer (send.py)

Producer — это приложение, которое отправляет сообщения в очередь.

```python
#!/usr/bin/env python
import pika

# Шаг 1: Устанавливаем соединение с RabbitMQ
# По умолчанию RabbitMQ работает на localhost:5672
connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost')
)

# Шаг 2: Создаём канал
# Канал — это виртуальное соединение внутри TCP-соединения
channel = connection.channel()

# Шаг 3: Объявляем очередь
# Если очередь не существует, она будет создана
# Если существует — ничего не произойдёт
channel.queue_declare(queue='hello')

# Шаг 4: Публикуем сообщение
# exchange='' означает default exchange
# routing_key='hello' указывает, в какую очередь отправить
channel.basic_publish(
    exchange='',
    routing_key='hello',
    body='Hello World!'
)

print(" [x] Sent 'Hello World!'")

# Шаг 5: Закрываем соединение
connection.close()
```

### Объяснение кода Producer

1. **ConnectionParameters**: Параметры подключения к серверу RabbitMQ. По умолчанию используется `localhost` и порт `5672`.

2. **channel()**: Создаёт канал — легковесное соединение для выполнения операций. Один TCP-сокет может иметь множество каналов.

3. **queue_declare()**: Объявляет очередь. Операция идемпотентна — очередь создаётся только если её нет.

4. **basic_publish()**: Отправляет сообщение:
   - `exchange=''` — используем default exchange (nameless exchange)
   - `routing_key` — имя очереди, куда направляется сообщение
   - `body` — тело сообщения (байты или строка)

## Шаг 2: Создание Consumer (receive.py)

Consumer — это приложение, которое получает и обрабатывает сообщения из очереди.

```python
#!/usr/bin/env python
import pika
import sys
import os

def main():
    # Шаг 1: Устанавливаем соединение
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(host='localhost')
    )
    channel = connection.channel()

    # Шаг 2: Объявляем очередь
    # Важно объявить очередь и здесь тоже!
    # Consumer может запуститься раньше Producer
    channel.queue_declare(queue='hello')

    # Шаг 3: Определяем callback-функцию
    # Она будет вызываться при получении каждого сообщения
    def callback(ch, method, properties, body):
        print(f" [x] Received {body.decode()}")

    # Шаг 4: Подписываемся на очередь
    # auto_ack=True означает автоматическое подтверждение
    channel.basic_consume(
        queue='hello',
        on_message_callback=callback,
        auto_ack=True
    )

    print(' [*] Waiting for messages. To exit press CTRL+C')

    # Шаг 5: Запускаем бесконечный цикл ожидания сообщений
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

### Объяснение кода Consumer

1. **queue_declare()**: Объявляем очередь повторно. Это важно, потому что consumer может запуститься раньше producer, и очередь должна существовать.

2. **callback**: Функция, вызываемая при получении сообщения:
   - `ch` — канал
   - `method` — метаданные доставки
   - `properties` — свойства сообщения
   - `body` — тело сообщения (bytes)

3. **basic_consume()**: Подписывает consumer на очередь:
   - `queue` — имя очереди
   - `on_message_callback` — функция обработки
   - `auto_ack=True` — автоподтверждение (сообщение удаляется сразу после получения)

4. **start_consuming()**: Блокирующий метод, ожидающий сообщения.

## Шаг 3: Запуск примера

### Терминал 1 — запускаем Consumer

```bash
python receive.py
# Вывод:
# [*] Waiting for messages. To exit press CTRL+C
```

### Терминал 2 — запускаем Producer

```bash
python send.py
# Вывод:
# [x] Sent 'Hello World!'
```

### Результат в Терминале 1

```
# [x] Received Hello World!
```

## Проверка через Management UI

Откройте в браузере `http://localhost:15672` (логин/пароль: `guest/guest`).

В разделе **Queues** вы увидите очередь `hello` с информацией:
- **Ready** — количество сообщений в очереди
- **Unacked** — неподтверждённые сообщения
- **Total** — общее количество

## Полезные команды CLI

```bash
# Список очередей
rabbitmqctl list_queues

# Информация о конкретной очереди
rabbitmqctl list_queues name messages consumers
```

## Важные концепции

### Default Exchange

Когда `exchange=''`, используется **default exchange**. Это специальный exchange, который направляет сообщения в очередь с именем, совпадающим с `routing_key`.

### Идемпотентность queue_declare

Объявление очереди — идемпотентная операция. Очередь создаётся только если её нет. Если очередь уже существует с теми же параметрами — ничего не происходит.

### Auto-acknowledgement

С `auto_ack=True` сообщение удаляется из очереди сразу после доставки. Это просто, но не надёжно — если consumer упадёт во время обработки, сообщение будет потеряно.

## Частые ошибки

### Ошибка подключения

```
pika.exceptions.AMQPConnectionError: Connection refused
```

**Решение**: Убедитесь, что RabbitMQ запущен.

### Очередь не найдена

```
pika.exceptions.ChannelClosedByBroker: (404, "NOT_FOUND - no queue 'hello'")
```

**Решение**: Всегда объявляйте очередь перед использованием.

## Резюме

В этом туториале мы:
1. Установили соединение с RabbitMQ
2. Создали канал и объявили очередь
3. Отправили сообщение через default exchange
4. Получили и обработали сообщение

Это базовая модель работы с RabbitMQ. В следующем туториале мы рассмотрим Work Queues — распределение задач между несколькими workers.
