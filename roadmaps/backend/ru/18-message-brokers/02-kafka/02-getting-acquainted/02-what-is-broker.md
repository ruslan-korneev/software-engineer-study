# Что такое брокер?

[prev: 01-send-receive-message](./01-send-receive-message.md) | [next: 03-kafka-tour](./03-kafka-tour.md)

---

## Описание

**Брокер (Broker)** — это отдельный сервер Kafka, который является частью кластера и отвечает за хранение данных и обработку запросов от клиентов. В контексте Apache Kafka брокер выполняет роль посредника между производителями (producers) и потребителями (consumers) сообщений.

Kafka брокер — это JVM-приложение, которое:
- Принимает сообщения от producers
- Сохраняет сообщения на диск
- Отдает сообщения consumers
- Участвует в репликации данных
- Координируется с другими брокерами в кластере

## Ключевые концепции

### Архитектура брокера

```
┌─────────────────────────────────────────────────────────────┐
│                        Kafka Broker                          │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   Topic A   │  │   Topic B   │  │   Topic C   │          │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤          │
│  │ Partition 0 │  │ Partition 0 │  │ Partition 0 │          │
│  │ Partition 1 │  │ Partition 1 │  │ Partition 1 │          │
│  │ Partition 2 │  │             │  │ Partition 2 │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
├─────────────────────────────────────────────────────────────┤
│  Network Layer │ Request Handler │ Log Manager │ Replicator │
└─────────────────────────────────────────────────────────────┘
```

### Broker ID

Каждый брокер в кластере имеет уникальный идентификатор — **broker.id**:

```properties
# server.properties
broker.id=0  # Уникальный ID для каждого брокера
```

```bash
# Просмотр информации о брокерах в кластере
kafka-broker-api-versions.sh --bootstrap-server localhost:9092
```

### Роли брокера в кластере

#### 1. Leader (Лидер)
Брокер, который является лидером для определенной партиции:
- Обрабатывает все операции записи и чтения для этой партиции
- Реплицирует данные на follower-брокеры
- Отвечает за поддержание консистентности

#### 2. Follower (Последователь)
Брокер, который хранит реплику партиции:
- Постоянно синхронизируется с лидером
- Готов стать лидером при отказе текущего лидера
- Не обрабатывает клиентские запросы напрямую (до Kafka 2.4)

#### 3. Controller (Контроллер)
Один брокер в кластере выбирается контроллером:
- Управляет назначением лидеров партиций
- Отслеживает состояние брокеров
- Координирует административные задачи

```bash
# Просмотр текущего контроллера
kafka-metadata.sh --snapshot /var/kafka-logs/__cluster_metadata-0/00000000000000000000.log --command "cluster"
```

### Кластер брокеров

```
┌────────────────────────────────────────────────────────────────┐
│                        Kafka Cluster                            │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Broker 0   │  │   Broker 1   │  │   Broker 2   │          │
│  │  (Controller)│  │              │  │              │          │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤          │
│  │ Topic A P0 L │  │ Topic A P0 F │  │ Topic A P1 L │          │
│  │ Topic A P1 F │  │ Topic A P1 F │  │ Topic A P0 F │          │
│  │ Topic B P0 F │  │ Topic B P0 L │  │ Topic B P1 L │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  L = Leader, F = Follower                                       │
└────────────────────────────────────────────────────────────────┘
```

### Внутренние компоненты брокера

#### 1. Network Layer
Обрабатывает TCP соединения с клиентами:
- Принимает входящие соединения
- Управляет пулом потоков для обработки запросов
- Использует Java NIO для эффективной работы

#### 2. Request Handler
Обрабатывает различные типы запросов:
- Produce — запись сообщений
- Fetch — чтение сообщений
- Metadata — информация о топиках и партициях
- Offset — управление смещениями

#### 3. Log Manager
Управляет хранением данных:
- Записывает сообщения в сегменты логов
- Управляет индексами
- Выполняет очистку старых данных

#### 4. Replica Manager
Координирует репликацию:
- Синхронизирует данные между брокерами
- Отслеживает ISR (In-Sync Replicas)
- Управляет переключением лидеров

