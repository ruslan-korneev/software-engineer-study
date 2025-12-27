# Replication Config (Настройка репликации)

Репликация PostgreSQL позволяет создавать копии данных на нескольких серверах для высокой доступности, балансировки нагрузки и резервного копирования.

## Типы репликации

### Физическая репликация (Streaming Replication)

Копирует WAL-записи на уровне байтов. Реплика идентична мастеру.

### Логическая репликация

Копирует изменения на уровне строк. Позволяет выборочную репликацию.

## Настройка Streaming Replication

### Конфигурация мастера (Primary)

```ini
# postgresql.conf на мастере

# Уровень WAL для репликации
wal_level = replica        # minimal, replica, или logical

# Максимум серверов репликации
max_wal_senders = 10       # количество реплик + резерв

# Размер WAL для отстающих реплик
wal_keep_size = 1GB        # PostgreSQL 13+
# wal_keep_segments = 64   # для старых версий (каждый сегмент = 16MB)

# Слоты репликации (предотвращают удаление WAL)
max_replication_slots = 10

# Архивирование WAL (опционально, но рекомендуется)
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/archive/%f'
```

```ini
# pg_hba.conf на мастере

# Разрешение подключения реплик
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    replication     replicator      192.168.1.0/24          scram-sha-256
host    replication     replicator      10.0.0.0/8              scram-sha-256
```

```sql
-- Создание пользователя для репликации
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'secure_password';
```

### Создание слота репликации

```sql
-- Создание физического слота
SELECT pg_create_physical_replication_slot('replica1_slot');

-- Просмотр слотов
SELECT slot_name, slot_type, active, restart_lsn
FROM pg_replication_slots;

-- Удаление слота
SELECT pg_drop_replication_slot('replica1_slot');
```

### Настройка реплики (Standby)

```bash
# Остановка PostgreSQL на реплике
sudo systemctl stop postgresql

# Создание базовой резервной копии с мастера
pg_basebackup -h master_host -D /var/lib/postgresql/16/main \
    -U replicator -P -Xs -R

# Флаг -R создаёт postgresql.auto.conf и standby.signal
```

```ini
# postgresql.conf на реплике

# Параметры подключения к мастеру (если не использовали -R)
primary_conninfo = 'host=master_host port=5432 user=replicator password=secure_password'

# Использование слота репликации
primary_slot_name = 'replica1_slot'

# Режим горячего резерва (чтение на реплике)
hot_standby = on

# Feedback для мастера
hot_standby_feedback = on
```

```bash
# Создание файла-сигнала (если не использовали -R)
touch /var/lib/postgresql/16/main/standby.signal

# Запуск реплики
sudo systemctl start postgresql
```

## Синхронная репликация

Гарантирует, что транзакция подтверждена на реплике перед коммитом.

### Конфигурация мастера

```ini
# postgresql.conf

# Имена синхронных реплик
synchronous_standby_names = 'FIRST 1 (replica1, replica2)'
# или
synchronous_standby_names = 'ANY 1 (replica1, replica2, replica3)'

# Уровень синхронности
synchronous_commit = on           # ждать синхронную реплику
# synchronous_commit = remote_apply  # ждать применения на реплике
# synchronous_commit = off          # асинхронно (быстрее, но менее надёжно)
```

### Именование реплик

```ini
# postgresql.conf на реплике
cluster_name = 'replica1'  # имя для synchronous_standby_names
```

## Логическая репликация

### Настройка публикатора

```ini
# postgresql.conf на публикаторе
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

```sql
-- Создание публикации
CREATE PUBLICATION my_publication FOR ALL TABLES;

-- Или для конкретных таблиц
CREATE PUBLICATION my_publication FOR TABLE users, orders, products;

-- Просмотр публикаций
SELECT * FROM pg_publication;
SELECT * FROM pg_publication_tables;
```

### Настройка подписчика

```sql
-- Создание подписки
CREATE SUBSCRIPTION my_subscription
CONNECTION 'host=publisher_host dbname=mydb user=replicator password=xxx'
PUBLICATION my_publication;

-- Просмотр подписок
SELECT * FROM pg_subscription;

