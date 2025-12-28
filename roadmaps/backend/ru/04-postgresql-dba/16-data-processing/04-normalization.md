# Нормализация базы данных (Database Normalization)

[prev: 03-sharding-patterns](./03-sharding-patterns.md) | [next: 05-queues](./05-queues.md)

---

## Введение

Нормализация - это процесс организации данных в базе данных для минимизации избыточности и зависимостей. Цель нормализации - разделить данные на логические таблицы и установить связи между ними, обеспечивая целостность данных.

## Зачем нужна нормализация

### Проблемы ненормализованных данных

```sql
-- Пример ненормализованной таблицы
CREATE TABLE orders_denormalized (
    order_id INTEGER,
    order_date DATE,
    customer_name VARCHAR(100),
    customer_email VARCHAR(255),
    customer_phone VARCHAR(20),
    customer_address TEXT,
    product_name VARCHAR(100),
    product_price DECIMAL(10,2),
    product_category VARCHAR(50),
    quantity INTEGER
);

-- Проблемы:
-- 1. Избыточность: данные клиента повторяются в каждом заказе
-- 2. Аномалия обновления: изменение email требует обновления многих строк
-- 3. Аномалия удаления: удаление заказа может удалить данные о клиенте
-- 4. Аномалия вставки: нельзя добавить клиента без заказа
```

### Преимущества нормализации

1. **Устранение избыточности** - данные хранятся в одном месте
2. **Целостность данных** - легче поддерживать согласованность
3. **Гибкость** - проще изменять структуру
4. **Экономия места** - меньше дублирования данных

## Нормальные формы

### Первая нормальная форма (1NF)

**Правила:**
- Каждая ячейка содержит только одно атомарное значение
- Нет повторяющихся групп столбцов
- Каждая строка уникальна (есть первичный ключ)

```sql
-- НЕ в 1NF: несколько значений в одной ячейке
CREATE TABLE orders_not_1nf (
    order_id INTEGER PRIMARY KEY,
    customer_name VARCHAR(100),
    products VARCHAR(500)  -- "Laptop, Mouse, Keyboard"
);

-- В 1NF: каждое значение атомарно
CREATE TABLE orders_1nf (
    order_id INTEGER,
    customer_name VARCHAR(100),
    product VARCHAR(100),
    PRIMARY KEY (order_id, product)
);

-- Или лучше с отдельной таблицей
CREATE TABLE orders (
    order_id INTEGER PRIMARY KEY,
    customer_name VARCHAR(100)
);

CREATE TABLE order_items (
    order_id INTEGER REFERENCES orders(order_id),
    product VARCHAR(100),
    PRIMARY KEY (order_id, product)
);
```

### Вторая нормальная форма (2NF)

**Правила:**
- Таблица в 1NF
- Все неключевые атрибуты полностью зависят от первичного ключа (нет частичной зависимости)

```sql
-- НЕ в 2NF: product_category зависит только от product, не от всего ключа
CREATE TABLE order_items_not_2nf (
    order_id INTEGER,
    product_id INTEGER,
    product_name VARCHAR(100),     -- Зависит только от product_id
    product_category VARCHAR(50),  -- Зависит только от product_id
    quantity INTEGER,              -- Зависит от полного ключа
    PRIMARY KEY (order_id, product_id)
);

-- В 2NF: разделение на две таблицы
CREATE TABLE products (
    product_id INTEGER PRIMARY KEY,
    product_name VARCHAR(100),
    product_category VARCHAR(50)
);

CREATE TABLE order_items_2nf (
    order_id INTEGER,
    product_id INTEGER REFERENCES products(product_id),
    quantity INTEGER,
    PRIMARY KEY (order_id, product_id)
);
```

### Третья нормальная форма (3NF)

**Правила:**
- Таблица в 2NF
- Нет транзитивных зависимостей (неключевой атрибут не зависит от другого неключевого атрибута)

```sql
-- НЕ в 3NF: region зависит от country, а не от первичного ключа
CREATE TABLE customers_not_3nf (
    customer_id INTEGER PRIMARY KEY,
    customer_name VARCHAR(100),
    country VARCHAR(50),
    region VARCHAR(50)  -- region зависит от country (транзитивная зависимость)
);

-- В 3NF: выносим зависимость в отдельную таблицу
CREATE TABLE countries (
    country VARCHAR(50) PRIMARY KEY,
    region VARCHAR(50)
);

CREATE TABLE customers_3nf (
    customer_id INTEGER PRIMARY KEY,
    customer_name VARCHAR(100),
    country VARCHAR(50) REFERENCES countries(country)
);
```

