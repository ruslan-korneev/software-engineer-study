# Acknowledgments (Подтверждения)

## Введение

Acknowledgments (подтверждения) - это механизм RabbitMQ, который гарантирует надежную доставку сообщений от брокера к потребителю. Когда потребитель получает сообщение, он должен сообщить брокеру о результате обработки. Это позволяет RabbitMQ знать, можно ли удалить сообщение из очереди или нужно его переотправить.

## Типы подтверждений

### Auto Acknowledgment (Автоматическое подтверждение)

При автоматическом подтверждении RabbitMQ считает сообщение доставленным сразу после его отправки потребителю, независимо от того, было ли оно успешно обработано.

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(queue='task_queue')

def callback(ch, method, properties, body):
    print(f"Получено сообщение: {body.decode()}")
    # Если здесь произойдет ошибка, сообщение будет потеряно!
    process_message(body)

# auto_ack=True - автоматическое подтверждение
channel.basic_consume(
    queue='task_queue',
    on_message_callback=callback,
    auto_ack=True  # Опасно! Сообщения могут быть потеряны
)

channel.start_consuming()
```

**Когда использовать auto_ack:**
- Для некритичных сообщений (логи, метрики)
- Когда потеря сообщений допустима
- Для максимальной производительности

### Manual Acknowledgment (Ручное подтверждение)

При ручном подтверждении потребитель явно сообщает RabbitMQ о результате обработки сообщения.

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(queue='task_queue', durable=True)

def callback(ch, method, properties, body):
    try:
        print(f"Получено сообщение: {body.decode()}")
        process_message(body)

        # Подтверждаем успешную обработку
        ch.basic_ack(delivery_tag=method.delivery_tag)
        print("Сообщение обработано успешно")

    except Exception as e:
        print(f"Ошибка обработки: {e}")
        # Отклоняем сообщение с возможностью requeue
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)

# auto_ack=False - ручное подтверждение
channel.basic_consume(
    queue='task_queue',
    on_message_callback=callback,
    auto_ack=False  # Безопасно! Сообщения не потеряются
)

channel.start_consuming()
```

## Методы подтверждения

### basic_ack - Положительное подтверждение

Сообщает брокеру, что сообщение успешно обработано и может быть удалено из очереди.

```python
def callback(ch, method, properties, body):
    # Обработка сообщения
    result = process_message(body)

    if result.success:
        # Подтверждаем одно сообщение
        ch.basic_ack(delivery_tag=method.delivery_tag)

        # Или подтверждаем все сообщения до этого включительно
        ch.basic_ack(delivery_tag=method.delivery_tag, multiple=True)
```

**Параметры basic_ack:**
- `delivery_tag` - уникальный идентификатор сообщения в канале
- `multiple` - если True, подтверждает все сообщения с меньшим delivery_tag

### basic_nack - Отрицательное подтверждение

Сообщает брокеру, что сообщение не было обработано успешно.

```python
def callback(ch, method, properties, body):
    try:
        process_message(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except TemporaryError:
        # Временная ошибка - возвращаем в очередь
        ch.basic_nack(
            delivery_tag=method.delivery_tag,
            multiple=False,
            requeue=True  # Сообщение вернется в очередь
        )
    except PermanentError:
        # Постоянная ошибка - отбрасываем или отправляем в DLQ
        ch.basic_nack(
            delivery_tag=method.delivery_tag,
            multiple=False,
            requeue=False  # Сообщение будет отброшено или попадет в DLQ
        )
```

**Параметры basic_nack:**
- `delivery_tag` - идентификатор сообщения
- `multiple` - если True, применяется ко всем непотвержденным сообщениям
- `requeue` - если True, сообщение вернется в очередь

### basic_reject - Отклонение одного сообщения

Аналог basic_nack, но работает только с одним сообщением.

```python
def callback(ch, method, properties, body):
    if not is_valid_message(body):
        # Отклоняем невалидное сообщение
        ch.basic_reject(
            delivery_tag=method.delivery_tag,
            requeue=False  # Не возвращаем в очередь
        )
        return

    process_message(body)
    ch.basic_ack(delivery_tag=method.delivery_tag)
```

**Разница между nack и reject:**
- `basic_reject` - может отклонить только одно сообщение
- `basic_nack` - может отклонить несколько сообщений с параметром `multiple=True`

## Requeue - Возврат сообщения в очередь

