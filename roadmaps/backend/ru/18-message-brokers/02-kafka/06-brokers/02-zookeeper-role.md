# Роль ZooKeeper

[prev: 01-broker-intro](./01-broker-intro.md) | [next: 03-broker-config](./03-broker-config.md)

---

## Описание

**Apache ZooKeeper** — это распределённый сервис координации, который исторически использовался Kafka для управления метаданными кластера, выбора контроллера и отслеживания состояния брокеров. ZooKeeper обеспечивает надёжное хранение критически важной информации о кластере.

> **Важно**: Начиная с Kafka 2.8, появился режим KRaft (Kafka Raft), который позволяет работать без ZooKeeper. В Kafka 3.5+ KRaft стал production-ready, а в Kafka 4.0 поддержка ZooKeeper будет полностью удалена.

## Ключевые концепции

### Что хранит ZooKeeper

```
/kafka (корневой znode)
├── /brokers
│   ├── /ids                    # Информация о брокерах
│   │   ├── /1                  # Данные брокера 1
│   │   ├── /2                  # Данные брокера 2
│   │   └── /3                  # Данные брокера 3
│   ├── /topics                 # Метаданные топиков
│   │   └── /my-topic
│   │       └── /partitions
│   │           ├── /0          # Partition 0
│   │           └── /1          # Partition 1
│   └── /seqid                  # Sequence ID для брокеров
├── /controller                 # Текущий контроллер
├── /controller_epoch           # Эпоха контроллера
├── /admin
│   └── /delete_topics          # Очередь удаления топиков
├── /isr_change_notification    # Уведомления об изменении ISR
├── /consumers                  # Старые consumer offsets (deprecated)
├── /config
│   ├── /topics                 # Конфигурация топиков
│   ├── /clients                # Квоты клиентов
│   └── /brokers                # Динамическая конфигурация брокеров
└── /cluster
    └── /id                     # Уникальный ID кластера
```

### Основные функции ZooKeeper в Kafka

1. **Регистрация брокеров** — каждый брокер создаёт эфемерную (ephemeral) ноду при старте
2. **Выбор контроллера** — определение, какой брокер будет контроллером
3. **Хранение метаданных топиков** — партиции, реплики, ISR
4. **Отслеживание состояния кластера** — обнаружение отказов брокеров
5. **Хранение конфигурации** — динамические настройки топиков и брокеров

### Эфемерные и постоянные ноды

```
┌─────────────────────────────────────────────────────────────┐
│                     ZooKeeper znodes                         │
│                                                              │
│  Ephemeral (эфемерные)          Persistent (постоянные)     │
│  ─────────────────────          ──────────────────────      │
│  • /brokers/ids/{id}            • /brokers/topics/{name}    │
│  • /controller                  • /config/topics/{name}     │
│                                 • /cluster/id               │
│                                                              │
│  Удаляются при отключении       Сохраняются навсегда        │
│  клиента (брокера)              (до явного удаления)        │
└─────────────────────────────────────────────────────────────┘
```

### Механизм выбора контроллера

```
┌─────────────────────────────────────────────────────────────┐
│                  Controller Election                         │
│                                                              │
│  1. Все брокеры пытаются создать /controller                │
│                                                              │
│     Broker 1 ──────┐                                        │
│     Broker 2 ──────┼─────▶ /controller (ephemeral)          │
│     Broker 3 ──────┘                                        │
│                                                              │
│  2. Только один успешно создаёт ноду (становится controller)│
│                                                              │
│  3. Остальные устанавливают watch на /controller            │
│                                                              │
│  4. При падении контроллера — ноды получают уведомление     │
│     и начинается новые выборы                               │
└─────────────────────────────────────────────────────────────┘
```

## Примеры конфигурации

### Конфигурация ZooKeeper (zoo.cfg)

