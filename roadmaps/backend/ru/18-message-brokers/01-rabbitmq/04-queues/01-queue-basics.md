# Основы очередей

## Что такое очередь в RabbitMQ

Очередь (Queue) — это буфер, который хранит сообщения до тех пор, пока они не будут обработаны потребителем (consumer). Очереди являются основным механизмом хранения сообщений в RabbitMQ и работают по принципу FIFO (First In, First Out) — первое пришедшее сообщение будет обработано первым.

## Основные характеристики очередей

### Именование очередей

Очереди идентифицируются по имени. Имя может содержать до 255 символов UTF-8. Существует два подхода к именованию:

1. **Явное именование** — вы сами задаёте имя очереди
2. **Автоматическое именование** — RabbitMQ генерирует уникальное имя (полезно для временных очередей)

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Явное именование
channel.queue_declare(queue='my_queue')

# Автоматическое именование (пустая строка)
result = channel.queue_declare(queue='')
auto_generated_name = result.method.queue
print(f"Сгенерированное имя: {auto_generated_name}")
# Пример вывода: amq.gen-JzTY20BRgKO-HjmUJj0wLg
```

## Создание очереди

### Базовое создание

```python
import pika

def create_basic_queue():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Создаём простую очередь
    channel.queue_declare(queue='task_queue')

    print("Очередь 'task_queue' создана успешно")

    connection.close()

create_basic_queue()
```

### Создание с параметрами

```python
import pika

def create_queue_with_options():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Создаём долговечную очередь с дополнительными параметрами
    channel.queue_declare(
        queue='persistent_queue',
        durable=True,           # Переживёт перезапуск брокера
        exclusive=False,        # Может использоваться несколькими соединениями
        auto_delete=False,      # Не удаляется при отключении всех потребителей
        arguments={
            'x-message-ttl': 60000,      # TTL сообщений 60 секунд
            'x-max-length': 10000,       # Максимум 10000 сообщений
            'x-overflow': 'reject-publish'  # Поведение при переполнении
        }
    )

    connection.close()

create_queue_with_options()
```

## Жизненный цикл очереди

### 1. Создание (Declaration)

Очередь создаётся при первом объявлении. Если очередь уже существует с теми же параметрами, объявление проходит успешно. Если параметры отличаются — возникает ошибка.

```python
import pika

def safe_queue_declare():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    try:
        # Идемпотентная операция — безопасно вызывать многократно
        channel.queue_declare(queue='idempotent_queue', durable=True)
        print("Очередь объявлена успешно")
    except pika.exceptions.ChannelClosedByBroker as e:
        print(f"Ошибка: параметры очереди не совпадают: {e}")

    connection.close()
```

### 2. Привязка (Binding)

После создания очередь может быть привязана к exchange для получения сообщений:

```python
import pika

def bind_queue_to_exchange():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Создаём exchange
    channel.exchange_declare(
        exchange='logs',
        exchange_type='topic'
    )

    # Создаём очередь
    channel.queue_declare(queue='error_logs')

    # Привязываем очередь к exchange
    channel.queue_bind(
        queue='error_logs',
        exchange='logs',
        routing_key='*.error'
    )

    print("Очередь привязана к exchange")
    connection.close()

bind_queue_to_exchange()
```

### 3. Использование (Publishing/Consuming)

```python
import pika

def publish_and_consume():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    queue_name = 'example_queue'
    channel.queue_declare(queue=queue_name)

    # Публикация сообщения
    channel.basic_publish(
        exchange='',
        routing_key=queue_name,
        body='Hello, RabbitMQ!'
    )
    print("Сообщение отправлено")

    # Получение сообщения
    def callback(ch, method, properties, body):
        print(f"Получено: {body.decode()}")
        ch.basic_ack(delivery_tag=method.delivery_tag)

    channel.basic_consume(
        queue=queue_name,
        on_message_callback=callback,
        auto_ack=False
    )

    print("Ожидание сообщений...")
    channel.start_consuming()
```

### 4. Удаление (Deletion)

```python
import pika

def delete_queue():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Удаление очереди
    channel.queue_delete(queue='example_queue')

    # Удаление с условиями
    channel.queue_delete(
        queue='conditional_queue',
        if_unused=True,    # Только если нет потребителей
        if_empty=True      # Только если очередь пуста
    )

    connection.close()
```

## Проверка существования очереди

```python
import pika

def check_queue_exists():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    try:
        # passive=True проверяет существование без создания
        result = channel.queue_declare(
            queue='check_me',
            passive=True
        )
        message_count = result.method.message_count
        consumer_count = result.method.consumer_count
        print(f"Очередь существует: {message_count} сообщений, {consumer_count} потребителей")
    except pika.exceptions.ChannelClosedByBroker:
        print("Очередь не существует")

    connection.close()
```

## Получение информации об очереди

```python
import pika

def get_queue_info():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Объявляем очередь и получаем информацию
    result = channel.queue_declare(queue='info_queue', passive=True)

    print(f"Имя очереди: {result.method.queue}")
    print(f"Количество сообщений: {result.method.message_count}")
    print(f"Количество потребителей: {result.method.consumer_count}")

    connection.close()
```

## Очистка очереди

```python
import pika

def purge_queue():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()

    # Очищаем все сообщения из очереди
    result = channel.queue_purge(queue='messages_queue')
    print(f"Удалено сообщений: {result.method.message_count}")

    connection.close()
```

## Best Practices

### 1. Используйте понятные имена

```python
# Хорошо
queue_names = [
    'orders.processing',
    'users.notifications',
    'payments.failed'
]

# Плохо
bad_names = ['q1', 'temp', 'test123']
```

### 2. Объявляйте очереди на обоих концах

```python
# И producer, и consumer должны объявлять очередь
# Это гарантирует, что очередь существует независимо от порядка запуска

def producer():
    channel.queue_declare(queue='shared_queue', durable=True)
    # ... публикация

def consumer():
    channel.queue_declare(queue='shared_queue', durable=True)
    # ... потребление
```

### 3. Используйте durable для важных данных

```python
# Для критичных сообщений
channel.queue_declare(queue='important_tasks', durable=True)

# Для временных/некритичных
channel.queue_declare(queue='temp_notifications', durable=False)
```

### 4. Ограничивайте размер очередей

```python
channel.queue_declare(
    queue='bounded_queue',
    arguments={
        'x-max-length': 100000,           # Максимум сообщений
        'x-max-length-bytes': 104857600,  # Максимум 100MB
        'x-overflow': 'reject-publish'    # Отклонять новые при переполнении
    }
)
```

## Типичные ошибки

### Несовпадение параметров

```python
# Первое объявление
channel.queue_declare(queue='my_queue', durable=True)

# Второе объявление с другими параметрами — ОШИБКА!
channel.queue_declare(queue='my_queue', durable=False)
# Вызовет: PRECONDITION_FAILED - inequivalent arg 'durable'
```

### Забытые acknowledgements

```python
# НЕПРАВИЛЬНО — сообщения потеряются при сбое
channel.basic_consume(queue='tasks', on_message_callback=callback, auto_ack=True)

# ПРАВИЛЬНО — явное подтверждение после обработки
def callback(ch, method, properties, body):
    process_message(body)
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(queue='tasks', on_message_callback=callback, auto_ack=False)
```

## Заключение

Очереди — фундаментальный компонент RabbitMQ, обеспечивающий надёжное хранение и доставку сообщений. Правильное понимание их жизненного цикла, свойств и методов работы критически важно для построения надёжных распределённых систем.
