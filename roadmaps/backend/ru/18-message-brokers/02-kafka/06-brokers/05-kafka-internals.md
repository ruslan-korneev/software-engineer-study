# Взгляд внутрь Kafka

## Описание

Понимание внутреннего устройства Kafka помогает принимать правильные архитектурные решения, оптимизировать производительность и эффективно решать проблемы. В этом разделе рассмотрим ключевые компоненты: контроллер, режим KRaft, межброкерную коммуникацию и внутренние механизмы работы.

## Ключевые концепции

### Архитектура Kafka

```
┌──────────────────────────────────────────────────────────────┐
│                    Kafka Cluster Architecture                 │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                   Metadata Management                    │ │
│  │  ┌───────────────┐  или  ┌────────────────────────────┐ │ │
│  │  │   ZooKeeper   │       │       KRaft Quorum         │ │ │
│  │  │   Ensemble    │       │  (Controller Quorum)       │ │ │
│  │  └───────────────┘       └────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────┘ │
│                              │                                │
│                              ▼                                │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                   Controller                             │ │
│  │    (один брокер, управляющий кластером)                 │ │
│  └─────────────────────────────────────────────────────────┘ │
│                              │                                │
│         ┌────────────────────┼────────────────────┐          │
│         ▼                    ▼                    ▼          │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐  │
│  │  Broker 1   │◀────▶│  Broker 2   │◀────▶│  Broker 3   │  │
│  │             │      │             │      │             │  │
│  │  Partitions │      │  Partitions │      │  Partitions │  │
│  │  Replicas   │      │  Replicas   │      │  Replicas   │  │
│  └─────────────┘      └─────────────┘      └─────────────┘  │
│         ▲                    ▲                    ▲          │
│         └────────────────────┼────────────────────┘          │
│                              │                                │
│                    ┌─────────┴─────────┐                     │
│                    │    Clients        │                     │
│                    │ (Producers,       │                     │
│                    │  Consumers)       │                     │
│                    └───────────────────┘                     │
└──────────────────────────────────────────────────────────────┘
```

## Контроллер (Controller)

### Роль контроллера

**Контроллер** — это один из брокеров кластера, выбранный для управления административными операциями. В каждый момент времени в кластере может быть только один активный контроллер.

### Функции контроллера

```
┌─────────────────────────────────────────────────────────────┐
│                    Controller Responsibilities               │
│                                                              │
│  1. Leader Election                                          │
│     ├─ Выбор лидеров партиций при создании топиков          │
│     ├─ Перевыборы при отказе брокера                        │
│     └─ Preferred leader election                             │
│                                                              │
│  2. Cluster Membership                                       │
│     ├─ Отслеживание брокеров (регистрация/отказ)            │
│     ├─ Обновление ISR при изменениях                        │
│     └─ Управление broker IDs                                 │
│                                                              │
│  3. Topic Management                                         │
│     ├─ Создание топиков                                      │
│     ├─ Удаление топиков                                      │
│     ├─ Изменение количества партиций                        │
│     └─ Обновление конфигурации топиков                      │
│                                                              │
│  4. Replica Assignment                                       │
│     ├─ Назначение реплик на брокеры                         │
│     ├─ Перераспределение партиций                           │
│     └─ Rack-aware размещение                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Выбор контроллера

```
┌─────────────────────────────────────────────────────────────┐
│               Controller Election (ZooKeeper mode)           │
│                                                              │
│  1. При старте все брокеры пытаются создать                 │
│     ephemeral node /controller в ZooKeeper                   │
│                                                              │
│  2. Первый успешный становится контроллером                 │
│                                                              │
│  3. Остальные устанавливают watch на /controller            │
│                                                              │
│  4. При падении контроллера:                                 │
│     - ZooKeeper удаляет ephemeral node                      │
│     - Watches срабатывают                                   │
│     - Начинаются новые выборы                               │
│                                                              │
│  Controller Epoch: монотонно растущий счётчик               │
│  Используется для игнорирования устаревших команд          │
└─────────────────────────────────────────────────────────────┘
```

### Метрики контроллера

```bash
# Ключевые метрики контроллера
kafka.controller:type=KafkaController,name=ActiveControllerCount  # Должен быть 1
kafka.controller:type=KafkaController,name=OfflinePartitionsCount  # Желательно 0
kafka.controller:type=ControllerStats,name=LeaderElectionRateAndTimeMs
kafka.controller:type=ControllerStats,name=UncleanLeaderElectionsPerSec
```

## KRaft Mode

### Что такое KRaft

**KRaft (Kafka Raft)** — это новый режим работы Kafka, который заменяет ZooKeeper встроенным протоколом консенсуса на основе Raft. Представлен в Kafka 2.8, стал production-ready в Kafka 3.3+.

### Преимущества KRaft

```
┌─────────────────────────────────────────────────────────────┐
│                    KRaft Benefits                            │
│                                                              │
│  ✓ Упрощённая архитектура                                   │
│    • Нет отдельного кластера ZooKeeper                      │
│    • Меньше компонентов для управления                      │
│    • Единая система мониторинга                             │
│                                                              │
│  ✓ Улучшенная производительность                            │
│    • Быстрее выборы контроллера                             │
│    • Быстрее обновление метаданных                          │
│    • Меньше latency при изменениях                          │
│                                                              │
│  ✓ Масштабируемость                                         │
│    • Поддержка миллионов партиций                           │
│    • Нет ограничений ZooKeeper на znodes                    │
│                                                              │
│  ✓ Единый протокол безопасности                             │
│    • Использует Kafka security features                     │
│    • Нет отдельной настройки ZK security                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Архитектура KRaft

