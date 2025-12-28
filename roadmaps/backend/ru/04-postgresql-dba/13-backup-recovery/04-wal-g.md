# WAL-G - Инструмент для WAL архивации и бэкапов PostgreSQL

[prev: 03-pg_basebackup](./03-pg_basebackup.md) | [next: 05-pgbackrest](./05-pgbackrest.md)

---

## Введение

**WAL-G** - это современный инструмент для архивного резервного копирования PostgreSQL (и других СУБД), разработанный как преемник WAL-E. Он обеспечивает эффективное создание бэкапов с поддержкой множества облачных хранилищ и возможностью Point-in-Time Recovery (PITR).

### Ключевые особенности WAL-G

- **Множество хранилищ** - S3, GCS, Azure, Swift, файловая система
- **Инкрементальные бэкапы** - delta backups для экономии места
- **Параллельное выполнение** - многопоточная загрузка/выгрузка
- **Сжатие** - LZ4, LZMA, Brotli, ZStd
- **Шифрование** - AES-256, libsodium
- **PITR** - Point-in-Time Recovery через WAL архивацию
- **Copy-on-Write** - эффективное использование дискового пространства

### Архитектура WAL-G

```
┌─────────────────┐     ┌─────────────────┐     ┌──────────────────┐
│   PostgreSQL    │────▶│     WAL-G       │────▶│   Object Store   │
│                 │     │                 │     │  (S3/GCS/Azure)  │
│  - Data files   │     │  - Compression  │     │                  │
│  - WAL files    │     │  - Encryption   │     │  - base backups  │
│                 │     │  - Parallelism  │     │  - WAL segments  │
└─────────────────┘     └─────────────────┘     └──────────────────┘
```

## Установка

### Установка из бинарных файлов

```bash
# Скачать последнюю версию
curl -L "https://github.com/wal-g/wal-g/releases/latest/download/wal-g-pg-ubuntu-20.04-amd64.tar.gz" \
  -o wal-g.tar.gz

# Распаковать
tar -xzf wal-g.tar.gz

# Установить
sudo mv wal-g-pg-ubuntu-20.04-amd64 /usr/local/bin/wal-g
sudo chmod +x /usr/local/bin/wal-g

# Проверить установку
wal-g --version
```

### Установка через Docker

```bash
# Использование официального образа
docker run --rm wal-g/wal-g:latest --version

# В docker-compose
services:
  postgres:
    image: postgres:15
    volumes:
      - ./wal-g:/usr/local/bin/wal-g
    environment:
      - WALG_S3_PREFIX=s3://bucket/path
```

### Сборка из исходников

```bash
# Установить зависимости
sudo apt-get install golang liblzo2-dev libsodium-dev

# Клонировать репозиторий
git clone https://github.com/wal-g/wal-g.git
cd wal-g

# Собрать для PostgreSQL
make pg_build

# Установить
sudo mv main/pg/wal-g /usr/local/bin/
```

## Конфигурация

### Переменные окружения (основной способ)

```bash
# Основные настройки хранилища

# AWS S3
export WALG_S3_PREFIX=s3://bucket-name/path/to/backups
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
export AWS_REGION=us-east-1

# Google Cloud Storage
export WALG_GS_PREFIX=gs://bucket-name/path/to/backups
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/credentials.json

# Azure Blob Storage
export WALG_AZ_PREFIX=azure://container-name/path/to/backups
export AZURE_STORAGE_ACCOUNT=account_name
export AZURE_STORAGE_ACCESS_KEY=access_key

# Локальная файловая система
export WALG_FILE_PREFIX=/var/backups/postgres

# S3-совместимое хранилище (MinIO, Yandex Cloud, и т.д.)
export WALG_S3_PREFIX=s3://bucket/backups
export AWS_ENDPOINT=https://storage.yandexcloud.net
export AWS_S3_FORCE_PATH_STYLE=true
```

### Настройки сжатия и шифрования

