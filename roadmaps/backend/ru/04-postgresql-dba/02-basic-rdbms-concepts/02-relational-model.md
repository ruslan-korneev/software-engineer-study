# Реляционная модель данных

[prev: 01-object-model](./01-object-model.md) | [next: 03-tables-rows-columns](./03-tables-rows-columns.md)
---

## Введение

**Реляционная модель** — это математическая модель организации данных, предложенная Эдгаром Коддом в 1970 году. Она является основой всех современных реляционных СУБД, включая PostgreSQL.

Ключевая идея: данные представляются в виде **отношений** (relations), которые визуально выглядят как таблицы.

## Основные понятия реляционной модели

### Формальная терминология vs Практическая

| Формальный термин | Практический термин | Описание |
|-------------------|---------------------|----------|
| **Relation** | Table | Набор кортежей с одинаковой структурой |
| **Tuple** | Row, Record | Одна запись в отношении |
| **Attribute** | Column, Field | Именованное свойство с типом данных |
| **Domain** | Data Type | Допустимое множество значений атрибута |
| **Cardinality** | Row count | Количество кортежей в отношении |
| **Degree** | Column count | Количество атрибутов в отношении |

### Визуализация

```
Отношение (Relation) = Таблица "employees"
┌─────────────────────────────────────────────────────┐
│  Атрибуты (Attributes) = Столбцы                    │
│  ↓         ↓           ↓              ↓             │
├──────┬──────────┬─────────────┬──────────────┤
│  id  │   name   │  department │    salary    │
├──────┼──────────┼─────────────┼──────────────┤
│  1   │  Иван    │  IT         │  100000      │ ← Кортеж (Tuple) = Строка
│  2   │  Мария   │  HR         │  80000       │ ← Кортеж
│  3   │  Пётр    │  IT         │  120000      │ ← Кортеж
└──────┴──────────┴─────────────┴──────────────┘
        Degree = 4 (столбца)
        Cardinality = 3 (строки)
```

## Свойства отношений

Реляционная модель определяет строгие правила для отношений:

### 1. Атомарность значений (1NF)

Каждая ячейка содержит только **одно неделимое значение**:

```sql
-- НЕПРАВИЛЬНО (нарушает атомарность)
CREATE TABLE employees_bad (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    phones VARCHAR(255)  -- "123-456, 789-012" - несколько значений в одной ячейке
);

-- ПРАВИЛЬНО (отдельная таблица для телефонов)
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE employee_phones (
    id SERIAL PRIMARY KEY,
    employee_id INTEGER REFERENCES employees(id),
    phone VARCHAR(20)
);
```

### 2. Уникальность кортежей

Каждая строка в отношении должна быть **уникальной**. Это достигается через первичный ключ:

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,  -- гарантирует уникальность
    sku VARCHAR(50) UNIQUE, -- дополнительный уникальный идентификатор
    name VARCHAR(100)
);
```

### 3. Отсутствие упорядоченности

Строки в отношении **не имеют порядка** по определению. Порядок задаётся явно при запросе:

```sql
-- Порядок строк не гарантирован
SELECT * FROM employees;

-- Явное указание порядка
SELECT * FROM employees ORDER BY salary DESC;
```

### 4. Уникальность имён атрибутов

Все столбцы в таблице должны иметь **уникальные имена**:

```sql
-- Это вызовет ошибку
CREATE TABLE bad_table (
    name VARCHAR(50),
    name VARCHAR(100)  -- ERROR: column "name" specified more than once
);
```

## Ключи в реляционной модели

### Первичный ключ (Primary Key)

**Первичный ключ** — минимальный набор атрибутов, однозначно идентифицирующий кортеж:

```sql
-- Простой первичный ключ
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL
);

-- Составной первичный ключ
CREATE TABLE order_items (
    order_id INTEGER,
    product_id INTEGER,
    quantity INTEGER,
    PRIMARY KEY (order_id, product_id)
);
```

### Суррогатный vs Естественный ключ

```sql
-- Суррогатный ключ (искусственный, автогенерируемый)
CREATE TABLE countries (
    id SERIAL PRIMARY KEY,  -- суррогатный
    code CHAR(2) UNIQUE,    -- ISO код страны
    name VARCHAR(100)
);

-- Естественный ключ (имеет смысл в предметной области)
CREATE TABLE countries_natural (
    code CHAR(2) PRIMARY KEY,  -- естественный ключ
    name VARCHAR(100)
);
```

**Рекомендация**: В большинстве случаев используйте суррогатные ключи — они стабильны и не зависят от бизнес-логики.

### Внешний ключ (Foreign Key)

**Внешний ключ** обеспечивает ссылочную целостность между отношениями:

```sql
CREATE TABLE departments (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    department_id INTEGER REFERENCES departments(id)  -- внешний ключ
);

-- Попытка вставить несуществующий department_id вызовет ошибку
INSERT INTO employees (name, department_id) VALUES ('Иван', 999);
-- ERROR: insert or update on table "employees" violates foreign key constraint
```

### Альтернативный ключ (Candidate Key)

Любой набор атрибутов, который мог бы быть первичным ключом:

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,           -- выбран как первичный
    email VARCHAR(255) UNIQUE,       -- альтернативный ключ
    passport_number VARCHAR(20) UNIQUE  -- альтернативный ключ
);
```

