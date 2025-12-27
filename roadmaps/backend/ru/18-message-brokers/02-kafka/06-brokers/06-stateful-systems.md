# Системы с сохранением состояния

## Описание

Kafka является **stateful системой** — брокеры хранят данные на диске и поддерживают состояние (метаданные, оффсеты, ISR). Это отличает Kafka от stateless систем и требует особого подхода к проектированию, развёртыванию и операционному обслуживанию кластера.

## Ключевые концепции

### Stateful vs Stateless

```
┌─────────────────────────────────────────────────────────────┐
│              Stateful vs Stateless Systems                   │
│                                                              │
│  Stateless (например, web server)                           │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  • Не хранит данные между запросами                 │    │
│  │  • Легко масштабировать горизонтально               │    │
│  │  • Любой экземпляр может обработать любой запрос   │    │
│  │  • Простое восстановление — просто перезапуск       │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  Stateful (Kafka, базы данных)                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  • Хранит данные на диске                           │    │
│  │  • Масштабирование требует перераспределения данных │    │
│  │  • Запросы привязаны к конкретным узлам             │    │
│  │  • Восстановление требует синхронизации данных      │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Что хранит Kafka

```
┌─────────────────────────────────────────────────────────────┐
│                    Kafka State                               │
│                                                              │
│  На диске (log.dirs):                                        │
│  ├── Topic data                                              │
│  │   ├── Сегменты логов (.log)                              │
│  │   ├── Индексы оффсетов (.index)                          │
│  │   └── Индексы временных меток (.timeindex)               │
│  │                                                           │
│  ├── Consumer offsets (__consumer_offsets)                  │
│  │   └── Оффсеты всех consumer groups                       │
│  │                                                           │
│  └── Transaction state (__transaction_state)                │
│      └── Состояние активных транзакций                      │
│                                                              │
│  В памяти:                                                   │
│  ├── Метаданные топиков и партиций                          │
│  ├── ISR для каждой партиции                                │
│  ├── Page cache (кэш ОС для данных)                         │
│  └── Состояние соединений клиентов                          │
│                                                              │
│  В ZooKeeper/KRaft:                                          │
│  ├── Cluster metadata                                        │
│  ├── Controller election                                     │
│  ├── Broker registration                                     │
│  └── Topic configuration                                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Структура хранения на диске

```
/var/kafka-logs/
├── my-topic-0/                    # Партиция 0 топика my-topic
│   ├── 00000000000000000000.log   # Первый сегмент
│   ├── 00000000000000000000.index # Offset index
│   ├── 00000000000000000000.timeindex  # Time index
│   ├── 00000000000012345678.log   # Второй сегмент (начало с offset 12345678)
│   ├── 00000000000012345678.index
│   ├── 00000000000012345678.timeindex
│   ├── leader-epoch-checkpoint    # Эпохи лидеров
│   └── partition.metadata         # Метаданные партиции
├── my-topic-1/
│   └── ...
├── __consumer_offsets-0/          # Consumer offsets партиция 0
│   └── ...
├── cleaner-offset-checkpoint      # Checkpoint для log compaction
├── log-start-offset-checkpoint    # Минимальные оффсеты партиций
├── recovery-point-offset-checkpoint # Точки восстановления
└── replication-offset-checkpoint  # Оффсеты репликации
```

## Примеры конфигурации

### Настройки хранения

```properties
############################# Storage Settings #############################

# Директории для данных (несколько для JBOD)
log.dirs=/data/kafka-logs-1,/data/kafka-logs-2,/data/kafka-logs-3

# Размер сегмента лога
log.segment.bytes=1073741824  # 1 GB

# Время до ротации сегмента
log.roll.hours=168  # 7 дней

# Время хранения данных
log.retention.hours=168  # 7 дней
# Или по размеру:
log.retention.bytes=-1  # без лимита

# Интервал проверки для удаления
log.retention.check.interval.ms=300000  # 5 минут

# Настройки индексов
log.index.size.max.bytes=10485760  # 10 MB
log.index.interval.bytes=4096  # 4 KB
```

### Настройки для надёжности данных

```properties
############################# Durability Settings #############################

# Репликация
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false

# Flush настройки (обычно не рекомендуется менять)
log.flush.interval.messages=10000
log.flush.interval.ms=1000

# Гарантии при записи
# Используйте acks=all на producer side

# Восстановление
num.recovery.threads.per.data.dir=2
```

### Настройки для больших объёмов данных

```properties
############################# Large Volume Settings #############################

# Использование JBOD (Just a Bunch of Disks)
log.dirs=/disk1/kafka,/disk2/kafka,/disk3/kafka,/disk4/kafka

# Увеличенные потоки для восстановления
num.recovery.threads.per.data.dir=4

# Log compaction для систем с обновлениями
log.cleanup.policy=compact
log.cleaner.enable=true
log.cleaner.threads=4

# Для tiered storage (Kafka 3.6+)
# remote.log.storage.system.enable=true
```