```bash
# Сжатие (lz4, lzma, brotli, zstd)
export WALG_COMPRESSION_METHOD=zstd

# Шифрование
export WALG_PGP_KEY_PATH=/path/to/public.key  # PGP
# или
export WALG_LIBSODIUM_KEY=your_base64_encoded_key  # libsodium
# или
export WALG_LIBSODIUM_KEY_PATH=/path/to/key_file
```

### Настройки производительности

```bash
# Количество параллельных потоков загрузки
export WALG_UPLOAD_CONCURRENCY=16

# Количество потоков скачивания
export WALG_DOWNLOAD_CONCURRENCY=10

# Размер буфера загрузки
export WALG_UPLOAD_DISK_CONCURRENCY=1

# Использовать дельта-бэкапы
export WALG_DELTA_MAX_STEPS=6

# Сетевой rate limit (байт/сек)
export WALG_NETWORK_RATE_LIMIT=10485760  # 10 MB/s
```

### Конфигурационный файл

```bash
# /etc/wal-g/wal-g.yaml
WALG_S3_PREFIX: s3://my-bucket/postgres-backups
AWS_REGION: eu-west-1
WALG_COMPRESSION_METHOD: zstd
WALG_UPLOAD_CONCURRENCY: 16
WALG_DELTA_MAX_STEPS: 6
PGHOST: /var/run/postgresql
PGUSER: postgres
PGDATABASE: postgres
```

```bash
# Использование конфигурационного файла
export WALG_CONFIG=/etc/wal-g/wal-g.yaml
wal-g backup-push /var/lib/postgresql/14/main
```

## Настройка PostgreSQL

### postgresql.conf

```bash
# Включение архивации WAL
wal_level = replica
archive_mode = on
archive_command = 'wal-g wal-push %p'
archive_timeout = 60  # Принудительная архивация каждые 60 секунд

# Для PITR
max_wal_senders = 3
```

### Восстановление (recovery.conf / postgresql.conf)

```bash
# PostgreSQL 12+
restore_command = 'wal-g wal-fetch %f %p'

# Для PITR до определенного времени
recovery_target_time = '2024-01-15 14:30:00'
recovery_target_action = 'promote'
```

## Основные команды

### Создание бэкапов

```bash
# Полный бэкап (базовый бэкап)
wal-g backup-push /var/lib/postgresql/14/main

# С явным указанием хранилища
WALG_S3_PREFIX=s3://bucket/backups wal-g backup-push /var/lib/postgresql/14/main

# Дельта-бэкап (если настроен WALG_DELTA_MAX_STEPS)
# WAL-G автоматически создаст дельта, если есть предыдущий бэкап
wal-g backup-push /var/lib/postgresql/14/main

# С дополнительными метаданными
wal-g backup-push --add-user-data '{"env":"production"}' /var/lib/postgresql/14/main
```

### Архивация WAL

```bash
# Ручная архивация WAL сегмента
wal-g wal-push /var/lib/postgresql/14/main/pg_wal/000000010000000000000001

# Архивация всех накопившихся WAL
wal-g wal-push /var/lib/postgresql/14/main/pg_wal/
```

### Просмотр бэкапов

```bash
# Список всех бэкапов
wal-g backup-list

# Пример вывода:
# name                          modified             wal_segment_backup_start
# base_000000010000000000000010 2024-01-15T10:30:00Z 000000010000000000000010
# base_000000010000000000000020 2024-01-16T10:30:00Z 000000010000000000000020

# Подробная информация
wal-g backup-list --detail

# В формате JSON
wal-g backup-list --json
```

### Восстановление

```bash
# Восстановление последнего бэкапа
wal-g backup-fetch /var/lib/postgresql/14/main LATEST

# Восстановление конкретного бэкапа
wal-g backup-fetch /var/lib/postgresql/14/main base_000000010000000000000010

# Получение WAL сегмента
wal-g wal-fetch 000000010000000000000010 /var/lib/postgresql/14/main/pg_wal/000000010000000000000010
```

### Удаление бэкапов