```
┌──────────────────────────────────────────────────────────────┐
│                    KRaft Architecture                         │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │              Controller Quorum                           │ │
│  │                                                          │ │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐           │ │
│  │  │Controller │  │Controller │  │Controller │           │ │
│  │  │     1     │  │     2     │  │     3     │           │ │
│  │  │  (Active) │  │ (Standby) │  │ (Standby) │           │ │
│  │  └───────────┘  └───────────┘  └───────────┘           │ │
│  │        │              │              │                  │ │
│  │        └──────────────┼──────────────┘                  │ │
│  │                       │                                  │ │
│  │            Raft Consensus Protocol                       │ │
│  │        (metadata log replication)                        │ │
│  └─────────────────────────────────────────────────────────┘ │
│                           │                                   │
│                           ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                   Brokers                                │ │
│  │                                                          │ │
│  │  ┌─────────┐    ┌─────────┐    ┌─────────┐             │ │
│  │  │ Broker 1│    │ Broker 2│    │ Broker 3│             │ │
│  │  │         │    │         │    │         │             │ │
│  │  │ Data    │    │ Data    │    │ Data    │             │ │
│  │  │ logs    │    │ logs    │    │ logs    │             │ │
│  │  └─────────┘    └─────────┘    └─────────┘             │ │
│  │                                                          │ │
│  │  Получают метаданные от Controller Quorum               │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Роли процессов в KRaft

```
┌─────────────────────────────────────────────────────────────┐
│                    Process Roles                             │
│                                                              │
│  process.roles=controller                                    │
│  ┌────────────────────────────────────────────┐             │
│  │  Dedicated Controller                       │             │
│  │  • Только управление метаданными           │             │
│  │  • Участвует в Raft quorum                 │             │
│  │  • Не хранит данные топиков                │             │
│  └────────────────────────────────────────────┘             │
│                                                              │
│  process.roles=broker                                        │
│  ┌────────────────────────────────────────────┐             │
│  │  Dedicated Broker                           │             │
│  │  • Только хранение данных                  │             │
│  │  • Получает метаданные от контроллеров     │             │
│  │  • Обрабатывает запросы клиентов           │             │
│  └────────────────────────────────────────────┘             │
│                                                              │
│  process.roles=broker,controller                             │
│  ┌────────────────────────────────────────────┐             │
│  │  Combined Mode                              │             │
│  │  • Обе роли в одном процессе               │             │
│  │  • Подходит для небольших кластеров        │             │
│  │  • Упрощает deployment                      │             │
│  └────────────────────────────────────────────┘             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Конфигурация KRaft

