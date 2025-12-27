# Prefetch (QoS)

## Введение

Prefetch (Quality of Service, QoS) - это механизм RabbitMQ, который контролирует количество непотвержденных сообщений, которые могут быть отправлены потребителю одновременно. Правильная настройка prefetch критически важна для производительности, балансировки нагрузки и корректной работы приоритетных очередей.

## Как работает Prefetch

```
Без prefetch (по умолчанию):
RabbitMQ → [Msg1][Msg2][Msg3][Msg4][Msg5] → Consumer
           (все сообщения отправлены сразу)

С prefetch_count=2:
RabbitMQ → [Msg1][Msg2] → Consumer (обрабатывает)
           [Msg3][Msg4][Msg5] (ждут в очереди)

После ACK для Msg1:
RabbitMQ → [Msg2][Msg3] → Consumer
           [Msg4][Msg5] (ждут в очереди)
```

## Настройка Prefetch

### Базовая настройка

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(queue='tasks', durable=True)

# Устанавливаем prefetch_count
channel.basic_qos(prefetch_count=1)

def callback(ch, method, properties, body):
    print(f"Обработка: {body.decode()}")
    # Длительная обработка...
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(
    queue='tasks',
    on_message_callback=callback,
    auto_ack=False  # Обязательно False для работы prefetch!
)

channel.start_consuming()
```

### Параметры basic_qos

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Полная сигнатура basic_qos
channel.basic_qos(
    prefetch_size=0,    # Лимит по размеру (байты), 0 = без лимита
    prefetch_count=10,  # Лимит по количеству сообщений
    global_qos=False    # Применить ко всему каналу или только к consumer
)

"""
prefetch_size:
- Лимит в байтах для непотвержденных сообщений
- 0 = без лимита (рекомендуется)
- Редко используется на практике

prefetch_count:
- Количество непотвержденных сообщений
- 0 = без лимита (опасно!)
- Рекомендуется: 1-100 в зависимости от задачи

global_qos:
- False: применяется к каждому consumer отдельно
- True: общий лимит на весь канал
"""
```

### Разница между global и per-consumer

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Per-consumer prefetch (по умолчанию)
# Каждый consumer на этом канале получит максимум 5 сообщений
channel.basic_qos(prefetch_count=5, global_qos=False)

# Если у нас 3 consumer на канале:
# Consumer1: до 5 сообщений
# Consumer2: до 5 сообщений
# Consumer3: до 5 сообщений
# Итого: до 15 сообщений в работе

# Global prefetch
# Все consumers на канале делят 5 сообщений между собой
channel.basic_qos(prefetch_count=5, global_qos=True)

# Если у нас 3 consumer на канале:
# Все вместе: максимум 5 сообщений
# Например: Consumer1=2, Consumer2=2, Consumer3=1
```

## Fair Dispatch (Справедливое распределение)

### Проблема без prefetch

```python
"""
Без prefetch (round-robin распределение):

Сообщения: [1][2][3][4][5][6][7][8][9][10]

Consumer1 (медленный): получает 1, 3, 5, 7, 9
Consumer2 (быстрый):   получает 2, 4, 6, 8, 10

Результат:
- Consumer2 обработал 2, 4, 6, 8, 10 и простаивает
- Consumer1 все еще обрабатывает 1, 3, 5, 7, 9

Это несправедливо! Быстрый worker простаивает.
"""
```

### Решение с prefetch_count=1

```python
import pika
import time
import random

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(queue='fair_tasks', durable=True)

# Ключевая настройка для fair dispatch
channel.basic_qos(prefetch_count=1)

def callback(ch, method, properties, body):
    task = body.decode()
    processing_time = random.uniform(0.5, 2.0)

    print(f"[Worker] Начало: {task}, время: {processing_time:.1f}с")
    time.sleep(processing_time)
    print(f"[Worker] Готово: {task}")

    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(
    queue='fair_tasks',
    on_message_callback=callback,
    auto_ack=False
)

print("Ожидание задач (fair dispatch)...")
channel.start_consuming()