```bash
# Удалить бэкапы старше определенного
wal-g delete before base_000000010000000000000020

# Удалить бэкапы, оставив N последних
wal-g delete retain FULL 5  # оставить 5 полных бэкапов

# С подтверждением
wal-g delete retain FULL 5 --confirm

# Удалить все бэкапы до определенной даты
wal-g delete before FIND_FULL 2024-01-01T00:00:00Z --confirm
```

## Практические сценарии

### Сценарий 1: Автоматизация бэкапов через systemd

```bash
# /etc/systemd/system/wal-g-backup.service
[Unit]
Description=WAL-G PostgreSQL Backup
After=postgresql.service

[Service]
Type=oneshot
User=postgres
Environment="WALG_S3_PREFIX=s3://my-bucket/postgres"
Environment="AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE"
Environment="AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG"
Environment="WALG_COMPRESSION_METHOD=zstd"
ExecStart=/usr/local/bin/wal-g backup-push /var/lib/postgresql/14/main

[Install]
WantedBy=multi-user.target
```

```bash
# /etc/systemd/system/wal-g-backup.timer
[Unit]
Description=Daily WAL-G Backup

[Timer]
OnCalendar=daily
Persistent=true
RandomizedDelaySec=3600

[Install]
WantedBy=timers.target
```

```bash
# Активировать таймер
sudo systemctl daemon-reload
sudo systemctl enable --now wal-g-backup.timer
```

### Сценарий 2: Скрипт полного бэкапа с очисткой

```bash
#!/bin/bash
# /usr/local/bin/wal-g-full-backup.sh

set -e

# Конфигурация
export WALG_S3_PREFIX=s3://my-bucket/postgres-backups
export AWS_REGION=us-east-1
export WALG_COMPRESSION_METHOD=zstd
export WALG_UPLOAD_CONCURRENCY=16
PGDATA=/var/lib/postgresql/14/main
RETAIN_FULL=7

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

log "Starting backup..."

# Создание бэкапа
wal-g backup-push $PGDATA

if [ $? -eq 0 ]; then
    log "Backup completed successfully"

    # Очистка старых бэкапов
    log "Cleaning old backups (retain $RETAIN_FULL)..."
    wal-g delete retain FULL $RETAIN_FULL --confirm

    # Отправка уведомления (опционально)
    # curl -X POST "https://hooks.slack.com/services/..." \
    #     -H "Content-Type: application/json" \
    #     -d '{"text":"PostgreSQL backup completed"}'

    log "Cleanup completed"
else
    log "Backup FAILED!"
    exit 1
fi
```

### Сценарий 3: Point-in-Time Recovery (PITR)

```bash
#!/bin/bash
# pitr_restore.sh

set -e

# Конфигурация
export WALG_S3_PREFIX=s3://my-bucket/postgres-backups
export AWS_REGION=us-east-1
PGDATA=/var/lib/postgresql/14/main
RECOVERY_TARGET_TIME="2024-01-15 14:30:00 UTC"

echo "=== Point-in-Time Recovery ==="
echo "Target time: $RECOVERY_TARGET_TIME"

# 1. Остановить PostgreSQL
sudo systemctl stop postgresql

# 2. Очистить директорию данных
rm -rf ${PGDATA:?}/*

# 3. Восстановить последний бэкап до целевого времени
wal-g backup-fetch $PGDATA LATEST

# 4. Создать recovery.signal
touch $PGDATA/recovery.signal

# 5. Настроить восстановление
cat >> $PGDATA/postgresql.auto.conf << EOF
restore_command = 'wal-g wal-fetch %f %p'
recovery_target_time = '$RECOVERY_TARGET_TIME'
recovery_target_action = 'promote'
EOF

# 6. Установить права
chown -R postgres:postgres $PGDATA
chmod 700 $PGDATA

# 7. Запустить PostgreSQL
sudo systemctl start postgresql

echo "Recovery initiated. Check logs: sudo journalctl -u postgresql -f"
```

### Сценарий 4: Миграция между регионами/облаками

