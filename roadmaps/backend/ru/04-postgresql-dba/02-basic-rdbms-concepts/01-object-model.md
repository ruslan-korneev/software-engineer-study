# Object Model в PostgreSQL

[prev: 04-postgresql-vs-nosql](../01-introduction/04-postgresql-vs-nosql.md) | [next: 02-relational-model](./02-relational-model.md)
---

## Что такое объектная модель?

**Объектная модель** (Object Model) — это способ организации и представления данных в СУБД, где все сущности рассматриваются как объекты. PostgreSQL называют **объектно-реляционной СУБД (ORDBMS)**, потому что она сочетает классическую реляционную модель с объектно-ориентированными возможностями.

## Ключевые объекты в PostgreSQL

PostgreSQL организует данные в иерархию объектов:

```
Cluster (Кластер)
└── Database (База данных)
    └── Schema (Схема)
        ├── Table (Таблица)
        ├── View (Представление)
        ├── Index (Индекс)
        ├── Sequence (Последовательность)
        ├── Function (Функция)
        ├── Type (Тип данных)
        └── ...другие объекты
```

### Основные типы объектов

| Объект | Описание |
|--------|----------|
| **Database** | Изолированный контейнер для схем и данных |
| **Schema** | Пространство имён внутри базы данных |
| **Table** | Основное хранилище данных (строки и столбцы) |
| **View** | Виртуальная таблица на основе запроса |
| **Materialized View** | Кешированный результат запроса |
| **Index** | Структура для ускорения поиска |
| **Sequence** | Генератор уникальных числовых значений |
| **Function** | Хранимая процедура/функция |
| **Trigger** | Автоматически выполняемый код при событиях |
| **Type** | Пользовательский тип данных |
| **Domain** | Тип с ограничениями |
| **Extension** | Расширение функциональности PostgreSQL |

## Объектно-ориентированные возможности PostgreSQL

### 1. Наследование таблиц (Table Inheritance)

PostgreSQL поддерживает наследование таблиц — дочерняя таблица получает все столбцы родительской:

```sql
-- Родительская таблица
CREATE TABLE vehicles (
    id SERIAL PRIMARY KEY,
    brand VARCHAR(50) NOT NULL,
    model VARCHAR(50) NOT NULL,
    year INTEGER
);

-- Дочерняя таблица наследует столбцы vehicles
CREATE TABLE cars (
    num_doors INTEGER,
    fuel_type VARCHAR(20)
) INHERITS (vehicles);

-- Дочерняя таблица для мотоциклов
CREATE TABLE motorcycles (
    engine_cc INTEGER
) INHERITS (vehicles);

-- Вставка данных
INSERT INTO cars (brand, model, year, num_doors, fuel_type)
VALUES ('Toyota', 'Camry', 2023, 4, 'hybrid');

INSERT INTO motorcycles (brand, model, year, engine_cc)
VALUES ('Honda', 'CBR600', 2022, 600);

-- Запрос к родительской таблице возвращает все записи (включая дочерние)
SELECT * FROM vehicles;

-- Запрос ONLY к родительской таблице (без дочерних)
SELECT * FROM ONLY vehicles;
```

### 2. Пользовательские типы данных

```sql
-- Составной тип (composite type)
CREATE TYPE address AS (
    street VARCHAR(100),
    city VARCHAR(50),
    postal_code VARCHAR(10),
    country VARCHAR(50)
);

-- Использование составного типа
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    home_address address,
    work_address address
);

-- Вставка данных
INSERT INTO customers (name, home_address, work_address) VALUES (
    'Иван Петров',
    ROW('ул. Ленина, 15', 'Москва', '101000', 'Россия'),
    ROW('ул. Гагарина, 42', 'Москва', '105000', 'Россия')
);

-- Доступ к полям составного типа
SELECT name, (home_address).city, (work_address).street
FROM customers;
```

### 3. Перечисления (ENUM)

