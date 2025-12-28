# DDL Queries (Data Definition Language)

[prev: 03-using-pg_ctlcluster](../05-managing-postgres/03-using-pg_ctlcluster.md) | [next: 02-dml-queries](./02-dml-queries.md)

---

## Что такое DDL?

**DDL (Data Definition Language)** — это подмножество SQL, предназначенное для определения и управления структурой базы данных. DDL-команды используются для создания, изменения и удаления объектов базы данных: таблиц, индексов, схем, представлений и других объектов.

### Основные DDL-команды:
- `CREATE` — создание объектов
- `ALTER` — изменение объектов
- `DROP` — удаление объектов
- `TRUNCATE` — быстрая очистка таблицы
- `COMMENT` — добавление комментариев к объектам

---

## CREATE — Создание объектов

### Создание базы данных

```sql
-- Создание базы данных
CREATE DATABASE myapp;

-- С указанием параметров
CREATE DATABASE myapp
    WITH OWNER = postgres
    ENCODING = 'UTF8'
    LC_COLLATE = 'ru_RU.UTF-8'
    LC_CTYPE = 'ru_RU.UTF-8'
    TEMPLATE = template0;
```

### Создание схемы

```sql
-- Создание схемы
CREATE SCHEMA IF NOT EXISTS sales;

-- Создание схемы с владельцем
CREATE SCHEMA inventory AUTHORIZATION admin_user;
```

### Создание таблицы

```sql
-- Базовое создание таблицы
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица с различными ограничениями
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    order_date DATE NOT NULL DEFAULT CURRENT_DATE,
    total_amount NUMERIC(10, 2) CHECK (total_amount >= 0),
    status VARCHAR(20) DEFAULT 'pending',

    -- Составной уникальный индекс
    CONSTRAINT unique_user_order UNIQUE (user_id, order_date)
);

-- Таблица с партиционированием
CREATE TABLE logs (
    id BIGSERIAL,
    log_date DATE NOT NULL,
    message TEXT,
    level VARCHAR(10)
) PARTITION BY RANGE (log_date);

-- Создание партиции
CREATE TABLE logs_2024_01 PARTITION OF logs
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

### Создание индексов

```sql
-- Простой индекс
CREATE INDEX idx_users_email ON users(email);

-- Уникальный индекс
CREATE UNIQUE INDEX idx_users_username ON users(username);

-- Составной индекс
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);

-- Частичный индекс
CREATE INDEX idx_active_users ON users(email)
    WHERE is_active = true;

-- Индекс с использованием выражения
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- GIN-индекс для полнотекстового поиска
CREATE INDEX idx_products_search ON products
    USING GIN (to_tsvector('russian', description));

-- BRIN-индекс для больших таблиц с последовательными данными
CREATE INDEX idx_logs_date ON logs USING BRIN (log_date);
```

### Создание представлений (Views)

```sql
-- Обычное представление
CREATE VIEW active_users AS
    SELECT id, username, email
    FROM users
    WHERE is_active = true;

-- Представление с CHECK OPTION
CREATE VIEW premium_users AS
    SELECT * FROM users WHERE subscription_type = 'premium'
    WITH CHECK OPTION;

-- Материализованное представление
CREATE MATERIALIZED VIEW monthly_sales AS
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        SUM(total_amount) AS total_sales
    FROM orders
    GROUP BY DATE_TRUNC('month', order_date);
```

### Создание последовательностей

```sql
-- Создание последовательности
CREATE SEQUENCE order_number_seq
    START WITH 1000
    INCREMENT BY 1
    NO MAXVALUE
    CACHE 10;

-- Использование последовательности
SELECT nextval('order_number_seq');
```

---

## ALTER — Изменение объектов

### Изменение таблицы

```sql
-- Добавление столбца
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Добавление столбца с ограничением
ALTER TABLE users ADD COLUMN age INTEGER CHECK (age >= 0 AND age <= 150);

-- Удаление столбца
ALTER TABLE users DROP COLUMN phone;

-- Изменение типа данных столбца
ALTER TABLE users ALTER COLUMN username TYPE VARCHAR(100);

-- Установка значения по умолчанию
ALTER TABLE users ALTER COLUMN is_active SET DEFAULT true;

-- Удаление значения по умолчанию
ALTER TABLE users ALTER COLUMN is_active DROP DEFAULT;

-- Добавление NOT NULL
ALTER TABLE users ALTER COLUMN email SET NOT NULL;

-- Удаление NOT NULL
ALTER TABLE users ALTER COLUMN phone DROP NOT NULL;

-- Переименование столбца
ALTER TABLE users RENAME COLUMN username TO login;

-- Переименование таблицы
ALTER TABLE users RENAME TO app_users;

-- Добавление ограничения
ALTER TABLE orders ADD CONSTRAINT fk_user
    FOREIGN KEY (user_id) REFERENCES users(id);

-- Удаление ограничения
ALTER TABLE orders DROP CONSTRAINT fk_user;

-- Добавление первичного ключа
ALTER TABLE products ADD PRIMARY KEY (id);
```

### Изменение индекса

```sql
-- Переименование индекса
ALTER INDEX idx_users_email RENAME TO idx_email;

-- Перемещение индекса в другое табличное пространство
ALTER INDEX idx_email SET TABLESPACE fast_ssd;
```

### Изменение последовательности

```sql
-- Изменение параметров последовательности
ALTER SEQUENCE order_number_seq RESTART WITH 5000;
ALTER SEQUENCE order_number_seq INCREMENT BY 5;
```

---

## DROP — Удаление объектов

```sql
-- Удаление таблицы
DROP TABLE users;

