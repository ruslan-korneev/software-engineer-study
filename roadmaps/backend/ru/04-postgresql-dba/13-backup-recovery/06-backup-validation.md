# Backup Validation - Проверка резервных копий PostgreSQL

[prev: 05-pgbackrest](./05-pgbackrest.md) | [next: 14-capacity-planning](../14-capacity-planning.md)

---

## Введение

**Проверка резервных копий** (Backup Validation) - критически важный процесс, который подтверждает, что созданные бэкапы можно успешно использовать для восстановления данных. Бэкап, который нельзя восстановить, бесполезен, поэтому регулярное тестирование является обязательной частью любой стратегии резервного копирования.

### Почему проверка бэкапов критична

- **Обнаружение проблем заранее** - до реальной аварии
- **Подтверждение RTO** - реальное время восстановления
- **Проверка целостности данных** - данные не повреждены
- **Соответствие требованиям** - compliance и аудит
- **Уверенность команды** - бэкапы работают

### Уровни проверки

```
Уровень 1: Базовая проверка
├── Файл существует
├── Размер файла корректен
└── Архив читается

Уровень 2: Структурная проверка
├── Метаданные валидны
├── Список объектов корректен
└── Checksums совпадают

Уровень 3: Функциональная проверка
├── Полное восстановление
├── Проверка данных
└── Тестирование запросов

Уровень 4: Интеграционная проверка
├── Приложение работает
├── Данные консистентны
└── Производительность адекватна
```

## Проверка pg_dump бэкапов

### Базовая проверка архива

```bash
#!/bin/bash
# validate_pgdump_basic.sh

BACKUP_FILE=$1

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup_file>"
    exit 1
fi

echo "=== Basic Validation: $BACKUP_FILE ==="

# Проверка существования файла
if [ ! -f "$BACKUP_FILE" ]; then
    echo "FAIL: File does not exist"
    exit 1
fi

# Проверка размера
SIZE=$(stat -c %s "$BACKUP_FILE" 2>/dev/null || stat -f %z "$BACKUP_FILE")
if [ "$SIZE" -lt 1000 ]; then
    echo "WARN: File size is suspiciously small ($SIZE bytes)"
fi

# Определение типа файла
FILE_TYPE=$(file "$BACKUP_FILE")
echo "File type: $FILE_TYPE"

# Проверка в зависимости от формата
case "$BACKUP_FILE" in
    *.sql)
        # Plain SQL - проверить что начинается с PostgreSQL header
        if head -n 5 "$BACKUP_FILE" | grep -q "PostgreSQL database dump"; then
            echo "OK: Valid PostgreSQL SQL dump"
        else
            echo "WARN: May not be a PostgreSQL dump"
        fi
        ;;
    *.dump)
        # Custom format - использовать pg_restore -l
        if pg_restore -l "$BACKUP_FILE" > /dev/null 2>&1; then
            echo "OK: Valid custom format dump"
            OBJECTS=$(pg_restore -l "$BACKUP_FILE" | wc -l)
            echo "Objects in dump: $OBJECTS"
        else
            echo "FAIL: Invalid custom format dump"
            exit 1
        fi
        ;;
    *.tar)
        # Tar format
        if tar -tf "$BACKUP_FILE" > /dev/null 2>&1; then
            echo "OK: Valid tar archive"
        else
            echo "FAIL: Invalid tar archive"
            exit 1
        fi
        ;;
    *.gz)
        # Gzipped
        if gzip -t "$BACKUP_FILE" 2>&1; then
            echo "OK: Valid gzip archive"
        else
            echo "FAIL: Corrupted gzip archive"
            exit 1
        fi
        ;;
esac

echo "Basic validation: PASSED"
```

### Проверка списка объектов (TOC)

