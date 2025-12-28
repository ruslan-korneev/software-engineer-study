# Потоковая репликация PostgreSQL (Streaming Replication)

[prev: 01-logical-replication](./01-logical-replication.md) | [next: 03-ha-clustering](./03-ha-clustering.md)

---

## Введение

Потоковая репликация (Streaming Replication) — это физический метод репликации в PostgreSQL, при котором изменения передаются на уровне WAL-записей (Write-Ahead Log) в режиме реального времени. Это основной метод создания отказоустойчивых кластеров PostgreSQL.

## Архитектура потоковой репликации

### Основные компоненты

```
┌─────────────────────────────────────────────────────────────────┐
│                         PRIMARY                                  │
│  ┌───────────┐    ┌──────────────┐    ┌────────────────────┐    │
│  │  Backend  │───►│ WAL Buffers  │───►│  WAL Files         │    │
│  │ Processes │    └──────────────┘    │  (pg_wal/)         │    │
│  └───────────┘           │            └────────────────────┘    │
│                          ▼                                       │
│                   ┌──────────────┐                               │
│                   │  WAL Sender  │──────────────────┐            │
│                   │  Process     │                  │            │
│                   └──────────────┘                  │            │
└─────────────────────────────────────────────────────┼────────────┘
                                                      │
                              TCP Connection (streaming)
                                                      │
                                                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                         STANDBY                                  │
│  ┌──────────────┐    ┌────────────────────┐    ┌──────────────┐ │
│  │ WAL Receiver │───►│  WAL Files         │───►│  Startup     │ │
│  │ Process      │    │  (pg_wal/)         │    │  Process     │ │
│  └──────────────┘    └────────────────────┘    └──────────────┘ │
│                                                       │          │
│                                                       ▼          │
│                                              ┌──────────────┐    │
│                                              │ Data Files   │    │
│                                              └──────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### Режимы работы standby

1. **Warm Standby** — реплика только принимает WAL, но недоступна для чтения
2. **Hot Standby** — реплика доступна для read-only запросов
3. **Cascading Replication** — standby может быть источником для других standby

## Настройка потоковой репликации

### Шаг 1: Конфигурация Primary сервера

```bash
# postgresql.conf на primary
# Уровень WAL для репликации
wal_level = replica              # Минимум для потоковой репликации

# Параметры WAL
max_wal_senders = 10             # Макс. количество процессов отправки WAL
wal_keep_size = 1GB              # Размер WAL для хранения (PostgreSQL 13+)
# wal_keep_segments = 64         # Для версий до PostgreSQL 13

# Слоты репликации (рекомендуется)
max_replication_slots = 10

# Архивирование (опционально, но рекомендуется)
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/archive/%f'

# Для синхронной репликации (опционально)
synchronous_commit = on
synchronous_standby_names = 'standby1'  # Имя standby
```

### Шаг 2: Настройка аутентификации

```bash
# pg_hba.conf на primary
# Разрешить подключение для репликации
host    replication     replicator    192.168.1.0/24        scram-sha-256
host    replication     replicator    standby_ip/32         scram-sha-256
```

### Шаг 3: Создание пользователя для репликации

```sql
-- На primary
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'secure_password';
```

### Шаг 4: Создание слота репликации (рекомендуется)

```sql
-- На primary
SELECT pg_create_physical_replication_slot('standby1_slot');

-- Проверить созданные слоты
SELECT slot_name, slot_type, active, restart_lsn
FROM pg_replication_slots;
```

### Шаг 5: Создание базовой резервной копии для standby

```bash
# Метод 1: pg_basebackup (рекомендуется)
pg_basebackup -h primary_host -U replicator -D /var/lib/postgresql/16/main \
    -Fp -Xs -P -R -S standby1_slot

# Параметры:
# -Fp: формат plain (обычные файлы)
# -Xs: включить WAL в streaming режиме
# -P: показывать прогресс
# -R: создать standby.signal и настроить recovery
# -S: использовать слот репликации

# Метод 2: pg_basebackup с tar архивом
pg_basebackup -h primary_host -U replicator -D /backup \
    -Ft -z -P -R

# Метод 3: pg_basebackup с параллельным копированием
pg_basebackup -h primary_host -U replicator -D /var/lib/postgresql/16/main \
    -Fp -Xs -P -R --checkpoint=fast -j 4
```

### Шаг 6: Конфигурация Standby сервера

При использовании pg_basebackup с флагом -R, файлы создаются автоматически:

```bash
# Файл standby.signal создается автоматически
# Проверить наличие файла
ls -la /var/lib/postgresql/16/main/standby.signal

# postgresql.auto.conf будет содержать параметры подключения
cat /var/lib/postgresql/16/main/postgresql.auto.conf
```

Если настраиваете вручную:

```bash
# postgresql.conf на standby
hot_standby = on                 # Разрешить read-only запросы
primary_conninfo = 'host=primary_host port=5432 user=replicator password=secure_password application_name=standby1'
primary_slot_name = 'standby1_slot'

