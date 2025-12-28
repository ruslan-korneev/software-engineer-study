# pgBackRest - Продвинутый инструмент резервного копирования PostgreSQL

[prev: 04-wal-g](./04-wal-g.md) | [next: 06-backup-validation](./06-backup-validation.md)

---

## Введение

**pgBackRest** - это надежное и многофункциональное решение для резервного копирования и восстановления PostgreSQL. Он разработан специально для работы с большими базами данных и обеспечивает полный цикл управления бэкапами с поддержкой инкрементального копирования, параллельного выполнения и Point-in-Time Recovery.

### Ключевые особенности pgBackRest

- **Параллельный backup/restore** - использование нескольких ядер CPU
- **Инкрементальные бэкапы** - full, differential, incremental
- **Локальные и удаленные репозитории** - файловая система, S3, GCS, Azure
- **Дельта-восстановление** - только измененные файлы
- **Async WAL архивация** - буферизация для высокой нагрузки
- **Контроль целостности** - checksum verification
- **PITR** - Point-in-Time Recovery

### Архитектура

```
┌─────────────────┐     ┌─────────────────┐     ┌──────────────────┐
│   PostgreSQL    │────▶│   pgBackRest    │────▶│   Repository     │
│   Primary       │     │   (backup)      │     │   (local/S3/GCS) │
│                 │     │                 │     │                  │
│   Standby       │     │   (archive)     │     │   - backups      │
│                 │     │   (restore)     │     │   - WAL archive  │
└─────────────────┘     └─────────────────┘     └──────────────────┘
```

## Установка

### Ubuntu/Debian

```bash
# Добавить репозиторий PostgreSQL
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# Установить pgBackRest
sudo apt-get update
sudo apt-get install pgbackrest

# Проверить версию
pgbackrest version
```

### CentOS/RHEL

```bash
# Установить репозиторий PGDG
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Установить pgBackRest
sudo yum install pgbackrest

# Проверить
pgbackrest version
```

### Из исходников

```bash
# Установить зависимости
sudo apt-get install build-essential libssl-dev libxml2-dev \
    libpq-dev liblz4-dev libzstd-dev libbz2-dev

# Скачать и распаковать
wget https://github.com/pgbackrest/pgbackrest/archive/release/2.48.tar.gz
tar xzf 2.48.tar.gz
cd pgbackrest-release-2.48/src

# Собрать
./configure
make

# Установить
sudo make install
```

## Конфигурация

### Структура конфигурационного файла

```ini
# /etc/pgbackrest/pgbackrest.conf

[global]
# Настройки репозитория
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
repo1-retention-diff=4

# Сжатие
compress-type=zst
compress-level=3

# Параллелизм
process-max=4

# Логирование
log-level-console=info
log-level-file=detail

[main]
# Stanza (конфигурация для конкретной БД)
pg1-path=/var/lib/postgresql/14/main
pg1-port=5432
pg1-user=postgres
```

### Минимальная конфигурация

```ini
# /etc/pgbackrest/pgbackrest.conf

[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2

[main]
pg1-path=/var/lib/postgresql/14/main
```

### Конфигурация для S3

```ini
[global]
repo1-type=s3
repo1-path=/backups
repo1-s3-bucket=my-backup-bucket
repo1-s3-endpoint=s3.amazonaws.com
repo1-s3-region=us-east-1
repo1-s3-key=AKIAIOSFODNN7EXAMPLE
repo1-s3-key-secret=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

repo1-retention-full=4
repo1-retention-diff=7

compress-type=zst
compress-level=3

[main]
pg1-path=/var/lib/postgresql/14/main
```

### Конфигурация для GCS

```ini
[global]
repo1-type=gcs
repo1-path=/backups
repo1-gcs-bucket=my-backup-bucket
repo1-gcs-key=/path/to/service-account.json

repo1-retention-full=4
repo1-retention-diff=7

[main]
pg1-path=/var/lib/postgresql/14/main
```

### Конфигурация для нескольких репозиториев

```ini
[global]
# Локальный репозиторий
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
repo1-retention-diff=4

# Удаленный репозиторий (S3)
repo2-type=s3
repo2-path=/backups
repo2-s3-bucket=offsite-backups
repo2-s3-region=eu-west-1
repo2-s3-key=...
repo2-s3-key-secret=...
repo2-retention-full=4
repo2-retention-diff=7

[main]
pg1-path=/var/lib/postgresql/14/main
```

