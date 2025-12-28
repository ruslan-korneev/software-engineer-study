# Паттерны отправки сообщений - Sending Patterns

[prev: 02-producer-options](./02-producer-options.md) | [next: 01-example](../05-consumers/01-example.md)

---

## Описание

Kafka Producer поддерживает различные паттерны отправки сообщений, которые отличаются по гарантиям доставки, производительности и сложности реализации. Выбор правильного паттерна зависит от требований к надёжности, задержкам и обработке ошибок.

Основные паттерны:
- **Fire-and-forget** — отправить и забыть (максимальная производительность)
- **Синхронная отправка** — ждать подтверждения (простота обработки ошибок)
- **Асинхронная отправка с callback** — неблокирующая с обработкой результата
- **Идемпотентная отправка** — гарантия exactly-once на уровне Producer

## Ключевые концепции

### Паттерн Fire-and-Forget

Отправка без ожидания результата. Самый быстрый, но без гарантий доставки.

```
Producer ──▶ send() ──▶ Broker
    │
    └─── (не ждём ответа)
```

### Синхронная отправка

Блокирующее ожидание подтверждения от брокера. Простая обработка ошибок, но низкая производительность.

```
Producer ──▶ send() ──▶ Broker
    │                      │
    │◀─────── ACK ─────────┘
    │
    └─── (продолжаем после ответа)
```

### Асинхронная отправка с Callback

Неблокирующая отправка с callback-функцией для обработки результата.

```
Producer ──▶ send(callback) ──▶ Broker
    │                              │
    └─── (продолжаем работу)       │
                                   │
    callback(result) ◀─────────────┘
```

### Идемпотентность

Гарантия, что повторная отправка того же сообщения не создаст дубликат.

```
Producer ──▶ msg(PID=1, seq=1) ──▶ Broker (записал)
    │
    │ (сетевая ошибка, retry)
    │
Producer ──▶ msg(PID=1, seq=1) ──▶ Broker (дубликат, игнорирует)
```

## Примеры кода

### Python (confluent-kafka) — Все паттерны

```python
from confluent_kafka import Producer, KafkaError
import json
import time

config = {
    'bootstrap.servers': 'localhost:9092',
    'client.id': 'patterns-demo'
}

producer = Producer(config)

# ============================================
# Паттерн 1: Fire-and-Forget
# ============================================
def fire_and_forget():
    """Максимальная производительность, минимальные гарантии"""
    for i in range(1000):
        producer.produce(
            topic='fast-topic',
            value=json.dumps({'id': i}).encode('utf-8')
        )
        # poll(0) нужен для обработки внутренних событий
        # но не ждём подтверждения
        producer.poll(0)

    # В конце всё равно вызываем flush для завершения
    producer.flush()


# ============================================
# Паттерн 2: Синхронная отправка
# ============================================
def synchronous_send():
    """Простая обработка ошибок, низкая производительность"""
    from confluent_kafka import KafkaException

    for i in range(10):
        try:
            # produce() сам по себе неблокирующий
            producer.produce(
                topic='sync-topic',
                value=json.dumps({'id': i}).encode('utf-8')
            )
            # flush() блокирует до отправки всех сообщений
            # Для синхронной отправки вызываем после каждого сообщения
            producer.flush(timeout=10)  # Таймаут 10 секунд
            print(f'Сообщение {i} отправлено')
        except KafkaException as e:
            print(f'Ошибка отправки {i}: {e}')


# ============================================
# Паттерн 3: Асинхронная отправка с Callback
# ============================================
class AsyncProducer:
    def __init__(self):
        self.success_count = 0
        self.error_count = 0
        self.errors = []

    def delivery_callback(self, err, msg):
        """Callback вызывается для каждого сообщения"""
        if err:
            self.error_count += 1
            self.errors.append({
                'error': str(err),
                'topic': msg.topic(),
                'key': msg.key()
            })
            print(f'ОШИБКА: {err}')
        else:
            self.success_count += 1
            print(f'OK: {msg.topic()} [{msg.partition()}] @ {msg.offset()}')

    def send_async(self, topic, messages):
        """Асинхронная отправка с callback"""
        for msg in messages:
            producer.produce(
                topic=topic,
                key=msg.get('key'),
                value=json.dumps(msg['value']).encode('utf-8'),
                callback=self.delivery_callback
            )
            # poll(0) обрабатывает callback'и без блокировки
            producer.poll(0)

        # В конце ждём все подтверждения
        producer.flush()

        return {
            'success': self.success_count,
            'errors': self.error_count,
            'error_details': self.errors
        }


async_producer = AsyncProducer()
messages = [
    {'key': 'user-1', 'value': {'action': 'login'}},
    {'key': 'user-2', 'value': {'action': 'purchase'}},
]
result = async_producer.send_async('events', messages)
print(f'Результат: {result}')


# ============================================
# Паттерн 4: Асинхронная отправка с Future (kafka-python)
# ============================================
# Для kafka-python можно использовать futures

from kafka import KafkaProducer as KPProducer
from kafka.errors import KafkaError
import concurrent.futures

def async_with_futures():
    kp_producer = KPProducer(
        bootstrap_servers=['localhost:9092'],
        value_serializer=lambda v: json.dumps(v).encode('utf-8')
    )

    futures = []
    for i in range(100):
        future = kp_producer.send('futures-topic', value={'id': i})
        futures.append(future)

    # Дождаться всех результатов
    for i, future in enumerate(futures):
        try:
            record_metadata = future.get(timeout=10)
            print(f'{i}: partition={record_metadata.partition}, offset={record_metadata.offset}')
        except KafkaError as e:
            print(f'{i}: ОШИБКА - {e}')

    kp_producer.close()
```

