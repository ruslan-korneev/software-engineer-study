# Режимы отказов баз данных (Failure Modes)

Режимы отказов (Failure Modes) — это различные способы, которыми база данных может перестать корректно функционировать. Понимание этих режимов критически важно для проектирования отказоустойчивых систем и разработки стратегий восстановления.

## Типы отказов

### 1. Отказ оборудования (Hardware Failures)

Аппаратные сбои — одна из наиболее распространённых причин недоступности баз данных.

**Типичные проблемы:**

| Компонент | Тип отказа | Последствия |
|-----------|------------|-------------|
| Жёсткий диск/SSD | Выход из строя, битые секторы | Потеря данных, невозможность чтения/записи |
| Оперативная память | Ошибки ECC, физический износ | Повреждение данных в кэше, крэши |
| Процессор | Перегрев, отказ | Полная остановка сервера |
| Блок питания | Скачки напряжения, выход из строя | Внезапное отключение, потеря незаписанных данных |
| RAID-контроллер | Сбой микропрограммы | Недоступность всего массива дисков |

**Признаки надвигающегося отказа диска:**
```bash
# Проверка SMART-статуса диска
sudo smartctl -a /dev/sda

# Ключевые параметры для мониторинга:
# - Reallocated_Sector_Ct (переназначенные секторы)
# - Current_Pending_Sector (ожидающие переназначения)
# - Offline_Uncorrectable (неисправимые ошибки)
```

### 2. Сетевые сбои (Network Partitions)

Сетевой раздел (network partition) — ситуация, когда узлы распределённой системы не могут обмениваться данными друг с другом.

**Сценарии сетевых сбоев:**

```
Сценарий 1: Полная изоляция
┌─────────┐     X     ┌─────────┐
│  Node A │ ───────── │  Node B │
└─────────┘           └─────────┘
    │                     │
    X                     X
    │                     │
┌─────────┐           ┌─────────┐
│  Node C │           │  Node D │
└─────────┘           └─────────┘

Сценарий 2: Split-brain (разделение мозга)
┌─────────────────┐   X   ┌─────────────────┐
│  Partition A    │ ───── │  Partition B    │
│  Node1, Node2   │       │  Node3, Node4   │
│  (думает что    │       │  (думает что    │
│   он мастер)    │       │   он мастер)    │
└─────────────────┘       └─────────────────┘
```

**Типы сетевых проблем:**
- **Полный разрыв соединения** — узлы полностью недоступны
- **Частичная потеря пакетов** — соединение нестабильное
- **Увеличенная латентность** — задержки превышают таймауты
- **Асимметричный раздел** — A видит B, но B не видит A

### 3. Отказ программного обеспечения (Software Failures)

**Категории программных сбоев:**

```
┌─────────────────────────────────────────────────────────┐
│                  Программные сбои                        │
├─────────────────┬─────────────────┬─────────────────────┤
│    Баги СУБД    │  Ошибки в ОС   │   Проблемы config   │
├─────────────────┼─────────────────┼─────────────────────┤
│ - Deadlocks     │ - OOM Killer   │ - Неверные лимиты   │
│ - Memory leaks  │ - File system  │ - Конфликты версий  │
│ - Race условия  │   corruption   │ - Некорректные      │
│ - Buffer        │ - Kernel panic │   параметры         │
│   overflow      │                │                     │
└─────────────────┴─────────────────┴─────────────────────┘
```

**Пример: OOM Killer в Linux**
```bash
# Проверка, был ли процесс убит OOM Killer
dmesg | grep -i "killed process"

# Результат:
# Out of memory: Killed process 12345 (postgres) total-vm:8388608kB

# Настройка защиты PostgreSQL от OOM Killer
echo -1000 > /proc/$(pidof postgres)/oom_score_adj
```

### 4. Человеческий фактор

Статистика показывает, что до 70% всех инцидентов связаны с человеческими ошибками.

**Типичные ошибки операторов:**

| Категория | Пример | Последствия |
|-----------|--------|-------------|
| Неверные команды | `DROP DATABASE production;` | Полная потеря данных |
| Ошибки конфигурации | Неправильный `max_connections` | Отказ в обслуживании |
| Неудачные миграции | Удаление колонки с FK | Нарушение целостности |
| Проблемы с backup | Не проверенный restore | Невозможность восстановления |
| Ошибки деплоя | Откат на несовместимую версию | Corruption данных |

