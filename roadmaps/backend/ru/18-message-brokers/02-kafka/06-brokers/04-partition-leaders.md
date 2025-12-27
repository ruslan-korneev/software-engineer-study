# Ведущие реплики разделов

## Описание

**Репликация** в Kafka — это механизм обеспечения отказоустойчивости и высокой доступности данных. Каждая партиция может иметь несколько реплик, распределённых по разным брокерам. Одна из реплик назначается **лидером (leader)**, остальные — **фолловерами (followers)**. Все операции чтения и записи проходят через лидера, а фолловеры синхронизируют данные с лидером.

## Ключевые концепции

### Архитектура репликации

```
┌─────────────────────────────────────────────────────────────┐
│                  Topic: orders (3 partitions, RF=3)          │
│                                                              │
│  Partition 0          Partition 1          Partition 2       │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐  │
│  │ Broker 1    │      │ Broker 2    │      │ Broker 3    │  │
│  │ ★ Leader    │      │ ★ Leader    │      │ ★ Leader    │  │
│  └─────────────┘      └─────────────┘      └─────────────┘  │
│         │                   │                    │          │
│         ▼                   ▼                    ▼          │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐  │
│  │ Broker 2    │      │ Broker 3    │      │ Broker 1    │  │
│  │   Follower  │      │   Follower  │      │   Follower  │  │
│  └─────────────┘      └─────────────┘      └─────────────┘  │
│         │                   │                    │          │
│         ▼                   ▼                    ▼          │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐  │
│  │ Broker 3    │      │ Broker 1    │      │ Broker 2    │  │
│  │   Follower  │      │   Follower  │      │   Follower  │  │
│  └─────────────┘      └─────────────┘      └─────────────┘  │
│                                                              │
│  ★ = Leader     RF = Replication Factor                     │
└─────────────────────────────────────────────────────────────┘
```

### Терминология

| Термин | Описание |
|--------|----------|
| **Leader** | Реплика, обрабатывающая все запросы чтения/записи |
| **Follower** | Реплика, синхронизирующая данные с лидера |
| **ISR (In-Sync Replicas)** | Набор реплик, синхронизированных с лидером |
| **OSR (Out-of-Sync Replicas)** | Реплики, отставшие от лидера |
| **LEO (Log End Offset)** | Оффсет последнего сообщения в реплике |
| **HW (High Watermark)** | Оффсет, до которого все ISR синхронизированы |
| **Preferred Leader** | Брокер, предпочтительный для роли лидера |

### Процесс репликации

```
┌──────────────────────────────────────────────────────────────┐
│                    Replication Process                        │
│                                                               │
│  Producer                                                     │
│     │                                                         │
│     ▼                                                         │
│  ┌─────────────────────────────────────────────────────┐     │
│  │                    Leader                            │     │
│  │  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┐       │     │
│  │  │  0  │  1  │  2  │  3  │  4  │  5  │  6  │ LEO=7 │     │
│  │  └─────┴─────┴─────┴─────┴─────┴─────┴─────┘       │     │
│  │                              ▲                       │     │
│  │                              │ HW=5                  │     │
│  └─────────────────────────────────────────────────────┘     │
│           │                                                   │
│           │ Fetch requests                                    │
│           ▼                                                   │
│  ┌─────────────────────────────────────────────────────┐     │
│  │                  Follower 1 (ISR)                    │     │
│  │  ┌─────┬─────┬─────┬─────┬─────┬─────┐             │     │
│  │  │  0  │  1  │  2  │  3  │  4  │  5  │ LEO=6       │     │
│  │  └─────┴─────┴─────┴─────┴─────┴─────┘             │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐     │
│  │                  Follower 2 (ISR)                    │     │
│  │  ┌─────┬─────┬─────┬─────┬─────┐                   │     │
│  │  │  0  │  1  │  2  │  3  │  4  │ LEO=5             │     │
│  │  └─────┴─────┴─────┴─────┴─────┘                   │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                               │
│  HW (High Watermark) = min(LEO of all ISR) = 5               │
│  Consumers can read up to HW                                  │
└──────────────────────────────────────────────────────────────┘
```

