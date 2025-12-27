# Системные представления PostgreSQL для диагностики

PostgreSQL предоставляет множество системных представлений (system views) и каталогов для мониторинга и диагностики. Эти представления позволяют анализировать активность, блокировки, производительность и состояние базы данных в реальном времени.

## pg_stat_activity

Главное представление для мониторинга активных сессий и запросов.

### Основные запросы

```sql
-- Все активные соединения
SELECT
    pid,
    usename,
    datname,
    state,
    query_start,
    now() - query_start as duration,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

-- Количество соединений по состояниям
SELECT
    state,
    count(*) as connections
FROM pg_stat_activity
GROUP BY state
ORDER BY count(*) DESC;

-- Соединения по пользователям и базам
SELECT
    usename,
    datname,
    count(*) as connections,
    count(*) FILTER (WHERE state = 'active') as active,
    count(*) FILTER (WHERE state = 'idle') as idle,
    count(*) FILTER (WHERE state = 'idle in transaction') as idle_in_transaction
FROM pg_stat_activity
GROUP BY usename, datname
ORDER BY count(*) DESC;
```

### Поиск проблемных сессий

```sql
-- Долгие запросы (> 5 минут)
SELECT
    pid,
    usename,
    datname,
    now() - query_start as duration,
    state,
    query
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - query_start > interval '5 minutes'
ORDER BY query_start;

-- Сессии с открытыми транзакциями
SELECT
    pid,
    usename,
    datname,
    now() - xact_start as transaction_duration,
    now() - query_start as query_duration,
    state,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND state != 'active'
ORDER BY xact_start;

-- Ожидающие сессии
SELECT
    pid,
    usename,
    wait_event_type,
    wait_event,
    state,
    query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
ORDER BY wait_event_type;
```

### Завершение проблемных сессий

```sql
-- Мягкое завершение (отмена запроса)
SELECT pg_cancel_backend(pid);

-- Принудительное завершение (терминация)
SELECT pg_terminate_backend(pid);

-- Завершить все сессии конкретного пользователя
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE usename = 'problem_user'
  AND pid != pg_backend_pid();

-- Завершить все idle in transaction > 10 минут
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - state_change > interval '10 minutes';
```

## pg_locks

Представление для анализа блокировок.

### Просмотр текущих блокировок

```sql
-- Все блокировки
SELECT
    l.pid,
    l.locktype,
    l.mode,
    l.granted,
    l.relation::regclass as table_name,
    a.usename,
    a.query
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE l.relation IS NOT NULL
ORDER BY l.relation;

-- Ожидающие блокировки
SELECT
    l.pid,
    l.locktype,
    l.mode,
    l.relation::regclass as table_name,
    a.query,
    a.wait_event
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE NOT l.granted
ORDER BY l.pid;
```

### Анализ блокировок между сессиями

```sql
-- Кто кого блокирует
SELECT
    blocked.pid AS blocked_pid,
    blocked.usename AS blocked_user,
    blocking.pid AS blocking_pid,
    blocking.usename AS blocking_user,
    blocked.query AS blocked_query,
    blocking.query AS blocking_query
FROM pg_locks blocked_locks
JOIN pg_stat_activity blocked ON blocked.pid = blocked_locks.pid
JOIN pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.relation = blocked_locks.relation
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_stat_activity blocking ON blocking.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted
  AND blocking_locks.granted;

-- Дерево блокировок (PostgreSQL 14+)
SELECT
    pid,
    usename,
    pg_blocking_pids(pid) as blocked_by,
    query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

### Deadlock анализ

```sql
-- Циклические зависимости (потенциальные deadlock)
WITH RECURSIVE lock_chain AS (
    SELECT
        pid,
        ARRAY[pid] as chain,
        false as is_cycle
    FROM pg_locks
    WHERE NOT granted

    UNION ALL

    SELECT
        pl.pid,
        lc.chain || pl.pid,
        pl.pid = ANY(lc.chain)
    FROM lock_chain lc
    JOIN pg_locks pl ON pl.granted
        AND pl.relation IN (
            SELECT relation FROM pg_locks WHERE pid = lc.chain[array_upper(lc.chain, 1)]
        )
    WHERE NOT lc.is_cycle
)
SELECT DISTINCT chain
FROM lock_chain
WHERE is_cycle;
```

## pg_stat_* представления для таблиц

### pg_stat_user_tables

```sql
-- Статистика по таблицам
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
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_ratio,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Таблицы без индексных сканов (потенциальные проблемы)
SELECT
    schemaname,
    relname,
    seq_scan,
    seq_tup_read,
    idx_scan