## Примеры кода

### Получение информации о брокерах

```python
from kafka import KafkaAdminClient
from kafka.admin import ConfigResource, ConfigResourceType

admin = KafkaAdminClient(
    bootstrap_servers=['localhost:9092'],
    client_id='broker-info-client'
)

# Получение метаданных кластера
cluster_metadata = admin.describe_cluster()
print(f"Cluster ID: {cluster_metadata['cluster_id']}")
print(f"Controller ID: {cluster_metadata['controller_id']}")

print("\nБрокеры в кластере:")
for broker in cluster_metadata['brokers']:
    print(f"  ID: {broker['node_id']}, Host: {broker['host']}:{broker['port']}")

admin.close()
```

### Проверка состояния брокера

```python
from kafka import KafkaAdminClient

def check_broker_health(bootstrap_servers):
    """Проверка доступности брокеров"""
    try:
        admin = KafkaAdminClient(
            bootstrap_servers=bootstrap_servers,
            client_id='health-checker',
            request_timeout_ms=5000
        )

        cluster = admin.describe_cluster()
        active_brokers = len(cluster['brokers'])

        print(f"Активных брокеров: {active_brokers}")
        print(f"Controller: Broker {cluster['controller_id']}")

        admin.close()
        return True

    except Exception as e:
        print(f"Ошибка подключения: {e}")
        return False

# Использование
check_broker_health(['localhost:9092'])
```

### Получение конфигурации брокера

```python
from kafka import KafkaAdminClient
from kafka.admin import ConfigResource, ConfigResourceType

admin = KafkaAdminClient(bootstrap_servers=['localhost:9092'])

# Получение конфигурации брокера с ID=0
config_resource = ConfigResource(ConfigResourceType.BROKER, "0")
configs = admin.describe_configs([config_resource])

for resource, config in configs.items():
    print(f"\nКонфигурация брокера {resource.name}:")
    for name, value in sorted(config.items()):
        if not value.is_default:  # Только измененные настройки
            print(f"  {name}: {value.value}")

admin.close()
```

### Мониторинг брокеров через JMX

```python
# Пример подключения к JMX метрикам (требует py4j или jmxquery)
# Обычно используется через Prometheus JMX Exporter

# Основные метрики брокера:
# - kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec
# - kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec
# - kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec
# - kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
# - kafka.controller:type=KafkaController,name=ActiveControllerCount
```

## Конфигурация брокера

### Основные параметры

```properties
# server.properties

############################# Server Basics #############################

# Уникальный ID брокера
broker.id=0

############################# Socket Server Settings #############################

# Порты для прослушивания
listeners=PLAINTEXT://:9092

# Адрес, который брокер сообщает клиентам
advertised.listeners=PLAINTEXT://kafka-broker-1:9092

# Количество потоков для обработки сетевых запросов
num.network.threads=3

# Количество потоков для обработки запросов (включая I/O)
num.io.threads=8

# Размер буфера отправки/приема
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400

# Максимальный размер запроса
socket.request.max.bytes=104857600

############################# Log Basics #############################

# Директория для хранения логов
log.dirs=/var/kafka-logs

# Количество партиций по умолчанию
num.partitions=3

# Количество потоков для восстановления логов при старте
num.recovery.threads.per.data.dir=1

############################# Internal Topic Settings #############################

# Фактор репликации для внутренних топиков
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2

############################# Log Retention Policy #############################

# Время хранения сообщений
log.retention.hours=168

# Максимальный размер лога
log.retention.bytes=-1

# Размер сегмента лога
log.segment.bytes=1073741824

# Интервал проверки логов для очистки
log.retention.check.interval.ms=300000
```

### Параметры производительности

```properties
# Оптимизация производительности

# Количество реплик, которые должны подтвердить запись
min.insync.replicas=2

# Время ожидания синхронизации реплик
replica.lag.time.max.ms=30000

# Использование zero-copy для передачи данных
socket.send.buffer.bytes=1048576
socket.receive.buffer.bytes=1048576

# Сжатие сообщений
compression.type=producer  # или lz4, snappy, zstd

# Параметры для batch обработки
num.replica.fetchers=4
```