### Python — Обработка ошибок

```python
from confluent_kafka import Producer, KafkaError, KafkaException
import json

class RobustProducer:
    def __init__(self, config):
        self.producer = Producer(config)
        self.failed_messages = []

    def delivery_callback(self, err, msg):
        if err:
            # Сохраняем неудачные сообщения для повторной отправки
            self.failed_messages.append({
                'topic': msg.topic(),
                'key': msg.key(),
                'value': msg.value(),
                'error': str(err)
            })

    def send_with_retry(self, topic, key, value, max_retries=3):
        """Отправка с повторными попытками"""
        for attempt in range(max_retries):
            try:
                self.producer.produce(
                    topic=topic,
                    key=key,
                    value=json.dumps(value).encode('utf-8'),
                    callback=self.delivery_callback
                )
                self.producer.flush(timeout=10)

                if not self.failed_messages:
                    return True

                # Была ошибка в callback
                last_error = self.failed_messages.pop()
                print(f'Попытка {attempt + 1}: {last_error["error"]}')

            except BufferError:
                # Буфер переполнен - ждём и повторяем
                print('Буфер полон, ожидание...')
                self.producer.poll(1)

            except KafkaException as e:
                print(f'Kafka ошибка: {e}')
                if attempt < max_retries - 1:
                    time.sleep(2 ** attempt)  # Exponential backoff

        return False

    def handle_retriable_errors(self, err):
        """Определение, можно ли повторить"""
        retriable_errors = [
            KafkaError._TIMED_OUT,
            KafkaError._TRANSPORT,
            KafkaError.REQUEST_TIMED_OUT,
            KafkaError.NOT_LEADER_FOR_PARTITION,
            KafkaError.LEADER_NOT_AVAILABLE,
        ]
        return err.code() in retriable_errors


# Использование
config = {
    'bootstrap.servers': 'localhost:9092',
    'acks': 'all',
    'retries': 5,
    'retry.backoff.ms': 200,
}

robust = RobustProducer(config)
success = robust.send_with_retry('orders', 'order-123', {'amount': 99.99})
```

### Python — Идемпотентный Producer

```python
from confluent_kafka import Producer
import json

# Идемпотентная конфигурация
idempotent_config = {
    'bootstrap.servers': 'localhost:9092',

    # Включение идемпотентности
    'enable.idempotence': True,

    # Эти параметры устанавливаются автоматически при idempotence=True:
    # 'acks': 'all',
    # 'retries': 2147483647,
    # 'max.in.flight.requests.per.connection': 5,

    'client.id': 'idempotent-producer'
}

producer = Producer(idempotent_config)

def send_exactly_once(topic, key, value):
    """
    Гарантия exactly-once на уровне Producer.
    Даже при повторных попытках сообщение будет записано один раз.
    """
    def callback(err, msg):
        if err:
            print(f'Ошибка: {err}')
            # При idempotence=True Kafka сама дедуплицирует повторы
        else:
            print(f'Доставлено: {msg.topic()} [{msg.partition()}] @ {msg.offset()}')

    producer.produce(
        topic=topic,
        key=key,
        value=json.dumps(value).encode('utf-8'),
        callback=callback
    )
    producer.poll(0)


# Отправка
for i in range(10):
    send_exactly_once('transactions', f'tx-{i}', {'amount': i * 10})

producer.flush()
```

### Java — Паттерны отправки

