# pg_basebackup - Физический бэкап PostgreSQL

## Введение

**pg_basebackup** - это встроенная утилита PostgreSQL для создания физических (бинарных) резервных копий всего кластера базы данных. В отличие от логического бэкапа (pg_dump), pg_basebackup копирует файлы данных напрямую, что обеспечивает быстрое восстановление и возможность Point-in-Time Recovery (PITR).

### Ключевые особенности pg_basebackup

- **Физический бэкап** - побайтовое копирование файлов данных
- **Консистентность** - гарантированно консистентный снимок через WAL
- **Быстрое восстановление** - не требует повторного выполнения SQL
- **PITR поддержка** - основа для Point-in-Time Recovery
- **Streaming** - получение данных через протокол репликации
- **Не блокирует** - минимальное влияние на работающую базу

### Сравнение с pg_dump

| Характеристика | pg_basebackup | pg_dump |
|---------------|---------------|---------|
| Тип бэкапа | Физический | Логический |
| Что копируется | Все файлы кластера | SQL-представление |
| Скорость бэкапа | Быстрее | Медленнее |
| Размер бэкапа | Больше | Меньше |
| Скорость восстановления | Очень быстро | Медленнее |
| PITR | Да (с WAL) | Нет |
| Кросс-версионность | Нет | Да |
| Выборочное восстановление | Нет | Да |

## Предварительная настройка

### Настройка PostgreSQL для репликации

```bash
# postgresql.conf
wal_level = replica              # минимум для pg_basebackup
max_wal_senders = 3              # количество одновременных подключений репликации
max_replication_slots = 3        # опционально, для надежности

# pg_hba.conf - разрешить подключения для репликации
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    replication     replication_user 192.168.1.0/24         scram-sha-256
local   replication     postgres                                peer
```

```bash
# Создание пользователя для репликации
psql -c "CREATE ROLE replication_user WITH REPLICATION LOGIN PASSWORD 'secure_password';"

# Перезагрузка конфигурации
sudo systemctl reload postgresql
```

## Базовое использование

### Простой бэкап

```bash
# Бэкап в директорию
pg_basebackup -D /var/backups/postgres/base_backup -Fp -Xs -P

# Бэкап в tar-архив
pg_basebackup -D /var/backups/postgres/base_backup -Ft -Xs -z -P

# С явным указанием хоста и пользователя
pg_basebackup -h localhost -p 5432 -U replication_user -D /backup/base -Fp -Xs -P
```

### Основные параметры

```bash
# Формат вывода (-F)
-Fp  # plain - обычная директория с файлами
-Ft  # tar - tar-архив(ы)

# Метод включения WAL (-X)
-Xn  # none - не включать WAL
-Xf  # fetch - забрать WAL в конце бэкапа
-Xs  # stream - streaming параллельно с бэкапом (рекомендуется)

# Сжатие
-z       # gzip сжатие (для tar формата)
-Z 0-9   # уровень сжатия (0=нет, 9=максимум)
--compress=METHOD:LEVEL  # PostgreSQL 15+: gzip, lz4, zstd

# Прогресс и информация
-P       # показывать прогресс
-v       # verbose режим
```

## Форматы вывода

### Plain формат (-Fp)

```bash
# Создает директорию с файлами данных
pg_basebackup -D /backup/base_20240115 -Fp -Xs -P

# Структура результата:
# /backup/base_20240115/
# ├── base/           # данные баз
# ├── global/         # глобальные объекты
# ├── pg_wal/         # WAL файлы
# ├── postgresql.conf # конфигурация
# ├── pg_hba.conf
# └── ...
```

### Tar формат (-Ft)

```bash
# Создает tar-архивы
pg_basebackup -D /backup/tar_20240115 -Ft -Xs -z -P

# Структура результата:
# /backup/tar_20240115/
# ├── base.tar.gz     # основные данные
# └── pg_wal.tar.gz   # WAL файлы (если -Xs)

# С отдельными tablespace
# ├── base.tar.gz
# ├── pg_wal.tar.gz
# └── 16384.tar.gz    # tablespace OID
```

### Сжатие (PostgreSQL 15+)

```bash
# Gzip сжатие
pg_basebackup -D /backup -Ft --compress=gzip:6

# LZ4 сжатие (быстрее)
pg_basebackup -D /backup -Ft --compress=lz4

# Zstd сжатие (лучший баланс)
pg_basebackup -D /backup -Ft --compress=zstd:3

# Сжатие для plain формата (client-side, PostgreSQL 15+)
pg_basebackup -D /backup -Fp --compress=client-gzip:6
```

## Методы передачи WAL

### Stream (-Xs) - Рекомендуемый

