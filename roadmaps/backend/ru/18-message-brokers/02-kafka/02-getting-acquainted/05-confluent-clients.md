# Клиенты Confluent

## Описание

Confluent — компания, основанная создателями Apache Kafka, которая предоставляет коммерческую поддержку и дополнительные инструменты для работы с Kafka. Клиенты Confluent (Confluent Clients) — это набор высокопроизводительных клиентских библиотек для различных языков программирования, основанных на библиотеке librdkafka.

## Ключевые концепции

### librdkafka

**librdkafka** — это высокопроизводительная реализация клиента Kafka на языке C. Она является основой для большинства клиентов Confluent.

```
┌─────────────────────────────────────────────────────────────┐
│                     Приложение                               │
├─────────────────────────────────────────────────────────────┤
│  confluent-kafka-python │ confluent-kafka-go │ Confluent.Kafka │
├─────────────────────────────────────────────────────────────┤
│                       librdkafka (C)                         │
├─────────────────────────────────────────────────────────────┤
│                      Kafka Protocol                          │
└─────────────────────────────────────────────────────────────┘
```

**Преимущества librdkafka:**
- Высокая производительность (написана на C)
- Полная поддержка Kafka протокола
- Встроенная буферизация и батчинг
- Идемпотентный producer из коробки
- Транзакционная поддержка

### Семейство клиентов Confluent

| Язык | Пакет | Основа |
|------|-------|--------|
| Python | confluent-kafka | librdkafka |
| Go | confluent-kafka-go | librdkafka |
| .NET | Confluent.Kafka | librdkafka |
| C/C++ | librdkafka | — |
| Java | kafka-clients | Нативная реализация |
| Node.js | node-rdkafka | librdkafka |

## Примеры кода

### Python: confluent-kafka

#### Установка

```bash
pip install confluent-kafka

# С поддержкой Avro
pip install confluent-kafka[avro]

# С поддержкой Schema Registry
pip install confluent-kafka[schemaregistry]
```

#### Producer

```python
from confluent_kafka import Producer
import json
import socket

# Конфигурация
config = {
    'bootstrap.servers': 'localhost:9092',
    'client.id': socket.gethostname(),

    # Настройки производительности
    'linger.ms': 5,                    # Задержка перед отправкой batch
    'batch.size': 16384,               # Размер batch в байтах

    # Надежность
    'acks': 'all',                     # Ждать всех реплик
    'retries': 5,                      # Количество повторов
    'retry.backoff.ms': 500,           # Задержка между повторами

    # Идемпотентность
    'enable.idempotence': True,        # Гарантия exactly-once для producer

    # Сжатие
    'compression.type': 'lz4',         # lz4, snappy, gzip, zstd
}

producer = Producer(config)


def delivery_callback(err, msg):
    """Callback вызывается после доставки сообщения"""
    if err:
        print(f'Ошибка доставки: {err}')
    else:
        print(f'Доставлено в {msg.topic()} [{msg.partition()}] @ offset {msg.offset()}')


def send_message(topic, key, value):
    """Отправка сообщения"""
    try:
        producer.produce(
            topic=topic,
            key=key.encode('utf-8') if key else None,
            value=json.dumps(value).encode('utf-8'),
            callback=delivery_callback
        )
        # poll для обработки callbacks
        producer.poll(0)

    except BufferError:
        print('Буфер переполнен, ожидание...')
        producer.poll(1)
        producer.produce(topic, key=key, value=json.dumps(value))


# Отправка сообщений
for i in range(10):
    send_message('orders', f'order-{i}', {'id': i, 'amount': 100 + i})

# Ожидание доставки всех сообщений
producer.flush()
```

#### Consumer

