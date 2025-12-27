# Различные пакеты исходного кода

## Описание

Apache Kafka имеет модульную структуру и распространяется в нескольких формах. Понимание различных пакетов исходного кода и дистрибутивов поможет вам выбрать правильный вариант для вашего проекта и понять, как устроена экосистема Kafka.

## Официальные дистрибутивы

### 1. Apache Kafka (основной дистрибутив)

Официальный релиз от Apache Software Foundation.

```bash
# Скачивание с официального зеркала
wget https://downloads.apache.org/kafka/3.6.1/kafka_2.13-3.6.1.tgz

# Формат имени: kafka_{scala_version}-{kafka_version}.tgz
# kafka_2.13-3.6.1.tgz означает:
# - Scala версия: 2.13
# - Kafka версия: 3.6.1
```

**Структура архива:**

```
kafka_2.13-3.6.1/
├── bin/                    # Скрипты командной строки
│   ├── kafka-server-start.sh
│   ├── kafka-topics.sh
│   ├── kafka-console-producer.sh
│   ├── kafka-console-consumer.sh
│   ├── zookeeper-server-start.sh
│   └── windows/            # Windows-версии скриптов
├── config/                 # Файлы конфигурации
│   ├── server.properties
│   ├── zookeeper.properties
│   ├── producer.properties
│   └── consumer.properties
├── libs/                   # JAR-файлы (зависимости)
│   ├── kafka_2.13-3.6.1.jar
│   ├── kafka-clients-3.6.1.jar
│   ├── kafka-streams-3.6.1.jar
│   └── ... (множество зависимостей)
├── licenses/               # Лицензии
└── site-docs/              # Документация
```

### 2. Confluent Platform

Корпоративный дистрибутив от Confluent (компания основателей Kafka).

```bash
# Community Edition (бесплатная)
curl -O https://packages.confluent.io/archive/7.5/confluent-community-7.5.0.tar.gz

# Enterprise Edition (платная)
curl -O https://packages.confluent.io/archive/7.5/confluent-7.5.0.tar.gz
```

**Дополнительные компоненты Confluent:**

| Компонент | Описание | Community | Enterprise |
|-----------|----------|-----------|------------|
| Kafka | Ядро Kafka | Да | Да |
| Schema Registry | Управление схемами | Да | Да |
| REST Proxy | HTTP интерфейс | Да | Да |
| ksqlDB | SQL для потоков | Да | Да |
| Control Center | UI для управления | Нет | Да |
| Replicator | Репликация между кластерами | Нет | Да |

### 3. Amazon MSK (Managed Streaming for Kafka)

AWS-управляемый сервис Kafka.

```python
# Подключение к MSK
from kafka import KafkaProducer

producer = KafkaProducer(
    bootstrap_servers=[
        'b-1.mycluster.abc123.c1.kafka.us-east-1.amazonaws.com:9092',
        'b-2.mycluster.abc123.c1.kafka.us-east-1.amazonaws.com:9092'
    ],
    security_protocol='SASL_SSL',
    sasl_mechanism='AWS_MSK_IAM'
)
```

## Ключевые модули Kafka

### 1. kafka-clients

Основная клиентская библиотека для Java/Kotlin/Scala.

```xml
<!-- Maven -->
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>3.6.1</version>
</dependency>
```

```gradle
// Gradle
implementation 'org.apache.kafka:kafka-clients:3.6.1'
```

**Основные классы:**
- `KafkaProducer` — отправка сообщений
- `KafkaConsumer` — получение сообщений
- `AdminClient` — администрирование

### 2. kafka-streams

Библиотека для потоковой обработки данных.

```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-streams</artifactId>
    <version>3.6.1</version>
</dependency>
```

```java
// Пример Kafka Streams
StreamsBuilder builder = new StreamsBuilder();

KStream<String, String> source = builder.stream("input-topic");

source
    .filter((key, value) -> value.contains("important"))
    .mapValues(value -> value.toUpperCase())
    .to("output-topic");

KafkaStreams streams = new KafkaStreams(builder.build(), props);
streams.start();
```

### 3. connect-api

API для создания Kafka Connect коннекторов.

```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>connect-api</artifactId>
    <version>3.6.1</version>
</dependency>
```

```java
// Пример Source Connector
public class MySourceConnector extends SourceConnector {
    @Override
    public Class<? extends Task> taskClass() {
        return MySourceTask.class;
    }

    @Override
    public List<Map<String, String>> taskConfigs(int maxTasks) {
        // Конфигурация задач
    }
}
```

## Клиентские библиотеки для разных языков

### Python

```bash
# kafka-python — чистый Python
pip install kafka-python

# confluent-kafka — обертка над librdkafka (быстрее)
pip install confluent-kafka
```

