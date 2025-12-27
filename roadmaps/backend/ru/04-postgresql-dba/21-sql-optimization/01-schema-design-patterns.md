# Паттерны проектирования схемы БД

## Введение

Правильное проектирование схемы базы данных — фундамент производительности приложения. Выбор структуры таблиц, связей и типов данных напрямую влияет на скорость запросов, объём хранимых данных и удобство разработки.

---

## 1. Нормализация

### Что такое нормализация

Нормализация — процесс организации данных для минимизации избыточности и обеспечения целостности данных.

### Нормальные формы

**1NF (Первая нормальная форма):**
- Атомарные значения (без массивов в ячейках)
- Уникальный идентификатор для каждой строки

```sql
-- Плохо (нарушает 1NF)
CREATE TABLE orders_bad (
    id SERIAL PRIMARY KEY,
    customer_name TEXT,
    products TEXT  -- "Молоко, Хлеб, Масло"
);

-- Хорошо (1NF)
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_name TEXT
);

CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(id),
    product_name TEXT
);
```

**2NF (Вторая нормальная форма):**
- Соответствует 1NF
- Все неключевые атрибуты полностью зависят от первичного ключа

```sql
-- Плохо (нарушает 2NF) - customer_city зависит только от customer_id
CREATE TABLE orders_bad (
    order_id INTEGER,
    customer_id INTEGER,
    customer_city TEXT,  -- Зависит только от customer_id
    product_id INTEGER,
    PRIMARY KEY (order_id, customer_id)
);

-- Хорошо (2NF)
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    city TEXT
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    product_id INTEGER
);
```

**3NF (Третья нормальная форма):**
- Соответствует 2NF
- Нет транзитивных зависимостей (неключевые атрибуты не зависят друг от друга)

```sql
-- Плохо (нарушает 3NF) - region зависит от city
CREATE TABLE customers_bad (
    id SERIAL PRIMARY KEY,
    city TEXT,
    region TEXT  -- Moscow -> Central Region (транзитивная зависимость)
);

-- Хорошо (3NF)
CREATE TABLE cities (
    id SERIAL PRIMARY KEY,
    name TEXT,
    region TEXT
);

CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    city_id INTEGER REFERENCES cities(id)
);
```

### Преимущества нормализации
- Минимум дублирования данных
- Легче поддерживать целостность
- Проще обновление данных (меньше мест для изменений)

### Недостатки нормализации
- Много JOIN-ов при запросах
- Сложнее читать структуру
- Возможно снижение производительности на чтение

---

## 2. Денормализация

### Когда применять

Денормализация — намеренное добавление избыточности для ускорения чтения.

```sql
-- Нормализованная структура (много JOIN)
SELECT
    o.id,
    c.name,
    c.email,
    p.name AS product_name,
    p.price
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON oi.product_id = p.id;

-- Денормализованная структура (быстрее на чтение)
CREATE TABLE orders_denormalized (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    customer_name TEXT,      -- Дублирование
    customer_email TEXT,     -- Дублирование
    total_amount NUMERIC,    -- Предвычисленное значение
    items_count INTEGER,     -- Предвычисленное значение
    created_at TIMESTAMPTZ
);
```

### Техники денормализации

**1. Дублирование полей:**
```sql
-- Храним имя автора прямо в посте
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    author_id INTEGER REFERENCES users(id),
    author_name TEXT,  -- Дублирование для быстрого отображения
    title TEXT,
    content TEXT
);
```

**2. Предвычисленные агрегаты:**
```sql
-- Счётчики в родительской таблице
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name TEXT,
    products_count INTEGER DEFAULT 0  -- Обновляется триггером
);

-- Триггер для поддержки счётчика
CREATE OR REPLACE FUNCTION update_products_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE categories SET products_count = products_count + 1
        WHERE id = NEW.category_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE categories SET products_count = products_count - 1
        WHERE id = OLD.category_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER products_count_trigger
AFTER INSERT OR DELETE ON products
FOR EACH ROW EXECUTE FUNCTION update_products_count();
```