```python
from confluent_kafka import Consumer, KafkaError, KafkaException
import json
import signal
import sys

# Конфигурация
config = {
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'order-processors',
    'client.id': 'order-processor-1',

    # Offset management
    'auto.offset.reset': 'earliest',     # earliest, latest, none
    'enable.auto.commit': False,          # Ручной commit

    # Производительность
    'fetch.min.bytes': 1,
    'fetch.wait.max.ms': 500,
    'max.poll.interval.ms': 300000,

    # Heartbeat
    'session.timeout.ms': 45000,
    'heartbeat.interval.ms': 3000,
}

consumer = Consumer(config)


class GracefulShutdown:
    """Обработчик graceful shutdown"""
    shutdown = False

    def __init__(self):
        signal.signal(signal.SIGINT, self.exit_gracefully)
        signal.signal(signal.SIGTERM, self.exit_gracefully)

    def exit_gracefully(self, signum, frame):
        print('Получен сигнал остановки')
        self.shutdown = True


def process_message(msg):
    """Обработка сообщения"""
    try:
        value = json.loads(msg.value().decode('utf-8'))
        print(f'Обработка заказа: {value}')
        return True
    except Exception as e:
        print(f'Ошибка обработки: {e}')
        return False


def main():
    shutdown = GracefulShutdown()

    # Подписка на топики
    consumer.subscribe(['orders'])

    try:
        while not shutdown.shutdown:
            # Poll с таймаутом
            msg = consumer.poll(timeout=1.0)

            if msg is None:
                continue

            if msg.error():
                if msg.error().code() == KafkaError._PARTITION_EOF:
                    # Достигнут конец партиции
                    print(f'Достигнут конец {msg.topic()} [{msg.partition()}]')
                elif msg.error():
                    raise KafkaException(msg.error())
            else:
                # Обработка сообщения
                if process_message(msg):
                    # Commit после успешной обработки
                    consumer.commit(asynchronous=False)

    except KeyboardInterrupt:
        pass

    finally:
        # Закрытие consumer
        consumer.close()
        print('Consumer остановлен')


if __name__ == '__main__':
    main()
```

#### Batch Consumer

```python
from confluent_kafka import Consumer, TopicPartition
import json

consumer = Consumer({
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'batch-processor',
    'auto.offset.reset': 'earliest',
    'enable.auto.commit': False,
})

consumer.subscribe(['orders'])

def process_batch(messages):
    """Обработка batch сообщений"""
    for msg in messages:
        value = json.loads(msg.value().decode('utf-8'))
        print(f'Batch item: {value}')
    return True

try:
    while True:
        # Получаем batch сообщений
        messages = consumer.consume(num_messages=100, timeout=1.0)

        if not messages:
            continue

        # Фильтруем ошибки
        valid_messages = [m for m in messages if m.error() is None]
        errors = [m for m in messages if m.error() is not None]

        for err in errors:
            print(f'Ошибка: {err.error()}')

        if valid_messages and process_batch(valid_messages):
            consumer.commit(asynchronous=False)

except KeyboardInterrupt:
    pass
finally:
    consumer.close()
```

### Go: confluent-kafka-go

#### Установка

```bash
go get github.com/confluentinc/confluent-kafka-go/v2/kafka
```

#### Producer

```go
package main

import (
    "encoding/json"
    "fmt"
    "github.com/confluentinc/confluent-kafka-go/v2/kafka"
)

type Order struct {
    ID     int     `json:"id"`
    Amount float64 `json:"amount"`
}

func main() {
    producer, err := kafka.NewProducer(&kafka.ConfigMap{
        "bootstrap.servers": "localhost:9092",
        "acks":              "all",
        "retries":           5,
        "linger.ms":         5,
        "compression.type":  "lz4",
    })
    if err != nil {
        panic(err)
    }
    defer producer.Close()

    // Goroutine для обработки delivery reports
    go func() {
        for e := range producer.Events() {
            switch ev := e.(type) {
            case *kafka.Message:
                if ev.TopicPartition.Error != nil {
                    fmt.Printf("Ошибка доставки: %v\n", ev.TopicPartition.Error)
                } else {
                    fmt.Printf("Доставлено в %v [%d] @ %d\n",
                        *ev.TopicPartition.Topic,
                        ev.TopicPartition.Partition,
                        ev.TopicPartition.Offset)
                }
            }
        }
    }()

    topic := "orders"

    for i := 0; i < 10; i++ {
        order := Order{ID: i, Amount: 100.0 + float64(i)}
        value, _ := json.Marshal(order)

        err := producer.Produce(&kafka.Message{
            TopicPartition: kafka.TopicPartition{Topic: &topic, Partition: kafka.PartitionAny},
            Key:            []byte(fmt.Sprintf("order-%d", i)),
            Value:          value,
        }, nil)

        if err != nil {
            fmt.Printf("Ошибка отправки: %v\n", err)
        }
    }

    // Ожидание доставки всех сообщений
    producer.Flush(15 * 1000)
}
```

