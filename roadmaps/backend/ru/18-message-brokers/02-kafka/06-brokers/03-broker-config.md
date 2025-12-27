# Конфигурационные параметры брокеров

## Описание

Конфигурация брокера Kafka определяет его поведение, производительность и взаимодействие с другими компонентами кластера. Параметры настраиваются в файле `server.properties` или передаются при запуске. Понимание ключевых параметров критически важно для построения надёжного и производительного кластера.

## Ключевые концепции

### Категории параметров

```
┌─────────────────────────────────────────────────────────────┐
│              Категории конфигурации Kafka                    │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   Broker    │  │   Network   │  │    Log      │          │
│  │  Identity   │  │  Settings   │  │  Settings   │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │ Replication │  │   Thread    │  │  Security   │          │
│  │  Settings   │  │    Pools    │  │  Settings   │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   Quotas    │  │ Transaction │  │  ZK/KRaft   │          │
│  │  & Limits   │  │  Settings   │  │  Settings   │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

### Типы параметров

| Тип | Описание | Изменение |
|-----|----------|-----------|
| **Read-only** | Требуют перезапуска брокера | Статичные |
| **Per-broker** | Можно менять динамически | Применяются к одному брокеру |
| **Cluster-wide** | Динамические, глобальные | Применяются ко всему кластеру |

## Примеры конфигурации

### Базовая идентификация брокера

```properties
############################# Server Basics #############################

# Уникальный ID брокера в кластере (обязательный параметр)
broker.id=1

# Если не указан, будет сгенерирован автоматически (KRaft mode)
# broker.id.generation.enable=true

# Rack ID для rack-aware репликации
broker.rack=rack-1

# Включение удаления топиков
delete.topic.enable=true

# Авто-создание топиков (не рекомендуется для production)
auto.create.topics.enable=false
```

### Сетевые настройки

```properties
############################# Socket Server Settings #############################

# Слушатели — определяют протоколы и порты
listeners=PLAINTEXT://0.0.0.0:9092,SSL://0.0.0.0:9093,SASL_PLAINTEXT://0.0.0.0:9094

# Advertised listeners — адреса для клиентов
advertised.listeners=PLAINTEXT://kafka1.example.com:9092,SSL://kafka1.example.com:9093

# Протокол для межброкерной коммуникации
inter.broker.listener.name=PLAINTEXT

# Маппинг listener -> security protocol
listener.security.protocol.map=PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT

# Отдельный listener для контроллера (KRaft)
controller.listener.names=CONTROLLER

# Размеры буферов сокетов
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400

# Максимальный размер запроса
socket.request.max.bytes=104857600

# Максимальное количество соединений от одного IP
max.connections.per.ip=1000

# Или с overrides для конкретных IP
max.connections.per.ip.overrides=127.0.0.1:200,192.168.1.0/24:100
```

### Настройки потоков

```properties
############################# Thread Settings #############################

# Потоки для обработки сетевых запросов
num.network.threads=8

# Потоки для обработки I/O операций
num.io.threads=16

# Потоки для обработки запросов в фоне
background.threads=10

# Размер очереди запросов
queued.max.requests=500

# Потоки для репликации
num.replica.fetchers=4

# Потоки для очистки логов
num.recovery.threads.per.data.dir=1
log.cleaner.threads=2
```

### Настройки логов (хранения данных)

```properties
############################# Log Settings #############################

# Директории для хранения данных (можно указать несколько через запятую)
log.dirs=/var/kafka-logs,/var/kafka-logs-2,/var/kafka-logs-3

# Количество партиций по умолчанию
num.partitions=3

# Фактор репликации по умолчанию
default.replication.factor=3

# Минимальное количество ISR для подтверждения записи
min.insync.replicas=2

# Размер сегмента лога
log.segment.bytes=1073741824

# Время до ротации сегмента (7 дней)
log.roll.hours=168

# Время хранения сообщений (7 дней)
log.retention.hours=168
# Альтернативно в минутах или миллисекундах:
# log.retention.minutes=10080
# log.retention.ms=604800000

# Максимальный размер лога на партицию
log.retention.bytes=-1

# Интервал проверки для удаления старых сегментов
log.retention.check.interval.ms=300000

# Размер индексного файла
log.index.size.max.bytes=10485760

# Интервал добавления записей в индекс
log.index.interval.bytes=4096

# Время до flush на диск
log.flush.interval.messages=10000
log.flush.interval.ms=1000
```

### Настройки сжатия

```properties
############################# Compression Settings #############################