```bash
# WAL передается параллельно с бэкапом
pg_basebackup -D /backup -Xs -P

# Преимущества:
# - Гарантированно все нужные WAL включены
# - Работает даже если WAL был переписан
# - Требует 2 подключения (wal_sender)
```

### Fetch (-Xf)

```bash
# WAL забирается после завершения бэкапа
pg_basebackup -D /backup -Xf -P

# Преимущества:
# - Требует только 1 подключение
# Недостатки:
# - WAL может быть переписан до завершения
# - Требует достаточного wal_keep_size
```

### None (-Xn)

```bash
# WAL не включается
pg_basebackup -D /backup -Xn -P

# Используется когда:
# - WAL архивируется отдельно (archive_command)
# - Настроен continuous archiving
```

## Расширенные опции

### Слоты репликации

```bash
# Создать временный слот на время бэкапа
pg_basebackup -D /backup -Xs --slot=temp_slot --create-slot -P

# Использовать существующий слот
pg_basebackup -D /backup -Xs --slot=backup_slot -P

# Преимущества слотов:
# - Гарантия сохранения WAL на primary
# - Защита от потери WAL при долгом бэкапе
```

### Checkpoint опции

```bash
# Быстрый checkpoint (больше нагрузки на I/O)
pg_basebackup -D /backup --checkpoint=fast -P

# Spread checkpoint (по умолчанию, меньше нагрузки)
pg_basebackup -D /backup --checkpoint=spread -P
```

### Ограничение скорости

```bash
# Ограничить скорость передачи данных
pg_basebackup -D /backup --max-rate=100M -P  # 100 MB/s

# Полезно для:
# - Снижения нагрузки на production сервер
# - Ограничения сетевого трафика
```

### Tablespaces

```bash
# Показать mapping tablespace
pg_basebackup -D /backup -Fp -P --tablespace-mapping=/old/path=/new/path

# Несколько tablespace mappings
pg_basebackup -D /backup -Fp -P \
  -T /var/lib/postgresql/ts1=/backup/ts1 \
  -T /var/lib/postgresql/ts2=/backup/ts2
```

### Проверка целостности

```bash
# Проверка CRC checksums (если включены)
pg_basebackup -D /backup --no-verify-checksums  # отключить проверку

# По умолчанию checksums проверяются, если включены на сервере
```

## Практические сценарии

### Сценарий 1: Ежедневный бэкап с ротацией

```bash
#!/bin/bash
# daily_basebackup.sh

BACKUP_BASE="/var/backups/postgres"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="${BACKUP_BASE}/base_${DATE}"
RETENTION_DAYS=7
PG_USER="replication_user"

echo "Starting backup: ${BACKUP_DIR}"

# Создание бэкапа
pg_basebackup \
    -h localhost \
    -U $PG_USER \
    -D "$BACKUP_DIR" \
    -Ft \
    --compress=zstd:3 \
    -Xs \
    --checkpoint=fast \
    -P \
    -v 2>&1 | tee "${BACKUP_DIR}.log"

if [ $? -eq 0 ]; then
    echo "Backup completed successfully"

    # Создание метафайла
    cat > "${BACKUP_DIR}/backup_info.txt" << EOF
Backup Date: $(date)
Server: $(hostname)
PostgreSQL Version: $(psql -t -c "SELECT version();")
Database Size: $(psql -t -c "SELECT pg_size_pretty(sum(pg_database_size(datname))) FROM pg_database;")
EOF

    # Удаление старых бэкапов
    find $BACKUP_BASE -name "base_*" -type d -mtime +$RETENTION_DAYS -exec rm -rf {} \;
    echo "Old backups cleaned up"
else
    echo "Backup FAILED!" >&2
    exit 1
fi
```

### Сценарий 2: Бэкап в удаленное хранилище

```bash
#!/bin/bash
# remote_basebackup.sh

REMOTE_HOST="backup-server"
REMOTE_PATH="/backups/postgres"
DATE=$(date +%Y%m%d_%H%M%S)
LOCAL_TEMP="/tmp/pg_backup_${DATE}"

# Создать локальный бэкап
pg_basebackup -D "$LOCAL_TEMP" -Ft -z -Xs -P

if [ $? -eq 0 ]; then
    # Передать на удаленный сервер
    rsync -avz --progress "$LOCAL_TEMP/" "${REMOTE_HOST}:${REMOTE_PATH}/base_${DATE}/"

    # Очистить локальную копию
    rm -rf "$LOCAL_TEMP"

    echo "Backup transferred to ${REMOTE_HOST}"
else
    echo "Backup failed!"
    exit 1
fi
```

### Сценарий 3: Streaming бэкап в S3

