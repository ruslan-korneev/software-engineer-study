# RPC (Remote Procedure Call)

## Введение

До сих пор мы рассматривали односторонние сценарии: producer отправляет, consumer получает. Но что если нам нужен ответ? В этом туториале мы реализуем паттерн **Request/Reply** (он же RPC) через RabbitMQ.

```
┌──────────────┐                              ┌──────────────┐
│    Client    │                              │    Server    │
│              │                              │              │
│  request ────┼──> [rpc_queue] ─────────────>┤  вычисляет   │
│              │                              │  fib(n)      │
│  <───────────┼── [reply_queue] <────────────┼── ответ      │
│   response   │   correlation_id             │              │
└──────────────┘                              └──────────────┘
```

## Концепция RPC через очереди

### Как это работает

1. Клиент отправляет запрос в очередь `rpc_queue`
2. Клиент указывает `reply_to` — очередь для ответа
3. Клиент указывает `correlation_id` — уникальный ID запроса
4. Сервер обрабатывает запрос
5. Сервер отправляет ответ в `reply_to` очередь с тем же `correlation_id`
6. Клиент сопоставляет ответ с запросом по `correlation_id`

### Ключевые свойства сообщения

| Свойство | Описание |
|----------|----------|
| `reply_to` | Имя очереди для ответа |
| `correlation_id` | Уникальный ID для сопоставления запроса и ответа |

## Пример: Сервис вычисления чисел Фибоначчи

Клиент отправляет число `n`, сервер возвращает `fib(n)`.

## Шаг 1: Создание RPC Server (rpc_server.py)

```python
#!/usr/bin/env python
import pika
import sys
import os

def fib(n):
    """Вычисление числа Фибоначчи (простая реализация)."""
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        return fib(n - 1) + fib(n - 2)

def main():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(host='localhost')
    )
    channel = connection.channel()

    # Объявляем очередь для запросов
    channel.queue_declare(queue='rpc_queue')

    def on_request(ch, method, props, body):
        n = int(body)
        print(f" [.] fib({n})")

        # Вычисляем результат
        response = fib(n)

        # Отправляем ответ в reply_to очередь
        ch.basic_publish(
            exchange='',
            routing_key=props.reply_to,  # Очередь клиента
            properties=pika.BasicProperties(
                correlation_id=props.correlation_id  # Тот же ID
            ),
            body=str(response)
        )

        # Подтверждаем обработку
        ch.basic_ack(delivery_tag=method.delivery_tag)

    # Обрабатываем по одному запросу за раз
    channel.basic_qos(prefetch_count=1)

    channel.basic_consume(
        queue='rpc_queue',
        on_message_callback=on_request
    )

    print(" [x] Awaiting RPC requests")
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

### Объяснение кода сервера

1. **on_request**: Callback для обработки входящих запросов
2. **props.reply_to**: Очередь, куда отправить ответ
3. **props.correlation_id**: ID для сопоставления ответа с запросом
4. **basic_qos(prefetch_count=1)**: Не брать следующий запрос, пока не обработан текущий

## Шаг 2: Создание RPC Client (rpc_client.py)

```python
#!/usr/bin/env python
import pika
import uuid
import sys

class FibonacciRpcClient:
    def __init__(self):
        # Устанавливаем соединение
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host='localhost')
        )
        self.channel = self.connection.channel()

        # Создаём эксклюзивную очередь для ответов
        result = self.channel.queue_declare(queue='', exclusive=True)
        self.callback_queue = result.method.queue

        # Подписываемся на ответы
        self.channel.basic_consume(
            queue=self.callback_queue,
            on_message_callback=self.on_response,
            auto_ack=True
        )

        self.response = None
        self.corr_id = None

    def on_response(self, ch, method, props, body):
        """Callback для получения ответа."""
        # Проверяем, что это ответ на НАШ запрос
        if self.corr_id == props.correlation_id:
            self.response = body

    def call(self, n):
        """Отправка RPC запроса и ожидание ответа."""
        self.response = None
        self.corr_id = str(uuid.uuid4())  # Генерируем уникальный ID

        # Отправляем запрос
        self.channel.basic_publish(
            exchange='',
            routing_key='rpc_queue',
            properties=pika.BasicProperties(
                reply_to=self.callback_queue,    # Куда отправить ответ
                correlation_id=self.corr_id,     # ID для сопоставления
            ),
            body=str(n)
        )

        # Ждём ответа
        print(f" [x] Requesting fib({n})")
        while self.response is None:
            self.connection.process_data_events(time_limit=None)

        return int(self.response)

    def close(self):
        self.connection.close()