```properties
############################# KRaft Configuration #############################

# Уникальный ID узла (обязательно)
node.id=1

# Роль процесса
process.roles=broker,controller

# Для combined mode: broker,controller
# Для dedicated controller: controller
# Для dedicated broker: broker

# Кворум контроллеров (все узлы с ролью controller)
controller.quorum.voters=1@controller1:9093,2@controller2:9093,3@controller3:9093

# Listener для контроллера
controller.listener.names=CONTROLLER

# Listeners
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
advertised.listeners=PLAINTEXT://broker1:9092

# Протоколы безопасности
listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT

# Директория для метаданных (должна быть отдельной)
metadata.log.dir=/var/kafka-metadata

# Для брокеров: где хранить данные топиков
log.dirs=/var/kafka-logs
```

### Инициализация кластера KRaft

```bash
# 1. Генерация cluster ID
KAFKA_CLUSTER_ID=$(bin/kafka-storage.sh random-uuid)

# 2. Форматирование хранилища на каждом узле
bin/kafka-storage.sh format \
  --config /etc/kafka/server.properties \
  --cluster-id $KAFKA_CLUSTER_ID

# 3. Запуск узлов
bin/kafka-server-start.sh /etc/kafka/server.properties
```

### Миграция с ZooKeeper на KRaft

```
┌─────────────────────────────────────────────────────────────┐
│                    Migration Path                            │
│                                                              │
│  Kafka 3.4+: Поддержка онлайн-миграции                      │
│                                                              │
│  Шаги миграции:                                              │
│                                                              │
│  1. Подготовка                                               │
│     ├─ Обновить Kafka до версии 3.4+                        │
│     ├─ Развернуть KRaft контроллеры                         │
│     └─ Настроить migration mode                              │
│                                                              │
│  2. Dual-write фаза                                          │
│     ├─ Метаданные пишутся в ZK и KRaft                      │
│     └─ Брокеры продолжают работать                          │
│                                                              │
│  3. Переключение                                             │
│     ├─ Брокеры переключаются на KRaft                       │
│     └─ ZooKeeper отключается                                │
│                                                              │
│  4. Очистка                                                  │
│     └─ Удаление ZooKeeper кластера                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Межброкерная коммуникация

### Типы inter-broker запросов

```
┌─────────────────────────────────────────────────────────────┐
│              Inter-Broker Communication                      │
│                                                              │
│  Fetch (репликация)                                          │
│  ┌─────────┐          FetchRequest          ┌─────────┐     │
│  │Follower │ ────────────────────────────▶ │ Leader  │     │
│  │         │ ◀──────────────────────────── │         │     │
│  └─────────┘          FetchResponse         └─────────┘     │
│                                                              │
│  UpdateMetadata (от контроллера)                            │
│  ┌──────────┐    UpdateMetadataRequest      ┌─────────┐    │
│  │Controller│ ─────────────────────────────▶│ Broker  │    │
│  │          │ ◀─────────────────────────────│         │    │
│  └──────────┘    UpdateMetadataResponse     └─────────┘    │
│                                                              │
│  LeaderAndIsr (от контроллера)                              │
│  ┌──────────┐    LeaderAndIsrRequest        ┌─────────┐    │
│  │Controller│ ─────────────────────────────▶│ Broker  │    │
│  └──────────┘                               └─────────┘    │
│                                                              │
│  StopReplica (от контроллера)                               │
│  При удалении топика или переназначении реплик             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Конфигурация inter-broker

```properties
# Listener для межброкерной коммуникации
inter.broker.listener.name=INTERNAL

# Listeners с разделением internal/external
listeners=EXTERNAL://0.0.0.0:9092,INTERNAL://0.0.0.0:9093
advertised.listeners=EXTERNAL://public.kafka.com:9092,INTERNAL://broker1.internal:9093

# Протокол безопасности для inter-broker
listener.security.protocol.map=EXTERNAL:SSL,INTERNAL:SASL_SSL

# Настройки соединений
connections.max.idle.ms=600000
inter.broker.protocol.version=3.6
```

