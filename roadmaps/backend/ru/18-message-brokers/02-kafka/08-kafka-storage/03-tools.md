# Инструменты для работы с Kafka Storage

## Описание

Kafka предоставляет набор CLI-инструментов для управления хранилищем, диагностики проблем и работы с данными. Эти утилиты критически важны для администрирования кластера и отладки.

## Ключевые концепции

### Категории инструментов

1. **Управление топиками и данными** — создание, удаление, настройка
2. **Диагностика хранилища** — анализ логов, индексов, сегментов
3. **Мониторинг** — проверка состояния брокеров и партиций
4. **Миграция данных** — перемещение партиций между брокерами

## Основные CLI инструменты

### kafka-topics.sh

Управление топиками:

```bash
# Создание топика
kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic events \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=604800000 \
  --config cleanup.policy=delete

# Список всех топиков
kafka-topics.sh --list --bootstrap-server localhost:9092

# Описание топика
kafka-topics.sh --describe \
  --bootstrap-server localhost:9092 \
  --topic events

# Увеличение количества партиций
kafka-topics.sh --alter \
  --bootstrap-server localhost:9092 \
  --topic events \
  --partitions 24

# Удаление топика
kafka-topics.sh --delete \
  --bootstrap-server localhost:9092 \
  --topic events

# Поиск проблемных партиций
kafka-topics.sh --describe \
  --bootstrap-server localhost:9092 \
  --under-replicated-partitions

kafka-topics.sh --describe \
  --bootstrap-server localhost:9092 \
  --unavailable-partitions
```

### kafka-configs.sh

Управление конфигурацией:

```bash
# Просмотр конфигурации топика
kafka-configs.sh --describe \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name events

# Изменение конфигурации топика
kafka-configs.sh --alter \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name events \
  --add-config retention.ms=172800000,max.message.bytes=10485760

# Удаление переопределённой конфигурации
kafka-configs.sh --alter \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name events \
  --delete-config retention.ms

# Конфигурация брокера
kafka-configs.sh --describe \
  --bootstrap-server localhost:9092 \
  --entity-type brokers \
  --entity-name 0

# Динамическое изменение настроек брокера
kafka-configs.sh --alter \
  --bootstrap-server localhost:9092 \
  --entity-type brokers \
  --entity-name 0 \
  --add-config log.cleaner.threads=4

# Конфигурация клиентских квот
kafka-configs.sh --alter \
  --bootstrap-server localhost:9092 \
  --entity-type users \
  --entity-name admin \
  --add-config producer_byte_rate=1048576,consumer_byte_rate=2097152
```

### kafka-log-dirs.sh

Информация о директориях логов:

```bash
# Описание всех log directories
kafka-log-dirs.sh --describe \
  --bootstrap-server localhost:9092

# Информация по конкретному брокеру
kafka-log-dirs.sh --describe \
  --bootstrap-server localhost:9092 \
  --broker-list 0,1,2

# Информация по конкретным топикам
kafka-log-dirs.sh --describe \
  --bootstrap-server localhost:9092 \
  --topic-list events,orders

# Формат вывода JSON
kafka-log-dirs.sh --describe \
  --bootstrap-server localhost:9092 \
  --topic-list events 2>/dev/null | jq '.brokers[].logDirs[].partitions'
```

Пример вывода:
```json
{
  "brokers": [{
    "broker": 0,
    "logDirs": [{
      "logDir": "/kafka-data",
      "error": null,
      "partitions": [{
        "partition": "events-0",
        "size": 1073741824,
        "offsetLag": 0,
        "isFuture": false
      }]
    }]
  }]
}
```

### kafka-dump-log.sh

Анализ файлов логов и индексов:

```bash
# Дамп содержимого лога (метаданные)
kafka-dump-log.sh \
  --files /kafka-data/events-0/00000000000000000000.log

# Дамп с содержимым сообщений
kafka-dump-log.sh \
  --files /kafka-data/events-0/00000000000000000000.log \
  --print-data-log

# Дамп offset index
kafka-dump-log.sh \
  --files /kafka-data/events-0/00000000000000000000.index

# Дамп timestamp index
kafka-dump-log.sh \
  --files /kafka-data/events-0/00000000000000000000.timeindex

# Проверка целостности индекса
kafka-dump-log.sh \
  --files /kafka-data/events-0/00000000000000000000.index \
  --index-sanity-check

# Проверка producer state snapshot
kafka-dump-log.sh \
  --files /kafka-data/events-0/00000000000000000000.snapshot

# Глубокий анализ итератором
kafka-dump-log.sh \
  --files /kafka-data/events-0/00000000000000000000.log \
  --deep-iteration \
  --print-data-log
```

Пример вывода:
```
Dumping /kafka-data/events-0/00000000000000000000.log
Starting offset: 0
baseOffset: 0 lastOffset: 9 count: 10 baseSequence: -1 lastSequence: -1
producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0
isTransactional: false isControl: false position: 0
CreateTime: 1640000000000 size: 512 magic: 2 compresscodec: none
crc: 1234567890 isvalid: true
| offset: 0 CreateTime: 1640000000000 keySize: 4 valueSize: 100 ...
| offset: 1 CreateTime: 1640000000100 keySize: 4 valueSize: 100 ...
```

