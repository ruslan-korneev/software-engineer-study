# Инструменты мониторинга PostgreSQL

[prev: 04-helm-operators](./04-helm-operators.md) | [next: 01-pgbouncer](../11-connection-pooling/01-pgbouncer.md)

---

## Введение

Мониторинг PostgreSQL критически важен для обеспечения производительности, доступности и своевременного обнаружения проблем. Эффективная система мониторинга должна отслеживать метрики на уровне ОС, самой базы данных и приложений.

## Уровни мониторинга

```
┌─────────────────────────────────────────────────────┐
│                   Уровень приложения                │
│         (время запросов, ошибки, throughput)        │
├─────────────────────────────────────────────────────┤
│                   Уровень базы данных               │
│    (соединения, блокировки, репликация, кэш)        │
├─────────────────────────────────────────────────────┤
│                   Уровень ОС                        │
│        (CPU, память, диск I/O, сеть)                │
└─────────────────────────────────────────────────────┘
```

## Встроенные средства PostgreSQL

### Системные представления (System Views)

#### pg_stat_activity — активные сессии

```sql
-- Все активные соединения
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    query_start,
    NOW() - query_start AS query_duration,
    LEFT(query, 100) AS query_preview
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

-- Количество соединений по состояниям
SELECT state, COUNT(*)
FROM pg_stat_activity
GROUP BY state;

-- Долгие транзакции
SELECT
    pid,
    usename,
    NOW() - xact_start AS transaction_duration,
    state,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND NOW() - xact_start > INTERVAL '5 minutes';
```

#### pg_stat_database — статистика базы данных

```sql
-- Общая статистика по базам данных
SELECT
    datname,
    numbackends AS connections,
    xact_commit AS commits,
    xact_rollback AS rollbacks,
    blks_read AS disk_reads,
    blks_hit AS cache_hits,
    ROUND(blks_hit::numeric / NULLIF(blks_hit + blks_read, 0) * 100, 2) AS cache_hit_ratio,
    tup_returned AS rows_returned,
    tup_fetched AS rows_fetched,
    tup_inserted AS rows_inserted,
    tup_updated AS rows_updated,
    tup_deleted AS rows_deleted,
    conflicts,
    temp_files,
    pg_size_pretty(temp_bytes) AS temp_bytes,
    deadlocks
FROM pg_stat_database
WHERE datname NOT LIKE 'template%';
```

#### pg_stat_user_tables — статистика таблиц

```sql
-- Статистика по таблицам
SELECT
    schemaname,
    relname AS table_name,
    seq_scan AS sequential_scans,
    seq_tup_read AS seq_rows_read,
    idx_scan AS index_scans,
    idx_tup_fetch AS index_rows_fetched,
    n_tup_ins AS rows_inserted,
    n_tup_upd AS rows_updated,
    n_tup_del AS rows_deleted,
    n_live_tup AS live_rows,
    n_dead_tup AS dead_rows,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;

-- Таблицы с большим количеством dead tuples
SELECT
    schemaname || '.' || relname AS table_name,
    n_dead_tup,
    n_live_tup,
    ROUND(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 2) AS dead_ratio,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

#### pg_stat_user_indexes — статистика индексов

```sql
-- Неиспользуемые индексы
SELECT
    schemaname || '.' || relname AS table_name,
    indexrelname AS index_name,
    idx_scan AS times_used,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Эффективность индексов
SELECT
    schemaname || '.' || relname AS table_name,
    indexrelname AS index_name,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    ROUND(idx_tup_fetch::numeric / NULLIF(idx_tup_read, 0) * 100, 2) AS fetch_ratio
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

### pg_stat_statements

#### Установка и настройка

```sql
-- Добавить в postgresql.conf
-- shared_preload_libraries = 'pg_stat_statements'

-- Создать расширение
CREATE EXTENSION pg_stat_statements;
```

#### Анализ запросов