## Управление состоянием кластера

### Добавление нового брокера

```bash
# 1. Настроить server.properties с новым broker.id
broker.id=4

# 2. Запустить брокер
bin/kafka-server-start.sh config/server.properties

# 3. Перераспределить партиции на новый брокер
cat > reassignment.json << 'EOF'
{
  "version": 1,
  "partitions": [
    {"topic": "my-topic", "partition": 0, "replicas": [1, 2, 4]},
    {"topic": "my-topic", "partition": 1, "replicas": [2, 3, 4]},
    {"topic": "my-topic", "partition": 2, "replicas": [3, 4, 1]}
  ]
}
EOF

bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --execute

# 4. Мониторить прогресс
bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --verify
```

### Удаление брокера

```bash
# 1. Перенести все партиции с удаляемого брокера
# Сгенерировать план перераспределения
bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --topics-to-move-json-file topics.json \
  --broker-list "1,2,3" \
  --generate

# 2. Выполнить перераспределение
bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --execute

# 3. Дождаться завершения
bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --verify

# 4. Остановить брокер
bin/kafka-server-stop.sh
```

### Замена диска

```
┌─────────────────────────────────────────────────────────────┐
│                    Disk Replacement                          │
│                                                              │
│  Сценарий: Диск /disk2 вышел из строя                       │
│                                                              │
│  1. Если брокер упал — перезапустить без /disk2             │
│     log.dirs=/disk1/kafka,/disk3/kafka                      │
│                                                              │
│  2. Партиции с /disk2 станут under-replicated               │
│                                                              │
│  3. Заменить диск физически                                 │
│                                                              │
│  4. Добавить /disk2 обратно в log.dirs                      │
│     log.dirs=/disk1/kafka,/disk2/kafka,/disk3/kafka         │
│                                                              │
│  5. Перезапустить брокер                                    │
│                                                              │
│  6. Перераспределить партиции на новый диск                 │
│     bin/kafka-reassign-partitions.sh ...                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Резервное копирование и восстановление

### Стратегии резервного копирования

```
┌─────────────────────────────────────────────────────────────┐
│                    Backup Strategies                         │
│                                                              │
│  1. MirrorMaker 2 (рекомендуется)                           │
│     ├─ Реплицирует данные в другой кластер                 │
│     ├─ Сохраняет оффсеты и consumer groups                 │
│     └─ Подходит для DR (Disaster Recovery)                  │
│                                                              │
│  2. Filesystem Snapshots                                     │
│     ├─ ZFS/LVM snapshots дисков                             │
│     ├─ Быстро, но требует остановки или pause              │
│     └─ Консистентность только при остановке                │
│                                                              │
│  3. Object Storage (Tiered Storage)                         │
│     ├─ Kafka 3.6+ с tiered storage                         │
│     ├─ Старые данные автоматически в S3/GCS                │
│     └─ Экономия места на брокерах                          │
│                                                              │
│  4. Consumer-based backup                                    │
│     ├─ Consumer читает данные и сохраняет                  │
│     ├─ Гибко, но медленно                                  │
│     └─ Подходит для выборочного бэкапа                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### MirrorMaker 2 конфигурация

```properties
# mm2.properties

# Кластеры
clusters=source,target
source.bootstrap.servers=source-kafka:9092
target.bootstrap.servers=target-kafka:9092

# Репликация
source->target.enabled=true
source->target.topics=.*

# Синхронизация consumer groups
source->target.groups=.*
sync.group.offsets.enabled=true
sync.group.offsets.interval.seconds=60

# Heartbeats
heartbeats.topic.replication.factor=3
checkpoints.topic.replication.factor=3
offset-syncs.topic.replication.factor=3

# Производительность
replication.factor=3
tasks.max=10
```

```bash
# Запуск MirrorMaker 2
bin/connect-mirror-maker.sh mm2.properties
```

## Disaster Recovery

### Сценарии отказов

```
┌─────────────────────────────────────────────────────────────┐
│                    Failure Scenarios                         │
│                                                              │
│  1. Отказ одного брокера                                    │
│     ├─ Автоматический failover лидеров                     │
│     ├─ Данные доступны через реплики                       │
│     └─ Восстановление: перезапуск или замена               │
│                                                              │
│  2. Отказ нескольких брокеров                               │
│     ├─ Если < min.insync.replicas — запись невозможна     │
│     ├─ Чтение возможно с оставшихся реплик                 │
│     └─ Приоритет: восстановить ISR                         │
│                                                              │
│  3. Потеря всех реплик партиции                             │
│     ├─ Партиция offline                                     │
│     ├─ Если unclean.leader.election=true — потеря данных   │
│     └─ Восстановление из бэкапа                            │
│                                                              │
│  4. Потеря датацентра                                       │
│     ├─ Failover на DR кластер                              │
│     ├─ Переключение клиентов                               │
│     └─ Синхронизация после восстановления                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Recovery процедуры

```bash
# Сценарий: Брокер не запускается из-за повреждённых данных