**3. Материализованные представления:**
```sql
CREATE MATERIALIZED VIEW monthly_sales AS
SELECT
    date_trunc('month', created_at) AS month,
    category_id,
    SUM(amount) AS total_sales,
    COUNT(*) AS orders_count
FROM orders
GROUP BY 1, 2;

-- Обновление по расписанию
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_sales;
```

---

## 3. Паттерн EAV (Entity-Attribute-Value)

### Описание

EAV позволяет хранить произвольные атрибуты для сущностей без изменения схемы.

```sql
-- Классическая структура EAV
CREATE TABLE entities (
    id SERIAL PRIMARY KEY,
    entity_type TEXT,
    name TEXT
);

CREATE TABLE attributes (
    id SERIAL PRIMARY KEY,
    name TEXT UNIQUE,
    data_type TEXT  -- 'string', 'integer', 'boolean', etc.
);

CREATE TABLE entity_values (
    id SERIAL PRIMARY KEY,
    entity_id INTEGER REFERENCES entities(id),
    attribute_id INTEGER REFERENCES attributes(id),
    value_string TEXT,
    value_integer INTEGER,
    value_boolean BOOLEAN,
    value_timestamp TIMESTAMPTZ,
    UNIQUE (entity_id, attribute_id)
);
```

### Пример использования

```sql
-- Вставка продукта с динамическими атрибутами
INSERT INTO entities (entity_type, name) VALUES ('product', 'iPhone 15');

INSERT INTO entity_values (entity_id, attribute_id, value_string)
SELECT 1, id, 'Apple' FROM attributes WHERE name = 'brand';

INSERT INTO entity_values (entity_id, attribute_id, value_integer)
SELECT 1, id, 128 FROM attributes WHERE name = 'storage_gb';

-- Запрос продуктов с атрибутами
SELECT
    e.name,
    a.name AS attribute,
    COALESCE(ev.value_string, ev.value_integer::TEXT) AS value
FROM entities e
JOIN entity_values ev ON ev.entity_id = e.id
JOIN attributes a ON a.id = ev.attribute_id
WHERE e.id = 1;
```

### Когда использовать EAV
- Каталоги с очень разными типами товаров
- CMS с пользовательскими полями
- Системы опросов/анкет

### Проблемы EAV
- Сложные запросы с множеством JOIN
- Трудно обеспечить целостность данных
- Плохая производительность на больших объёмах
- Сложнее индексировать

---

## 4. Наследование таблиц в PostgreSQL

### Table Inheritance

PostgreSQL поддерживает наследование таблиц на уровне СУБД:

```sql
-- Родительская таблица
CREATE TABLE vehicles (
    id SERIAL PRIMARY KEY,
    brand TEXT,
    model TEXT,
    year INTEGER
);

-- Дочерние таблицы
CREATE TABLE cars (
    doors INTEGER,
    trunk_volume INTEGER
) INHERITS (vehicles);

CREATE TABLE motorcycles (
    engine_cc INTEGER,
    has_sidecar BOOLEAN
) INHERITS (vehicles);

-- Вставка данных
INSERT INTO cars (brand, model, year, doors, trunk_volume)
VALUES ('Toyota', 'Camry', 2023, 4, 500);

INSERT INTO motorcycles (brand, model, year, engine_cc, has_sidecar)
VALUES ('Honda', 'CBR', 2023, 600, false);

-- Запрос ко всем транспортным средствам
SELECT * FROM vehicles;  -- Покажет и cars, и motorcycles

-- Только машины
SELECT * FROM ONLY vehicles;  -- Только записи из vehicles
SELECT * FROM cars;  -- Только машины
```

### Ограничения наследования
- PRIMARY KEY и UNIQUE не наследуются автоматически
- Foreign key не работают с родительской таблицей
- Индексы не наследуются
- Нет автоматического переключения партиций

### Современная альтернатива: партиционирование

