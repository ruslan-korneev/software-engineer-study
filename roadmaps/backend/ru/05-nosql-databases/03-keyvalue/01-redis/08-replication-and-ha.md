# Репликация и высокая доступность в Redis

## Введение

Высокая доступность (High Availability, HA) — критически важный аспект для production-систем. Redis предоставляет несколько механизмов для обеспечения отказоустойчивости:

1. **Репликация** — копирование данных с master на replica
2. **Redis Sentinel** — автоматический failover для standalone Redis
3. **Redis Cluster** — распределённая система с шардированием и репликацией

---

## 1. Репликация в Redis — основы

### Master-Replica архитектура

```
┌─────────────────┐
│     Master      │
│  (read/write)   │
└────────┬────────┘
         │ асинхронная репликация
    ┌────┴────┬────────────┐
    ▼         ▼            ▼
┌───────┐ ┌───────┐   ┌───────┐
│Replica│ │Replica│   │Replica│
│  #1   │ │  #2   │   │  #N   │
│(read) │ │(read) │   │(read) │
└───────┘ └───────┘   └───────┘
```

**Основные принципы:**

- **Master** — принимает все операции записи
- **Replica (ранее slave)** — копия данных master, только для чтения
- Один master может иметь множество replica
- Replica может быть master для других replica (каскадная репликация)

### Асинхронная репликация

Redis использует **асинхронную репликацию** по умолчанию:

```
Клиент ──► Master ──► OK (ответ клиенту)
              │
              └──► Replica (асинхронно)
```

**Важно понимать:**
- Клиент получает подтверждение до репликации данных
- Возможна потеря данных при падении master
- Более высокая производительность по сравнению с синхронной репликацией

### Синхронная репликация (WAIT)

Для критичных данных можно использовать команду `WAIT`:

```bash
# Записываем данные
SET important-key "critical-value"

# Ждём подтверждения от 2 реплик в течение 5000 мс
WAIT 2 5000
# Возвращает количество реплик, подтвердивших запись
```

### Команды REPLICAOF / SLAVEOF

```bash
# Современная команда (Redis 5.0+)
REPLICAOF <master-ip> <master-port>

# Устаревшая команда (работает для совместимости)
SLAVEOF <master-ip> <master-port>

# Прекратить репликацию и стать master
REPLICAOF NO ONE
```

**Примеры:**

```bash
# Подключить replica к master
127.0.0.1:6380> REPLICAOF 192.168.1.10 6379
OK

# Проверить статус
127.0.0.1:6380> INFO replication
# role:slave
# master_host:192.168.1.10
# master_port:6379
# master_link_status:up

# Отключить репликацию
127.0.0.1:6380> REPLICAOF NO ONE
OK
```

---

## 2. Настройка репликации

### Конфигурация Master

```bash
# redis-master.conf

# Основные настройки
bind 0.0.0.0
port 6379

# Аутентификация (рекомендуется для production)
requirepass "master_password"

# Пароль для подключения реплик
masterauth "master_password"

# Минимальное количество реплик для записи
min-replicas-to-write 1

# Максимальная задержка реплик (в секундах)
min-replicas-max-lag 10

# Резервирование дискового пространства для RDB
rdb-save-incremental-fsync yes
```

### Конфигурация Replica

```bash
# redis-replica.conf

bind 0.0.0.0
port 6380

# Указываем master
replicaof 192.168.1.10 6379

# Пароль для подключения к master
masterauth "master_password"

# Собственный пароль (если нужен)
requirepass "replica_password"

# Режим только для чтения (по умолчанию yes)
replica-read-only yes

# Отдавать устаревшие данные при потере связи с master
replica-serve-stale-data yes

# Приоритет для выбора нового master (меньше = приоритетнее)
replica-priority 100
```

### Синхронизация данных

#### Full Sync (полная синхронизация)

Происходит при:
- Первом подключении replica к master
- Невозможности partial sync
- Команда `DEBUG RELOAD` на replica