```properties
# Базовые настройки
tickTime=2000
dataDir=/var/zookeeper/data
dataLogDir=/var/zookeeper/logs
clientPort=2181

# Настройки кластера (для 3 узлов)
initLimit=10
syncLimit=5

# Члены кластера
server.1=zk1.example.com:2888:3888
server.2=zk2.example.com:2888:3888
server.3=zk3.example.com:2888:3888

# Автоматическая очистка снапшотов
autopurge.snapRetainCount=3
autopurge.purgeInterval=1

# Лимиты соединений
maxClientCnxns=60
```

### Конфигурация Kafka для подключения к ZooKeeper

```properties
# Строка подключения к ZooKeeper
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181/kafka

# Таймаут сессии
zookeeper.session.timeout.ms=18000

# Таймаут подключения
zookeeper.connection.timeout.ms=18000

# Максимальное время синхронизации
zookeeper.max.in.flight.requests=10
```

### Пример кластера с chroot

```properties
# Использование chroot для изоляции
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181/kafka-production

# Разные кластеры Kafka на одном ZooKeeper
# Кластер 1: zk1:2181,zk2:2181,zk3:2181/kafka-prod
# Кластер 2: zk1:2181,zk2:2181,zk3:2181/kafka-staging
# Кластер 3: zk1:2181,zk2:2181,zk3:2181/kafka-dev
```

## Работа с ZooKeeper CLI

### Подключение и просмотр данных

```bash
# Запуск ZooKeeper shell
bin/zookeeper-shell.sh localhost:2181

# Просмотр корневых нод
ls /

# Просмотр брокеров
ls /brokers/ids

# Информация о брокере
get /brokers/ids/1

# Пример вывода:
# {"listener_security_protocol_map":{"PLAINTEXT":"PLAINTEXT"},
#  "endpoints":["PLAINTEXT://kafka1:9092"],
#  "rack":null,
#  "jmx_port":-1,
#  "host":"kafka1",
#  "timestamp":"1699999999999",
#  "port":9092,
#  "version":4}

# Просмотр текущего контроллера
get /controller

# Просмотр топиков
ls /brokers/topics

# Информация о партициях топика
get /brokers/topics/my-topic/partitions/0/state
```

### Мониторинг состояния

```bash
# Проверка статуса ZooKeeper (4-letter commands)
echo "stat" | nc localhost 2181

# Проверка здоровья
echo "ruok" | nc localhost 2181  # Ответ: imok

# Конфигурация
echo "conf" | nc localhost 2181

# Соединения
echo "cons" | nc localhost 2181

# Дамп информации о сессиях
echo "dump" | nc localhost 2181
```

## Архитектура ZooKeeper

```
┌──────────────────────────────────────────────────────────────┐
│                    ZooKeeper Ensemble                         │
│                                                               │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐         │
│  │  ZK Node 1  │   │  ZK Node 2  │   │  ZK Node 3  │         │
│  │   (Leader)  │◀──│  (Follower) │──▶│  (Follower) │         │
│  │             │   │             │   │             │         │
│  │  ┌───────┐  │   │  ┌───────┐  │   │  ┌───────┐  │         │
│  │  │ Data  │  │   │  │ Data  │  │   │  │ Data  │  │         │
│  │  │ Store │  │   │  │ Store │  │   │  │ Store │  │         │
│  │  └───────┘  │   │  └───────┘  │   │  └───────┘  │         │
│  └─────────────┘   └─────────────┘   └─────────────┘         │
│         ▲                 ▲                 ▲                │
│         │                 │                 │                │
│         └─────────────────┼─────────────────┘                │
│                           │                                  │
│                    ┌──────┴──────┐                           │
│                    │    Kafka    │                           │
│                    │   Brokers   │                           │
│                    └─────────────┘                           │
└──────────────────────────────────────────────────────────────┘
```

### Механизм Zab (ZooKeeper Atomic Broadcast)

1. **Leader Election** — выбор лидера среди ZK нод
2. **Atomic Broadcast** — все записи идут через лидера
3. **Replication** — лидер реплицирует на фолловеров
4. **Quorum** — операция успешна при подтверждении большинством

## Best Practices

### 1. Размер кластера ZooKeeper