# Создать файл standby.signal
touch /var/lib/postgresql/16/main/standby.signal
```

### Шаг 7: Запуск standby сервера

```bash
# Установить владельца файлов
chown -R postgres:postgres /var/lib/postgresql/16/main

# Запустить PostgreSQL
systemctl start postgresql@16-main
# или
pg_ctl -D /var/lib/postgresql/16/main start
```

## Проверка статуса репликации

### На Primary сервере

```sql
-- Проверка процессов отправки WAL
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    sync_state
FROM pg_stat_replication;

-- Проверка задержки репликации
SELECT
    client_addr,
    application_name,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replication_lag_bytes,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) / 1024 / 1024 AS lag_mb
FROM pg_stat_replication;

-- Проверка слотов репликации
SELECT
    slot_name,
    active,
    restart_lsn,
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes
FROM pg_replication_slots;
```

### На Standby сервере

```sql
-- Проверить, что сервер в режиме восстановления
SELECT pg_is_in_recovery();

-- Информация о WAL receiver
SELECT
    pid,
    status,
    receive_start_lsn,
    received_lsn,
    last_msg_send_time,
    last_msg_receipt_time,
    latest_end_lsn
FROM pg_stat_wal_receiver;

-- Позиция последнего воспроизведенного WAL
SELECT pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn();

-- Время последней транзакции
SELECT pg_last_xact_replay_timestamp();

-- Задержка в секундах
SELECT
    EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS lag_seconds;
```

## Синхронная репликация

### Настройка синхронной репликации

```bash
# postgresql.conf на primary
synchronous_commit = on          # Уровень синхронности

# Варианты synchronous_commit:
# off - асинхронный коммит (быстрее, но риск потери данных)
# local - синхронный коммит только на primary
# remote_write - ждать записи на standby
# on (или remote_apply) - ждать применения на standby

# Список синхронных standby серверов
synchronous_standby_names = 'standby1'

# Кворумная синхронная репликация (PostgreSQL 9.6+)
synchronous_standby_names = 'ANY 2 (standby1, standby2, standby3)'

# Приоритетная синхронная репликация
synchronous_standby_names = 'FIRST 2 (standby1, standby2, standby3)'
```

### Управление синхронностью на уровне транзакции

```sql
-- Отключить синхронный коммит для конкретной транзакции
SET synchronous_commit = off;
INSERT INTO logs (message) VALUES ('non-critical log');
COMMIT;

-- Вернуть синхронный режим
SET synchronous_commit = on;
```

## Каскадная репликация

### Архитектура каскадной репликации

```
Primary ──► Standby1 (Cascade) ──► Standby2
                      │
                      └──────────► Standby3
```

### Настройка каскадной репликации

```bash
# На Standby1 (каскадный источник)
# postgresql.conf
hot_standby = on
max_wal_senders = 5              # Для отправки на другие standby

# pg_hba.conf - разрешить подключение для репликации
host    replication     replicator    standby2_ip/32    scram-sha-256
host    replication     replicator    standby3_ip/32    scram-sha-256

# На Standby2 и Standby3
primary_conninfo = 'host=standby1_host port=5432 user=replicator password=pass'
```

## Failover и Switchover

### Ручной Failover (продвижение standby)

```bash
# Метод 1: pg_ctl promote
pg_ctl -D /var/lib/postgresql/16/main promote

# Метод 2: SQL функция (PostgreSQL 12+)
SELECT pg_promote();

# Метод 3: Создать файл promote trigger
touch /var/lib/postgresql/16/main/promote
```

### Переключение бывшего primary в standby

```bash
# После failover, бывший primary нужно переконфигурировать

# 1. Проверить, что новый primary работает
psql -h new_primary -c "SELECT pg_is_in_recovery();"  # Должен вернуть false

# 2. Остановить бывший primary
pg_ctl -D /var/lib/postgresql/16/main stop

# 3. Использовать pg_rewind для синхронизации (если timeline разошлись)
pg_rewind --target-pgdata=/var/lib/postgresql/16/main \
    --source-server='host=new_primary user=postgres'

# 4. Настроить как standby
# Создать standby.signal
touch /var/lib/postgresql/16/main/standby.signal

# Добавить primary_conninfo
cat >> /var/lib/postgresql/16/main/postgresql.auto.conf << EOF
primary_conninfo = 'host=new_primary port=5432 user=replicator password=pass'
primary_slot_name = 'old_primary_slot'
EOF

# 5. Запустить как standby
pg_ctl -D /var/lib/postgresql/16/main start
```

### pg_rewind для быстрого восстановления

```bash
# Требования для pg_rewind:
# - wal_log_hints = on на исходном сервере
# - или включены контрольные суммы при initdb

# Проверить наличие контрольных сумм
pg_controldata /var/lib/postgresql/16/main | grep checksum

# Синхронизация расхождений
pg_rewind \
    --target-pgdata=/var/lib/postgresql/16/main \
    --source-server='host=new_primary port=5432 user=postgres' \
    --progress