### In-Sync Replicas (ISR)

Реплика считается синхронизированной (in-sync), если:
1. Она поддерживает активную сессию с ZooKeeper/контроллером
2. Она получила все сообщения за последние `replica.lag.time.max.ms` миллисекунд

```
┌─────────────────────────────────────────────────────────────┐
│                    ISR Management                            │
│                                                              │
│  Partition 0: Leader = Broker 1                              │
│                                                              │
│  Time T0:  ISR = [1, 2, 3]    (все синхронизированы)        │
│                                                              │
│  Time T1:  Broker 3 отстаёт > replica.lag.time.max.ms       │
│            ISR = [1, 2]       (Broker 3 исключён)           │
│                                                              │
│  Time T2:  Broker 3 догнал лидера                           │
│            ISR = [1, 2, 3]    (Broker 3 возвращён)          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Выбор лидера

```
┌─────────────────────────────────────────────────────────────┐
│                    Leader Election                           │
│                                                              │
│  Сценарий: Leader (Broker 1) падает                         │
│                                                              │
│  До:                                                         │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐        │
│  │  Broker 1   │   │  Broker 2   │   │  Broker 3   │        │
│  │  ★ Leader   │   │  Follower   │   │  Follower   │        │
│  │   (ISR)     │   │   (ISR)     │   │   (OSR)     │        │
│  └─────────────┘   └─────────────┘   └─────────────┘        │
│        ✕                                                     │
│                                                              │
│  После:                                                      │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐        │
│  │  Broker 1   │   │  Broker 2   │   │  Broker 3   │        │
│  │    DOWN     │   │  ★ Leader   │   │  Follower   │        │
│  │             │   │   (ISR)     │   │   (OSR)     │        │
│  └─────────────┘   └─────────────┘   └─────────────┘        │
│                                                              │
│  Контроллер выбирает нового лидера из ISR (Broker 2)        │
└─────────────────────────────────────────────────────────────┘
```

## Примеры конфигурации

### Настройки репликации на уровне брокера

```properties
# Фактор репликации по умолчанию
default.replication.factor=3

# Минимальное количество ISR для записи с acks=all
min.insync.replicas=2

# Запрет выбора не-ISR реплики лидером (критично для durability)
unclean.leader.election.enable=false

# Таймаут для определения "отставшей" реплики
replica.lag.time.max.ms=30000

# Настройки fetcher потоков
num.replica.fetchers=4
replica.fetch.min.bytes=1
replica.fetch.max.bytes=1048576
replica.fetch.wait.max.ms=500
replica.fetch.backoff.ms=1000

# Проверка баланса лидеров
leader.imbalance.check.interval.seconds=300
leader.imbalance.per.broker.percentage=10
auto.leader.rebalance.enable=true
```

### Настройки на уровне топика

```bash
# Создание топика с конкретным фактором репликации
bin/kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic critical-data \
  --partitions 6 \
  --replication-factor 3 \
  --config min.insync.replicas=2

# Изменение min.insync.replicas для существующего топика
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name critical-data \
  --alter \
  --add-config min.insync.replicas=2
```

### Ручное назначение реплик

```bash
# Файл reassignment.json
cat > reassignment.json << 'EOF'
{
  "version": 1,
  "partitions": [
    {
      "topic": "my-topic",
      "partition": 0,
      "replicas": [1, 2, 3],
      "log_dirs": ["any", "any", "any"]
    },
    {
      "topic": "my-topic",
      "partition": 1,
      "replicas": [2, 3, 1],
      "log_dirs": ["any", "any", "any"]
    }
  ]
}
EOF

# Выполнение переназначения
bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --execute

# Проверка статуса
bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --verify
```

## Просмотр информации о репликах

### Использование kafka-topics.sh

```bash
# Описание топика с информацией о репликах
bin/kafka-topics.sh --describe \
  --bootstrap-server localhost:9092 \
  --topic my-topic