```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;

public class SendingPatterns {

    private final KafkaProducer<String, String> producer;

    public SendingPatterns() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);

        this.producer = new KafkaProducer<>(props);
    }

    // Паттерн 1: Fire-and-Forget
    public void fireAndForget(String topic, String key, String value) {
        ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, value);
        producer.send(record);
        // Не ждём результат
    }

    // Паттерн 2: Синхронная отправка
    public RecordMetadata sendSync(String topic, String key, String value)
            throws ExecutionException, InterruptedException {
        ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, value);

        // get() блокирует до получения результата
        return producer.send(record).get();
    }

    // Паттерн 3: Асинхронная отправка с Callback
    public Future<RecordMetadata> sendAsync(String topic, String key, String value) {
        ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, value);

        return producer.send(record, (metadata, exception) -> {
            if (exception != null) {
                System.err.println("Ошибка отправки: " + exception.getMessage());
                // Логирование, повторная отправка, алерт
                handleError(record, exception);
            } else {
                System.out.printf("Отправлено в %s [%d] @ %d%n",
                    metadata.topic(), metadata.partition(), metadata.offset());
            }
        });
    }

    // Обработка ошибок
    private void handleError(ProducerRecord<String, String> record, Exception e) {
        // Можно сохранить в dead letter queue
        // Или повторить отправку
        System.err.println("Не удалось отправить: " + record.key());
    }

    // Паттерн 4: Batch отправка
    public void sendBatch(String topic, java.util.List<String> messages) {
        java.util.List<Future<RecordMetadata>> futures = new java.util.ArrayList<>();

        for (String msg : messages) {
            ProducerRecord<String, String> record = new ProducerRecord<>(topic, msg);
            futures.add(producer.send(record));
        }

        // Ждём все результаты
        for (Future<RecordMetadata> future : futures) {
            try {
                RecordMetadata metadata = future.get();
                System.out.println("OK: offset=" + metadata.offset());
            } catch (Exception e) {
                System.err.println("Error: " + e.getMessage());
            }
        }
    }

    public void close() {
        producer.flush();
        producer.close();
    }
}
```

### Python — Транзакции (Exactly-Once Semantics)

```python
from confluent_kafka import Producer, Consumer, KafkaError
import json

# Транзакционный Producer для exactly-once между Producer и Consumer
transactional_config = {
    'bootstrap.servers': 'localhost:9092',
    'transactional.id': 'my-transactional-producer',  # Обязательно для транзакций
    'enable.idempotence': True,
}

producer = Producer(transactional_config)

# Инициализация транзакций (вызывается один раз)
producer.init_transactions()

def process_and_produce(input_topic, output_topic, consumer):
    """
    Паттерн: читаем, обрабатываем, пишем — атомарно
    """
    while True:
        msg = consumer.poll(1.0)
        if msg is None:
            continue
        if msg.error():
            print(f'Consumer error: {msg.error()}')
            continue

        try:
            # Начало транзакции
            producer.begin_transaction()

            # Обработка сообщения
            input_data = json.loads(msg.value())
            output_data = transform(input_data)

            # Отправка результата
            producer.produce(
                topic=output_topic,
                key=msg.key(),
                value=json.dumps(output_data).encode('utf-8')
            )

            # Коммит оффсета consumer как часть транзакции
            producer.send_offsets_to_transaction(
                consumer.position(consumer.assignment()),
                consumer.consumer_group_metadata()
            )

            # Коммит транзакции
            producer.commit_transaction()

        except Exception as e:
            print(f'Ошибка в транзакции: {e}')
            # Откат транзакции
            producer.abort_transaction()

def transform(data):
    # Трансформация данных
    data['processed'] = True
    return data
```

## Best Practices

### 1. Выбор паттерна в зависимости от требований

```python
# Высокая производительность, потеря допустима (логи, метрики)
# → Fire-and-Forget

# Простота, нужна уверенность в доставке
# → Синхронная отправка

# Производительность + обработка ошибок
# → Асинхронная с Callback

# Критичные данные, дубликаты недопустимы
# → Идемпотентность или Транзакции
```

### 2. Обработка ошибок в Callback

```python
def smart_callback(err, msg):
    if err:
        error_code = err.code()

        # Retriable ошибки - логируем, но не паникуем
        if error_code in [KafkaError._TIMED_OUT, KafkaError.REQUEST_TIMED_OUT]:
            log.warning(f'Временная ошибка, будет повтор: {err}')

        # Fatal ошибки - алерт
        elif error_code in [KafkaError.TOPIC_AUTHORIZATION_FAILED]:
            alert.critical(f'Критическая ошибка доступа: {err}')

        # Сохранение для анализа
        save_to_dlq(msg)
    else:
        metrics.increment('messages_sent')
```

### 3. Правильное использование poll()

