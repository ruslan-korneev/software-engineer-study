# Настройка PostgreSQL под рабочую нагрузку

## Типы рабочих нагрузок

### OLTP (Online Transaction Processing)

**Характеристики:**
- Много коротких транзакций
- Простые запросы (SELECT по ключу, INSERT, UPDATE)
- Высокий concurrency
- Малый объем данных в каждом запросе
- Низкая латентность критична

**Примеры:** e-commerce, банкинг, CRM

### OLAP (Online Analytical Processing)

**Характеристики:**
- Мало длинных запросов
- Сложные аналитические запросы с агрегацией
- Низкий concurrency
- Большие объемы данных
- Throughput важнее латентности

**Примеры:** отчетность, BI, data warehouse

### Mixed Workload

Комбинация OLTP и OLAP — наиболее сложный случай для оптимизации.

## Настройка для OLTP

### Память

```sql
-- OLTP: много коротких запросов, нужен быстрый доступ к данным
ALTER SYSTEM SET shared_buffers = '8GB';        -- 25% RAM
ALTER SYSTEM SET effective_cache_size = '24GB'; -- 75% RAM
ALTER SYSTEM SET work_mem = '16MB';             -- небольшой (много сессий)
ALTER SYSTEM SET maintenance_work_mem = '1GB';

-- Минимизация контекстных переключений
ALTER SYSTEM SET huge_pages = 'try';
```

### WAL и Checkpoints

```sql
-- Частые мелкие записи — оптимизация WAL
ALTER SYSTEM SET wal_level = 'replica';
ALTER SYSTEM SET synchronous_commit = 'on';     -- или 'off' для большей скорости
ALTER SYSTEM SET wal_buffers = '64MB';
ALTER SYSTEM SET checkpoint_completion_target = 0.9;
ALTER SYSTEM SET checkpoint_timeout = '15min';

-- max_wal_size влияет на время восстановления
ALTER SYSTEM SET max_wal_size = '4GB';
```

### Параллелизм

```sql
-- OLTP: много соединений, мало параллельных воркеров на запрос
ALTER SYSTEM SET max_connections = 200;
ALTER SYSTEM SET max_parallel_workers_per_gather = 0;  -- выключаем параллелизм
ALTER SYSTEM SET max_parallel_workers = 0;
```

### Планировщик

```sql
-- OLTP: простые запросы, меньше времени на планирование
ALTER SYSTEM SET random_page_cost = 1.1;        -- SSD
ALTER SYSTEM SET effective_io_concurrency = 200; -- SSD
```

## Настройка для OLAP

### Память

```sql
-- OLAP: мало соединений, большие запросы
ALTER SYSTEM SET shared_buffers = '16GB';       -- можно больше
ALTER SYSTEM SET effective_cache_size = '48GB';
ALTER SYSTEM SET work_mem = '256MB';            -- большой (мало сессий, сложные сортировки)
ALTER SYSTEM SET maintenance_work_mem = '4GB';
ALTER SYSTEM SET temp_buffers = '256MB';
```

### Параллелизм

```sql
-- OLAP: максимальный параллелизм для сложных запросов
ALTER SYSTEM SET max_parallel_workers_per_gather = 4;
ALTER SYSTEM SET max_parallel_workers = 8;
ALTER SYSTEM SET parallel_tuple_cost = 0.01;
ALTER SYSTEM SET parallel_setup_cost = 100;
ALTER SYSTEM SET min_parallel_table_scan_size = '8MB';
ALTER SYSTEM SET min_parallel_index_scan_size = '512kB';
```

### Планировщик

```sql
-- OLAP: более агрессивное использование индексов и join-стратегий
ALTER SYSTEM SET random_page_cost = 1.1;
ALTER SYSTEM SET effective_io_concurrency = 200;
ALTER SYSTEM SET enable_partitionwise_join = on;
ALTER SYSTEM SET enable_partitionwise_aggregate = on;
ALTER SYSTEM SET jit = on;                       -- Just-In-Time компиляция
ALTER SYSTEM SET jit_above_cost = 100000;
```

### WAL

```sql
-- OLAP: меньше записей, можно реже checkpoints
ALTER SYSTEM SET checkpoint_timeout = '30min';
ALTER SYSTEM SET max_wal_size = '16GB';
```

## Профилирование запросов

### pg_stat_statements

```sql
-- Установка
CREATE EXTENSION pg_stat_statements;

-- Топ запросов по времени
SELECT
    substring(query, 1, 100) as query,
    calls,
    round(total_exec_time::numeric, 2) as total_ms,
    round(mean_exec_time::numeric, 2) as mean_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Запросы с наибольшим количеством чтений
SELECT
    substring(query, 1, 100) as query,
    calls,
    shared_blks_read,
    shared_blks_hit,
    round(100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0), 2) as hit_ratio
FROM pg_stat_statements
ORDER BY shared_blks_read DESC
LIMIT 10;

-- Сброс статистики
SELECT pg_stat_statements_reset();
```

### EXPLAIN ANALYZE

