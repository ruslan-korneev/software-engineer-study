# Перемещение данных (Data Movement)

[prev: 01-data-retention](./01-data-retention.md) | [next: 03-tools](./03-tools.md)

---

## Описание

Перемещение данных в Kafka охватывает механизмы передачи сообщений между брокерами, кластерами и внешними системами. Понимание внутренней структуры хранения (log segments, indexes) и механизмов репликации критически важно для эффективной работы с Kafka.

## Ключевые концепции

### Структура хранения данных

#### Log Segments (Сегменты лога)

Каждая партиция состоит из упорядоченной последовательности сегментов:

```
partition-0/
├── 00000000000000000000.log      # Сегмент с offset 0
├── 00000000000000000000.index    # Offset index
├── 00000000000000000000.timeindex # Timestamp index
├── 00000000000000012345.log      # Сегмент с offset 12345
├── 00000000000000012345.index
├── 00000000000000012345.timeindex
└── leader-epoch-checkpoint
```

**Формат имени файла:** первый offset в сегменте, дополненный нулями до 20 символов.

#### Структура .log файла

```
┌─────────────────────────────────────────────────────────────┐
│ Record Batch                                                │
├─────────────────────────────────────────────────────────────┤
│ Base Offset (8 bytes)          │ 0                          │
│ Batch Length (4 bytes)         │ 150                        │
│ Partition Leader Epoch (4)     │ 1                          │
│ Magic Byte (1)                 │ 2 (Kafka 0.11+)            │
│ CRC (4 bytes)                  │ checksum                   │
│ Attributes (2 bytes)           │ compression, timestamp     │
│ Last Offset Delta (4)          │ 9                          │
│ First Timestamp (8)            │ 1640000000000              │
│ Max Timestamp (8)              │ 1640000001000              │
│ Producer ID (8)                │ -1 (no idempotent)         │
│ Producer Epoch (2)             │ -1                         │
│ Base Sequence (4)              │ -1                         │
│ Records Count (4)              │ 10                         │
├─────────────────────────────────────────────────────────────┤
│ Record 0                                                    │
│ Record 1                                                    │
│ ...                                                         │
│ Record 9                                                    │
└─────────────────────────────────────────────────────────────┘
```

#### Offset Index (.index)

Sparse index для быстрого поиска по offset:

```
┌──────────────────────────────────────┐
│ Relative Offset │ Physical Position │
├──────────────────────────────────────┤
│ 0               │ 0                 │
│ 100             │ 8192              │
│ 200             │ 16384             │
│ 300             │ 24576             │
└──────────────────────────────────────┘
```

```properties
# Интервал записи в index (каждые 4KB данных)
log.index.interval.bytes=4096
```

#### Timestamp Index (.timeindex)

Индекс для поиска по времени:

```
┌──────────────────────────────────────┐
│ Timestamp       │ Relative Offset   │
├──────────────────────────────────────┤
│ 1640000000000   │ 0                 │
│ 1640000100000   │ 100               │
│ 1640000200000   │ 200               │
└──────────────────────────────────────┘
```

### Механизмы перемещения данных

#### 1. Репликация между брокерами

```
Producer → Leader Broker → Follower Broker 1
                        → Follower Broker 2
```

```properties
# Настройки репликации
num.replica.fetchers=1
replica.fetch.min.bytes=1
replica.fetch.max.bytes=1048576
replica.fetch.wait.max.ms=500

# High Watermark
replica.high.watermark.checkpoint.interval.ms=5000
```

#### 2. Fetch Protocol

Consumer и follower брокеры используют одинаковый протокол:

```
FetchRequest {
  replica_id: -1 (consumer) или broker_id (follower)
  max_wait_ms: 500
  min_bytes: 1
  max_bytes: 52428800
  topics: [
    {
      topic: "events"
      partitions: [
        {
          partition: 0
          fetch_offset: 12345
          partition_max_bytes: 1048576
        }
      ]
    }
  ]
}
```

#### 3. Zero-Copy Transfer

Kafka использует sendfile() для эффективной передачи данных:

```
Традиционный путь:
Disk → Kernel Buffer → User Buffer → Socket Buffer → NIC

Zero-Copy путь:
Disk → Kernel Buffer → NIC (через DMA)
```

```java
// Java NIO FileChannel.transferTo()
fileChannel.transferTo(position, count, socketChannel);
```

### MirrorMaker 2 для межкластерной репликации

#### Архитектура MM2

```
┌─────────────────┐         ┌─────────────────┐
│  Source Cluster │  ──→    │  Target Cluster │
│    (us-east)    │  MM2    │    (us-west)    │
└─────────────────┘         └─────────────────┘
```

#### Конфигурация MM2