-- Статус синхронизации
SELECT * FROM pg_stat_subscription;
```

### Управление подпиской

```sql
-- Отключение подписки
ALTER SUBSCRIPTION my_subscription DISABLE;

-- Включение
ALTER SUBSCRIPTION my_subscription ENABLE;

-- Обновление параметров
ALTER SUBSCRIPTION my_subscription
SET PUBLICATION new_publication;

-- Удаление
DROP SUBSCRIPTION my_subscription;
```

## Мониторинг репликации

### Состояние на мастере

```sql
-- Статус WAL senders
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

-- Отставание реплик
SELECT
    client_addr,
    application_name,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS lag_pretty
FROM pg_stat_replication;
```

### Состояние на реплике

```sql
-- Информация о получателе WAL
SELECT * FROM pg_stat_wal_receiver;

-- Проверка режима восстановления
SELECT pg_is_in_recovery();

-- Последняя полученная и применённая позиция WAL
SELECT
    pg_last_wal_receive_lsn() AS receive_lsn,
    pg_last_wal_replay_lsn() AS replay_lsn,
    pg_last_xact_replay_timestamp() AS last_replay_time;

-- Отставание по времени
SELECT
    CASE WHEN pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn()
         THEN 0
         ELSE EXTRACT(EPOCH FROM now() - pg_last_xact_replay_timestamp())
    END AS lag_seconds;
```

## Failover и Switchover

### Промоутинг реплики в мастер

```sql
-- На реплике: промоутинг
SELECT pg_promote();
```

```bash
# Или через утилиту
pg_ctl promote -D /var/lib/postgresql/16/main
```

### Перенастройка старого мастера как реплики

```bash
# После промоутинга реплики, старый мастер нужно перенастроить

# Остановить старый мастер
sudo systemctl stop postgresql

# Использовать pg_rewind для синхронизации
pg_rewind --target-pgdata=/var/lib/postgresql/16/main \
    --source-server="host=new_master port=5432 user=replicator password=xxx"

# Создать standby.signal
touch /var/lib/postgresql/16/main/standby.signal

# Настроить primary_conninfo в postgresql.auto.conf
# Запустить
sudo systemctl start postgresql
```

## Cascading Replication

Реплика может быть источником для других реплик.

```ini
# На промежуточной реплике
# postgresql.conf
max_wal_senders = 5
hot_standby = on
```

```ini
# На каскадной реплике
primary_conninfo = 'host=intermediate_replica port=5432 ...'
```

## Параметры производительности репликации

```ini
# postgresql.conf

# Размер буфера отправки WAL
wal_sender_timeout = 60s

# Задержка применения на реплике (для защиты от ошибок)
recovery_min_apply_delay = 5min  # реплика отстаёт на 5 минут

# Параллельное восстановление
max_parallel_apply_workers_per_subscription = 2  # PostgreSQL 15+
```

## Типичные ошибки

1. **Отсутствие слотов репликации** — WAL может быть удалён до получения репликой
2. **Синхронная репликация без резерва** — падение реплики блокирует мастер
3. **Недостаточный wal_keep_size** — реплика не может догнать мастер
4. **Игнорирование hot_standby_feedback** — конфликты с vacuum
5. **Отсутствие мониторинга лага** — незамеченное отставание реплики

## Best Practices

1. **Всегда используйте слоты репликации** для предотвращения потери WAL
2. **Настраивайте archive_mode** как дополнительную защиту
3. **Мониторьте отставание реплик** — настройте алерты
4. **Используйте FIRST N для синхронной репликации** — отказоустойчивость
5. **Тестируйте failover процедуры** регулярно
6. **Документируйте процедуры переключения** для команды
7. **Настройте recovery_min_apply_delay** на одной реплике для защиты от ошибок

```sql
-- Пример мониторинга для alerting
SELECT
    CASE
        WHEN pg_wal_lsn_diff(sent_lsn, replay_lsn) > 1073741824 THEN 'CRITICAL'
        WHEN pg_wal_lsn_diff(sent_lsn, replay_lsn) > 104857600 THEN 'WARNING'
        ELSE 'OK'
    END as status,
    client_addr,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) as lag
FROM pg_stat_replication;
```