-- Удаление с проверкой существования
DROP TABLE IF EXISTS users;

-- Каскадное удаление (удаляет зависимые объекты)
DROP TABLE users CASCADE;

-- Удаление нескольких таблиц
DROP TABLE orders, order_items, payments;

-- Удаление индекса
DROP INDEX IF EXISTS idx_users_email;

-- Удаление представления
DROP VIEW active_users;
DROP MATERIALIZED VIEW monthly_sales;

-- Удаление схемы
DROP SCHEMA IF EXISTS sales CASCADE;

-- Удаление базы данных
DROP DATABASE myapp;
```

---

## TRUNCATE — Быстрая очистка таблицы

```sql
-- Очистка таблицы
TRUNCATE TABLE logs;

-- Очистка с каскадом (очищает связанные таблицы)
TRUNCATE TABLE users CASCADE;

-- Очистка нескольких таблиц
TRUNCATE TABLE orders, order_items;

-- Очистка с перезапуском последовательностей
TRUNCATE TABLE users RESTART IDENTITY;

-- Очистка с сохранением последовательностей
TRUNCATE TABLE users CONTINUE IDENTITY;
```

### Отличие TRUNCATE от DELETE:
| TRUNCATE | DELETE |
|----------|--------|
| Мгновенная очистка | Удаляет строки по одной |
| Не записывает в WAL каждую строку | Записывает в WAL каждую строку |
| Нельзя откатить без MVCC | Можно откатить в транзакции |
| Сбрасывает счетчики SERIAL | Не сбрасывает счетчики |
| Не активирует триггеры на уровне строк | Активирует триггеры |

---

## COMMENT — Комментарии к объектам

```sql
-- Комментарий к таблице
COMMENT ON TABLE users IS 'Таблица пользователей системы';

-- Комментарий к столбцу
COMMENT ON COLUMN users.email IS 'Уникальный email адрес пользователя';

-- Комментарий к функции
COMMENT ON FUNCTION calculate_total(integer) IS 'Вычисляет итоговую сумму заказа';

-- Удаление комментария
COMMENT ON TABLE users IS NULL;
```

---

## Лучшие практики DDL

### 1. Именование объектов

```sql
-- Хорошо: понятные имена в snake_case
CREATE TABLE user_orders (
    order_id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Плохо: непонятные сокращения
CREATE TABLE uo (
    oid SERIAL,
    uid INTEGER,
    ca TIMESTAMP
);
```

### 2. Всегда используйте IF EXISTS / IF NOT EXISTS

```sql
-- Безопасное создание
CREATE TABLE IF NOT EXISTS users (...);
CREATE INDEX IF NOT EXISTS idx_email ON users(email);

-- Безопасное удаление
DROP TABLE IF EXISTS temp_data;
```

### 3. Транзакции для DDL-операций

```sql
BEGIN;

-- Изменения структуры
ALTER TABLE users ADD COLUMN new_field VARCHAR(50);
CREATE INDEX idx_new_field ON users(new_field);

-- Проверка
SELECT * FROM users LIMIT 1;

-- Фиксация или откат
COMMIT;
-- или ROLLBACK;
```

### 4. Версионирование миграций

```sql
-- Миграция v001_create_users.sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL
);

-- Миграция v002_add_username.sql
ALTER TABLE users ADD COLUMN username VARCHAR(50);
CREATE UNIQUE INDEX idx_users_username ON users(username);
```

---

## Типичные ошибки

### 1. Забыть про каскадные зависимости

```sql
-- Ошибка: не удалится, если есть ссылки
DROP TABLE users;  -- ERROR: cannot drop table users because other objects depend on it

-- Решение
DROP TABLE users CASCADE;
```

### 2. Изменение типа данных с потерей данных

```sql
-- Опасно: может потерять данные
ALTER TABLE users ALTER COLUMN age TYPE SMALLINT;

-- Безопаснее: использовать USING для конвертации
ALTER TABLE users ALTER COLUMN age TYPE SMALLINT USING age::SMALLINT;
```

### 3. Блокировки при ALTER TABLE

```sql
-- Добавление NOT NULL блокирует таблицу для сканирования
-- На больших таблицах лучше:

-- 1. Добавить столбец без NOT NULL
ALTER TABLE users ADD COLUMN status VARCHAR(20);

-- 2. Заполнить данные
UPDATE users SET status = 'active' WHERE status IS NULL;

-- 3. Добавить NOT NULL
ALTER TABLE users ALTER COLUMN status SET NOT NULL;
```

### 4. Создание индекса на продакшене

```sql
-- Блокирует таблицу на время создания
CREATE INDEX idx_large_table ON large_table(column);

-- Лучше: конкурентное создание индекса
CREATE INDEX CONCURRENTLY idx_large_table ON large_table(column);
```

---

## Системные каталоги для DDL

```sql
-- Список таблиц
SELECT tablename FROM pg_tables WHERE schemaname = 'public';

-- Список столбцов таблицы
SELECT column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_name = 'users';

-- Список индексов
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'users';

-- Список ограничений
SELECT conname, contype, pg_get_constraintdef(oid)
FROM pg_constraint
WHERE conrelid = 'users'::regclass;
```

---

[prev: 03-using-pg_ctlcluster](../05-managing-postgres/03-using-pg_ctlcluster.md) | [next: 02-dml-queries](./02-dml-queries.md)
