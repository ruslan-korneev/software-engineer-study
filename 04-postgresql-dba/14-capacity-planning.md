# Планирование ёмкости PostgreSQL (Capacity Planning)

## Введение

**Capacity Planning** (планирование ёмкости) — это процесс определения и прогнозирования ресурсов, необходимых для обеспечения стабильной и производительной работы базы данных PostgreSQL. Это критически важная практика для предотвращения проблем с производительностью, минимизации простоев и оптимизации затрат на инфраструктуру.

Основные цели capacity planning:
- Обеспечение достаточных ресурсов для текущей нагрузки
- Прогнозирование будущих потребностей
- Предотвращение неожиданных сбоев из-за нехватки ресурсов
- Оптимизация соотношения цена/производительность

---

## Ключевые метрики для мониторинга

### 1. CPU Usage (Использование процессора)

CPU — критический ресурс для PostgreSQL, особенно для:
- Выполнения сложных запросов
- Сортировки и агрегации данных
- Обработки соединений (JOIN)
- Шифрования SSL-соединений

```sql
-- Просмотр активных процессов и их состояния
SELECT pid, usename, state, query,
       now() - query_start AS duration
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;
```

**Пороговые значения:**
- < 70% — нормальная нагрузка
- 70-85% — повышенная нагрузка, требует внимания
- > 85% — критическая нагрузка, необходимо масштабирование

**Мониторинг на уровне ОС:**
```bash
# Linux: использование CPU
top -bn1 | grep "Cpu(s)"
mpstat -P ALL 1

# Нагрузка на систему (load average)
uptime

# Детальная информация о процессах PostgreSQL
ps aux | grep postgres
```

### 2. Memory Usage (Использование памяти)

PostgreSQL активно использует память для:
- **shared_buffers** — кэш страниц данных
- **work_mem** — память для сортировки и хэш-таблиц
- **maintenance_work_mem** — память для VACUUM и CREATE INDEX
- **effective_cache_size** — оценка доступного кэша ОС

```sql
-- Текущие настройки памяти
SHOW shared_buffers;
SHOW work_mem;
SHOW maintenance_work_mem;
SHOW effective_cache_size;

-- Статистика использования буферов
SELECT
    sum(heap_blks_read) AS heap_read,
    sum(heap_blks_hit) AS heap_hit,
    round(sum(heap_blks_hit)::numeric /
          nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100, 2)
          AS cache_hit_ratio
FROM pg_statio_user_tables;
```

**Рекомендации по настройке:**
```
# shared_buffers: 25% от общей RAM (но не более 8GB для большинства случаев)
shared_buffers = 4GB  # для сервера с 16GB RAM

# work_mem: зависит от количества соединений
# Формула: (Total RAM - shared_buffers) / (max_connections * 3)
work_mem = 64MB

# maintenance_work_mem: для операций обслуживания
maintenance_work_mem = 1GB

# effective_cache_size: 75% от общей RAM
effective_cache_size = 12GB  # для сервера с 16GB RAM
```

**Мониторинг памяти на уровне ОС:**
```bash
# Использование памяти
free -h
vmstat 1

# Детальная информация о памяти
cat /proc/meminfo

# Память процессов PostgreSQL
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | grep postgres
```

### 3. Disk I/O (Дисковые операции)

Диск — часто узкое место для баз данных. Важно отслеживать:
- **IOPS** (Input/Output Operations Per Second)
- **Throughput** (пропускная способность в MB/s)
- **Latency** (задержка операций)

```sql
-- Статистика дисковых операций по таблицам
SELECT
    schemaname,
    relname,
    heap_blks_read,
    heap_blks_hit,
    idx_blks_read,
    idx_blks_hit
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC
LIMIT 10;

-- Статистика по базе данных
SELECT
    datname,
    blks_read,
    blks_hit,
    round(blks_hit::numeric / nullif(blks_hit + blks_read, 0) * 100, 2)
        AS cache_hit_ratio
FROM pg_stat_database
WHERE datname = current_database();
```