```python
# kafka-python
from kafka import KafkaProducer, KafkaConsumer

# confluent-kafka
from confluent_kafka import Producer, Consumer
```

**Сравнение библиотек:**

| Характеристика | kafka-python | confluent-kafka |
|----------------|--------------|-----------------|
| Производительность | Средняя | Высокая |
| Зависимости | Нет (чистый Python) | librdkafka (C) |
| Установка | Простая | Требует компиляции |
| API | Похож на Java | Более низкоуровневый |

### Go

```bash
# confluent-kafka-go
go get github.com/confluentinc/confluent-kafka-go/kafka

# segmentio/kafka-go (чистый Go)
go get github.com/segmentio/kafka-go

# sarama (IBM)
go get github.com/IBM/sarama
```

```go
// confluent-kafka-go
import "github.com/confluentinc/confluent-kafka-go/kafka"

producer, _ := kafka.NewProducer(&kafka.ConfigMap{
    "bootstrap.servers": "localhost:9092",
})

// segmentio/kafka-go
import "github.com/segmentio/kafka-go"

writer := kafka.NewWriter(kafka.WriterConfig{
    Brokers: []string{"localhost:9092"},
    Topic:   "my-topic",
})
```

### Node.js

```bash
# kafkajs — современная библиотека
npm install kafkajs

# node-rdkafka — обертка над librdkafka
npm install node-rdkafka
```

```javascript
// kafkajs
const { Kafka } = require('kafkajs')

const kafka = new Kafka({
  clientId: 'my-app',
  brokers: ['localhost:9092']
})

const producer = kafka.producer()
await producer.connect()
await producer.send({
  topic: 'test-topic',
  messages: [{ value: 'Hello Kafka!' }]
})
```

### Rust

```bash
# rdkafka — обертка над librdkafka
cargo add rdkafka
```

```rust
use rdkafka::producer::{FutureProducer, FutureRecord};
use rdkafka::ClientConfig;

let producer: FutureProducer = ClientConfig::new()
    .set("bootstrap.servers", "localhost:9092")
    .create()
    .expect("Producer creation failed");

let record = FutureRecord::to("test-topic")
    .payload("Hello Kafka!")
    .key("key1");

producer.send(record, Duration::from_secs(0)).await;
```

### .NET / C#

```bash
# Confluent.Kafka
dotnet add package Confluent.Kafka
```

```csharp
using Confluent.Kafka;

var config = new ProducerConfig { BootstrapServers = "localhost:9092" };

using var producer = new ProducerBuilder<string, string>(config).Build();

var result = await producer.ProduceAsync("test-topic",
    new Message<string, string> { Key = "key1", Value = "Hello Kafka!" });
```

## Примеры кода

### Установка клиентских библиотек

```bash
# Python
pip install kafka-python confluent-kafka

# Проверка установки
python -c "from kafka import KafkaProducer; print('kafka-python OK')"
python -c "from confluent_kafka import Producer; print('confluent-kafka OK')"
```

### Сравнение API разных библиотек

```python
# ==================== kafka-python ====================
from kafka import KafkaProducer, KafkaConsumer
import json

# Producer
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

future = producer.send('my-topic', {'key': 'value'})
result = future.get(timeout=10)
print(f"Sent to partition {result.partition} offset {result.offset}")

# Consumer
consumer = KafkaConsumer(
    'my-topic',
    bootstrap_servers=['localhost:9092'],
    auto_offset_reset='earliest',
    group_id='my-group',
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

for message in consumer:
    print(f"Received: {message.value}")


# ==================== confluent-kafka ====================
from confluent_kafka import Producer, Consumer
import json

# Producer
producer = Producer({
    'bootstrap.servers': 'localhost:9092'
})

def delivery_callback(err, msg):
    if err:
        print(f'Delivery failed: {err}')
    else:
        print(f'Sent to {msg.topic()} [{msg.partition()}] @ {msg.offset()}')

producer.produce(
    'my-topic',
    key='key1',
    value=json.dumps({'key': 'value'}),
    callback=delivery_callback
)
producer.flush()

# Consumer
consumer = Consumer({
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'my-group',
    'auto.offset.reset': 'earliest'
})

consumer.subscribe(['my-topic'])

while True:
    msg = consumer.poll(1.0)
    if msg is None:
        continue
    if msg.error():
        print(f"Error: {msg.error()}")
        continue
    print(f"Received: {msg.value().decode('utf-8')}")
```

### Работа с Kafka Streams (Java)