### kafka-consumer-groups.sh

Управление consumer groups:

```bash
# Список consumer groups
kafka-consumer-groups.sh --list \
  --bootstrap-server localhost:9092

# Описание consumer group
kafka-consumer-groups.sh --describe \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group

# Описание всех групп
kafka-consumer-groups.sh --describe \
  --bootstrap-server localhost:9092 \
  --all-groups

# Сброс offsets на начало
kafka-consumer-groups.sh --reset-offsets \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --topic events \
  --to-earliest \
  --execute

# Сброс на конец
kafka-consumer-groups.sh --reset-offsets \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --topic events \
  --to-latest \
  --execute

# Сброс на конкретный offset
kafka-consumer-groups.sh --reset-offsets \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --topic events:0 \
  --to-offset 1000 \
  --execute

# Сброс по времени
kafka-consumer-groups.sh --reset-offsets \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --topic events \
  --to-datetime 2024-01-01T00:00:00.000 \
  --execute

# Сдвиг offsets
kafka-consumer-groups.sh --reset-offsets \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --topic events \
  --shift-by -100 \
  --execute

# Удаление consumer group
kafka-consumer-groups.sh --delete \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group
```

### kafka-reassign-partitions.sh

Перемещение партиций между брокерами:

```bash
# Шаг 1: Генерация плана reassignment
# Создайте файл topics-to-move.json:
cat > topics-to-move.json << 'EOF'
{
  "topics": [
    {"topic": "events"},
    {"topic": "orders"}
  ],
  "version": 1
}
EOF

# Генерация плана
kafka-reassign-partitions.sh \
  --bootstrap-server localhost:9092 \
  --topics-to-move-json-file topics-to-move.json \
  --broker-list "0,1,2,3" \
  --generate

# Шаг 2: Ручное создание плана
cat > reassignment.json << 'EOF'
{
  "version": 1,
  "partitions": [
    {"topic": "events", "partition": 0, "replicas": [1, 2, 3]},
    {"topic": "events", "partition": 1, "replicas": [2, 3, 0]},
    {"topic": "events", "partition": 2, "replicas": [3, 0, 1]}
  ]
}
EOF

# Шаг 3: Выполнение reassignment
kafka-reassign-partitions.sh \
  --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --execute

# Шаг 4: Проверка прогресса
kafka-reassign-partitions.sh \
  --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --verify

# Отмена reassignment
kafka-reassign-partitions.sh \
  --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --cancel

# Ограничение скорости репликации
kafka-reassign-partitions.sh \
  --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --execute \
  --throttle 50000000
```

### kafka-delete-records.sh

Удаление записей до определённого offset:

```bash
# Создайте файл с offsets для удаления
cat > offsets.json << 'EOF'
{
  "partitions": [
    {"topic": "events", "partition": 0, "offset": 1000},
    {"topic": "events", "partition": 1, "offset": 2000}
  ],
  "version": 1
}
EOF

# Удаление записей
kafka-delete-records.sh \
  --bootstrap-server localhost:9092 \
  --offset-json-file offsets.json
```

### kafka-replica-verification.sh

Проверка консистентности реплик:

```bash
# Проверка всех топиков
kafka-replica-verification.sh \
  --broker-list localhost:9092

# Проверка конкретных топиков
kafka-replica-verification.sh \
  --broker-list localhost:9092 \
  --topic-white-list "events.*"

# С интервалом проверки
kafka-replica-verification.sh \
  --broker-list localhost:9092 \
  --fetch-size 1048576 \
  --report-interval-ms 10000
```

## Примеры конфигурации

### Скрипт для анализа хранилища

```bash
#!/bin/bash
# storage-analysis.sh

BOOTSTRAP="localhost:9092"
DATA_DIR="/kafka-data"

echo "=== Kafka Storage Analysis ==="

# Размер топиков
echo -e "\n--- Topic Sizes ---"
kafka-log-dirs.sh --describe \
  --bootstrap-server $BOOTSTRAP 2>/dev/null | \
  jq -r '.brokers[].logDirs[].partitions[] | "\(.partition): \(.size / 1024 / 1024 | floor) MB"' | \
  sort -t: -k2 -rn | head -20

# Under-replicated partitions
echo -e "\n--- Under-replicated Partitions ---"
kafka-topics.sh --describe \
  --bootstrap-server $BOOTSTRAP \
  --under-replicated-partitions

# Consumer group lag
echo -e "\n--- Consumer Lag (top 10) ---"
kafka-consumer-groups.sh --list \
  --bootstrap-server $BOOTSTRAP 2>/dev/null | \
while read group; do
  kafka-consumer-groups.sh --describe \
    --bootstrap-server $BOOTSTRAP \
    --group "$group" 2>/dev/null | tail -n +3
done | awk '{sum[$1]+=$6} END {for(t in sum) print t, sum[t]}' | \
  sort -k2 -rn | head -10

# Disk usage
echo -e "\n--- Disk Usage ---"
du -sh $DATA_DIR/*/ 2>/dev/null | sort -rh | head -10
```

