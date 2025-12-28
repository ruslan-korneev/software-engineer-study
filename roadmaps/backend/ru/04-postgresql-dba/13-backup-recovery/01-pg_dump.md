# pg_dump - Утилита логического бэкапа PostgreSQL

[prev: 06-load-balancing](../12-replication-clustering/06-load-balancing.md) | [next: 02-pg_restore](./02-pg_restore.md)

---

## Введение

**pg_dump** - это стандартная утилита PostgreSQL для создания логических резервных копий баз данных. Она извлекает содержимое базы данных в виде SQL-скрипта или архивного файла, который можно использовать для восстановления базы данных.

### Ключевые особенности pg_dump

- **Логический бэкап** - сохраняет структуру и данные в виде SQL-команд
- **Консистентность** - создает снимок базы данных на момент начала выполнения
- **Гибкость** - позволяет выбирать конкретные объекты для бэкапа
- **Переносимость** - дампы можно восстанавливать на разных версиях PostgreSQL
- **Не блокирует** - работает без эксклюзивных блокировок (использует MVCC)

## Форматы вывода

pg_dump поддерживает несколько форматов вывода:

| Формат | Ключ | Описание | Сжатие | Параллельное восстановление |
|--------|------|----------|--------|----------------------------|
| Plain | `-F p` | SQL-скрипт (по умолчанию) | Нет (внешнее) | Нет |
| Custom | `-F c` | Архив pg_dump | Да | Да |
| Directory | `-F d` | Директория с файлами | Да | Да |
| Tar | `-F t` | tar-архив | Нет | Нет |

## Базовое использование

### Простой дамп базы данных

```bash
# Дамп всей базы данных в SQL-формате
pg_dump mydb > mydb_backup.sql

# Дамп с явным указанием хоста и пользователя
pg_dump -h localhost -p 5432 -U postgres mydb > mydb_backup.sql

# Дамп в custom-формате (рекомендуется)
pg_dump -Fc mydb > mydb_backup.dump

# Дамп в directory-формате с параллельным выполнением
pg_dump -Fd -j 4 mydb -f mydb_backup_dir/
```

### Параметры подключения

```bash
# Через параметры командной строки
pg_dump -h hostname -p 5432 -U username -d dbname

# Через переменные окружения
export PGHOST=localhost
export PGPORT=5432
export PGUSER=postgres
export PGPASSWORD=secret  # Не рекомендуется в production
pg_dump mydb > backup.sql

# Через connection string
pg_dump "postgresql://user:password@localhost:5432/mydb" > backup.sql

# С использованием .pgpass файла (рекомендуется)
# ~/.pgpass: hostname:port:database:username:password
pg_dump -h localhost -U postgres mydb > backup.sql
```

## Опции выбора объектов

### Выбор конкретных объектов

```bash
# Дамп только определенной таблицы
pg_dump -t users mydb > users_table.sql

# Дамп нескольких таблиц
pg_dump -t users -t orders -t products mydb > selected_tables.sql

# Дамп таблиц по шаблону (wildcard)
pg_dump -t 'public.user*' mydb > user_tables.sql

# Дамп конкретной схемы
pg_dump -n public mydb > public_schema.sql

# Исключение таблиц
pg_dump -T logs -T audit_* mydb > backup_without_logs.sql

# Исключение схемы
pg_dump -N temp_schema mydb > backup.sql
```

### Выбор типов данных

```bash
# Только схема (DDL), без данных
pg_dump -s mydb > schema_only.sql

# Только данные, без схемы
pg_dump -a mydb > data_only.sql

# Исключить данные больших объектов (BLOB)
pg_dump --no-blobs mydb > backup.sql

# Включить большие объекты
pg_dump -b mydb > backup_with_blobs.sql
```

## Расширенные опции

### Управление зависимостями и ограничениями