```
┌────────────────────────────────────────────────────────┐
│                    FULL SYNC                           │
├────────────────────────────────────────────────────────┤
│ 1. Replica → Master: PSYNC ? -1                        │
│ 2. Master: создаёт RDB snapshot в фоне (BGSAVE)        │
│ 3. Master: буферизирует новые команды записи           │
│ 4. Master → Replica: отправляет RDB файл               │
│ 5. Replica: загружает RDB (очищает старые данные)      │
│ 6. Master → Replica: отправляет буферизированные       │
│    команды                                             │
│ 7. Replica: применяет команды, синхронизация завершена │
└────────────────────────────────────────────────────────┘
```

#### Partial Sync (частичная синхронизация)

Используется при:
- Кратковременном разрыве соединения
- Рестарте replica (если backlog сохранился)

```
┌────────────────────────────────────────────────────────┐
│                   PARTIAL SYNC                         │
├────────────────────────────────────────────────────────┤
│ Master хранит replication backlog (кольцевой буфер)    │
│                                                        │
│ 1. Replica → Master: PSYNC <replid> <offset>           │
│ 2. Master проверяет: есть ли данные в backlog?         │
│ 3. Если да: +CONTINUE                                  │
│    Master → Replica: отправляет только пропущенные     │
│    команды                                             │
│ 4. Если нет: +FULLRESYNC                               │
│    Выполняется полная синхронизация                    │
└────────────────────────────────────────────────────────┘
```

**Настройка backlog:**

```bash
# Размер буфера репликации (по умолчанию 1MB)
repl-backlog-size 256mb

# Время хранения backlog после отключения всех реплик
repl-backlog-ttl 3600
```

### Diskless репликация

Для систем с медленными дисками можно использовать diskless репликацию:

```bash
# Отправлять RDB напрямую в сокет, минуя диск
repl-diskless-sync yes

# Задержка перед отправкой (ожидание подключения других реплик)
repl-diskless-sync-delay 5

# Diskless загрузка на стороне реплики
repl-diskless-load on-empty-db
```

---

## 3. Redis Sentinel

### Что это и зачем нужен

**Redis Sentinel** — отдельный процесс для:
- **Мониторинга** — проверка доступности master и replica
- **Уведомлений** — оповещение при сбоях
- **Автоматического failover** — повышение replica до master
- **Предоставления конфигурации** — клиенты получают адрес текущего master

```
┌─────────────────────────────────────────────────────────┐
│                 SENTINEL CLUSTER                        │
│                                                         │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐            │
│  │Sentinel 1│   │Sentinel 2│   │Sentinel 3│            │
│  │  :26379  │   │  :26379  │   │  :26379  │            │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘            │
│       │              │              │                   │
│       └──────────────┼──────────────┘                   │
│                      │ мониторинг                       │
│       ┌──────────────┼──────────────┐                   │
│       ▼              ▼              ▼                   │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐               │
│  │ Master  │◄──│ Replica │   │ Replica │               │
│  │  :6379  │   │  :6380  │   │  :6381  │               │
│  └─────────┘   └─────────┘   └─────────┘               │
└─────────────────────────────────────────────────────────┘
```

### Почему минимум 3 Sentinel?

**Кворум (quorum)** — минимальное количество Sentinel, которые должны согласиться, что master недоступен.

```
Для надёжного failover нужно:
- Минимум 3 Sentinel для отказоустойчивости
- Quorum обычно = N/2 + 1 (большинство)
- При 3 Sentinel: quorum = 2

┌───────────────────────────────────────────────────────┐
│ Количество Sentinel │ Quorum │ Допустимые отказы     │
├─────────────────────┼────────┼───────────────────────┤
│          3          │   2    │         1             │
│          5          │   3    │         2             │
│          7          │   4    │         3             │
└───────────────────────────────────────────────────────┘
```

### Процесс Failover