```

## Delayed Standby (отложенная репликация)

```bash
# postgresql.conf на standby
recovery_min_apply_delay = '1h'   # Применять WAL с задержкой в 1 час

# Полезно для:
# - Защиты от человеческих ошибок (случайный DELETE)
# - Возможности "отмотать" на момент до ошибки
```

## Мониторинг и алерты

### Скрипт мониторинга задержки

```sql
-- Создать функцию для мониторинга
CREATE OR REPLACE FUNCTION check_replication_lag()
RETURNS TABLE (
    standby_name text,
    lag_bytes bigint,
    lag_mb numeric,
    lag_seconds numeric
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        r.application_name::text,
        pg_wal_lsn_diff(pg_current_wal_lsn(), r.replay_lsn)::bigint,
        ROUND(pg_wal_lsn_diff(pg_current_wal_lsn(), r.replay_lsn) / 1024.0 / 1024.0, 2),
        ROUND(EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())), 2)
    FROM pg_stat_replication r;
END;
$$ LANGUAGE plpgsql;

-- Использование
SELECT * FROM check_replication_lag();
```

### Мониторинг с помощью pg_stat_statements

```sql
-- Настройка алертов по задержке
CREATE TABLE replication_alerts (
    id SERIAL PRIMARY KEY,
    alert_time TIMESTAMP DEFAULT now(),
    standby_name TEXT,
    lag_seconds NUMERIC,
    message TEXT
);

CREATE OR REPLACE FUNCTION log_replication_lag()
RETURNS void AS $$
DECLARE
    lag_threshold NUMERIC := 60;  -- 60 секунд
    current_lag NUMERIC;
    standby TEXT;
BEGIN
    SELECT
        application_name,
        EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))
    INTO standby, current_lag
    FROM pg_stat_replication
    ORDER BY replay_lsn ASC
    LIMIT 1;

    IF current_lag > lag_threshold THEN
        INSERT INTO replication_alerts (standby_name, lag_seconds, message)
        VALUES (standby, current_lag, 'Replication lag exceeded threshold');
    END IF;
END;
$$ LANGUAGE plpgsql;
```

## Best Practices

### 1. Используйте слоты репликации

```sql
-- Слоты гарантируют, что WAL не будет удален до получения standby
SELECT pg_create_physical_replication_slot('standby1_slot');

-- Но установите ограничение на размер WAL
ALTER SYSTEM SET max_slot_wal_keep_size = '10GB';
SELECT pg_reload_conf();
```

### 2. Настройте архивирование WAL

```bash
# postgresql.conf
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
restore_command = 'cp /archive/%f %p'

# Для надежности используйте pgBackRest или Barman
archive_command = 'pgbackrest --stanza=main archive-push %p'
```

### 3. Мониторинг критических метрик

```bash
# Метрики для мониторинга:
# 1. Replication lag (bytes и seconds)
# 2. Количество активных WAL senders
# 3. Размер неотправленных WAL
# 4. Статус слотов репликации
# 5. Timeline ID (для обнаружения split-brain)
```

### 4. Тестирование failover

```bash
# Регулярно тестируйте процедуру failover:
# 1. Документируйте каждый шаг
# 2. Автоматизируйте где возможно
# 3. Проводите учения по DR (Disaster Recovery)
```

## Типичные ошибки и решения

### Ошибка: "requested WAL segment has already been removed"

```bash
# Причина: WAL был удален до получения standby
# Решение 1: Увеличить wal_keep_size
ALTER SYSTEM SET wal_keep_size = '2GB';
SELECT pg_reload_conf();

# Решение 2: Использовать слоты репликации
SELECT pg_create_physical_replication_slot('standby_slot');

# Решение 3: Восстановить standby заново
pg_basebackup -h primary -U replicator -D /data -R -S slot_name
```

### Ошибка: "timeline mismatch"

```bash
# Причина: standby и primary на разных timeline (после failover)
# Решение: использовать pg_rewind или пересоздать standby

pg_rewind --target-pgdata=/data \
    --source-server='host=new_primary user=postgres'
```

### Ошибка: "connection refused" на standby

```bash
# Проверить pg_hba.conf на primary
# Проверить firewall
# Проверить, что max_wal_senders достаточен

# На primary:
SHOW max_wal_senders;
SELECT count(*) FROM pg_stat_replication;
```

## Заключение

Потоковая репликация PostgreSQL — это надежный и проверенный механизм для:
- Создания отказоустойчивых кластеров
- Масштабирования чтения
- Disaster Recovery
- Zero-downtime обновлений

Ключевые моменты:
- Используйте слоты репликации для надежности
- Настройте синхронную репликацию для критичных данных
- Регулярно тестируйте процедуры failover
- Мониторьте задержку репликации

---

[prev: 01-logical-replication](./01-logical-replication.md) | [next: 03-ha-clustering](./03-ha-clustering.md)