```
Рекомендуемые размеры:
- Development: 1 нода (не для production!)
- Small production: 3 ноды
- Large production: 5 нод
- Enterprise: 7 нод (редко необходимо)

Формула кворума: (N/2) + 1
- 3 ноды: кворум = 2, допускается отказ 1 ноды
- 5 нод: кворум = 3, допускается отказ 2 нод
- 7 нод: кворум = 4, допускается отказ 3 нод
```

### 2. Выделенные ресурсы

```properties
# Рекомендации по ресурсам ZooKeeper
# CPU: 2-4 ядра
# RAM: 4-8 GB (heap: 1-2 GB)
# Disk: SSD для transaction logs, отдельный диск для snapshots

# JVM настройки
JVMFLAGS="-Xms2g -Xmx2g -XX:+UseG1GC"
```

### 3. Изоляция от Kafka

```
НЕ РЕКОМЕНДУЕТСЯ:
┌─────────────────────────────────┐
│          Один сервер            │
│  ┌──────────┐  ┌─────────────┐  │
│  │ ZooKeeper│  │    Kafka    │  │
│  └──────────┘  └─────────────┘  │
└─────────────────────────────────┘

РЕКОМЕНДУЕТСЯ:
┌─────────────────┐  ┌─────────────────┐
│   ZK Server 1   │  │  Kafka Server 1 │
│   ┌──────────┐  │  │ ┌─────────────┐ │
│   │ ZooKeeper│  │  │ │    Kafka    │ │
│   └──────────┘  │  │ └─────────────┘ │
└─────────────────┘  └─────────────────┘
```

### 4. Мониторинг

```bash
# Ключевые метрики ZooKeeper для мониторинга:

# Latency
# - avg_latency: средняя задержка обработки запросов
# - max_latency: максимальная задержка

# Throughput
# - packets_received: входящие пакеты
# - packets_sent: исходящие пакеты

# Connections
# - num_alive_connections: активные соединения
# - outstanding_requests: запросы в очереди

# Data
# - znode_count: количество znodes
# - watch_count: количество watches
# - ephemerals_count: эфемерные ноды
```

### 5. Безопасность

```properties
# Включение SASL аутентификации
authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
requireClientAuthScheme=sasl

# ACL для Kafka znodes
# В jaas.conf:
Client {
    org.apache.zookeeper.server.auth.DigestLoginModule required
    username="kafka"
    password="kafka-secret";
};
```

## Проблемы и решения

### Session Timeout

```properties
# Симптомы: частые disconnects брокеров
# Решение: увеличить таймаут сессии

# На Kafka:
zookeeper.session.timeout.ms=30000

# На ZooKeeper:
minSessionTimeout=10000
maxSessionTimeout=60000
```

### Переполнение Transaction Log

```bash
# Симптомы: ZooKeeper замедляется или падает
# Решение: настроить autopurge

autopurge.snapRetainCount=3
autopurge.purgeInterval=1

# Или вручную:
bin/zkCleanup.sh -n 3
```

### Split-brain

```
Проблема: Сетевой partition разделяет ZK кластер

Решение:
1. Используйте нечётное число нод (3, 5, 7)
2. Размещайте ноды в разных failure domains
3. Настройте правильные таймауты
4. Используйте стабильную сеть между нодами
```

## Миграция на KRaft

```
┌─────────────────────────────────────────────────────────────┐
│                   Путь миграции                              │
│                                                              │
│  Kafka 2.8+     │  Kafka 3.3+     │  Kafka 3.5+    │ Kafka 4.0 │
│  ──────────────│────────────────│───────────────│─────────── │
│  KRaft preview │  KRaft GA       │  Production   │ ZK удалён │
│  (не для prod) │  (тестирование) │  ready        │            │
│                                                              │
│  Рекомендация: начинайте планировать миграцию на KRaft     │
└─────────────────────────────────────────────────────────────┘
```

Подробнее о KRaft — в файле `05-kafka-internals.md`.

---

[prev: 01-broker-intro](./01-broker-intro.md) | [next: 03-broker-config](./03-broker-config.md)