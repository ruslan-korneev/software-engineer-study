# Shell Scripts для автоматизации PostgreSQL

## Введение

Shell-скрипты являются основным инструментом автоматизации рутинных задач администрирования PostgreSQL. Они позволяют автоматизировать резервное копирование, мониторинг, обслуживание базы данных и многие другие операции.

## Основы работы с psql в скриптах

### Подключение к базе данных

```bash
#!/bin/bash

# Переменные подключения
export PGHOST="localhost"
export PGPORT="5432"
export PGDATABASE="mydb"
export PGUSER="postgres"
export PGPASSWORD="secret"  # Не рекомендуется в production

# Лучше использовать .pgpass файл
# ~/.pgpass формат: hostname:port:database:username:password
# chmod 600 ~/.pgpass
```

### Выполнение SQL-запросов

```bash
#!/bin/bash

# Простой запрос
psql -c "SELECT version();"

# Запрос с выводом только данных (без заголовков)
psql -t -c "SELECT count(*) FROM users;"

# Запрос из файла
psql -f /path/to/script.sql

# Запрос с параметрами
psql -v table_name='users' -c "SELECT * FROM :table_name LIMIT 10;"

# Формат вывода CSV
psql -A -F',' -c "SELECT id, name FROM users;" > users.csv
```

## Скрипты резервного копирования

### Полное резервное копирование

```bash
#!/bin/bash

# Конфигурация
BACKUP_DIR="/var/backups/postgresql"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=7
DB_NAME="production_db"

# Создание директории
mkdir -p "$BACKUP_DIR"

# Полный дамп базы данных
pg_dump -Fc -v "$DB_NAME" > "$BACKUP_DIR/${DB_NAME}_${DATE}.dump"

# Проверка успешности
if [ $? -eq 0 ]; then
    echo "Backup completed successfully: ${DB_NAME}_${DATE}.dump"

    # Удаление старых бэкапов
    find "$BACKUP_DIR" -name "*.dump" -mtime +$RETENTION_DAYS -delete
    echo "Old backups cleaned up"
else
    echo "Backup failed!" >&2
    exit 1
fi
```

### Инкрементальное резервное копирование с WAL

```bash
#!/bin/bash

# Базовый бэкап для PITR
BACKUP_DIR="/var/backups/postgresql/base"
WAL_DIR="/var/backups/postgresql/wal"
DATE=$(date +%Y%m%d_%H%M%S)

# Создание базового бэкапа
pg_basebackup -D "$BACKUP_DIR/$DATE" -Ft -z -P -X stream

# Архивация WAL файлов настраивается в postgresql.conf:
# archive_mode = on
# archive_command = 'cp %p /var/backups/postgresql/wal/%f'

echo "Base backup created at $BACKUP_DIR/$DATE"
```

### Скрипт восстановления

```bash
#!/bin/bash

BACKUP_FILE=$1
DB_NAME=$2

if [ -z "$BACKUP_FILE" ] || [ -z "$DB_NAME" ]; then
    echo "Usage: $0 <backup_file> <database_name>"
    exit 1
fi

# Удаление существующей базы (осторожно!)
psql -c "DROP DATABASE IF EXISTS $DB_NAME;"

# Создание новой базы
psql -c "CREATE DATABASE $DB_NAME;"

# Восстановление
pg_restore -d "$DB_NAME" -v "$BACKUP_FILE"

if [ $? -eq 0 ]; then
    echo "Database restored successfully"
else
    echo "Restore failed!" >&2
    exit 1
fi
```

## Скрипты мониторинга

### Проверка состояния базы данных