```bash
#!/bin/bash
# validate_pgdump_toc.sh

BACKUP_FILE=$1

echo "=== TOC Validation ==="

# Получить список объектов
pg_restore -l "$BACKUP_FILE" > /tmp/toc_$$.txt 2>&1

if [ $? -ne 0 ]; then
    echo "FAIL: Cannot read TOC"
    exit 1
fi

# Анализ TOC
TOTAL=$(grep -c "^[0-9]" /tmp/toc_$$.txt)
TABLES=$(grep -c "TABLE" /tmp/toc_$$.txt)
INDEXES=$(grep -c "INDEX" /tmp/toc_$$.txt)
CONSTRAINTS=$(grep -c "CONSTRAINT" /tmp/toc_$$.txt)
FUNCTIONS=$(grep -c "FUNCTION" /tmp/toc_$$.txt)
DATA=$(grep -c "TABLE DATA" /tmp/toc_$$.txt)

echo "Total objects: $TOTAL"
echo "  Tables: $TABLES"
echo "  Indexes: $INDEXES"
echo "  Constraints: $CONSTRAINTS"
echo "  Functions: $FUNCTIONS"
echo "  Table data: $DATA"

# Проверка минимальных ожиданий
if [ "$TABLES" -eq 0 ]; then
    echo "WARN: No tables found in dump"
fi

if [ "$DATA" -eq 0 ]; then
    echo "WARN: No data found in dump (schema only?)"
fi

rm /tmp/toc_$$.txt
echo "TOC validation: PASSED"
```

### Тестовое восстановление

```bash
#!/bin/bash
# test_restore_pgdump.sh

BACKUP_FILE=$1
TEST_DB="restore_test_$(date +%Y%m%d_%H%M%S)"

echo "=== Full Restore Test ==="
echo "Backup: $BACKUP_FILE"
echo "Test DB: $TEST_DB"

# Создать тестовую базу
createdb "$TEST_DB"
if [ $? -ne 0 ]; then
    echo "FAIL: Cannot create test database"
    exit 1
fi

# Восстановить
START_TIME=$(date +%s)

case "$BACKUP_FILE" in
    *.sql)
        psql -d "$TEST_DB" -f "$BACKUP_FILE" > /tmp/restore_$$.log 2>&1
        ;;
    *)
        pg_restore -d "$TEST_DB" "$BACKUP_FILE" > /tmp/restore_$$.log 2>&1
        ;;
esac

RESTORE_STATUS=$?
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))

echo "Restore duration: ${DURATION} seconds"

# Проверить результат
if [ $RESTORE_STATUS -ne 0 ]; then
    echo "WARN: Restore completed with errors"
    grep -i "error" /tmp/restore_$$.log | head -10
fi

# Проверка данных
echo ""
echo "=== Data Verification ==="

TABLE_COUNT=$(psql -t -d "$TEST_DB" -c \
    "SELECT count(*) FROM information_schema.tables WHERE table_schema = 'public';")
echo "Tables: $TABLE_COUNT"

ROW_COUNT=$(psql -t -d "$TEST_DB" -c \
    "SELECT coalesce(sum(n_live_tup), 0) FROM pg_stat_user_tables;")
echo "Total rows: $ROW_COUNT"

# Проверка на ошибки
ERROR_COUNT=$(grep -c -i "error" /tmp/restore_$$.log 2>/dev/null || echo 0)
echo "Errors during restore: $ERROR_COUNT"

# Очистка
read -p "Delete test database? [Y/n] " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Nn]$ ]]; then
    dropdb "$TEST_DB"
    echo "Test database deleted"
fi

rm /tmp/restore_$$.log

if [ "$ERROR_COUNT" -eq 0 ]; then
    echo ""
    echo "Restore test: PASSED"
else
    echo ""
    echo "Restore test: COMPLETED WITH WARNINGS"
fi
```

## Проверка pg_basebackup бэкапов

### Проверка физического бэкапа