```sql
-- Топ-10 запросов по времени выполнения
SELECT
    LEFT(query, 100) AS query_preview,
    calls,
    ROUND(total_exec_time::numeric, 2) AS total_time_ms,
    ROUND(mean_exec_time::numeric, 2) AS avg_time_ms,
    ROUND(stddev_exec_time::numeric, 2) AS stddev_ms,
    rows,
    ROUND(shared_blks_hit::numeric / NULLIF(shared_blks_hit + shared_blks_read, 0) * 100, 2) AS cache_hit_ratio
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Запросы с большим количеством чтений с диска
SELECT
    LEFT(query, 100) AS query_preview,
    calls,
    shared_blks_read AS disk_reads,
    shared_blks_hit AS cache_hits,
    ROUND(shared_blks_read::numeric / NULLIF(calls, 0), 2) AS disk_reads_per_call
FROM pg_stat_statements
WHERE shared_blks_read > 0
ORDER BY shared_blks_read DESC
LIMIT 10;

-- Сброс статистики
SELECT pg_stat_statements_reset();
```

### pg_stat_replication — репликация

```sql
-- Статус репликации
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
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS replication_lag,
    sync_state
FROM pg_stat_replication;

-- Отставание репликации в секундах
SELECT
    application_name,
    EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp())) AS lag_seconds
FROM pg_stat_replication;
```

## Prometheus и Grafana

### Архитектура

```
┌────────────┐     ┌────────────────┐     ┌───────────┐
│ PostgreSQL │────▶│ postgres_      │────▶│ Prometheus│
│            │     │ exporter       │     │           │
└────────────┘     └────────────────┘     └─────┬─────┘
                                                │
                                                ▼
                                          ┌───────────┐
                                          │  Grafana  │
                                          │  Dashboards│
                                          └───────────┘
```

### Установка postgres_exporter

```bash
# Скачать бинарник
wget https://github.com/prometheus-community/postgres_exporter/releases/download/v0.15.0/postgres_exporter-0.15.0.linux-amd64.tar.gz
tar xvf postgres_exporter-0.15.0.linux-amd64.tar.gz
cd postgres_exporter-0.15.0.linux-amd64

# Запуск
export DATA_SOURCE_NAME="postgresql://postgres:password@localhost:5432/postgres?sslmode=disable"
./postgres_exporter
```

### Docker Compose с мониторингом

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:latest
    environment:
      DATA_SOURCE_NAME: "postgresql://postgres:password@postgres:5432/postgres?sslmode=disable"
    ports:
      - "9187:9187"
    depends_on:
      - postgres

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'

  grafana:
    image: grafana/grafana:latest
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin

volumes:
  postgres_data:
  prometheus_data:
  grafana_data:
```

### Конфигурация Prometheus

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        replacement: 'postgres-primary'
```

### Кастомные метрики

```yaml
# queries.yaml для postgres_exporter
pg_stat_user_tables:
  query: |
    SELECT
      schemaname,
      relname,
      seq_scan,
      seq_tup_read,
      idx_scan,
      n_live_tup,
      n_dead_tup
    FROM pg_stat_user_tables
  metrics:
    - schemaname:
        usage: "LABEL"
        description: "Schema name"
    - relname:
        usage: "LABEL"
        description: "Table name"
    - seq_scan:
        usage: "COUNTER"
        description: "Sequential scans"
    - idx_scan:
        usage: "COUNTER"
        description: "Index scans"
    - n_live_tup:
        usage: "GAUGE"
        description: "Live tuples"
    - n_dead_tup:
        usage: "GAUGE"
        description: "Dead tuples"

pg_replication_lag:
  query: |
    SELECT
      application_name,
      EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp())) AS lag_seconds
    FROM pg_stat_replication
  metrics:
    - application_name:
        usage: "LABEL"
        description: "Replica name"
    - lag_seconds:
        usage: "GAUGE"
        description: "Replication lag in seconds"
```

### Grafana Dashboards

#### Рекомендуемые дашборды

