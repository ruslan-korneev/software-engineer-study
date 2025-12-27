# Журналы Kafka

## Описание

Журналы (логи) в Apache Kafka делятся на два типа: логи данных (commit logs) и логи приложения (application logs). Логи данных — это файлы, в которых хранятся сообщения топиков. Логи приложения — это журналы работы брокера, записываемые через Log4j.

Правильное управление журналами критически важно для мониторинга состояния кластера, диагностики проблем и обеспечения соответствия требованиям хранения данных.

## Ключевые концепции

### Типы журналов

| Тип журнала | Назначение | Расположение |
|-------------|------------|--------------|
| Commit logs | Хранение сообщений топиков | `log.dirs` (data directory) |
| Server logs | Журналы работы брокера | `logs/` или `/var/log/kafka/` |
| Controller logs | Журналы контроллера | `logs/controller.log` |
| State change logs | Изменения состояния | `logs/state-change.log` |
| Log cleaner logs | Журналы очистки логов | `logs/log-cleaner.log` |
| GC logs | Журналы сборки мусора JVM | `logs/kafkaServer-gc.log` |

### Структура директории данных

```
/var/lib/kafka/
├── meta.properties                    # Метаданные брокера
├── __consumer_offsets-0/             # Системный топик оффсетов
│   ├── 00000000000000000000.log      # Сегмент лога
│   ├── 00000000000000000000.index    # Индекс оффсетов
│   └── 00000000000000000000.timeindex # Временной индекс
├── my-topic-0/                       # Партиция 0 топика my-topic
│   ├── 00000000000000000000.log
│   ├── 00000000000000000000.index
│   ├── 00000000000000000000.timeindex
│   ├── 00000000000012345678.log      # Следующий сегмент
│   ├── leader-epoch-checkpoint
│   └── partition.metadata
└── my-topic-1/                       # Партиция 1 топика my-topic
```

### Ключевые параметры логирования

| Параметр | Описание | Значение по умолчанию |
|----------|----------|----------------------|
| `log.dirs` | Директории для хранения данных | `/tmp/kafka-logs` |
| `log.segment.bytes` | Размер сегмента | 1 GB |
| `log.segment.ms` | Время жизни сегмента | 7 дней |
| `log.retention.hours` | Время хранения логов | 168 (7 дней) |
| `log.retention.bytes` | Максимальный размер | -1 (без ограничения) |
| `log.cleanup.policy` | Политика очистки | delete |

## Примеры

### Настройка Log4j

```properties
# /opt/kafka/config/log4j.properties

# Корневой логгер
log4j.rootLogger=INFO, stdout, kafkaAppender

# Консольный вывод
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=[%d] %p %m (%c)%n

# Основной файловый логгер
log4j.appender.kafkaAppender=org.apache.log4j.RollingFileAppender
log4j.appender.kafkaAppender.File=/var/log/kafka/server.log
log4j.appender.kafkaAppender.MaxFileSize=100MB
log4j.appender.kafkaAppender.MaxBackupIndex=10
log4j.appender.kafkaAppender.layout=org.apache.log4j.PatternLayout
log4j.appender.kafkaAppender.layout.ConversionPattern=[%d] %p %m (%c)%n

# State change логгер
log4j.appender.stateChangeAppender=org.apache.log4j.RollingFileAppender
log4j.appender.stateChangeAppender.File=/var/log/kafka/state-change.log
log4j.appender.stateChangeAppender.MaxFileSize=50MB
log4j.appender.stateChangeAppender.MaxBackupIndex=5
log4j.appender.stateChangeAppender.layout=org.apache.log4j.PatternLayout
log4j.appender.stateChangeAppender.layout.ConversionPattern=[%d] %p %m (%c)%n

# Request логгер
log4j.appender.requestAppender=org.apache.log4j.RollingFileAppender
log4j.appender.requestAppender.File=/var/log/kafka/kafka-request.log
log4j.appender.requestAppender.MaxFileSize=100MB
log4j.appender.requestAppender.MaxBackupIndex=5
log4j.appender.requestAppender.layout=org.apache.log4j.PatternLayout
log4j.appender.requestAppender.layout.ConversionPattern=[%d] %p %m (%c)%n

# Controller логгер
log4j.appender.controllerAppender=org.apache.log4j.RollingFileAppender
log4j.appender.controllerAppender.File=/var/log/kafka/controller.log
log4j.appender.controllerAppender.MaxFileSize=50MB
log4j.appender.controllerAppender.MaxBackupIndex=5
log4j.appender.controllerAppender.layout=org.apache.log4j.PatternLayout
log4j.appender.controllerAppender.layout.ConversionPattern=[%d] %p %m (%c)%n

# Log cleaner логгер
log4j.appender.cleanerAppender=org.apache.log4j.RollingFileAppender
log4j.appender.cleanerAppender.File=/var/log/kafka/log-cleaner.log
log4j.appender.cleanerAppender.MaxFileSize=50MB
log4j.appender.cleanerAppender.MaxBackupIndex=5
log4j.appender.cleanerAppender.layout=org.apache.log4j.PatternLayout
log4j.appender.cleanerAppender.layout.ConversionPattern=[%d] %p %m (%c)%n

# Authorizer логгер
log4j.appender.authorizerAppender=org.apache.log4j.RollingFileAppender
log4j.appender.authorizerAppender.File=/var/log/kafka/kafka-authorizer.log
log4j.appender.authorizerAppender.MaxFileSize=50MB
log4j.appender.authorizerAppender.MaxBackupIndex=5
log4j.appender.authorizerAppender.layout=org.apache.log4j.PatternLayout
log4j.appender.authorizerAppender.layout.ConversionPattern=[%d] %p %m (%c)%n

# Настройка уровней логирования
log4j.logger.kafka=INFO
log4j.logger.kafka.network.RequestChannel$=WARN
log4j.logger.kafka.producer.async.DefaultEventHandler=DEBUG
log4j.logger.kafka.request.logger=WARN, requestAppender
log4j.additivity.kafka.request.logger=false
log4j.logger.kafka.controller=TRACE, controllerAppender
log4j.additivity.kafka.controller=false
log4j.logger.kafka.log.LogCleaner=INFO, cleanerAppender
log4j.additivity.kafka.log.LogCleaner=false
log4j.logger.state.change.logger=INFO, stateChangeAppender
log4j.additivity.state.change.logger=false
log4j.logger.kafka.authorizer.logger=INFO, authorizerAppender
log4j.additivity.kafka.authorizer.logger=false

# Снижение шума от ZooKeeper
log4j.logger.org.apache.zookeeper=WARN
log4j.logger.org.apache.kafka.clients=WARN
```