FROM pg_stat_user_tables
WHERE seq_scan > 100
  AND (idx_scan = 0 OR idx_scan IS NULL)
  AND seq_tup_read > 10000
ORDER BY seq_tup_read DESC;

-- Таблицы требующие VACUUM
SELECT
    schemaname,
    relname,
    n_dead_tup,
    n_live_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup, 0), 2) as dead_pct,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

### pg_stat_user_indexes

```sql
-- Использование индексов
SELECT
    schemaname,
    relname,
    indexrelname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Неиспользуемые индексы
SELECT
    schemaname,
    relname,
    indexrelname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelid NOT IN (
      SELECT conindid FROM pg_constraint
      WHERE contype IN ('p', 'u')  -- primary key, unique
  )
ORDER BY pg_relation_size(indexrelid) DESC;
```

## pg_stat_* представления для I/O

### pg_stat_io (PostgreSQL 16+)

```sql
-- Статистика I/O операций
SELECT
    backend_type,
    object,
    context,
    reads,
    writes,
    extends,
    op_bytes,
    hits,
    evictions
FROM pg_stat_io
WHERE reads > 0 OR writes > 0
ORDER BY reads + writes DESC;
```

### pg_statio_user_tables

```sql
-- Статистика I/O по таблицам
SELECT
    schemaname,
    relname,
    heap_blks_read,
    heap_blks_hit,
    round(100.0 * heap_blks_hit / NULLIF(heap_blks_hit + heap_blks_read, 0), 2) as cache_hit_ratio,
    idx_blks_read,
    idx_blks_hit
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC;

-- Таблицы с плохим cache hit ratio
SELECT
    schemaname,
    relname,
    heap_blks_read,
    heap_blks_hit,
    round(100.0 * heap_blks_hit / NULLIF(heap_blks_hit + heap_blks_read, 0), 2) as cache_hit_ratio
FROM pg_statio_user_tables
WHERE heap_blks_read + heap_blks_hit > 1000
  AND heap_blks_hit / NULLIF(heap_blks_hit + heap_blks_read, 0) < 0.9
ORDER BY heap_blks_read DESC;
```

## pg_stat_bgwriter

```sql
-- Статистика фонового писателя
SELECT
    checkpoints_timed,
    checkpoints_req,
    checkpoint_write_time,
    checkpoint_sync_time,
    buffers_checkpoint,
    buffers_clean,
    maxwritten_clean,
    buffers_backend,
    buffers_backend_fsync,
    buffers_alloc,
    stats_reset
FROM pg_stat_bgwriter;

-- Анализ чекпоинтов
SELECT
    checkpoints_timed as scheduled,
    checkpoints_req as requested,
    round(100.0 * checkpoints_req / NULLIF(checkpoints_timed + checkpoints_req, 0), 2) as requested_pct,
    round(checkpoint_write_time / 1000.0, 2) as write_time_sec,
    round(checkpoint_sync_time / 1000.0, 2) as sync_time_sec
FROM pg_stat_bgwriter;
```

## pg_stat_replication

```sql
-- Статус репликации
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    client_hostname,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag,
    sync_state,
    sync_priority
FROM pg_stat_replication;

-- Отставание репликации
SELECT
    client_addr,
    application_name,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) as bytes_lag,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) as lag_pretty,
    replay_lag
FROM pg_stat_replication;
```

## pg_stat_progress_*

### Мониторинг долгих операций

```sql
-- Прогресс VACUUM
SELECT
    pid,
    datname,
    relid::regclass as table_name,
    phase,
    heap_blks_total,
    heap_blks_scanned,
    round(100.0 * heap_blks_scanned / NULLIF(heap_blks_total, 0), 2) as pct_complete,
    index_vacuum_count,
    max_dead_tuples,
    num_dead_tuples
FROM pg_stat_progress_vacuum;

-- Прогресс CREATE INDEX
SELECT
    pid,
    datname,
    relid::regclass as table_name,
    index_relid::regclass as index_name,
    phase,
    blocks_total,
    blocks_done,
    round(100.0 * blocks_done / NULLIF(blocks_total, 0), 2) as pct_complete,
    tuples_total,
    tuples_done
FROM pg_stat_progress_create_index;

-- Прогресс CLUSTER/VACUUM FULL
SELECT
    pid,
    datname,
    relid::regclass as table_name,
    phase,
    heap_tuples_scanned,
    heap_tuples_written,
    heap_blks_total,
    heap_blks_scanned
FROM pg_stat_progress_cluster;

-- Прогресс COPY
SELECT
    pid,
    datname,
    relid::regclass as table_name,
    command,
    type,
    bytes_processed,
    bytes_total,
    round(100.0 * bytes_processed / NULLIF(bytes_total, 0), 2) as pct_complete,
    tuples_processed,
    tuples_excluded
FROM pg_stat_progress_copy;

-- Прогресс ANALYZE
SELECT
    pid,
    datname,
    relid::regclass as table_name,
    phase,
    sample_blks_total,
    sample_blks_scanned,
    ext_stats_total,
    ext_stats_computed
FROM pg_stat_progress_analyze;
```