"""
С prefetch_count=1:

Сообщения: [1][2][3][4][5][6][7][8][9][10]

Consumer1 (медленный): получает 1, ждет...
Consumer2 (быстрый):   получает 2, ACK, получает 3, ACK, получает 4...

Результат:
- Consumer2 обрабатывает больше сообщений
- Оба worker заняты
- Нагрузка распределена по возможностям
"""
```

## Выбор оптимального Prefetch Count

### Факторы выбора

```python
"""
Низкий prefetch_count (1-5):
+ Справедливое распределение
+ Хорошо для приоритетных очередей
+ Быстрая реакция на изменения
- Высокий network overhead
- Меньшая пропускная способность

Высокий prefetch_count (50-100+):
+ Высокая пропускная способность
+ Меньше network overhead
+ Эффективно для однородных задач
- Несправедливое распределение
- Память consumer'а может переполниться
- Долго "разгружаться" при остановке
"""

import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Для разных сценариев:

# 1. Критичная балансировка (приоритеты, разное время обработки)
channel.basic_qos(prefetch_count=1)

# 2. Сбалансированный вариант (большинство случаев)
channel.basic_qos(prefetch_count=10)

# 3. Высокая пропускная способность (однородные быстрые задачи)
channel.basic_qos(prefetch_count=50)

# 4. Batch processing (очень быстрые задачи)
channel.basic_qos(prefetch_count=100)
```

### Формула для расчета

```python
"""
Рекомендуемая формула:

prefetch_count = (RTT / processing_time) * safety_factor

Где:
- RTT = round-trip time до брокера (мс)
- processing_time = среднее время обработки сообщения (мс)
- safety_factor = 2-3 (запас)

Примеры:
- RTT = 1мс, processing = 10мс → prefetch = (1/10) * 2 = 1
- RTT = 1мс, processing = 1мс → prefetch = (1/1) * 2 = 2
- RTT = 10мс, processing = 100мс → prefetch = (10/100) * 2 = 1
- RTT = 10мс, processing = 1мс → prefetch = (10/1) * 2 = 20
"""

def calculate_prefetch(rtt_ms, processing_time_ms, safety_factor=2):
    """Расчет оптимального prefetch_count"""
    if processing_time_ms == 0:
        return 1

    raw_prefetch = (rtt_ms / processing_time_ms) * safety_factor
    return max(1, min(100, int(raw_prefetch)))

# Примеры
print(calculate_prefetch(1, 10))    # 1
print(calculate_prefetch(1, 1))     # 2
print(calculate_prefetch(10, 100))  # 1
print(calculate_prefetch(10, 1))    # 20
```

## Динамическая настройка Prefetch

### Адаптивный prefetch

```python
import pika
import time
from threading import Thread
from collections import deque

class AdaptivePrefetchConsumer:
    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()
        self.channel.queue_declare(queue='adaptive_queue', durable=True)

        # Статистика обработки
        self.processing_times = deque(maxlen=100)
        self.current_prefetch = 10

        # Начальный prefetch
        self.channel.basic_qos(prefetch_count=self.current_prefetch)

    def callback(self, ch, method, properties, body):
        start = time.time()

        # Обработка сообщения
        self.process_message(body)

        processing_time = time.time() - start
        self.processing_times.append(processing_time)

        ch.basic_ack(delivery_tag=method.delivery_tag)

        # Адаптируем prefetch
        self.adapt_prefetch()

    def process_message(self, body):
        """Имитация обработки"""
        time.sleep(0.1)

    def adapt_prefetch(self):
        """Адаптация prefetch на основе статистики"""
        if len(self.processing_times) < 10:
            return

        avg_time = sum(self.processing_times) / len(self.processing_times)

        # Целевое время "в пути" - 10мс
        target_rtt = 0.01
        new_prefetch = int((target_rtt / avg_time) * 2)
        new_prefetch = max(1, min(100, new_prefetch))

        if new_prefetch != self.current_prefetch:
            print(f"Изменение prefetch: {self.current_prefetch} → {new_prefetch}")
            self.current_prefetch = new_prefetch
            self.channel.basic_qos(prefetch_count=new_prefetch)

    def start(self):
        self.channel.basic_consume(
            queue='adaptive_queue',
            on_message_callback=self.callback,
            auto_ack=False
        )
        self.channel.start_consuming()


# consumer = AdaptivePrefetchConsumer()
# consumer.start()
```

## Prefetch и несколько очередей

### Один consumer - несколько очередей

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Создаем очереди
channel.queue_declare(queue='high_priority', durable=True)
channel.queue_declare(queue='low_priority', durable=True)

# Один prefetch для всех очередей на канале
channel.basic_qos(prefetch_count=5)

def high_callback(ch, method, properties, body):
    print(f"[HIGH] {body.decode()}")
    ch.basic_ack(delivery_tag=method.delivery_tag)

def low_callback(ch, method, properties, body):
    print(f"[LOW] {body.decode()}")
    ch.basic_ack(delivery_tag=method.delivery_tag)

# Один consumer может получить до 5 сообщений суммарно из обеих очередей
channel.basic_consume(queue='high_priority', on_message_callback=high_callback, auto_ack=False)
channel.basic_consume(queue='low_priority', on_message_callback=low_callback, auto_ack=False)

channel.start_consuming()
```

