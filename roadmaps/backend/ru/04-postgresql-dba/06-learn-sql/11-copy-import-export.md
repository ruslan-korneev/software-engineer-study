# COPY: Импорт и экспорт данных

[prev: 10-set-operations](./10-set-operations.md) | [next: 01-procedures-and-functions](../07-advanced-sql/01-procedures-and-functions.md)

---

## Обзор команды COPY

**COPY** — это команда PostgreSQL для быстрого импорта и экспорта данных между таблицами и файлами. Существует две формы:

| Команда | Описание | Доступ к файлам |
|---------|----------|-----------------|
| `COPY` | Серверная команда | Файлы на сервере PostgreSQL |
| `\copy` | Клиентская команда psql | Файлы на клиентской машине |

---

## COPY TO — Экспорт данных

### Базовый синтаксис

```sql
-- Экспорт всей таблицы
COPY users TO '/tmp/users.csv';

-- Экспорт с форматированием CSV
COPY users TO '/tmp/users.csv' WITH (FORMAT CSV, HEADER);

-- Экспорт в stdout
COPY users TO STDOUT;
```

### Экспорт выбранных столбцов

```sql
-- Только определенные столбцы
COPY users (id, name, email) TO '/tmp/users_partial.csv'
WITH (FORMAT CSV, HEADER);

-- Экспорт результата запроса
COPY (
    SELECT id, name, email
    FROM users
    WHERE status = 'active'
    ORDER BY created_at DESC
) TO '/tmp/active_users.csv'
WITH (FORMAT CSV, HEADER);
```

### Опции форматирования

```sql
-- CSV с настройками
COPY users TO '/tmp/users.csv' WITH (
    FORMAT CSV,           -- формат файла (CSV, TEXT, BINARY)
    HEADER true,          -- включить заголовки
    DELIMITER ';',        -- разделитель полей (по умолчанию ',')
    QUOTE '"',            -- символ кавычек
    ESCAPE '\',           -- символ экранирования
    NULL 'NULL',          -- представление NULL
    ENCODING 'UTF8'       -- кодировка
);

-- Сжатие (PostgreSQL 16+)
COPY users TO PROGRAM 'gzip > /tmp/users.csv.gz'
WITH (FORMAT CSV, HEADER);

-- Текстовый формат с табуляцией
COPY users TO '/tmp/users.tsv' WITH (
    FORMAT TEXT,
    DELIMITER E'\t'
);
```

### Экспорт в BINARY формат

```sql
-- Бинарный формат (быстрее, но не читаемый)
COPY users TO '/tmp/users.bin' WITH (FORMAT BINARY);
```

---

## COPY FROM — Импорт данных

### Базовый синтаксис

```sql
-- Импорт всей таблицы
COPY users FROM '/tmp/users.csv';

-- Импорт CSV с заголовками
COPY users FROM '/tmp/users.csv' WITH (FORMAT CSV, HEADER);

-- Импорт из stdin
COPY users FROM STDIN WITH (FORMAT CSV, HEADER);
```

### Импорт в определенные столбцы

```sql
-- Указание столбцов
COPY users (name, email) FROM '/tmp/users_partial.csv'
WITH (FORMAT CSV, HEADER);

-- Столбец id будет заполнен из SERIAL/DEFAULT
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    price NUMERIC(10,2)
);

COPY products (name, price) FROM '/tmp/products.csv'
WITH (FORMAT CSV, HEADER);
```

### Опции для импорта

```sql
COPY users FROM '/tmp/users.csv' WITH (
    FORMAT CSV,
    HEADER true,
    DELIMITER ';',
    QUOTE '"',
    ESCAPE '\',
    NULL '',              -- пустая строка как NULL
    ENCODING 'UTF8',
    FORCE_NULL (email, phone)  -- преобразовать пустые строки в NULL
);
```

### Импорт из сжатого файла

```sql
-- Через PROGRAM
COPY users FROM PROGRAM 'zcat /tmp/users.csv.gz'
WITH (FORMAT CSV, HEADER);

-- Или gunzip
COPY users FROM PROGRAM 'gunzip -c /tmp/users.csv.gz'
WITH (FORMAT CSV, HEADER);
```

---

## \copy — Клиентская команда psql

