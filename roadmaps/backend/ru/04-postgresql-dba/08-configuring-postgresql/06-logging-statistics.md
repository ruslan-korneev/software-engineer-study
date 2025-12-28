# Logging & Statistics (Логирование и статистика)

[prev: 05-checkpoints-bgwriter](./05-checkpoints-bgwriter.md) | [next: 07-extensions](./07-extensions.md)

---

Логирование и сбор статистики в PostgreSQL — ключевые инструменты для мониторинга, отладки и оптимизации производительности базы данных.

## Конфигурация логирования

### Куда писать логи

```ini
# postgresql.conf

# Метод логирования
logging_collector = on        # использовать встроенный коллектор
log_destination = 'stderr'    # stderr, csvlog, jsonlog, syslog

# Директория для логов
log_directory = 'log'         # относительно PGDATA или абсолютный путь

# Имя файла лога
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
# Примеры:
# log_filename = 'postgresql-%a.log'       # по дням недели
# log_filename = 'postgresql-%Y-%m-%d.log' # по датам
```

### Ротация логов

```ini
# Ротация по времени
log_rotation_age = 1d         # новый файл каждые 24 часа

# Ротация по размеру
log_rotation_size = 100MB     # новый файл при достижении размера

# Перезапись старых файлов (при циклическом именовании)
log_truncate_on_rotation = on
```

### Форматы вывода

```ini
# CSV формат (удобно для анализа)
log_destination = 'csvlog'
logging_collector = on

# JSON формат (PostgreSQL 15+)
log_destination = 'jsonlog'

# Префикс строки лога
log_line_prefix = '%m [%p] %q%u@%d '

# Расшифровка:
# %m - timestamp с миллисекундами
# %p - PID процесса
# %q - ничего для background processes
# %u - имя пользователя
# %d - имя базы данных
```

**Рекомендуемый log_line_prefix:**
```ini
log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '
```

## Что логировать

### Логирование подключений

```ini
# Подключения и отключения
log_connections = on
log_disconnections = on

# Hostname вместо IP (медленнее)
log_hostname = off
```

### Логирование запросов

```ini
# Логировать все запросы
log_statement = 'none'        # none, ddl, mod, all
log_statement = 'ddl'         # только DDL (CREATE, ALTER, DROP)
log_statement = 'mod'         # DDL + INSERT, UPDATE, DELETE
log_statement = 'all'         # все запросы (для отладки)

# Логирование медленных запросов
log_min_duration_statement = 1000   # запросы дольше 1 секунды
log_min_duration_statement = -1     # отключено

# Логирование автоматически отменённых запросов
log_min_duration_sample = 100       # выборочно логировать от 100ms
log_statement_sample_rate = 0.1     # 10% запросов
```

### Логирование ошибок

```ini
# Минимальный уровень для лога
log_min_messages = warning     # DEBUG5..PANIC
log_min_error_statement = error # показывать запрос при ошибке

# Логирование ошибок клиенту
client_min_messages = notice
```

### Логирование checkpoints и autovacuum

```ini
log_checkpoints = on
log_autovacuum_min_duration = 0     # все операции autovacuum
log_autovacuum_min_duration = 1000  # только дольше 1 секунды
```

### Логирование блокировок

```ini
# Таймаут детекции deadlock
deadlock_timeout = 1s

# Логирование ожидания блокировок
log_lock_waits = on
```

### Логирование временных файлов

```ini
# Логировать создание временных файлов
log_temp_files = 0      # все временные файлы
log_temp_files = 10240  # только больше 10MB
```

## Расширенное логирование с auto_explain

```sql
-- Загрузка расширения
LOAD 'auto_explain';

-- Или в postgresql.conf
shared_preload_libraries = 'auto_explain'
```

```ini
# Параметры auto_explain
auto_explain.log_min_duration = '1s'    # логировать планы запросов > 1s
auto_explain.log_analyze = on           # включить ANALYZE
auto_explain.log_buffers = on           # включить информацию о буферах
auto_explain.log_timing = on            # включить timing
auto_explain.log_nested_statements = on # вложенные запросы
auto_explain.log_format = 'text'        # text, xml, json, yaml
```

## Статистика: pg_stat_statements

Расширение для сбора статистики по запросам.

### Установка и настройка

```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'

# Параметры
pg_stat_statements.max = 10000          # максимум отслеживаемых запросов
pg_stat_statements.track = all          # top, all, none
pg_stat_statements.track_utility = on   # отслеживать служебные команды
pg_stat_statements.track_planning = on  # отслеживать время планирования
```

```sql
-- Создание расширения
CREATE EXTENSION pg_stat_statements;
```

### Использование pg_stat_statements

```sql
-- Топ запросов по общему времени
SELECT
    substring(query, 1, 100) as query,
    calls,
    round(total_exec_time::numeric, 2) as total_time_ms,
    round(mean_exec_time::numeric, 2) as mean_time_ms,
    round((100 * total_exec_time / sum(total_exec_time) over())::numeric, 2) as pct
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Запросы с наибольшим временем планирования
SELECT
    substring(query, 1, 100),
    calls,
    round(mean_plan_time::numeric, 2) as mean_plan_ms,
    round(mean_exec_time::numeric, 2) as mean_exec_ms
FROM pg_stat_statements
WHERE mean_plan_time > 10
ORDER BY mean_plan_time DESC
LIMIT 10;

-- Запросы с высоким I/O
SELECT
    substring(query, 1, 100),
    calls,
    shared_blks_hit,
    shared_blks_read,
    round(100.0 * shared_blks_hit /
        NULLIF(shared_blks_hit + shared_blks_read, 0), 2) as cache_hit_pct
FROM pg_stat_statements
WHERE shared_blks_read > 1000
ORDER BY shared_blks_read DESC
LIMIT 20;

-- Сброс статистики
SELECT pg_stat_statements_reset();
```