```bash
#!/bin/bash
# migrate_backup.sh

# Источник
export SOURCE_WALG_S3_PREFIX=s3://source-bucket/postgres
export SOURCE_AWS_REGION=us-east-1

# Назначение
export DEST_WALG_S3_PREFIX=s3://dest-bucket/postgres
export DEST_AWS_REGION=eu-west-1

TEMP_DIR=/tmp/pg_migration

mkdir -p $TEMP_DIR

# 1. Скачать последний бэкап из источника
WALG_S3_PREFIX=$SOURCE_WALG_S3_PREFIX \
AWS_REGION=$SOURCE_AWS_REGION \
wal-g backup-fetch $TEMP_DIR LATEST

# 2. Загрузить в новое хранилище
WALG_S3_PREFIX=$DEST_WALG_S3_PREFIX \
AWS_REGION=$DEST_AWS_REGION \
wal-g backup-push $TEMP_DIR

# 3. Синхронизировать WAL (если нужно)
# Это сложнее и требует дополнительной логики

# 4. Очистить временные файлы
rm -rf $TEMP_DIR
```

### Сценарий 5: Мониторинг бэкапов

```bash
#!/bin/bash
# check_backup_freshness.sh

export WALG_S3_PREFIX=s3://my-bucket/postgres-backups
MAX_AGE_HOURS=25

# Получить время последнего бэкапа
LAST_BACKUP=$(wal-g backup-list --json | jq -r '.[-1].time')

if [ -z "$LAST_BACKUP" ] || [ "$LAST_BACKUP" == "null" ]; then
    echo "CRITICAL: No backups found!"
    exit 2
fi

# Вычислить возраст бэкапа
BACKUP_TIMESTAMP=$(date -d "$LAST_BACKUP" +%s)
CURRENT_TIMESTAMP=$(date +%s)
AGE_HOURS=$(( (CURRENT_TIMESTAMP - BACKUP_TIMESTAMP) / 3600 ))

if [ $AGE_HOURS -gt $MAX_AGE_HOURS ]; then
    echo "WARNING: Last backup is $AGE_HOURS hours old (threshold: $MAX_AGE_HOURS)"
    exit 1
else
    echo "OK: Last backup is $AGE_HOURS hours old"
    exit 0
fi
```

## Delta (инкрементальные) бэкапы

### Настройка delta бэкапов

```bash
# Максимальное количество шагов дельты до полного бэкапа
export WALG_DELTA_MAX_STEPS=6

# Первый бэкап всегда полный
wal-g backup-push $PGDATA  # FULL

# Последующие будут delta, если выгодно
wal-g backup-push $PGDATA  # DELTA from FULL
wal-g backup-push $PGDATA  # DELTA from DELTA
# ...
wal-g backup-push $PGDATA  # После 6 шагов снова FULL
```

### Принудительный полный бэкап

```bash
# Создать полный бэкап независимо от delta настроек
wal-g backup-push --full $PGDATA
```

### Просмотр типов бэкапов

```bash
# Показать delta-цепочки
wal-g backup-list --detail

# Пример вывода:
# name                          modified             wal_segment_backup_start  backup_type
# base_00000001000000000000010  2024-01-15T10:00:00Z 000000010000000000000010  full
# base_00000001000000000000020  2024-01-15T22:00:00Z 000000010000000000000020  delta
# base_00000001000000000000030  2024-01-16T10:00:00Z 000000010000000000000030  delta
```

## Шифрование

### PGP шифрование

```bash
# Генерация ключа
gpg --gen-key

# Экспорт публичного ключа
gpg --export -a "backup@example.com" > /etc/wal-g/public.key

# Настройка WAL-G
export WALG_PGP_KEY_PATH=/etc/wal-g/public.key

# Для расшифровки (при восстановлении)
export WALG_PGP_KEY_PATH=/etc/wal-g/private.key
# или через gpg agent
```

### Libsodium шифрование

```bash
# Генерация ключа
openssl rand -base64 32 > /etc/wal-g/encryption.key
chmod 600 /etc/wal-g/encryption.key

# Настройка
export WALG_LIBSODIUM_KEY_PATH=/etc/wal-g/encryption.key
# или
export WALG_LIBSODIUM_KEY=$(cat /etc/wal-g/encryption.key)
```