#### Consumer

```go
package main

import (
    "encoding/json"
    "fmt"
    "os"
    "os/signal"
    "syscall"

    "github.com/confluentinc/confluent-kafka-go/v2/kafka"
)

func main() {
    consumer, err := kafka.NewConsumer(&kafka.ConfigMap{
        "bootstrap.servers": "localhost:9092",
        "group.id":          "order-processors-go",
        "auto.offset.reset": "earliest",
        "enable.auto.commit": false,
    })
    if err != nil {
        panic(err)
    }
    defer consumer.Close()

    err = consumer.SubscribeTopics([]string{"orders"}, nil)
    if err != nil {
        panic(err)
    }

    // Обработка сигналов
    sigchan := make(chan os.Signal, 1)
    signal.Notify(sigchan, syscall.SIGINT, syscall.SIGTERM)

    run := true
    for run {
        select {
        case sig := <-sigchan:
            fmt.Printf("Получен сигнал %v, завершение...\n", sig)
            run = false

        default:
            ev := consumer.Poll(100)
            if ev == nil {
                continue
            }

            switch e := ev.(type) {
            case *kafka.Message:
                var order map[string]interface{}
                json.Unmarshal(e.Value, &order)
                fmt.Printf("Получено: %v (partition %d, offset %d)\n",
                    order, e.TopicPartition.Partition, e.TopicPartition.Offset)

                // Commit offset
                consumer.CommitMessage(e)

            case kafka.Error:
                fmt.Printf("Ошибка: %v\n", e)
                if e.Code() == kafka.ErrAllBrokersDown {
                    run = false
                }
            }
        }
    }
}
```

### .NET: Confluent.Kafka

#### Установка

```bash
dotnet add package Confluent.Kafka
```

#### Producer

```csharp
using Confluent.Kafka;
using System.Text.Json;

public class KafkaProducerExample
{
    public static async Task Main()
    {
        var config = new ProducerConfig
        {
            BootstrapServers = "localhost:9092",
            Acks = Acks.All,
            EnableIdempotence = true,
            CompressionType = CompressionType.Lz4,
            LingerMs = 5,
            BatchSize = 16384
        };

        using var producer = new ProducerBuilder<string, string>(config)
            .SetErrorHandler((_, e) => Console.WriteLine($"Ошибка: {e.Reason}"))
            .Build();

        try
        {
            for (int i = 0; i < 10; i++)
            {
                var order = new { Id = i, Amount = 100 + i };
                var value = JsonSerializer.Serialize(order);

                var result = await producer.ProduceAsync("orders",
                    new Message<string, string>
                    {
                        Key = $"order-{i}",
                        Value = value
                    });

                Console.WriteLine($"Доставлено в {result.TopicPartitionOffset}");
            }
        }
        catch (ProduceException<string, string> e)
        {
            Console.WriteLine($"Ошибка отправки: {e.Error.Reason}");
        }

        producer.Flush(TimeSpan.FromSeconds(10));
    }
}
```

#### Consumer