### Нормальная форма Бойса-Кодда (BCNF)

**Правила:**
- Таблица в 3NF
- Каждый детерминант является потенциальным ключом

```sql
-- Пример проблемы BCNF
-- Предположим: каждый преподаватель ведет только один предмет
CREATE TABLE enrollments_not_bcnf (
    student_id INTEGER,
    subject VARCHAR(50),
    teacher VARCHAR(100),
    PRIMARY KEY (student_id, subject)
    -- teacher -> subject (детерминант не является ключом)
);

-- В BCNF
CREATE TABLE teachers (
    teacher VARCHAR(100) PRIMARY KEY,
    subject VARCHAR(50)
);

CREATE TABLE enrollments_bcnf (
    student_id INTEGER,
    teacher VARCHAR(100) REFERENCES teachers(teacher),
    PRIMARY KEY (student_id, teacher)
);
```

### Четвертая нормальная форма (4NF)

**Правила:**
- Таблица в BCNF
- Нет многозначных зависимостей

```sql
-- НЕ в 4NF: многозначные зависимости
-- Сотрудник может знать несколько языков И работать над несколькими проектами
-- Эти факты независимы друг от друга
CREATE TABLE employee_skills_not_4nf (
    employee_id INTEGER,
    language VARCHAR(50),
    project VARCHAR(100),
    PRIMARY KEY (employee_id, language, project)
    -- employee_id ->> language (многозначная зависимость)
    -- employee_id ->> project (многозначная зависимость)
);

-- В 4NF: разделение на независимые таблицы
CREATE TABLE employee_languages (
    employee_id INTEGER,
    language VARCHAR(50),
    PRIMARY KEY (employee_id, language)
);

CREATE TABLE employee_projects (
    employee_id INTEGER,
    project VARCHAR(100),
    PRIMARY KEY (employee_id, project)
);
```

### Пятая нормальная форма (5NF)

**Правила:**
- Таблица в 4NF
- Нет зависимостей соединения, которые не следуют из ключей

```sql
-- 5NF требуется редко, обычно в сложных случаях с тройными связями
-- Пример: поставщик поставляет продукт в проект
-- Но связь не разбивается на пары

-- НЕ в 5NF
CREATE TABLE supplier_product_project (
    supplier_id INTEGER,
    product_id INTEGER,
    project_id INTEGER,
    PRIMARY KEY (supplier_id, product_id, project_id)
);

-- Если связь действительно тройная и не разбивается,
-- таблица уже в 5NF
```

## Практический пример нормализации

### Исходная ненормализованная таблица

```sql
-- Исходная "плоская" таблица
CREATE TABLE shop_data (
    order_id INTEGER,
    order_date DATE,
    customer_name VARCHAR(100),
    customer_email VARCHAR(255),
    customer_city VARCHAR(100),
    customer_country VARCHAR(100),
    country_region VARCHAR(50),
    products TEXT,  -- "Laptop:1200:2, Mouse:25:3"
    total_amount DECIMAL(10,2)
);
```

### Шаг 1: Приведение к 1NF

```sql
-- Атомарные значения
CREATE TABLE orders_step1 (
    order_id INTEGER,
    order_date DATE,
    customer_name VARCHAR(100),
    customer_email VARCHAR(255),
    customer_city VARCHAR(100),
    customer_country VARCHAR(100),
    country_region VARCHAR(50),
    product_name VARCHAR(100),
    product_price DECIMAL(10,2),
    quantity INTEGER,
    PRIMARY KEY (order_id, product_name)
);
```

### Шаг 2: Приведение к 2NF

```sql
-- Устранение частичных зависимостей
CREATE TABLE orders_step2 (
    order_id INTEGER PRIMARY KEY,
    order_date DATE,
    customer_name VARCHAR(100),
    customer_email VARCHAR(255),
    customer_city VARCHAR(100),
    customer_country VARCHAR(100),
    country_region VARCHAR(50)
);

CREATE TABLE products_step2 (
    product_name VARCHAR(100) PRIMARY KEY,
    product_price DECIMAL(10,2)
);

CREATE TABLE order_items_step2 (
    order_id INTEGER REFERENCES orders_step2(order_id),
    product_name VARCHAR(100) REFERENCES products_step2(product_name),
    quantity INTEGER,
    PRIMARY KEY (order_id, product_name)
);
```