**Мониторинг I/O на уровне ОС:**
```bash
# iostat для мониторинга дисков
iostat -xz 1

# iotop для процессов с наибольшим I/O
sudo iotop -o

# Статистика файловой системы
df -h
du -sh /var/lib/postgresql/data/*
```

**Рекомендации по конфигурации для I/O:**
```
# Для SSD дисков
random_page_cost = 1.1
effective_io_concurrency = 200

# Для HDD дисков
random_page_cost = 4.0
effective_io_concurrency = 2

# Checkpoint настройки для снижения I/O peaks
checkpoint_completion_target = 0.9
checkpoint_timeout = 15min
max_wal_size = 4GB
```

### 4. Connection Count (Количество соединений)

```sql
-- Текущие соединения
SELECT
    count(*) AS total_connections,
    count(*) FILTER (WHERE state = 'active') AS active,
    count(*) FILTER (WHERE state = 'idle') AS idle,
    count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_transaction
FROM pg_stat_activity
WHERE backend_type = 'client backend';

-- Соединения по пользователям и базам
SELECT
    usename,
    datname,
    count(*) AS connections
FROM pg_stat_activity
WHERE backend_type = 'client backend'
GROUP BY usename, datname
ORDER BY connections DESC;

-- Максимальное количество соединений
SHOW max_connections;
```

**Формула для расчёта max_connections:**
```
max_connections = (RAM в MB - shared_buffers в MB) / (work_mem в MB * expected_active_queries)
```

**Пример:**
```
# Для сервера с 16GB RAM
# shared_buffers = 4GB, work_mem = 64MB
# (16384 - 4096) / (64 * 3) = ~64 активных соединений
# С учётом idle соединений: max_connections = 200-300
```

### 5. Query Throughput (Пропускная способность запросов)

```sql
-- Статистика выполнения запросов (требуется pg_stat_statements)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT
    round(total_exec_time::numeric, 2) AS total_time_ms,
    calls,
    round(mean_exec_time::numeric, 2) AS avg_time_ms,
    rows,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Запросы в секунду
SELECT
    sum(calls) /
    EXTRACT(EPOCH FROM (now() - stats_reset)) AS queries_per_second
FROM pg_stat_statements, pg_stat_database
WHERE datname = current_database();

-- Транзакции в секунду
SELECT
    datname,
    xact_commit + xact_rollback AS total_xact,
    xact_commit AS commits,
    xact_rollback AS rollbacks
FROM pg_stat_database
WHERE datname = current_database();
```

---

## Инструменты для анализа нагрузки

### 1. Встроенные инструменты PostgreSQL

#### pg_stat_statements
```sql
-- Топ запросов по общему времени выполнения
SELECT
    substring(query, 1, 100) AS query_snippet,
    calls,
    round(total_exec_time::numeric, 2) AS total_ms,
    round(mean_exec_time::numeric, 2) AS avg_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

#### pg_stat_activity
```sql
-- Долгие запросы
SELECT
    pid,
    now() - query_start AS duration,
    state,
    query
FROM pg_stat_activity
WHERE state != 'idle'
  AND query_start < now() - interval '5 minutes'
ORDER BY duration DESC;
```

#### pg_stat_bgwriter
```sql
-- Статистика фонового записи
SELECT
    checkpoints_timed,
    checkpoints_req,
    buffers_checkpoint,
    buffers_clean,
    buffers_backend,
    stats_reset
FROM pg_stat_bgwriter;
```

### 2. Внешние инструменты мониторинга

#### pgBadger — анализатор логов
```bash
# Установка
apt install pgbadger  # Debian/Ubuntu
brew install pgbadger # macOS

# Анализ логов
pgbadger /var/log/postgresql/postgresql-*.log -o report.html