**Защита от человеческих ошибок:**
```sql
-- Защита от случайного удаления
BEGIN;
DROP TABLE important_data;
-- Увидели ошибку?
ROLLBACK;

-- Или подтвердили правильность?
-- COMMIT;

-- Настройка подтверждения опасных операций в PostgreSQL
-- В postgresql.conf:
-- statement_timeout = '30s'  -- Прерывать длинные запросы
```

---

## CAP теорема

CAP теорема (теорема Брюера) — фундаментальный принцип распределённых систем.

### Три свойства

```
                    Consistency (C)
                    Согласованность
                         /\
                        /  \
                       /    \
                      /  CA  \
                     /________\
                    /\        /\
                   /  \  CP  /  \
                  / AP \    /    \
                 /______\  /______\
        Availability (A)    Partition
        Доступность         Tolerance (P)
                            Устойчивость
                            к разделению
```

### Детальное объяснение каждого свойства

**Consistency (Согласованность):**
- Все узлы видят одни и те же данные в один момент времени
- После успешной записи все последующие чтения вернут новое значение
- Linearizability — самая строгая форма согласованности

**Availability (Доступность):**
- Каждый запрос получает ответ (успех или ошибка)
- Система продолжает работать даже при частичных отказах
- Нет бесконечного ожидания ответа

**Partition Tolerance (Устойчивость к разделению):**
- Система продолжает работать при потере сообщений между узлами
- Обязательное свойство для распределённых систем в реальном мире

### Выбор стратегии

```
┌─────────────────────────────────────────────────────────────────┐
│                    Выбор в условиях partition                    │
├─────────────────────────────────┬───────────────────────────────┤
│           CP-система            │          AP-система           │
├─────────────────────────────────┼───────────────────────────────┤
│ • Отклоняет запросы при         │ • Продолжает отвечать         │
│   отсутствии кворума            │   на все запросы              │
│ • Гарантирует согласованность   │ • Может вернуть устаревшие    │
│ • Часть узлов недоступна        │   данные                      │
├─────────────────────────────────┼───────────────────────────────┤
│ Примеры:                        │ Примеры:                      │
│ - PostgreSQL                    │ - Cassandra                   │
│ - MongoDB (по умолчанию)        │ - DynamoDB                    │
│ - etcd, Consul, ZooKeeper       │ - CouchDB                     │
└─────────────────────────────────┴───────────────────────────────┘
```

### PACELC: расширение CAP

PACELC добавляет компромисс Latency vs Consistency при отсутствии partition:

```
If Partition:
    Choose between Availability and Consistency (A/C)
Else (normal operation):
    Choose between Latency and Consistency (L/C)

Примеры:
- Cassandra: PA/EL (при partition выбирает доступность, иначе - низкую латентность)
- MongoDB:   PC/EC (при partition выбирает согласованность, иначе - согласованность)
- PostgreSQL: PC/EC (всегда согласованность)
```

---

## Стратегии обеспечения надёжности

### 1. Репликация

Репликация — создание и поддержание копий данных на нескольких узлах.

**Синхронная репликация:**
```
┌──────────┐    1. Write     ┌──────────┐
│  Client  │ ───────────────▶│  Primary │
└──────────┘                 └──────────┘
     ▲                            │
     │                   2. Replicate (sync)
     │                            │
     │                            ▼
     │                       ┌──────────┐
     │   4. ACK              │ Replica  │
     └───────────────────────│  (sync)  │
                             └──────────┘
                                  │
                             3. Write to disk
                             + Send ACK to Primary
```

**Конфигурация PostgreSQL для синхронной репликации:**
```ini
# postgresql.conf на Primary
wal_level = replica
max_wal_senders = 10
synchronous_standby_names = 'replica1'
synchronous_commit = on

# Уровни synchronous_commit:
# off            - не ждать записи WAL (быстро, но опасно)
# local          - ждать записи в локальный WAL
# remote_write   - ждать записи в WAL реплики (без fsync)
# on             - ждать fsync на реплике
# remote_apply   - ждать применения на реплике (самый безопасный)
```

**Асинхронная репликация:**
```
┌──────────┐    1. Write     ┌──────────┐
│  Client  │ ───────────────▶│  Primary │
└──────────┘                 └──────────┘
     ▲                            │
     │   2. ACK (сразу)           │
     └────────────────────────────┤
                                  │
                    3. Replicate (async, позже)
                                  │
                                  ▼
                             ┌──────────┐
                             │ Replica  │
                             │ (async)  │
                             └──────────┘
```