```sql
-- Партиционированная таблица (PostgreSQL 10+)
CREATE TABLE logs (
    id BIGSERIAL,
    created_at TIMESTAMPTZ,
    message TEXT
) PARTITION BY RANGE (created_at);

CREATE TABLE logs_2024_01 PARTITION OF logs
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE logs_2024_02 PARTITION OF logs
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

---

## 5. JSONB vs Реляционные таблицы

### Когда использовать JSONB

```sql
-- JSONB для гибких схем
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    category TEXT,
    attributes JSONB  -- Разные атрибуты для разных категорий
);

-- Вставка
INSERT INTO products (name, category, attributes) VALUES
('iPhone 15', 'phones', '{"storage": 128, "color": "black", "5g": true}'),
('MacBook Pro', 'laptops', '{"ram": 16, "ssd": 512, "display": 14}');

-- Запросы к JSONB
SELECT * FROM products WHERE attributes->>'color' = 'black';
SELECT * FROM products WHERE (attributes->>'storage')::int > 64;
SELECT * FROM products WHERE attributes @> '{"5g": true}';
```

### Индексирование JSONB

```sql
-- GIN индекс для произвольных запросов
CREATE INDEX idx_products_attrs ON products USING GIN (attributes);

-- Индекс на конкретный ключ
CREATE INDEX idx_products_color ON products ((attributes->>'color'));
```

### Сравнение подходов

| Критерий | Реляционные таблицы | JSONB |
|----------|--------------------|----|
| Типизация | Строгая | Гибкая |
| Валидация | На уровне БД | На уровне приложения |
| Индексы | Эффективнее | GIN/выражения |
| Изменение схемы | ALTER TABLE | Без миграций |
| JOIN | Легко | Сложнее |
| Агрегаты | Эффективно | Менее эффективно |

### Гибридный подход

```sql
-- Основные поля реляционные, дополнительные в JSONB
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    status TEXT NOT NULL,
    total_amount NUMERIC NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    -- Гибкие метаданные
    metadata JSONB DEFAULT '{}'
);

-- Важные поля индексируются отдельно
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created ON orders(created_at);
-- JSONB для расширенного поиска
CREATE INDEX idx_orders_meta ON orders USING GIN (metadata);
```

---

## Best Practices

### 1. Выбор первичного ключа

```sql
-- SERIAL/BIGSERIAL для большинства случаев
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    ...
);

-- UUID для распределённых систем
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ...
);

-- Composite key для связующих таблиц
CREATE TABLE user_roles (
    user_id INTEGER REFERENCES users(id),
    role_id INTEGER REFERENCES roles(id),
    PRIMARY KEY (user_id, role_id)
);
```

### 2. Правильные типы данных

```sql
-- Используйте подходящие типы
CREATE TABLE example (
    -- Для денег используйте NUMERIC, не FLOAT
    price NUMERIC(10, 2),

    -- Для IP адресов - INET
    ip_address INET,

    -- Для временных данных с зоной
    created_at TIMESTAMPTZ DEFAULT NOW(),

    -- Для статусов - ENUM или TEXT с CHECK
    status TEXT CHECK (status IN ('pending', 'active', 'closed'))
);
```

### 3. Консистентное именование

```sql
-- snake_case для всех идентификаторов
CREATE TABLE user_profiles (
    id SERIAL PRIMARY KEY,
    user_id INTEGER,
    first_name TEXT,
    last_name TEXT,
    created_at TIMESTAMPTZ
);

-- Суффиксы для внешних ключей: _id
-- Суффиксы для временных меток: _at
-- Суффиксы для boolean: is_, has_, can_
```

---

## Типичные ошибки

1. **Избыточная нормализация** — таблица с одним-двумя полями, которые редко меняются
2. **Отсутствие индексов на внешних ключах** — тормозит JOIN и DELETE CASCADE
3. **Использование EAV там, где подошёл бы JSONB** — JSONB проще и быстрее
4. **TEXT вместо VARCHAR с ограничением** — нет валидации на уровне БД
5. **Хранение дат как строк** — невозможны операции сравнения и диапазонов
6. **Nullable поля без явного смысла NULL** — усложняет логику запросов
