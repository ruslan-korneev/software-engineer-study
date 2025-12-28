# Встроенные инструменты PostgreSQL для диагностики

[prev: 04-system-views](./04-system-views.md) | [next: 01-explain](../19-query-analysis/01-explain.md)

---

PostgreSQL включает множество встроенных инструментов и функций для анализа производительности и диагностики проблем. В этом разделе рассмотрим основные инструменты: pg_stat_*, функции для анализа размеров и EXPLAIN ANALYZE.

## Функции pg_stat_*

### pg_stat_database

Статистика на уровне базы данных.

```sql
-- Общая статистика базы данных
SELECT
    datname,
    numbackends,
    xact_commit,
    xact_rollback,
    round(100.0 * xact_rollback / NULLIF(xact_commit + xact_rollback, 0), 2) as rollback_pct,
    blks_read,
    blks_hit,
    round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) as cache_hit_ratio,
    tup_returned,
    tup_fetched,
    tup_inserted,
    tup_updated,
    tup_deleted,
    conflicts,
    temp_files,
    pg_size_pretty(temp_bytes) as temp_bytes,
    deadlocks,
    checksum_failures,
    stats_reset
FROM pg_stat_database
WHERE datname = current_database();

-- Сравнение баз данных
SELECT
    datname,
    numbackends as connections,
    round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) as cache_hit,
    temp_files,
    deadlocks
FROM pg_stat_database
WHERE datname NOT LIKE 'template%'
ORDER BY numbackends DESC;
```

### pg_stat_all_tables / pg_stat_user_tables

```sql
-- Полная статистика таблиц
SELECT
    schemaname,
    relname,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    n_tup_ins as inserts,
    n_tup_upd as updates,
    n_tup_del as deletes,
    n_tup_hot_upd as hot_updates,
    n_live_tup as live_rows,
    n_dead_tup as dead_rows,
    n_mod_since_analyze as mods_since_analyze,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    vacuum_count,
    autovacuum_count,
    analyze_count,
    autoanalyze_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;

-- Эффективность HOT updates
SELECT
    schemaname,
    relname,
    n_tup_upd as total_updates,
    n_tup_hot_upd as hot_updates,
    round(100.0 * n_tup_hot_upd / NULLIF(n_tup_upd, 0), 2) as hot_update_pct
FROM pg_stat_user_tables
WHERE n_tup_upd > 100
ORDER BY n_tup_upd DESC;
```

### pg_stat_all_indexes / pg_stat_user_indexes

```sql
-- Статистика использования индексов
SELECT
    schemaname,
    relname as table_name,
    indexrelname as index_name,
    idx_scan as scans,
    idx_tup_read as tuples_read,
    idx_tup_fetch as tuples_fetched
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC
LIMIT 20;

-- Индексы которые никогда не использовались
SELECT
    schemaname,
    relname as table_name,
    indexrelname as index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Редко используемые индексы (кандидаты на удаление)
SELECT
    schemaname,
    relname,
    indexrelname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) as size,
    pg_relation_size(indexrelid) as size_bytes
FROM pg_stat_user_indexes
WHERE idx_scan < 50
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;
```

## Функции для анализа размеров

### Размеры объектов

```sql
-- Размер базы данных
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) as size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;

-- Размер текущей базы данных
SELECT pg_size_pretty(pg_database_size(current_database()));

-- Размеры таблиц с индексами
SELECT
    schemaname,
    relname,
    pg_size_pretty(pg_total_relation_size(relid)) as total_size,
    pg_size_pretty(pg_relation_size(relid)) as table_size,
    pg_size_pretty(pg_indexes_size(relid)) as indexes_size,
    pg_size_pretty(pg_total_relation_size(relid) - pg_relation_size(relid) - pg_indexes_size(relid)) as toast_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 20;

-- Детальный размер таблицы
SELECT
    relname,
    pg_size_pretty(pg_relation_size(oid)) as table_size,
    pg_size_pretty(pg_table_size(oid)) as table_with_toast,
    pg_size_pretty(pg_indexes_size(oid)) as indexes_size,
    pg_size_pretty(pg_total_relation_size(oid)) as total_size
FROM pg_class
WHERE relname = 'my_table';
```

### Размеры индексов

```sql
-- Все индексы с размерами
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexname::regclass)) as size
FROM pg_indexes
WHERE schemaname = 'public'
ORDER BY pg_relation_size(indexname::regclass) DESC;

-- Соотношение размера индексов к таблице
SELECT
    t.relname as table_name,
    pg_size_pretty(pg_relation_size(t.oid)) as table_size,
    pg_size_pretty(pg_indexes_size(t.oid)) as indexes_size,
    round(100.0 * pg_indexes_size(t.oid) / NULLIF(pg_relation_size(t.oid), 0), 2) as index_to_table_pct
FROM pg_class t
WHERE t.relkind = 'r'
  AND t.relnamespace = 'public'::regnamespace
ORDER BY pg_total_relation_size(t.oid) DESC
LIMIT 20;
```