## Настройка PostgreSQL

### postgresql.conf

```bash
# Архивация WAL
archive_mode = on
archive_command = 'pgbackrest --stanza=main archive-push %p'

# Для PITR
wal_level = replica
max_wal_senders = 3

# Опционально: async архивация для высокой нагрузки
# archive_command = 'pgbackrest --stanza=main archive-push --archive-async %p'
```

### Инициализация stanza

```bash
# Создать директорию репозитория
sudo mkdir -p /var/lib/pgbackrest
sudo chown postgres:postgres /var/lib/pgbackrest

# Инициализировать stanza
sudo -u postgres pgbackrest --stanza=main stanza-create

# Проверить конфигурацию
sudo -u postgres pgbackrest --stanza=main check
```

## Основные команды

### Создание бэкапов

```bash
# Полный бэкап
pgbackrest --stanza=main --type=full backup

# Дифференциальный бэкап (изменения с последнего полного)
pgbackrest --stanza=main --type=diff backup

# Инкрементальный бэкап (изменения с последнего любого бэкапа)
pgbackrest --stanza=main --type=incr backup

# Бэкап с указанием репозитория
pgbackrest --stanza=main --repo=2 --type=full backup
```

### Просмотр информации

```bash
# Список всех бэкапов
pgbackrest --stanza=main info

# Подробная информация
pgbackrest --stanza=main info --output=json

# Пример вывода:
# stanza: main
#     status: ok
#     cipher: none
#
#     db (current)
#         wal archive min/max (14): 000000010000000000000001/000000010000000000000003
#
#         full backup: 20240115-120000F
#             timestamp start/stop: 2024-01-15 12:00:00 / 2024-01-15 12:15:00
#             wal start/stop: 000000010000000000000001 / 000000010000000000000001
#             database size: 1GB, database backup size: 1GB
#             repo1: backup set size: 100MB, backup size: 100MB
```

### Восстановление

```bash
# Восстановление последнего бэкапа
pgbackrest --stanza=main restore

# Восстановление в конкретную директорию
pgbackrest --stanza=main --pg1-path=/var/lib/postgresql/14/restore restore

# PITR - восстановление на определенное время
pgbackrest --stanza=main --type=time \
    --target="2024-01-15 14:30:00" restore

# PITR - восстановление до конкретной транзакции
pgbackrest --stanza=main --type=xid \
    --target="1234567" restore

# PITR - восстановление до именованной точки
pgbackrest --stanza=main --type=name \
    --target="before_migration" restore

# Delta restore (только изменившиеся файлы)
pgbackrest --stanza=main --delta restore
```

### Архивация WAL

```bash
# Ручная архивация WAL сегмента
pgbackrest --stanza=main archive-push /path/to/wal/segment

# Получение WAL сегмента
pgbackrest --stanza=main archive-get 000000010000000000000001 /path/to/destination

# Проверка архива WAL
pgbackrest --stanza=main archive-get --check
```

### Управление бэкапами

```bash
# Удаление устаревших бэкапов (согласно retention policy)
pgbackrest --stanza=main expire

# Принудительное удаление бэкапа
pgbackrest --stanza=main expire --set=20240115-120000F

# Проверка конфигурации и состояния
pgbackrest --stanza=main check
```

## Типы бэкапов

### Full Backup

```bash
# Полная копия всех данных
pgbackrest --stanza=main --type=full backup

# Характеристики:
# - Независимый, не требует других бэкапов
# - Самый большой размер
# - Самое долгое время создания
# - Рекомендуется еженедельно
```

### Differential Backup

```bash
# Изменения с последнего FULL бэкапа
pgbackrest --stanza=main --type=diff backup

# Характеристики:
# - Требует только последний full для восстановления
# - Средний размер (накапливается со временем)
# - Среднее время создания
# - Рекомендуется ежедневно
```

### Incremental Backup

```bash
# Изменения с последнего ЛЮБОГО бэкапа
pgbackrest --stanza=main --type=incr backup

# Характеристики:
# - Требует цепочку: full -> diff/incr -> ... -> incr
# - Минимальный размер
# - Минимальное время создания
# - Рекомендуется несколько раз в день
```