### Разный prefetch для разных очередей

```python
import pika
from threading import Thread

def create_consumer(queue_name, prefetch_count):
    """Создание consumer с индивидуальным prefetch"""
    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    channel = connection.channel()

    channel.queue_declare(queue=queue_name, durable=True)
    channel.basic_qos(prefetch_count=prefetch_count)

    def callback(ch, method, properties, body):
        print(f"[{queue_name}] {body.decode()}")
        ch.basic_ack(delivery_tag=method.delivery_tag)

    channel.basic_consume(
        queue=queue_name,
        on_message_callback=callback,
        auto_ack=False
    )

    print(f"Consumer для {queue_name} запущен (prefetch={prefetch_count})")
    channel.start_consuming()


# Запускаем consumers в разных потоках с разным prefetch
threads = [
    Thread(target=create_consumer, args=('critical_tasks', 1)),   # Строгий порядок
    Thread(target=create_consumer, args=('normal_tasks', 10)),    # Баланс
    Thread(target=create_consumer, args=('batch_tasks', 50)),     # Производительность
]

for t in threads:
    t.daemon = True
    t.start()

for t in threads:
    t.join()
```

## Мониторинг Prefetch

### Отслеживание unacked сообщений

```python
import pika
import requests

def monitor_prefetch_health(queue_name, vhost='%2F'):
    """Мониторинг здоровья prefetch"""
    response = requests.get(
        f'http://localhost:15672/api/queues/{vhost}/{queue_name}',
        auth=('guest', 'guest')
    )

    if response.status_code == 200:
        queue = response.json()

        total = queue['messages']
        ready = queue['messages_ready']
        unacked = queue['messages_unacknowledged']
        consumers = queue['consumers']

        print(f"Очередь: {queue_name}")
        print(f"  Всего: {total}")
        print(f"  Готовых: {ready}")
        print(f"  Unacked: {unacked}")
        print(f"  Consumers: {consumers}")

        if consumers > 0:
            avg_unacked = unacked / consumers
            print(f"  Среднее unacked/consumer: {avg_unacked:.1f}")

            # Предупреждения
            if avg_unacked > 100:
                print("  ПРЕДУПРЕЖДЕНИЕ: Слишком много unacked!")
            if ready > 1000 and unacked < consumers * 5:
                print("  СОВЕТ: Увеличьте prefetch для производительности")

        return queue

    return None


def get_consumer_prefetch_stats():
    """Получение статистики prefetch по всем consumers"""
    response = requests.get(
        'http://localhost:15672/api/consumers',
        auth=('guest', 'guest')
    )

    if response.status_code == 200:
        consumers = response.json()

        print("Consumer Prefetch Statistics:")
        for consumer in consumers:
            queue = consumer['queue']['name']
            prefetch = consumer.get('prefetch_count', 0)
            ack_required = consumer.get('ack_required', True)

            print(f"  {queue}: prefetch={prefetch}, ack_required={ack_required}")

        return consumers

    return None


# Мониторинг
monitor_prefetch_health('tasks')
get_consumer_prefetch_stats()
```

## Практические сценарии

### Обработка изображений

```python
import pika
import time

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(queue='image_processing', durable=True)

# Обработка изображений - тяжелая операция
# Используем низкий prefetch для балансировки
channel.basic_qos(prefetch_count=2)

def process_image(ch, method, properties, body):
    image_id = body.decode()
    print(f"Начало обработки изображения: {image_id}")

    # Тяжелая обработка (resize, filters, etc.)
    time.sleep(5)

    print(f"Изображение обработано: {image_id}")
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(
    queue='image_processing',
    on_message_callback=process_image,
    auto_ack=False
)

channel.start_consuming()
```

### Отправка email

```python
import pika
import time

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(queue='email_queue', durable=True)

# Email отправка - быстрая операция, можно брать больше
channel.basic_qos(prefetch_count=20)

def send_email(ch, method, properties, body):
    email_data = body.decode()
    # Отправка email через SMTP - обычно быстро
    time.sleep(0.1)
    print(f"Email отправлен: {email_data}")
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(
    queue='email_queue',
    on_message_callback=send_email,
    auto_ack=False
)

channel.start_consuming()
```