```bash
#!/bin/bash
# validate_basebackup.sh

BACKUP_DIR=$1

echo "=== Physical Backup Validation ==="

if [ ! -d "$BACKUP_DIR" ] && [ ! -f "$BACKUP_DIR" ]; then
    echo "FAIL: Backup not found: $BACKUP_DIR"
    exit 1
fi

# Для tar формата
if [ -f "$BACKUP_DIR/base.tar" ] || [ -f "$BACKUP_DIR/base.tar.gz" ]; then
    echo "Format: TAR"

    # Проверить целостность tar
    TAR_FILE=$(ls "$BACKUP_DIR"/base.tar* 2>/dev/null | head -1)
    if tar -tf "$TAR_FILE" > /dev/null 2>&1; then
        echo "OK: Base archive is valid"
    else
        echo "FAIL: Base archive is corrupted"
        exit 1
    fi

    # Проверить WAL archive
    WAL_FILE=$(ls "$BACKUP_DIR"/pg_wal.tar* 2>/dev/null | head -1)
    if [ -n "$WAL_FILE" ]; then
        if tar -tf "$WAL_FILE" > /dev/null 2>&1; then
            echo "OK: WAL archive is valid"
        else
            echo "FAIL: WAL archive is corrupted"
            exit 1
        fi
    fi
fi

# Для plain формата
if [ -d "$BACKUP_DIR/base" ] || [ -f "$BACKUP_DIR/PG_VERSION" ]; then
    echo "Format: PLAIN"

    # Проверить наличие критических файлов
    REQUIRED_FILES=("PG_VERSION" "pg_hba.conf" "postgresql.conf")

    for FILE in "${REQUIRED_FILES[@]}"; do
        if [ -f "$BACKUP_DIR/$FILE" ]; then
            echo "OK: $FILE exists"
        else
            echo "FAIL: $FILE is missing"
            exit 1
        fi
    done

    # Проверить версию PostgreSQL
    PG_VERSION=$(cat "$BACKUP_DIR/PG_VERSION")
    echo "PostgreSQL version: $PG_VERSION"

    # Проверить директорию base
    if [ -d "$BACKUP_DIR/base" ]; then
        DB_COUNT=$(ls "$BACKUP_DIR/base" | wc -l)
        echo "Database directories: $DB_COUNT"
    fi

    # Проверить наличие WAL
    if [ -d "$BACKUP_DIR/pg_wal" ]; then
        WAL_COUNT=$(ls "$BACKUP_DIR/pg_wal" 2>/dev/null | wc -l)
        echo "WAL segments: $WAL_COUNT"
    fi
fi

echo ""
echo "Basic validation: PASSED"
```

### Проверка с использованием pg_verifybackup (PostgreSQL 13+)

```bash
#!/bin/bash
# verify_backup_pg13.sh

BACKUP_DIR=$1

echo "=== pg_verifybackup Validation ==="

if ! command -v pg_verifybackup &> /dev/null; then
    echo "WARN: pg_verifybackup not available (PostgreSQL 13+ required)"
    exit 0
fi

# Запустить верификацию
pg_verifybackup "$BACKUP_DIR"

if [ $? -eq 0 ]; then
    echo "Verification: PASSED"
else
    echo "Verification: FAILED"
    exit 1
fi
```

### Тестовое восстановление физического бэкапа

```bash
#!/bin/bash
# test_restore_basebackup.sh

BACKUP_DIR=$1
TEST_DIR="/tmp/pg_restore_test_$$"
TEST_PORT=15432

echo "=== Physical Restore Test ==="
echo "Backup: $BACKUP_DIR"
echo "Test directory: $TEST_DIR"
echo "Test port: $TEST_PORT"

mkdir -p "$TEST_DIR"

# Распаковать или скопировать бэкап
if [ -f "$BACKUP_DIR/base.tar.gz" ]; then
    echo "Extracting tar.gz backup..."
    tar -xzf "$BACKUP_DIR/base.tar.gz" -C "$TEST_DIR"
    if [ -f "$BACKUP_DIR/pg_wal.tar.gz" ]; then
        mkdir -p "$TEST_DIR/pg_wal"
        tar -xzf "$BACKUP_DIR/pg_wal.tar.gz" -C "$TEST_DIR/pg_wal"
    fi
elif [ -f "$BACKUP_DIR/base.tar" ]; then
    echo "Extracting tar backup..."
    tar -xf "$BACKUP_DIR/base.tar" -C "$TEST_DIR"
    if [ -f "$BACKUP_DIR/pg_wal.tar" ]; then
        mkdir -p "$TEST_DIR/pg_wal"
        tar -xf "$BACKUP_DIR/pg_wal.tar" -C "$TEST_DIR/pg_wal"
    fi
else
    echo "Copying plain backup..."
    cp -a "$BACKUP_DIR/"* "$TEST_DIR/"
fi

# Удалить recovery файлы если есть
rm -f "$TEST_DIR/standby.signal"
rm -f "$TEST_DIR/recovery.signal"

# Установить права
chmod 700 "$TEST_DIR"

# Изменить порт в конфигурации
echo "port = $TEST_PORT" >> "$TEST_DIR/postgresql.auto.conf"

# Запустить PostgreSQL
echo "Starting test instance..."
pg_ctl -D "$TEST_DIR" -o "-p $TEST_PORT" -l "$TEST_DIR/postgresql.log" start

if [ $? -ne 0 ]; then
    echo "FAIL: Cannot start PostgreSQL"
    cat "$TEST_DIR/postgresql.log"
    rm -rf "$TEST_DIR"
    exit 1
fi

# Дождаться запуска
sleep 5

# Проверить подключение
if pg_isready -p $TEST_PORT; then
    echo "OK: PostgreSQL is running"
else
    echo "FAIL: PostgreSQL not responding"
    pg_ctl -D "$TEST_DIR" stop
    rm -rf "$TEST_DIR"
    exit 1
fi

# Базовые проверки
echo ""
echo "=== Data Verification ==="

DB_LIST=$(psql -p $TEST_PORT -t -c "SELECT datname FROM pg_database WHERE datistemplate = false;")
echo "Databases: $DB_LIST"

for DB in $DB_LIST; do
    TABLE_COUNT=$(psql -p $TEST_PORT -d "$DB" -t -c \
        "SELECT count(*) FROM information_schema.tables WHERE table_schema = 'public';" 2>/dev/null)
    echo "  $DB: $TABLE_COUNT tables"
done

# Остановить тестовый инстанс
echo ""
echo "Stopping test instance..."
pg_ctl -D "$TEST_DIR" -m fast stop

# Очистка
read -p "Delete test directory? [Y/n] " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Nn]$ ]]; then
    rm -rf "$TEST_DIR"
    echo "Test directory deleted"
fi

echo ""
echo "Restore test: PASSED"
```