```bash
#!/bin/bash
# basebackup_to_s3.sh

S3_BUCKET="s3://my-backups/postgres"
DATE=$(date +%Y%m%d_%H%M%S)
TEMP_DIR="/tmp/pg_backup_${DATE}"

mkdir -p "$TEMP_DIR"

# Бэкап в tar формате
pg_basebackup -D "$TEMP_DIR" -Ft -z -Xs -P

# Загрузка в S3
aws s3 sync "$TEMP_DIR" "${S3_BUCKET}/base_${DATE}/"

# Очистка
rm -rf "$TEMP_DIR"

# Удаление старых бэкапов из S3 (старше 7 дней)
# Используйте S3 Lifecycle policies для этого
```

### Сценарий 4: Создание standby сервера

```bash
#!/bin/bash
# create_standby.sh

PRIMARY_HOST="primary.example.com"
STANDBY_DATADIR="/var/lib/postgresql/14/main"
REPLICATION_USER="replication_user"

# Остановить standby если запущен
sudo systemctl stop postgresql

# Очистить старые данные
rm -rf "${STANDBY_DATADIR:?}/"*

# Создать базовый бэкап с primary
pg_basebackup \
    -h $PRIMARY_HOST \
    -U $REPLICATION_USER \
    -D $STANDBY_DATADIR \
    -Fp \
    -Xs \
    -P \
    -R  # Создает standby.signal и настраивает recovery

# Установить права
chown -R postgres:postgres $STANDBY_DATADIR
chmod 700 $STANDBY_DATADIR

# Запустить standby
sudo systemctl start postgresql

echo "Standby server created and started"
```

### Сценарий 5: Подготовка к PITR

```bash
#!/bin/bash
# setup_pitr.sh

# 1. Настройка архивации WAL на primary
cat >> /etc/postgresql/14/main/postgresql.conf << EOF
archive_mode = on
archive_command = 'test ! -f /var/lib/postgresql/wal_archive/%f && cp %p /var/lib/postgresql/wal_archive/%f'
EOF

# Создать директорию для архива
mkdir -p /var/lib/postgresql/wal_archive
chown postgres:postgres /var/lib/postgresql/wal_archive

# Перезапустить PostgreSQL
sudo systemctl restart postgresql

# 2. Создать базовый бэкап
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/var/backups/postgres/base_${DATE}"

pg_basebackup \
    -D "$BACKUP_DIR" \
    -Fp \
    -Xn \  # WAL архивируется отдельно через archive_command
    --checkpoint=fast \
    -P

# 3. Записать label для PITR
echo "Backup label: base_${DATE}" > "${BACKUP_DIR}/backup_label_info"
echo "Archive location: /var/lib/postgresql/wal_archive" >> "${BACKUP_DIR}/backup_label_info"
```

## Восстановление из pg_basebackup

### Простое восстановление (plain формат)

```bash
#!/bin/bash
# restore_base.sh

BACKUP_DIR="/var/backups/postgres/base_20240115"
DATA_DIR="/var/lib/postgresql/14/main"

# 1. Остановить PostgreSQL
sudo systemctl stop postgresql

# 2. Очистить и восстановить данные
rm -rf "${DATA_DIR:?}/"*
cp -a "${BACKUP_DIR}/"* "$DATA_DIR/"

# 3. Удалить recovery файлы если не нужен recovery
rm -f "$DATA_DIR/standby.signal"
rm -f "$DATA_DIR/recovery.signal"

# 4. Установить права
chown -R postgres:postgres $DATA_DIR
chmod 700 $DATA_DIR

# 5. Запустить PostgreSQL
sudo systemctl start postgresql
```

### Восстановление из tar формата

```bash
#!/bin/bash
# restore_tar.sh

BACKUP_DIR="/var/backups/postgres/tar_20240115"
DATA_DIR="/var/lib/postgresql/14/main"

# 1. Остановить PostgreSQL
sudo systemctl stop postgresql

# 2. Очистить директорию данных
rm -rf "${DATA_DIR:?}/"*

# 3. Распаковать архивы
cd "$DATA_DIR"
tar xzf "${BACKUP_DIR}/base.tar.gz"
tar xzf "${BACKUP_DIR}/pg_wal.tar.gz" -C pg_wal/

# 4. Установить права
chown -R postgres:postgres $DATA_DIR
chmod 700 $DATA_DIR

# 5. Запустить PostgreSQL
sudo systemctl start postgresql
```

### Point-in-Time Recovery (PITR)