# Тип сжатия: none, gzip, snappy, lz4, zstd
compression.type=producer

# Настройки log compaction
log.cleaner.enable=true
log.cleanup.policy=delete  # delete, compact, или delete,compact

# Для compact топиков
log.cleaner.min.cleanable.ratio=0.5
log.cleaner.min.compaction.lag.ms=0
log.cleaner.max.compaction.lag.ms=9223372036854775807

# Размер буфера для log cleaner
log.cleaner.dedupe.buffer.size=134217728
log.cleaner.io.buffer.size=524288
log.cleaner.io.buffer.load.factor=0.9
```

### Настройки репликации

```properties
############################# Replication Settings #############################

# Фактор репликации для internal топиков
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2

# Таймауты репликации
replica.lag.time.max.ms=30000
replica.fetch.min.bytes=1
replica.fetch.max.bytes=1048576
replica.fetch.wait.max.ms=500
replica.fetch.backoff.ms=1000

# Таймаут сессии для контроллера
controller.socket.timeout.ms=30000

# Размер буфера для репликации
replica.socket.receive.buffer.bytes=65536
replica.socket.timeout.ms=30000

# Выбор лидера
unclean.leader.election.enable=false
leader.imbalance.check.interval.seconds=300
leader.imbalance.per.broker.percentage=10
```

### Настройки ZooKeeper

```properties
############################# ZooKeeper Settings #############################

# Адрес ZooKeeper (для режима ZooKeeper)
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181/kafka

# Таймаут сессии
zookeeper.session.timeout.ms=18000

# Таймаут подключения
zookeeper.connection.timeout.ms=18000

# Максимум запросов в очереди к ZK
zookeeper.max.in.flight.requests=10

# SSL для ZooKeeper
zookeeper.ssl.client.enable=false
# zookeeper.ssl.keystore.location=/path/to/keystore
# zookeeper.ssl.keystore.password=password
# zookeeper.ssl.truststore.location=/path/to/truststore
# zookeeper.ssl.truststore.password=password
```

### Настройки KRaft

```properties
############################# KRaft Settings #############################

# Идентификатор узла (уникальный в кластере)
node.id=1

# Роль процесса: broker, controller, или оба
process.roles=broker,controller

# Кворум контроллеров
controller.quorum.voters=1@kafka1:9093,2@kafka2:9093,3@kafka3:9093

# Директория для метаданных контроллера
metadata.log.dir=/var/kafka-metadata

# Listener для контроллера
controller.listener.names=CONTROLLER
listeners=PLAINTEXT://:9092,CONTROLLER://:9093

# Максимальное время ожидания fetch от контроллера
controller.quorum.fetch.timeout.ms=2000

# Интервал append для кворума
controller.quorum.append.linger.ms=25

# Таймауты выборов
controller.quorum.election.timeout.ms=1000
controller.quorum.election.backoff.max.ms=1000
```

### Настройки безопасности

```properties
############################# Security Settings #############################

# SSL настройки
ssl.keystore.location=/var/private/ssl/server.keystore.jks
ssl.keystore.password=keystore-password
ssl.key.password=key-password
ssl.truststore.location=/var/private/ssl/server.truststore.jks
ssl.truststore.password=truststore-password

# SSL протоколы и шифры
ssl.enabled.protocols=TLSv1.2,TLSv1.3
ssl.protocol=TLSv1.3
ssl.cipher.suites=TLS_AES_256_GCM_SHA384

# Аутентификация клиентов
ssl.client.auth=required  # required, requested, none

# SASL настройки
sasl.enabled.mechanisms=SCRAM-SHA-256,SCRAM-SHA-512
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256

# Авторизация
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
super.users=User:admin

# Аудит
# audit.log.enable=true
```

### Настройки квот

```properties
############################# Quota Settings #############################

# Квоты по умолчанию (байт/сек)
quota.producer.default=10485760
quota.consumer.default=10485760

# Квота на request rate
quota.window.num=11
quota.window.size.seconds=1

# Клиентские квоты настраиваются динамически через kafka-configs.sh
```

### Настройки транзакций

```properties
############################# Transaction Settings #############################

# Таймаут транзакций
transaction.max.timeout.ms=900000

# Настройки coordinator
transaction.state.log.segment.bytes=104857600
transaction.state.log.load.buffer.size=5242880

