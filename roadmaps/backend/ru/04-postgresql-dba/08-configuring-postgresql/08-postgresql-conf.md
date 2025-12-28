# postgresql.conf (Основной файл конфигурации)

[prev: 07-extensions](./07-extensions.md) | [next: 01-object-privileges](../09-security/01-object-privileges.md)

---

postgresql.conf — главный файл конфигурации PostgreSQL, содержащий все параметры настройки сервера. Понимание структуры и методов работы с этим файлом критически важно для администрирования.

## Расположение конфигурационных файлов

```sql
-- Показать расположение файлов конфигурации
SHOW config_file;
SHOW data_directory;
SHOW hba_file;

-- Все пути
SELECT name, setting FROM pg_settings
WHERE name IN ('config_file', 'data_directory', 'hba_file', 'ident_file');
```

Типичные расположения:
- **Debian/Ubuntu**: `/etc/postgresql/16/main/postgresql.conf`
- **RHEL/CentOS**: `/var/lib/pgsql/16/data/postgresql.conf`
- **Из исходников**: `$PGDATA/postgresql.conf`

## Иерархия конфигурационных файлов

```
postgresql.conf          # основной файл
├── postgresql.auto.conf # изменения через ALTER SYSTEM
└── conf.d/              # включаемые файлы
    ├── 01-memory.conf
    ├── 02-replication.conf
    └── 99-custom.conf
```

### Включение дополнительных файлов

```ini
# postgresql.conf

# Включение одного файла
include = 'custom.conf'

# Включение, если файл существует
include_if_exists = 'optional.conf'

# Включение всех файлов из директории
include_dir = 'conf.d'
```

### Порядок применения

1. `postgresql.conf` — базовые настройки
2. Файлы из `include_dir` (в алфавитном порядке)
3. `postgresql.auto.conf` — перезаписывает предыдущие

## Категории параметров

### По контексту применения

```sql
-- Показать контекст всех параметров
SELECT name, context, setting
FROM pg_settings
ORDER BY context, name;
```

| context | Когда применяется |
|---------|-------------------|
| internal | Нельзя изменить |
| postmaster | Требуется перезапуск |
| sighup | После reload |
| superuser | Суперпользователь, немедленно |
| user | Любой пользователь, немедленно |
| superuser-backend | Суперпользователь, для новых сессий |
| backend | При подключении |

### Проверка параметра

```sql
-- Информация о параметре
SELECT name, setting, unit, context, short_desc, boot_val, reset_val
FROM pg_settings
WHERE name = 'shared_buffers';

-- Все параметры с отличием от default
SELECT name, setting, boot_val
FROM pg_settings
WHERE setting != boot_val
ORDER BY name;
```

## Изменение параметров

### Через ALTER SYSTEM

```sql
-- Изменение параметра (записывается в postgresql.auto.conf)
ALTER SYSTEM SET shared_buffers = '4GB';
ALTER SYSTEM SET max_connections = 200;

-- Сброс к значению по умолчанию
ALTER SYSTEM RESET shared_buffers;

-- Сброс всех
ALTER SYSTEM RESET ALL;

-- Применение (для sighup параметров)
SELECT pg_reload_conf();
```

### Через SET

```sql
-- Для текущей сессии
SET work_mem = '256MB';
SET statement_timeout = '30s';

-- Для текущей транзакции
SET LOCAL work_mem = '512MB';

-- Сброс
RESET work_mem;
RESET ALL;

-- Показать текущее значение
SHOW work_mem;
```

### Через команду pg_ctl

```bash
# Reload конфигурации
pg_ctl reload -D /var/lib/postgresql/16/main

# Или через systemctl
sudo systemctl reload postgresql
```

## Основные секции конфигурации

### Connections and Authentication

