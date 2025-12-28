# RED Method для мониторинга PostgreSQL

[prev: 01-use-method](./01-use-method.md) | [next: 03-golden-signals](./03-golden-signals.md)

---

## Что такое RED Method

**RED Method** — методология мониторинга, разработанная Томом Уилки (Tom Wilkie) из Weaveworks. Ориентирована на мониторинг **сервисов**, а не ресурсов. Аббревиатура:

- **R**ate (Скорость) — количество запросов в секунду
- **E**rrors (Ошибки) — количество неуспешных запросов
- **D**uration (Длительность) — время выполнения запросов

## Отличие от USE Method

```
┌────────────────────────────────────────────────────────────┐
│                USE vs RED                                   │
├────────────────────────────────────────────────────────────┤
│  USE Method          │  RED Method                         │
│  ─────────────────── │ ──────────────────                  │
│  Фокус: ресурсы      │  Фокус: сервисы                     │
│  (CPU, RAM, Disk)    │  (API, база данных)                 │
│                      │                                      │
│  Вопрос: "Что        │  Вопрос: "Как сервис               │
│  перегружено?"       │  обслуживает пользователей?"        │
└────────────────────────────────────────────────────────────┘
```

## Применение RED к PostgreSQL

PostgreSQL можно рассматривать как сервис, обрабатывающий запросы. RED идеально подходит для мониторинга с точки зрения приложения.

### 1. Rate (Скорость запросов)

#### Общий Rate
```sql
-- Количество запросов в секунду (требует pg_stat_statements)
SELECT
    sum(calls) as total_calls,
    sum(calls) / extract(epoch from (now() - stats_reset)) as queries_per_second
FROM pg_stat_statements, pg_stat_database
WHERE pg_stat_database.datname = current_database()
LIMIT 1;
```

#### Rate по типам запросов
```sql
-- Разбивка по типам операций
SELECT
    CASE
        WHEN query ILIKE 'SELECT%' THEN 'SELECT'
        WHEN query ILIKE 'INSERT%' THEN 'INSERT'
        WHEN query ILIKE 'UPDATE%' THEN 'UPDATE'
        WHEN query ILIKE 'DELETE%' THEN 'DELETE'
        ELSE 'OTHER'
    END as query_type,
    sum(calls) as total_calls,
    round(sum(calls) * 100.0 / (SELECT sum(calls) FROM pg_stat_statements), 2) as percentage
FROM pg_stat_statements
GROUP BY 1
ORDER BY total_calls DESC;
```

#### Rate транзакций
```sql
-- Транзакции в секунду
SELECT
    datname,
    xact_commit + xact_rollback as total_transactions,
    xact_commit as commits,
    xact_rollback as rollbacks,
    round((xact_commit + xact_rollback) /
          extract(epoch from (now() - stats_reset)), 2) as tps
FROM pg_stat_database
WHERE datname = current_database();
```

#### Мониторинг Rate в реальном времени
```sql
-- Снимок для расчета дельты (запускать периодически)
CREATE TABLE IF NOT EXISTS rate_snapshots (
    snapshot_time timestamp DEFAULT now(),
    total_calls bigint,
    total_xact bigint
);

-- Вставка снимка
INSERT INTO rate_snapshots (total_calls, total_xact)
SELECT
    (SELECT sum(calls) FROM pg_stat_statements),
    (SELECT xact_commit + xact_rollback FROM pg_stat_database WHERE datname = current_database());

-- Расчет Rate между снимками
SELECT
    s2.snapshot_time,
    (s2.total_calls - s1.total_calls) /
        extract(epoch from (s2.snapshot_time - s1.snapshot_time)) as qps,
    (s2.total_xact - s1.total_xact) /
        extract(epoch from (s2.snapshot_time - s1.snapshot_time)) as tps
FROM rate_snapshots s1
JOIN rate_snapshots s2 ON s2.snapshot_time > s1.snapshot_time
ORDER BY s2.snapshot_time DESC
LIMIT 10;
```

### 2. Errors (Ошибки)

#### Error Rate из статистики
```sql
-- Процент откатов транзакций (индикатор ошибок)
SELECT
    datname,
    xact_commit,
    xact_rollback,
    round(xact_rollback * 100.0 / nullif(xact_commit + xact_rollback, 0), 2) as rollback_percentage
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1');
```

#### Ошибки deadlock
```sql
-- Количество deadlock'ов
SELECT
    datname,
    deadlocks,
    conflicts
FROM pg_stat_database
WHERE datname = current_database();
```

#### Ошибки в запросах (из логов)
```sql
-- Настройка логирования ошибок в postgresql.conf:
-- log_min_error_statement = 'error'
-- log_min_messages = 'warning'

-- Парсинг логов через pgBadger или вручную
```

#### Мониторинг конфликтов репликации
```sql
-- Конфликты на replica
SELECT
    datname,
    confl_tablespace,
    confl_lock,
    confl_snapshot,
    confl_bufferpin,
    confl_deadlock
FROM pg_stat_database_conflicts
WHERE datname = current_database();
```

#### Ошибки соединений
```sql
-- Отклоненные соединения (косвенно)
SELECT
    count(*) as current_connections,
    (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') as max_conn,
    CASE
        WHEN count(*) >= (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') - 5
        THEN 'WARNING: Near limit, connections may be rejected'
        ELSE 'OK'
    END as status
FROM pg_stat_activity;
```

### 3. Duration (Длительность)

