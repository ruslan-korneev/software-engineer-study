# Producer API - Основы API производителя

[prev: 03-data-format](../03-project-development/03-data-format.md) | [next: 02-producer-options](./02-producer-options.md)

---

## Описание

Kafka Producer — это клиентское приложение, которое публикует (записывает) сообщения в топики Kafka. Producer API предоставляет интерфейс для отправки записей в кластер Kafka. Каждая запись состоит из ключа, значения и опциональных заголовков.

Producer отвечает за:
- Сериализацию данных перед отправкой
- Определение партиции, в которую будет записано сообщение
- Буферизацию и пакетную отправку сообщений
- Обработку ошибок и повторные попытки

## Ключевые концепции

### Структура сообщения (ProducerRecord)

Сообщение в Kafka состоит из следующих компонентов:

| Поле | Обязательное | Описание |
|------|--------------|----------|
| topic | Да | Имя топика для отправки |
| value | Да | Содержимое сообщения |
| key | Нет | Ключ для партиционирования |
| partition | Нет | Номер партиции (если не указан, определяется автоматически) |
| timestamp | Нет | Временная метка (если не указана, используется текущее время) |
| headers | Нет | Метаданные в формате ключ-значение |

### Жизненный цикл Producer

1. **Создание** — инициализация с конфигурацией
2. **Отправка** — вызов метода send()
3. **Буферизация** — накопление сообщений в пакеты
4. **Отправка в сеть** — фоновый поток отправляет пакеты брокерам
5. **Подтверждение** — получение ack от брокеров
6. **Закрытие** — освобождение ресурсов

### Партиционирование

Выбор партиции происходит по следующей логике:
- Если указана партиция явно — используется она
- Если есть ключ — хеш ключа определяет партицию
- Если ключа нет — используется round-robin или sticky partitioning

## Примеры кода

### Python (kafka-python)

```python
from kafka import KafkaProducer
import json

# Создание Producer с базовой конфигурацией
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    key_serializer=lambda k: k.encode('utf-8') if k else None,
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Простая отправка сообщения
producer.send('my-topic', value={'message': 'Hello, Kafka!'})

# Отправка с ключом (гарантирует порядок для одного ключа)
producer.send(
    'user-events',
    key='user-123',
    value={'event': 'login', 'timestamp': '2024-01-15T10:30:00'}
)

# Отправка в конкретную партицию
producer.send(
    'my-topic',
    partition=0,
    value={'message': 'Точно в партицию 0'}
)

# Отправка с заголовками
producer.send(
    'my-topic',
    value={'data': 'payload'},
    headers=[
        ('content-type', b'application/json'),
        ('source', b'my-service')
    ]
)

# Обязательно дождаться отправки всех сообщений перед закрытием
producer.flush()
producer.close()
```

### Python (confluent-kafka) — рекомендуемая библиотека

```python
from confluent_kafka import Producer
import json

# Конфигурация Producer
config = {
    'bootstrap.servers': 'localhost:9092',
    'client.id': 'my-producer'
}

producer = Producer(config)

# Callback для обработки результата отправки
def delivery_callback(err, msg):
    if err:
        print(f'Ошибка доставки: {err}')
    else:
        print(f'Сообщение доставлено в {msg.topic()} [{msg.partition()}] @ {msg.offset()}')

# Отправка сообщения
producer.produce(
    topic='my-topic',
    key='my-key',
    value=json.dumps({'message': 'Hello!'}).encode('utf-8'),
    callback=delivery_callback
)

# Важно: poll() обрабатывает callback'и
producer.poll(0)

# Дождаться отправки всех сообщений
producer.flush()
```

### Java

```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;

public class SimpleProducer {
    public static void main(String[] args) {
        // Конфигурация Producer
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.put(ProducerConfig.CLIENT_ID_CONFIG, "my-producer");

        // Создание Producer
        try (KafkaProducer<String, String> producer = new KafkaProducer<>(props)) {

            // Создание записи
            ProducerRecord<String, String> record = new ProducerRecord<>(
                "my-topic",           // topic
                "user-123",           // key
                "{\"event\":\"login\"}" // value
            );

            // Синхронная отправка (блокирующая)
            RecordMetadata metadata = producer.send(record).get();
            System.out.printf("Отправлено в partition %d, offset %d%n",
                metadata.partition(), metadata.offset());

            // Асинхронная отправка с callback
            producer.send(record, (meta, exception) -> {
                if (exception != null) {
                    exception.printStackTrace();
                } else {
                    System.out.printf("Успешно: partition=%d, offset=%d%n",
                        meta.partition(), meta.offset());
                }
            });

            // Дождаться отправки всех сообщений
            producer.flush();
        }
    }
}
```

### Пример с заголовками (Java)

```java
import org.apache.kafka.common.header.Headers;
import org.apache.kafka.common.header.internals.RecordHeaders;

// Создание заголовков
Headers headers = new RecordHeaders();
headers.add("content-type", "application/json".getBytes());
headers.add("correlation-id", "abc-123".getBytes());

// Запись с заголовками
ProducerRecord<String, String> record = new ProducerRecord<>(
    "my-topic",    // topic
    null,          // partition (null = автоматически)
    System.currentTimeMillis(), // timestamp
    "my-key",      // key
    "{\"data\":\"value\"}", // value
    headers        // headers
);

producer.send(record);
```

