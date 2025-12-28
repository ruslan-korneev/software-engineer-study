# Знакомство с брокерами

[prev: 06-factory-requirements](../05-consumers/06-factory-requirements.md) | [next: 02-zookeeper-role](./02-zookeeper-role.md)

---

## Описание

**Брокер Kafka** — это сервер, который принимает сообщения от продюсеров, сохраняет их на диск и отдаёт консьюмерам. Каждый брокер — это отдельный экземпляр Kafka, работающий как самостоятельный процесс на сервере или в контейнере.

Брокеры являются основой распределённой архитектуры Kafka. Они обеспечивают:
- Приём и хранение сообщений
- Репликацию данных между узлами
- Обработку запросов от клиентов
- Балансировку нагрузки

## Ключевые концепции

### Идентификация брокера

Каждый брокер в кластере имеет уникальный числовой идентификатор (`broker.id`). Этот ID используется для:
- Идентификации брокера в кластере
- Назначения лидеров партиций
- Отслеживания реплик

```properties
# server.properties
broker.id=1
```

### Роль брокера в кластере

```
┌─────────────────────────────────────────────────────────────┐
│                     Kafka Cluster                            │
│                                                              │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐        │
│  │  Broker 1   │   │  Broker 2   │   │  Broker 3   │        │
│  │             │   │             │   │             │        │
│  │ Topic A P0  │   │ Topic A P1  │   │ Topic A P2  │        │
│  │ (Leader)    │   │ (Leader)    │   │ (Leader)    │        │
│  │             │   │             │   │             │        │
│  │ Topic A P1  │   │ Topic A P2  │   │ Topic A P0  │        │
│  │ (Follower)  │   │ (Follower)  │   │ (Follower)  │        │
│  └─────────────┘   └─────────────┘   └─────────────┘        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Компоненты брокера

1. **Network Layer** — обработка сетевых соединений
2. **Request Handler** — обработка API-запросов
3. **Log Manager** — управление логами (партициями)
4. **Replica Manager** — управление репликами
5. **Group Coordinator** — координация consumer groups
6. **Transaction Coordinator** — управление транзакциями

### Порты и сетевые слушатели

Брокер может принимать соединения на нескольких портах:

```properties
# Listeners — адреса, на которых брокер принимает соединения
listeners=PLAINTEXT://0.0.0.0:9092,SSL://0.0.0.0:9093

# Advertised listeners — адреса, которые брокер сообщает клиентам
advertised.listeners=PLAINTEXT://kafka1.example.com:9092,SSL://kafka1.example.com:9093

# Протокол безопасности для inter-broker коммуникации
inter.broker.listener.name=PLAINTEXT
```

## Примеры конфигурации

### Минимальная конфигурация брокера

```properties
# Уникальный ID брокера
broker.id=1

# Подключение к ZooKeeper (для старых версий)
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181

# Директория для хранения логов
log.dirs=/var/kafka-logs

# Сетевые настройки
listeners=PLAINTEXT://0.0.0.0:9092
advertised.listeners=PLAINTEXT://kafka1.example.com:9092

# Количество потоков
num.network.threads=3
num.io.threads=8

# Размер буферов
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
```

### Продвинутая конфигурация

```properties
# Репликация
default.replication.factor=3
min.insync.replicas=2

# Управление логами
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000

# Сжатие
compression.type=producer
log.cleaner.enable=true

# Производительность
num.replica.fetchers=4
replica.fetch.max.bytes=1048576
```

## Архитектура обработки запросов

```
┌──────────────────────────────────────────────────────────────┐
│                         Kafka Broker                          │
│                                                               │
│  ┌─────────────┐    ┌─────────────────┐    ┌──────────────┐  │
│  │   Network   │───▶│  Request Queue  │───▶│   Request    │  │
│  │   Threads   │    │                 │    │   Handlers   │  │
│  └─────────────┘    └─────────────────┘    └──────────────┘  │
│         │                                         │          │
│         │                                         ▼          │
│         │                                  ┌──────────────┐  │
│         │                                  │  Log/Replica │  │
│         │                                  │   Manager    │  │
│         │                                  └──────────────┘  │
│         │                                         │          │
│         ▼                                         ▼          │
│  ┌─────────────┐    ┌─────────────────┐    ┌──────────────┐  │
│  │  Response   │◀───│  Response Queue │◀───│     Disk     │  │
│  │   Threads   │    │                 │    │   Storage    │  │
│  └─────────────┘    └─────────────────┘    └──────────────┘  │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Типы API-запросов

