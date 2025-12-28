# Golden Signals для мониторинга PostgreSQL

[prev: 02-red-method](./02-red-method.md) | [next: 01-schema-design-patterns](../21-sql-optimization/01-schema-design-patterns.md)

---

## Что такое Golden Signals

**Golden Signals** — методология мониторинга от Google SRE, описанная в книге "Site Reliability Engineering". Это четыре ключевых сигнала для мониторинга любой системы:

- **L**atency (Задержка) — время обработки запроса
- **T**raffic (Трафик) — объем нагрузки на систему
- **E**rrors (Ошибки) — частота неуспешных запросов
- **S**aturation (Насыщение) — насколько система близка к пределу

## Golden Signals vs USE vs RED

```
┌─────────────────────────────────────────────────────────────────┐
│                   Сравнение методологий                          │
├─────────────────────────────────────────────────────────────────┤
│  Метрика        │ USE      │ RED      │ Golden Signals          │
│  ────────────── │ ──────── │ ──────── │ ──────────────          │
│  Latency        │    -     │ Duration │      ✓                  │
│  Traffic        │    -     │ Rate     │      ✓                  │
│  Errors         │ Errors   │ Errors   │      ✓                  │
│  Saturation     │ Saturation│   -     │      ✓                  │
│  Utilization    │    ✓     │    -     │      -                  │
├─────────────────────────────────────────────────────────────────┤
│  Фокус          │ Ресурсы  │ Сервисы  │ Пользовательский опыт   │
│  Источник       │ B. Gregg │ T. Wilkie│ Google SRE              │
└─────────────────────────────────────────────────────────────────┘
```

Golden Signals объединяет лучшее из USE и RED, добавляя фокус на пользовательском опыте.

## Применение Golden Signals к PostgreSQL

### 1. Latency (Задержка)

Latency — время от получения запроса до отправки ответа. **Важно разделять успешные и неуспешные запросы**.

#### Latency успешных запросов
```sql
-- Средняя задержка по запросам (pg_stat_statements)
SELECT
    substring(query, 1, 60) as query,
    calls,
    round((total_exec_time / calls)::numeric, 3) as avg_latency_ms,
    round(min_exec_time::numeric, 3) as min_latency_ms,
    round(max_exec_time::numeric, 3) as max_latency_ms,
    round(stddev_exec_time::numeric, 3) as stddev_ms
FROM pg_stat_statements
WHERE calls > 100
ORDER BY avg_latency_ms DESC
LIMIT 15;
```

#### Распределение Latency (перцентили)
```sql
-- Приблизительные перцентили на основе статистики
SELECT
    query,
    calls,
    round((total_exec_time / calls)::numeric, 2) as p50_approx_ms,
    round(((total_exec_time / calls) + stddev_exec_time)::numeric, 2) as p84_approx_ms,
    round(((total_exec_time / calls) + 2 * stddev_exec_time)::numeric, 2) as p95_approx_ms,
    round(max_exec_time::numeric, 2) as p100_ms
FROM pg_stat_statements
WHERE calls > 1000 AND stddev_exec_time > 0
ORDER BY total_exec_time DESC
LIMIT 10;
```

#### Latency в реальном времени
```sql
-- Текущие запросы и их длительность
SELECT
    pid,
    usename,
    extract(milliseconds from (now() - query_start)) as latency_ms,
    state,
    left(query, 80) as query
FROM pg_stat_activity
WHERE state = 'active'
  AND query NOT LIKE '%pg_stat_activity%'
ORDER BY latency_ms DESC;
```

#### Latency соединений
```sql
-- Время установки соединения (из приложения или pgbouncer)
-- PostgreSQL не отслеживает это напрямую

-- Но можно замерить через backend_start
SELECT
    pid,
    usename,
    application_name,
    extract(epoch from (query_start - backend_start)) as time_to_first_query_sec
FROM pg_stat_activity
WHERE query_start IS NOT NULL
  AND backend_start IS NOT NULL
ORDER BY time_to_first_query_sec DESC
LIMIT 10;
```

### 2. Traffic (Трафик)

Traffic — измерение объема работы системы. Для базы данных это запросы, транзакции, переданные данные.