**Сравнение типов репликации:**

| Характеристика | Синхронная | Асинхронная |
|----------------|------------|-------------|
| Потеря данных при отказе primary | Нет (RPO=0) | Возможна (RPO>0) |
| Латентность записи | Выше | Ниже |
| Доступность при сбое replica | Снижается | Не влияет |
| Сложность настройки | Выше | Ниже |

### 2. Резервное копирование

**Типы резервных копий:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Стратегия резервирования                  │
├───────────────┬───────────────┬─────────────────────────────┤
│    Полный     │ Инкрементный  │     Дифференциальный        │
│    backup     │    backup     │         backup              │
├───────────────┼───────────────┼─────────────────────────────┤
│ Понедельник:  │ Понедельник:  │ Понедельник:                │
│ [FULL 100GB]  │ [FULL 100GB]  │ [FULL 100GB]                │
│               │               │                             │
│ Вторник:      │ Вторник:      │ Вторник:                    │
│ [FULL 100GB]  │ [INCR 2GB]    │ [DIFF 2GB] (vs понедельник) │
│               │ (vs Пн)       │                             │
│ Среда:        │ Среда:        │ Среда:                      │
│ [FULL 100GB]  │ [INCR 1GB]    │ [DIFF 3GB] (vs понедельник) │
│               │ (vs Вт)       │                             │
│ Четверг:      │ Четверг:      │ Четверг:                    │
│ [FULL 100GB]  │ [INCR 3GB]    │ [DIFF 6GB] (vs понедельник) │
│               │ (vs Ср)       │                             │
├───────────────┼───────────────┼─────────────────────────────┤
│ Восстановление│ Восстановление│ Восстановление:             │
│ из 1 файла    │ из ВСЕХ файлов│ из 2 файлов                 │
│               │ по порядку    │ (full + последний diff)     │
└───────────────┴───────────────┴─────────────────────────────┘
```

**Пример настройки pg_basebackup:**
```bash
# Создание полного backup с PostgreSQL
pg_basebackup \
    -h primary_host \
    -U replication_user \
    -D /backup/base_backup \
    -Fp \            # Формат plain (директория)
    -Xs \            # Включить WAL методом stream
    -P \             # Показывать прогресс
    -R               # Создать standby.signal и recovery.conf

# Автоматизация через cron
# /etc/cron.d/pg_backup
0 2 * * 0 postgres /usr/local/bin/pg_full_backup.sh    # Полный по воскресеньям
0 2 * * 1-6 postgres /usr/local/bin/pg_incr_backup.sh  # Инкрементный в остальные дни
```

**Использование pgBackRest для enterprise backup:**
```ini
# /etc/pgbackrest/pgbackrest.conf
[global]
repo1-path=/backup/pgbackrest
repo1-retention-full=4
repo1-retention-diff=7
process-max=4
compress-type=zst
compress-level=6

[main]
pg1-path=/var/lib/postgresql/15/main
```

```bash
# Полный backup
pgbackrest --stanza=main backup --type=full

# Дифференциальный backup
pgbackrest --stanza=main backup --type=diff

# Инкрементный backup
pgbackrest --stanza=main backup --type=incr

# Проверка целостности backup
pgbackrest --stanza=main verify
```

### 3. Failover механизмы

**Автоматический failover с Patroni:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Архитектура Patroni                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│    ┌─────────┐     ┌─────────┐     ┌─────────┐             │
│    │  etcd   │◄───▶│  etcd   │◄───▶│  etcd   │             │
│    │  node1  │     │  node2  │     │  node3  │             │
│    └─────────┘     └─────────┘     └─────────┘             │
│         ▲               ▲               ▲                   │
│         │               │               │                   │
│         ▼               ▼               ▼                   │
│    ┌─────────┐     ┌─────────┐     ┌─────────┐             │
│    │ Patroni │     │ Patroni │     │ Patroni │             │
│    │   +     │     │   +     │     │   +     │             │
│    │PostgreSQL│    │PostgreSQL│    │PostgreSQL│            │
│    │ PRIMARY │     │ STANDBY │     │ STANDBY │             │
│    └─────────┘     └─────────┘     └─────────┘             │
│         │                                                   │
│         ▼                                                   │
│    ┌─────────┐                                              │
│    │ HAProxy │ ◄─── Направляет трафик на текущий primary   │
│    └─────────┘                                              │
│         ▲                                                   │
│         │                                                   │
│    ┌─────────┐                                              │
│    │ Clients │                                              │
│    └─────────┘                                              │
└─────────────────────────────────────────────────────────────┘
```