### Шаг 3: Приведение к 3NF

```sql
-- Устранение транзитивных зависимостей

-- Страны и регионы
CREATE TABLE countries (
    country VARCHAR(100) PRIMARY KEY,
    region VARCHAR(50)
);

-- Клиенты
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255) UNIQUE,
    city VARCHAR(100),
    country VARCHAR(100) REFERENCES countries(country)
);

-- Продукты (с добавлением категорий)
CREATE TABLE categories (
    category_id SERIAL PRIMARY KEY,
    category_name VARCHAR(50)
);

CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2),
    category_id INTEGER REFERENCES categories(category_id)
);

-- Заказы
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    order_date DATE DEFAULT CURRENT_DATE,
    customer_id INTEGER REFERENCES customers(customer_id)
);

-- Позиции заказа
CREATE TABLE order_items (
    order_id INTEGER REFERENCES orders(order_id),
    product_id INTEGER REFERENCES products(product_id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    price_at_order DECIMAL(10,2),  -- Цена на момент заказа
    PRIMARY KEY (order_id, product_id)
);
```

## Денормализация

Иногда нормализация снижает производительность. В таких случаях применяют контролируемую денормализацию.

### Когда денормализовать

1. **Частые JOIN-операции** - объединение данных для чтения
2. **Отчеты и аналитика** - агрегированные данные
3. **Кеширование** - предвычисленные значения
4. **Read-heavy нагрузка** - много чтений, мало записей

### Примеры денормализации

#### 1. Хранение вычисляемых значений

```sql
-- Нормализованная версия требует подзапрос
SELECT
    o.order_id,
    (SELECT SUM(oi.quantity * oi.price_at_order)
     FROM order_items oi WHERE oi.order_id = o.order_id) as total
FROM orders o;

-- Денормализованная версия с хранением total
ALTER TABLE orders ADD COLUMN total_amount DECIMAL(10,2);

-- Триггер для поддержания актуальности
CREATE OR REPLACE FUNCTION update_order_total()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE orders
    SET total_amount = (
        SELECT COALESCE(SUM(quantity * price_at_order), 0)
        FROM order_items
        WHERE order_id = COALESCE(NEW.order_id, OLD.order_id)
    )
    WHERE order_id = COALESCE(NEW.order_id, OLD.order_id);
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_order_total
AFTER INSERT OR UPDATE OR DELETE ON order_items
FOR EACH ROW EXECUTE FUNCTION update_order_total();
```

#### 2. Хранение имени в связанной таблице

```sql
-- Денормализация: хранение customer_name в orders
ALTER TABLE orders ADD COLUMN customer_name VARCHAR(100);

-- Теперь не нужен JOIN для отображения заказов с именем клиента
SELECT order_id, order_date, customer_name, total_amount
FROM orders
WHERE order_date > '2024-01-01';
```

#### 3. Materialized Views для отчетов

```sql
-- Материализованное представление для dashboard
CREATE MATERIALIZED VIEW sales_summary AS
SELECT
    DATE_TRUNC('month', o.order_date) as month,
    c.country,
    cat.category_name,
    COUNT(DISTINCT o.order_id) as order_count,
    SUM(oi.quantity) as items_sold,
    SUM(oi.quantity * oi.price_at_order) as revenue
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
JOIN categories cat ON p.category_id = cat.category_id
GROUP BY 1, 2, 3;

-- Индекс для быстрого доступа
CREATE INDEX idx_sales_summary_month ON sales_summary(month);

-- Обновление данных
REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary;
```

#### 4. JSON для гибких данных

```sql
-- Денормализация через JSON
ALTER TABLE products ADD COLUMN attributes JSONB;

UPDATE products SET attributes = '{
    "color": "black",
    "weight": "1.5kg",
    "dimensions": {"width": 30, "height": 20, "depth": 5}
}'::jsonb WHERE product_id = 1;

-- Индекс для быстрого поиска по JSON
CREATE INDEX idx_products_attributes ON products USING GIN (attributes);

-- Поиск по JSON атрибутам
SELECT * FROM products
WHERE attributes @> '{"color": "black"}';
```