## Проверка WAL-G и pgBackRest бэкапов

### Проверка WAL-G бэкапа

```bash
#!/bin/bash
# validate_walg.sh

export WALG_S3_PREFIX=${WALG_S3_PREFIX:-"s3://bucket/path"}

echo "=== WAL-G Backup Validation ==="

# Список бэкапов
echo "Available backups:"
wal-g backup-list

# Получить последний бэкап
LATEST=$(wal-g backup-list --json | jq -r '.[-1].backup_name')
echo ""
echo "Latest backup: $LATEST"

# Проверить детали
wal-g backup-list --detail | grep "$LATEST"

# Тестовое восстановление
TEST_DIR="/tmp/walg_test_$$"
mkdir -p "$TEST_DIR"

echo ""
echo "Testing restore to $TEST_DIR..."

wal-g backup-fetch "$TEST_DIR" LATEST

if [ $? -eq 0 ]; then
    echo "OK: Backup fetch successful"

    # Проверить файлы
    if [ -f "$TEST_DIR/PG_VERSION" ]; then
        echo "OK: PG_VERSION exists"
        cat "$TEST_DIR/PG_VERSION"
    else
        echo "FAIL: PG_VERSION missing"
    fi
else
    echo "FAIL: Backup fetch failed"
    rm -rf "$TEST_DIR"
    exit 1
fi

# Очистка
rm -rf "$TEST_DIR"

echo ""
echo "WAL-G validation: PASSED"
```

### Проверка pgBackRest бэкапа

```bash
#!/bin/bash
# validate_pgbackrest.sh

STANZA=${1:-"main"}

echo "=== pgBackRest Backup Validation ==="

# Проверка конфигурации
echo "Checking configuration..."
pgbackrest --stanza=$STANZA check

if [ $? -ne 0 ]; then
    echo "FAIL: Configuration check failed"
    exit 1
fi

echo ""
echo "Backup information:"
pgbackrest --stanza=$STANZA info

# Верификация последнего бэкапа
echo ""
echo "Verifying backup integrity..."
pgbackrest --stanza=$STANZA verify

if [ $? -eq 0 ]; then
    echo "Backup verification: PASSED"
else
    echo "Backup verification: FAILED"
    exit 1
fi
```

### Полное тестовое восстановление pgBackRest