```properties
# mm2.properties
clusters = source, target

source.bootstrap.servers = source-kafka:9092
target.bootstrap.servers = target-kafka:9092

# Репликация source → target
source->target.enabled = true
source->target.topics = events.*

# Синхронизация consumer offsets
sync.group.offsets.enabled = true
sync.group.offsets.interval.seconds = 60

# Heartbeats для мониторинга
emit.heartbeats.enabled = true
emit.checkpoints.enabled = true
```

```bash
# Запуск MirrorMaker 2
connect-mirror-maker.sh mm2.properties
```

## Примеры конфигурации

### Оптимизация Disk I/O

```properties
# Настройки файловой системы
log.dirs=/kafka-data/disk1,/kafka-data/disk2,/kafka-data/disk3

# Flush политика (обычно полагаемся на OS page cache)
log.flush.interval.messages=10000
log.flush.interval.ms=1000

# Размер буфера для записи
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
```

### Настройка сегментов для разных сценариев

```properties
# Высоконагруженный топик (частые ротации)
log.segment.bytes=536870912      # 512 MB
log.roll.ms=3600000              # 1 час

# Топик с редкими записями
log.segment.bytes=1073741824     # 1 GB
log.roll.hours=168               # 7 дней

# Compacted топик (state store)
log.segment.bytes=104857600      # 100 MB
log.cleanup.policy=compact
```

### Настройка репликации

```properties
# Количество потоков для fetch
num.replica.fetchers=4

# Таймауты репликации
replica.lag.time.max.ms=30000
replica.socket.timeout.ms=30000
replica.socket.receive.buffer.bytes=65536

# Unclean leader election (осторожно!)
unclean.leader.election.enable=false
```

### Kafka Connect для внешних систем

```json
{
  "name": "jdbc-sink-connector",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "tasks.max": "3",
    "topics": "orders",
    "connection.url": "jdbc:postgresql://db:5432/warehouse",
    "auto.create": "true",
    "insert.mode": "upsert",
    "pk.mode": "record_key",
    "pk.fields": "order_id"
  }
}
```

## Best Practices

### 1. Выбор файловой системы

| FS | Рекомендация |
|----|--------------|
| XFS | Предпочтительно для Linux |
| ext4 | Хороший выбор, широко поддерживается |
| ZFS | Хорошо для tiered storage |

```bash
# Рекомендуемые mount options для XFS
mount -o noatime,nodiratime,nobarrier /dev/sda1 /kafka-data
```

### 2. RAID конфигурация

```
Рекомендация: JBOD (Just a Bunch of Disks)
- Kafka уже обеспечивает репликацию
- JBOD даёт лучшую производительность
- Проще управлять отказами дисков

НЕ рекомендуется: RAID 5/6
- Накладные расходы на parity
- Write penalty
```

### 3. Размер страницы памяти

```bash
# Включение huge pages для лучшей производительности
echo 'vm.nr_hugepages=1024' >> /etc/sysctl.conf
sysctl -p
```

### 4. Мониторинг перемещения данных

```bash
# Проверка lag репликации
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group __consumer_offsets

# Проверка ISR
kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe --topic events

# Under-replicated partitions
kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe --under-replicated-partitions
```

### 5. Типичные проблемы и решения

**Проблема:** Медленная репликация (high replica lag)
```properties
# Увеличьте количество fetcher потоков
num.replica.fetchers=4

# Увеличьте размер batch
replica.fetch.max.bytes=10485760
```

**Проблема:** Долгий поиск по offset
```properties
# Уменьшите интервал индексирования
log.index.interval.bytes=2048
```

**Проблема:** Высокий disk I/O при чтении
```properties
# Увеличьте размер OS page cache
# Kafka активно использует page cache
# Рекомендуется 25-50% RAM для page cache
```

## Инструменты для работы с данными

### kafka-dump-log

```bash
# Просмотр содержимого сегмента
kafka-dump-log.sh --files /kafka-data/events-0/00000000000000000000.log \
  --print-data-log

# Проверка индекса
kafka-dump-log.sh --files /kafka-data/events-0/00000000000000000000.index \
  --index-sanity-check
```

### kafka-replica-verification

```bash
# Проверка консистентности реплик
kafka-replica-verification.sh \
  --broker-list broker1:9092,broker2:9092 \
  --topic-white-list "events.*"
```

## Связанные темы

- [Срок хранения данных](./01-data-retention.md) — политики retention
- [Инструменты](./03-tools.md) — CLI утилиты
- [Архитектуры](./05-architectures.md) — паттерны архитектуры
- [Мульти-кластер](./06-multi-cluster.md) — межкластерная репликация

---

[prev: 01-data-retention](./01-data-retention.md) | [next: 03-tools](./03-tools.md)