```bash
#!/bin/bash
# pitr_restore.sh

BACKUP_DIR="/var/backups/postgres/base_20240115"
DATA_DIR="/var/lib/postgresql/14/main"
WAL_ARCHIVE="/var/lib/postgresql/wal_archive"
RECOVERY_TARGET="2024-01-15 14:30:00"

# 1. Остановить PostgreSQL
sudo systemctl stop postgresql

# 2. Восстановить базовый бэкап
rm -rf "${DATA_DIR:?}/"*
cp -a "${BACKUP_DIR}/"* "$DATA_DIR/"

# 3. Настроить recovery
cat > "$DATA_DIR/postgresql.auto.conf" << EOF
restore_command = 'cp ${WAL_ARCHIVE}/%f %p'
recovery_target_time = '${RECOVERY_TARGET}'
recovery_target_action = 'promote'
EOF

# 4. Создать сигнальный файл для recovery
touch "$DATA_DIR/recovery.signal"

# 5. Установить права
chown -R postgres:postgres $DATA_DIR
chmod 700 $DATA_DIR

# 6. Запустить PostgreSQL (начнется recovery)
sudo systemctl start postgresql

echo "Recovery started. Monitor with: tail -f /var/log/postgresql/postgresql-14-main.log"
```

## Мониторинг и диагностика

### Проверка прогресса бэкапа

```sql
-- На сервере источнике
SELECT * FROM pg_stat_progress_basebackup;

-- Информация о текущих подключениях репликации
SELECT * FROM pg_stat_replication;

-- Проверка слотов репликации
SELECT * FROM pg_replication_slots;
```

### Логирование бэкапа

```bash
# Подробный вывод
pg_basebackup -D /backup -Xs -v -P 2>&1 | tee backup.log

# Timestamp в логе
pg_basebackup -D /backup -Xs -v -P 2>&1 | while read line; do
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $line"
done | tee backup_timestamped.log
```

## Best Practices

### 1. Настройка сервера

```bash
# Рекомендуемые настройки postgresql.conf
wal_level = replica
max_wal_senders = 5
max_replication_slots = 5
wal_keep_size = 1GB  # или используйте слоты репликации
```

### 2. Выбор параметров бэкапа

```bash
# Рекомендуемые параметры для production
pg_basebackup \
    -D /backup/base_$(date +%Y%m%d) \
    -Ft \                    # tar формат
    --compress=zstd:3 \      # или -z для gzip
    -Xs \                    # streaming WAL
    --slot=backup_slot \     # слот репликации
    --checkpoint=fast \      # быстрый checkpoint
    --max-rate=100M \        # ограничение скорости
    -P \                     # прогресс
    -v                       # verbose
```

### 3. Безопасность

```bash
# Использовать .pgpass для паролей
echo "hostname:5432:replication:replication_user:password" >> ~/.pgpass
chmod 600 ~/.pgpass

# SSL подключение
pg_basebackup -h remote_host -D /backup --sslmode=require
```

### 4. Проверка бэкапа

```bash
# Проверить целостность tar архива
tar -tzf /backup/base.tar.gz > /dev/null && echo "Archive OK"

# Тестовое восстановление
# (см. раздел "Восстановление")
```

## Типичные проблемы и решения

### Проблема 1: "FATAL: no pg_hba.conf entry for replication"

```bash
# Добавить в pg_hba.conf:
host    replication     replication_user    192.168.1.0/24    scram-sha-256

# Перезагрузить конфигурацию
sudo systemctl reload postgresql
```

### Проблема 2: "requested WAL segment has already been removed"

```bash
# Решение 1: Использовать streaming (-Xs)
pg_basebackup -D /backup -Xs -P

# Решение 2: Увеличить wal_keep_size
ALTER SYSTEM SET wal_keep_size = '2GB';
SELECT pg_reload_conf();

# Решение 3: Использовать слот репликации
pg_basebackup -D /backup --slot=backup_slot --create-slot -Xs -P
```

### Проблема 3: "could not connect to server"

```bash
# Проверить параметры подключения
pg_isready -h localhost -p 5432

# Проверить max_connections и max_wal_senders
psql -c "SHOW max_wal_senders;"
psql -c "SELECT count(*) FROM pg_stat_replication;"
```

### Проблема 4: Медленный бэкап

```bash
# Использовать быстрый checkpoint
pg_basebackup -D /backup --checkpoint=fast -Xs -P

# Проверить I/O на сервере
iostat -x 1

# Использовать сжатие на стороне клиента
pg_basebackup -D /backup --compress=client-zstd:3 -Xs -P
```

## Заключение

pg_basebackup - незаменимый инструмент для создания физических резервных копий PostgreSQL. Ключевые рекомендации:

1. **Используйте streaming WAL (-Xs)** для гарантированной консистентности
2. **Применяйте сжатие** для экономии места
3. **Настройте слоты репликации** для надежности
4. **Регулярно тестируйте восстановление** из бэкапов
5. **Комбинируйте с WAL архивацией** для PITR

pg_basebackup является основой для более продвинутых решений резервного копирования, таких как WAL-G и pgBackRest.