```java
import org.apache.kafka.streams.*;
import org.apache.kafka.streams.kstream.*;
import org.apache.kafka.common.serialization.Serdes;
import java.util.Properties;

public class WordCount {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "wordcount-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();

        KStream<String, String> textLines = builder.stream("input-topic");

        KTable<String, Long> wordCounts = textLines
            .flatMapValues(line -> Arrays.asList(line.toLowerCase().split("\\W+")))
            .groupBy((key, word) -> word)
            .count();

        wordCounts.toStream().to("output-topic",
            Produced.with(Serdes.String(), Serdes.Long()));

        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();

        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```

## Сборка из исходников

### Клонирование и сборка Apache Kafka

```bash
# Клонирование репозитория
git clone https://github.com/apache/kafka.git
cd kafka

# Просмотр доступных версий
git tag | grep "^[0-9]"

# Переключение на конкретную версию
git checkout 3.6.1

# Сборка с помощью Gradle
./gradlew jar

# Сборка дистрибутива
./gradlew releaseTarGz

# Результат в: core/build/distributions/
```

### Структура исходного кода

```
kafka/
├── clients/                # Клиентские библиотеки (Java)
│   └── src/main/java/org/apache/kafka/clients/
│       ├── producer/       # KafkaProducer
│       ├── consumer/       # KafkaConsumer
│       └── admin/          # AdminClient
├── core/                   # Ядро брокера (Scala)
│   └── src/main/scala/kafka/
│       ├── server/         # Логика сервера
│       ├── log/            # Управление логами
│       └── network/        # Сетевой слой
├── streams/                # Kafka Streams
│   └── src/main/java/org/apache/kafka/streams/
├── connect/                # Kafka Connect
│   ├── api/                # Connect API
│   ├── runtime/            # Runtime коннекторов
│   └── transforms/         # Трансформации
├── tools/                  # CLI инструменты
├── raft/                   # KRaft реализация
└── metadata/               # Управление метаданными
```

## Версионирование и совместимость

### Версии Kafka

```
Формат: MAJOR.MINOR.PATCH

Примеры:
- 3.6.1 — текущая стабильная
- 3.5.2 — предыдущая стабильная
- 3.7.0 — следующий релиз

Изменения между версиями:
- MAJOR — несовместимые изменения API
- MINOR — новые функции, обратная совместимость
- PATCH — исправления багов
```

### Совместимость клиентов и брокеров

```
Правила совместимости:
- Клиенты могут быть СТАРШЕ брокера (с ограничениями)
- Клиенты могут быть НОВЕЕ брокера (лучше)
- Рекомендуется: версия клиента <= версия брокера

Таблица совместимости:
┌─────────────┬────────────────────────────────────┐
│ Client      │ Broker 2.8 │ Broker 3.0 │ Broker 3.6 │
├─────────────┼────────────┼────────────┼────────────┤
│ Client 2.8  │     ✓      │     ✓      │     ✓      │
│ Client 3.0  │     △      │     ✓      │     ✓      │
│ Client 3.6  │     △      │     △      │     ✓      │
└─────────────┴────────────┴────────────┴────────────┘
✓ = полная совместимость
△ = работает, но некоторые функции недоступны
```

## Best Practices

### 1. Выбор дистрибутива

```
Для разработки:
- Docker образы (bitnami/kafka, confluentinc/cp-kafka)
- Локальная установка из архива

Для production:
- Confluent Platform (если нужны дополнительные инструменты)
- Amazon MSK / Azure Event Hubs (управляемые сервисы)
- Apache Kafka (полный контроль)
```

### 2. Выбор клиентской библиотеки

```python
# Python: выбор библиотеки

# Используйте confluent-kafka когда:
# - Важна производительность
# - Нужны продвинутые функции (transactions, idempotence)
# - Готовы к сложной установке

# Используйте kafka-python когда:
# - Простота важнее производительности
# - Нужна кросс-платформенность
# - Хотите избежать C-зависимостей
```

### 3. Управление зависимостями

```xml
<!-- Фиксируйте версии в production -->
<properties>
    <kafka.version>3.6.1</kafka.version>
</properties>

<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>${kafka.version}</version>
</dependency>
```

### 4. Обновление версий

```bash
# Порядок обновления:
1. Обновите брокеры (rolling upgrade)
2. Обновите клиентов
3. Обновите inter.broker.protocol.version
4. Обновите log.message.format.version

# Проверка совместимости
kafka-broker-api-versions.sh --bootstrap-server localhost:9092
```

## Дополнительные ресурсы

- [Apache Kafka Downloads](https://kafka.apache.org/downloads)
- [Confluent Platform Downloads](https://www.confluent.io/installation/)
- [Kafka Clients](https://cwiki.apache.org/confluence/display/KAFKA/Clients)
- [librdkafka](https://github.com/edenhill/librdkafka)