### Отличия от COPY

| COPY | \copy |
|------|-------|
| Выполняется на сервере | Выполняется в psql |
| Доступ к файлам сервера | Доступ к локальным файлам |
| Требует права суперпользователя | Работает с обычными правами |
| Быстрее для локальных файлов | Передача через сетевое соединение |

### Примеры использования

```sql
-- В psql (обратите внимание: БЕЗ точки с запятой!)
\copy users TO '/home/user/users.csv' WITH (FORMAT CSV, HEADER)

\copy users FROM '/home/user/users.csv' WITH (FORMAT CSV, HEADER)

-- Экспорт результата запроса
\copy (SELECT * FROM users WHERE active = true) TO '~/active_users.csv' CSV HEADER
```

---

## Обработка ошибок

### WHERE-условие (PostgreSQL 12+)

```sql
-- Импорт только строк, удовлетворяющих условию
COPY users FROM '/tmp/users.csv' WITH (FORMAT CSV, HEADER)
WHERE status != 'deleted';
```

### ON_ERROR (PostgreSQL 17+)

```sql
-- Пропустить проблемные строки
COPY users FROM '/tmp/users.csv' WITH (
    FORMAT CSV,
    HEADER,
    ON_ERROR ignore
);
```

### Обработка в старых версиях

```sql
-- Использовать промежуточную таблицу
CREATE TEMP TABLE temp_import (LIKE users);

-- Импорт в промежуточную таблицу
COPY temp_import FROM '/tmp/users.csv' WITH (FORMAT CSV, HEADER);

-- Перенос валидных данных
INSERT INTO users
SELECT * FROM temp_import
WHERE email ~ '^[^@]+@[^@]+\.[^@]+$';

DROP TABLE temp_import;
```

---

## Работа с внешними программами

### COPY TO PROGRAM

```sql
-- Сжатие при экспорте
COPY users TO PROGRAM 'gzip > /tmp/users.csv.gz'
WITH (FORMAT CSV, HEADER);

-- Отправка на S3 (требует aws cli)
COPY users TO PROGRAM 'aws s3 cp - s3://bucket/users.csv'
WITH (FORMAT CSV, HEADER);

-- Отправка по email (пример)
COPY (SELECT * FROM reports WHERE date = CURRENT_DATE)
TO PROGRAM 'mail -s "Daily Report" admin@example.com'
WITH (FORMAT CSV, HEADER);
```

### COPY FROM PROGRAM

```sql
-- Импорт из сжатого файла
COPY users FROM PROGRAM 'zcat /tmp/users.csv.gz'
WITH (FORMAT CSV, HEADER);

-- Импорт с S3
COPY users FROM PROGRAM 'aws s3 cp s3://bucket/users.csv -'
WITH (FORMAT CSV, HEADER);

-- Импорт с преобразованием (например, изменение кодировки)
COPY users FROM PROGRAM 'iconv -f WINDOWS-1251 -t UTF-8 /tmp/users.csv'
WITH (FORMAT CSV, HEADER);

-- Импорт с предобработкой
COPY users FROM PROGRAM 'cat /tmp/users.csv | sed "s/NULL//g"'
WITH (FORMAT CSV, HEADER);
```

---

## Практические примеры

### Резервное копирование таблицы

```sql
-- Полная копия с данными и структурой (через pg_dump лучше)
-- Но для простого бэкапа данных:
COPY users TO '/backup/users_backup_20240101.csv'
WITH (FORMAT CSV, HEADER);

-- Инкрементальный бэкап
COPY (
    SELECT * FROM orders
    WHERE created_at >= CURRENT_DATE - INTERVAL '1 day'
) TO '/backup/orders_daily.csv'
WITH (FORMAT CSV, HEADER);
```

### Миграция между серверами

```sql
-- На исходном сервере
COPY users TO '/tmp/users_export.csv' WITH (FORMAT CSV, HEADER);

-- Передача файла (scp, rsync и т.д.)

-- На целевом сервере
COPY users FROM '/tmp/users_export.csv' WITH (FORMAT CSV, HEADER);
```

### Загрузка справочных данных