## Реляционная алгебра

Реляционная модель определяет операции над отношениями:

### Основные операции

```sql
-- 1. ВЫБОРКА (Selection) — σ (sigma)
-- Фильтрация строк по условию
SELECT * FROM employees WHERE department = 'IT';

-- 2. ПРОЕКЦИЯ (Projection) — π (pi)
-- Выбор определённых столбцов
SELECT name, salary FROM employees;

-- 3. ОБЪЕДИНЕНИЕ (Union) — ∪
-- Объединение двух совместимых отношений
SELECT name FROM employees
UNION
SELECT name FROM contractors;

-- 4. РАЗНОСТЬ (Difference) — −
-- Строки из первого отношения, отсутствующие во втором
SELECT name FROM all_users
EXCEPT
SELECT name FROM banned_users;

-- 5. ПЕРЕСЕЧЕНИЕ (Intersection) — ∩
-- Общие строки двух отношений
SELECT name FROM employees
INTERSECT
SELECT name FROM managers;

-- 6. ДЕКАРТОВО ПРОИЗВЕДЕНИЕ (Cartesian Product) — ×
-- Все комбинации строк
SELECT * FROM colors, sizes;

-- 7. СОЕДИНЕНИЕ (Join) — ⋈
-- Объединение связанных данных
SELECT e.name, d.name AS department
FROM employees e
JOIN departments d ON e.department_id = d.id;
```

### Типы соединений (JOIN)

```sql
-- Подготовка данных
CREATE TABLE authors (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE books (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200),
    author_id INTEGER REFERENCES authors(id)
);

INSERT INTO authors (name) VALUES ('Толстой'), ('Достоевский'), ('Чехов');
INSERT INTO books (title, author_id) VALUES
    ('Война и мир', 1),
    ('Преступление и наказание', 2),
    ('Неизвестный автор', NULL);

-- INNER JOIN — только совпадающие записи
SELECT b.title, a.name
FROM books b
INNER JOIN authors a ON b.author_id = a.id;
-- Результат: 2 строки (без "Неизвестный автор")

-- LEFT JOIN — все из левой + совпадения из правой
SELECT b.title, a.name
FROM books b
LEFT JOIN authors a ON b.author_id = a.id;
-- Результат: 3 строки (включая NULL для "Неизвестный автор")

-- RIGHT JOIN — все из правой + совпадения из левой
SELECT b.title, a.name
FROM books b
RIGHT JOIN authors a ON b.author_id = a.id;
-- Результат: 3 строки (включая NULL для Чехова без книг)

-- FULL OUTER JOIN — все записи с обеих сторон
SELECT b.title, a.name
FROM books b
FULL OUTER JOIN authors a ON b.author_id = a.id;
-- Результат: 4 строки
```

## Нормализация

Нормализация — процесс организации данных для минимизации избыточности:

### Первая нормальная форма (1NF)
- Все значения атомарны
- Нет повторяющихся групп

### Вторая нормальная форма (2NF)
- Выполняется 1NF
- Все неключевые атрибуты полностью зависят от первичного ключа

### Третья нормальная форма (3NF)
- Выполняется 2NF
- Нет транзитивных зависимостей

```sql
-- Пример денормализованной таблицы
CREATE TABLE orders_denorm (
    order_id INTEGER,
    customer_name VARCHAR(100),
    customer_email VARCHAR(255),  -- зависит от customer, не от order
    product_name VARCHAR(100),
    product_price DECIMAL(10,2),  -- зависит от product, не от order
    quantity INTEGER
);

-- Нормализованная структура (3NF)
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255)
);

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2)
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE order_items (
    order_id INTEGER REFERENCES orders(id),
    product_id INTEGER REFERENCES products(id),
    quantity INTEGER,
    price_at_order DECIMAL(10,2),  -- цена на момент заказа
    PRIMARY KEY (order_id, product_id)
);
```

## Best Practices

1. **Всегда определяйте первичный ключ** — каждая таблица должна иметь способ уникальной идентификации строк.

2. **Используйте внешние ключи** для обеспечения ссылочной целостности.

3. **Нормализуйте до 3NF** как базовое правило, денормализуйте осознанно для производительности.

4. **Выбирайте правильные типы данных** — это часть домена (domain) в терминах реляционной модели.

5. **Документируйте связи** между таблицами с помощью ER-диаграмм.

## Ограничения реляционной модели

- Не оптимальна для иерархических и графовых данных
- Сложность работы с полуструктурированными данными (решается через JSON/JSONB в PostgreSQL)
- Горизонтальное масштабирование сложнее, чем в NoSQL

PostgreSQL расширяет классическую реляционную модель объектными возможностями, сохраняя все её преимущества.

---
[prev: 01-object-model](./01-object-model.md) | [next: 03-tables-rows-columns](./03-tables-rows-columns.md)