### Bloat анализ

```sql
-- Оценка bloat таблиц
WITH constants AS (
    SELECT current_setting('block_size')::numeric AS bs
),
table_stats AS (
    SELECT
        schemaname,
        tablename,
        reltuples::bigint AS est_rows,
        relpages * (SELECT bs FROM constants) AS table_bytes
    FROM pg_class c
    JOIN pg_stat_user_tables s ON c.relname = s.relname
    WHERE relpages > 0
)
SELECT
    schemaname,
    tablename,
    est_rows,
    pg_size_pretty(table_bytes) as size,
    pg_size_pretty(pg_total_relation_size(tablename::regclass)) as actual_size
FROM table_stats
ORDER BY table_bytes DESC
LIMIT 20;

-- Более точный анализ bloat (требует pgstattuple)
-- CREATE EXTENSION pgstattuple;
SELECT
    relname,
    pg_size_pretty(table_len) as table_size,
    round(100.0 * dead_tuple_len / NULLIF(table_len, 0), 2) as dead_pct,
    pg_size_pretty(dead_tuple_len) as dead_size,
    pg_size_pretty(free_space) as free_space
FROM (
    SELECT
        relname,
        (pgstattuple(oid)).*
    FROM pg_class
    WHERE relkind = 'r'
      AND relnamespace = 'public'::regnamespace
) s
ORDER BY dead_tuple_len DESC
LIMIT 20;
```

## EXPLAIN и EXPLAIN ANALYZE

### Базовое использование

```sql
-- План без выполнения
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;

-- План с реальными метриками
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 123;

-- С информацией о буферах
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE customer_id = 123;

-- Полная информация
EXPLAIN (ANALYZE, BUFFERS, TIMING, COSTS, VERBOSE)
SELECT * FROM orders WHERE customer_id = 123;

-- Форматы вывода
EXPLAIN (FORMAT JSON) SELECT * FROM orders;
EXPLAIN (FORMAT YAML) SELECT * FROM orders;
EXPLAIN (FORMAT XML) SELECT * FROM orders;
```

### Чтение плана выполнения

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at > '2024-01-01';

/*
Hash Join  (cost=10.00..50.00 rows=100 width=200) (actual time=0.5..2.3 rows=95 loops=1)
  Hash Cond: (o.customer_id = c.id)
  Buffers: shared hit=15 read=5
  ->  Seq Scan on orders o  (cost=0.00..35.00 rows=100 width=150) (actual time=0.1..1.0 rows=95 loops=1)
        Filter: (created_at > '2024-01-01'::date)
        Rows Removed by Filter: 905
        Buffers: shared hit=10 read=5
  ->  Hash  (cost=8.00..8.00 rows=50 width=50) (actual time=0.3..0.3 rows=50 loops=1)
        Buckets: 1024  Batches: 1  Memory Usage: 10kB
        Buffers: shared hit=5
        ->  Seq Scan on customers c  (cost=0.00..8.00 rows=50 width=50) (actual time=0.05..0.2 rows=50 loops=1)
              Buffers: shared hit=5
Planning Time: 0.2 ms
Execution Time: 2.5 ms
*/
```

**Ключевые метрики:**
- **cost** - оценка стоимости (startup..total)
- **rows** - ожидаемое количество строк
- **actual time** - реальное время (startup..total в мс)
- **rows** (actual) - реальное количество строк
- **loops** - количество итераций
- **Buffers: shared hit** - страницы из кэша
- **Buffers: shared read** - страницы с диска

### Типичные проблемы в планах

```sql
-- 1. Seq Scan на большой таблице вместо Index Scan
-- Проблема: отсутствует индекс или не используется
EXPLAIN ANALYZE SELECT * FROM big_table WHERE status = 'active';
-- Решение: создать индекс или проверить статистику

-- 2. Nested Loop с большим количеством loops
-- Проблема: неэффективный JOIN
EXPLAIN ANALYZE
SELECT * FROM orders o
JOIN order_items i ON o.id = i.order_id;
-- Решение: проверить индексы на колонках JOIN

-- 3. Sort с высоким cost
-- Проблема: сортировка в памяти или на диске
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products ORDER BY price LIMIT 100;
-- Решение: создать индекс для сортировки или увеличить work_mem

-- 4. Hash/Sort с "Batches > 1" или "Sort Method: external merge"
-- Проблема: недостаточно work_mem
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, count(*)
FROM orders
GROUP BY customer_id;
-- Решение: SET work_mem = '256MB';
```

### Сравнение планов

```sql
-- Сохранение планов для сравнения
CREATE TABLE query_plans (
    id serial PRIMARY KEY,
    query_name text,
    plan_json jsonb,
    created_at timestamp DEFAULT now()
);