### Настройки безопасности

```properties
# SSL настройки
listeners=SSL://:9093
ssl.keystore.location=/var/ssl/kafka.server.keystore.jks
ssl.keystore.password=secret
ssl.key.password=secret
ssl.truststore.location=/var/ssl/kafka.server.truststore.jks
ssl.truststore.password=secret

# SASL настройки
sasl.enabled.mechanisms=PLAIN,SCRAM-SHA-256
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256

# ACL
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
super.users=User:admin
```

## Best Practices

### 1. Планирование кластера

```
Рекомендации по количеству брокеров:

Минимум для production: 3 брокера
- Обеспечивает отказоустойчивость
- Позволяет replication-factor=3
- Один брокер может выйти из строя без потери данных

Для высоконагруженных систем: 5+ брокеров
- Лучше распределение нагрузки
- Быстрее восстановление после сбоев
- Проще масштабирование
```

### 2. Размещение брокеров

```yaml
# Пример для Kubernetes с anti-affinity
apiVersion: apps/v1
kind: StatefulSet
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: kafka
              topologyKey: "kubernetes.io/hostname"
```

### 3. Мониторинг брокеров

```bash
# Ключевые метрики для мониторинга:

# 1. Under-replicated partitions (должно быть 0)
kafka-topics.sh --describe --under-replicated-partitions \
  --bootstrap-server localhost:9092

# 2. Offline partitions (должно быть 0)
kafka-topics.sh --describe --unavailable-partitions \
  --bootstrap-server localhost:9092

# 3. ISR shrink rate
# Если ISR уменьшается — проблема с репликацией
```

### 4. Обслуживание брокеров

```bash
# Graceful shutdown брокера
# Kafka автоматически перенесет лидерство на другие брокеры

# 1. Перенос лидерства перед остановкой
kafka-leader-election.sh --election-type preferred \
  --bootstrap-server localhost:9092 --all-topic-partitions

# 2. Остановка брокера
bin/kafka-server-stop.sh

# 3. После обновления/обслуживания — запуск
bin/kafka-server-start.sh config/server.properties
```

### 5. Балансировка нагрузки

```bash
# Перераспределение партиций между брокерами
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --execute
```

```json
// reassignment.json
{
  "version": 1,
  "partitions": [
    {
      "topic": "my-topic",
      "partition": 0,
      "replicas": [1, 2, 3]
    },
    {
      "topic": "my-topic",
      "partition": 1,
      "replicas": [2, 3, 1]
    }
  ]
}
```

## Частые проблемы

### 1. Брокер недоступен

```bash
# Проверка статуса
kafka-broker-api-versions.sh --bootstrap-server localhost:9092

# Проверка логов
tail -f /var/log/kafka/server.log

# Частые причины:
# - Неправильный advertised.listeners
# - Проблемы с сетью/firewall
# - Нехватка памяти
# - Проблемы с ZooKeeper
```

### 2. Under-replicated partitions

```bash
# Диагностика
kafka-topics.sh --describe --under-replicated-partitions \
  --bootstrap-server localhost:9092

# Возможные причины:
# - Один из брокеров недоступен
# - Сетевые проблемы между брокерами
# - Перегрузка брокера
# - Медленный диск
```

### 3. Переполнение диска

```bash
# Проверка использования диска
df -h /var/kafka-logs

# Настройка retention для освобождения места
kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter --entity-type topics --entity-name my-topic \
  --add-config retention.ms=86400000  # 1 день
```

## Дополнительные ресурсы

- [Kafka Broker Configuration](https://kafka.apache.org/documentation/#brokerconfigs)
- [Kafka Operations](https://kafka.apache.org/documentation/#operations)
- [Monitoring Kafka](https://kafka.apache.org/documentation/#monitoring)

---

[prev: 01-send-receive-message](./01-send-receive-message.md) | [next: 03-kafka-tour](./03-kafka-tour.md)