```python
# poll() обязателен для:
# 1. Обработки callback'ов
# 2. Поддержания соединения с брокером
# 3. Обработки внутренних событий

for msg in messages:
    producer.produce(topic, value=msg)

    # Вызывайте poll() регулярно, даже если не ждёте callback
    producer.poll(0)

# В конце обязательно flush()
producer.flush()
```

### 4. Graceful shutdown

```python
import signal
import sys

running = True

def shutdown(signum, frame):
    global running
    print('Получен сигнал завершения, очищаем буфер...')
    running = False

signal.signal(signal.SIGTERM, shutdown)
signal.signal(signal.SIGINT, shutdown)

while running:
    producer.produce(topic, value=generate_message())
    producer.poll(0)

# Гарантируем отправку всех сообщений
remaining = producer.flush(timeout=30)
if remaining > 0:
    print(f'ВНИМАНИЕ: {remaining} сообщений не отправлено!')
sys.exit(0 if remaining == 0 else 1)
```

### 5. Мониторинг латентности

```python
import time

def timed_callback(start_time):
    def callback(err, msg):
        latency = (time.time() - start_time) * 1000
        if err:
            metrics.record('produce_error', 1)
        else:
            metrics.record('produce_latency_ms', latency)
    return callback

# Использование
start = time.time()
producer.produce(topic, value=msg, callback=timed_callback(start))
```

## Распространённые ошибки

### 1. Не вызывать poll() в цикле отправки

```python
# НЕПРАВИЛЬНО - callback'и не обрабатываются
for msg in messages:
    producer.produce(topic, value=msg, callback=my_callback)
# callback'и вызовутся только при flush()

# ПРАВИЛЬНО
for msg in messages:
    producer.produce(topic, value=msg, callback=my_callback)
    producer.poll(0)  # Обработка готовых callback'ов
producer.flush()
```

### 2. Синхронная отправка в цикле

```python
# НЕПРАВИЛЬНО - очень медленно
for msg in messages:
    producer.produce(topic, value=msg)
    producer.flush()  # Блокировка на каждом сообщении!

# ПРАВИЛЬНО - batch отправка
for msg in messages:
    producer.produce(topic, value=msg)
    producer.poll(0)
producer.flush()  # Один раз в конце
```

### 3. Игнорирование BufferError

```python
# НЕПРАВИЛЬНО
producer.produce(topic, value=large_message)  # Может бросить BufferError

# ПРАВИЛЬНО
while True:
    try:
        producer.produce(topic, value=large_message)
        break
    except BufferError:
        print('Буфер полон, ожидание...')
        producer.poll(1)  # Ждём освобождения
```

### 4. Блокировка в callback

```python
# НЕПРАВИЛЬНО - callback выполняется в потоке библиотеки!
def slow_callback(err, msg):
    if err:
        time.sleep(5)  # Блокирует все callback'и
        send_alert_to_slack(err)  # Сетевой вызов

# ПРАВИЛЬНО - быстрый callback, обработка отдельно
error_queue = queue.Queue()

def fast_callback(err, msg):
    if err:
        error_queue.put((err, msg))

# Обработка в отдельном потоке
def error_handler():
    while True:
        err, msg = error_queue.get()
        send_alert_to_slack(err)
```

### 5. Неправильное использование транзакций

```python
# НЕПРАВИЛЬНО - транзакция на каждое сообщение
for msg in messages:
    producer.begin_transaction()
    producer.produce(topic, value=msg)
    producer.commit_transaction()  # Очень медленно!

# ПРАВИЛЬНО - batch в одной транзакции
producer.begin_transaction()
for msg in messages:
    producer.produce(topic, value=msg)
producer.commit_transaction()  # Одна транзакция на batch
```

## Сравнение паттернов

| Паттерн | Производительность | Надёжность | Сложность |
|---------|-------------------|------------|-----------|
| Fire-and-Forget | Максимальная | Низкая | Минимальная |
| Синхронный | Низкая | Высокая | Низкая |
| Async + Callback | Высокая | Высокая | Средняя |
| Идемпотентный | Высокая | Очень высокая | Средняя |
| Транзакционный | Средняя | Максимальная | Высокая |

## Дополнительные ресурсы

- [Kafka Producer Best Practices](https://docs.confluent.io/platform/current/installation/configuration/producer-configs.html)
- [Idempotent Producer](https://kafka.apache.org/documentation/#producerconfigs_enable.idempotence)
- [Transactions in Kafka](https://kafka.apache.org/documentation/#semantics)
- [Exactly-Once Semantics](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/)

---

[prev: 02-producer-options](./02-producer-options.md) | [next: 01-example](../05-consumers/01-example.md)
