# Возврат данных в Kafka

[prev: 03-tools](./03-tools.md) | [next: 05-architectures](./05-architectures.md)

---

## Описание

Возврат данных в Kafka (Data Return) — это процесс восстановления или повторной загрузки данных обратно в топики Kafka. Это может быть необходимо после аварии, при миграции между кластерами, или для восстановления данных из внешних хранилищ (S3, HDFS, базы данных).

## Ключевые концепции

### Сценарии возврата данных

1. **Disaster Recovery** — восстановление после сбоя кластера
2. **Миграция** — перенос данных между кластерами или регионами
3. **Реплей событий** — повторная обработка исторических данных
4. **Backfill** — загрузка исторических данных в новый топик
5. **CDC Recovery** — восстановление состояния из внешнего хранилища

### Источники данных для восстановления

```
┌─────────────────────────────────────────────────────────────┐
│                    Источники данных                         │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────────────┐ │
│  │   S3    │  │  HDFS   │  │Database │  │ Another Kafka   │ │
│  │ Backup  │  │ Archive │  │  (CDC)  │  │    Cluster      │ │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────────┬────────┘ │
│       │            │            │                │          │
│       └────────────┴────────────┴────────────────┘          │
│                            │                                 │
│                     ┌──────▼──────┐                         │
│                     │   Kafka     │                         │
│                     │   Cluster   │                         │
│                     └─────────────┘                         │
└─────────────────────────────────────────────────────────────┘
```

## Методы возврата данных

### 1. MirrorMaker 2 (Межкластерная репликация)

Для восстановления из резервного кластера:

```properties
# mm2-recovery.properties
clusters = backup, primary

backup.bootstrap.servers = backup-kafka:9092
primary.bootstrap.servers = primary-kafka:9092

# Восстановление из backup в primary
backup->primary.enabled = true
backup->primary.topics = .*

# Сохранение оригинальных offsets
replication.policy.class = org.apache.kafka.connect.mirror.IdentityReplicationPolicy

# Синхронизация consumer offsets
sync.group.offsets.enabled = true
emit.checkpoints.enabled = true
```

```bash
# Запуск восстановления
connect-mirror-maker.sh mm2-recovery.properties
```

### 2. Kafka Connect S3 Source

Восстановление из S3:

```json
{
  "name": "s3-source-recovery",
  "config": {
    "connector.class": "io.confluent.connect.s3.source.S3SourceConnector",
    "tasks.max": "10",
    "s3.bucket.name": "kafka-backup-bucket",
    "s3.region": "us-east-1",
    "topics.dir": "topics",
    "format.class": "io.confluent.connect.s3.format.avro.AvroFormat",
    "mode": "RESTORE_BACKUP",
    "confluent.topic.bootstrap.servers": "kafka:9092",

    "behavior.on.error": "FAIL",
    "s3.proxy.url": "",
    "s3.proxy.user": "",
    "s3.proxy.password": ""
  }
}
```

### 3. Kafka Connect JDBC Source

Восстановление из базы данных:

```json
{
  "name": "jdbc-source-recovery",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "tasks.max": "4",
    "connection.url": "jdbc:postgresql://db:5432/archive",
    "connection.user": "kafka",
    "connection.password": "${file:/secrets/db.properties:password}",

    "mode": "timestamp+incrementing",
    "timestamp.column.name": "updated_at",
    "incrementing.column.name": "id",

    "query": "SELECT * FROM events WHERE archived = true",
    "topic.prefix": "recovered-",

    "batch.max.rows": 10000,
    "poll.interval.ms": 1000
  }
}
```

### 4. Программный подход (Java)

```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;
import java.io.*;
import java.util.*;
import java.util.concurrent.*;

public class DataRecoveryProducer {

    private final KafkaProducer<String, String> producer;
    private final ExecutorService executor;

    public DataRecoveryProducer(String bootstrapServers) {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

        // Оптимизация для bulk загрузки
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 65536);
        props.put(ProducerConfig.LINGER_MS_CONFIG, 20);
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 67108864);
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");

        // Надёжность
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.RETRIES_CONFIG, 10);
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);

        this.producer = new KafkaProducer<>(props);
        this.executor = Executors.newFixedThreadPool(4);
    }

    public CompletableFuture<RecordMetadata> sendAsync(String topic,
                                                        String key,
                                                        String value) {
        CompletableFuture<RecordMetadata> future = new CompletableFuture<>();

        ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, value);

        producer.send(record, (metadata, exception) -> {
            if (exception != null) {
                future.completeExceptionally(exception);
            } else {
                future.complete(metadata);
            }
        });

        return future;
    }

    public void recoverFromFile(String topic, String filePath) throws Exception {
        List<CompletableFuture<RecordMetadata>> futures = new ArrayList<>();

        try (BufferedReader reader = new BufferedReader(new FileReader(filePath))) {
            String line;
            int count = 0;

            while ((line = reader.readLine()) != null) {
                String[] parts = line.split("\t");
                String key = parts[0];
                String value = parts[1];

                futures.add(sendAsync(topic, key, value));
                count++;

                // Периодическая очистка futures
                if (count % 10000 == 0) {
                    CompletableFuture.allOf(
                        futures.toArray(new CompletableFuture[0])
                    ).get(60, TimeUnit.SECONDS);
                    futures.clear();
                    System.out.println("Recovered " + count + " records");
                }
            }

            // Финальная очистка
            CompletableFuture.allOf(
                futures.toArray(new CompletableFuture[0])
            ).get(60, TimeUnit.SECONDS);

            System.out.println("Total recovered: " + count + " records");
        }
    }

    public void close() {
        producer.flush();
        producer.close();
        executor.shutdown();
    }
}
```