```bash
#!/bin/bash

# Проверка доступности PostgreSQL
check_postgres_alive() {
    pg_isready -h localhost -p 5432
    return $?
}

# Проверка размера базы данных
check_database_size() {
    psql -t -c "SELECT pg_size_pretty(pg_database_size('$1'));"
}

# Проверка активных подключений
check_connections() {
    psql -t -c "
        SELECT count(*) as total,
               count(*) FILTER (WHERE state = 'active') as active,
               count(*) FILTER (WHERE state = 'idle') as idle
        FROM pg_stat_activity
        WHERE datname = '$1';
    "
}

# Проверка долгих запросов
check_long_queries() {
    local threshold_seconds=${1:-60}
    psql -c "
        SELECT pid,
               now() - pg_stat_activity.query_start AS duration,
               query,
               state
        FROM pg_stat_activity
        WHERE (now() - pg_stat_activity.query_start) > interval '$threshold_seconds seconds'
          AND state != 'idle';
    "
}

# Проверка репликации
check_replication_lag() {
    psql -t -c "
        SELECT client_addr,
               state,
               pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes
        FROM pg_stat_replication;
    "
}

# Основной скрипт мониторинга
main() {
    echo "=== PostgreSQL Health Check ==="
    echo "Timestamp: $(date)"
    echo

    if check_postgres_alive; then
        echo "[OK] PostgreSQL is running"
    else
        echo "[CRITICAL] PostgreSQL is not responding!"
        exit 2
    fi

    echo
    echo "Database sizes:"
    for db in $(psql -t -c "SELECT datname FROM pg_database WHERE datistemplate = false;"); do
        echo "  $db: $(check_database_size $db)"
    done

    echo
    echo "Connections (production_db):"
    check_connections "production_db"

    echo
    echo "Long running queries (>60s):"
    check_long_queries 60
}

main
```

### Скрипт оповещения

```bash
#!/bin/bash

LOG_FILE="/var/log/postgresql/alerts.log"
EMAIL="dba@company.com"
SLACK_WEBHOOK="https://hooks.slack.com/services/xxx"

# Функция отправки оповещения
send_alert() {
    local level=$1
    local message=$2
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')

    # Запись в лог
    echo "[$timestamp] [$level] $message" >> "$LOG_FILE"

    # Отправка email
    echo "$message" | mail -s "PostgreSQL Alert: $level" "$EMAIL"

    # Отправка в Slack
    curl -X POST -H 'Content-type: application/json' \
        --data "{\"text\":\"[$level] $message\"}" \
        "$SLACK_WEBHOOK"
}

# Проверка места на диске
check_disk_space() {
    local threshold=80
    local usage=$(df -h /var/lib/postgresql | awk 'NR==2 {print $5}' | tr -d '%')

    if [ "$usage" -gt "$threshold" ]; then
        send_alert "WARNING" "Disk usage is ${usage}% (threshold: ${threshold}%)"
    fi
}

# Проверка заблокированных запросов
check_blocked_queries() {
    local blocked_count=$(psql -t -c "
        SELECT count(*)
        FROM pg_stat_activity
        WHERE wait_event_type = 'Lock';
    " | tr -d ' ')

    if [ "$blocked_count" -gt 5 ]; then
        send_alert "WARNING" "Found $blocked_count blocked queries"
    fi
}

# Проверка deadlocks
check_deadlocks() {
    local deadlocks=$(psql -t -c "
        SELECT deadlocks
        FROM pg_stat_database
        WHERE datname = current_database();
    " | tr -d ' ')

    # Сравнение с предыдущим значением (сохраненным в файле)
    local prev_file="/tmp/pg_deadlocks_count"
    local prev_deadlocks=0

    if [ -f "$prev_file" ]; then
        prev_deadlocks=$(cat "$prev_file")
    fi

    if [ "$deadlocks" -gt "$prev_deadlocks" ]; then
        send_alert "CRITICAL" "New deadlock detected! Total: $deadlocks"
    fi

    echo "$deadlocks" > "$prev_file"
}

# Запуск всех проверок
check_disk_space
check_blocked_queries
check_deadlocks
```

## Скрипты обслуживания

### Автоматический VACUUM