| Dashboard ID | Название | Описание |
|--------------|----------|----------|
| 9628 | PostgreSQL Database | Общий обзор |
| 455 | PostgreSQL Overview | Детальные метрики |
| 12273 | PostgreSQL Exporter | Метрики от postgres_exporter |

#### Импорт дашборда

1. Grafana → Dashboards → Import
2. Ввести ID дашборда (например, 9628)
3. Выбрать источник данных Prometheus

### Основные метрики для алертинга

```yaml
# alerting_rules.yml
groups:
  - name: PostgreSQL
    rules:
      - alert: PostgreSQLDown
        expr: pg_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL is down"

      - alert: PostgreSQLHighConnections
        expr: pg_stat_activity_count > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High number of PostgreSQL connections"

      - alert: PostgreSQLReplicationLag
        expr: pg_replication_lag_seconds > 300
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL replication lag is high"

      - alert: PostgreSQLLowCacheHitRatio
        expr: pg_stat_database_blks_hit / (pg_stat_database_blks_hit + pg_stat_database_blks_read) < 0.95
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL cache hit ratio is low"

      - alert: PostgreSQLDeadlocks
        expr: increase(pg_stat_database_deadlocks[5m]) > 0
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL deadlocks detected"

      - alert: PostgreSQLSlowQueries
        expr: pg_stat_statements_mean_exec_time_seconds > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow queries detected"
```

## pgAdmin

### Установка через Docker

```yaml
version: '3.8'

services:
  pgadmin:
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    volumes:
      - pgadmin_data:/var/lib/pgadmin

volumes:
  pgadmin_data:
```

### Возможности мониторинга

- Dashboard с метриками сервера
- Просмотр активных сессий
- Анализ планов выполнения запросов
- Управление пользователями и правами
- Просмотр логов

## pgwatch2

### Описание

**pgwatch2** — это комплексное решение для мониторинга PostgreSQL, включающее:
- Сбор метрик
- Хранение в InfluxDB или Prometheus
- Готовые Grafana дашборды
- Веб-интерфейс для управления

### Установка через Docker

```bash
# Запуск с InfluxDB
docker run -d --name pw2 \
  -p 8080:8080 \
  -p 3000:3000 \
  -e PW2_TESTDB=true \
  cybertec/pgwatch2-postgres
```

### Добавление базы данных

```bash
# Через Web UI (port 8080)
# Или через API
curl -X POST http://localhost:8080/dbs \
  -H "Content-Type: application/json" \
  -d '{
    "md_unique_name": "my-postgres",
    "md_hostname": "postgres-host",
    "md_port": 5432,
    "md_dbname": "postgres",
    "md_user": "pgwatch2",
    "md_password": "password",
    "md_preset_config_name": "full"
  }'
```

## check_postgres (Nagios)

### Установка

```bash
# Debian/Ubuntu
apt-get install check-postgres

# Из исходников
git clone https://github.com/bucardo/check_postgres.git
cd check_postgres
perl Makefile.PL
make install
```

### Примеры проверок

```bash
# Проверка соединения
check_postgres.pl --action=connection --dbname=mydb

# Проверка количества соединений
check_postgres.pl --action=backends --dbname=mydb --warning=80 --critical=90

# Проверка размера базы данных
check_postgres.pl --action=database_size --dbname=mydb --warning=10GB --critical=20GB

# Проверка репликации
check_postgres.pl --action=hot_standby_delay --dbname=mydb --warning=5min --critical=15min

# Проверка блокировок
check_postgres.pl --action=locks --dbname=mydb --warning=10 --critical=20

# Проверка последнего vacuum
check_postgres.pl --action=last_vacuum --dbname=mydb --warning='7 days' --critical='30 days'

# Проверка bloat (раздутие таблиц)
check_postgres.pl --action=bloat --dbname=mydb --warning=50% --critical=75%
```

### Интеграция с Nagios/Icinga