### 5. Python подход

```python
from kafka import KafkaProducer
from kafka.errors import KafkaError
import json
import boto3
import gzip
from concurrent.futures import ThreadPoolExecutor, as_completed

class KafkaDataRecovery:
    def __init__(self, bootstrap_servers: str):
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            key_serializer=lambda k: k.encode('utf-8') if k else None,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            acks='all',
            retries=10,
            batch_size=65536,
            linger_ms=20,
            compression_type='lz4',
            max_in_flight_requests_per_connection=5
        )
        self.s3_client = boto3.client('s3')

    def recover_from_s3(self, bucket: str, prefix: str, topic: str):
        """Восстановление данных из S3"""
        paginator = self.s3_client.get_paginator('list_objects_v2')

        for page in paginator.paginate(Bucket=bucket, Prefix=prefix):
            for obj in page.get('Contents', []):
                self._process_s3_object(bucket, obj['Key'], topic)

    def _process_s3_object(self, bucket: str, key: str, topic: str):
        """Обработка одного S3 объекта"""
        response = self.s3_client.get_object(Bucket=bucket, Key=key)

        # Если файл сжат
        if key.endswith('.gz'):
            content = gzip.decompress(response['Body'].read())
        else:
            content = response['Body'].read()

        # Парсинг и отправка
        for line in content.decode('utf-8').strip().split('\n'):
            record = json.loads(line)
            key = record.get('key')
            value = record.get('value')

            future = self.producer.send(topic, key=key, value=value)
            future.add_callback(self._on_success)
            future.add_errback(self._on_error)

    def _on_success(self, metadata):
        pass  # Успешная отправка

    def _on_error(self, exception):
        print(f"Error sending record: {exception}")

    def recover_with_ordering(self, topic: str, records: list):
        """Восстановление с сохранением порядка"""
        # Группировка по partition key
        partitioned = {}
        for record in records:
            key = record.get('key', '')
            if key not in partitioned:
                partitioned[key] = []
            partitioned[key].append(record)

        # Параллельная отправка по ключам
        with ThreadPoolExecutor(max_workers=10) as executor:
            futures = []
            for key, key_records in partitioned.items():
                future = executor.submit(
                    self._send_ordered, topic, key, key_records
                )
                futures.append(future)

            for future in as_completed(futures):
                try:
                    future.result()
                except Exception as e:
                    print(f"Error: {e}")

    def _send_ordered(self, topic: str, key: str, records: list):
        """Отправка записей с одним ключом последовательно"""
        for record in records:
            future = self.producer.send(
                topic,
                key=key,
                value=record['value']
            )
            # Ждём подтверждения для сохранения порядка
            future.get(timeout=30)

    def close(self):
        self.producer.flush()
        self.producer.close()


# Использование
if __name__ == "__main__":
    recovery = KafkaDataRecovery('localhost:9092')
    recovery.recover_from_s3(
        bucket='kafka-backup',
        prefix='events/2024/01/',
        topic='events-recovered'
    )
    recovery.close()
```

## Примеры конфигурации

### Конфигурация для быстрого восстановления

```properties
# server.properties для приёма большого объёма данных

# Увеличенные буферы
socket.send.buffer.bytes=1048576
socket.receive.buffer.bytes=1048576
socket.request.max.bytes=104857600

# Быстрая запись
log.flush.interval.messages=50000
log.flush.interval.ms=10000

# Репликация
num.replica.fetchers=4
replica.fetch.max.bytes=10485760
replica.fetch.wait.max.ms=100
```

### Kafka Connect для массового восстановления

```properties
# connect-distributed.properties

# Увеличенные ресурсы
tasks.max=20
offset.flush.interval.ms=10000
offset.flush.timeout.ms=30000

# Producer для восстановления
producer.batch.size=131072
producer.linger.ms=50
producer.buffer.memory=134217728
producer.compression.type=lz4
producer.acks=all
```

### Скрипт восстановления из backup