```sql
-- Детальный анализ запроса
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders
JOIN customers ON orders.customer_id = customers.id
WHERE orders.created_at > '2024-01-01';

-- Понимание вывода
/*
Hash Join  (cost=1.25..2.50 rows=10 width=100) (actual time=0.05..0.07 rows=10 loops=1)
  Hash Cond: (orders.customer_id = customers.id)
  Buffers: shared hit=5
  ->  Seq Scan on orders  (cost=0.00..1.10 rows=10 width=50) (actual time=0.01..0.02 rows=10 loops=1)
        Buffers: shared hit=1
  ->  Hash  (cost=1.10..1.10 rows=10 width=50) (actual time=0.02..0.02 rows=10 loops=1)
        Buckets: 1024  Batches: 1  Memory Usage: 9kB
        ->  Seq Scan on customers  (cost=0.00..1.10 rows=10 width=50)
Planning Time: 0.15 ms
Execution Time: 0.10 ms
*/
```

### auto_explain

```sql
-- Логирование планов медленных запросов
ALTER SYSTEM SET auto_explain.log_min_duration = '1s';
ALTER SYSTEM SET auto_explain.log_analyze = true;
ALTER SYSTEM SET auto_explain.log_buffers = true;

-- Добавить в postgresql.conf
-- shared_preload_libraries = 'auto_explain'
```

## Мониторинг нагрузки

### Текущая активность

```sql
-- Активные запросы
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    query_start,
    now() - query_start as duration,
    substring(query, 1, 100) as query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start;

-- Ожидающие блокировки
SELECT
    pid,
    wait_event_type,
    wait_event,
    count(*)
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
GROUP BY 1, 2, 3
ORDER BY 4 DESC;
```

### Статистика базы данных

```sql
-- Общая статистика
SELECT
    datname,
    numbackends,
    xact_commit,
    xact_rollback,
    blks_read,
    blks_hit,
    round(100.0 * blks_hit / nullif(blks_hit + blks_read, 0), 2) as cache_hit_ratio,
    tup_returned,
    tup_fetched,
    tup_inserted,
    tup_updated,
    tup_deleted
FROM pg_stat_database
WHERE datname = current_database();
```

### Использование индексов

```sql
-- Эффективность индексов
SELECT
    schemaname || '.' || relname as table,
    indexrelname as index,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC
LIMIT 20;

-- Таблицы с высоким seq scan
SELECT
    schemaname || '.' || relname as table,
    seq_scan,
    seq_tup_read,
    idx_scan,
    round(100.0 * idx_scan / nullif(seq_scan + idx_scan, 0), 2) as idx_scan_pct
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC
LIMIT 10;
```

## Оптимизация конкретных сценариев

### Bulk Insert

```sql
-- Отключение ограничений на время загрузки
ALTER TABLE big_table DISABLE TRIGGER ALL;
ALTER TABLE big_table SET (autovacuum_enabled = false);

-- Использование COPY вместо INSERT
COPY big_table FROM '/path/to/data.csv' WITH CSV;

-- Включение обратно
ALTER TABLE big_table ENABLE TRIGGER ALL;
ALTER TABLE big_table SET (autovacuum_enabled = true);
ANALYZE big_table;
```

### Batch Processing

```sql
-- Обработка пакетами с LIMIT/OFFSET (плохо для больших offset)
-- Лучше: keyset pagination
WITH batch AS (
    SELECT id FROM tasks
    WHERE status = 'pending' AND id > $last_id
    ORDER BY id
    LIMIT 1000
)
UPDATE tasks SET status = 'processing'
FROM batch
WHERE tasks.id = batch.id;
```

### Connection Pooling

```sql
-- Настройка для работы с пулером
ALTER SYSTEM SET max_connections = 100;  -- реальных соединений мало
ALTER SYSTEM SET superuser_reserved_connections = 3;

-- Оптимизация для pgbouncer
-- Избегайте: prepared statements, advisory locks в transaction mode
```

## Автоматическая настройка

### PGTune

Онлайн-калькулятор настроек: https://pgtune.leopard.in.ua/

### TimescaleDB tune

```bash
# Автоматическая настройка на основе ресурсов
timescaledb-tune --memory 32GB --cpus 8
```

## Best Practices

1. **Определите тип нагрузки** перед настройкой
2. **Используйте pg_stat_statements** для выявления проблемных запросов
3. **Мониторьте cache hit ratio** — должен быть > 99%
4. **Настройте autovacuum** для вашей нагрузки
5. **Используйте connection pooler** для OLTP
6. **Партиционирование** для больших таблиц в OLAP
7. **Регулярно обновляйте статистику** — ANALYZE после изменений
8. **Тестируйте изменения** на staging перед production

```sql
-- Итоговая проверка настроек
SELECT name, setting, unit, context
FROM pg_settings
WHERE name IN (
    'shared_buffers', 'work_mem', 'maintenance_work_mem',
    'effective_cache_size', 'max_connections',
    'max_parallel_workers_per_gather', 'random_page_cost'
)
ORDER BY name;
```