```bash
# Отключить триггеры при восстановлении (для data-only)
pg_dump -a --disable-triggers mydb > data_with_disabled_triggers.sql

# Не дампить владельцев объектов
pg_dump --no-owner mydb > backup.sql

# Не дампить привилегии (GRANT/REVOKE)
pg_dump --no-privileges mydb > backup.sql

# Не дампить tablespace assignments
pg_dump --no-tablespaces mydb > backup.sql

# Создавать IF NOT EXISTS для CREATE
pg_dump --if-exists mydb > backup.sql
```

### Опции производительности

```bash
# Параллельный дамп (только для directory формата)
pg_dump -Fd -j 8 mydb -f backup_dir/

# Сжатие для custom-формата (уровень 0-9)
pg_dump -Fc -Z 9 mydb > backup.dump.gz

# Сжатие для directory-формата
pg_dump -Fd -Z 6 -j 4 mydb -f backup_dir/

# Использование другого алгоритма сжатия (PostgreSQL 16+)
pg_dump -Fc --compress=zstd:5 mydb > backup.dump.zst
pg_dump -Fc --compress=lz4 mydb > backup.dump.lz4
```

### Блокировки и изоляция

```bash
# Ожидать блокировки вместо немедленного отказа
pg_dump --lock-wait-timeout=300000 mydb > backup.sql  # 5 минут

# Использовать snapshot для консистентности
pg_dump --snapshot=00000003-00000064-1 mydb > backup.sql

# Дамп с serializable изоляцией
pg_dump --serializable-deferrable mydb > backup.sql
```

## pg_dumpall - Дамп всего кластера

**pg_dumpall** используется для создания дампа всего кластера PostgreSQL, включая глобальные объекты.

```bash
# Полный дамп кластера
pg_dumpall > full_cluster.sql

# Только глобальные объекты (роли, tablespaces)
pg_dumpall -g > globals_only.sql

# Только роли
pg_dumpall -r > roles_only.sql

# Только tablespaces
pg_dumpall -t > tablespaces_only.sql

# Дамп с очисткой перед восстановлением
pg_dumpall -c > full_cluster_clean.sql
```

## Практические сценарии

### Сценарий 1: Ежедневный бэкап с ротацией

```bash
#!/bin/bash
# daily_backup.sh

DB_NAME="production"
BACKUP_DIR="/var/backups/postgres"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=7

# Создание бэкапа
pg_dump -Fc -Z 6 $DB_NAME > "${BACKUP_DIR}/${DB_NAME}_${DATE}.dump"

# Проверка успешности
if [ $? -eq 0 ]; then
    echo "Backup completed: ${DB_NAME}_${DATE}.dump"

    # Удаление старых бэкапов
    find $BACKUP_DIR -name "${DB_NAME}_*.dump" -mtime +$RETENTION_DAYS -delete
else
    echo "Backup failed!" >&2
    exit 1
fi
```

### Сценарий 2: Бэкап с загрузкой в S3

```bash
#!/bin/bash
# backup_to_s3.sh

DB_NAME="production"
S3_BUCKET="s3://my-backups/postgres"
DATE=$(date +%Y%m%d_%H%M%S)

# Дамп напрямую в S3 через pipe
pg_dump -Fc $DB_NAME | aws s3 cp - "${S3_BUCKET}/${DB_NAME}_${DATE}.dump"

# Или с промежуточным сжатием
pg_dump $DB_NAME | gzip | aws s3 cp - "${S3_BUCKET}/${DB_NAME}_${DATE}.sql.gz"
```

### Сценарий 3: Миграция между серверами

```bash
# Прямая передача между серверами
pg_dump -h source_host -U postgres mydb | psql -h dest_host -U postgres mydb

# С промежуточным файлом (более надежно)
pg_dump -Fc -h source_host mydb > migration.dump
pg_restore -h dest_host -d mydb migration.dump
```

### Сценарий 4: Частичный дамп с фильтрацией данных

```bash
# Дамп схемы полностью + данные только за последний месяц
pg_dump -s mydb > schema.sql
pg_dump -a -t orders --where="created_at > NOW() - INTERVAL '1 month'" mydb > recent_orders.sql

# Примечание: --where доступен начиная с PostgreSQL 16
# Для более ранних версий используйте COPY или создайте временную таблицу
```