-- Сохранение плана
INSERT INTO query_plans (query_name, plan_json)
SELECT
    'orders_by_date',
    (SELECT * FROM json_agg(q) FROM (
        EXPLAIN (FORMAT JSON)
        SELECT * FROM orders WHERE created_at > '2024-01-01'
    ) q);

-- Анализ различий в планах
SELECT
    query_name,
    plan_json->>0->'Plan'->'Node Type' as node_type,
    (plan_json->>0->'Plan'->>'Total Cost')::numeric as cost,
    created_at
FROM query_plans
WHERE query_name = 'orders_by_date'
ORDER BY created_at DESC
LIMIT 5;
```

## auto_explain для автоматического сбора планов

```sql
-- Включение auto_explain для сессии
LOAD 'auto_explain';
SET auto_explain.log_min_duration = '100ms';
SET auto_explain.log_analyze = true;
SET auto_explain.log_buffers = true;
SET auto_explain.log_timing = true;
SET auto_explain.log_nested_statements = true;

-- Теперь все запросы > 100ms будут логироваться с планами
```

## pg_stat_statements + EXPLAIN

```sql
-- Получить план для часто выполняемого запроса
-- 1. Найти queryid проблемного запроса
SELECT queryid, calls, mean_exec_time, query
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- 2. Скопировать query и выполнить EXPLAIN ANALYZE
EXPLAIN (ANALYZE, BUFFERS)
<скопированный запрос>;
```

## Дополнительные диагностические функции

### pg_stat_get_* функции

```sql
-- Статистика блокировок
SELECT
    locktype,
    mode,
    count(*)
FROM pg_locks
GROUP BY locktype, mode
ORDER BY count(*) DESC;

-- Статистика WAL
SELECT * FROM pg_stat_wal;

-- Статистика архивации
SELECT * FROM pg_stat_archiver;

-- Статистика SSL соединений
SELECT
    ssl,
    count(*) as connections
FROM pg_stat_ssl
GROUP BY ssl;
```

### Информация о конфигурации

```sql
-- Текущие настройки
SELECT name, setting, unit, context
FROM pg_settings
WHERE name IN (
    'shared_buffers',
    'work_mem',
    'maintenance_work_mem',
    'effective_cache_size',
    'max_connections',
    'checkpoint_completion_target'
);

-- Настройки отличающиеся от defaults
SELECT name, setting, boot_val, source
FROM pg_settings
WHERE setting != boot_val
  AND source != 'default'
ORDER BY source, name;

-- Настройки с non-default источником
SELECT name, setting, source, sourcefile, sourceline
FROM pg_settings
WHERE source NOT IN ('default', 'override')
ORDER BY source;
```

## Скрипт комплексной диагностики

```sql
-- Полный отчёт о состоянии базы данных
\echo '=== Database Overview ==='
SELECT
    current_database() as database,
    pg_size_pretty(pg_database_size(current_database())) as size,
    (SELECT count(*) FROM pg_stat_activity WHERE datname = current_database()) as connections;

\echo '\n=== Top Tables by Size ==='
SELECT
    relname,
    pg_size_pretty(pg_total_relation_size(relid)) as total_size,
    n_live_tup as rows
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 10;

\echo '\n=== Index Usage ==='
SELECT
    relname,
    indexrelname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
ORDER BY idx_scan
LIMIT 10;

\echo '\n=== Tables Needing Vacuum ==='
SELECT
    relname,
    n_dead_tup,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC
LIMIT 10;

\echo '\n=== Cache Hit Ratio ==='
SELECT
    round(100.0 * sum(heap_blks_hit) /
          NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) as cache_hit_ratio
FROM pg_statio_user_tables;

\echo '\n=== Active Queries ==='
SELECT
    pid,
    now() - query_start as duration,
    state,
    left(query, 50) as query
FROM pg_stat_activity
WHERE state = 'active'
  AND pid != pg_backend_pid()
ORDER BY query_start;
```

## Инструменты визуализации планов

### explain.depesz.com

```sql
-- Получить план в текстовом формате
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) <ваш запрос>;
-- Скопировать вывод на https://explain.depesz.com
```

### pgMustard, explain.dalibo.com

```sql
-- JSON формат для визуализации
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) <ваш запрос>;
```

## Заключение

Встроенные инструменты PostgreSQL предоставляют:

1. **pg_stat_* представления** - мониторинг на всех уровнях
2. **Функции размеров** - анализ использования дискового пространства
3. **EXPLAIN ANALYZE** - детальный анализ выполнения запросов
4. **pg_settings** - анализ конфигурации

Регулярное использование этих инструментов помогает:
- Находить проблемы производительности до их критического проявления
- Понимать поведение запросов
- Планировать оптимизации и масштабирование
- Поддерживать здоровое состояние базы данных

---

[prev: 04-system-views](./04-system-views.md) | [next: 01-explain](../19-query-analysis/01-explain.md)