```
┌──────────────────────────────────────────────────────────────┐
│                    FAILOVER PROCESS                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│ 1. SDOWN (Subjective Down)                                   │
│    └─ Один Sentinel считает master недоступным               │
│       (превышен down-after-milliseconds)                     │
│                                                              │
│ 2. ODOWN (Objective Down)                                    │
│    └─ Quorum Sentinel согласны: master недоступен            │
│                                                              │
│ 3. Leader Election                                           │
│    └─ Sentinel выбирают лидера для проведения failover       │
│       (алгоритм Raft)                                        │
│                                                              │
│ 4. Failover                                                  │
│    └─ Лидер выбирает лучшую replica:                         │
│       - Наименьший replica-priority (не 0)                   │
│       - Наибольший replication offset                        │
│       - Наименьший runid (лексикографически)                 │
│                                                              │
│ 5. Promotion                                                 │
│    └─ Выбранная replica становится master                    │
│       (REPLICAOF NO ONE)                                     │
│                                                              │
│ 6. Reconfiguration                                           │
│    └─ Остальные replica переключаются на нового master       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Конфигурация Sentinel

```bash
# sentinel.conf

# Порт Sentinel
port 26379

# Мониторинг master (имя, ip, порт, quorum)
sentinel monitor mymaster 192.168.1.10 6379 2

# Пароль для подключения к Redis
sentinel auth-pass mymaster master_password

# Время до объявления master недоступным (мс)
sentinel down-after-milliseconds mymaster 30000

# Максимальное количество реплик, синхронизируемых параллельно
# после failover (меньше = меньше нагрузка)
sentinel parallel-syncs mymaster 1

# Таймаут failover (мс)
sentinel failover-timeout mymaster 180000

# Скрипты для уведомлений
sentinel notification-script mymaster /var/redis/notify.sh
sentinel client-reconfig-script mymaster /var/redis/reconfig.sh

# Защита от split-brain
sentinel deny-scripts-reconfig yes
```

### Команды Sentinel

```bash
# Подключение к Sentinel
redis-cli -p 26379

# Получить адрес master
SENTINEL GET-MASTER-ADDR-BY-NAME mymaster
# 1) "192.168.1.10"
# 2) "6379"

# Список всех master
SENTINEL MASTERS

# Информация о конкретном master
SENTINEL MASTER mymaster

# Список реплик
SENTINEL REPLICAS mymaster
# или (устаревшее)
SENTINEL SLAVES mymaster

# Список Sentinel
SENTINEL SENTINELS mymaster

# Проверка кворума
SENTINEL CKQUORUM mymaster

# Принудительный failover
SENTINEL FAILOVER mymaster

# Сбросить состояние master
SENTINEL RESET mymaster

# Мониторинг событий
SENTINEL PENDING-SCRIPTS
```

### Пример клиентского подключения (Python)

```python
from redis.sentinel import Sentinel

# Подключение через Sentinel
sentinel = Sentinel([
    ('sentinel-1.example.com', 26379),
    ('sentinel-2.example.com', 26379),
    ('sentinel-3.example.com', 26379),
], socket_timeout=0.5)

# Получить master для записи
master = sentinel.master_for('mymaster', socket_timeout=0.5)
master.set('key', 'value')

# Получить replica для чтения
replica = sentinel.slave_for('mymaster', socket_timeout=0.5)
value = replica.get('key')
```

---

## 4. Redis Cluster

### Архитектура кластера

**Redis Cluster** — распределённая система с:
- **Автоматическим шардированием** данных
- **Встроенной репликацией** (без Sentinel)
- **Автоматическим failover**
- **Горизонтальным масштабированием**

```
┌─────────────────────────────────────────────────────────────┐
│                     REDIS CLUSTER                           │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │   Shard 1       │  │   Shard 2       │  │   Shard 3    │ │
│  │ Slots: 0-5460   │  │ Slots: 5461-10922│  │ Slots: 10923│ │
│  │                 │  │                 │  │    -16383   │ │
│  │ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌──────────┐│ │
│  │ │   Master    │ │  │ │   Master    │ │  │ │  Master  ││ │
│  │ │   Node 1    │ │  │ │   Node 2    │ │  │ │  Node 3  ││ │
│  │ └──────┬──────┘ │  │ └──────┬──────┘ │  │ └────┬─────┘│ │
│  │        │        │  │        │        │  │      │      │ │
│  │ ┌──────▼──────┐ │  │ ┌──────▼──────┐ │  │ ┌────▼─────┐│ │
│  │ │   Replica   │ │  │ │   Replica   │ │  │ │ Replica  ││ │
│  │ │   Node 4    │ │  │ │   Node 5    │ │  │ │  Node 6  ││ │
│  │ └─────────────┘ │  │ └─────────────┘ │  │ └──────────┘│ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
│                                                             │
│  ◄─────────────────── Gossip Protocol ───────────────────►  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Hash Slots (хэш-слоты)