## Best Practices

### 1. Всегда закрывайте Producer

```python
# Python - используйте context manager
from contextlib import contextmanager

@contextmanager
def kafka_producer(**config):
    producer = KafkaProducer(**config)
    try:
        yield producer
    finally:
        producer.flush()
        producer.close()

# Использование
with kafka_producer(bootstrap_servers=['localhost:9092']) as producer:
    producer.send('topic', value=b'message')
```

```java
// Java - используйте try-with-resources
try (KafkaProducer<String, String> producer = new KafkaProducer<>(props)) {
    // работа с producer
}
```

### 2. Используйте ключи для упорядоченности

```python
# Все сообщения одного пользователя попадут в одну партицию
# и будут обработаны в правильном порядке
producer.send('user-events', key='user-123', value={'action': 'first'})
producer.send('user-events', key='user-123', value={'action': 'second'})
```

### 3. Переиспользуйте Producer

```python
# ПЛОХО - создание нового producer на каждое сообщение
for event in events:
    producer = KafkaProducer(bootstrap_servers=['localhost:9092'])
    producer.send('topic', value=event)
    producer.close()

# ХОРОШО - один producer для всех сообщений
producer = KafkaProducer(bootstrap_servers=['localhost:9092'])
for event in events:
    producer.send('topic', value=event)
producer.flush()
producer.close()
```

### 4. Используйте сериализаторы

```python
# JSON сериализатор
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Avro сериализатор (для схемной валидации)
from confluent_kafka.schema_registry.avro import AvroSerializer

avro_serializer = AvroSerializer(schema_registry_client, schema_str)
```

### 5. Мониторинг метрик

```java
// Получение метрик Producer
Map<MetricName, ? extends Metric> metrics = producer.metrics();
for (Map.Entry<MetricName, ? extends Metric> entry : metrics.entrySet()) {
    if (entry.getKey().name().contains("record-send")) {
        System.out.printf("%s: %s%n", entry.getKey().name(), entry.getValue().metricValue());
    }
}
```

## Распространённые ошибки

### 1. Забыть вызвать flush() или close()

```python
# НЕПРАВИЛЬНО - сообщения могут не отправиться
producer = KafkaProducer(bootstrap_servers=['localhost:9092'])
producer.send('topic', value=b'message')
# Программа завершается, буфер не сброшен

# ПРАВИЛЬНО
producer = KafkaProducer(bootstrap_servers=['localhost:9092'])
producer.send('topic', value=b'message')
producer.flush()  # Дождаться отправки
producer.close()  # Освободить ресурсы
```

### 2. Не обрабатывать исключения при отправке

```python
from kafka.errors import KafkaError

try:
    future = producer.send('topic', value=b'message')
    # Синхронное ожидание с таймаутом
    record_metadata = future.get(timeout=10)
except KafkaError as e:
    print(f'Ошибка отправки: {e}')
```

### 3. Использовать слишком много партиций с одним Producer

```python
# Если партиций очень много, рассмотрите несколько Producer
# или используйте partitioner для балансировки нагрузки
```

### 4. Отправка слишком больших сообщений

```python
# Проверьте max.request.size на Producer
# и message.max.bytes на брокере
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    max_request_size=10485760  # 10 MB
)
```

### 5. Неправильная сериализация ключей

```python
# НЕПРАВИЛЬНО - ключ None при ожидании строки
producer.send('topic', key=None, value=b'data')
# Это отправит в случайную партицию каждый раз

# ПРАВИЛЬНО - используйте осмысленный ключ
producer.send('topic', key='consistent-key', value=b'data')
```

## Архитектура Producer

```
┌─────────────────────────────────────────────────────────────┐
│                      Kafka Producer                         │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐    ┌──────────────┐    ┌───────────────┐  │
│  │ Application │───▶│  Serializer  │───▶│ Partitioner   │  │
│  │   Thread    │    │  (key/value) │    │               │  │
│  └─────────────┘    └──────────────┘    └───────┬───────┘  │
│                                                  │          │
│                           ┌──────────────────────▼────────┐ │
│                           │     Record Accumulator        │ │
│                           │  ┌─────┐ ┌─────┐ ┌─────┐     │ │
│                           │  │Batch│ │Batch│ │Batch│     │ │
│                           │  │ P0  │ │ P1  │ │ P2  │     │ │
│                           │  └─────┘ └─────┘ └─────┘     │ │
│                           └──────────────────────┬────────┘ │
│                                                  │          │
│  ┌─────────────┐                                 │          │
│  │   Sender    │◀────────────────────────────────┘          │
│  │   Thread    │                                            │
│  └──────┬──────┘                                            │
└─────────┼───────────────────────────────────────────────────┘
          │
          ▼
    ┌───────────┐
    │  Broker   │
    └───────────┘
```

## Дополнительные ресурсы

- [Официальная документация Kafka Producer](https://kafka.apache.org/documentation/#producerapi)
- [confluent-kafka-python](https://github.com/confluentinc/confluent-kafka-python)
- [kafka-python](https://kafka-python.readthedocs.io/)

---

[prev: 03-data-format](../03-project-development/03-data-format.md) | [next: 02-producer-options](./02-producer-options.md)
