# pg_restore - Восстановление из дампа PostgreSQL

[prev: 01-pg_dump](./01-pg_dump.md) | [next: 03-pg_basebackup](./03-pg_basebackup.md)

---

## Введение

**pg_restore** - это утилита PostgreSQL для восстановления базы данных из архива, созданного pg_dump. Она работает с форматами custom (-Fc), directory (-Fd) и tar (-Ft), но не с plain text SQL (-Fp).

### Ключевые особенности pg_restore

- **Гибкое восстановление** - можно выбирать конкретные объекты для восстановления
- **Параллельное выполнение** - поддержка многопоточного восстановления
- **Редактирование списка** - возможность модифицировать порядок и состав объектов
- **Обработка ошибок** - продолжение работы при ошибках с логированием
- **Интеграция с psql** - можно генерировать SQL для ручного выполнения

## Базовое использование

### Восстановление всей базы данных

```bash
# Восстановление в существующую базу данных
pg_restore -d mydb backup.dump

# Создание новой базы и восстановление
pg_restore -C -d postgres backup.dump

# Восстановление с предварительной очисткой (DROP + CREATE)
pg_restore -c -d mydb backup.dump

# Восстановление только если база не существует
pg_restore -C --if-exists -d postgres backup.dump
```

### Параметры подключения

```bash
# Явное указание параметров
pg_restore -h localhost -p 5432 -U postgres -d mydb backup.dump

# С использованием connection string
pg_restore -d "postgresql://user:pass@localhost:5432/mydb" backup.dump

# Переменные окружения
export PGHOST=localhost
export PGUSER=postgres
pg_restore -d mydb backup.dump
```

## Форматы входных данных

### Работа с разными форматами

```bash
# Custom format (.dump)
pg_restore -d mydb backup.dump

# Directory format
pg_restore -d mydb backup_dir/

# Tar format
pg_restore -d mydb backup.tar

# Plain SQL format - используется psql, не pg_restore!
psql -d mydb -f backup.sql
```

## Выборочное восстановление

### Восстановление конкретных объектов

```bash
# Восстановление только одной таблицы
pg_restore -d mydb -t users backup.dump

# Несколько таблиц
pg_restore -d mydb -t users -t orders -t products backup.dump

# Восстановление таблиц по шаблону
pg_restore -d mydb -t 'user*' backup.dump

# Восстановление конкретной схемы
pg_restore -d mydb -n public backup.dump

# Исключение таблиц
pg_restore -d mydb -T logs -T audit_events backup.dump

# Исключение схемы
pg_restore -d mydb -N temp_schema backup.dump
```

### Восстановление типов данных

```bash
# Только схема (DDL), без данных
pg_restore -s -d mydb backup.dump

# Только данные, без схемы
pg_restore -a -d mydb backup.dump

# Только индексы
pg_restore -d mydb -I index_name backup.dump

# Только функции
pg_restore -d mydb -P 'function_name(arg_types)' backup.dump

# Только триггеры
pg_restore -d mydb -T trigger_name backup.dump
```

## Работа со списком объектов (TOC)

### Просмотр содержимого дампа

```bash
# Получить список всех объектов в дампе
pg_restore -l backup.dump > toc_list.txt

# Пример вывода:
# ;
# ; Archive created at 2024-01-15 10:30:00 UTC
# ;     dbname: mydb
# ;     TOC Entries: 125
# ;
# 1; 2615 16384 SCHEMA - public postgres
# 2; 1259 16385 TABLE public users postgres
# 3; 1259 16390 TABLE public orders postgres
# 4; 0 16385 TABLE DATA public users postgres
```

### Редактирование списка

```bash
# 1. Экспортировать список
pg_restore -l backup.dump > toc_list.txt

# 2. Отредактировать файл (закомментировать ненужные строки)
# ; 4; 0 16385 TABLE DATA public users postgres  <- закомментировано

# 3. Восстановить с модифицированным списком
pg_restore -d mydb -L toc_list.txt backup.dump
```