```ini
#------------------------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------------------------

# Connection Settings
listen_addresses = 'localhost'          # или '*' для всех интерфейсов
port = 5432
max_connections = 100                   # максимум соединений
superuser_reserved_connections = 3

# TCP Settings
tcp_keepalives_idle = 300
tcp_keepalives_interval = 60
tcp_keepalives_count = 10

# Authentication
authentication_timeout = 1min
password_encryption = scram-sha-256
```

### Resource Usage

```ini
#------------------------------------------------------------------------------
# RESOURCE USAGE (except WAL)
#------------------------------------------------------------------------------

# Memory
shared_buffers = 4GB                    # 25% RAM
huge_pages = try
temp_buffers = 8MB
work_mem = 64MB
hash_mem_multiplier = 2.0
maintenance_work_mem = 512MB
autovacuum_work_mem = -1                # -1 = maintenance_work_mem

# Disk
temp_file_limit = -1                    # -1 = без лимита

# Kernel Resources
max_files_per_process = 1000

# Background Writer
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
bgwriter_lru_multiplier = 2.0
bgwriter_flush_after = 512kB
```

### Write-Ahead Log

```ini
#------------------------------------------------------------------------------
# WRITE-AHEAD LOG
#------------------------------------------------------------------------------

# Settings
wal_level = replica                     # minimal, replica, logical
fsync = on
synchronous_commit = on
wal_sync_method = fdatasync

# Checkpoints
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9
max_wal_size = 4GB
min_wal_size = 1GB

# Archiving
archive_mode = on
archive_command = 'cp %p /archive/%f'
archive_timeout = 300
```

### Replication

```ini
#------------------------------------------------------------------------------
# REPLICATION
#------------------------------------------------------------------------------

# Primary Server
max_wal_senders = 10
wal_keep_size = 1GB
max_replication_slots = 10
synchronous_standby_names = ''

# Standby Server
primary_conninfo = ''
hot_standby = on
hot_standby_feedback = on
```

### Query Tuning

```ini
#------------------------------------------------------------------------------
# QUERY TUNING
#------------------------------------------------------------------------------

# Planner Cost Constants
seq_page_cost = 1.0
random_page_cost = 1.1                  # для SSD
effective_cache_size = 12GB             # 75% RAM

# Parallelism
max_parallel_workers_per_gather = 2
max_parallel_workers = 8
max_parallel_maintenance_workers = 4
parallel_leader_participation = on

# JIT
jit = on
jit_above_cost = 100000
```

### Logging

```ini
#------------------------------------------------------------------------------
# ERROR REPORTING AND LOGGING
#------------------------------------------------------------------------------

# Where to Log
log_destination = 'csvlog'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d.log'
log_rotation_age = 1d

# What to Log
log_min_duration_statement = 500
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_statement = 'ddl'
log_temp_files = 0
log_autovacuum_min_duration = 0
log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a '

# Statistics
log_parser_stats = off
log_planner_stats = off
log_executor_stats = off
log_statement_stats = off
```

### Statistics

```ini
#------------------------------------------------------------------------------
# STATISTICS
#------------------------------------------------------------------------------

# Cumulative Statistics
track_activities = on
track_counts = on
track_io_timing = on
track_wal_io_timing = on
track_functions = all
track_activity_query_size = 4096
stats_temp_directory = 'pg_stat_tmp'
```

### Autovacuum

```ini
#------------------------------------------------------------------------------
# AUTOVACUUM
#------------------------------------------------------------------------------

autovacuum = on
autovacuum_max_workers = 3
autovacuum_naptime = 1min
autovacuum_vacuum_threshold = 50
autovacuum_vacuum_scale_factor = 0.02
autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.01
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 1000
```

### Client Connection Defaults

