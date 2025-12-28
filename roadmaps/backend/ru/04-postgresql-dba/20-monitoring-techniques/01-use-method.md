# USE Method для мониторинга PostgreSQL

[prev: 04-explain-dalibo](../19-query-analysis/04-explain-dalibo.md) | [next: 02-red-method](./02-red-method.md)

---

## Что такое USE Method

**USE Method** — методология мониторинга, разработанная Бренданом Греггом (Brendan Gregg) для анализа производительности систем. Название — аббревиатура:

- **U**tilization (Использование) — процент времени, когда ресурс занят работой
- **S**aturation (Насыщение) — степень перегрузки, очередь работы, которую ресурс не может обработать
- **E**rrors (Ошибки) — количество ошибок при работе с ресурсом

## Принцип работы

USE Method фокусируется на **ресурсах системы** (CPU, память, диск, сеть), а не на сервисах. Для каждого ресурса проверяются три метрики, что позволяет быстро локализовать узкое место.

```
┌─────────────────────────────────────────────────────────┐
│                    USE Method                            │
├─────────────────────────────────────────────────────────┤
│  Ресурс → Utilization → Saturation → Errors             │
│                                                          │
│  CPU        ✓              ✓            ✓               │
│  Memory     ✓              ✓            ✓               │
│  Disk I/O   ✓              ✓            ✓               │
│  Network    ✓              ✓            ✓               │
└─────────────────────────────────────────────────────────┘
```

## Применение USE к PostgreSQL

### 1. CPU (Процессор)

#### Utilization
```sql
-- Проверка активных процессов PostgreSQL
SELECT count(*) as active_backends,
       state,
       wait_event_type
FROM pg_stat_activity
WHERE state = 'active'
GROUP BY state, wait_event_type;
```

**Системные метрики:**
```bash
# Использование CPU процессами PostgreSQL
top -p $(pgrep -d',' postgres)

# Или через /proc
cat /proc/stat | grep cpu
```

#### Saturation
```sql
-- Очередь ожидающих запросов
SELECT count(*) as waiting_queries
FROM pg_stat_activity
WHERE wait_event_type IS NOT NULL
  AND state = 'active';

-- Количество заблокированных процессов
SELECT count(*) as blocked
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

#### Errors
```sql
-- Ошибки в логах (требует pg_stat_statements)
SELECT calls,
       total_exec_time,
       query
FROM pg_stat_statements
WHERE query LIKE '%ERROR%'
ORDER BY calls DESC
LIMIT 10;
```

### 2. Memory (Память)

#### Utilization
```sql
-- Использование shared_buffers
SELECT
    pg_size_pretty(pg_database_size(current_database())) as db_size,
    current_setting('shared_buffers') as shared_buffers;

-- Статистика буферного кеша
SELECT
    c.relname,
    pg_size_pretty(count(*) * 8192) as buffered,
    round(100.0 * count(*) / (SELECT setting FROM pg_settings WHERE name = 'shared_buffers')::integer, 2) as percent_of_buffer
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
GROUP BY c.relname
ORDER BY count(*) DESC
LIMIT 10;
```

#### Saturation
```sql
-- Эффективность кеша (cache hit ratio)
SELECT
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit) as heap_hit,
    round(sum(heap_blks_hit) * 100.0 /
          nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) as cache_hit_ratio
FROM pg_statio_user_tables;

-- Если cache_hit_ratio < 99%, память насыщена
```

#### Errors
```sql
-- OOM события и проблемы с памятью смотрим в логах
-- В postgresql.conf: log_min_messages = 'warning'
```

**Системная проверка:**
```bash
# Использование памяти PostgreSQL
ps aux | grep postgres | awk '{sum+=$6} END {print sum/1024 " MB"}'

# Проверка swap (если используется - память насыщена)
free -m | grep Swap
```

### 3. Disk I/O (Дисковая подсистема)

#### Utilization
```sql
-- Статистика чтения/записи таблиц
SELECT
    schemaname,
    relname,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    n_tup_ins,
    n_tup_upd,
    n_tup_del
FROM pg_stat_user_tables
ORDER BY seq_tup_read DESC
LIMIT 10;