#### QPS (Queries Per Second)
```sql
-- Запросов в секунду
SELECT
    round(sum(calls)::numeric /
          extract(epoch from (now() - stats_reset)), 2) as qps
FROM pg_stat_statements, pg_stat_database
WHERE pg_stat_database.datname = current_database()
LIMIT 1;
```

#### TPS (Transactions Per Second)
```sql
-- Транзакций в секунду
SELECT
    datname,
    round((xact_commit + xact_rollback)::numeric /
          extract(epoch from (now() - stats_reset)), 2) as tps,
    round(xact_commit::numeric /
          extract(epoch from (now() - stats_reset)), 2) as commits_per_sec,
    round(xact_rollback::numeric /
          extract(epoch from (now() - stats_reset)), 2) as rollbacks_per_sec
FROM pg_stat_database
WHERE datname = current_database();
```

#### Traffic по типам операций
```sql
-- Разбивка по операциям
SELECT
    CASE
        WHEN query ILIKE 'SELECT%' THEN 'READ'
        WHEN query ILIKE 'INSERT%' OR query ILIKE 'UPDATE%' OR query ILIKE 'DELETE%' THEN 'WRITE'
        ELSE 'OTHER'
    END as operation_type,
    sum(calls) as total_calls,
    round(sum(calls) * 100.0 / (SELECT sum(calls) FROM pg_stat_statements), 2) as percent
FROM pg_stat_statements
GROUP BY 1
ORDER BY total_calls DESC;
```

#### Traffic данных (rows)
```sql
-- Количество обработанных строк по таблицам
SELECT
    schemaname || '.' || relname as table_name,
    seq_tup_read + idx_tup_fetch as rows_read,
    n_tup_ins as rows_inserted,
    n_tup_upd as rows_updated,
    n_tup_del as rows_deleted,
    (n_tup_ins + n_tup_upd + n_tup_del) as rows_modified
FROM pg_stat_user_tables
ORDER BY (seq_tup_read + idx_tup_fetch) DESC
LIMIT 15;
```

#### Traffic соединений
```sql
-- Динамика соединений
SELECT
    state,
    count(*) as count,
    round(count(*) * 100.0 / (SELECT count(*) FROM pg_stat_activity), 2) as percent
FROM pg_stat_activity
GROUP BY state
ORDER BY count DESC;
```

### 3. Errors (Ошибки)

Errors — любые неуспешные запросы. Google SRE рекомендует отслеживать как явные (HTTP 500), так и неявные (некорректные результаты).

#### Rollback Rate
```sql
-- Процент откатов (основной индикатор ошибок в PostgreSQL)
SELECT
    datname,
    xact_commit,
    xact_rollback,
    round(xact_rollback * 100.0 / nullif(xact_commit + xact_rollback, 0), 4) as error_rate_percent,
    CASE
        WHEN xact_rollback * 100.0 / nullif(xact_commit + xact_rollback, 0) > 5 THEN 'CRITICAL'
        WHEN xact_rollback * 100.0 / nullif(xact_commit + xact_rollback, 0) > 1 THEN 'WARNING'
        ELSE 'OK'
    END as status
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1', 'postgres');
```

#### Deadlocks
```sql
-- Deadlock — серьезная ошибка
SELECT
    datname,
    deadlocks,
    CASE
        WHEN deadlocks > 100 THEN 'CRITICAL: High deadlock count'
        WHEN deadlocks > 10 THEN 'WARNING: Deadlocks detected'
        ELSE 'OK'
    END as status
FROM pg_stat_database
WHERE datname = current_database();
```

#### Конфликты (для реплик)
```sql
-- Ошибки репликации
SELECT
    datname,
    confl_tablespace + confl_lock + confl_snapshot + confl_bufferpin + confl_deadlock as total_conflicts,
    confl_tablespace,
    confl_lock,
    confl_snapshot,
    confl_bufferpin,
    confl_deadlock
FROM pg_stat_database_conflicts
WHERE datname = current_database();
```

#### Ошибки constraint violations
```sql
-- Включить логирование всех ошибок
-- postgresql.conf: log_min_error_statement = 'error'

-- Анализ через pgBadger или grep логов:
-- grep "ERROR" /var/log/postgresql/postgresql-*.log | wc -l
```

