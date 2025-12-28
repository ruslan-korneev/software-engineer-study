# Массовая загрузка данных (Bulk Loading)

[prev: 04-migration-tools](../15-automation/04-migration-tools.md) | [next: 02-data-partitioning](./02-data-partitioning.md)

---

## Введение

Массовая загрузка данных (Bulk Loading) - это процесс эффективной загрузки больших объемов данных в PostgreSQL. Стандартные INSERT-операции неэффективны для миллионов записей, поэтому PostgreSQL предоставляет специализированные инструменты для быстрой загрузки данных.

## Команда COPY

COPY - это самый быстрый способ загрузки данных в PostgreSQL. Она работает напрямую с файловой системой сервера.

### Синтаксис COPY

```sql
-- Загрузка данных из файла в таблицу
COPY table_name FROM '/path/to/file.csv' WITH (FORMAT csv, HEADER true);

-- Выгрузка данных из таблицы в файл
COPY table_name TO '/path/to/file.csv' WITH (FORMAT csv, HEADER true);

-- COPY с указанием конкретных столбцов
COPY table_name (column1, column2, column3)
FROM '/path/to/file.csv'
WITH (FORMAT csv, HEADER true);
```

### Параметры COPY

```sql
-- Полный синтаксис с параметрами
COPY users FROM '/data/users.csv' WITH (
    FORMAT csv,           -- Формат: csv, text, binary
    HEADER true,          -- Первая строка - заголовки
    DELIMITER ',',        -- Разделитель полей
    NULL 'NULL',          -- Представление NULL
    QUOTE '"',            -- Символ кавычек
    ESCAPE '\',           -- Escape-символ
    ENCODING 'UTF8'       -- Кодировка файла
);
```

### Пример использования COPY

```sql
-- Создание таблицы для загрузки
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    product_name VARCHAR(255),
    quantity INTEGER,
    price DECIMAL(10,2),
    order_date DATE
);

-- Загрузка данных из CSV
COPY orders (customer_id, product_name, quantity, price, order_date)
FROM '/tmp/orders.csv'
WITH (FORMAT csv, HEADER true, DELIMITER ',');

-- Проверка количества загруженных записей
SELECT COUNT(*) FROM orders;
```

## Команда \copy в psql

В отличие от COPY, команда `\copy` работает на стороне клиента и не требует доступа к файловой системе сервера.

```sql
-- Загрузка данных с клиента
\copy users FROM '/local/path/users.csv' WITH (FORMAT csv, HEADER true);

-- Выгрузка данных на клиент
\copy users TO '/local/path/users_backup.csv' WITH (FORMAT csv, HEADER true);
```

### Сравнение COPY и \copy

| Характеристика | COPY | \copy |
|---------------|------|-------|
| Выполнение | На сервере | На клиенте |
| Скорость | Быстрее | Медленнее |
| Права доступа | Требует SUPERUSER или pg_read_server_files | Обычные права |
| Путь к файлу | Серверный | Клиентский |

## Оптимизация массовой загрузки

### 1. Отключение индексов и ограничений

```sql
-- Отключение триггеров
ALTER TABLE orders DISABLE TRIGGER ALL;

-- Удаление индексов перед загрузкой
DROP INDEX IF EXISTS idx_orders_customer;
DROP INDEX IF EXISTS idx_orders_date;

-- Загрузка данных
COPY orders FROM '/tmp/orders.csv' WITH (FORMAT csv, HEADER true);

-- Пересоздание индексов после загрузки
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_date ON orders(order_date);

-- Включение триггеров
ALTER TABLE orders ENABLE TRIGGER ALL;
```

### 2. Настройка параметров для массовой загрузки

```sql
-- Увеличение work_mem для сессии
SET work_mem = '256MB';

-- Увеличение maintenance_work_mem для создания индексов
SET maintenance_work_mem = '1GB';

-- Отключение синхронной записи (только для загрузки!)
SET synchronous_commit = OFF;

-- Увеличение checkpoint_segments (для старых версий)
-- или max_wal_size для новых версий
```

### 3. Использование UNLOGGED таблиц

```sql
-- Создание UNLOGGED таблицы (не пишется в WAL)
CREATE UNLOGGED TABLE temp_orders (
    id INTEGER,
    customer_id INTEGER,
    amount DECIMAL(10,2)
);

-- Быстрая загрузка
COPY temp_orders FROM '/tmp/orders.csv' WITH (FORMAT csv);

-- После загрузки можно преобразовать в обычную таблицу
-- или перенести данные
INSERT INTO orders SELECT * FROM temp_orders;
```

## Bulk INSERT с использованием многозначных INSERT

```sql
-- Вместо множества отдельных INSERT
INSERT INTO users (name, email) VALUES ('User1', 'user1@example.com');
INSERT INTO users (name, email) VALUES ('User2', 'user2@example.com');
INSERT INTO users (name, email) VALUES ('User3', 'user3@example.com');

-- Используйте многозначный INSERT
INSERT INTO users (name, email) VALUES
    ('User1', 'user1@example.com'),
    ('User2', 'user2@example.com'),
    ('User3', 'user3@example.com');
```

### Batch INSERT из приложения (Python пример)