При использовании `requeue=True` сообщение возвращается в очередь и может быть повторно доставлено.

```python
import pika
import time

MAX_RETRIES = 3

def callback(ch, method, properties, body):
    # Получаем счетчик повторов из заголовков
    headers = properties.headers or {}
    retry_count = headers.get('x-retry-count', 0)

    try:
        process_message(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)

    except Exception as e:
        if retry_count < MAX_RETRIES:
            print(f"Повторная попытка {retry_count + 1}/{MAX_RETRIES}")

            # Отклоняем без requeue
            ch.basic_nack(
                delivery_tag=method.delivery_tag,
                requeue=False
            )

            # Публикуем заново с увеличенным счетчиком
            ch.basic_publish(
                exchange='',
                routing_key='task_queue',
                body=body,
                properties=pika.BasicProperties(
                    headers={'x-retry-count': retry_count + 1}
                )
            )
        else:
            print(f"Превышен лимит попыток, отправляем в DLQ")
            ch.basic_nack(
                delivery_tag=method.delivery_tag,
                requeue=False
            )
```

## Unacked Messages - Неподтвержденные сообщения

Когда потребитель получает сообщение, но не отправляет подтверждение, сообщение считается "unacked". Такие сообщения:
- Остаются в памяти RabbitMQ
- Не доставляются другим потребителям
- Возвращаются в очередь при разрыве соединения

```python
# Мониторинг unacked сообщений через Management API
import requests

response = requests.get(
    'http://localhost:15672/api/queues/%2F/task_queue',
    auth=('guest', 'guest')
)
queue_info = response.json()

print(f"Всего сообщений: {queue_info['messages']}")
print(f"Готовых к доставке: {queue_info['messages_ready']}")
print(f"Неподтвержденных: {queue_info['messages_unacknowledged']}")
```

## Best Practices

### 1. Всегда используйте ручные подтверждения для важных данных

```python
# Правильно
channel.basic_consume(
    queue='important_queue',
    on_message_callback=callback,
    auto_ack=False
)

# Неправильно для важных данных
channel.basic_consume(
    queue='important_queue',
    on_message_callback=callback,
    auto_ack=True  # Риск потери данных!
)
```

### 2. Всегда отправляйте подтверждение

```python
def callback(ch, method, properties, body):
    try:
        process_message(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception as e:
        # Обязательно отправляем nack при ошибке
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
```

### 3. Используйте таймауты для долгих операций

```python
import signal

class TimeoutError(Exception):
    pass

def timeout_handler(signum, frame):
    raise TimeoutError("Обработка превысила таймаут")

def callback(ch, method, properties, body):
    signal.signal(signal.SIGALRM, timeout_handler)
    signal.alarm(30)  # 30 секунд таймаут

    try:
        process_message(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except TimeoutError:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)
    finally:
        signal.alarm(0)
```

## Частые ошибки

### 1. Забытое подтверждение

```python
# НЕПРАВИЛЬНО - подтверждение никогда не отправляется
def callback(ch, method, properties, body):
    process_message(body)
    # Забыли basic_ack!

# ПРАВИЛЬНО
def callback(ch, method, properties, body):
    process_message(body)
    ch.basic_ack(delivery_tag=method.delivery_tag)
```

### 2. Двойное подтверждение

```python
# НЕПРАВИЛЬНО - вызовет ошибку
def callback(ch, method, properties, body):
    ch.basic_ack(delivery_tag=method.delivery_tag)
    process_message(body)
    ch.basic_ack(delivery_tag=method.delivery_tag)  # Ошибка!

# ПРАВИЛЬНО
def callback(ch, method, properties, body):
    process_message(body)
    ch.basic_ack(delivery_tag=method.delivery_tag)
```

### 3. Бесконечный requeue

```python
# НЕПРАВИЛЬНО - сообщение будет бесконечно возвращаться
def callback(ch, method, properties, body):
    try:
        process_message(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)  # Бесконечный цикл!

# ПРАВИЛЬНО - ограничиваем количество повторов
def callback(ch, method, properties, body):
    try:
        process_message(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception:
        if should_retry(method, properties):
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)
        else:
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
```

## Заключение

Механизм acknowledgments - ключевой элемент надежности RabbitMQ. Правильное использование подтверждений гарантирует, что ни одно важное сообщение не будет потеряно, а проблемные сообщения будут корректно обработаны.