```sql
-- Загрузка стран
COPY countries (code, name, region) FROM STDIN WITH (FORMAT CSV);
RU,Russia,Europe
US,United States,North America
CN,China,Asia
\.

-- Загрузка конфигурации из файла
COPY settings FROM '/etc/myapp/initial_settings.csv'
WITH (FORMAT CSV, HEADER);
```

### ETL-процесс

```sql
-- Этап 1: Загрузка в staging таблицу
CREATE TEMP TABLE staging_orders (
    order_id TEXT,
    customer_email TEXT,
    amount TEXT,
    order_date TEXT
);

COPY staging_orders FROM '/tmp/new_orders.csv'
WITH (FORMAT CSV, HEADER);

-- Этап 2: Преобразование и загрузка
INSERT INTO orders (id, customer_id, amount, order_date)
SELECT
    order_id::INTEGER,
    c.id,
    amount::NUMERIC(10,2),
    order_date::DATE
FROM staging_orders s
JOIN customers c ON c.email = s.customer_email
WHERE amount::NUMERIC > 0;

DROP TABLE staging_orders;
```

### Экспорт для аналитики

```sql
-- Агрегированные данные для BI
COPY (
    SELECT
        DATE_TRUNC('day', created_at) AS date,
        category,
        COUNT(*) AS order_count,
        SUM(amount) AS total_amount,
        AVG(amount) AS avg_amount
    FROM orders
    GROUP BY DATE_TRUNC('day', created_at), category
    ORDER BY date, category
) TO '/analytics/daily_orders.csv'
WITH (FORMAT CSV, HEADER);
```

---

## Оптимизация производительности

### Для больших импортов

```sql
-- 1. Отключить автовакуум на время импорта
ALTER TABLE large_table SET (autovacuum_enabled = false);

-- 2. Отключить триггеры
ALTER TABLE large_table DISABLE TRIGGER ALL;

-- 3. Удалить индексы перед импортом
DROP INDEX idx_large_table_column;

-- 4. Выполнить COPY
COPY large_table FROM '/tmp/large_data.csv' WITH (FORMAT CSV);

-- 5. Создать индексы заново
CREATE INDEX idx_large_table_column ON large_table(column);

-- 6. Включить триггеры
ALTER TABLE large_table ENABLE TRIGGER ALL;

-- 7. Включить автовакуум
ALTER TABLE large_table SET (autovacuum_enabled = true);

-- 8. Обновить статистику
ANALYZE large_table;
```

### Параллельный импорт

```sql
-- Разбить файл на части и импортировать параллельно
-- В bash:
-- split -l 1000000 large_file.csv part_

-- Параллельный импорт (в разных сессиях или через GNU parallel)
COPY table FROM '/tmp/part_aa' WITH (FORMAT CSV);
COPY table FROM '/tmp/part_ab' WITH (FORMAT CSV);
-- и т.д.
```

### Unlogged таблицы для промежуточных данных

```sql
-- Быстрее, но не сохраняется при сбое
CREATE UNLOGGED TABLE staging_data (...);
COPY staging_data FROM '/tmp/data.csv' WITH (FORMAT CSV, HEADER);

-- Перенос в основную таблицу
INSERT INTO main_table SELECT * FROM staging_data;
DROP TABLE staging_data;
```

---

## Форматы данных

### CSV формат

```
id,name,email,created_at
1,John Doe,john@example.com,2024-01-15
2,Jane Smith,jane@example.com,2024-01-16
3,"Bob, Jr.",bob@example.com,2024-01-17
```

```sql
COPY users FROM '/tmp/users.csv' WITH (
    FORMAT CSV,
    HEADER true,
    DELIMITER ',',
    QUOTE '"'
);
```

### TEXT формат (по умолчанию)

```
1	John Doe	john@example.com	2024-01-15
2	Jane Smith	jane@example.com	2024-01-16
3	Bob, Jr.	bob@example.com	2024-01-17
```

```sql
COPY users FROM '/tmp/users.txt' WITH (
    FORMAT TEXT,
    DELIMITER E'\t',
    NULL '\N'
);
```

### BINARY формат

```sql
-- Бинарный формат — самый быстрый, но непортабельный
COPY users TO '/tmp/users.bin' WITH (FORMAT BINARY);
COPY users FROM '/tmp/users.bin' WITH (FORMAT BINARY);

-- Внимание: бинарный формат зависит от версии PostgreSQL
-- и архитектуры системы!
```