```bash
#!/bin/bash

DB_NAME="production_db"
LOG_FILE="/var/log/postgresql/vacuum.log"

echo "Starting vacuum at $(date)" >> "$LOG_FILE"

# Получение списка таблиц с большим количеством dead tuples
tables_to_vacuum=$(psql -t -d "$DB_NAME" -c "
    SELECT schemaname || '.' || relname
    FROM pg_stat_user_tables
    WHERE n_dead_tup > 10000
    ORDER BY n_dead_tup DESC
    LIMIT 10;
")

for table in $tables_to_vacuum; do
    echo "Vacuuming $table..." >> "$LOG_FILE"
    psql -d "$DB_NAME" -c "VACUUM (VERBOSE, ANALYZE) $table;" >> "$LOG_FILE" 2>&1
done

echo "Vacuum completed at $(date)" >> "$LOG_FILE"
```

### Переиндексация

```bash
#!/bin/bash

DB_NAME="production_db"

# Поиск раздутых индексов
bloated_indexes=$(psql -t -d "$DB_NAME" -c "
    SELECT schemaname || '.' || indexrelname
    FROM pg_stat_user_indexes ui
    JOIN pg_index i ON ui.indexrelid = i.indexrelid
    WHERE pg_relation_size(ui.indexrelid) > 100 * 1024 * 1024  -- > 100MB
    ORDER BY pg_relation_size(ui.indexrelid) DESC
    LIMIT 5;
")

for index in $bloated_indexes; do
    echo "Reindexing $index..."
    psql -d "$DB_NAME" -c "REINDEX INDEX CONCURRENTLY $index;"
done
```

### Очистка логов

```bash
#!/bin/bash

LOG_DIR="/var/log/postgresql"
RETENTION_DAYS=30

# Удаление старых логов
find "$LOG_DIR" -name "postgresql-*.log" -mtime +$RETENTION_DAYS -delete

# Сжатие логов старше 7 дней
find "$LOG_DIR" -name "postgresql-*.log" -mtime +7 ! -name "*.gz" -exec gzip {} \;

echo "Log cleanup completed"
```

## Скрипты управления пользователями

### Создание пользователя с правами

```bash
#!/bin/bash

create_app_user() {
    local username=$1
    local password=$2
    local database=$3

    # Создание пользователя
    psql -c "CREATE USER $username WITH PASSWORD '$password';"

    # Назначение прав
    psql -d "$database" -c "
        GRANT CONNECT ON DATABASE $database TO $username;
        GRANT USAGE ON SCHEMA public TO $username;
        GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO $username;
        GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO $username;

        -- Права на будущие объекты
        ALTER DEFAULT PRIVILEGES IN SCHEMA public
            GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO $username;
        ALTER DEFAULT PRIVILEGES IN SCHEMA public
            GRANT USAGE, SELECT ON SEQUENCES TO $username;
    "

    echo "User $username created with access to $database"
}

# Использование
create_app_user "app_user" "secure_password" "production_db"
```

### Аудит пользователей

```bash
#!/bin/bash

echo "=== User Audit Report ==="
echo "Generated: $(date)"
echo

echo "All roles:"
psql -c "
    SELECT rolname,
           rolsuper,
           rolcreaterole,
           rolcreatedb,
           rolcanlogin,
           rolconnlimit
    FROM pg_roles
    ORDER BY rolname;
"

echo
echo "Role memberships:"
psql -c "
    SELECT r.rolname as role,
           m.rolname as member
    FROM pg_auth_members am
    JOIN pg_roles r ON am.roleid = r.oid
    JOIN pg_roles m ON am.member = m.oid
    ORDER BY r.rolname, m.rolname;
"

echo
echo "Table privileges:"
psql -c "
    SELECT grantee, table_schema, table_name, privilege_type
    FROM information_schema.table_privileges
    WHERE table_schema = 'public'
    ORDER BY grantee, table_name;
"
```

## Best Practices

### Безопасность скриптов