**Пример конфигурации Patroni:**
```yaml
# /etc/patroni/patroni.yml
scope: postgres-cluster
name: pg-node-1

restapi:
  listen: 0.0.0.0:8008
  connect_address: pg-node-1:8008

etcd:
  hosts:
    - etcd1:2379
    - etcd2:2379
    - etcd3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576  # 1MB
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"

  initdb:
    - encoding: UTF8
    - data-checksums

postgresql:
  listen: 0.0.0.0:5432
  connect_address: pg-node-1:5432
  data_dir: /var/lib/postgresql/15/main
  authentication:
    replication:
      username: replicator
      password: rep_password
    superuser:
      username: postgres
      password: postgres_password
```

### 4. Write-Ahead Logging (WAL)

WAL — механизм журналирования, обеспечивающий ACID-свойства и восстановление после сбоев.

**Принцип работы WAL:**

```
┌───────────────────────────────────────────────────────────────┐
│                    Процесс записи с WAL                        │
├───────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Клиент отправляет        2. Данные записываются           │
│     транзакцию                  в WAL буфер                    │
│     │                           │                              │
│     ▼                           ▼                              │
│  ┌────────┐               ┌──────────────┐                    │
│  │ Client │ ────────────▶ │  WAL Buffer  │                    │
│  └────────┘               │  (в памяти)  │                    │
│                           └──────────────┘                    │
│                                  │                             │
│                                  │ 3. При COMMIT               │
│                                  │    записывается на диск     │
│                                  ▼                             │
│                           ┌──────────────┐                    │
│                           │   WAL файлы  │                    │
│                           │  (на диске)  │                    │
│                           └──────────────┘                    │
│                                  │                             │
│                                  │ 4. Фоновый checkpoint       │
│                                  │    переносит данные         │
│                                  ▼                             │
│                           ┌──────────────┐                    │
│                           │  Data files  │                    │
│                           │  (таблицы)   │                    │
│                           └──────────────┘                    │
│                                                                │
└───────────────────────────────────────────────────────────────┘
```

**Конфигурация WAL в PostgreSQL:**
```ini
# postgresql.conf

# Уровень WAL (minimal, replica, logical)
wal_level = replica

# Размер WAL файлов
wal_segment_size = 16MB    # По умолчанию, задаётся при initdb
min_wal_size = 1GB
max_wal_size = 4GB

# Буферы WAL
wal_buffers = 64MB         # Рекомендуется 1/32 от shared_buffers

# Синхронизация
fsync = on                 # Никогда не выключайте в production!
synchronous_commit = on    # Уровень гарантии commit
wal_sync_method = fdatasync

# Checkpoint настройки
checkpoint_timeout = 10min
checkpoint_completion_target = 0.9
checkpoint_warning = 30s

# Архивирование WAL (для PITR)
archive_mode = on
archive_command = 'pgbackrest --stanza=main archive-push %p'
archive_timeout = 60       # Архивировать неполный WAL каждые 60 сек
```

**Point-in-Time Recovery (PITR):**
```bash
# Восстановление на определённый момент времени
# recovery.signal + postgresql.conf

# В postgresql.conf:
restore_command = 'pgbackrest --stanza=main archive-get %f %p'
recovery_target_time = '2024-01-15 14:30:00'
recovery_target_action = 'promote'  # или 'pause', 'shutdown'

# Запуск восстановления
pg_ctl start -D /var/lib/postgresql/15/main

# PostgreSQL восстановит данные до указанного момента
```

---

## Восстановление после сбоев

### Автоматическое восстановление PostgreSQL

