# Ограничения (Constraints) в PostgreSQL

## Введение

**Ограничения** (Constraints) — это правила, которые обеспечивают целостность данных на уровне базы данных. Они гарантируют, что данные соответствуют определённым условиям, независимо от того, как они были введены.

## Типы ограничений

PostgreSQL поддерживает следующие типы ограничений:

| Ограничение | Назначение |
|-------------|-----------|
| `NOT NULL` | Запрещает NULL значения |
| `UNIQUE` | Уникальные значения в столбце(ах) |
| `PRIMARY KEY` | NOT NULL + UNIQUE, идентификатор строки |
| `FOREIGN KEY` | Ссылочная целостность между таблицами |
| `CHECK` | Проверка произвольного условия |
| `EXCLUSION` | Исключение конфликтующих данных |
| `DEFAULT` | Значение по умолчанию (технически не constraint) |

## NOT NULL

Гарантирует, что столбец не может содержать NULL:

```sql
-- При создании таблицы
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL,       -- обязательное поле
    username VARCHAR(50) NOT NULL,
    phone VARCHAR(20)                  -- опциональное поле (NULL разрешён)
);

-- Добавление NOT NULL к существующему столбцу
-- Сначала нужно заполнить NULL значения!
UPDATE users SET phone = 'unknown' WHERE phone IS NULL;
ALTER TABLE users ALTER COLUMN phone SET NOT NULL;

-- Удаление NOT NULL
ALTER TABLE users ALTER COLUMN phone DROP NOT NULL;

-- NOT NULL нельзя именовать, но можно использовать CHECK
ALTER TABLE users ADD CONSTRAINT users_phone_not_null CHECK (phone IS NOT NULL);
```

## UNIQUE

Гарантирует уникальность значений в столбце или комбинации столбцов:

```sql
-- На одном столбце
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    sku VARCHAR(50) UNIQUE,
    name VARCHAR(100)
);

-- На нескольких столбцах
CREATE TABLE subscriptions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    plan_type VARCHAR(20) NOT NULL,
    UNIQUE (user_id, plan_type)  -- комбинация должна быть уникальной
);

-- Именованное ограничение
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255),
    CONSTRAINT employees_email_unique UNIQUE (email)
);

-- Добавление к существующей таблице
ALTER TABLE products ADD CONSTRAINT products_name_unique UNIQUE (name);

-- UNIQUE с NULL: NULL != NULL, поэтому несколько NULL разрешены
INSERT INTO products (sku, name) VALUES (NULL, 'Product 1');  -- OK
INSERT INTO products (sku, name) VALUES (NULL, 'Product 2');  -- OK

-- PostgreSQL 15+: UNIQUE NULLS NOT DISTINCT
ALTER TABLE products ADD CONSTRAINT products_sku_unique
    UNIQUE NULLS NOT DISTINCT (sku);  -- только один NULL разрешён
```

## PRIMARY KEY

Первичный ключ = NOT NULL + UNIQUE. Каждая таблица должна иметь первичный ключ:

```sql
-- Простой первичный ключ
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

-- Составной первичный ключ
CREATE TABLE order_items (
    order_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL,
    PRIMARY KEY (order_id, product_id)
);

-- Именованный первичный ключ
CREATE TABLE tags (
    id SERIAL,
    name VARCHAR(50) NOT NULL,
    CONSTRAINT tags_pkey PRIMARY KEY (id)
);

-- Добавление первичного ключа к существующей таблице
ALTER TABLE existing_table ADD PRIMARY KEY (id);

-- Или именованный
ALTER TABLE existing_table ADD CONSTRAINT existing_table_pkey PRIMARY KEY (id);
```

### Рекомендации по первичным ключам

```sql
-- Современный подход: IDENTITY вместо SERIAL
CREATE TABLE users (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email VARCHAR(255) NOT NULL
);

-- UUID как первичный ключ
CREATE TABLE sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id INTEGER NOT NULL,
    expires_at TIMESTAMPTZ
);

-- Составной естественный ключ (когда имеет смысл)
CREATE TABLE country_translations (
    country_code CHAR(2) NOT NULL,
    language_code CHAR(2) NOT NULL,
    name VARCHAR(100) NOT NULL,
    PRIMARY KEY (country_code, language_code)
);
```

## FOREIGN KEY

Обеспечивает ссылочную целостность между таблицами:

```sql
-- Базовый внешний ключ
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),  -- краткий синтаксис
    order_date DATE DEFAULT CURRENT_DATE
);

-- Полный синтаксис с именем
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL,
    CONSTRAINT fk_order FOREIGN KEY (order_id) REFERENCES orders(id),
    CONSTRAINT fk_product FOREIGN KEY (product_id) REFERENCES products(id)
);

-- Составной внешний ключ
CREATE TABLE order_shipments (
    id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    shipped_at TIMESTAMPTZ,
    FOREIGN KEY (order_id, product_id) REFERENCES order_items(order_id, product_id)
);
```