Redis Cluster использует **16384 hash slots** для распределения данных:

```
Slot = CRC16(key) mod 16384

Пример:
- CRC16("user:1000") = 7186
- 7186 mod 16384 = 7186
- Ключ попадает в slot 7186 → Shard 2 (5461-10922)
```

**Hash Tags** — способ гарантировать, что ключи попадут в один слот:

```bash
# Обычные ключи могут попасть в разные слоты
SET user:1000:name "John"    # slot X
SET user:1000:email "j@x.com" # slot Y

# Hash tags — только часть в {} хэшируется
SET {user:1000}:name "John"    # slot = CRC16("user:1000")
SET {user:1000}:email "j@x.com" # тот же slot

# Это позволяет выполнять multi-key операции
MGET {user:1000}:name {user:1000}:email
```

### Репликация внутри кластера

Каждый master-узел может иметь реплики:

```bash
# Минимальная конфигурация (6 узлов):
# 3 master + 3 replica

# Настройка replica в cluster-mode
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000

# Минимальное количество реплик для записи
cluster-replica-validity-factor 10
cluster-migration-barrier 1
cluster-require-full-coverage yes
```

### Failover в кластере

```
┌───────────────────────────────────────────────────────────┐
│              CLUSTER FAILOVER PROCESS                     │
├───────────────────────────────────────────────────────────┤
│                                                           │
│ 1. Обнаружение сбоя                                       │
│    └─ Replica не получает PING от master                  │
│       (cluster-node-timeout)                              │
│                                                           │
│ 2. Голосование                                            │
│    └─ Replica запрашивает голоса у других master          │
│    └─ Для победы нужно большинство master (N/2 + 1)       │
│                                                           │
│ 3. Epoch увеличение                                       │
│    └─ Новый master получает новый configuration epoch     │
│                                                           │
│ 4. Slot takeover                                          │
│    └─ Replica забирает slots упавшего master              │
│                                                           │
│ 5. Уведомление                                            │
│    └─ Gossip protocol распространяет новую конфигурацию   │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

**Ручной failover:**

```bash
# На replica, которую хотим повысить
redis-cli -c -p 7002

# Мягкий failover (дождаться синхронизации)
CLUSTER FAILOVER

# Принудительный (без ожидания master)
CLUSTER FAILOVER FORCE

# Захватить слоты даже без согласия кластера
CLUSTER FAILOVER TAKEOVER
```

### Создание кластера

#### Подготовка конфигурации

```bash
# redis-7000.conf (повторить для 7001-7005)
port 7000
cluster-enabled yes
cluster-config-file nodes-7000.conf
cluster-node-timeout 5000
appendonly yes
appendfilename "appendonly-7000.aof"
dbfilename "dump-7000.rdb"
dir /var/redis/7000
```

#### Запуск узлов

```bash
# Запуск 6 инстансов
for port in 7000 7001 7002 7003 7004 7005; do
    redis-server /etc/redis/redis-$port.conf &
done
```

#### Создание кластера (redis-cli)

```bash
# Redis 5.0+
redis-cli --cluster create \
    192.168.1.10:7000 \
    192.168.1.10:7001 \
    192.168.1.10:7002 \
    192.168.1.11:7003 \
    192.168.1.11:7004 \
    192.168.1.11:7005 \
    --cluster-replicas 1

# --cluster-replicas 1 означает 1 replica на каждый master
```

### Управление кластером

```bash
# Подключение в cluster-mode
redis-cli -c -p 7000

# Информация о кластере
CLUSTER INFO

# Список узлов
CLUSTER NODES

# Слоты конкретного узла
CLUSTER SLOTS