### Сценарий 5: Параллельный бэкап большой базы

```bash
#!/bin/bash
# parallel_backup.sh

DB_NAME="large_database"
BACKUP_DIR="/var/backups/postgres/parallel_$(date +%Y%m%d)"
JOBS=8

# Создание директории
mkdir -p $BACKUP_DIR

# Параллельный дамп
pg_dump -Fd -j $JOBS -Z 6 $DB_NAME -f $BACKUP_DIR

# Проверка размера
echo "Backup size: $(du -sh $BACKUP_DIR)"
```

## Best Practices

### 1. Выбор формата

```text
- Custom (-Fc): Рекомендуется для большинства случаев
  - Поддерживает сжатие
  - Позволяет выборочное восстановление
  - Поддерживает параллельное восстановление

- Directory (-Fd): Для очень больших баз данных
  - Параллельный дамп и восстановление
  - Каждая таблица в отдельном файле

- Plain (-Fp): Для простых случаев и читаемости
  - Можно редактировать вручную
  - Легко просматривать
```

### 2. Безопасность

```bash
# Используйте .pgpass вместо паролей в скриптах
# ~/.pgpass с правами 600
hostname:5432:dbname:username:password

# Или через переменную окружения PGPASSFILE
export PGPASSFILE=/secure/path/.pgpass
pg_dump mydb > backup.sql
```

### 3. Мониторинг и логирование

```bash
# Логирование процесса дампа
pg_dump -v mydb > backup.sql 2> backup.log

# С временными метками
pg_dump -v mydb 2>&1 | while read line; do
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $line"
done > backup_with_timestamps.log
```

### 4. Проверка целостности

```bash
# Всегда проверяйте дамп после создания
pg_restore -l backup.dump > /dev/null && echo "Dump is valid"

# Тестовое восстановление в отдельную базу
createdb test_restore
pg_restore -d test_restore backup.dump
# Проверить данные
dropdb test_restore
```

## Типичные проблемы и решения

### Проблема 1: Timeout при дампе большой таблицы

```bash
# Увеличить timeout
pg_dump --lock-wait-timeout=600000 mydb > backup.sql

# Или исключить проблемную таблицу и дампить отдельно
pg_dump -T huge_table mydb > main_backup.sql
pg_dump -t huge_table mydb > huge_table_backup.sql
```

### Проблема 2: Недостаточно места на диске

```bash
# Использовать сжатие на лету
pg_dump mydb | gzip > backup.sql.gz

# Или streaming в удаленное хранилище
pg_dump -Fc mydb | ssh backup_server "cat > /backups/mydb.dump"
```

### Проблема 3: Зависимости внешних ключей при восстановлении

```bash
# Дамп с отключением триггеров (для data-only)
pg_dump -a --disable-triggers mydb > data_only.sql

# При восстановлении full dump - порядок соблюдается автоматически
```

## Сравнение с другими методами бэкапа

| Характеристика | pg_dump | pg_basebackup | WAL-G/pgBackRest |
|---------------|---------|---------------|------------------|
| Тип | Логический | Физический | Физический + WAL |
| PITR | Нет | С WAL архивацией | Да |
| Скорость бэкапа | Медленнее | Быстрее | Быстрее |
| Размер бэкапа | Меньше | Больше | Инкрементальный |
| Кросс-версионность | Да | Нет | Нет |
| Выборочное восстановление | Да | Нет | Нет |

## Заключение

pg_dump остается основным инструментом для логического бэкапа PostgreSQL благодаря своей гибкости и надежности. Для критичных production-систем рекомендуется комбинировать pg_dump с физическими бэкапами (pg_basebackup, WAL-G, pgBackRest) для обеспечения полноценной стратегии резервного копирования с возможностью Point-in-Time Recovery.

---

[prev: 06-load-balancing](../12-replication-clustering/06-load-balancing.md) | [next: 02-pg_restore](./02-pg_restore.md)