def main():
    n = int(sys.argv[1]) if len(sys.argv) > 1 else 30

    fibonacci_rpc = FibonacciRpcClient()

    print(f" [x] Requesting fib({n})")
    response = fibonacci_rpc.call(n)
    print(f" [.] Got {response}")

    fibonacci_rpc.close()


if __name__ == '__main__':
    main()
```

### Объяснение кода клиента

1. **callback_queue**: Эксклюзивная очередь для получения ответов
2. **correlation_id**: UUID для идентификации запроса
3. **on_response**: Проверяет correlation_id перед принятием ответа
4. **process_data_events**: Блокирующее ожидание ответа

## Шаг 3: Запуск примера

### Терминал 1 — запускаем сервер

```bash
python rpc_server.py
# [x] Awaiting RPC requests
```

### Терминал 2 — делаем запросы

```bash
python rpc_client.py 30
# [x] Requesting fib(30)
# [.] Got 832040

python rpc_client.py 10
# [x] Requesting fib(10)
# [.] Got 55
```

### Вывод сервера

```
[.] fib(30)
[.] fib(10)
```

## Улучшенная версия с таймаутом

### RPC Client с таймаутом (rpc_client_timeout.py)

```python
#!/usr/bin/env python
import pika
import uuid
import sys
import time

class FibonacciRpcClient:
    def __init__(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host='localhost')
        )
        self.channel = self.connection.channel()

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
            self.response = body

    def call(self, n, timeout=30):
        """Отправка RPC запроса с таймаутом."""
        self.response = None
        self.corr_id = str(uuid.uuid4())

        self.channel.basic_publish(
            exchange='',
            routing_key='rpc_queue',
            properties=pika.BasicProperties(
                reply_to=self.callback_queue,
                correlation_id=self.corr_id,
            ),
            body=str(n)
        )

        # Ждём ответа с таймаутом
        start_time = time.time()
        while self.response is None:
            self.connection.process_data_events(time_limit=1)
            if time.time() - start_time > timeout:
                raise TimeoutError(f"RPC call timed out after {timeout} seconds")

        return int(self.response)

    def close(self):
        self.connection.close()


def main():
    n = int(sys.argv[1]) if len(sys.argv) > 1 else 10
    timeout = int(sys.argv[2]) if len(sys.argv) > 2 else 30

    fibonacci_rpc = FibonacciRpcClient()

    try:
        print(f" [x] Requesting fib({n}) with timeout={timeout}s")
        response = fibonacci_rpc.call(n, timeout=timeout)
        print(f" [.] Got {response}")
    except TimeoutError as e:
        print(f" [!] Error: {e}")
    finally:
        fibonacci_rpc.close()


if __name__ == '__main__':
    main()
```

## Масштабирование RPC

### Несколько серверов

Можно запустить несколько серверов — они будут распределять запросы:

```bash
# Терминал 1
python rpc_server.py

# Терминал 2
python rpc_server.py

# Запросы будут распределяться между серверами
```

### Параллельные запросы от клиента

```python
#!/usr/bin/env python
import pika
import uuid
import threading
import time

class AsyncFibonacciRpcClient:
    def __init__(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host='localhost')
        )
        self.channel = self.connection.channel()

        result = self.channel.queue_declare(queue='', exclusive=True)
        self.callback_queue = result.method.queue

        # Словарь для хранения pending запросов
        self.pending = {}
        self.lock = threading.Lock()

        self.channel.basic_consume(
            queue=self.callback_queue,
            on_message_callback=self.on_response,
            auto_ack=True
        )

        # Запускаем consumer в отдельном потоке
        self.consumer_thread = threading.Thread(target=self._consume)
        self.consumer_thread.daemon = True
        self.consumer_thread.start()

    def _consume(self):
        """Фоновый consumer."""
        self.channel.start_consuming()

    def on_response(self, ch, method, props, body):
        """Обработка ответа."""
        with self.lock:
            if props.correlation_id in self.pending:
                self.pending[props.correlation_id] = body

    def call_async(self, n):
        """Асинхронный вызов, возвращает correlation_id."""
        corr_id = str(uuid.uuid4())

        with self.lock:
            self.pending[corr_id] = None

        self.channel.basic_publish(
            exchange='',
            routing_key='rpc_queue',
            properties=pika.BasicProperties(
                reply_to=self.callback_queue,
                correlation_id=corr_id,
            ),
            body=str(n)
        )

        return corr_id

    def get_result(self, corr_id, timeout=30):
        """Получение результата по correlation_id."""
        start_time = time.time()
        while True:
            with self.lock:
                if self.pending.get(corr_id) is not None:
                    result = self.pending.pop(corr_id)
                    return int(result)

            if time.time() - start_time > timeout:
                raise TimeoutError("Timeout waiting for result")

            time.sleep(0.1)

    def close(self):
        self.channel.stop_consuming()
        self.connection.close()