---

## Работа со специальными типами

### JSON

```sql
-- Экспорт таблицы в JSON
COPY (
    SELECT json_agg(row_to_json(u))
    FROM users u
) TO '/tmp/users.json';

-- Или построчно JSONL (JSON Lines)
COPY (
    SELECT row_to_json(u)
    FROM users u
) TO '/tmp/users.jsonl';
```

### Arrays

```sql
-- Массивы экспортируются в формате PostgreSQL
-- {value1,value2,value3}
COPY (SELECT id, tags FROM products) TO '/tmp/products.csv' CSV HEADER;

-- Результат:
-- id,tags
-- 1,"{tag1,tag2,tag3}"
-- 2,"{tag4,tag5}"
```

### Timestamps

```sql
-- Контроль формата даты
SET datestyle = 'ISO, YMD';
COPY orders TO '/tmp/orders.csv' CSV HEADER;

-- Или преобразование в запросе
COPY (
    SELECT
        id,
        TO_CHAR(created_at, 'YYYY-MM-DD HH24:MI:SS') AS created_at
    FROM orders
) TO '/tmp/orders.csv' CSV HEADER;
```

---

## Типичные ошибки

### 1. Неправильный путь к файлу

```sql
-- Ошибка: файл недоступен серверу
COPY users FROM '/home/myuser/data.csv';
-- ERROR: could not open file: Permission denied

-- Решение: использовать /tmp или настроить права
COPY users FROM '/tmp/data.csv';

-- Или использовать \copy в psql
\copy users FROM '/home/myuser/data.csv' CSV HEADER
```

### 2. Несоответствие столбцов

```sql
-- Ошибка: количество столбцов не совпадает
COPY users FROM '/tmp/users.csv';
-- ERROR: extra data after last expected column

-- Решение: указать столбцы явно
COPY users (name, email) FROM '/tmp/users.csv' CSV HEADER;
```

### 3. Проблемы с кодировкой

```sql
-- Ошибка при неправильной кодировке
COPY users FROM '/tmp/users.csv';
-- ERROR: invalid byte sequence for encoding "UTF8"

-- Решение: указать правильную кодировку
COPY users FROM '/tmp/users.csv' WITH (ENCODING 'LATIN1');

-- Или преобразовать через PROGRAM
COPY users FROM PROGRAM 'iconv -f WINDOWS-1251 -t UTF-8 /tmp/users.csv'
CSV HEADER;
```

### 4. NULL значения

```sql
-- Пустые строки не становятся NULL автоматически
-- Решение: использовать FORCE_NULL
COPY users FROM '/tmp/users.csv' WITH (
    FORMAT CSV,
    HEADER,
    FORCE_NULL (middle_name, phone)
);
```

---

## Безопасность

### Ограничения прав

```sql
-- Только суперпользователь может использовать COPY с файлами
-- Обычные пользователи могут:
-- 1. Использовать \copy в psql
-- 2. Использовать COPY с STDIN/STDOUT

-- Предоставить права на использование COPY
-- (только для определенных директорий, требует настройки pg_hba.conf)
```

### Безопасный импорт

```sql
-- Всегда проверяйте данные в staging таблице
CREATE TEMP TABLE staging AS
SELECT * FROM users WHERE false;

COPY staging FROM '/tmp/users.csv' CSV HEADER;

-- Проверка данных
SELECT * FROM staging WHERE email !~ '^[^@]+@[^@]+\.[^@]+$';

-- Только потом импорт
INSERT INTO users SELECT * FROM staging;
```

---

## Лучшие практики

1. **Используйте \copy** для работы с локальными файлами без прав суперпользователя
2. **Указывайте столбцы явно** для надежности
3. **Используйте HEADER** для самодокументирования файлов
4. **Отключайте индексы и триггеры** при массовом импорте
5. **Проверяйте данные** через staging таблицу
6. **Используйте транзакции** для атомарности импорта
7. **Сжимайте большие файлы** через PROGRAM
8. **Логируйте операции** импорта/экспорта

---

[prev: 10-set-operations](./10-set-operations.md) | [next: 01-procedures-and-functions](../07-advanced-sql/01-procedures-and-functions.md)