# Добавление нового master
redis-cli --cluster add-node 192.168.1.12:7006 192.168.1.10:7000

# Добавление replica
redis-cli --cluster add-node 192.168.1.12:7007 192.168.1.10:7000 \
    --cluster-slave --cluster-master-id <master-node-id>

# Перераспределение слотов
redis-cli --cluster reshard 192.168.1.10:7000

# Проверка кластера
redis-cli --cluster check 192.168.1.10:7000

# Удаление узла
redis-cli --cluster del-node 192.168.1.10:7000 <node-id>
```

### Ограничения Redis Cluster

```
┌─────────────────────────────────────────────────────────┐
│                   ОГРАНИЧЕНИЯ                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ • Multi-key операции работают только для ключей        │
│   в одном слоте (используйте hash tags)                 │
│                                                         │
│ • SELECT (выбор базы) недоступен — только database 0    │
│                                                         │
│ • Транзакции (MULTI/EXEC) — только в пределах слота    │
│                                                         │
│ • Lua скрипты — все ключи должны быть в одном слоте    │
│                                                         │
│ • PUB/SUB работает, но PUBLISH рассылает на все узлы   │
│                                                         │
│ • Большие значения (>512MB) не поддерживаются           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 5. Сравнение Sentinel vs Cluster

| Характеристика | Sentinel | Cluster |
|---------------|----------|---------|
| **Назначение** | HA для standalone | Шардирование + HA |
| **Масштабирование** | Вертикальное (1 master) | Горизонтальное (N masters) |
| **Размер данных** | Ограничен RAM одного сервера | Распределён по узлам |
| **Сложность настройки** | Низкая | Средняя-высокая |
| **Multi-key операции** | Полная поддержка | Только в пределах слота |
| **Клиентская библиотека** | Sentinel-aware | Cluster-aware |
| **Минимум узлов** | 3 Sentinel + 1 Master + N Replica | 6 (3 master + 3 replica) |
| **Failover** | Sentinel выбирает leader | Replica голосует с master |
| **Пропускная способность** | 1x (один master) | Nx (N masters) |

### Когда использовать что?

```
┌─────────────────────────────────────────────────────────────┐
│                    ВЫБОР АРХИТЕКТУРЫ                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Sentinel подходит когда:                                    │
│ • Данные помещаются на один сервер                          │
│ • Нужна простая HA без шардирования                         │
│ • Важны multi-key операции (MGET, MSET, SUNION)             │
│ • Используются Lua скрипты с несколькими ключами            │
│                                                             │
│ Cluster подходит когда:                                     │
│ • Данные не помещаются на один сервер                       │
│ • Нужна высокая пропускная способность записи               │
│ • Можно организовать данные по hash tags                    │
│ • Требуется горизонтальное масштабирование                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Мониторинг репликации

### INFO replication

```bash
redis-cli INFO replication

# На Master:
# role:master
# connected_slaves:2
# slave0:ip=192.168.1.11,port=6380,state=online,offset=1289372,lag=0
# slave1:ip=192.168.1.12,port=6381,state=online,offset=1289372,lag=1
# master_failover_state:no-failover
# master_replid:8e7a4bf1d2f6c8e9a0b3c4d5e6f7a8b9c0d1e2f3
# master_replid2:0000000000000000000000000000000000000000
# master_repl_offset:1289372
# second_repl_offset:-1
# repl_backlog_active:1
# repl_backlog_size:1048576
# repl_backlog_first_byte_offset:240797
# repl_backlog_histlen:1048576

# На Replica:
# role:slave
# master_host:192.168.1.10
# master_port:6379
# master_link_status:up
# master_last_io_seconds_ago:0
# master_sync_in_progress:0
# slave_read_repl_offset:1289372
# slave_repl_offset:1289372
# slave_priority:100
# slave_read_only:1
# replica_announced:1
```

### Важные метрики

```bash
# Задержка репликации (lag) — разница в offset
# На master:
redis-cli INFO replication | grep slave

# Проверка задержки на каждой replica
slave0:ip=192.168.1.11,port=6380,state=online,offset=1289372,lag=0
#                                                           ^^^^
# lag=0 — идеально, lag>10 — проблема