# Вывод:
# Topic: my-topic	TopicId: xyz	PartitionCount: 3	ReplicationFactor: 3
# 	Topic: my-topic	Partition: 0	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
# 	Topic: my-topic	Partition: 1	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
# 	Topic: my-topic	Partition: 2	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2

# Показать under-replicated партиции
bin/kafka-topics.sh --describe \
  --bootstrap-server localhost:9092 \
  --under-replicated-partitions

# Показать партиции без лидера
bin/kafka-topics.sh --describe \
  --bootstrap-server localhost:9092 \
  --unavailable-partitions
```

### Использование kafka-metadata.sh (KRaft)

```bash
# Информация о лидерах в KRaft mode
bin/kafka-metadata.sh --snapshot /var/kafka-metadata/__cluster_metadata-0/00000000000000000000.log \
  --command "cat /brokers"
```

## Preferred Leader Election

### Что такое Preferred Leader

**Preferred Leader** — это первая реплика в списке реплик партиции. При равномерном распределении preferred leaders по брокерам достигается баланс нагрузки.

```bash
# Партиция с Replicas: [2, 1, 3]
# Preferred Leader = Broker 2

# Если текущий Leader != Preferred Leader,
# партиция считается "imbalanced"
```

### Перебалансировка лидеров

```bash
# Автоматическая перебалансировка (включена по умолчанию)
# Настройки в server.properties:
auto.leader.rebalance.enable=true
leader.imbalance.check.interval.seconds=300
leader.imbalance.per.broker.percentage=10

# Ручная перебалансировка для конкретного топика
bin/kafka-leader-election.sh --bootstrap-server localhost:9092 \
  --election-type PREFERRED \
  --topic my-topic

# Для всех топиков
bin/kafka-leader-election.sh --bootstrap-server localhost:9092 \
  --election-type PREFERRED \
  --all-topic-partitions
```

## Unclean Leader Election

### Когда происходит

Если все ISR реплики недоступны, а `unclean.leader.election.enable=true`, лидером может стать реплика из OSR. Это приводит к потере данных.

```
┌─────────────────────────────────────────────────────────────┐
│              Unclean Leader Election                         │
│                                                              │
│  Состояние: ISR = [1, 2], OSR = [3]                         │
│  Leader = Broker 1, LEO = 100                                │
│  Broker 2: LEO = 100 (синхронизирован)                      │
│  Broker 3: LEO = 80 (отстаёт на 20 сообщений)               │
│                                                              │
│  Событие: Broker 1 и Broker 2 падают одновременно           │
│                                                              │
│  С unclean.leader.election.enable=true:                     │
│    Broker 3 становится лидером                              │
│    Сообщения 81-100 ПОТЕРЯНЫ                                │
│                                                              │
│  С unclean.leader.election.enable=false:                    │
│    Партиция НЕДОСТУПНА до восстановления ISR               │
│    Данные НЕ ПОТЕРЯНЫ                                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Рекомендации

```properties
# Для критически важных данных — всегда false
unclean.leader.election.enable=false

# Можно настроить на уровне топика
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name critical-topic \
  --alter \
  --add-config unclean.leader.election.enable=false
```

## Rack Awareness

### Распределение реплик по стойкам

```
┌─────────────────────────────────────────────────────────────┐
│                    Rack-Aware Replication                    │
│                                                              │
│   Rack A              Rack B              Rack C             │
│  ┌─────────┐        ┌─────────┐        ┌─────────┐          │
│  │Broker 1 │        │Broker 2 │        │Broker 3 │          │
│  │         │        │         │        │         │          │
│  │ P0-L    │        │ P0-F    │        │ P0-F    │          │
│  │ P1-F    │        │ P1-L    │        │ P1-F    │          │
│  │ P2-F    │        │ P2-F    │        │ P2-L    │          │
│  └─────────┘        └─────────┘        └─────────┘          │
│                                                              │
│  L = Leader, F = Follower                                   │
│  Каждая партиция имеет реплики в разных стойках            │
└─────────────────────────────────────────────────────────────┘
```

### Конфигурация

```properties
# На каждом брокере указать его rack
# Broker 1 (server.properties)
broker.rack=rack-a

# Broker 2 (server.properties)
broker.rack=rack-b

# Broker 3 (server.properties)
broker.rack=rack-c
```