## Анти-паттерны нормализации

### 1. Чрезмерная нормализация

```sql
-- Слишком много: отдельная таблица для каждого атрибута
CREATE TABLE customer_names (
    customer_id INTEGER PRIMARY KEY,
    value VARCHAR(100)
);

CREATE TABLE customer_emails (
    customer_id INTEGER PRIMARY KEY,
    value VARCHAR(255)
);

-- Лучше: один осмысленный объект в одной таблице
CREATE TABLE customers (
    customer_id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255)
);
```

### 2. EAV (Entity-Attribute-Value) злоупотребление

```sql
-- Анти-паттерн: EAV для всего
CREATE TABLE entity_attributes (
    entity_id INTEGER,
    attribute_name VARCHAR(50),
    attribute_value TEXT,
    PRIMARY KEY (entity_id, attribute_name)
);

-- Проблемы:
-- - Нет типизации
-- - Сложные запросы
-- - Нет ограничений целостности

-- Лучше: конкретные таблицы с типизированными столбцами
-- EAV оправдан только для действительно динамических атрибутов
```

### 3. One True Lookup Table (OTLT)

```sql
-- Анти-паттерн: одна таблица для всех справочников
CREATE TABLE lookup (
    type VARCHAR(50),
    code VARCHAR(20),
    value VARCHAR(100),
    PRIMARY KEY (type, code)
);

INSERT INTO lookup VALUES
    ('COUNTRY', 'US', 'United States'),
    ('COUNTRY', 'UK', 'United Kingdom'),
    ('STATUS', 'A', 'Active'),
    ('STATUS', 'I', 'Inactive');

-- Проблемы:
-- - Нет FK constraints
-- - Сложные JOIN
-- - Нет типизации

-- Лучше: отдельные таблицы для каждого справочника
CREATE TABLE countries (code CHAR(2) PRIMARY KEY, name VARCHAR(100));
CREATE TABLE statuses (code CHAR(1) PRIMARY KEY, name VARCHAR(50));
```

## Best Practices

### 1. Начинайте с нормализации

```sql
-- Всегда начинайте с нормализованной схемы
-- Денормализуйте только при доказанной необходимости

-- Шаги:
-- 1. Проектирование в 3NF
-- 2. Тестирование производительности
-- 3. Выявление bottlenecks
-- 4. Точечная денормализация с документированием
```

### 2. Используйте суррогатные ключи

```sql
-- Суррогатный ключ vs естественный
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,  -- Суррогатный ключ
    email VARCHAR(255) UNIQUE,       -- Естественный ключ как альтернатива
    name VARCHAR(100)
);

-- Преимущества суррогатных ключей:
-- - Не меняются
-- - Компактные (INTEGER vs VARCHAR)
-- - Проще для JOIN
```

### 3. Документируйте денормализацию

```sql
-- Комментарии для денормализованных полей
COMMENT ON COLUMN orders.customer_name IS
    'Денормализовано из customers.name для ускорения отчетов.
     Обновляется триггером trg_sync_customer_name.';

COMMENT ON COLUMN orders.total_amount IS
    'Денормализованная сумма заказа.
     Обновляется триггером trg_update_order_total.';
```

## Инструменты анализа

```sql
-- Поиск избыточности (повторяющиеся данные)
SELECT customer_name, customer_email, COUNT(*) as occurrences
FROM orders_denormalized
GROUP BY customer_name, customer_email
HAVING COUNT(*) > 1
ORDER BY occurrences DESC;

-- Анализ зависимостей (для 3NF)
-- Проверка: зависит ли region только от country?
SELECT country, COUNT(DISTINCT region) as region_count
FROM customers_not_3nf
GROUP BY country
HAVING COUNT(DISTINCT region) > 1;  -- Если > 1, возможно нарушение 3NF
```

## Типичные ошибки

1. **Недостаточная нормализация** - избыточность и аномалии данных
2. **Чрезмерная нормализация** - слишком много JOIN, низкая производительность
3. **Денормализация без триггеров** - данные рассинхронизируются
4. **Игнорирование бизнес-требований** - нормализация ради нормализации
5. **Отсутствие индексов на FK** - медленные JOIN операции

---

[prev: 03-sharding-patterns](./03-sharding-patterns.md) | [next: 05-queues](./05-queues.md)