#### Ошибки соединений
```sql
-- Проверка близости к лимиту (потенциальные отказы)
SELECT
    count(*) as current,
    (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') as max,
    (SELECT setting::int FROM pg_settings WHERE name = 'superuser_reserved_connections') as reserved,
    CASE
        WHEN count(*) >= (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') -
             (SELECT setting::int FROM pg_settings WHERE name = 'superuser_reserved_connections')
        THEN 'ERROR: Connection limit reached'
        WHEN count(*) >= (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') * 0.9
        THEN 'WARNING: Near connection limit'
        ELSE 'OK'
    END as status
FROM pg_stat_activity;
```

### 4. Saturation (Насыщение)

Saturation показывает, насколько система близка к своему пределу. Это **предсказатель будущих проблем**.

#### Connection Pool Saturation
```sql
-- Насыщенность пула соединений
SELECT
    count(*) as used_connections,
    (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') as max_connections,
    round(count(*) * 100.0 /
          (SELECT setting::int FROM pg_settings WHERE name = 'max_connections'), 2) as saturation_percent,
    CASE
        WHEN count(*) * 100.0 / (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') > 90 THEN 'CRITICAL'
        WHEN count(*) * 100.0 / (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') > 70 THEN 'WARNING'
        ELSE 'OK'
    END as status
FROM pg_stat_activity;
```

#### Memory Saturation (Buffer Cache)
```sql
-- Насыщенность буферного кеша
SELECT
    sum(heap_blks_hit) as cache_hits,
    sum(heap_blks_read) as disk_reads,
    round(sum(heap_blks_hit) * 100.0 /
          nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) as cache_hit_ratio,
    CASE
        WHEN sum(heap_blks_hit) * 100.0 / nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) < 95 THEN 'WARNING: Memory saturated'
        WHEN sum(heap_blks_hit) * 100.0 / nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) < 99 THEN 'INFO: Consider more memory'
        ELSE 'OK'
    END as status
FROM pg_statio_user_tables;
```

#### Lock Saturation
```sql
-- Очередь блокировок
SELECT
    count(*) as waiting_for_locks,
    CASE
        WHEN count(*) > 50 THEN 'CRITICAL: Lock queue saturated'
        WHEN count(*) > 10 THEN 'WARNING: Lock contention'
        ELSE 'OK'
    END as status
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

#### WAL Saturation
```sql
-- Состояние WAL
SELECT
    pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) as replication_lag_bytes,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn)) as lag_pretty
FROM pg_stat_replication;

-- Checkpoint pressure
SELECT
    checkpoints_timed,
    checkpoints_req,
    round(checkpoints_req * 100.0 / nullif(checkpoints_timed + checkpoints_req, 0), 2) as forced_checkpoint_percent,
    CASE
        WHEN checkpoints_req * 100.0 / nullif(checkpoints_timed + checkpoints_req, 0) > 50 THEN 'WARNING: Too many forced checkpoints'
        ELSE 'OK'
    END as status
FROM pg_stat_bgwriter;
```

#### Disk I/O Saturation
```sql
-- Временные файлы (признак нехватки work_mem)
SELECT
    datname,
    temp_files,
    pg_size_pretty(temp_bytes) as temp_bytes_used,
    CASE
        WHEN temp_files > 1000 THEN 'WARNING: High temp file usage'
        ELSE 'OK'
    END as status