### Настройка GC логирования

```bash
# Для Java 11+
KAFKA_GC_LOG_OPTS="-Xlog:gc*:file=/var/log/kafka/gc.log:time,tags:filecount=10,filesize=100M"

# Для Java 8
KAFKA_GC_LOG_OPTS="-Xloggc:/var/log/kafka/gc.log -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M"
```

### Конфигурация брокера для логов

```properties
# server.properties

# Директории для хранения данных
log.dirs=/var/lib/kafka/data1,/var/lib/kafka/data2

# Размер сегмента лога (1GB)
log.segment.bytes=1073741824

# Время до создания нового сегмента (7 дней)
log.segment.ms=604800000

# Время хранения логов
log.retention.hours=168
# или в минутах/миллисекундах
# log.retention.minutes=10080
# log.retention.ms=604800000

# Максимальный размер лога на партицию (-1 = без ограничения)
log.retention.bytes=-1

# Политика очистки (delete или compact)
log.cleanup.policy=delete

# Интервал проверки логов для очистки
log.retention.check.interval.ms=300000

# Минимальный размер для compaction
log.cleaner.min.cleanable.ratio=0.5

# Количество потоков для очистки
log.cleaner.threads=2

# Буфер для очистки
log.cleaner.dedupe.buffer.size=134217728

# Удаление файлов после удаления сегмента
log.segment.delete.delay.ms=60000

# Flush настройки
log.flush.interval.messages=10000
log.flush.interval.ms=1000
```

### Просмотр логов данных

```bash
# Просмотр содержимого лог-сегмента
kafka-dump-log.sh --files /var/lib/kafka/my-topic-0/00000000000000000000.log \
    --print-data-log

# Проверка индекса
kafka-dump-log.sh --files /var/lib/kafka/my-topic-0/00000000000000000000.index

# Проверка временного индекса
kafka-dump-log.sh --files /var/lib/kafka/my-topic-0/00000000000000000000.timeindex

# Просмотр с декодированием ключей и значений
kafka-dump-log.sh --files /var/lib/kafka/my-topic-0/00000000000000000000.log \
    --print-data-log \
    --deep-iteration

# Проверка целостности лога
kafka-log-dirs.sh --bootstrap-server localhost:9092 \
    --describe \
    --topic-list my-topic
```

### Анализ логов приложения

```bash
# Просмотр последних записей
tail -f /var/log/kafka/server.log

# Поиск ошибок
grep -i "error\|exception\|warn" /var/log/kafka/server.log

# Анализ изменений лидера
grep "Leader" /var/log/kafka/state-change.log

# Поиск проблем с репликацией
grep -i "UnderReplicated\|ISR" /var/log/kafka/server.log

# Анализ контроллера
grep -i "controller\|election" /var/log/kafka/controller.log

# Поиск проблем с ZooKeeper
grep -i "zookeeper\|session\|expired" /var/log/kafka/server.log

# Статистика по уровням логирования
awk '{print $2}' /var/log/kafka/server.log | sort | uniq -c | sort -rn
```