```bash
#!/bin/bash

# 1. Никогда не храните пароли в скриптах
# Используйте .pgpass или переменные окружения

# 2. Устанавливайте правильные права на скрипты
# chmod 700 script.sh
# chown postgres:postgres script.sh

# 3. Используйте set для безопасности
set -e          # Выход при ошибке
set -u          # Ошибка при использовании неопределенной переменной
set -o pipefail # Ошибка в pipeline передается

# 4. Логируйте все действия
exec > >(tee -a /var/log/postgresql/script.log) 2>&1

# 5. Используйте trap для очистки
cleanup() {
    rm -f /tmp/temp_file_$$
}
trap cleanup EXIT
```

### Обработка ошибок

```bash
#!/bin/bash

run_query() {
    local query=$1
    local result

    result=$(psql -t -c "$query" 2>&1)
    local exit_code=$?

    if [ $exit_code -ne 0 ]; then
        echo "Error executing query: $query" >&2
        echo "Error message: $result" >&2
        return 1
    fi

    echo "$result"
    return 0
}

# Использование с проверкой
if ! count=$(run_query "SELECT count(*) FROM users;"); then
    echo "Failed to get user count"
    exit 1
fi

echo "User count: $count"
```

### Параллельное выполнение

```bash
#!/bin/bash

# Параллельный vacuum нескольких баз
databases=("db1" "db2" "db3" "db4")
max_parallel=2

vacuum_db() {
    local db=$1
    echo "Starting vacuum on $db"
    psql -d "$db" -c "VACUUM ANALYZE;"
    echo "Completed vacuum on $db"
}

# Запуск с ограничением параллельности
for db in "${databases[@]}"; do
    vacuum_db "$db" &

    # Ограничение количества параллельных процессов
    while [ $(jobs -r | wc -l) -ge $max_parallel ]; do
        sleep 1
    done
done

# Ожидание завершения всех процессов
wait
echo "All vacuum operations completed"
```

## Cron-задания

```bash
# /etc/cron.d/postgresql-maintenance

# Ежедневный бэкап в 2:00
0 2 * * * postgres /opt/scripts/backup.sh >> /var/log/postgresql/backup.log 2>&1

# Мониторинг каждые 5 минут
*/5 * * * * postgres /opt/scripts/monitoring.sh >> /var/log/postgresql/monitoring.log 2>&1

# Еженедельная очистка логов в воскресенье в 3:00
0 3 * * 0 postgres /opt/scripts/cleanup_logs.sh >> /var/log/postgresql/cleanup.log 2>&1

# Ежемесячный полный VACUUM в первое воскресенье в 4:00
0 4 1-7 * 0 postgres /opt/scripts/full_vacuum.sh >> /var/log/postgresql/vacuum.log 2>&1
```

## Полезные утилиты

### Скрипт проверки pg_hba.conf

```bash
#!/bin/bash

PG_HBA="/etc/postgresql/15/main/pg_hba.conf"

echo "Current pg_hba.conf rules:"
grep -v "^#" "$PG_HBA" | grep -v "^$"

echo
echo "Testing connection rules:"
pg_hba_test() {
    local type=$1
    local database=$2
    local user=$3
    local address=$4

    psql -h "$address" -U "$user" -d "$database" -c "SELECT 1;" 2>&1 | head -1
}
```

## Заключение

Shell-скрипты являются мощным инструментом для автоматизации задач PostgreSQL DBA. Ключевые моменты:

1. **Всегда используйте обработку ошибок** (`set -e`, проверка кодов возврата)
2. **Логируйте все действия** для отладки и аудита
3. **Не храните пароли в скриптах** - используйте `.pgpass` или переменные окружения
4. **Тестируйте скрипты** на тестовом окружении перед production
5. **Используйте cron** для регулярных задач обслуживания
6. **Отправляйте оповещения** при критических событиях

Следующий шаг - изучение инструментов управления конфигурацией (Ansible, Puppet) для более масштабной автоматизации.