```csharp
using Confluent.Kafka;
using System.Text.Json;

public class KafkaConsumerExample
{
    public static void Main()
    {
        var config = new ConsumerConfig
        {
            BootstrapServers = "localhost:9092",
            GroupId = "order-processors-dotnet",
            AutoOffsetReset = AutoOffsetReset.Earliest,
            EnableAutoCommit = false
        };

        using var consumer = new ConsumerBuilder<string, string>(config)
            .SetErrorHandler((_, e) => Console.WriteLine($"Ошибка: {e.Reason}"))
            .Build();

        consumer.Subscribe("orders");

        var cts = new CancellationTokenSource();
        Console.CancelKeyPress += (_, e) =>
        {
            e.Cancel = true;
            cts.Cancel();
        };

        try
        {
            while (true)
            {
                try
                {
                    var result = consumer.Consume(cts.Token);

                    Console.WriteLine(
                        $"Получено: {result.Message.Value} " +
                        $"(partition {result.Partition}, offset {result.Offset})");

                    consumer.Commit(result);
                }
                catch (ConsumeException e)
                {
                    Console.WriteLine($"Ошибка: {e.Error.Reason}");
                }
            }
        }
        catch (OperationCanceledException)
        {
            consumer.Close();
        }
    }
}
```

### Schema Registry

Confluent Schema Registry — сервис для управления схемами данных (Avro, JSON Schema, Protobuf).

#### Python с Schema Registry

```python
from confluent_kafka import Producer, Consumer
from confluent_kafka.serialization import SerializationContext, MessageField
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer, AvroDeserializer

# Схема Avro
order_schema = """
{
    "type": "record",
    "name": "Order",
    "namespace": "com.example",
    "fields": [
        {"name": "id", "type": "int"},
        {"name": "user_id", "type": "string"},
        {"name": "amount", "type": "double"},
        {"name": "created_at", "type": "string"}
    ]
}
"""

# Schema Registry клиент
schema_registry_client = SchemaRegistryClient({
    'url': 'http://localhost:8081'
})

# Сериализатор
avro_serializer = AvroSerializer(
    schema_registry_client,
    order_schema,
    lambda order, ctx: order  # to_dict function
)

# Producer с Avro
producer = Producer({
    'bootstrap.servers': 'localhost:9092'
})

def send_order(order):
    producer.produce(
        topic='orders-avro',
        key=str(order['id']),
        value=avro_serializer(
            order,
            SerializationContext('orders-avro', MessageField.VALUE)
        )
    )
    producer.flush()

# Десериализатор
avro_deserializer = AvroDeserializer(
    schema_registry_client,
    order_schema,
    lambda data, ctx: data  # from_dict function
)

# Consumer с Avro
consumer = Consumer({
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'avro-consumer',
    'auto.offset.reset': 'earliest'
})

consumer.subscribe(['orders-avro'])

while True:
    msg = consumer.poll(1.0)
    if msg is None:
        continue

    order = avro_deserializer(
        msg.value(),
        SerializationContext('orders-avro', MessageField.VALUE)
    )
    print(f'Получен заказ: {order}')
```

## Продвинутые возможности

### Транзакции

```python
from confluent_kafka import Producer

producer = Producer({
    'bootstrap.servers': 'localhost:9092',
    'transactional.id': 'my-transaction-id',
    'enable.idempotence': True
})

# Инициализация транзакций
producer.init_transactions()

try:
    # Начало транзакции
    producer.begin_transaction()

    # Отправка сообщений
    producer.produce('topic1', key='key1', value='value1')
    producer.produce('topic2', key='key2', value='value2')

    # Commit транзакции
    producer.commit_transaction()

except Exception as e:
    # Abort при ошибке
    producer.abort_transaction()
    raise e
```

### Админ-клиент