# Инкрементальный анализ
pgbadger -I -O /var/www/pgbadger/ /var/log/postgresql/*.log
```

**Конфигурация для pgBadger:**
```
# postgresql.conf
log_min_duration_statement = 100  # логировать запросы > 100ms
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 0
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
```

#### pg_top — аналог top для PostgreSQL
```bash
# Установка
apt install pgtop

# Запуск
pg_top -h localhost -U postgres -d mydb
```

#### pgCenter — интерактивный мониторинг
```bash
# Установка
apt install pgcenter

# Запуск
pgcenter top -h localhost -U postgres -d mydb
```

### 3. Системы мониторинга

#### Prometheus + Grafana
```yaml
# docker-compose.yml для postgres_exporter
version: '3'
services:
  postgres-exporter:
    image: prometheuscommunity/postgres-exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://user:pass@postgres:5432/db?sslmode=disable"
    ports:
      - "9187:9187"
```

**Ключевые метрики для Grafana:**
- `pg_stat_database_tup_fetched` — кортежи прочитанные
- `pg_stat_database_tup_inserted` — кортежи вставленные
- `pg_stat_database_blks_hit` — попадания в кэш
- `pg_stat_database_blks_read` — чтения с диска
- `pg_stat_activity_count` — количество соединений

#### Zabbix
```
# Ключевые параметры для мониторинга
pgsql.connections
pgsql.db.size
pgsql.queries.duration
pgsql.replication.lag
pgsql.cache.hit.ratio
```

---

## Расчёт требований к ресурсам

### Формулы для расчёта

#### RAM (оперативная память)
```
Минимум RAM = shared_buffers + (max_connections × work_mem) + OS_overhead

Пример для production:
- shared_buffers: 25% от RAM
- work_mem: 64MB × 100 соединений = 6.4GB (в худшем случае)
- OS и прочее: 2-4GB

Итого для 100 активных соединений: минимум 16GB RAM
```

#### CPU (процессор)
```
Рекомендации:
- 1 CPU core на каждые 50-100 активных соединений
- Дополнительные ядра для:
  - Background workers
  - Autovacuum workers
  - WAL writer
  - Checkpointer

Пример: 100 активных соединений → 4-8 CPU cores
```

#### Disk (дисковое пространство)
```sql
-- Текущий размер базы данных
SELECT pg_size_pretty(pg_database_size(current_database()));

-- Размер таблиц с индексами
SELECT
    schemaname || '.' || relname AS table_name,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
    pg_size_pretty(pg_relation_size(relid)) AS table_size,
    pg_size_pretty(pg_indexes_size(relid)) AS index_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 10;
```

**Формула для расчёта роста:**
```
Прогноз_размера = Текущий_размер × (1 + monthly_growth_rate)^months

# Учитывайте:
# - WAL файлы: 2-3x от основных данных при активной записи
# - VACUUM overhead: +20-30% для bloat
# - Резерв: +30% для безопасности
```

### Пример расчёта для проекта

```
Исходные данные:
- 1 миллион пользователей
- 100 запросов/сек в среднем, 1000 запросов/сек в пике
- Рост данных: 10GB/месяц
- Текущий размер: 50GB

Расчёт:
1. RAM:
   - shared_buffers: 4GB
   - work_mem: 64MB × 200 соединений max = 12.8GB
   - maintenance_work_mem: 1GB
   - OS: 2GB
   → Итого: 32GB RAM

2. CPU:
   - Базовая нагрузка: 4 cores
   - Пиковая нагрузка: 8 cores
   → Итого: 8 CPU cores

3. Disk:
   - Данные: 50GB + (10GB × 12 месяцев) = 170GB через год
   - WAL: 50GB
   - Bloat и резерв: 70GB
   → Итого: 300GB SSD (NVMe для production)
```

---

## Планирование роста базы данных

### Анализ трендов роста

```sql
-- Отслеживание размера таблиц со временем
CREATE TABLE IF NOT EXISTS db_size_history (
    recorded_at TIMESTAMP DEFAULT now(),
    table_name TEXT,
    row_count BIGINT,
    total_size BIGINT
);

-- Регулярная запись статистики (через cron или pg_cron)
INSERT INTO db_size_history (table_name, row_count, total_size)
SELECT
    schemaname || '.' || relname,
    n_live_tup,
    pg_total_relation_size(relid)
FROM pg_stat_user_tables;

-- Анализ роста
SELECT
    table_name,
    min(total_size) AS size_start,
    max(total_size) AS size_end,
    max(total_size) - min(total_size) AS growth,
    pg_size_pretty(max(total_size) - min(total_size)) AS growth_pretty
FROM db_size_history
WHERE recorded_at > now() - interval '30 days'
GROUP BY table_name
ORDER BY growth DESC;
```

### Прогнозирование

```sql
-- Линейная экстраполяция роста
WITH daily_sizes AS (
    SELECT
        date_trunc('day', recorded_at) AS day,
        max(total_size) AS daily_size
    FROM db_size_history
    WHERE table_name = 'public.users'
    GROUP BY date_trunc('day', recorded_at)
),
growth_rate AS (
    SELECT
        (max(daily_size) - min(daily_size))::float /
        EXTRACT(days FROM max(day) - min(day)) AS daily_growth
    FROM daily_sizes
)
SELECT
    pg_size_pretty((SELECT max(daily_size) FROM daily_sizes)::bigint) AS current_size,
    pg_size_pretty(
        ((SELECT max(daily_size) FROM daily_sizes) +
         (SELECT daily_growth * 365 FROM growth_rate))::bigint
    ) AS projected_size_1_year;
```

---

## Горизонтальное и вертикальное масштабирование

### Вертикальное масштабирование (Scale Up)

**Преимущества:**
- Простота реализации
- Не требует изменений в приложении
- Единая точка управления

**Ограничения:**
- Физический предел железа
- Стоимость растёт экспоненциально
- Single point of failure

```
Путь вертикального масштабирования:
1. Начало: 4 CPU, 16GB RAM, 100GB SSD
2. Этап 2: 8 CPU, 32GB RAM, 500GB SSD
3. Этап 3: 16 CPU, 64GB RAM, 1TB NVMe
4. Этап 4: 32 CPU, 128GB RAM, 2TB NVMe RAID
5. Предел: 64+ CPU, 256GB+ RAM, NVMe arrays
```

### Горизонтальное масштабирование (Scale Out)

#### Read Replicas (Реплики для чтения)
```sql
-- Настройка streaming replication
-- На primary сервере:
-- postgresql.conf
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB

-- pg_hba.conf
host replication replicator 192.168.1.0/24 md5

-- На replica:
-- Создание replica
pg_basebackup -h primary -D /var/lib/postgresql/data -U replicator -P -R
```

**Балансировка нагрузки:**
```python
# Пример с pgpool-II или приложением
import psycopg2
import random

read_replicas = [
    "host=replica1 dbname=mydb",
    "host=replica2 dbname=mydb",
]
primary = "host=primary dbname=mydb"

def get_connection(read_only=False):
    if read_only:
        return psycopg2.connect(random.choice(read_replicas))
    return psycopg2.connect(primary)
```

#### Партиционирование (Partitioning)
```sql
-- Партиционирование по диапазону дат
CREATE TABLE orders (
    id BIGSERIAL,
    created_at TIMESTAMP NOT NULL,
    customer_id INTEGER,
    amount DECIMAL(10,2)
) PARTITION BY RANGE (created_at);

-- Создание партиций
CREATE TABLE orders_2024_q1 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE orders_2024_q2 PARTITION OF orders
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- Автоматическое создание партиций через pg_partman
CREATE EXTENSION pg_partman;
SELECT partman.create_parent(
    'public.orders',
    'created_at',
    'native',
    'monthly'
);
```

#### Шардинование (Sharding)
```sql
-- С использованием Citus (распределённый PostgreSQL)
CREATE EXTENSION citus;

-- Создание распределённой таблицы
SELECT create_distributed_table('orders', 'customer_id');

-- Добавление worker nodes
SELECT * from citus_add_node('worker-1', 5432);
SELECT * from citus_add_node('worker-2', 5432);
```

---

## Best Practices для Capacity Planning

### 1. Регулярный мониторинг
```sql
-- Создание представления для мониторинга
CREATE OR REPLACE VIEW v_capacity_metrics AS
SELECT
    -- Database size
    pg_database_size(current_database()) AS db_size_bytes,
    pg_size_pretty(pg_database_size(current_database())) AS db_size_pretty,

    -- Connections
    (SELECT count(*) FROM pg_stat_activity
     WHERE backend_type = 'client backend') AS current_connections,
    (SELECT setting::int FROM pg_settings
     WHERE name = 'max_connections') AS max_connections,

    -- Cache hit ratio
    (SELECT round(
        sum(heap_blks_hit)::numeric /
        nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100, 2
    ) FROM pg_statio_user_tables) AS cache_hit_ratio,

    -- Transaction rate
    (SELECT xact_commit + xact_rollback
     FROM pg_stat_database
     WHERE datname = current_database()) AS total_transactions,

    now() AS measured_at;
```

### 2. Установка алертов

**Пороговые значения для алертов:**
```yaml
alerts:
  - name: "High CPU Usage"
    condition: "cpu_usage > 80%"
    severity: warning

  - name: "Critical CPU Usage"
    condition: "cpu_usage > 95%"
    severity: critical

  - name: "Low Disk Space"
    condition: "disk_free < 20%"
    severity: warning

  - name: "Critical Disk Space"
    condition: "disk_free < 10%"
    severity: critical

  - name: "Connection Pool Exhaustion"
    condition: "connections > max_connections * 0.8"
    severity: warning

  - name: "Cache Hit Ratio Low"
    condition: "cache_hit_ratio < 95%"
    severity: warning

  - name: "Replication Lag"
    condition: "replication_lag > 30s"
    severity: warning
```

### 3. Документирование и планирование

```markdown
## Чек-лист capacity planning

### Ежедневно
- [ ] Проверка алертов мониторинга
- [ ] Обзор медленных запросов

### Еженедельно
- [ ] Анализ трендов роста данных
- [ ] Проверка использования ресурсов
- [ ] Обзор pg_stat_statements

### Ежемесячно
- [ ] Полный отчёт о производительности
- [ ] Пересмотр прогнозов роста
- [ ] Оценка необходимости масштабирования

### Ежеквартально
- [ ] Стратегический обзор инфраструктуры
- [ ] Планирование бюджета на ресурсы
- [ ] Тестирование failover и recovery
```

### 4. Автоматизация

```bash
#!/bin/bash
# capacity_check.sh - скрипт ежедневной проверки

DB_HOST="localhost"
DB_NAME="mydb"
DB_USER="postgres"

# Проверка размера БД
psql -h $DB_HOST -U $DB_USER -d $DB_NAME -c "
SELECT
    pg_size_pretty(pg_database_size('$DB_NAME')) AS db_size,
    (SELECT count(*) FROM pg_stat_activity) AS connections,
    (SELECT round(
        sum(heap_blks_hit)::numeric /
        nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100, 2
    ) FROM pg_statio_user_tables) AS cache_hit_ratio;
"

# Проверка дискового пространства
df -h /var/lib/postgresql

# Отправка отчёта (например, в Slack)
# curl -X POST -H 'Content-type: application/json' \
#   --data '{"text":"Daily capacity report: ..."}' \
#   $SLACK_WEBHOOK_URL
```

### 5. Резервирование ресурсов

**Золотые правила:**
- Всегда иметь 30% запас по RAM
- Дисковое пространство: предупреждение на 20%, критично на 10%
- CPU: средняя загрузка не более 70%
- Соединения: использовать не более 80% от max_connections

---

## Заключение

Эффективное планирование ёмкости PostgreSQL требует:

1. **Систематического мониторинга** — нельзя управлять тем, что не измеряешь
2. **Проактивного подхода** — реагировать до возникновения проблем
3. **Понимания характера нагрузки** — OLTP vs OLAP, пики vs средняя нагрузка
4. **Гибкой стратегии масштабирования** — комбинация вертикального и горизонтального подходов
5. **Регулярного пересмотра** — потребности меняются со временем

Правильно настроенный capacity planning позволяет избежать неожиданных простоев, оптимизировать расходы на инфраструктуру и обеспечить стабильную работу приложений.