```bash
#!/bin/bash
# test_restore_pgbackrest.sh

STANZA="main"
TEST_DIR="/tmp/pgbackrest_test_$$"
TEST_PORT=15432

echo "=== pgBackRest Full Restore Test ==="

mkdir -p "$TEST_DIR"

# Восстановить
echo "Restoring to $TEST_DIR..."
pgbackrest --stanza=$STANZA \
    --pg1-path="$TEST_DIR" \
    --delta \
    restore

if [ $? -ne 0 ]; then
    echo "FAIL: Restore failed"
    rm -rf "$TEST_DIR"
    exit 1
fi

# Удалить recovery файлы
rm -f "$TEST_DIR/standby.signal"
rm -f "$TEST_DIR/recovery.signal"

# Настроить порт
echo "port = $TEST_PORT" >> "$TEST_DIR/postgresql.auto.conf"

# Запустить
pg_ctl -D "$TEST_DIR" -o "-p $TEST_PORT" -l "$TEST_DIR/postgresql.log" start

sleep 5

if pg_isready -p $TEST_PORT; then
    echo "OK: PostgreSQL started successfully"

    # Проверка данных
    TABLE_COUNT=$(psql -p $TEST_PORT -t -c \
        "SELECT count(*) FROM information_schema.tables WHERE table_schema = 'public';")
    echo "Tables: $TABLE_COUNT"

    # Остановить
    pg_ctl -D "$TEST_DIR" -m fast stop
else
    echo "FAIL: PostgreSQL did not start"
    cat "$TEST_DIR/postgresql.log"
fi

# Очистка
rm -rf "$TEST_DIR"

echo ""
echo "pgBackRest restore test: COMPLETED"
```

## Автоматизация проверки бэкапов

### Ежедневная автоматическая проверка

```bash
#!/bin/bash
# daily_backup_validation.sh

set -e

LOG_FILE="/var/log/backup_validation.log"
ALERT_EMAIL="admin@example.com"
BACKUP_DIR="/var/backups/postgres"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

send_alert() {
    echo "$1" | mail -s "Backup Validation Alert" "$ALERT_EMAIL"
}

log "=== Starting daily backup validation ==="

# Найти последний бэкап
LATEST_BACKUP=$(ls -t "$BACKUP_DIR"/*.dump 2>/dev/null | head -1)

if [ -z "$LATEST_BACKUP" ]; then
    log "CRITICAL: No backups found!"
    send_alert "No PostgreSQL backups found in $BACKUP_DIR"
    exit 1
fi

log "Validating: $LATEST_BACKUP"

# Базовая проверка
if ! pg_restore -l "$LATEST_BACKUP" > /dev/null 2>&1; then
    log "CRITICAL: Backup is corrupted!"
    send_alert "PostgreSQL backup is corrupted: $LATEST_BACKUP"
    exit 1
fi

log "Basic validation: PASSED"

# Проверка возраста бэкапа
BACKUP_AGE=$(( ($(date +%s) - $(stat -c %Y "$LATEST_BACKUP")) / 3600 ))
if [ "$BACKUP_AGE" -gt 25 ]; then
    log "WARNING: Backup is $BACKUP_AGE hours old"
    send_alert "PostgreSQL backup is stale: $BACKUP_AGE hours old"
fi

# Тестовое восстановление (еженедельно)
DAY_OF_WEEK=$(date +%u)
if [ "$DAY_OF_WEEK" -eq 7 ]; then
    log "Performing weekly restore test..."

    TEST_DB="validation_test_$(date +%Y%m%d)"

    createdb "$TEST_DB"
    if pg_restore -d "$TEST_DB" "$LATEST_BACKUP" 2>&1 | tee -a "$LOG_FILE"; then
        log "Restore test: PASSED"
    else
        log "WARNING: Restore completed with errors"
    fi

    dropdb "$TEST_DB"
fi

log "=== Validation completed ==="
```

### Systemd timer для автоматической проверки

```bash
# /etc/systemd/system/backup-validation.service
[Unit]
Description=PostgreSQL Backup Validation
After=postgresql.service

[Service]
Type=oneshot
User=postgres
ExecStart=/usr/local/bin/daily_backup_validation.sh
StandardOutput=journal
StandardError=journal

# /etc/systemd/system/backup-validation.timer
[Unit]
Description=Daily Backup Validation

[Timer]
OnCalendar=*-*-* 06:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
# Активировать
sudo systemctl daemon-reload
sudo systemctl enable --now backup-validation.timer
```

## Проверка целостности данных

### Сравнение количества записей

