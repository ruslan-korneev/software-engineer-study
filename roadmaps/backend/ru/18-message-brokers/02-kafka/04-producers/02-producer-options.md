# Конфигурация Producer - Параметры производителя

[prev: 01-example](./01-example.md) | [next: 03-code-generation](./03-code-generation.md)

---

## Описание

Конфигурация Kafka Producer определяет поведение отправки сообщений, включая надёжность доставки, производительность, обработку ошибок и использование ресурсов. Правильная настройка параметров критически важна для достижения баланса между производительностью и гарантиями доставки.

Параметры Producer можно разделить на несколько категорий:
- **Гарантии доставки** (acks, retries, enable.idempotence)
- **Производительность** (batch.size, linger.ms, buffer.memory)
- **Сжатие** (compression.type)
- **Таймауты** (request.timeout.ms, delivery.timeout.ms)
- **Сетевые настройки** (max.in.flight.requests.per.connection)

## Ключевые концепции

### Параметр acks (acknowledgments)

Определяет количество подтверждений от брокеров, необходимых для считывания записи успешной:

| Значение | Описание | Надёжность | Производительность |
|----------|----------|------------|-------------------|
| `acks=0` | Не ждать подтверждения | Низкая (возможна потеря) | Максимальная |
| `acks=1` | Подтверждение от лидера | Средняя | Высокая |
| `acks=all` / `acks=-1` | Подтверждение от всех ISR | Максимальная | Ниже |

```
acks=0:  Producer ──▶ Leader (нет ответа)
                     ↓
                   Follower

acks=1:  Producer ──▶ Leader ──▶ ACK
                     ↓
                   Follower (возможно отстаёт)

acks=all: Producer ──▶ Leader
                      ↓ (репликация)
                    Follower
                      ↓
                    ACK (после репликации)
```

### Параметр retries

Количество повторных попыток отправки при сбое:

- **retries** — максимальное число повторов (по умолчанию: 2147483647 в новых версиях)
- **retry.backoff.ms** — задержка между повторами (по умолчанию: 100ms)
- **delivery.timeout.ms** — общий таймаут доставки (по умолчанию: 120000ms)

### Батчинг (пакетная отправка)

Producer накапливает сообщения в пакеты для эффективной отправки:

- **batch.size** — максимальный размер пакета в байтах
- **linger.ms** — время ожидания накопления пакета
- **buffer.memory** — общий объём памяти для буферизации

```
linger.ms = 0:      ┌─────┐──────────────▶ Broker
                    │msg1 │ (сразу)

linger.ms = 100:    ┌─────┬─────┬─────┐──▶ Broker
                    │msg1 │msg2 │msg3 │ (через 100ms или при заполнении batch.size)
                    └─────┴─────┴─────┘
```

### Сжатие (compression)

Сжатие уменьшает объём передаваемых данных:

| Тип | Степень сжатия | CPU нагрузка | Скорость |
|-----|----------------|--------------|----------|
| `none` | 0% | Нет | Максимальная |
| `gzip` | Высокая (~70%) | Высокая | Низкая |
| `snappy` | Средняя (~50%) | Низкая | Высокая |
| `lz4` | Средняя (~55%) | Низкая | Очень высокая |
| `zstd` | Высокая (~65%) | Средняя | Высокая |

## Примеры кода

### Python (confluent-kafka) — полная конфигурация

```python
from confluent_kafka import Producer

# Конфигурация для высокой надёжности
reliable_config = {
    'bootstrap.servers': 'localhost:9092',

    # Гарантии доставки
    'acks': 'all',                      # Ждать подтверждения от всех реплик
    'enable.idempotence': True,         # Включить идемпотентность
    'max.in.flight.requests.per.connection': 5,  # Макс. запросов без ответа

    # Повторные попытки
    'retries': 10,                      # Количество повторов
    'retry.backoff.ms': 100,            # Задержка между повторами
    'delivery.timeout.ms': 120000,      # Общий таймаут доставки (2 минуты)

    # Батчинг
    'batch.size': 16384,                # 16 KB на пакет
    'linger.ms': 5,                     # Ждать 5ms для накопления пакета
    'buffer.memory': 33554432,          # 32 MB буфер

    # Сжатие
    'compression.type': 'lz4',          # Быстрое сжатие

    # Сеть
    'request.timeout.ms': 30000,        # Таймаут запроса (30 сек)
    'socket.timeout.ms': 60000,         # Таймаут сокета

    # Идентификация
    'client.id': 'my-reliable-producer'
}

producer = Producer(reliable_config)
```

