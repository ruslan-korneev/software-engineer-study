# Отправка и прием сообщения

## Описание

Отправка и прием сообщений — это базовые операции при работе с Apache Kafka. Прежде чем погружаться в сложные концепции, важно понять, как настроить окружение и выполнить первые операции с брокером сообщений.

## Установка Kafka

### Локальная установка

#### Предварительные требования
- Java 8+ (рекомендуется Java 11 или 17)
- Минимум 4 ГБ оперативной памяти

#### Шаг 1: Скачивание Kafka

```bash
# Скачиваем последнюю версию Kafka
wget https://downloads.apache.org/kafka/3.6.1/kafka_2.13-3.6.1.tgz

# Распаковываем архив
tar -xzf kafka_2.13-3.6.1.tgz
cd kafka_2.13-3.6.1
```

#### Шаг 2: Запуск ZooKeeper (для версий до 3.3)

```bash
# Запуск ZooKeeper в отдельном терминале
bin/zookeeper-server-start.sh config/zookeeper.properties
```

#### Шаг 3: Запуск Kafka Broker

```bash
# Запуск Kafka в отдельном терминале
bin/kafka-server-start.sh config/server.properties
```

### Установка через Docker

Docker — наиболее удобный способ для разработки и тестирования.

#### Docker Compose (рекомендуемый способ)

```yaml
# docker-compose.yml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    hostname: kafka
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "29092:29092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
```

```bash
# Запуск
docker-compose up -d

# Проверка статуса
docker-compose ps
```

#### KRaft режим (без ZooKeeper, Kafka 3.3+)

```yaml
# docker-compose-kraft.yml
version: '3.8'

services:
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    hostname: kafka
    container_name: kafka-kraft
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092'
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka:29093'
      KAFKA_LISTENERS: 'PLAINTEXT://kafka:29092,CONTROLLER://kafka:29093,PLAINTEXT_HOST://0.0.0.0:9092'
      KAFKA_INTER_BROKER_LISTENER_NAME: 'PLAINTEXT'
      KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
      CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'
```

## CLI инструменты

### kafka-topics.sh — управление топиками

```bash
# Создание топика
bin/kafka-topics.sh --create \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1

# Список всех топиков
bin/kafka-topics.sh --list \
  --bootstrap-server localhost:9092

# Информация о топике
bin/kafka-topics.sh --describe \
  --topic my-first-topic \
  --bootstrap-server localhost:9092

# Удаление топика
bin/kafka-topics.sh --delete \
  --topic my-first-topic \
  --bootstrap-server localhost:9092

# Изменение количества партиций (только увеличение)
bin/kafka-topics.sh --alter \
  --topic my-first-topic \
  --partitions 5 \
  --bootstrap-server localhost:9092
```

### kafka-console-producer.sh — отправка сообщений

```bash
# Простой producer
bin/kafka-console-producer.sh \
  --topic my-first-topic \
  --bootstrap-server localhost:9092

# Producer с ключами сообщений
bin/kafka-console-producer.sh \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --property "parse.key=true" \
  --property "key.separator=:"

# Пример ввода с ключами:
# key1:value1
# key2:value2

# Producer с подтверждением записи
bin/kafka-console-producer.sh \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --producer-property acks=all
```

### kafka-console-consumer.sh — получение сообщений

```bash
# Чтение новых сообщений
bin/kafka-console-consumer.sh \
  --topic my-first-topic \
  --bootstrap-server localhost:9092

# Чтение с начала топика
bin/kafka-console-consumer.sh \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --from-beginning

# Чтение с отображением ключей
bin/kafka-console-consumer.sh \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --from-beginning \
  --property print.key=true \
  --property key.separator=":"

# Чтение определенного количества сообщений
bin/kafka-console-consumer.sh \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --from-beginning \
  --max-messages 10

# Чтение с указанием consumer group
bin/kafka-console-consumer.sh \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group
```

## Отправка первых сообщений

### Практическое упражнение

#### Шаг 1: Создаем топик

```bash
bin/kafka-topics.sh --create \
  --topic test-messages \
  --bootstrap-server localhost:9092 \
  --partitions 1 \
  --replication-factor 1
```

#### Шаг 2: Открываем consumer в первом терминале

```bash
bin/kafka-console-consumer.sh \
  --topic test-messages \
  --bootstrap-server localhost:9092
```

#### Шаг 3: Открываем producer во втором терминале

```bash
bin/kafka-console-producer.sh \
  --topic test-messages \
  --bootstrap-server localhost:9092
```

#### Шаг 4: Отправляем сообщения

Введите в терминале producer:
```
Hello Kafka!
This is my first message
Привет, мир!
```

Вы увидите эти сообщения в терминале consumer.

### Программная отправка и получение (Python)

```python
# producer.py
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Отправка сообщения
message = {"user": "alice", "action": "login", "timestamp": "2024-01-15T10:30:00"}
future = producer.send('test-messages', value=message)

# Ожидание подтверждения
result = future.get(timeout=10)
print(f"Сообщение отправлено в партицию {result.partition}, offset {result.offset}")

producer.close()
```

```python
# consumer.py
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'test-messages',
    bootstrap_servers=['localhost:9092'],
    auto_offset_reset='earliest',
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

print("Ожидание сообщений...")
for message in consumer:
    print(f"Получено: {message.value}")
    print(f"Партиция: {message.partition}, Offset: {message.offset}")
```

## Базовая конфигурация