-- Статистика I/O по таблицам
SELECT
    relname,
    heap_blks_read,
    heap_blks_hit,
    idx_blks_read,
    idx_blks_hit
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC
LIMIT 10;
```

#### Saturation
```sql
-- Очередь записи WAL
SELECT
    pg_current_wal_lsn(),
    pg_walfile_name(pg_current_wal_lsn()),
    pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') as wal_bytes_written;

-- Checkpoint статистика
SELECT * FROM pg_stat_bgwriter;
```

**Системные метрики:**
```bash
# Загрузка дисков
iostat -x 1

# Очередь I/O (avgqu-sz > 1 означает насыщение)
```

#### Errors
```sql
-- Проверка целостности данных
SELECT * FROM pg_stat_database
WHERE datname = current_database();

-- Ошибки checksum (если включены)
SELECT datname, checksum_failures, checksum_last_failure
FROM pg_stat_database
WHERE checksum_failures > 0;
```

### 4. Network (Сеть)

#### Utilization
```sql
-- Количество соединений
SELECT
    count(*) as total_connections,
    max_conn.setting::int as max_connections,
    round(count(*) * 100.0 / max_conn.setting::int, 2) as percent_used
FROM pg_stat_activity,
     (SELECT setting FROM pg_settings WHERE name = 'max_connections') max_conn
GROUP BY max_conn.setting;
```

#### Saturation
```sql
-- Ожидание соединений (если используется pgbouncer)
-- Отклоненные соединения
SELECT count(*) FROM pg_stat_activity WHERE state = 'idle';

-- Слишком много idle соединений = насыщение connection pool
```

#### Errors
```sql
-- Прерванные соединения в логах
-- log_disconnections = on в postgresql.conf
```

## Практический чеклист USE для PostgreSQL

```markdown
## USE Checklist

### CPU
- [ ] Utilization: pg_stat_activity active count, top/htop
- [ ] Saturation: wait_event_type, run queue
- [ ] Errors: log errors, query failures

### Memory
- [ ] Utilization: shared_buffers usage, pg_buffercache
- [ ] Saturation: cache hit ratio < 99%, swap usage
- [ ] Errors: OOM in logs, memory allocation failures

### Disk
- [ ] Utilization: pg_statio_*, iostat %util
- [ ] Saturation: avgqu-sz, checkpoint warnings
- [ ] Errors: checksum failures, I/O errors in dmesg

### Network
- [ ] Utilization: connection count vs max_connections
- [ ] Saturation: connection queue, idle connections
- [ ] Errors: connection refused, timeouts in logs
```

## Автоматизация сбора USE метрик

```sql
-- Единый запрос для USE метрик PostgreSQL
WITH use_metrics AS (
    SELECT
        -- CPU Utilization
        (SELECT count(*) FROM pg_stat_activity WHERE state = 'active') as cpu_util,
        -- CPU Saturation
        (SELECT count(*) FROM pg_stat_activity WHERE wait_event_type = 'Lock') as cpu_sat,
        -- Memory Utilization (cache hit ratio)
        (SELECT round(sum(heap_blks_hit)*100.0/nullif(sum(heap_blks_hit)+sum(heap_blks_read),0),2)
         FROM pg_statio_user_tables) as mem_util,
        -- Connection Utilization
        (SELECT round(count(*)*100.0/(SELECT setting::int FROM pg_settings WHERE name='max_connections'),2)
         FROM pg_stat_activity) as conn_util
)
SELECT * FROM use_metrics;
```

## Когда использовать USE Method

| Сценарий | USE подходит? |
|----------|---------------|
| Диагностика проблем производительности | Да |
| Capacity planning | Да |
| Поиск узких мест в инфраструктуре | Да |
| Мониторинг SLA сервиса | Нет (используй RED) |
| Бизнес-метрики | Нет |

## Инструменты для сбора USE метрик

- **pg_stat_statements** — статистика запросов
- **pg_buffercache** — состояние буферного кеша
- **pg_stat_activity** — текущие сессии
- **pg_stat_bgwriter** — статистика фоновых процессов
- **Prometheus + postgres_exporter** — автоматический сбор метрик
- **Grafana** — визуализация

## Ссылки

- [Brendan Gregg - USE Method](https://www.brendangregg.com/usemethod.html)
- [PostgreSQL Statistics Collector](https://www.postgresql.org/docs/current/monitoring-stats.html)

---

[prev: 04-explain-dalibo](../19-query-analysis/04-explain-dalibo.md) | [next: 02-red-method](./02-red-method.md)