# Interval для transaction abort
transaction.abort.timed.out.transaction.cleanup.interval.ms=10000
transaction.remove.expired.transaction.cleanup.interval.ms=3600000
```

## Динамическое изменение конфигурации

### Просмотр текущей конфигурации

```bash
# Показать все настройки брокера
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type brokers \
  --entity-name 1 \
  --describe

# Показать настройки топика
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name my-topic \
  --describe

# Показать настройки пользователя
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type users \
  --entity-name alice \
  --describe
```

### Изменение конфигурации

```bash
# Изменить настройку брокера (динамическую)
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type brokers \
  --entity-name 1 \
  --alter \
  --add-config log.cleaner.threads=4

# Изменить настройку для всех брокеров
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type brokers \
  --entity-default \
  --alter \
  --add-config log.retention.ms=604800000

# Установить квоту для клиента
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type clients \
  --entity-name my-client \
  --alter \
  --add-config producer_byte_rate=1048576,consumer_byte_rate=2097152

# Удалить динамическую настройку
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type brokers \
  --entity-name 1 \
  --alter \
  --delete-config log.cleaner.threads
```

## Best Practices

### 1. Production-ready конфигурация

```properties
# Базовые настройки для production
broker.id=1
log.dirs=/data/kafka-logs

# Сеть
listeners=SASL_SSL://0.0.0.0:9093
advertised.listeners=SASL_SSL://kafka1.example.com:9093
inter.broker.listener.name=SASL_SSL

# Репликация и надёжность
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false

# Производительность
num.network.threads=8
num.io.threads=16
num.replica.fetchers=4

# Хранение
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000

# Безопасность
auto.create.topics.enable=false
delete.topic.enable=true
```

### 2. Настройки для высокой пропускной способности

```properties
# Увеличенные буферы
socket.send.buffer.bytes=1048576
socket.receive.buffer.bytes=1048576
socket.request.max.bytes=104857600

# Больше потоков
num.network.threads=16
num.io.threads=32

# Оптимизация записи
log.flush.interval.messages=50000
log.flush.interval.ms=10000

# Сжатие
compression.type=lz4
```

### 3. Настройки для низкой задержки

```properties
# Минимальные задержки на запись
log.flush.interval.messages=1
log.flush.interval.ms=0

# Быстрая репликация
replica.fetch.min.bytes=1
replica.fetch.wait.max.ms=100

# Меньше потоков для уменьшения context switching
num.network.threads=4
num.io.threads=8
```

### 4. Мониторинг конфигурации

```bash
# Важные метрики для мониторинга:

# JMX метрики конфигурации
kafka.server:type=KafkaConfig,name=*

# Проверка консистентности конфигурации между брокерами
for broker in 1 2 3; do
  echo "=== Broker $broker ==="
  bin/kafka-configs.sh --bootstrap-server localhost:9092 \
    --entity-type brokers --entity-name $broker --describe
done
```

### 5. Документирование изменений

```properties
# Всегда документируйте причины изменений в комментариях
# server.properties

# [2024-01-15] Увеличено для обработки возросшей нагрузки (TICKET-123)
num.io.threads=16

# [2024-02-01] Изменено для совместимости с новым consumer (TICKET-456)
log.message.format.version=3.0
```

## Типичные ошибки конфигурации

### 1. Несоответствие advertised.listeners

```properties
# НЕПРАВИЛЬНО: клиенты не смогут подключиться
listeners=PLAINTEXT://0.0.0.0:9092
advertised.listeners=PLAINTEXT://localhost:9092  # недоступен извне

# ПРАВИЛЬНО
listeners=PLAINTEXT://0.0.0.0:9092
advertised.listeners=PLAINTEXT://kafka1.example.com:9092
```

### 2. Недостаточный min.insync.replicas

```properties
# ОПАСНО: возможна потеря данных
replication.factor=3
min.insync.replicas=1

# РЕКОМЕНДУЕТСЯ
replication.factor=3
min.insync.replicas=2
```

### 3. Включённый unclean.leader.election

```properties
# ОПАСНО: возможна потеря данных
unclean.leader.election.enable=true

# РЕКОМЕНДУЕТСЯ для большинства случаев
unclean.leader.election.enable=false
```

### 4. Неправильные таймауты

```properties
# Слишком агрессивные таймауты
replica.lag.time.max.ms=5000  # слишком мало

# Рекомендуемые значения
replica.lag.time.max.ms=30000
zookeeper.session.timeout.ms=18000
```