```bash
#!/bin/bash
# compare_row_counts.sh

PRODUCTION_DB="production"
RESTORED_DB="restored_backup"

echo "=== Row Count Comparison ==="

# Получить список таблиц
TABLES=$(psql -t -d "$PRODUCTION_DB" -c \
    "SELECT schemaname || '.' || tablename FROM pg_tables WHERE schemaname = 'public';")

MISMATCH=0

for TABLE in $TABLES; do
    PROD_COUNT=$(psql -t -d "$PRODUCTION_DB" -c "SELECT count(*) FROM $TABLE;")
    REST_COUNT=$(psql -t -d "$RESTORED_DB" -c "SELECT count(*) FROM $TABLE;" 2>/dev/null || echo "N/A")

    if [ "$PROD_COUNT" != "$REST_COUNT" ]; then
        echo "MISMATCH: $TABLE - Production: $PROD_COUNT, Restored: $REST_COUNT"
        MISMATCH=$((MISMATCH + 1))
    fi
done

if [ "$MISMATCH" -eq 0 ]; then
    echo "All row counts match!"
else
    echo "Found $MISMATCH mismatches"
fi
```

### Проверка checksum данных

```bash
#!/bin/bash
# verify_data_checksum.sh

DB_NAME=$1
TABLE_NAME=$2

echo "=== Data Checksum Verification ==="

# Вычислить checksum данных таблицы
CHECKSUM=$(psql -t -d "$DB_NAME" -c "
SELECT md5(string_agg(row_data::text, ''))
FROM (
    SELECT * FROM $TABLE_NAME ORDER BY 1
) t(row_data);
")

echo "Table: $TABLE_NAME"
echo "Checksum: $CHECKSUM"
```

### Проверка внешних ключей

```sql
-- check_foreign_keys.sql
-- Найти "сиротские" записи

DO $$
DECLARE
    r RECORD;
    orphan_count INTEGER;
BEGIN
    FOR r IN
        SELECT
            tc.table_name,
            kcu.column_name,
            ccu.table_name AS foreign_table,
            ccu.column_name AS foreign_column
        FROM information_schema.table_constraints AS tc
        JOIN information_schema.key_column_usage AS kcu
            ON tc.constraint_name = kcu.constraint_name
        JOIN information_schema.constraint_column_usage AS ccu
            ON ccu.constraint_name = tc.constraint_name
        WHERE tc.constraint_type = 'FOREIGN KEY'
    LOOP
        EXECUTE format(
            'SELECT count(*) FROM %I WHERE %I IS NOT NULL AND %I NOT IN (SELECT %I FROM %I)',
            r.table_name, r.column_name, r.column_name,
            r.foreign_column, r.foreign_table
        ) INTO orphan_count;

        IF orphan_count > 0 THEN
            RAISE NOTICE 'Orphan records in %.%: %',
                r.table_name, r.column_name, orphan_count;
        END IF;
    END LOOP;
END $$;
```

## Мониторинг и алертинг

### Prometheus метрики

```yaml
# prometheus.yml scrape config
scrape_configs:
  - job_name: 'backup_validation'
    static_configs:
      - targets: ['localhost:9100']
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'backup_.*'
        action: keep
```

```bash
#!/bin/bash
# expose_backup_metrics.sh

METRICS_FILE="/var/lib/node_exporter/backup.prom"

# Время последнего бэкапа
LATEST_BACKUP_TIME=$(stat -c %Y /var/backups/postgres/*.dump 2>/dev/null | sort -rn | head -1)
CURRENT_TIME=$(date +%s)
BACKUP_AGE=$((CURRENT_TIME - LATEST_BACKUP_TIME))

# Размер последнего бэкапа
LATEST_BACKUP_SIZE=$(stat -c %s /var/backups/postgres/*.dump 2>/dev/null | sort -rn | head -1)

# Количество бэкапов
BACKUP_COUNT=$(ls /var/backups/postgres/*.dump 2>/dev/null | wc -l)

# Записать метрики
cat > "$METRICS_FILE" << EOF
# HELP backup_age_seconds Age of the latest backup in seconds
# TYPE backup_age_seconds gauge
backup_age_seconds $BACKUP_AGE

# HELP backup_size_bytes Size of the latest backup in bytes
# TYPE backup_size_bytes gauge
backup_size_bytes $LATEST_BACKUP_SIZE

# HELP backup_count Total number of backups
# TYPE backup_count gauge
backup_count $BACKUP_COUNT
EOF
```