```ini
#------------------------------------------------------------------------------
# CLIENT CONNECTION DEFAULTS
#------------------------------------------------------------------------------

# Statement Behavior
statement_timeout = 0                   # 0 = отключено
lock_timeout = 0
idle_in_transaction_session_timeout = 0
idle_session_timeout = 0

# Locale and Formatting
datestyle = 'iso, mdy'
timezone = 'UTC'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
default_text_search_config = 'pg_catalog.english'

# Shared Library Preloading
shared_preload_libraries = 'pg_stat_statements, auto_explain'
```

## Примеры конфигураций

### Сервер для OLTP (16GB RAM, SSD)

```ini
# Memory
shared_buffers = 4GB
effective_cache_size = 12GB
work_mem = 32MB
maintenance_work_mem = 512MB

# Connections
max_connections = 200

# WAL & Checkpoints
wal_level = replica
checkpoint_timeout = 15min
max_wal_size = 4GB
checkpoint_completion_target = 0.9

# Planner
random_page_cost = 1.1
effective_io_concurrency = 200

# Parallelism
max_parallel_workers_per_gather = 2
max_parallel_workers = 8

# Autovacuum (агрессивный)
autovacuum_max_workers = 4
autovacuum_vacuum_scale_factor = 0.02
autovacuum_vacuum_cost_delay = 2ms
```

### Сервер для OLAP (64GB RAM, NVMe)

```ini
# Memory
shared_buffers = 16GB
effective_cache_size = 48GB
work_mem = 512MB
maintenance_work_mem = 2GB

# Connections
max_connections = 50

# WAL & Checkpoints
checkpoint_timeout = 30min
max_wal_size = 16GB

# Planner
random_page_cost = 1.0
effective_io_concurrency = 200

# Parallelism (максимальный)
max_parallel_workers_per_gather = 8
max_parallel_workers = 16
max_parallel_maintenance_workers = 8

# JIT
jit = on
jit_above_cost = 50000
```

## Валидация конфигурации

```bash
# Проверка синтаксиса
postgres -C config_file -D /var/lib/postgresql/16/main

# Тест запуска (dry-run недоступен, но можно проверить)
pg_ctl start -D /var/lib/postgresql/16/main -o "-c config_file=/path/test.conf" -w -t 5
```

```sql
-- Проверка pending restart параметров
SELECT name, setting, pending_restart
FROM pg_settings
WHERE pending_restart = true;

-- Источник текущего значения
SELECT name, setting, source, sourcefile, sourceline
FROM pg_settings
WHERE source != 'default'
ORDER BY source, name;
```

## Инструменты для настройки

### PGTune

Онлайн-калькулятор рекомендуемых настроек:
- https://pgtune.leopard.in.ua/

### postgresqltuner

```bash
# Perl скрипт для анализа
git clone https://github.com/jfcoz/postgresqltuner.git
perl postgresqltuner.pl --host=localhost --user=postgres --password=xxx
```

## Типичные ошибки

1. **Редактирование postgresql.auto.conf вручную** — используйте ALTER SYSTEM
2. **Забыли reload после изменения** — параметры не применились
3. **shared_buffers больше 25% RAM** — конкуренция с OS cache
4. **Изменение postmaster параметров без перезапуска** — ожидание не даёт результата
5. **Игнорирование pending_restart** — параметр не применён

## Best Practices

1. **Используйте include_dir** для организации настроек
2. **Документируйте изменения** комментариями
3. **Храните конфигурацию в VCS** (git)
4. **Тестируйте изменения** на staging
5. **Мониторьте pending_restart** после reload
6. **Используйте ALTER SYSTEM** вместо ручного редактирования
7. **Регулярно пересматривайте** настройки при изменении нагрузки
8. **Сравнивайте с PGTune** как базовую линию

```sql
-- Экспорт текущей конфигурации
COPY (
    SELECT name, setting, source
    FROM pg_settings
    WHERE source != 'default'
    ORDER BY name
) TO '/tmp/current_settings.csv' CSV HEADER;
```

---

[prev: 07-extensions](./07-extensions.md) | [next: 01-object-privileges](../09-security/01-object-privileges.md)