### Примеры редактирования TOC

```bash
# Исключить все данные, оставить только схему
pg_restore -l backup.dump | grep -v "TABLE DATA" > schema_only.txt
pg_restore -d mydb -L schema_only.txt backup.dump

# Восстановить только определенные таблицы
pg_restore -l backup.dump | grep -E "(users|orders)" > selected.txt
pg_restore -d mydb -L selected.txt backup.dump

# Изменить порядок восстановления (осторожно с зависимостями!)
# Ручное редактирование toc_list.txt
```

## Параллельное восстановление

### Многопоточное выполнение

```bash
# Параллельное восстановление с 8 потоками
pg_restore -j 8 -d mydb backup.dump

# Параллельное восстановление из directory формата
pg_restore -j 8 -d mydb backup_dir/

# Рекомендации по количеству потоков:
# - Обычно: количество ядер CPU
# - С SSD: можно больше (до 2x ядер)
# - С HDD: меньше (ограничено I/O)
```

### Оптимизация параллельного восстановления

```bash
# Предварительная очистка + параллельное восстановление
pg_restore -c -j 8 -d mydb backup.dump

# Восстановление схемы последовательно, данных параллельно
pg_restore -s -d mydb backup.dump  # схема
pg_restore -a -j 8 -d mydb backup.dump  # данные

# Отключение индексов для ускорения загрузки данных
# (индексы создаются после данных автоматически)
```

## Обработка ошибок

### Режимы обработки ошибок

```bash
# Остановка при первой ошибке (по умолчанию)
pg_restore -d mydb backup.dump

# Продолжение при ошибках (с выводом в stderr)
pg_restore --exit-on-error=false -d mydb backup.dump

# Вывод SQL без выполнения (для отладки)
pg_restore -f output.sql backup.dump
psql -d mydb -f output.sql

# Подробный вывод для отладки
pg_restore -v -d mydb backup.dump 2>&1 | tee restore.log
```

### Обработка конфликтов

```bash
# Очистка перед восстановлением
pg_restore -c -d mydb backup.dump

# Очистка с IF EXISTS (меньше ошибок)
pg_restore -c --if-exists -d mydb backup.dump

# Без владельцев (если пользователи не существуют)
pg_restore --no-owner -d mydb backup.dump

# Без привилегий
pg_restore --no-privileges -d mydb backup.dump

# Без tablespace (если tablespace не существует)
pg_restore --no-tablespaces -d mydb backup.dump
```

## Практические сценарии

### Сценарий 1: Восстановление production базы

```bash
#!/bin/bash
# restore_production.sh

BACKUP_FILE=$1
DB_NAME="production"
JOBS=8

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup_file>"
    exit 1
fi

# Проверка дампа
echo "Validating backup..."
pg_restore -l "$BACKUP_FILE" > /dev/null
if [ $? -ne 0 ]; then
    echo "Invalid backup file!"
    exit 1
fi

# Отключение подключений
psql -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='$DB_NAME' AND pid <> pg_backend_pid();"

# Пересоздание базы
dropdb --if-exists $DB_NAME
createdb $DB_NAME

# Параллельное восстановление
echo "Restoring database..."
pg_restore -j $JOBS -d $DB_NAME "$BACKUP_FILE"

if [ $? -eq 0 ]; then
    echo "Restore completed successfully!"
    # Обновление статистики
    psql -d $DB_NAME -c "ANALYZE;"
else
    echo "Restore failed!"
    exit 1
fi
```

### Сценарий 2: Частичное восстановление таблицы