```python
import psycopg2
from psycopg2.extras import execute_values

conn = psycopg2.connect("dbname=mydb user=postgres")
cursor = conn.cursor()

# Данные для вставки
data = [
    (1, 'Product A', 100),
    (2, 'Product B', 200),
    (3, 'Product C', 150),
    # ... тысячи записей
]

# Эффективная массовая вставка
execute_values(
    cursor,
    "INSERT INTO products (id, name, price) VALUES %s",
    data,
    page_size=1000  # Размер batch
)

conn.commit()
```

### COPY из приложения (Python)

```python
import psycopg2
from io import StringIO

conn = psycopg2.connect("dbname=mydb user=postgres")
cursor = conn.cursor()

# Создание данных в памяти
buffer = StringIO()
for i in range(1000000):
    buffer.write(f"{i}\tProduct_{i}\t{i*10}\n")
buffer.seek(0)

# Быстрая загрузка через COPY
cursor.copy_from(buffer, 'products', columns=('id', 'name', 'price'))
conn.commit()
```

## pg_bulkload - расширение для сверхбыстрой загрузки

pg_bulkload - это расширение, которое обходит WAL для максимальной скорости.

```bash
# Установка pg_bulkload
# На Ubuntu/Debian
sudo apt-get install postgresql-15-pg-bulkload

# Использование
pg_bulkload -d mydb -i /path/to/data.csv -O "orders" -l /tmp/bulkload.log
```

### Конфигурационный файл pg_bulkload

```ini
# bulkload.ctl
OUTPUT = orders
INPUT = /path/to/data.csv
TYPE = CSV
DELIMITER = ,
QUOTE = "\""
ESCAPE = "\\"
NULL = ""
SKIP = 1
WRITER = DIRECT
TRUNCATE = YES
```

## Foreign Data Wrapper для загрузки

```sql
-- Создание расширения file_fdw
CREATE EXTENSION file_fdw;

-- Создание сервера
CREATE SERVER csv_server FOREIGN DATA WRAPPER file_fdw;

-- Создание внешней таблицы
CREATE FOREIGN TABLE external_orders (
    id INTEGER,
    customer_id INTEGER,
    product_name VARCHAR(255),
    quantity INTEGER,
    price DECIMAL(10,2)
) SERVER csv_server
OPTIONS (filename '/data/orders.csv', format 'csv', header 'true');

-- Загрузка данных через INSERT ... SELECT
INSERT INTO orders SELECT * FROM external_orders;
```

## Best Practices

### 1. Подготовка к массовой загрузке

```sql
-- Чеклист перед загрузкой
BEGIN;

-- 1. Сохранить текущие настройки
SHOW work_mem;
SHOW maintenance_work_mem;

-- 2. Увеличить память
SET work_mem = '512MB';
SET maintenance_work_mem = '2GB';

-- 3. Отключить автовакуум для таблицы
ALTER TABLE orders SET (autovacuum_enabled = false);

-- 4. Удалить индексы
-- ... (см. выше)

-- 5. Выполнить загрузку
COPY orders FROM '/tmp/orders.csv' WITH (FORMAT csv);

-- 6. Пересоздать индексы
-- ...

-- 7. Включить автовакуум
ALTER TABLE orders SET (autovacuum_enabled = true);

-- 8. Запустить ANALYZE
ANALYZE orders;

COMMIT;
```

### 2. Обработка ошибок при загрузке

```sql
-- Создание таблицы для ошибочных записей
CREATE TABLE orders_errors (
    line_number BIGINT,
    raw_data TEXT,
    error_message TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- В PostgreSQL 17+ можно использовать ON_ERROR
COPY orders FROM '/tmp/orders.csv'
WITH (FORMAT csv, ON_ERROR ignore);
```

### 3. Мониторинг процесса загрузки

```sql
-- Проверка прогресса COPY в pg_stat_progress_copy (PostgreSQL 14+)
SELECT
    datname,
    relname,
    command,
    type,
    bytes_processed,
    bytes_total,
    tuples_processed,
    tuples_excluded
FROM pg_stat_progress_copy;
```

## Сравнение методов загрузки

| Метод | Скорость | Сложность | Транзакционность |
|-------|----------|-----------|------------------|
| INSERT | Низкая | Простая | Да |
| Multi-value INSERT | Средняя | Простая | Да |
| COPY | Высокая | Средняя | Да |
| pg_bulkload | Очень высокая | Высокая | Нет |

## Когда использовать

- **COPY** - загрузка данных из файлов, миграции, импорт из других систем
- **Multi-value INSERT** - загрузка небольших объемов из приложения
- **pg_bulkload** - загрузка огромных объемов данных, когда критична скорость
- **FDW** - регулярная интеграция с внешними источниками данных

## Типичные ошибки

1. **Не отключать индексы** - каждая вставка обновляет все индексы
2. **Слишком маленький batch size** - увеличьте размер пакета
3. **Синхронный коммит** - отключите для массовой загрузки
4. **Игнорирование ANALYZE** - после загрузки обязательно обновите статистику
5. **Неправильная кодировка файла** - всегда указывайте ENCODING

---

[prev: 04-migration-tools](../15-automation/04-migration-tools.md) | [next: 02-data-partitioning](./02-data-partitioning.md)