## Комплексный мониторинг

### Dashboard-запрос

```sql
-- Общий статус базы данных
WITH
connections AS (
    SELECT
        count(*) as total,
        count(*) FILTER (WHERE state = 'active') as active,
        count(*) FILTER (WHERE state = 'idle') as idle,
        count(*) FILTER (WHERE state = 'idle in transaction') as idle_in_tx,
        count(*) FILTER (WHERE wait_event IS NOT NULL) as waiting
    FROM pg_stat_activity
),
locks AS (
    SELECT count(*) as blocked FROM pg_locks WHERE NOT granted
),
cache AS (
    SELECT
        round(100.0 * sum(heap_blks_hit) /
              NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) as ratio
    FROM pg_statio_user_tables
),
replication AS (
    SELECT
        count(*) as replicas,
        max(replay_lag) as max_lag
    FROM pg_stat_replication
)
SELECT
    c.total as connections,
    c.active as active_queries,
    c.idle_in_tx as idle_in_transaction,
    l.blocked as blocked_queries,
    ca.ratio as cache_hit_ratio,
    r.replicas,
    r.max_lag as replication_lag
FROM connections c, locks l, cache ca, replication r;
```

### Функция для регулярного мониторинга

```sql
CREATE OR REPLACE FUNCTION get_database_health()
RETURNS TABLE (
    metric text,
    value text,
    status text
) AS $$
BEGIN
    -- Connections
    RETURN QUERY
    SELECT
        'connections'::text,
        count(*)::text,
        CASE
            WHEN count(*) > current_setting('max_connections')::int * 0.8
            THEN 'warning' ELSE 'ok'
        END
    FROM pg_stat_activity;

    -- Blocked queries
    RETURN QUERY
    SELECT
        'blocked_queries'::text,
        count(*)::text,
        CASE WHEN count(*) > 0 THEN 'warning' ELSE 'ok' END
    FROM pg_locks
    WHERE NOT granted;

    -- Long running queries
    RETURN QUERY
    SELECT
        'long_queries_5min'::text,
        count(*)::text,
        CASE WHEN count(*) > 0 THEN 'warning' ELSE 'ok' END
    FROM pg_stat_activity
    WHERE state = 'active'
      AND now() - query_start > interval '5 minutes';

    -- Cache hit ratio
    RETURN QUERY
    SELECT
        'cache_hit_ratio'::text,
        round(100.0 * sum(heap_blks_hit) /
              NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2)::text,
        CASE
            WHEN round(100.0 * sum(heap_blks_hit) /
                       NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) < 90
            THEN 'warning' ELSE 'ok'
        END
    FROM pg_statio_user_tables;

    RETURN;
END;
$$ LANGUAGE plpgsql;

-- Использование
SELECT * FROM get_database_health();
```

## Сброс статистики

```sql
-- Сброс всей статистики
SELECT pg_stat_reset();

-- Сброс статистики для конкретной таблицы
SELECT pg_stat_reset_single_table_counters('public.my_table'::regclass);

-- Сброс статистики bgwriter
SELECT pg_stat_reset_shared('bgwriter');

-- Сброс статистики archiver
SELECT pg_stat_reset_shared('archiver');
```

## Заключение

Системные представления PostgreSQL предоставляют всю необходимую информацию для:
- Мониторинга активных сессий и запросов
- Анализа блокировок и deadlock
- Отслеживания производительности I/O
- Мониторинга репликации
- Наблюдения за долгими операциями

Рекомендуется создать набор мониторинговых запросов и использовать их регулярно или интегрировать в систему мониторинга (Prometheus, Zabbix, Grafana).