### Batch processing с большим prefetch

```python
import pika
import time
from collections import deque

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(queue='batch_queue', durable=True)

# Большой prefetch для batch обработки
channel.basic_qos(prefetch_count=100)

batch = []
batch_size = 50
last_flush = time.time()

def batch_callback(ch, method, properties, body):
    global batch, last_flush

    batch.append((method.delivery_tag, body.decode()))

    # Обрабатываем batch когда накопилось достаточно
    # или прошло больше 5 секунд
    if len(batch) >= batch_size or (time.time() - last_flush) > 5:
        process_batch(ch)

def process_batch(ch):
    global batch, last_flush

    if not batch:
        return

    print(f"Обработка batch из {len(batch)} элементов")

    # Batch операция (например, bulk insert в БД)
    time.sleep(0.5)

    # Подтверждаем все сообщения
    for delivery_tag, _ in batch:
        ch.basic_ack(delivery_tag=delivery_tag)

    batch = []
    last_flush = time.time()

channel.basic_consume(
    queue='batch_queue',
    on_message_callback=batch_callback,
    auto_ack=False
)

channel.start_consuming()
```

## Best Practices

### 1. Всегда устанавливайте prefetch

```python
# НЕПРАВИЛЬНО - без prefetch брокер отправит все сообщения сразу
channel.basic_consume(queue='tasks', on_message_callback=callback, auto_ack=False)

# ПРАВИЛЬНО
channel.basic_qos(prefetch_count=10)
channel.basic_consume(queue='tasks', on_message_callback=callback, auto_ack=False)
```

### 2. Используйте auto_ack=False

```python
# НЕПРАВИЛЬНО - prefetch не работает с auto_ack=True
channel.basic_qos(prefetch_count=1)  # Бессмысленно!
channel.basic_consume(queue='tasks', on_message_callback=callback, auto_ack=True)

# ПРАВИЛЬНО
channel.basic_qos(prefetch_count=1)
channel.basic_consume(queue='tasks', on_message_callback=callback, auto_ack=False)
```

### 3. Подбирайте prefetch под задачу

```python
# Для задач с разным временем обработки
channel.basic_qos(prefetch_count=1)  # Fair dispatch

# Для однородных быстрых задач
channel.basic_qos(prefetch_count=50)  # Производительность

# Для приоритетных очередей
channel.basic_qos(prefetch_count=1)  # Строгий порядок
```

### 4. Мониторьте и адаптируйте

```python
"""
Признаки неправильного prefetch:

Слишком низкий:
- Низкая пропускная способность
- Consumers часто простаивают
- Очередь растет при наличии свободных consumers

Слишком высокий:
- Неравномерное распределение нагрузки
- Долгое "разгружение" при остановке
- Высокое потребление памяти consumer'ами
"""
```

## Частые ошибки

### 1. Prefetch с auto_ack=True

```python
# НЕПРАВИЛЬНО - prefetch игнорируется
channel.basic_qos(prefetch_count=1)
channel.basic_consume(queue='q', on_message_callback=cb, auto_ack=True)

# ПРАВИЛЬНО
channel.basic_qos(prefetch_count=1)
channel.basic_consume(queue='q', on_message_callback=cb, auto_ack=False)
```

### 2. Забыли вызвать basic_ack

```python
# НЕПРАВИЛЬНО - unacked копятся, новые сообщения не приходят
def callback(ch, method, properties, body):
    process(body)
    # Забыли ack!

# ПРАВИЛЬНО
def callback(ch, method, properties, body):
    process(body)
    ch.basic_ack(delivery_tag=method.delivery_tag)
```

### 3. prefetch_count=0

```python
# НЕПРАВИЛЬНО - 0 означает "без лимита"
channel.basic_qos(prefetch_count=0)  # Опасно!

# ПРАВИЛЬНО
channel.basic_qos(prefetch_count=10)  # Разумный лимит
```

## Заключение

Prefetch (QoS) - ключевой механизм управления потоком сообщений:

1. Всегда устанавливайте prefetch_count > 0
2. Используйте prefetch_count=1 для fair dispatch и приоритетных очередей
3. Увеличивайте prefetch для быстрых однородных задач
4. Обязательно используйте auto_ack=False
5. Мониторьте unacked сообщения и адаптируйте настройки
6. Помните: prefetch влияет на производительность, балансировку и использование памяти