## Встроенные представления статистики

### pg_stat_activity

Текущие сессии и запросы.

```sql
-- Активные запросы
SELECT
    pid,
    usename,
    datname,
    state,
    query,
    query_start,
    now() - query_start as duration,
    wait_event_type,
    wait_event
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start;

-- Блокирующие и заблокированные
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.pid != blocking.pid;
```

### pg_stat_user_tables

Статистика по пользовательским таблицам.

```sql
-- Статистика чтения таблиц
SELECT
    schemaname,
    relname,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    n_tup_ins,
    n_tup_upd,
    n_tup_del,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_analyze
FROM pg_stat_user_tables
ORDER BY seq_tup_read DESC
LIMIT 20;

-- Таблицы без индексов или с неэффективными индексами
SELECT
    relname,
    seq_scan,
    idx_scan,
    CASE WHEN seq_scan > 0
         THEN round(100.0 * idx_scan / (seq_scan + idx_scan), 2)
         ELSE 100
    END as idx_scan_pct
FROM pg_stat_user_tables
WHERE seq_scan + idx_scan > 100
ORDER BY idx_scan_pct ASC
LIMIT 20;
```

### pg_stat_user_indexes

Статистика по индексам.

```sql
-- Неиспользуемые индексы
SELECT
    schemaname,
    relname,
    indexrelname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelid NOT IN (SELECT conindid FROM pg_constraint)
ORDER BY pg_relation_size(indexrelid) DESC;
```

### pg_stat_database

Статистика по базам данных.

```sql
SELECT
    datname,
    numbackends,
    xact_commit,
    xact_rollback,
    blks_read,
    blks_hit,
    round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) as cache_hit_ratio,
    tup_returned,
    tup_fetched,
    tup_inserted,
    tup_updated,
    tup_deleted,
    conflicts,
    deadlocks
FROM pg_stat_database
WHERE datname = current_database();
```

### pg_stat_bgwriter

Статистика checkpoints и bgwriter (рассмотрено в предыдущей теме).

## Мониторинг в реальном времени

### pg_stat_progress_*

```sql
-- Прогресс VACUUM
SELECT * FROM pg_stat_progress_vacuum;

-- Прогресс CREATE INDEX
SELECT * FROM pg_stat_progress_create_index;

-- Прогресс ANALYZE
SELECT * FROM pg_stat_progress_analyze;

-- Прогресс CLUSTER
SELECT * FROM pg_stat_progress_cluster;

-- Прогресс COPY
SELECT * FROM pg_stat_progress_copy;
```

## Конфигурация сбора статистики

```ini
# postgresql.conf

# Сбор статистики (по умолчанию включено)
track_activities = on          # pg_stat_activity
track_counts = on              # pg_stat_* для таблиц/индексов
track_io_timing = on           # детальный I/O timing (небольшой overhead)
track_wal_io_timing = on       # WAL I/O timing (PostgreSQL 14+)
track_functions = all          # отслеживание функций (none, pl, all)

# Размер буфера для запросов в pg_stat_activity
track_activity_query_size = 4096  # по умолчанию 1024
```

## Полезные запросы для мониторинга

### Общее здоровье системы

```sql
-- Сводка по базе
SELECT
    'Database Size' as metric, pg_size_pretty(pg_database_size(current_database())) as value
UNION ALL
SELECT
    'Active Connections', count(*)::text
FROM pg_stat_activity WHERE state = 'active'
UNION ALL
SELECT
    'Cache Hit Ratio',
    round(100.0 * sum(blks_hit) / NULLIF(sum(blks_hit + blks_read), 0), 2)::text || '%'
FROM pg_stat_database WHERE datname = current_database();
```

### Долгие транзакции

```sql
SELECT
    pid,
    now() - xact_start as transaction_duration,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND state != 'idle'
ORDER BY xact_start
LIMIT 10;
```

## Типичные ошибки

1. **log_statement = 'all' в production** — огромные логи, падение производительности
2. **Отключённый track_io_timing** — нет информации о I/O
3. **Малый track_activity_query_size** — обрезанные запросы
4. **Игнорирование pg_stat_statements** — нет истории запросов
5. **Отсутствие ротации логов** — заканчивается место на диске

## Best Practices

1. **Используйте log_min_duration_statement** вместо log_statement = 'all'
2. **Включите pg_stat_statements** — незаменимо для анализа
3. **Настройте CSV или JSON логи** для автоматического анализа
4. **Включите log_checkpoints и log_autovacuum** для диагностики
5. **Мониторьте cache_hit_ratio** — должен быть > 99%
6. **Настройте алерты** на deadlocks, long transactions, connection count
7. **Регулярно анализируйте pg_stat_statements** для оптимизации запросов

```ini
# Рекомендуемая конфигурация для production
logging_collector = on
log_destination = 'csvlog'
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d.log'
log_rotation_age = 1d
log_rotation_size = 0
log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '
log_min_duration_statement = 500
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 0

track_activities = on
track_counts = on
track_io_timing = on
track_functions = all
track_activity_query_size = 4096
```

---

[prev: 05-checkpoints-bgwriter](./05-checkpoints-bgwriter.md) | [next: 07-extensions](./07-extensions.md)