### AlertManager правила

```yaml
# backup_alerts.yml
groups:
  - name: backup_alerts
    rules:
      - alert: BackupTooOld
        expr: backup_age_seconds > 90000  # 25 hours
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL backup is stale"
          description: "Last backup is {{ $value | humanizeDuration }} old"

      - alert: BackupSizeAnomaly
        expr: |
          abs(backup_size_bytes - avg_over_time(backup_size_bytes[7d])) >
          stddev_over_time(backup_size_bytes[7d]) * 3
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Backup size anomaly detected"
          description: "Current backup size significantly differs from average"

      - alert: NoBackupsFound
        expr: backup_count == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "No PostgreSQL backups found"
```

## Best Practices

### 1. Регулярное тестирование

```text
Рекомендуемый график:
- Базовая проверка: После каждого бэкапа
- Проверка TOC/метаданных: Ежедневно
- Тестовое восстановление: Еженедельно
- Полное восстановление + проверка данных: Ежемесячно
- DR-тестирование: Ежеквартально
```

### 2. Документирование процедур

```markdown
## Процедура проверки бэкапа

1. Проверить наличие и возраст бэкапа
2. Проверить целостность архива
3. Проверить список объектов (TOC)
4. Выполнить тестовое восстановление
5. Проверить количество записей в ключевых таблицах
6. Документировать результаты
```

### 3. Автоматизация

```bash
# Интеграция в CI/CD pipeline
# .github/workflows/backup-validation.yml

name: Backup Validation
on:
  schedule:
    - cron: '0 6 * * *'  # Ежедневно в 6:00

jobs:
  validate:
    runs-on: self-hosted
    steps:
      - name: Validate latest backup
        run: /usr/local/bin/validate_backup.sh

      - name: Notify on failure
        if: failure()
        uses: actions/slack-notify@v1
        with:
          status: failure
          message: 'Backup validation failed!'
```

### 4. Метрики RTO

```bash
#!/bin/bash
# measure_rto.sh

# Измерение реального времени восстановления

START_TIME=$(date +%s.%N)

# Выполнить полное восстановление
# ...

END_TIME=$(date +%s.%N)

RTO=$(echo "$END_TIME - $START_TIME" | bc)
echo "Actual RTO: $RTO seconds"

# Сравнить с целевым RTO
TARGET_RTO=3600  # 1 час
if (( $(echo "$RTO > $TARGET_RTO" | bc -l) )); then
    echo "WARNING: RTO exceeds target!"
fi
```

## Типичные проблемы и решения

### Проблема 1: Бэкап не восстанавливается

```bash
# Проверить версию PostgreSQL
pg_restore --version
psql -c "SELECT version();"

# Проверить права
ls -la /var/backups/postgres/

# Проверить место на диске
df -h

# Посмотреть детальные ошибки
pg_restore -v -d test_db backup.dump 2>&1 | grep -i error
```

### Проблема 2: Данные не совпадают

```sql
-- Проверить время бэкапа
-- В логах pg_dump/pg_restore должна быть метка времени

-- Проверить были ли изменения после бэкапа
SELECT * FROM pg_stat_user_tables WHERE last_vacuum > 'backup_time';
```

### Проблема 3: Долгое восстановление

```bash
# Оптимизировать параллельность
pg_restore -j 8 -d mydb backup.dump

# Отключить синхронный коммит
psql -c "ALTER SYSTEM SET synchronous_commit = off;"
# Восстановить
# Включить обратно
psql -c "ALTER SYSTEM SET synchronous_commit = on;"
```

## Заключение

Проверка бэкапов - это не опциональная практика, а обязательная часть стратегии резервного копирования. Ключевые принципы:

1. **Регулярность** - проверяйте каждый бэкап
2. **Автоматизация** - минимизируйте человеческий фактор
3. **Разнообразие** - используйте разные уровни проверки
4. **Мониторинг** - отслеживайте метрики и алерты
5. **Документация** - фиксируйте процедуры и результаты

Помните: бэкап существует только если он успешно восстанавливается.

---

[prev: 05-pgbackrest](./05-pgbackrest.md) | [next: 14-capacity-planning](../14-capacity-planning.md)