### Скрипт для сброса offsets

```bash
#!/bin/bash
# reset-consumer-offsets.sh

BOOTSTRAP="localhost:9092"
GROUP=$1
TOPIC=$2
STRATEGY=$3  # earliest, latest, datetime, offset

if [ -z "$GROUP" ] || [ -z "$TOPIC" ] || [ -z "$STRATEGY" ]; then
  echo "Usage: $0 <group> <topic> <earliest|latest|datetime|offset> [value]"
  exit 1
fi

case $STRATEGY in
  earliest)
    kafka-consumer-groups.sh --reset-offsets \
      --bootstrap-server $BOOTSTRAP \
      --group $GROUP \
      --topic $TOPIC \
      --to-earliest \
      --execute
    ;;
  latest)
    kafka-consumer-groups.sh --reset-offsets \
      --bootstrap-server $BOOTSTRAP \
      --group $GROUP \
      --topic $TOPIC \
      --to-latest \
      --execute
    ;;
  datetime)
    kafka-consumer-groups.sh --reset-offsets \
      --bootstrap-server $BOOTSTRAP \
      --group $GROUP \
      --topic $TOPIC \
      --to-datetime $4 \
      --execute
    ;;
  offset)
    kafka-consumer-groups.sh --reset-offsets \
      --bootstrap-server $BOOTSTRAP \
      --group $GROUP \
      --topic $TOPIC \
      --to-offset $4 \
      --execute
    ;;
esac
```

### Автоматизация reassignment

```bash
#!/bin/bash
# safe-reassign.sh

BOOTSTRAP="localhost:9092"
THROTTLE=100000000  # 100 MB/s
JSON_FILE=$1

if [ -z "$JSON_FILE" ]; then
  echo "Usage: $0 <reassignment.json>"
  exit 1
fi

echo "Starting reassignment with throttle: $THROTTLE bytes/sec"

# Запуск reassignment
kafka-reassign-partitions.sh \
  --bootstrap-server $BOOTSTRAP \
  --reassignment-json-file $JSON_FILE \
  --execute \
  --throttle $THROTTLE

# Мониторинг прогресса
while true; do
  STATUS=$(kafka-reassign-partitions.sh \
    --bootstrap-server $BOOTSTRAP \
    --reassignment-json-file $JSON_FILE \
    --verify 2>&1)

  echo "$STATUS"

  if echo "$STATUS" | grep -q "Reassignment of partition .* is complete"; then
    echo "Reassignment complete!"
    break
  fi

  sleep 30
done

# Удаление throttle
kafka-configs.sh --alter \
  --bootstrap-server $BOOTSTRAP \
  --entity-type brokers \
  --entity-default \
  --delete-config follower.replication.throttled.rate,leader.replication.throttled.rate
```

## Best Practices

### 1. Безопасное использование инструментов

```bash
# Всегда используйте --dry-run где возможно
kafka-consumer-groups.sh --reset-offsets \
  --bootstrap-server localhost:9092 \
  --group my-group \
  --topic events \
  --to-earliest \
  --dry-run  # Сначала проверьте план

# Проверяйте конфигурацию перед изменением
kafka-configs.sh --describe \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name events
```

### 2. Мониторинг операций

```bash
# Создайте алиас для частых команд
alias kafka-topics='kafka-topics.sh --bootstrap-server localhost:9092'
alias kafka-groups='kafka-consumer-groups.sh --bootstrap-server localhost:9092'
alias kafka-configs='kafka-configs.sh --bootstrap-server localhost:9092'

# Используйте watch для мониторинга
watch -n 5 'kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-group'
```

### 3. Резервное копирование перед операциями

```bash
# Экспорт текущих offsets
kafka-consumer-groups.sh --describe \
  --bootstrap-server localhost:9092 \
  --group my-group > offsets-backup-$(date +%Y%m%d).txt

# Экспорт конфигурации топика
kafka-configs.sh --describe \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name events > events-config-backup.txt
```

### 4. Работа с большими кластерами

```bash
# Используйте параллельную обработку
for topic in $(kafka-topics.sh --list --bootstrap-server localhost:9092); do
  kafka-topics.sh --describe \
    --bootstrap-server localhost:9092 \
    --topic $topic &
done
wait

# Ограничивайте вывод
kafka-log-dirs.sh --describe \
  --bootstrap-server localhost:9092 \
  --topic-list important-topic  # Только нужные топики
```

### 5. Типичные проблемы

| Проблема | Решение |
|----------|---------|
| Connection refused | Проверьте listeners и advertised.listeners |
| Topic not found | Проверьте auto.create.topics.enable |
| Timeout | Увеличьте request.timeout.ms |
| Access denied | Проверьте ACLs и SASL конфигурацию |

## Связанные темы

- [Срок хранения данных](./01-data-retention.md) — настройка retention
- [Перемещение данных](./02-data-movement.md) — структура хранения
- [Мульти-кластер](./06-multi-cluster.md) — управление кластерами