# 1. Проверить логи
tail -f /var/log/kafka/server.log

# 2. Если повреждён индекс — удалить для перестроения
rm /var/kafka-logs/my-topic-0/*.index
rm /var/kafka-logs/my-topic-0/*.timeindex

# 3. Если повреждён сегмент — удалить (потеря данных!)
# Лучше восстановить из реплики

# 4. Проверить checkpoint файлы
cat /var/kafka-logs/recovery-point-offset-checkpoint

# 5. При необходимости — очистить состояние ZK
# (только если понимаете последствия!)
bin/zookeeper-shell.sh localhost:2181
# rmr /brokers/ids/1
```

## Мониторинг состояния

### Ключевые метрики хранения

```bash
# Размер данных на диске
du -sh /var/kafka-logs/

# Метрики по топикам
kafka.log:type=Log,name=Size,topic=*,partition=*
kafka.log:type=Log,name=NumLogSegments,topic=*,partition=*

# Метрики очистки
kafka.log:type=LogCleaner,name=max-clean-time-secs
kafka.log:type=LogCleaner,name=cleaner-recopy-percent

# Метрики flush
kafka.log:type=LogFlushStats,name=LogFlushRateAndTimeMs

# Дисковое пространство
df -h /var/kafka-logs/
```

### Алерты

```yaml
# Prometheus alerting rules пример
groups:
  - name: kafka-storage
    rules:
      - alert: KafkaDiskSpaceLow
        expr: kafka_log_log_size / kafka_log_log_start_offset > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk space is running low on {{ $labels.instance }}"

      - alert: KafkaUnderReplicatedPartitions
        expr: kafka_server_replicamanager_underreplicatedpartitions > 0
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Under-replicated partitions detected"

      - alert: KafkaOfflinePartitions
        expr: kafka_controller_kafkacontroller_offlinepartitionscount > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Offline partitions detected"
```

## Best Practices

### 1. Планирование ёмкости

```
Расчёт дискового пространства:

Объём данных = (сообщений/сек) × (размер сообщения) × (retention time) × (replication factor)

Пример:
- 10,000 msg/sec
- 1 KB average size
- 7 days retention
- RF=3

= 10,000 × 1 KB × 604,800 sec × 3 = ~18 TB

+ 20% запас = ~22 TB на кластер
```

### 2. Конфигурация для durability

```properties
# Максимальная надёжность
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false

# Producer settings
acks=all
enable.idempotence=true
retries=Integer.MAX_VALUE
max.in.flight.requests.per.connection=5
```

### 3. Распределение данных

```properties
# Используйте несколько log.dirs на разных дисках
log.dirs=/disk1/kafka,/disk2/kafka,/disk3/kafka

# Включите автобалансировку (Kafka 3.0+)
log.dirs.failure.handling.policy=MoveCopy
```

### 4. Мониторинг и алертинг

```
Критические алерты:
- OfflinePartitionsCount > 0
- UnderReplicatedPartitions > 0 (более 10 минут)
- Disk usage > 85%
- ISR shrink rate высокий

Предупреждения:
- Disk usage > 70%
- Consumer lag растёт
- Request latency увеличивается
```

### 5. Документация и runbooks

```
Создайте runbooks для:
- Замена отказавшего брокера
- Восстановление из бэкапа
- Расширение кластера
- Обновление версии Kafka
- Миграция данных между кластерами
```

## Tiered Storage (Kafka 3.6+)

### Концепция

```
┌─────────────────────────────────────────────────────────────┐
│                    Tiered Storage                            │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    Local Tier                        │    │
│  │  • Горячие данные на локальных дисках брокеров      │    │
│  │  • Быстрый доступ                                   │    │
│  │  • Ограниченный retention                           │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Remote Tier                        │    │
│  │  • Холодные данные в object storage (S3, GCS, etc) │    │
│  │  • Неограниченное хранение                          │    │
│  │  • Экономичный доступ                               │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Конфигурация

```properties
# Включение tiered storage
remote.log.storage.system.enable=true
remote.log.storage.manager.class.name=org.apache.kafka.server.log.remote.storage.RemoteLogManager

# S3 конфигурация (пример)
remote.log.storage.manager.class.path=/path/to/s3-plugin.jar
remote.log.storage.s3.bucket.name=kafka-tiered-storage
remote.log.storage.s3.region=us-east-1

# Retention локального tier
local.retention.bytes=10737418240  # 10 GB
local.retention.ms=86400000  # 1 день

# Общий retention (включая remote)
log.retention.hours=8760  # 1 год
log.retention.bytes=-1
```