# Мониторинг с помощью redis-cli
redis-cli --latency -h replica-host
redis-cli --latency-history -h replica-host
```

### Prometheus метрики

```yaml
# Используя redis_exporter
- redis_connected_slaves
- redis_master_repl_offset
- redis_slave_repl_offset
- redis_replication_lag

# Alert правило
- alert: RedisReplicationLag
  expr: redis_replication_lag > 100
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Redis replication lag is high"
```

---

## 7. Best Practices для Production

### Общие рекомендации

```
┌─────────────────────────────────────────────────────────────┐
│                    BEST PRACTICES                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 1. АУТЕНТИФИКАЦИЯ                                           │
│    • Всегда используйте requirepass                         │
│    • ACL для fine-grained доступа (Redis 6+)                │
│    • Разные пароли для разных приложений                    │
│                                                             │
│ 2. СЕТЬ                                                     │
│    • Размещайте replica в разных availability zones         │
│    • Используйте приватные сети                             │
│    • TLS для шифрования трафика                             │
│                                                             │
│ 3. PERSISTENCE                                              │
│    • Включите AOF на master и replica                       │
│    • appendfsync everysec (баланс надёжности/скорости)      │
│    • Регулярные RDB snapshots для быстрого восстановления   │
│                                                             │
│ 4. ПАМЯТЬ                                                   │
│    • Оставляйте 20-30% RAM для fork/COW при BGSAVE         │
│    • maxmemory с политикой eviction                         │
│    • Мониторинг used_memory_rss                             │
│                                                             │
│ 5. МОНИТОРИНГ                                               │
│    • Отслеживайте replication lag                           │
│    • Алерты на master_link_status:down                      │
│    • Мониторинг connected_slaves                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Sentinel Best Practices

```bash
# 1. Минимум 3 Sentinel на разных серверах
# 2. Sentinel не на тех же серверах, что Redis (если возможно)

# 3. Правильные таймауты
sentinel down-after-milliseconds mymaster 5000   # 5 секунд
sentinel failover-timeout mymaster 60000         # 60 секунд

# 4. Скрипты для уведомлений
sentinel notification-script mymaster /path/to/notify.sh

# 5. Логирование
loglevel notice
logfile /var/log/redis/sentinel.log
```

### Cluster Best Practices

```bash
# 1. Минимум 6 узлов (3 master + 3 replica)
# 2. Чётное распределение слотов

# 3. Правильный таймаут
cluster-node-timeout 5000

# 4. Репликация
cluster-replica-validity-factor 10
cluster-migration-barrier 1

# 5. Требовать полное покрытие слотов
cluster-require-full-coverage yes

# 6. Используйте hash tags для связанных данных
# {user:123}:profile, {user:123}:settings
```

### Пример production-конфигурации

```bash
# redis-master-production.conf

# Сеть
bind 10.0.1.10
port 6379
protected-mode yes
tcp-backlog 511
tcp-keepalive 300

# Безопасность
requirepass "very-strong-password-here"
masterauth "very-strong-password-here"

# Память
maxmemory 8gb
maxmemory-policy allkeys-lru

# Persistence
appendonly yes
appendfsync everysec
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

save 900 1
save 300 10
save 60 10000

# Репликация
repl-backlog-size 512mb
repl-backlog-ttl 3600
min-replicas-to-write 1
min-replicas-max-lag 10

# Логирование
loglevel notice
logfile /var/log/redis/redis-server.log

# Производительность
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
```

---

## Заключение

Redis предоставляет гибкие механизмы для обеспечения высокой доступности:

1. **Репликация** — базовый механизм копирования данных
2. **Sentinel** — автоматический failover для standalone установок
3. **Cluster** — шардирование с встроенной репликацией и failover

Выбор между Sentinel и Cluster зависит от:
- Объёма данных
- Требований к пропускной способности
- Сложности приложения (multi-key операции)
- Бюджета на инфраструктуру

Для большинства приложений начинайте с **Sentinel** и переходите на **Cluster** при необходимости горизонтального масштабирования.