## Интеграция с облачными провайдерами

### AWS S3

```bash
# Минимальная конфигурация
export WALG_S3_PREFIX=s3://bucket/path
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_REGION=us-east-1

# С IAM ролью (EC2/EKS)
export WALG_S3_PREFIX=s3://bucket/path
export AWS_REGION=us-east-1
# Credentials через instance metadata

# Дополнительные опции
export AWS_S3_FORCE_PATH_STYLE=false
export WALG_S3_STORAGE_CLASS=STANDARD_IA  # или GLACIER
export WALG_S3_SSE=aws:kms
export WALG_S3_SSE_KMS_ID=key-id
```

### Google Cloud Storage

```bash
export WALG_GS_PREFIX=gs://bucket/path
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json
```

### Yandex Cloud Object Storage

```bash
export WALG_S3_PREFIX=s3://bucket/path
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_ENDPOINT=https://storage.yandexcloud.net
export AWS_REGION=ru-central1
export AWS_S3_FORCE_PATH_STYLE=true
```

### MinIO (self-hosted S3)

```bash
export WALG_S3_PREFIX=s3://bucket/path
export AWS_ACCESS_KEY_ID=minioadmin
export AWS_SECRET_ACCESS_KEY=minioadmin
export AWS_ENDPOINT=http://localhost:9000
export AWS_S3_FORCE_PATH_STYLE=true
```

## Best Practices

### 1. Конфигурация production

```bash
# Рекомендуемые настройки
export WALG_COMPRESSION_METHOD=zstd
export WALG_UPLOAD_CONCURRENCY=16
export WALG_DELTA_MAX_STEPS=6
export WALG_PREVENT_WAL_OVERWRITE=true

# PostgreSQL
archive_mode = on
archive_command = 'wal-g wal-push %p'
archive_timeout = 60
wal_level = replica
```

### 2. Безопасность

```bash
# Хранить credentials в безопасном месте
# Использовать IAM роли где возможно
# Включить шифрование
export WALG_LIBSODIUM_KEY_PATH=/secure/path/key

# Ограничить доступ к конфигурации
chmod 600 /etc/wal-g/wal-g.yaml
```

### 3. Мониторинг

```bash
# Проверять наличие свежих бэкапов
# Мониторить размер хранилища
# Логировать операции бэкапа

# Пример алерта для Prometheus/AlertManager
# alert: PostgresBackupMissing
#   expr: time() - postgres_backup_last_timestamp > 86400
#   for: 1h
#   labels:
#     severity: critical
```

### 4. Тестирование

```bash
# Регулярно тестировать восстановление
# Автоматизировать тестовое восстановление
# Проверять PITR на тестовой среде
```

## Типичные проблемы и решения

### Проблема 1: "failed to upload"

```bash
# Проверить credentials
aws s3 ls $WALG_S3_PREFIX

# Проверить права IAM
# Увеличить timeout
export WALG_UPLOAD_TIMEOUT=3600
```

### Проблема 2: "WAL segment not found"

```bash
# Проверить archive_command в PostgreSQL
SHOW archive_command;

# Проверить логи PostgreSQL
tail -f /var/log/postgresql/postgresql-14-main.log

# Убедиться что archive_mode = on
SHOW archive_mode;
```

### Проблема 3: Медленный бэкап

```bash
# Увеличить параллелизм
export WALG_UPLOAD_CONCURRENCY=32

# Использовать быстрое сжатие
export WALG_COMPRESSION_METHOD=lz4

# Проверить сетевую пропускную способность
```

## Заключение

WAL-G - мощный и гибкий инструмент для резервного копирования PostgreSQL, который особенно хорошо подходит для облачных окружений. Ключевые преимущества:

- Простая интеграция с облачными хранилищами
- Эффективные delta-бэкапы
- Встроенное сжатие и шифрование
- Поддержка PITR

Рекомендуется для production-систем, особенно при использовании облачной инфраструктуры.

---

[prev: 03-pg_basebackup](./03-pg_basebackup.md) | [next: 05-pgbackrest](./05-pgbackrest.md)