## Best Practices

### 1. Оптимальный фактор репликации

```properties
# Для production — минимум 3
default.replication.factor=3

# Для internal топиков
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3

# Формула:
# replication.factor = количество допустимых отказов + 1
# 3 реплики = выдержит отказ 1 брокера при min.insync.replicas=2
```

### 2. Правильная настройка min.insync.replicas

```properties
# Формула безопасности:
# min.insync.replicas <= replication.factor - 1
#
# replication.factor=3, min.insync.replicas=2
# Это позволяет продолжать работу при отказе 1 брокера

min.insync.replicas=2
```

### 3. Мониторинг реплик

```bash
# Ключевые метрики для мониторинга:

# UnderReplicatedPartitions — партиции с недостаточным ISR
kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions

# UnderMinIsrPartitionCount — партиции с ISR < min.insync.replicas
kafka.server:type=ReplicaManager,name=UnderMinIsrPartitionCount

# OfflinePartitionsCount — недоступные партиции (нет лидера)
kafka.controller:type=KafkaController,name=OfflinePartitionsCount

# IsrShrinkRate / IsrExpandRate — частота изменений ISR
kafka.server:type=ReplicaManager,name=IsrShrinksPerSec
kafka.server:type=ReplicaManager,name=IsrExpandsPerSec

# ReplicaLagTimeMax — максимальное отставание реплик
kafka.server:type=ReplicaFetcherManager,name=MaxLag,clientId=*
```

### 4. Настройки producer для гарантии доставки

```python
# Python пример с гарантией записи в ISR
from kafka import KafkaProducer

producer = KafkaProducer(
    bootstrap_servers=['kafka1:9092', 'kafka2:9092', 'kafka3:9092'],
    acks='all',  # Ждать подтверждения от всех ISR
    retries=5,
    retry_backoff_ms=100,
    max_in_flight_requests_per_connection=1,  # Для порядка сообщений
    enable_idempotence=True  # Exactly-once semantics
)
```

### 5. Действия при проблемах с репликами

```bash
# 1. Проверить состояние партиций
bin/kafka-topics.sh --describe --bootstrap-server localhost:9092 \
  --under-replicated-partitions

# 2. Проверить состояние брокеров
bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092

# 3. Проверить логи брокеров на ошибки репликации
grep -i "replica\|replication\|ISR" /var/log/kafka/server.log

# 4. Если брокер восстановился — дождаться синхронизации
# Реплика автоматически войдёт в ISR после синхронизации

# 5. При необходимости — перенести лидерство
bin/kafka-leader-election.sh --bootstrap-server localhost:9092 \
  --election-type PREFERRED \
  --all-topic-partitions
```

## Replica Fetcher

### Как работает синхронизация

```
┌──────────────────────────────────────────────────────────────┐
│                    Replica Fetcher                            │
│                                                               │
│  Follower                              Leader                 │
│     │                                     │                   │
│     │──── FetchRequest(offset=100) ─────▶│                   │
│     │                                     │                   │
│     │◀─── FetchResponse(records) ────────│                   │
│     │                                     │                   │
│     │  1. Записать в локальный лог       │                   │
│     │  2. Обновить LEO                    │                   │
│     │  3. Сообщить лидеру о прогрессе    │                   │
│     │                                     │                   │
│     │──── FetchRequest(offset=150) ─────▶│                   │
│     │                                     │                   │
└──────────────────────────────────────────────────────────────┘
```

### Параметры fetcher

```properties
# Количество потоков для репликации
num.replica.fetchers=4

# Минимальный размер данных для fetch
replica.fetch.min.bytes=1

# Максимальный размер данных для fetch
replica.fetch.max.bytes=1048576

# Максимальное время ожидания данных
replica.fetch.wait.max.ms=500

# Backoff при ошибках
replica.fetch.backoff.ms=1000

# Размер буфера сокета
replica.socket.receive.buffer.bytes=65536

# Таймаут сокета
replica.socket.timeout.ms=30000
```