### Стратегия бэкапов

```
Неделя:
  Пн: FULL
  Вт: DIFF
  Ср: INCR INCR INCR INCR
  Чт: DIFF
  Пт: INCR INCR INCR INCR
  Сб: DIFF
  Вс: INCR INCR

Восстановление из Пт INCR потребует: Пн FULL + Чт DIFF + все Пт INCR до нужного
```

## Практические сценарии

### Сценарий 1: Настройка локального бэкапа

```bash
#!/bin/bash
# setup_local_backup.sh

# 1. Создать конфигурацию
sudo tee /etc/pgbackrest/pgbackrest.conf << 'EOF'
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
repo1-retention-diff=7

compress-type=zst
compress-level=3
process-max=4

log-level-console=info
log-level-file=detail
log-path=/var/log/pgbackrest

[main]
pg1-path=/var/lib/postgresql/14/main
pg1-port=5432
EOF

# 2. Создать директории
sudo mkdir -p /var/lib/pgbackrest /var/log/pgbackrest
sudo chown -R postgres:postgres /var/lib/pgbackrest /var/log/pgbackrest

# 3. Инициализировать stanza
sudo -u postgres pgbackrest --stanza=main stanza-create

# 4. Настроить PostgreSQL
sudo -u postgres psql -c "ALTER SYSTEM SET archive_mode = 'on';"
sudo -u postgres psql -c "ALTER SYSTEM SET archive_command = 'pgbackrest --stanza=main archive-push %p';"

# 5. Перезапустить PostgreSQL
sudo systemctl restart postgresql

# 6. Проверить
sudo -u postgres pgbackrest --stanza=main check

# 7. Создать первый полный бэкап
sudo -u postgres pgbackrest --stanza=main --type=full backup
```

### Сценарий 2: Автоматизация через cron/systemd

```bash
# Crontab
# /etc/cron.d/pgbackrest

# Полный бэкап каждое воскресенье в 2:00
0 2 * * 0 postgres pgbackrest --stanza=main --type=full backup

# Дифференциальный бэкап каждый день (кроме воскресенья) в 2:00
0 2 * * 1-6 postgres pgbackrest --stanza=main --type=diff backup

# Инкрементальный бэкап каждые 6 часов
0 */6 * * * postgres pgbackrest --stanza=main --type=incr backup

# Очистка устаревших бэкапов
0 3 * * * postgres pgbackrest --stanza=main expire
```

```bash
# Systemd timer
# /etc/systemd/system/pgbackrest-full.timer
[Unit]
Description=Weekly pgBackRest Full Backup

[Timer]
OnCalendar=Sun 02:00
Persistent=true

[Install]
WantedBy=timers.target

# /etc/systemd/system/pgbackrest-full.service
[Unit]
Description=pgBackRest Full Backup
After=postgresql.service

[Service]
Type=oneshot
User=postgres
ExecStart=/usr/bin/pgbackrest --stanza=main --type=full backup
```

### Сценарий 3: Полное восстановление

```bash
#!/bin/bash
# full_restore.sh

STANZA="main"
PGDATA="/var/lib/postgresql/14/main"

echo "=== Full Database Restore ==="

# 1. Остановить PostgreSQL
sudo systemctl stop postgresql

# 2. Очистить директорию данных
rm -rf ${PGDATA:?}/*

# 3. Восстановить
sudo -u postgres pgbackrest --stanza=$STANZA restore

# 4. Запустить PostgreSQL
sudo systemctl start postgresql

# 5. Проверить
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"

echo "Restore completed!"
```

### Сценарий 4: Point-in-Time Recovery

```bash
#!/bin/bash
# pitr_restore.sh

STANZA="main"
PGDATA="/var/lib/postgresql/14/main"
TARGET_TIME="2024-01-15 14:30:00"

echo "=== PITR to $TARGET_TIME ==="

# 1. Остановить PostgreSQL
sudo systemctl stop postgresql

# 2. Очистить директорию данных
rm -rf ${PGDATA:?}/*

# 3. Восстановить с PITR
sudo -u postgres pgbackrest --stanza=$STANZA \
    --type=time \
    --target="$TARGET_TIME" \
    --target-action=promote \
    restore

# 4. Запустить PostgreSQL
sudo systemctl start postgresql

# 5. Мониторинг recovery
echo "Monitoring recovery..."
while sudo -u postgres psql -t -c "SELECT pg_is_in_recovery();" | grep -q "t"; do
    echo "Recovery in progress..."
    sleep 5
done

echo "PITR completed!"
```