### Конфигурация сервера (server.properties)

```properties
# Идентификатор брокера (уникальный для каждого брокера в кластере)
broker.id=0

# Порт для подключений
listeners=PLAINTEXT://:9092
advertised.listeners=PLAINTEXT://localhost:9092

# Директория для хранения логов (данных)
log.dirs=/tmp/kafka-logs

# Количество партиций по умолчанию
num.partitions=1

# Время хранения сообщений (7 дней по умолчанию)
log.retention.hours=168

# Максимальный размер лога перед ротацией
log.segment.bytes=1073741824

# Подключение к ZooKeeper (для версий до 3.3)
zookeeper.connect=localhost:2181

# Таймаут соединения с ZooKeeper
zookeeper.connection.timeout.ms=18000
```

### Конфигурация producer

```properties
# Адреса брокеров
bootstrap.servers=localhost:9092

# Гарантии доставки
# acks=0 - не ждать подтверждения
# acks=1 - ждать подтверждения от лидера
# acks=all - ждать подтверждения от всех реплик
acks=all

# Количество повторных попыток
retries=3

# Размер batch для отправки
batch.size=16384

# Задержка перед отправкой batch
linger.ms=1

# Размер буфера
buffer.memory=33554432

# Сериализаторы
key.serializer=org.apache.kafka.common.serialization.StringSerializer
value.serializer=org.apache.kafka.common.serialization.StringSerializer
```

### Конфигурация consumer

```properties
# Адреса брокеров
bootstrap.servers=localhost:9092

# Идентификатор группы потребителей
group.id=my-consumer-group

# Автоматический commit offset
enable.auto.commit=true
auto.commit.interval.ms=1000

# Начальная позиция при отсутствии offset
# earliest - с начала
# latest - только новые
auto.offset.reset=earliest

# Максимальное количество записей за poll
max.poll.records=500

# Десериализаторы
key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
```

## Настройка окружения разработки

### Структура проекта

```
kafka-learning/
├── docker-compose.yml
├── config/
│   ├── producer.properties
│   └── consumer.properties
├── scripts/
│   ├── create-topics.sh
│   └── cleanup.sh
└── src/
    ├── producer.py
    └── consumer.py
```

### Скрипт создания топиков

```bash
#!/bin/bash
# scripts/create-topics.sh

BOOTSTRAP_SERVER="localhost:9092"

# Создание топиков для разработки
kafka-topics.sh --create --topic users --bootstrap-server $BOOTSTRAP_SERVER --partitions 3 --replication-factor 1 --if-not-exists
kafka-topics.sh --create --topic orders --bootstrap-server $BOOTSTRAP_SERVER --partitions 3 --replication-factor 1 --if-not-exists
kafka-topics.sh --create --topic notifications --bootstrap-server $BOOTSTRAP_SERVER --partitions 1 --replication-factor 1 --if-not-exists

echo "Топики созданы:"
kafka-topics.sh --list --bootstrap-server $BOOTSTRAP_SERVER
```

### Python окружение

```bash
# Создание виртуального окружения
python -m venv venv
source venv/bin/activate

# Установка зависимостей
pip install kafka-python
pip install confluent-kafka  # альтернативная библиотека
```

## Best Practices

### При установке и настройке

1. **Используйте Docker для разработки** — проще управлять и очищать окружение
2. **Выделяйте отдельные диски для данных Kafka** — в production это критично для производительности
3. **Настройте мониторинг с первого дня** — JMX метрики, Prometheus, Grafana
4. **Используйте KRaft для новых проектов** — ZooKeeper постепенно устаревает

### При работе с CLI

1. **Создавайте топики явно** — не полагайтесь на auto.create.topics
2. **Используйте осмысленные имена топиков** — `orders.created`, `users.updated`
3. **Документируйте конфигурацию** — особенно retention и partitions
4. **Тестируйте consumer groups** — понимайте, как работает балансировка

### При отправке сообщений

1. **Всегда используйте ключи** — обеспечивают порядок и партиционирование
2. **Настройте acks=all для критичных данных** — гарантия сохранности
3. **Обрабатывайте ошибки отправки** — используйте callbacks и retry
4. **Выбирайте правильный serializer** — JSON, Avro, Protobuf

### При получении сообщений

1. **Используйте consumer groups** — для масштабирования и отказоустойчивости
2. **Управляйте offset вручную** — если важна exactly-once семантика
3. **Обрабатывайте rebalancing** — подписывайтесь на ConsumerRebalanceListener
4. **Не блокируйте poll() надолго** — используйте max.poll.interval.ms

## Частые ошибки

### Connection refused

```bash
# Проверьте, что Kafka запущена
docker-compose ps
# или
jps  # должен показать Kafka процесс

# Проверьте advertised.listeners
# localhost работает только локально
```

### TopicAuthorizationException

```bash
# Проверьте ACL, если используется авторизация
kafka-acls.sh --list --bootstrap-server localhost:9092
```

### Leader Not Available

```bash
# Обычно возникает при создании топика
# Подождите несколько секунд и повторите
kafka-topics.sh --describe --topic my-topic --bootstrap-server localhost:9092
```

## Дополнительные ресурсы

- [Официальная документация Apache Kafka](https://kafka.apache.org/documentation/)
- [Confluent Documentation](https://docs.confluent.io/)
- [Kafka: The Definitive Guide](https://www.confluent.io/resources/kafka-the-definitive-guide/)