### Python (kafka-python) — конфигурация

```python
from kafka import KafkaProducer
import json

# Конфигурация для высокой пропускной способности
high_throughput_config = {
    'bootstrap_servers': ['localhost:9092'],

    # Сериализация
    'key_serializer': lambda k: k.encode('utf-8') if k else None,
    'value_serializer': lambda v: json.dumps(v).encode('utf-8'),

    # Гарантии доставки
    'acks': 1,                          # Только лидер

    # Батчинг для производительности
    'batch_size': 32768,                # 32 KB
    'linger_ms': 50,                    # 50ms ожидания
    'buffer_memory': 67108864,          # 64 MB

    # Сжатие
    'compression_type': 'snappy',

    # Повторы
    'retries': 5,
    'retry_backoff_ms': 100,

    # Размер сообщений
    'max_request_size': 1048576,        # 1 MB
}

producer = KafkaProducer(**high_throughput_config)
```

### Java — подробная конфигурация

```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;

public class ConfiguredProducer {

    public static Properties getReliableConfig() {
        Properties props = new Properties();

        // Базовые настройки
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.put(ProducerConfig.CLIENT_ID_CONFIG, "reliable-producer");

        // Гарантии доставки
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);

        // Повторные попытки
        props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
        props.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, 100);
        props.put(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, 120000);

        // Ограничение для сохранения порядка
        props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);

        // Батчинг
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        props.put(ProducerConfig.LINGER_MS_CONFIG, 5);
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);

        // Сжатие
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");

        return props;
    }

    public static Properties getHighThroughputConfig() {
        Properties props = new Properties();

        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        // Производительность важнее надёжности
        props.put(ProducerConfig.ACKS_CONFIG, "1");

        // Большие пакеты, больше ожидания
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 65536);      // 64 KB
        props.put(ProducerConfig.LINGER_MS_CONFIG, 100);         // 100ms
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 134217728); // 128 MB

        // Агрессивное сжатие
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "zstd");

        return props;
    }
}
```

### Сравнение конфигураций для разных сценариев

```python
# Сценарий 1: Логирование (можно потерять некоторые сообщения)
logging_config = {
    'bootstrap.servers': 'localhost:9092',
    'acks': '0',                        # Не ждать подтверждения
    'batch.size': 65536,                # Большие пакеты
    'linger.ms': 100,                   # Накапливать 100ms
    'compression.type': 'snappy',
    'buffer.memory': 134217728,         # 128 MB
}

# Сценарий 2: Финансовые транзакции (потеря недопустима)
financial_config = {
    'bootstrap.servers': 'localhost:9092',
    'acks': 'all',                      # Все реплики
    'enable.idempotence': True,         # Идемпотентность
    'max.in.flight.requests.per.connection': 1,  # Строгий порядок
    'retries': 2147483647,              # Максимум повторов
    'batch.size': 16384,                # Меньшие пакеты
    'linger.ms': 0,                     # Отправлять сразу
    'compression.type': 'none',         # Без сжатия для скорости
}

# Сценарий 3: IoT данные (много мелких сообщений)
iot_config = {
    'bootstrap.servers': 'localhost:9092',
    'acks': '1',
    'batch.size': 32768,
    'linger.ms': 50,                    # Группировать сообщения
    'compression.type': 'lz4',          # Быстрое сжатие
    'buffer.memory': 67108864,          # 64 MB
}
```

## Best Practices

### 1. Выбор acks в зависимости от требований

```python
# Для критичных данных
'acks': 'all'

# Совместно с min.insync.replicas на брокере
# Например: min.insync.replicas=2 + acks=all = минимум 2 реплики подтвердят
```

### 2. Настройка батчинга для вашей нагрузки

```python
# Много мелких сообщений - увеличьте linger.ms
'linger.ms': 50,
'batch.size': 65536,

# Редкие большие сообщения - уменьшите ожидание
'linger.ms': 5,
'batch.size': 16384,
```

### 3. Мониторинг ключевых метрик