| Запрос | Описание |
|--------|----------|
| `Produce` | Публикация сообщений |
| `Fetch` | Получение сообщений |
| `Metadata` | Информация о топиках и партициях |
| `ListOffsets` | Получение оффсетов |
| `CreateTopics` | Создание топиков |
| `DeleteTopics` | Удаление топиков |
| `DescribeConfigs` | Получение конфигурации |
| `AlterConfigs` | Изменение конфигурации |

## Best Practices

### 1. Планирование ресурсов

```bash
# Рекомендуемые ресурсы для production
# CPU: 8-16 ядер
# RAM: 32-64 GB (heap: 6-8 GB, остальное — page cache)
# Disk: SSD или NVMe, JBOD для высокой пропускной способности
# Network: 10 Gbps
```

### 2. Настройка JVM

```bash
# Рекомендуемые параметры JVM
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g"
export KAFKA_JVM_PERFORMANCE_OPTS="-XX:+UseG1GC \
  -XX:MaxGCPauseMillis=20 \
  -XX:InitiatingHeapOccupancyPercent=35 \
  -XX:+ExplicitGCInvokesConcurrent"
```

### 3. Мониторинг брокера

```bash
# Ключевые метрики для мониторинга
# - UnderReplicatedPartitions: партиции с недостаточным числом реплик
# - OfflinePartitionsCount: недоступные партиции
# - ActiveControllerCount: должен быть 1 в кластере
# - RequestHandlerAvgIdlePercent: загрузка обработчиков запросов
# - NetworkProcessorAvgIdlePercent: загрузка сетевых потоков
```

### 4. Безопасность

```properties
# Включение SSL
ssl.keystore.location=/var/private/ssl/kafka.server.keystore.jks
ssl.keystore.password=password
ssl.key.password=password
ssl.truststore.location=/var/private/ssl/kafka.server.truststore.jks
ssl.truststore.password=password

# SASL аутентификация
sasl.enabled.mechanisms=SCRAM-SHA-256
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256
```

### 5. Отказоустойчивость

- Используйте минимум 3 брокера в production
- Распределяйте брокеры по разным стойкам/зонам доступности
- Настройте `rack.id` для rack-aware репликации
- Установите `min.insync.replicas` >= 2

```properties
# Rack-aware размещение реплик
broker.rack=rack1
```

## Жизненный цикл брокера

```
┌─────────┐    ┌──────────┐    ┌─────────┐    ┌──────────┐
│  Start  │───▶│ Register │───▶│ Running │───▶│ Shutdown │
│         │    │  (ZK/KRaft)   │         │    │          │
└─────────┘    └──────────┘    └─────────┘    └──────────┘
                    │               │
                    │               ▼
                    │         ┌─────────┐
                    └────────▶│ Failure │
                              │ Recovery│
                              └─────────┘
```

При запуске брокер:
1. Загружает конфигурацию
2. Регистрируется в ZooKeeper/KRaft
3. Восстанавливает логи с диска
4. Начинает принимать запросы
5. Синхронизирует реплики

При остановке брокер:
1. Прекращает принимать новые соединения
2. Завершает текущие запросы
3. Переносит лидерство партиций на другие брокеры
4. Закрывает соединения с ZooKeeper/контроллером

---

[prev: 06-factory-requirements](../05-consumers/06-factory-requirements.md) | [next: 02-zookeeper-role](./02-zookeeper-role.md)