### Сценарий 5: Delta Restore (обновление standby)

```bash
#!/bin/bash
# delta_restore.sh

# Быстрое восстановление standby с использованием delta

STANZA="main"
PGDATA="/var/lib/postgresql/14/main"

# 1. Остановить PostgreSQL
sudo systemctl stop postgresql

# 2. Delta restore - только изменившиеся файлы
sudo -u postgres pgbackrest --stanza=$STANZA \
    --delta \
    --type=standby \
    restore

# 3. Запустить PostgreSQL
sudo systemctl start postgresql

echo "Delta restore completed!"
```

### Сценарий 6: Бэкап со standby сервера

```ini
# pgbackrest.conf на standby

[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2

# Параметры репликации
backup-standby=y

[main]
pg1-path=/var/lib/postgresql/14/main
pg1-port=5432

# Primary сервер для получения WAL
pg2-host=primary.example.com
pg2-path=/var/lib/postgresql/14/main
pg2-port=5432
```

```bash
# Бэкап со standby (снижает нагрузку на primary)
pgbackrest --stanza=main --type=full backup
```

### Сценарий 7: Репликация бэкапов между репозиториями

```ini
# pgbackrest.conf

[global]
# Локальный репозиторий
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2

# Удаленный репозиторий (offsite)
repo2-type=s3
repo2-path=/backups
repo2-s3-bucket=offsite-backups
repo2-s3-region=eu-west-1
repo2-s3-key=...
repo2-s3-key-secret=...
repo2-retention-full=4

[main]
pg1-path=/var/lib/postgresql/14/main
```

```bash
# Бэкап в оба репозитория
pgbackrest --stanza=main --type=full backup

# Восстановление из удаленного репозитория
pgbackrest --stanza=main --repo=2 restore
```

## Расширенные возможности

### Async WAL Archiving

```ini
# pgbackrest.conf
[global]
archive-async=y
spool-path=/var/spool/pgbackrest
```

```bash
# postgresql.conf
archive_command = 'pgbackrest --stanza=main archive-push --archive-async %p'
```

### Параллельное выполнение

```ini
[global]
# Количество процессов для backup/restore
process-max=8

# Количество процессов для сжатия
compress-level-network=3
```

### Шифрование

```ini
[global]
# Тип шифрования
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=your-encryption-passphrase

# Или из файла
# repo1-cipher-pass=/path/to/passphrase-file
```

### Настройка retention

```ini
[global]
# Хранить 2 полных бэкапа
repo1-retention-full=2

# Хранить 7 дифференциальных бэкапов
repo1-retention-diff=7

# Retention type: count (по количеству) или time (по времени)
repo1-retention-full-type=count

# Или хранить бэкапы за 30 дней
# repo1-retention-full-type=time
# repo1-retention-full=30
```

### Bundle файлы (оптимизация для маленьких файлов)

```ini
[global]
# Группировать маленькие файлы в bundles
repo1-bundle=y

# Размер bundle
repo1-bundle-size=2MB
```

### Block Incremental (PostgreSQL 17+)

```ini
[global]
# Использовать block-level incremental
repo1-block=y
```

## Мониторинг

### Проверка состояния

```bash
# Проверка stanza
pgbackrest --stanza=main check

# Информация о бэкапах
pgbackrest --stanza=main info

# JSON вывод для парсинга
pgbackrest --stanza=main info --output=json | jq '.'
```

### Интеграция с мониторингом

```bash
#!/bin/bash
# check_pgbackrest.sh - для Nagios/Zabbix

STANZA="main"
MAX_AGE_HOURS=25

# Получить информацию о последнем бэкапе
LAST_BACKUP=$(pgbackrest --stanza=$STANZA info --output=json | \
    jq -r '.[0].backup[-1].timestamp.stop')

if [ -z "$LAST_BACKUP" ] || [ "$LAST_BACKUP" == "null" ]; then
    echo "CRITICAL: No backups found"
    exit 2
fi

# Вычислить возраст
BACKUP_EPOCH=$(date -d "$LAST_BACKUP" +%s)
CURRENT_EPOCH=$(date +%s)
AGE_HOURS=$(( (CURRENT_EPOCH - BACKUP_EPOCH) / 3600 ))

if [ $AGE_HOURS -gt $MAX_AGE_HOURS ]; then
    echo "WARNING: Last backup is $AGE_HOURS hours old"
    exit 1
else
    echo "OK: Last backup is $AGE_HOURS hours old"
    exit 0
fi
```