## Внутренние топики

### Системные топики Kafka

```
┌─────────────────────────────────────────────────────────────┐
│                    Internal Topics                           │
│                                                              │
│  __consumer_offsets                                          │
│  ├─ Хранит оффсеты consumer groups                          │
│  ├─ 50 партиций по умолчанию                                │
│  ├─ Compacted topic                                          │
│  └─ offsets.topic.replication.factor=3                      │
│                                                              │
│  __transaction_state                                         │
│  ├─ Состояние транзакций                                    │
│  ├─ 50 партиций по умолчанию                                │
│  ├─ Compacted topic                                          │
│  └─ transaction.state.log.replication.factor=3              │
│                                                              │
│  __cluster_metadata (KRaft only)                            │
│  ├─ Метаданные кластера                                     │
│  ├─ Одна партиция                                           │
│  └─ Реплицируется через Raft                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Обработка запросов

### Request Processing Pipeline

```
┌──────────────────────────────────────────────────────────────┐
│                  Request Processing                           │
│                                                               │
│  Client                                                       │
│     │                                                         │
│     ▼                                                         │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                Network Threads                        │    │
│  │  (num.network.threads)                               │    │
│  │  • Принимают соединения                              │    │
│  │  • Читают запросы                                    │    │
│  │  • Отправляют ответы                                 │    │
│  └──────────────────────────────────────────────────────┘    │
│                          │                                    │
│                          ▼                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                  Request Queue                        │    │
│  │  (queued.max.requests)                               │    │
│  └──────────────────────────────────────────────────────┘    │
│                          │                                    │
│                          ▼                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                 I/O Threads                           │    │
│  │  (num.io.threads)                                    │    │
│  │  • Обрабатывают запросы                              │    │
│  │  • Работают с LogManager                             │    │
│  │  • Взаимодействуют с диском                          │    │
│  └──────────────────────────────────────────────────────┘    │
│                          │                                    │
│                          ▼                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                 Response Queue                        │    │
│  └──────────────────────────────────────────────────────┘    │
│                          │                                    │
│                          ▼                                    │
│  Client                                                       │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

## Best Practices

### 1. Выбор между ZooKeeper и KRaft

```
Используйте KRaft если:
✓ Новая установка Kafka 3.3+
✓ Нужна простота операций
✓ Планируете масштабирование > 10K партиций

Оставайтесь на ZooKeeper если:
✓ Существующий production кластер
✓ Kafka < 3.3
✓ Используете ZooKeeper для других сервисов
```

### 2. Размещение контроллеров

```properties
# KRaft: рекомендуется нечётное число контроллеров
# Минимум: 3 контроллера
# Для высокой доступности: 5 контроллеров

# Размещайте контроллеры:
# - В разных failure domains
# - На выделенных серверах (для больших кластеров)
# - С быстрыми дисками для metadata log
```

### 3. Мониторинг внутренних компонентов

```bash
# Метрики контроллера
kafka.controller:type=KafkaController,name=GlobalTopicCount
kafka.controller:type=KafkaController,name=GlobalPartitionCount
kafka.controller:type=ControllerChannelManager,name=TotalQueueSize

# Метрики запросов
kafka.network:type=RequestChannel,name=RequestQueueSize
kafka.network:type=RequestMetrics,name=RequestsPerSec,request=*
kafka.network:type=RequestMetrics,name=TotalTimeMs,request=*

# Метрики репликации
kafka.server:type=ReplicaManager,name=IsrShrinksPerSec
kafka.server:type=ReplicaManager,name=IsrExpandsPerSec
kafka.server:type=ReplicaFetcherManager,name=MaxLag
```

### 4. Оптимизация производительности

```properties
# Для высокой пропускной способности
num.network.threads=8
num.io.threads=16
queued.max.requests=500

# Для низкой задержки
socket.send.buffer.bytes=1048576
socket.receive.buffer.bytes=1048576

# Настройки репликации
num.replica.fetchers=4
replica.fetch.max.bytes=10485760
```