```bash
#!/bin/bash
# restore_table.sh

BACKUP_FILE=$1
TABLE_NAME=$2
DB_NAME="production"

if [ -z "$TABLE_NAME" ]; then
    echo "Usage: $0 <backup_file> <table_name>"
    exit 1
fi

# Создание временной базы для извлечения данных
TEMP_DB="temp_restore_$$"
createdb $TEMP_DB

# Восстановление только нужной таблицы во временную базу
pg_restore -t $TABLE_NAME -d $TEMP_DB "$BACKUP_FILE"

# Экспорт данных из временной базы
pg_dump -t $TABLE_NAME --data-only $TEMP_DB > /tmp/table_data.sql

echo "Data exported to /tmp/table_data.sql"
echo "Review and import manually with:"
echo "  psql -d $DB_NAME -f /tmp/table_data.sql"

# Очистка
dropdb $TEMP_DB
```

### Сценарий 3: Миграция с изменением схемы

```bash
#!/bin/bash
# migration_restore.sh

BACKUP_FILE=$1
SOURCE_SCHEMA="old_schema"
TARGET_SCHEMA="new_schema"
DB_NAME="migrated_db"

# Создание базы
createdb $DB_NAME

# Экспорт списка объектов
pg_restore -l "$BACKUP_FILE" > /tmp/toc.txt

# Фильтрация объектов из нужной схемы
grep " $SOURCE_SCHEMA " /tmp/toc.txt > /tmp/filtered_toc.txt

# Генерация SQL
pg_restore -L /tmp/filtered_toc.txt -f /tmp/restore.sql "$BACKUP_FILE"

# Замена имени схемы в SQL
sed -i "s/$SOURCE_SCHEMA/$TARGET_SCHEMA/g" /tmp/restore.sql

# Создание новой схемы и выполнение
psql -d $DB_NAME -c "CREATE SCHEMA IF NOT EXISTS $TARGET_SCHEMA;"
psql -d $DB_NAME -f /tmp/restore.sql
```

### Сценарий 4: Восстановление из S3

```bash
#!/bin/bash
# restore_from_s3.sh

S3_PATH=$1
DB_NAME=$2

if [ -z "$DB_NAME" ]; then
    echo "Usage: $0 <s3://bucket/path/backup.dump> <db_name>"
    exit 1
fi

# Создание базы
createdb $DB_NAME

# Streaming восстановление напрямую из S3
aws s3 cp "$S3_PATH" - | pg_restore -d $DB_NAME

# Или с промежуточным файлом (для больших бэкапов)
# aws s3 cp "$S3_PATH" /tmp/backup.dump
# pg_restore -j 8 -d $DB_NAME /tmp/backup.dump
# rm /tmp/backup.dump
```

### Сценарий 5: Тестирование бэкапа

```bash
#!/bin/bash
# test_restore.sh

BACKUP_FILE=$1
TEST_DB="backup_test_$(date +%Y%m%d_%H%M%S)"

echo "Creating test database: $TEST_DB"
createdb $TEST_DB

echo "Restoring backup..."
pg_restore -j 4 -d $TEST_DB "$BACKUP_FILE" 2>&1 | tee /tmp/restore.log

# Подсчет объектов
TABLE_COUNT=$(psql -t -d $TEST_DB -c "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';")
ROW_COUNT=$(psql -t -d $TEST_DB -c "SELECT sum(n_live_tup) FROM pg_stat_user_tables;")

echo "Tables restored: $TABLE_COUNT"
echo "Total rows: $ROW_COUNT"

# Проверка на ошибки
ERROR_COUNT=$(grep -c "ERROR" /tmp/restore.log)
echo "Errors during restore: $ERROR_COUNT"

# Очистка
read -p "Delete test database? (y/n) " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    dropdb $TEST_DB
fi
```

## Расширенные опции

### Управление транзакциями

```bash
# Восстановление в одной транзакции (все или ничего)
pg_restore -1 -d mydb backup.dump

# Без одной транзакции (по умолчанию) - позволяет параллельное восстановление
pg_restore -d mydb backup.dump
```

### Триггеры и ограничения