### Prometheus exporter

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'pgbackrest'
    static_configs:
      - targets: ['localhost:9854']
```

## Best Practices

### 1. Планирование бэкапов

```
Рекомендуемая стратегия:
- Full: 1 раз в неделю
- Diff: 1 раз в день
- Incr: каждые 4-6 часов (для критичных систем)

Retention:
- Full: 2-4 бэкапа
- Diff: 7-14 бэкапов
- WAL: покрывать период между full бэкапами
```

### 2. Тестирование восстановления

```bash
#!/bin/bash
# test_restore.sh

TEST_DB="restore_test_$(date +%Y%m%d)"
TEST_DIR="/tmp/$TEST_DB"

mkdir -p $TEST_DIR

# Восстановить в тестовую директорию
pgbackrest --stanza=main \
    --pg1-path=$TEST_DIR \
    --delta \
    restore

# Запустить тестовый инстанс
pg_ctl -D $TEST_DIR -o "-p 5433" start

# Проверить
psql -p 5433 -c "SELECT count(*) FROM pg_tables;"

# Остановить и очистить
pg_ctl -D $TEST_DIR stop
rm -rf $TEST_DIR
```

### 3. Безопасность

```ini
# Использовать шифрование
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass-file=/secure/path/passphrase

# Ограничить доступ к конфигурации
# chmod 600 /etc/pgbackrest/pgbackrest.conf
# chown postgres:postgres /etc/pgbackrest/pgbackrest.conf
```

### 4. Логирование

```ini
[global]
log-level-console=info
log-level-file=detail
log-path=/var/log/pgbackrest
log-timestamp=y
```

## Типичные проблемы и решения

### Проблема 1: "unable to find primary cluster"

```bash
# Убедиться что PostgreSQL запущен
pg_isready

# Проверить путь в конфигурации
# pg1-path должен указывать на PGDATA
```

### Проблема 2: "WAL segment not found"

```bash
# Проверить archive_command
SHOW archive_command;

# Убедиться что WAL архивируется
ls -la /var/lib/pgbackrest/archive/main/

# Проверить логи
tail -f /var/log/pgbackrest/main-archive-push.log
```

### Проблема 3: Медленное восстановление

```bash
# Увеличить параллелизм
pgbackrest --stanza=main --process-max=16 restore

# Использовать delta restore
pgbackrest --stanza=main --delta restore
```

### Проблема 4: Большой размер репозитория

```bash
# Запустить очистку
pgbackrest --stanza=main expire

# Уменьшить retention
# В pgbackrest.conf:
# repo1-retention-full=1
# repo1-retention-diff=3

# Включить bundling для маленьких файлов
# repo1-bundle=y
```

## Сравнение с другими инструментами

| Характеристика | pgBackRest | WAL-G | pg_basebackup |
|---------------|------------|-------|---------------|
| Инкрементальные | Full/Diff/Incr | Delta | Нет |
| Параллельный backup | Да | Да | Нет |
| Параллельный restore | Да | Да | Нет |
| Delta restore | Да | Нет | Нет |
| Async WAL | Да | Нет | N/A |
| Облачные хранилища | S3/GCS/Azure | S3/GCS/Azure | Нет |
| Шифрование | AES-256 | PGP/libsodium | Нет |
| Конфигурация | Файл | Env vars | CLI args |

## Заключение

pgBackRest - это enterprise-grade решение для резервного копирования PostgreSQL, которое обеспечивает:

- Надежное и эффективное создание бэкапов
- Гибкие стратегии инкрементального копирования
- Быстрое восстановление с delta-функциональностью
- Отличную масштабируемость для больших баз данных

Рекомендуется для production-систем, особенно для крупных баз данных с высокими требованиями к RTO/RPO.

---

[prev: 04-wal-g](./04-wal-g.md) | [next: 06-backup-validation](./06-backup-validation.md)