```
┌─────────────────────────────────────────────────────────────┐
│              Процесс восстановления PostgreSQL               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌──────────────────┐                                       │
│   │   Сбой/Крэш      │                                       │
│   └────────┬─────────┘                                       │
│            │                                                 │
│            ▼                                                 │
│   ┌──────────────────┐                                       │
│   │ Запуск PostgreSQL │                                      │
│   └────────┬─────────┘                                       │
│            │                                                 │
│            ▼                                                 │
│   ┌──────────────────┐    Нет      ┌──────────────────┐     │
│   │ Обнаружен нечис- │ ──────────▶ │ Нормальный старт │     │
│   │ тый shutdown?    │             └──────────────────┘     │
│   └────────┬─────────┘                                       │
│            │ Да                                              │
│            ▼                                                 │
│   ┌──────────────────┐                                       │
│   │  Crash Recovery  │                                       │
│   │  - Чтение WAL    │                                       │
│   │  - Redo операций │                                       │
│   │  - Undo неполных │                                       │
│   └────────┬─────────┘                                       │
│            │                                                 │
│            ▼                                                 │
│   ┌──────────────────┐                                       │
│   │ База готова к    │                                       │
│   │ работе           │                                       │
│   └──────────────────┘                                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Recovery процедуры

**1. Восстановление из backup:**
```bash
# Остановка PostgreSQL
sudo systemctl stop postgresql