```python
from confluent_kafka import Producer

producer = Producer(config)

# Периодически проверяйте метрики
def log_metrics():
    metrics = producer.list_topics().topics
    # Также используйте внешний мониторинг (Prometheus, Grafana)
```

### 4. Идемпотентность для exactly-once семантики

```python
idempotent_config = {
    'bootstrap.servers': 'localhost:9092',
    'enable.idempotence': True,  # Автоматически включает:
                                  # - acks=all
                                  # - retries=MAX_INT
                                  # - max.in.flight.requests.per.connection=5
}
```

### 5. Правильное использование буфера памяти

```python
# buffer.memory должен быть достаточным для пиковой нагрузки
# Если буфер заполнен, send() заблокируется на max.block.ms

config = {
    'buffer.memory': 67108864,      # 64 MB
    'max.block.ms': 60000,          # Блокировка до 60 сек
}
```

## Распространённые ошибки

### 1. acks=all без настройки min.insync.replicas

```python
# НЕПРАВИЛЬНО - если есть только 1 реплика, acks=all = acks=1
'acks': 'all'  # На брокере min.insync.replicas=1 (по умолчанию)

# ПРАВИЛЬНО - настройте на брокере
# min.insync.replicas=2 (или больше)
# replication.factor=3
```

### 2. Слишком маленький delivery.timeout.ms

```python
# НЕПРАВИЛЬНО - не хватит времени на повторы
'delivery.timeout.ms': 5000,
'retries': 10,
'retry.backoff.ms': 1000  # 10 * 1000 = 10 сек > 5 сек

# ПРАВИЛЬНО
'delivery.timeout.ms': 120000,  # 2 минуты
```

### 3. Игнорирование порядка сообщений при повторах

```python
# ПРОБЛЕМА: max.in.flight.requests.per.connection > 1 может нарушить порядок
# Если запрос 1 провалился, а запрос 2 успешен, порядок нарушен

# Решение 1: Ограничить in-flight запросы
'max.in.flight.requests.per.connection': 1

# Решение 2: Использовать идемпотентность (рекомендуется)
'enable.idempotence': True  # Гарантирует порядок при max.in.flight=5
```

### 4. Слишком агрессивное сжатие для мелких сообщений

```python
# НЕПРАВИЛЬНО - сжатие мелких сообщений может увеличить размер
'compression.type': 'gzip',
'batch.size': 1024  # Маленький пакет

# ПРАВИЛЬНО - накапливайте больше перед сжатием
'compression.type': 'lz4',
'batch.size': 32768,
'linger.ms': 20
```

### 5. Не учитывать влияние на брокеры

```python
# НЕПРАВИЛЬНО - слишком большие пакеты могут перегрузить брокер
'batch.size': 10485760,  # 10 MB
'linger.ms': 1000        # 1 секунда

# Проверьте на брокере:
# - message.max.bytes
# - replica.fetch.max.bytes
```

## Таблица ключевых параметров

| Параметр | По умолчанию | Описание |
|----------|--------------|----------|
| `acks` | `all` (новые версии) | Уровень подтверждения |
| `retries` | `2147483647` | Количество повторов |
| `retry.backoff.ms` | `100` | Задержка между повторами |
| `delivery.timeout.ms` | `120000` | Общий таймаут доставки |
| `batch.size` | `16384` | Размер пакета (байты) |
| `linger.ms` | `0` | Ожидание накопления |
| `buffer.memory` | `33554432` | Память для буфера |
| `compression.type` | `none` | Тип сжатия |
| `max.in.flight.requests.per.connection` | `5` | Параллельные запросы |
| `enable.idempotence` | `true` (новые версии) | Идемпотентность |
| `max.block.ms` | `60000` | Таймаут блокировки send() |
| `request.timeout.ms` | `30000` | Таймаут запроса к брокеру |

## Дополнительные ресурсы

- [Kafka Producer Configs](https://kafka.apache.org/documentation/#producerconfigs)
- [Kafka: The Definitive Guide](https://www.confluent.io/resources/kafka-the-definitive-guide/)
- [Optimizing Kafka Producers](https://www.confluent.io/blog/optimizing-apache-kafka-producers/)

---

[prev: 01-example](./01-example.md) | [next: 03-code-generation](./03-code-generation.md)