#### Средняя длительность запросов
```sql
-- Средняя и общая длительность по запросам
SELECT
    substring(query, 1, 50) as query_preview,
    calls,
    round(total_exec_time::numeric, 2) as total_time_ms,
    round((total_exec_time / calls)::numeric, 2) as avg_time_ms,
    round(min_exec_time::numeric, 2) as min_ms,
    round(max_exec_time::numeric, 2) as max_ms,
    round(stddev_exec_time::numeric, 2) as stddev_ms
FROM pg_stat_statements
WHERE calls > 100
ORDER BY total_exec_time DESC
LIMIT 20;
```

#### Перцентили длительности
```sql
-- Для точных перцентилей нужно логировать каждый запрос
-- или использовать pg_stat_statements с планированием

-- Приблизительная оценка через стандартное отклонение
SELECT
    query,
    calls,
    round((total_exec_time / calls)::numeric, 2) as avg_ms,
    round(stddev_exec_time::numeric, 2) as stddev_ms,
    -- Приблизительный P95 (avg + 2*stddev для нормального распределения)
    round(((total_exec_time / calls) + 2 * stddev_exec_time)::numeric, 2) as approx_p95_ms
FROM pg_stat_statements
WHERE calls > 1000
ORDER BY total_exec_time DESC
LIMIT 10;
```

#### Медленные запросы
```sql
-- Включить логирование медленных запросов
-- postgresql.conf: log_min_duration_statement = 1000  -- 1 секунда

-- Текущие долгие запросы
SELECT
    pid,
    now() - pg_stat_activity.query_start AS duration,
    query,
    state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 seconds'
  AND state != 'idle'
ORDER BY duration DESC;
```

#### Duration по типам запросов
```sql
SELECT
    CASE
        WHEN query ILIKE 'SELECT%' THEN 'SELECT'
        WHEN query ILIKE 'INSERT%' THEN 'INSERT'
        WHEN query ILIKE 'UPDATE%' THEN 'UPDATE'
        WHEN query ILIKE 'DELETE%' THEN 'DELETE'
        ELSE 'OTHER'
    END as query_type,
    count(*) as query_count,
    round(avg(total_exec_time / calls)::numeric, 2) as avg_duration_ms,
    round(sum(total_exec_time)::numeric, 2) as total_time_ms
FROM pg_stat_statements
WHERE calls > 0
GROUP BY 1
ORDER BY total_time_ms DESC;
```

## RED Dashboard (единый запрос)

```sql
-- Сводка RED метрик
WITH red_metrics AS (
    SELECT
        -- Rate
        (SELECT round(sum(calls) / extract(epoch from (now() - min(stats_reset))), 2)
         FROM pg_stat_statements, pg_stat_database
         WHERE pg_stat_database.datname = current_database()) as queries_per_second,

        (SELECT round((xact_commit + xact_rollback) / extract(epoch from (now() - stats_reset)), 2)
         FROM pg_stat_database
         WHERE datname = current_database()) as transactions_per_second,

        -- Errors
        (SELECT round(xact_rollback * 100.0 / nullif(xact_commit + xact_rollback, 0), 2)
         FROM pg_stat_database
         WHERE datname = current_database()) as error_rate_percent,

        (SELECT deadlocks FROM pg_stat_database WHERE datname = current_database()) as total_deadlocks,

        -- Duration
        (SELECT round(avg(total_exec_time / nullif(calls, 0))::numeric, 2)
         FROM pg_stat_statements) as avg_query_duration_ms,

        (SELECT round(max(total_exec_time / nullif(calls, 0))::numeric, 2)
         FROM pg_stat_statements
         WHERE calls > 100) as max_avg_query_duration_ms
)
SELECT * FROM red_metrics;
```

## Алерты на основе RED

```yaml
# Пример правил для Prometheus/Alertmanager
groups:
  - name: postgresql_red
    rules:
      # Rate drop alert
      - alert: PostgreSQLLowQPS
        expr: rate(pg_stat_statements_calls_total[5m]) < 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low query rate on PostgreSQL"

      # Error rate alert
      - alert: PostgreSQLHighErrorRate
        expr: rate(pg_stat_database_xact_rollback[5m]) / rate(pg_stat_database_xact_commit[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High transaction rollback rate"

      # Duration alert
      - alert: PostgreSQLSlowQueries
        expr: pg_stat_statements_mean_time_seconds > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow queries detected"
```

## Когда использовать RED Method

| Сценарий | RED подходит? |
|----------|---------------|
| Мониторинг SLA/SLO | Да |
| Отслеживание качества сервиса | Да |
| Capacity planning | Частично |
| Поиск узких мест в инфраструктуре | Нет (используй USE) |
| Бизнес-метрики производительности | Да |
| Микросервисная архитектура | Да |

## Инструменты для RED мониторинга PostgreSQL

- **pg_stat_statements** — основа для Rate и Duration
- **pg_stat_database** — транзакции и ошибки
- **pgBadger** — анализ логов для Error и Duration
- **Prometheus + postgres_exporter** — автоматический сбор
- **Grafana** — визуализация RED дашбордов

## Best Practices

1. **Rate**: Установить baseline нормального QPS и отслеживать аномалии
2. **Errors**: Стремиться к error rate < 1%, алертить на > 5%
3. **Duration**: Отслеживать P50, P95, P99 для понимания распределения
4. **Сбрасывать статистику периодически**: `SELECT pg_stat_statements_reset()`

## Ссылки

- [Tom Wilkie - RED Method](https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/)
- [PostgreSQL pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)

---

[prev: 01-use-method](./01-use-method.md) | [next: 03-golden-signals](./03-golden-signals.md)