### Действия при удалении/обновлении (Referential Actions)

```sql
CREATE TABLE comments (
    id SERIAL PRIMARY KEY,
    post_id INTEGER NOT NULL,
    content TEXT NOT NULL,

    -- ON DELETE: что делать при удалении родительской записи
    -- ON UPDATE: что делать при изменении родительского ключа

    FOREIGN KEY (post_id) REFERENCES posts(id)
        ON DELETE CASCADE    -- удалить комментарии вместе с постом
        ON UPDATE CASCADE    -- обновить ссылку при изменении id поста
);
```

Доступные действия:

| Действие | Описание |
|----------|----------|
| `NO ACTION` | Ошибка при нарушении (по умолчанию, проверка отложенная) |
| `RESTRICT` | Ошибка при нарушении (немедленная проверка) |
| `CASCADE` | Удалить/обновить зависимые записи |
| `SET NULL` | Установить NULL в зависимых записях |
| `SET DEFAULT` | Установить значение по умолчанию |

```sql
-- Практические примеры
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    manager_id INTEGER,
    department_id INTEGER,

    -- Если менеджер удалён, установить NULL
    FOREIGN KEY (manager_id) REFERENCES employees(id) ON DELETE SET NULL,

    -- Если отдел удалён — ошибка (нельзя удалить отдел с сотрудниками)
    FOREIGN KEY (department_id) REFERENCES departments(id) ON DELETE RESTRICT
);

-- Самоссылающийся внешний ключ (иерархия)
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id INTEGER REFERENCES categories(id) ON DELETE CASCADE
);
```

### Отложенные ограничения (Deferrable Constraints)

```sql
-- Отложенная проверка — полезна для циклических ссылок
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    manager_id INTEGER,

    CONSTRAINT fk_manager FOREIGN KEY (manager_id)
        REFERENCES employees(id)
        DEFERRABLE INITIALLY DEFERRED
);

-- Теперь можно вставить взаимоссылающиеся записи в одной транзакции
BEGIN;
INSERT INTO employees (id, name, manager_id) VALUES (1, 'Alice', 2);
INSERT INTO employees (id, name, manager_id) VALUES (2, 'Bob', 1);
COMMIT;  -- проверка выполняется здесь

-- Управление в транзакции
SET CONSTRAINTS fk_manager DEFERRED;   -- отложить проверку
SET CONSTRAINTS fk_manager IMMEDIATE;  -- проверить сейчас
SET CONSTRAINTS ALL DEFERRED;          -- все ограничения
```

## CHECK

Проверяет произвольное условие для столбца или строки:

```sql
-- На уровне столбца
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price NUMERIC(10, 2) CHECK (price > 0),
    discount NUMERIC(5, 2) CHECK (discount >= 0 AND discount <= 100)
);

-- На уровне таблицы (может ссылаться на несколько столбцов)
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    order_date DATE NOT NULL,
    ship_date DATE,
    CONSTRAINT valid_dates CHECK (ship_date IS NULL OR ship_date >= order_date)
);

-- Именованные ограничения
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    salary NUMERIC(10, 2),
    hire_date DATE NOT NULL,
    termination_date DATE,

    CONSTRAINT valid_email CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'),
    CONSTRAINT positive_salary CHECK (salary > 0),
    CONSTRAINT valid_employment_dates CHECK (termination_date IS NULL OR termination_date > hire_date)
);

-- Добавление к существующей таблице
ALTER TABLE products ADD CONSTRAINT products_positive_price CHECK (price > 0);

-- CHECK с использованием функций
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    event_date DATE NOT NULL,
    CONSTRAINT future_date CHECK (event_date > CURRENT_DATE)
);

-- NOT VALID — не проверять существующие данные
ALTER TABLE old_table ADD CONSTRAINT positive_value CHECK (value > 0) NOT VALID;
-- Позже, после исправления данных:
ALTER TABLE old_table VALIDATE CONSTRAINT positive_value;
```

## EXCLUSION

Исключает конфликтующие строки на основе операторов сравнения:

```sql
-- Требует расширения btree_gist для некоторых типов
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- Предотвращение пересечения бронирований
CREATE TABLE room_reservations (
    id SERIAL PRIMARY KEY,
    room_id INTEGER NOT NULL,
    during TSTZRANGE NOT NULL,

    -- Нельзя забронировать одну комнату в пересекающееся время
    EXCLUDE USING GIST (room_id WITH =, during WITH &&)
);

-- Попытка забронировать пересекающееся время вызовет ошибку
INSERT INTO room_reservations (room_id, during) VALUES
    (1, '[2024-03-15 10:00, 2024-03-15 12:00)');  -- OK

INSERT INTO room_reservations (room_id, during) VALUES
    (1, '[2024-03-15 11:00, 2024-03-15 13:00)');  -- ERROR: conflicting key value

-- Исключение для IP-диапазонов
CREATE TABLE ip_allocations (
    id SERIAL PRIMARY KEY,
    network INET NOT NULL,
    owner VARCHAR(100),
    EXCLUDE USING GIST (network inet_ops WITH &&)
);
```