def main():
    client = AsyncFibonacciRpcClient()

    # Отправляем несколько запросов параллельно
    requests = [10, 15, 20, 25, 30]
    corr_ids = {}

    print("Sending requests...")
    for n in requests:
        corr_id = client.call_async(n)
        corr_ids[n] = corr_id
        print(f" [x] Sent fib({n}), corr_id={corr_id[:8]}...")

    print("\nWaiting for results...")
    for n, corr_id in corr_ids.items():
        result = client.get_result(corr_id)
        print(f" [.] fib({n}) = {result}")

    client.close()


if __name__ == '__main__':
    main()
```

## Важные моменты

### 1. Correlation ID обязателен

Без `correlation_id` клиент не сможет понять, на какой запрос пришёл ответ:

```python
# Плохо: correlation_id не проверяется
def on_response(self, ch, method, props, body):
    self.response = body

# Хорошо: проверяем correlation_id
def on_response(self, ch, method, props, body):
    if self.corr_id == props.correlation_id:
        self.response = body
```

### 2. Обработка ошибок на сервере

```python
def on_request(ch, method, props, body):
    try:
        n = int(body)
        response = fib(n)
        error = None
    except Exception as e:
        response = None
        error = str(e)

    # Отправляем либо результат, либо ошибку
    result = {'response': response, 'error': error}

    ch.basic_publish(
        exchange='',
        routing_key=props.reply_to,
        properties=pika.BasicProperties(
            correlation_id=props.correlation_id
        ),
        body=json.dumps(result)
    )
    ch.basic_ack(delivery_tag=method.delivery_tag)
```

### 3. Эксклюзивная очередь ответов

Используйте `exclusive=True` для очереди ответов — она автоматически удалится при отключении клиента:

```python
result = self.channel.queue_declare(queue='', exclusive=True)
```

## Альтернативы RPC через RabbitMQ

### Когда НЕ использовать RPC

- Для простых REST-запросов
- Когда нужна очень низкая latency
- Когда нет необходимости в гарантированной доставке

### Когда использовать RPC

- Нужна асинхронность (fire-and-forget с последующим получением результата)
- Нужно распределение нагрузки между серверами
- Нужна надёжность (сообщение не потеряется при падении сервера)

## Сравнение с HTTP RPC

| Аспект | RabbitMQ RPC | HTTP RPC |
|--------|--------------|----------|
| Latency | Выше | Ниже |
| Надёжность | Высокая (persistent messages) | Зависит от реализации |
| Масштабирование | Встроенное (много серверов на одну очередь) | Нужен load balancer |
| Асинхронность | Нативная | Требует доп. реализации |

## CLI команды

```bash
# Просмотр очереди запросов
rabbitmqctl list_queues name messages consumers

# Просмотр очередей ответов (amq.gen-*)
rabbitmqctl list_queues | grep amq.gen
```

## Частые ошибки

### Игнорирование correlation_id

Если не проверять `correlation_id`, можно получить чужой ответ:

```python
# НЕПРАВИЛЬНО
def on_response(self, ch, method, props, body):
    self.response = body

# ПРАВИЛЬНО
def on_response(self, ch, method, props, body):
    if self.corr_id == props.correlation_id:
        self.response = body
```

### Отсутствие таймаута

Без таймаута клиент может зависнуть навсегда:

```python
# НЕПРАВИЛЬНО
while self.response is None:
    self.connection.process_data_events()

# ПРАВИЛЬНО
start = time.time()
while self.response is None:
    self.connection.process_data_events(time_limit=1)
    if time.time() - start > timeout:
        raise TimeoutError()
```

## Резюме

В этом туториале мы изучили:
1. **RPC паттерн** — Request/Reply через очереди
2. **reply_to** — очередь для ответа
3. **correlation_id** — сопоставление запроса и ответа
4. **Масштабирование** — несколько серверов на одну очередь

Следующий туториал: **Publisher Confirms** — гарантированная публикация сообщений.