FROM pg_stat_database
WHERE datname = current_database();
```

## Golden Signals Dashboard

```sql
-- Единый запрос всех Golden Signals
WITH golden_signals AS (
    SELECT
        -- LATENCY
        (SELECT round(avg(total_exec_time / nullif(calls, 0))::numeric, 2)
         FROM pg_stat_statements) as avg_latency_ms,

        (SELECT round(max(total_exec_time / nullif(calls, 0))::numeric, 2)
         FROM pg_stat_statements WHERE calls > 100) as max_latency_ms,

        -- TRAFFIC
        (SELECT round((xact_commit + xact_rollback)::numeric /
                extract(epoch from (now() - stats_reset)), 2)
         FROM pg_stat_database WHERE datname = current_database()) as tps,

        (SELECT count(*) FROM pg_stat_activity WHERE state = 'active') as active_queries,

        -- ERRORS
        (SELECT round(xact_rollback * 100.0 / nullif(xact_commit + xact_rollback, 0), 4)
         FROM pg_stat_database WHERE datname = current_database()) as error_rate_percent,

        (SELECT deadlocks FROM pg_stat_database WHERE datname = current_database()) as deadlocks,

        -- SATURATION
        (SELECT round(count(*) * 100.0 /
                (SELECT setting::int FROM pg_settings WHERE name = 'max_connections'), 2)
         FROM pg_stat_activity) as connection_saturation_percent,

        (SELECT round(sum(heap_blks_hit) * 100.0 /
                nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2)
         FROM pg_statio_user_tables) as cache_hit_ratio
)
SELECT
    '=== LATENCY ===' as signal,
    avg_latency_ms || ' ms (avg), ' || max_latency_ms || ' ms (max)' as value
FROM golden_signals
UNION ALL
SELECT '=== TRAFFIC ===', tps || ' TPS, ' || active_queries || ' active' FROM golden_signals
UNION ALL
SELECT '=== ERRORS ===', error_rate_percent || '% error rate, ' || deadlocks || ' deadlocks' FROM golden_signals
UNION ALL
SELECT '=== SATURATION ===', connection_saturation_percent || '% connections, ' || cache_hit_ratio || '% cache hit' FROM golden_signals;
```

## SLI/SLO на основе Golden Signals

```yaml
# Пример SLO для PostgreSQL
SLOs:
  availability:
    target: 99.9%
    metric: error_rate < 0.1%

  latency:
    target: 99%
    metric: p99_latency < 100ms

  saturation:
    connection_pool:
      warning: 70%
      critical: 90%
    cache_hit_ratio:
      warning: 95%
      critical: 90%
```

## Алерты Golden Signals

```yaml
# Prometheus alerting rules
groups:
  - name: postgresql_golden_signals
    rules:
      # Latency
      - alert: HighLatency
        expr: pg_stat_statements_mean_time_seconds > 0.5
        for: 5m
        labels:
          severity: warning

      # Traffic anomaly
      - alert: TrafficDrop
        expr: rate(pg_stat_database_xact_commit[5m]) < 10
        for: 10m
        labels:
          severity: critical

      # Errors
      - alert: HighErrorRate
        expr: rate(pg_stat_database_xact_rollback[5m]) / rate(pg_stat_database_xact_commit[5m]) > 0.01
        for: 5m
        labels:
          severity: critical

      # Saturation
      - alert: ConnectionSaturation
        expr: pg_stat_activity_count / pg_settings_max_connections > 0.8
        for: 5m
        labels:
          severity: warning
```

## Когда использовать Golden Signals

| Сценарий | Рекомендация |
|----------|--------------|
| SRE практики, SLA/SLO | Golden Signals |
| Быстрая диагностика проблем инфраструктуры | USE Method |
| Мониторинг микросервисов | RED Method |
| Комплексный мониторинг базы данных | Golden Signals |
| Capacity planning | USE + Golden Signals |

## Best Practices

1. **Latency**: Всегда отслеживать перцентили (P50, P95, P99), не только среднее
2. **Traffic**: Установить baseline и детектировать аномалии в обе стороны
3. **Errors**: Любой error rate > 0 требует расследования
4. **Saturation**: Алертить заранее (на 70-80%), не на 100%

## Инструменты

- **pg_stat_statements** — основа для Latency и Traffic
- **pg_stat_database** — Errors и Traffic
- **pg_stat_activity** — Saturation и реальное время
- **Prometheus + postgres_exporter** — автоматизация
- **Grafana** — визуализация Golden Signals dashboard

## Ссылки

- [Google SRE Book - Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [The Four Golden Signals](https://sre.google/sre-book/monitoring-distributed-systems/#xref_monitoring_golden-signals)
- [PostgreSQL Monitoring Documentation](https://www.postgresql.org/docs/current/monitoring.html)

---

[prev: 02-red-method](./02-red-method.md) | [next: 01-schema-design-patterns](../21-sql-optimization/01-schema-design-patterns.md)