## Управление ограничениями

### Просмотр ограничений

```sql
-- Через information_schema
SELECT
    constraint_name,
    constraint_type,
    table_name
FROM information_schema.table_constraints
WHERE table_name = 'employees';

-- Более детально через pg_catalog
SELECT
    c.conname AS constraint_name,
    c.contype AS type,
    pg_get_constraintdef(c.oid) AS definition
FROM pg_constraint c
JOIN pg_class t ON c.conrelid = t.oid
WHERE t.relname = 'employees';

-- contype: 'p' = primary key, 'f' = foreign key, 'u' = unique, 'c' = check
```

### Добавление и удаление

```sql
-- Добавление
ALTER TABLE products ADD CONSTRAINT products_positive_price CHECK (price > 0);

-- Удаление
ALTER TABLE products DROP CONSTRAINT products_positive_price;

-- Удаление с каскадом (удаляет зависимые объекты)
ALTER TABLE products DROP CONSTRAINT products_pkey CASCADE;

-- Переименование
ALTER TABLE products RENAME CONSTRAINT old_name TO new_name;

-- Временное отключение (только для FK и триггеров)
ALTER TABLE orders DISABLE TRIGGER ALL;  -- отключает триггеры ограничений
ALTER TABLE orders ENABLE TRIGGER ALL;   -- включает обратно

-- Для CHECK/FK можно использовать NOT VALID
-- Но полное отключение CHECK невозможно без удаления
```

## Best Practices

### 1. Всегда используйте NOT NULL где возможно

```sql
-- Плохо: слишком много опциональных полей
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255),
    created_at TIMESTAMP
);

-- Хорошо: явно указаны обязательные поля
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(20),  -- явно опциональное
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

### 2. Именуйте ограничения

```sql
-- Плохо: автоматические имена
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    total NUMERIC CHECK (total > 0)
);
-- Имена: orders_pkey, orders_customer_id_fkey, orders_total_check

-- Хорошо: явные имена
CREATE TABLE orders (
    id SERIAL,
    customer_id INTEGER NOT NULL,
    total NUMERIC NOT NULL,

    CONSTRAINT orders_pk PRIMARY KEY (id),
    CONSTRAINT orders_customer_fk FOREIGN KEY (customer_id) REFERENCES customers(id),
    CONSTRAINT orders_total_positive CHECK (total > 0)
);
```

### 3. Выбирайте правильные действия для FK

```sql
-- Для обязательных связей: RESTRICT или NO ACTION
FOREIGN KEY (department_id) REFERENCES departments(id) ON DELETE RESTRICT

-- Для связей "часть целого": CASCADE
FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE

-- Для опциональных связей: SET NULL
FOREIGN KEY (assigned_to) REFERENCES users(id) ON DELETE SET NULL
```

### 4. Проверяйте данные на уровне БД

```sql
-- Бизнес-правила в CHECK
CREATE TABLE discounts (
    id SERIAL PRIMARY KEY,
    percentage NUMERIC(5, 2) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,

    CONSTRAINT valid_percentage CHECK (percentage BETWEEN 0 AND 100),
    CONSTRAINT valid_period CHECK (end_date > start_date)
);
```

### 5. Используйте EXCLUSION для сложных уникальностей

```sql
-- Когда UNIQUE недостаточно
CREATE TABLE employee_positions (
    id SERIAL PRIMARY KEY,
    employee_id INTEGER NOT NULL,
    position_id INTEGER NOT NULL,
    during DATERANGE NOT NULL,

    -- Сотрудник не может занимать две должности одновременно
    EXCLUDE USING GIST (employee_id WITH =, during WITH &&)
);
```

## Производительность

Ограничения влияют на производительность:

```sql
-- UNIQUE и PRIMARY KEY создают индексы автоматически
-- Их не нужно создавать отдельно

-- FOREIGN KEY НЕ создаёт индекс автоматически
-- Создайте индекс вручную для ускорения JOIN и проверок
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(id),
    product_id INTEGER REFERENCES products(id)
);

-- Важно: создайте индексы для FK
CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_order_items_product ON order_items(product_id);

-- CHECK выполняется при каждой вставке/обновлении
-- Избегайте сложных проверок или используйте триггеры
```

Ограничения — это фундамент целостности данных. Используйте их максимально, чтобы база данных сама защищала себя от некорректных данных.