# Очистка старых данных (ОСТОРОЖНО!)
rm -rf /var/lib/postgresql/15/main/*

# Восстановление из pgBackRest
pgbackrest --stanza=main restore

# Запуск PostgreSQL
sudo systemctl start postgresql

# Проверка состояния
psql -c "SELECT pg_is_in_recovery();"
```

**2. Восстановление после corruption:**
```bash
# Проверка целостности данных
pg_checksums --check -D /var/lib/postgresql/15/main

# Если обнаружены ошибки:
# 1. Попробовать pg_resetwal (ОПАСНО, потеря данных)
pg_resetwal -D /var/lib/postgresql/15/main

# 2. Или восстановление из backup (предпочтительно)
```

**3. pg_rewind для быстрого восстановления старого primary:**
```bash
# Когда старый primary стал доступен после failover
# и мы хотим его вернуть как replica

pg_rewind \
    --target-pgdata=/var/lib/postgresql/15/main \
    --source-server="host=new_primary port=5432 user=postgres" \
    --progress

# Создать standby.signal
touch /var/lib/postgresql/15/main/standby.signal

# Настроить подключение к новому primary
echo "primary_conninfo = 'host=new_primary port=5432 user=replicator'" \
    >> /var/lib/postgresql/15/main/postgresql.auto.conf

# Запустить как replica
pg_ctl start -D /var/lib/postgresql/15/main
```

---

## Мониторинг и алертинг

### Ключевые метрики для мониторинга

```
┌─────────────────────────────────────────────────────────────┐
│                 Dashboard мониторинга БД                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Connections │  │  Replicat.  │  │    Disk     │         │
│  │   85/100    │  │   Lag: 2s   │  │  Usage: 75% │         │
│  │   ▓▓▓▓▓▓▓░  │  │   ▓▓░░░░░░  │  │  ▓▓▓▓▓▓░░░  │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Memory    │  │  QPS/TPS    │  │ Long Queries│         │
│  │  Used: 12GB │  │  QPS: 5.2k  │  │   Count: 3  │         │
│  │   ▓▓▓▓░░░░  │  │   TPS: 850  │  │   ▓░░░░░░░  │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Prometheus + Grafana для PostgreSQL

**Конфигурация postgres_exporter:**
```yaml
# docker-compose.yml
services:
  postgres-exporter:
    image: prometheuscommunity/postgres-exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://monitor:password@postgres:5432/postgres?sslmode=disable"
    ports:
      - "9187:9187"
```

**Важные метрики Prometheus:**
```promql
# Количество активных соединений
pg_stat_activity_count{state="active"}

# Replication lag в секундах
pg_replication_lag_seconds

# Размер WAL
pg_wal_size_bytes

# Transaction rate
rate(pg_stat_database_xact_commit[5m])

# Deadlocks
rate(pg_stat_database_deadlocks[5m])

# Cache hit ratio (должен быть > 0.99)
pg_stat_database_blks_hit / (pg_stat_database_blks_hit + pg_stat_database_blks_read)
```

**Alerting rules:**
```yaml
# prometheus/alerts.yml
groups:
  - name: postgresql_alerts
    rules:
      - alert: PostgreSQLDown
        expr: pg_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL instance is down"

      - alert: PostgreSQLHighConnections
        expr: pg_stat_activity_count > (pg_settings_max_connections * 0.8)
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL connections > 80% of max"

      - alert: PostgreSQLReplicationLag
        expr: pg_replication_lag_seconds > 30
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Replication lag is {{ $value }} seconds"

      - alert: PostgreSQLDeadlocks
        expr: rate(pg_stat_database_deadlocks[1m]) > 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Deadlocks detected"

      - alert: PostgreSQLDiskSpace
        expr: pg_disk_usage_percent > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk usage > 80%"
```

### Скрипты мониторинга

**Проверка здоровья кластера:**
```sql
-- Проверка состояния репликации
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replication_lag_bytes,
    EXTRACT(EPOCH FROM (now() - backend_start)) AS connection_age_seconds
FROM pg_stat_replication;

-- Долгие запросы
SELECT
    pid,
    now() - pg_stat_activity.query_start AS duration,
    query,
    state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes'
  AND state != 'idle';

-- Блокировки
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity
    ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity
    ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

---

## Примеры конфигурации для PostgreSQL

### Production-ready конфигурация

```ini
# postgresql.conf - Production configuration

#------------------------------------------------------------------------------
# CONNECTIONS
#------------------------------------------------------------------------------
max_connections = 200
superuser_reserved_connections = 3

#------------------------------------------------------------------------------
# MEMORY
#------------------------------------------------------------------------------
shared_buffers = 8GB              # 25% от RAM
effective_cache_size = 24GB       # 75% от RAM
work_mem = 64MB                   # Осторожно: умножается на connections
maintenance_work_mem = 2GB
wal_buffers = 256MB

#------------------------------------------------------------------------------
# WAL & REPLICATION
#------------------------------------------------------------------------------
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = 1GB

# Synchronous replication
synchronous_standby_names = 'FIRST 1 (replica1, replica2)'
synchronous_commit = on

# WAL archiving
archive_mode = on
archive_command = 'pgbackrest --stanza=main archive-push %p'
archive_timeout = 60

#------------------------------------------------------------------------------
# CHECKPOINTS
#------------------------------------------------------------------------------
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9
max_wal_size = 8GB
min_wal_size = 2GB

#------------------------------------------------------------------------------
# RELIABILITY
#------------------------------------------------------------------------------
fsync = on
full_page_writes = on
wal_log_hints = on

#------------------------------------------------------------------------------
# MONITORING
#------------------------------------------------------------------------------
shared_preload_libraries = 'pg_stat_statements,auto_explain'
log_min_duration_statement = 1000   # Логировать запросы > 1 сек
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0

# pg_stat_statements
pg_stat_statements.max = 10000
pg_stat_statements.track = all
```

### pg_hba.conf для репликации

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local connections
local   all             postgres                                peer
local   all             all                                     peer

# IPv4 local connections
host    all             all             127.0.0.1/32            scram-sha-256

# Replication connections
host    replication     replicator      10.0.0.0/24             scram-sha-256

# Application connections
host    all             app_user        10.0.1.0/24             scram-sha-256
```

---

## Best Practices

### 1. Планирование отказоустойчивости

```
┌─────────────────────────────────────────────────────────────┐
│               Чеклист отказоустойчивости                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ ☑ Определить RTO и RPO                                      │
│   RTO (Recovery Time Objective) - максимальное время        │
│       простоя: ______ минут                                 │
│   RPO (Recovery Point Objective) - максимальная потеря      │
│       данных: ______ минут                                  │
│                                                              │
│ ☑ Настроить репликацию                                      │
│   □ Синхронная (для RPO=0)                                  │
│   □ Асинхронная (для низкой латентности)                    │
│                                                              │
│ ☑ Настроить backup                                          │
│   □ Полный backup: каждые ______ дней                       │
│   □ Инкрементный backup: каждые ______ часов                │
│   □ WAL archiving: непрерывно                               │
│                                                              │
│ ☑ Настроить failover                                        │
│   □ Автоматический (Patroni/repmgr)                         │
│   □ Ручной с документацией                                  │
│                                                              │
│ ☑ Настроить мониторинг                                      │
│   □ Alerting на критические метрики                         │
│   □ Dashboard для визуализации                              │
│                                                              │
│ ☑ Тестировать восстановление                                │
│   □ DR drill каждые ______ месяцев                          │
│   □ Документация актуальна                                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 2. Правила резервного копирования

**Правило 3-2-1:**
- **3** копии данных
- **2** разных типа носителей
- **1** копия offsite (в другом дата-центре)

```bash
# Пример реализации 3-2-1

# Копия 1: Primary база данных
# /var/lib/postgresql/15/main (SSD)

# Копия 2: Streaming replica
# standby-server:/var/lib/postgresql/15/main (SSD)

# Копия 3: Backup в облако
pgbackrest --stanza=main backup --type=full --repo=2
# repo2 = s3://bucket-name/pgbackrest (S3, другой регион)
```

### 3. Тестирование восстановления

```bash
#!/bin/bash
# dr_test.sh - Скрипт для тестирования DR

set -e

echo "=== DR Test Started at $(date) ==="

# 1. Создать временный сервер для восстановления
echo "Creating test environment..."
docker run -d --name pg_dr_test \
    -e POSTGRES_PASSWORD=test \
    postgres:15

# 2. Восстановить backup
echo "Restoring backup..."
pgbackrest --stanza=main restore \
    --target-server=pg_dr_test \
    --set=20240115-020000F

# 3. Проверить целостность данных
echo "Verifying data integrity..."
psql -h pg_dr_test -U postgres -c "
    SELECT
        (SELECT count(*) FROM users) as users_count,
        (SELECT count(*) FROM orders) as orders_count,
        (SELECT max(created_at) FROM orders) as last_order;
"

# 4. Очистить
echo "Cleaning up..."
docker rm -f pg_dr_test

echo "=== DR Test Completed Successfully ==="
```

### 4. Документация и runbooks

**Структура runbook для инцидентов:**

```markdown
# Runbook: PostgreSQL Primary Down

## Severity: P1 (Critical)

## Detection
- Alert: PostgreSQLDown
- Symptoms: Applications report connection errors

## Diagnosis
1. Check if instance is running:
   `systemctl status postgresql`
2. Check system logs:
   `journalctl -u postgresql -n 100`
3. Check PostgreSQL logs:
   `tail -100 /var/log/postgresql/postgresql-15-main.log`

## Resolution

### Scenario A: Instance can be restarted
1. `systemctl restart postgresql`
2. Verify: `psql -c "SELECT 1"`

### Scenario B: Failover required
1. Verify replica status:
   `patronictl list`
2. Initiate failover:
   `patronictl failover --master pg-node-1 --candidate pg-node-2`
3. Verify new primary:
   `patronictl list`
4. Update DNS/LB if needed

### Scenario C: Full restore required
1. Stop application traffic
2. Restore from backup:
   `pgbackrest --stanza=main restore`
3. Start PostgreSQL
4. Verify data integrity
5. Resume traffic

## Post-incident
- [ ] Root cause analysis
- [ ] Update documentation
- [ ] Improve monitoring if needed
```

---

## Краткое резюме

### Типы отказов
| Тип | Примеры | Защита |
|-----|---------|--------|
| Hardware | Диски, RAM, питание | RAID, replica, backup |
| Network | Partition, latency | Clustering, health checks |
| Software | Bugs, OOM, corruption | Monitoring, updates, checksums |
| Human | DROP TABLE, misconfig | RBAC, review, automation |

### CAP теорема
- **CP системы**: Согласованность важнее доступности (PostgreSQL, MongoDB)
- **AP системы**: Доступность важнее согласованности (Cassandra, DynamoDB)
- В реальном мире P неизбежна - выбираем между C и A

### Ключевые механизмы надёжности
1. **Репликация** - защита от отказа узла
2. **Backup + PITR** - защита от потери данных
3. **WAL** - гарантия ACID и восстановления
4. **Failover** - автоматическое переключение на replica
5. **Мониторинг** - раннее обнаружение проблем

### Метрики для мониторинга
- Replication lag (< 1-5 сек)
- Connection count (< 80% max)
- Disk usage (< 80%)
- Cache hit ratio (> 99%)
- Long running queries (alert > 5 min)
- Deadlocks (should be 0)

### Формула надёжности
```
Время простоя = MTTR × Частота отказов

Где:
MTTR (Mean Time To Recovery) - среднее время восстановления
Частота отказов = 1 / MTBF (Mean Time Between Failures)

Для 99.9% uptime (8.76 часов простоя в год):
- При 1 инциденте в месяц: MTTR должен быть < 44 минут
- При 1 инциденте в неделю: MTTR должен быть < 10 минут
```