```bash
# Отключение триггеров при загрузке данных
pg_restore --disable-triggers -a -d mydb backup.dump

# Отложенная проверка constraints
# (в pg_restore нет прямой опции, используйте в сессии)
psql -d mydb -c "SET session_replication_role = replica;"
pg_restore -a -d mydb backup.dump
psql -d mydb -c "SET session_replication_role = DEFAULT;"
```

### Секции и суперпользователь

```bash
# Восстановление как суперпользователь
pg_restore --superuser=postgres -d mydb backup.dump

# Использование конкретной роли
pg_restore --role=app_admin -d mydb backup.dump
```

## Best Practices

### 1. Перед восстановлением

```bash
# Проверить дамп
pg_restore -l backup.dump | head -20

# Оценить размер
pg_restore -l backup.dump | wc -l  # количество объектов

# Проверить версию PostgreSQL
pg_restore --version
psql -c "SELECT version();"
```

### 2. Оптимизация производительности

```bash
# Настройки PostgreSQL для быстрого восстановления
psql -c "ALTER SYSTEM SET maintenance_work_mem = '2GB';"
psql -c "ALTER SYSTEM SET max_wal_size = '4GB';"
psql -c "ALTER SYSTEM SET checkpoint_completion_target = 0.9;"
psql -c "SELECT pg_reload_conf();"

# После восстановления вернуть настройки
```

### 3. Логирование

```bash
# Полное логирование процесса
pg_restore -v -d mydb backup.dump 2>&1 | tee restore_$(date +%Y%m%d_%H%M%S).log
```

### 4. Post-restore действия

```bash
# После восстановления
psql -d mydb << EOF
-- Обновить статистику
ANALYZE;

-- Пересоздать статистику планировщика
VACUUM ANALYZE;

-- Проверить целостность
SELECT schemaname, tablename, n_live_tup
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC;
EOF
```

## Типичные проблемы и решения

### Проблема 1: "role does not exist"

```bash
# Решение 1: Создать роль перед восстановлением
psql -c "CREATE ROLE missing_role;"
pg_restore -d mydb backup.dump

# Решение 2: Игнорировать владельцев
pg_restore --no-owner -d mydb backup.dump
```

### Проблема 2: "relation already exists"

```bash
# Решение: Очистка перед восстановлением
pg_restore -c --if-exists -d mydb backup.dump

# Или пересоздать базу
dropdb mydb
createdb mydb
pg_restore -d mydb backup.dump
```

### Проблема 3: "tablespace does not exist"

```bash
# Решение: Игнорировать tablespace
pg_restore --no-tablespaces -d mydb backup.dump
```

### Проблема 4: Slow restore

```bash
# Использовать параллельное восстановление
pg_restore -j 8 -d mydb backup.dump

# Увеличить memory для maintenance операций
psql -c "SET maintenance_work_mem = '1GB';"
```

## Сравнение pg_restore и psql

| Характеристика | pg_restore | psql -f |
|---------------|------------|---------|
| Форматы | custom, directory, tar | plain SQL |
| Параллельное выполнение | Да (-j) | Нет |
| Выборочное восстановление | Да (-t, -n, -L) | Нет |
| Редактирование TOC | Да | Нет |
| Гибкость | Высокая | Низкая |
| Скорость | Быстрее с -j | Зависит |

## Заключение

pg_restore - мощный инструмент для восстановления баз данных PostgreSQL с богатыми возможностями управления процессом восстановления. Ключевые преимущества:

- Параллельное восстановление значительно ускоряет процесс
- Выборочное восстановление позволяет извлекать только нужные данные
- Редактирование TOC дает полный контроль над процессом
- Интеграция с системами автоматизации через скрипты

Для критичных систем рекомендуется всегда тестировать восстановление на отдельной базе перед применением в production.

---

[prev: 01-pg_dump](./01-pg_dump.md) | [next: 03-pg_basebackup](./03-pg_basebackup.md)