```sql
-- Создание перечисляемого типа
CREATE TYPE order_status AS ENUM (
    'pending',
    'confirmed',
    'shipped',
    'delivered',
    'cancelled'
);

-- Использование ENUM
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    status order_status DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW()
);

-- Порядок значений ENUM сохраняется
SELECT * FROM orders ORDER BY status;  -- сортировка по порядку в ENUM
```

### 4. Массивы (Arrays)

```sql
-- Столбец с массивом
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    tags TEXT[],
    prices NUMERIC[]
);

-- Вставка массивов
INSERT INTO products (name, tags, prices) VALUES
    ('Ноутбук', ARRAY['электроника', 'компьютеры', 'офис'], ARRAY[999.99, 899.99, 849.99]),
    ('Телефон', '{"электроника", "связь"}', '{599.99, 549.99}');

-- Работа с массивами
SELECT name, tags[1] AS first_tag FROM products;
SELECT * FROM products WHERE 'электроника' = ANY(tags);
SELECT * FROM products WHERE tags @> ARRAY['офис'];
```

## Системный каталог (System Catalog)

PostgreSQL хранит метаданные обо всех объектах в системных таблицах:

```sql
-- Список всех таблиц в текущей схеме
SELECT tablename FROM pg_tables WHERE schemaname = 'public';

-- Информация о столбцах таблицы
SELECT column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_name = 'customers';

-- Все объекты в базе данных
SELECT relname, relkind
FROM pg_class
WHERE relnamespace = (SELECT oid FROM pg_namespace WHERE nspname = 'public');
-- relkind: 'r' = table, 'i' = index, 'S' = sequence, 'v' = view, 'm' = materialized view
```

## OID — Object Identifier

Каждый объект в PostgreSQL имеет уникальный идентификатор — **OID**:

```sql
-- Получить OID таблицы
SELECT 'customers'::regclass::oid;

-- Или через системный каталог
SELECT oid, relname FROM pg_class WHERE relname = 'customers';

-- OID используется внутренне для связей между объектами
SELECT pg_relation_filepath('customers');  -- путь к файлу таблицы
```

## Best Practices

1. **Используйте наследование осторожно** — оно имеет ограничения (например, UNIQUE и FOREIGN KEY не наследуются автоматически). Рассмотрите партиционирование как альтернативу.

2. **Предпочитайте стандартные типы** — пользовательские типы удобны, но усложняют миграции и ORM-интеграцию.

3. **Группируйте объекты по схемам** — это упрощает управление правами и организацию кода.

4. **Документируйте объекты** с помощью `COMMENT`:
   ```sql
   COMMENT ON TABLE customers IS 'Таблица клиентов интернет-магазина';
   COMMENT ON COLUMN customers.id IS 'Уникальный идентификатор клиента';
   ```

5. **Используйте расширения** для дополнительной функциональности:
   ```sql
   CREATE EXTENSION IF NOT EXISTS "uuid-ossp";  -- генерация UUID
   CREATE EXTENSION IF NOT EXISTS "hstore";      -- хранение key-value
   ```

## Сравнение: реляционная vs объектно-реляционная модель

| Аспект | Реляционная СУБД | Объектно-реляционная (PostgreSQL) |
|--------|------------------|-----------------------------------|
| Типы данных | Только встроенные | + пользовательские типы |
| Наследование | Нет | Да (таблицы) |
| Массивы | Нет (нормализация) | Да (нативная поддержка) |
| JSON/JSONB | Как текст | Нативная поддержка с индексами |
| Функции | Процедуры | + перегрузка, полиморфизм |
| Расширяемость | Ограничена | Высокая (extensions) |

PostgreSQL предоставляет мощную объектную модель, сохраняя при этом полную совместимость с реляционными стандартами SQL.

---
[prev: 04-postgresql-vs-nosql](../01-introduction/04-postgresql-vs-nosql.md) | [next: 02-relational-model](./02-relational-model.md)