### Logrotate для Kafka

```bash
# /etc/logrotate.d/kafka

/var/log/kafka/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 kafka kafka
    sharedscripts
    postrotate
        # Kafka сам управляет ротацией через Log4j
        # но можно добавить уведомление
        /bin/true
    endscript
}

/var/log/kafka/gc.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 kafka kafka
    copytruncate
}
```

### Скрипт анализа логов

```bash
#!/bin/bash
# analyze-kafka-logs.sh

LOG_DIR=${1:-/var/log/kafka}
LOG_FILE="$LOG_DIR/server.log"

echo "=== Kafka Log Analysis ==="
echo "Log file: $LOG_FILE"
echo

# Количество записей по уровням
echo "=== Log Level Distribution ==="
awk '{print $2}' "$LOG_FILE" 2>/dev/null | sort | uniq -c | sort -rn

echo
echo "=== Recent Errors (last 50) ==="
grep -i "ERROR" "$LOG_FILE" 2>/dev/null | tail -50

echo
echo "=== Recent Warnings (last 20) ==="
grep -i "WARN" "$LOG_FILE" 2>/dev/null | tail -20

echo
echo "=== Controller Events ==="
grep -i "controller" "$LOG_FILE" 2>/dev/null | tail -20

echo
echo "=== Replication Issues ==="
grep -i "under.replicated\|isr\|replica" "$LOG_FILE" 2>/dev/null | tail -20

echo
echo "=== Connection Issues ==="
grep -i "connection\|disconnect\|timeout" "$LOG_FILE" 2>/dev/null | tail -20

echo
echo "=== GC Pauses (if gc.log exists) ==="
if [ -f "$LOG_DIR/gc.log" ]; then
    grep -i "pause" "$LOG_DIR/gc.log" 2>/dev/null | tail -10
else
    echo "GC log not found"
fi
```

### Централизованное логирование

```yaml
# docker-compose.yml с ELK stack

version: '3'
services:
  kafka:
    image: confluentinc/cp-kafka:latest
    volumes:
      - ./logs:/var/log/kafka
    logging:
      driver: "fluentd"
      options:
        fluentd-address: "localhost:24224"
        tag: "kafka.{{.Name}}"

  fluentd:
    image: fluent/fluentd:v1.16
    ports:
      - "24224:24224"
    volumes:
      - ./fluentd.conf:/fluentd/etc/fluent.conf
```

```xml
<!-- log4j.xml для отправки в Logstash -->
<appender name="LOGSTASH" class="net.logstash.log4j.LogStashSocketAppender">
    <param name="RemoteHost" value="logstash.example.com"/>
    <param name="Port" value="4560"/>
    <param name="Application" value="kafka-broker"/>
</appender>
```

## Best Practices

### Рекомендации по хранению логов данных

1. **Используйте несколько директорий** на разных дисках для распределения нагрузки
2. **Настройте правильный retention** в соответствии с бизнес-требованиями
3. **Мониторьте использование дискового пространства** и настройте алерты
4. **Используйте SSD** для лучшей производительности I/O

### Рекомендации по логам приложения

1. **Настройте уровень INFO** для production
2. **Включите DEBUG** только для диагностики проблем
3. **Используйте ротацию логов** для предотвращения заполнения диска
4. **Централизуйте логи** в ELK/Loki/Splunk для анализа

### Мониторинг логов

```bash
# Проверка размера директории логов
du -sh /var/lib/kafka/

# Проверка количества сегментов
find /var/lib/kafka/ -name "*.log" | wc -l

# Алерт при большом количестве ошибок
error_count=$(grep -c "ERROR" /var/log/kafka/server.log)
if [ "$error_count" -gt 100 ]; then
    echo "ALERT: High error count in Kafka logs: $error_count"
fi
```

### Политики хранения

```properties
# Для горячих данных (high throughput)
log.retention.hours=24
log.segment.bytes=536870912

# Для архивных данных (long retention)
log.retention.hours=8760  # 1 год
log.segment.bytes=1073741824

# Для compacted топиков
log.cleanup.policy=compact
log.cleaner.min.compaction.lag.ms=3600000
```

### Безопасность логов

```bash
# Установка правильных прав доступа
chmod 750 /var/log/kafka
chmod 640 /var/log/kafka/*.log
chown -R kafka:kafka /var/log/kafka

# Исключение чувствительных данных
# В log4j.properties добавить фильтры для паролей и токенов
```