```bash
#!/bin/bash
# recover-from-backup.sh

BACKUP_DIR="/backup/kafka"
KAFKA_BOOTSTRAP="localhost:9092"
RECOVERY_TOPIC="events-recovered"

# Проверка наличия backup
if [ ! -d "$BACKUP_DIR" ]; then
    echo "Backup directory not found: $BACKUP_DIR"
    exit 1
fi

# Создание recovery топика
kafka-topics.sh --create \
    --bootstrap-server $KAFKA_BOOTSTRAP \
    --topic $RECOVERY_TOPIC \
    --partitions 12 \
    --replication-factor 3 \
    --config retention.ms=-1 \
    --if-not-exists

# Восстановление каждого файла
find $BACKUP_DIR -name "*.json.gz" | while read file; do
    echo "Processing: $file"

    zcat "$file" | while read line; do
        echo "$line" | kafka-console-producer.sh \
            --bootstrap-server $KAFKA_BOOTSTRAP \
            --topic $RECOVERY_TOPIC \
            --property "parse.key=true" \
            --property "key.separator=:"
    done

    echo "Completed: $file"
done

echo "Recovery complete!"
```

## Best Practices

### 1. Планирование восстановления

```
Чек-лист восстановления:
□ Определить объём данных
□ Рассчитать время восстановления
□ Подготовить достаточно дискового пространства
□ Настроить throttling если нужно
□ Создать план отката
```

### 2. Сохранение порядка сообщений

```python
# При восстановлении важно сохранять порядок внутри partition

# Плохо: параллельная отправка без учёта ключей
for record in records:
    producer.send_async(topic, record)

# Хорошо: группировка по ключам, последовательная отправка внутри группы
from collections import defaultdict

by_key = defaultdict(list)
for record in records:
    by_key[record['key']].append(record)

for key, key_records in by_key.items():
    for record in key_records:
        producer.send(topic, key=key, value=record['value']).get()
```

### 3. Мониторинг восстановления

```bash
# Скрипт мониторинга прогресса восстановления
#!/bin/bash

TOPIC="events-recovered"
BOOTSTRAP="localhost:9092"

while true; do
    # Текущее количество сообщений
    LATEST=$(kafka-run-class.sh kafka.tools.GetOffsetShell \
        --broker-list $BOOTSTRAP \
        --topic $TOPIC \
        --time -1 | awk -F: '{sum+=$3} END {print sum}')

    # Скорость
    PREV=${PREV:-$LATEST}
    RATE=$((LATEST - PREV))
    PREV=$LATEST

    echo "$(date): Total: $LATEST, Rate: $RATE msg/sec"
    sleep 5
done
```

### 4. Обработка ошибок

```java
public class ResilientRecovery {
    private static final int MAX_RETRIES = 5;
    private static final long RETRY_DELAY_MS = 1000;

    public void sendWithRetry(String topic, String key, String value) {
        int attempts = 0;
        Exception lastException = null;

        while (attempts < MAX_RETRIES) {
            try {
                Future<RecordMetadata> future = producer.send(
                    new ProducerRecord<>(topic, key, value)
                );
                future.get(30, TimeUnit.SECONDS);
                return;  // Успех
            } catch (Exception e) {
                lastException = e;
                attempts++;

                if (attempts < MAX_RETRIES) {
                    try {
                        Thread.sleep(RETRY_DELAY_MS * attempts);
                    } catch (InterruptedException ie) {
                        Thread.currentThread().interrupt();
                        throw new RuntimeException(ie);
                    }
                }
            }
        }

        // Сохраняем неотправленные записи для анализа
        saveToDeadLetterQueue(topic, key, value, lastException);
    }
}
```

### 5. Валидация после восстановления

```bash
#!/bin/bash
# validate-recovery.sh

ORIGINAL_TOPIC="events"
RECOVERED_TOPIC="events-recovered"
BOOTSTRAP="localhost:9092"

# Сравнение количества сообщений
ORIGINAL_COUNT=$(kafka-run-class.sh kafka.tools.GetOffsetShell \
    --broker-list $BOOTSTRAP \
    --topic $ORIGINAL_TOPIC \
    --time -1 | awk -F: '{sum+=$3} END {print sum}')

RECOVERED_COUNT=$(kafka-run-class.sh kafka.tools.GetOffsetShell \
    --broker-list $BOOTSTRAP \
    --topic $RECOVERED_TOPIC \
    --time -1 | awk -F: '{sum+=$3} END {print sum}')

echo "Original: $ORIGINAL_COUNT"
echo "Recovered: $RECOVERED_COUNT"

if [ "$ORIGINAL_COUNT" -eq "$RECOVERED_COUNT" ]; then
    echo "✓ Count matches!"
else
    echo "✗ Count mismatch: difference = $((ORIGINAL_COUNT - RECOVERED_COUNT))"
fi
```

## Связанные темы

- [Срок хранения данных](./01-data-retention.md) — настройка retention для recovered данных
- [Инструменты](./03-tools.md) — CLI утилиты для восстановления
- [Мульти-кластер](./06-multi-cluster.md) — DR между кластерами
- [Хранение в облаке](./07-cloud-container-storage.md) — backup в облако

---

[prev: 03-tools](./03-tools.md) | [next: 05-architectures](./05-architectures.md)