```cfg
# commands.cfg
define command {
    command_name    check_postgres_backends
    command_line    /usr/local/bin/check_postgres.pl --action=backends --dbname=$ARG1$ --warning=$ARG2$ --critical=$ARG3$
}

# services.cfg
define service {
    use                     generic-service
    host_name               postgres-server
    service_description     PostgreSQL Backends
    check_command           check_postgres_backends!mydb!80!95
}
```

## Логирование

### Настройка логирования PostgreSQL

```ini
# postgresql.conf

# Куда писать логи
logging_collector = on
log_directory = 'pg_log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 100MB

# Что логировать
log_min_messages = warning
log_min_error_statement = error
log_min_duration_statement = 1000  # Логировать запросы дольше 1 секунды

# Детализация
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 0

# Для анализа производительности
log_statement = 'ddl'  # none, ddl, mod, all
log_duration = off
```

### pgBadger — анализ логов

```bash
# Установка
apt-get install pgbadger

# Или через cpan
cpan install pgBadger

# Генерация отчета
pgbadger /var/log/postgresql/postgresql-*.log -o report.html

# Инкрементальный анализ
pgbadger --incremental /var/log/postgresql/postgresql-*.log \
  -o /var/www/html/pgbadger/ \
  --outdir /var/www/html/pgbadger/

# С фильтрацией
pgbadger --begin '2024-01-01 00:00:00' \
  --end '2024-01-02 00:00:00' \
  -d mydb \
  /var/log/postgresql/*.log
```

## Best Practices

### Ключевые метрики для мониторинга

| Категория | Метрика | Пороговое значение |
|-----------|---------|-------------------|
| Соединения | backends / max_connections | < 80% |
| Кэш | cache hit ratio | > 99% |
| Репликация | lag_seconds | < 60 |
| Транзакции | commits + rollbacks | baseline |
| Блокировки | locks count | baseline |
| Deadlocks | deadlocks/min | 0 |
| Temp files | temp_bytes | baseline |
| Vacuum | dead tuples ratio | < 10% |

### Чек-лист настройки мониторинга

1. **Базовый уровень**
   - pg_stat_statements включен
   - Логирование медленных запросов
   - Алерт на недоступность

2. **Средний уровень**
   - Prometheus + Grafana
   - Мониторинг репликации
   - Алерты на ключевые метрики

3. **Продвинутый уровень**
   - Кастомные метрики
   - Анализ логов (pgBadger)
   - APM интеграция

### Автоматизация

```bash
#!/bin/bash
# health_check.sh

# Проверка доступности
if ! pg_isready -h localhost -p 5432 > /dev/null 2>&1; then
    echo "CRITICAL: PostgreSQL is not responding"
    exit 2
fi

# Проверка репликации (на реплике)
LAG=$(psql -t -c "SELECT EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp()))::int;")
if [ "$LAG" -gt 300 ]; then
    echo "WARNING: Replication lag is ${LAG}s"
    exit 1
fi

# Проверка соединений
CONN_USAGE=$(psql -t -c "SELECT ROUND(COUNT(*)::numeric / current_setting('max_connections')::numeric * 100) FROM pg_stat_activity;")
if [ "$CONN_USAGE" -gt 80 ]; then
    echo "WARNING: Connection usage is ${CONN_USAGE}%"
    exit 1
fi

echo "OK: PostgreSQL is healthy"
exit 0
```

## Полезные ссылки

- [PostgreSQL Statistics Collector](https://www.postgresql.org/docs/current/monitoring-stats.html)
- [postgres_exporter](https://github.com/prometheus-community/postgres_exporter)
- [pgwatch2](https://github.com/cybertec-postgresql/pgwatch2)
- [pgBadger](https://github.com/darold/pgbadger)
- [check_postgres](https://github.com/bucardo/check_postgres)
- [Grafana PostgreSQL Dashboards](https://grafana.com/grafana/dashboards/?search=postgresql)

---

[prev: 04-helm-operators](./04-helm-operators.md) | [next: 01-pgbouncer](../11-connection-pooling/01-pgbouncer.md)