```python
from confluent_kafka.admin import AdminClient, NewTopic, ConfigResource

admin = AdminClient({
    'bootstrap.servers': 'localhost:9092'
})

# Создание топиков
new_topics = [
    NewTopic('new-topic-1', num_partitions=3, replication_factor=1),
    NewTopic('new-topic-2', num_partitions=6, replication_factor=1,
             config={'retention.ms': '86400000'})
]

futures = admin.create_topics(new_topics)
for topic, future in futures.items():
    try:
        future.result()
        print(f'Топик {topic} создан')
    except Exception as e:
        print(f'Ошибка создания {topic}: {e}')

# Описание топиков
metadata = admin.list_topics(timeout=10)
for topic in metadata.topics.values():
    print(f'{topic.topic}: {len(topic.partitions)} партиций')

# Изменение конфигурации
config_resource = ConfigResource(ConfigResource.Type.TOPIC, 'my-topic')
futures = admin.describe_configs([config_resource])

for resource, future in futures.items():
    configs = future.result()
    for config in configs.values():
        print(f'{config.name}: {config.value}')
```

## Best Practices

### 1. Конфигурация Producer

```python
# Production конфигурация
production_config = {
    'bootstrap.servers': 'kafka1:9092,kafka2:9092,kafka3:9092',

    # Надежность
    'acks': 'all',
    'enable.idempotence': True,
    'max.in.flight.requests.per.connection': 5,
    'retries': 2147483647,  # MAX_INT
    'retry.backoff.ms': 100,

    # Производительность
    'linger.ms': 5,
    'batch.size': 32768,
    'compression.type': 'lz4',
    'buffer.memory': 67108864,

    # Мониторинг
    'statistics.interval.ms': 60000,
}
```

### 2. Конфигурация Consumer

```python
# Production конфигурация
production_config = {
    'bootstrap.servers': 'kafka1:9092,kafka2:9092,kafka3:9092',
    'group.id': 'my-consumer-group',

    # Offset management
    'enable.auto.commit': False,
    'auto.offset.reset': 'earliest',

    # Session management
    'session.timeout.ms': 45000,
    'heartbeat.interval.ms': 3000,
    'max.poll.interval.ms': 300000,

    # Производительность
    'fetch.min.bytes': 1,
    'fetch.max.wait.ms': 500,
    'max.partition.fetch.bytes': 1048576,
}
```

### 3. Обработка ошибок

```python
from confluent_kafka import KafkaError

def handle_error(err):
    """Обработка ошибок Kafka"""
    if err.code() == KafkaError._ALL_BROKERS_DOWN:
        print('Все брокеры недоступны')
        return False
    elif err.code() == KafkaError._TIMED_OUT:
        print('Timeout')
        return True  # Можно повторить
    elif err.code() == KafkaError.UNKNOWN_TOPIC_OR_PART:
        print('Топик не найден')
        return False
    else:
        print(f'Ошибка: {err}')
        return err.retriable()
```

### 4. Мониторинг

```python
def stats_callback(stats_json):
    """Callback для статистики"""
    import json
    stats = json.loads(stats_json)

    print(f"Сообщений отправлено: {stats.get('txmsgs', 0)}")
    print(f"Байт отправлено: {stats.get('txmsg_bytes', 0)}")

    # Статистика по брокерам
    for broker_id, broker_stats in stats.get('brokers', {}).items():
        print(f"Broker {broker_id}: RTT={broker_stats.get('rtt', {}).get('avg', 0)}ms")

producer = Producer({
    'bootstrap.servers': 'localhost:9092',
    'statistics.interval.ms': 10000,
    'stats_cb': stats_callback
})
```

## Дополнительные ресурсы

- [Confluent Python Client](https://docs.confluent.io/kafka-clients/python/current/overview.html)
- [Confluent Go Client](https://docs.confluent.io/kafka-clients/go/current/overview.html)
- [Confluent .NET Client](https://docs.confluent.io/kafka-clients/dotnet/current/overview.html)
- [librdkafka Configuration](https://github.com/edenhill/librdkafka/blob/master/CONFIGURATION.md)
- [Schema Registry](https://docs.confluent.io/platform/current/schema-registry/index.html)
