# Обработка NULL в PostgreSQL

[prev: 06-constraints](./06-constraints.md) | [next: 01-acid](../03-high-level-database-concepts/01-acid.md)
---

## Что такое NULL?

**NULL** — это специальное значение, означающее "отсутствие данных" или "неизвестное значение". NULL — это не пустая строка, не ноль, не false. Это особое состояние, которое требует специальной обработки.

```sql
-- NULL не равен ничему, даже самому себе
SELECT NULL = NULL;      -- Результат: NULL (не TRUE, не FALSE!)
SELECT NULL <> NULL;     -- Результат: NULL
SELECT NULL = '';        -- Результат: NULL
SELECT NULL = 0;         -- Результат: NULL
SELECT NULL = FALSE;     -- Результат: NULL
```

## Трёхзначная логика (Three-Valued Logic)

В SQL используется трёхзначная логика: TRUE, FALSE и UNKNOWN (NULL):

### Таблица истинности для AND

| A | B | A AND B |
|---|---|---------|
| TRUE | TRUE | TRUE |
| TRUE | FALSE | FALSE |
| TRUE | NULL | NULL |
| FALSE | TRUE | FALSE |
| FALSE | FALSE | FALSE |
| FALSE | NULL | FALSE |
| NULL | TRUE | NULL |
| NULL | FALSE | FALSE |
| NULL | NULL | NULL |

### Таблица истинности для OR

| A | B | A OR B |
|---|---|--------|
| TRUE | TRUE | TRUE |
| TRUE | FALSE | TRUE |
| TRUE | NULL | TRUE |
| FALSE | TRUE | TRUE |
| FALSE | FALSE | FALSE |
| FALSE | NULL | NULL |
| NULL | TRUE | TRUE |
| NULL | FALSE | NULL |
| NULL | NULL | NULL |

### Таблица истинности для NOT

| A | NOT A |
|---|-------|
| TRUE | FALSE |
| FALSE | TRUE |
| NULL | NULL |

```sql
-- Примеры трёхзначной логики
SELECT TRUE AND NULL;    -- NULL
SELECT FALSE AND NULL;   -- FALSE (FALSE "побеждает")
SELECT TRUE OR NULL;     -- TRUE (TRUE "побеждает")
SELECT FALSE OR NULL;    -- NULL
SELECT NOT NULL;         -- NULL
```

## Проверка на NULL

### IS NULL / IS NOT NULL

```sql
-- Правильный способ проверки на NULL
SELECT * FROM employees WHERE manager_id IS NULL;
SELECT * FROM employees WHERE manager_id IS NOT NULL;

-- НЕПРАВИЛЬНО! Никогда не используйте = NULL
SELECT * FROM employees WHERE manager_id = NULL;     -- вернёт 0 строк!
SELECT * FROM employees WHERE manager_id <> NULL;    -- вернёт 0 строк!
```

### IS DISTINCT FROM / IS NOT DISTINCT FROM

Эти операторы трактуют NULL как обычное значение:

```sql
-- Обычное сравнение
SELECT NULL = NULL;              -- NULL
SELECT 1 = 1;                    -- TRUE
SELECT 1 = 2;                    -- FALSE

-- IS DISTINCT FROM (NULL-safe неравенство)
SELECT NULL IS DISTINCT FROM NULL;     -- FALSE (NULL равен NULL)
SELECT 1 IS DISTINCT FROM 1;           -- FALSE
SELECT 1 IS DISTINCT FROM 2;           -- TRUE
SELECT 1 IS DISTINCT FROM NULL;        -- TRUE

-- IS NOT DISTINCT FROM (NULL-safe равенство)
SELECT NULL IS NOT DISTINCT FROM NULL; -- TRUE
SELECT 1 IS NOT DISTINCT FROM 1;       -- TRUE
SELECT 1 IS NOT DISTINCT FROM NULL;    -- FALSE

-- Практическое применение
SELECT * FROM employees
WHERE manager_id IS NOT DISTINCT FROM 5;  -- включает сравнение с NULL
```

## NULL в выражениях

### Арифметика с NULL

```sql
-- Любая арифметическая операция с NULL даёт NULL
SELECT 5 + NULL;          -- NULL
SELECT 10 * NULL;         -- NULL
SELECT NULL / 2;          -- NULL
SELECT NULL - NULL;       -- NULL

-- Практический пример: вычисление итога
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    quantity INTEGER NOT NULL,
    unit_price NUMERIC(10, 2) NOT NULL,
    discount NUMERIC(10, 2)  -- может быть NULL
);

-- Проблема: если discount = NULL, total = NULL
SELECT quantity * unit_price - discount AS total FROM order_items;

-- Решение: использовать COALESCE
SELECT quantity * unit_price - COALESCE(discount, 0) AS total FROM order_items;
```

### Конкатенация строк

```sql
-- По умолчанию: NULL "заражает" результат
SELECT 'Hello' || NULL || 'World';  -- NULL

-- Функция CONCAT игнорирует NULL
SELECT CONCAT('Hello', NULL, 'World');  -- 'HelloWorld'

-- CONCAT_WS с разделителем
SELECT CONCAT_WS(' ', 'Hello', NULL, 'World');  -- 'Hello World'
```

## Функции для работы с NULL

### COALESCE

Возвращает первое не-NULL значение:

```sql
SELECT COALESCE(NULL, NULL, 'default');  -- 'default'
SELECT COALESCE(NULL, 'first', 'second'); -- 'first'
SELECT COALESCE('value', 'default');      -- 'value'

-- Практические примеры
SELECT
    name,
    COALESCE(nickname, name) AS display_name,
    COALESCE(phone, email, 'No contact') AS primary_contact
FROM users;

-- Значение по умолчанию для вычислений
SELECT
    product_name,
    price * (1 - COALESCE(discount_percent, 0) / 100) AS final_price
FROM products;
```

### NULLIF

Возвращает NULL, если два значения равны:

```sql
SELECT NULLIF(10, 10);    -- NULL
SELECT NULLIF(10, 20);    -- 10
SELECT NULLIF('a', 'b');  -- 'a'

-- Практический пример: избежание деления на ноль
SELECT
    total_amount / NULLIF(item_count, 0) AS avg_price
FROM orders;
-- Если item_count = 0, результат NULL вместо ошибки деления на ноль

-- Преобразование пустых строк в NULL
SELECT NULLIF(TRIM(user_input), '') AS clean_input;
```

### GREATEST / LEAST

```sql
-- NULL игнорируется
SELECT GREATEST(1, 5, 3, NULL);  -- 5
SELECT LEAST(1, 5, 3, NULL);     -- 1

-- Если все NULL, результат NULL
SELECT GREATEST(NULL, NULL);     -- NULL
```

### CASE с NULL

```sql
-- CASE WHEN с явной проверкой на NULL
SELECT
    name,
    CASE
        WHEN manager_id IS NULL THEN 'Top Level'
        WHEN manager_id = 1 THEN 'Reports to CEO'
        ELSE 'Regular Employee'
    END AS level
FROM employees;

-- Простой CASE не работает с NULL!
SELECT
    CASE NULL
        WHEN NULL THEN 'Matched'  -- Не сработает!
        ELSE 'Not matched'
    END;
-- Результат: 'Not matched' (потому что NULL = NULL даёт NULL, не TRUE)
```

## NULL в агрегатных функциях

### COUNT

```sql
CREATE TABLE test_nulls (
    id SERIAL PRIMARY KEY,
    value INTEGER
);

INSERT INTO test_nulls (value) VALUES (1), (2), (NULL), (4), (NULL);

-- COUNT(*) считает все строки, включая NULL
SELECT COUNT(*) FROM test_nulls;  -- 5

-- COUNT(column) игнорирует NULL
SELECT COUNT(value) FROM test_nulls;  -- 3

-- COUNT(DISTINCT column) также игнорирует NULL
SELECT COUNT(DISTINCT value) FROM test_nulls;  -- 3
```

### Другие агрегаты

```sql
-- SUM, AVG, MIN, MAX игнорируют NULL
SELECT SUM(value) FROM test_nulls;   -- 7 (1+2+4)
SELECT AVG(value) FROM test_nulls;   -- 2.33 (7/3, не 7/5)
SELECT MIN(value) FROM test_nulls;   -- 1
SELECT MAX(value) FROM test_nulls;   -- 4

-- Если все значения NULL
SELECT SUM(NULL::INTEGER);  -- NULL
SELECT AVG(NULL::INTEGER);  -- NULL

-- ARRAY_AGG включает NULL по умолчанию
SELECT ARRAY_AGG(value) FROM test_nulls;  -- {1,2,NULL,4,NULL}

-- Для исключения NULL
SELECT ARRAY_AGG(value) FILTER (WHERE value IS NOT NULL) FROM test_nulls;  -- {1,2,4}
```

## NULL в ORDER BY

```sql
-- По умолчанию NULL считается "больше" всех значений
SELECT * FROM employees ORDER BY manager_id ASC;
-- NULL будут в конце

SELECT * FROM employees ORDER BY manager_id DESC;
-- NULL будут в начале

-- Явное управление позицией NULL
SELECT * FROM employees ORDER BY manager_id ASC NULLS FIRST;
SELECT * FROM employees ORDER BY manager_id ASC NULLS LAST;
SELECT * FROM employees ORDER BY manager_id DESC NULLS FIRST;
SELECT * FROM employees ORDER BY manager_id DESC NULLS LAST;
```

## NULL в UNIQUE и индексах

```sql
-- UNIQUE допускает несколько NULL (NULL != NULL)
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE
);

INSERT INTO users (email) VALUES (NULL);  -- OK
INSERT INTO users (email) VALUES (NULL);  -- OK! (несколько NULL разрешены)

-- PostgreSQL 15+: UNIQUE NULLS NOT DISTINCT
CREATE TABLE users_strict (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255),
    CONSTRAINT users_strict_email_unique UNIQUE NULLS NOT DISTINCT (email)
);

INSERT INTO users_strict (email) VALUES (NULL);  -- OK
INSERT INTO users_strict (email) VALUES (NULL);  -- ERROR!

-- Индексы и NULL
CREATE INDEX idx_manager ON employees (manager_id);
-- Этот индекс включает NULL значения

-- Частичный индекс без NULL
CREATE INDEX idx_active_manager ON employees (manager_id)
WHERE manager_id IS NOT NULL;
```

## NULL в JOIN

```sql
CREATE TABLE departments (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE employees (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    department_id INTEGER  -- может быть NULL
);

INSERT INTO departments VALUES (1, 'IT'), (2, 'HR');
INSERT INTO employees VALUES (1, 'Alice', 1), (2, 'Bob', 2), (3, 'Charlie', NULL);

-- INNER JOIN: строки с NULL не попадают в результат
SELECT e.name, d.name
FROM employees e
INNER JOIN departments d ON e.department_id = d.id;
-- Alice, IT
-- Bob, HR
-- (Charlie не попал — department_id = NULL)

-- LEFT JOIN: сохраняет строки с NULL
SELECT e.name, d.name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id;
-- Alice, IT
-- Bob, HR
-- Charlie, NULL
```

## NULL и подзапросы

### IN / NOT IN с NULL

```sql
-- IN с NULL работает как ожидается
SELECT * FROM employees WHERE department_id IN (1, 2, NULL);
-- Возвращает сотрудников с department_id = 1 или 2
-- NULL в списке не влияет на результат

-- NOT IN с NULL — опасная ловушка!
SELECT * FROM employees WHERE department_id NOT IN (1, 2, NULL);
-- Вернёт 0 строк! Потому что:
-- NOT (department_id = 1 OR department_id = 2 OR department_id = NULL)
-- Для любого department_id: department_id = NULL даёт NULL
-- NOT (TRUE OR FALSE OR NULL) = NOT NULL = NULL

-- Решение 1: исключить NULL из списка
SELECT * FROM employees
WHERE department_id NOT IN (SELECT id FROM inactive_depts WHERE id IS NOT NULL);

-- Решение 2: использовать NOT EXISTS
SELECT * FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM inactive_depts d WHERE d.id = e.department_id
);
```

### ALL / ANY с NULL

```sql
-- ALL с пустым результатом или NULL
SELECT * FROM products WHERE price > ALL (SELECT price FROM products WHERE category = 'none');
-- Если подзапрос возвращает 0 строк — TRUE для всех
-- Если подзапрос содержит NULL — результат может быть NULL
```

## Best Practices

### 1. Используйте NOT NULL по умолчанию

```sql
-- Плохо: много опциональных полей
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255)
);

-- Хорошо: явно указаны обязательные поля
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL,
    nickname VARCHAR(50)  -- явно опциональное
);
```

### 2. Не используйте NULL для "пустых" значений

```sql
-- Плохо: NULL = "нет скидки"
discount NUMERIC  -- NULL означает что?

-- Хорошо: значение по умолчанию
discount NUMERIC NOT NULL DEFAULT 0

-- Или используйте NULL осознанно с комментарием
discount NUMERIC  -- NULL = скидка не применяется
```

### 3. Всегда обрабатывайте NULL в вычислениях

```sql
-- Плохо
SELECT price * quantity - discount AS total FROM orders;

-- Хорошо
SELECT price * quantity - COALESCE(discount, 0) AS total FROM orders;
```

### 4. Избегайте NOT IN с подзапросами

```sql
-- Опасно
SELECT * FROM a WHERE id NOT IN (SELECT id FROM b);

-- Безопасно
SELECT * FROM a WHERE id NOT IN (SELECT id FROM b WHERE id IS NOT NULL);

-- Или лучше
SELECT * FROM a WHERE NOT EXISTS (SELECT 1 FROM b WHERE b.id = a.id);
```

### 5. Документируйте семантику NULL

```sql
-- Добавляйте комментарии
CREATE TABLE contracts (
    id SERIAL PRIMARY KEY,
    start_date DATE NOT NULL,
    end_date DATE,  -- NULL = бессрочный контракт

    CONSTRAINT valid_dates CHECK (end_date IS NULL OR end_date > start_date)
);

COMMENT ON COLUMN contracts.end_date IS 'Дата окончания контракта. NULL означает бессрочный контракт.';
```

### 6. Используйте CHECK для валидации

```sql
-- Бизнес-правило: либо оба поля заполнены, либо оба NULL
CREATE TABLE address (
    id SERIAL PRIMARY KEY,
    city VARCHAR(100),
    postal_code VARCHAR(10),

    CONSTRAINT city_postal_both_or_none
        CHECK ((city IS NULL AND postal_code IS NULL) OR
               (city IS NOT NULL AND postal_code IS NOT NULL))
);
```

## Резюме

| Операция | Поведение с NULL |
|----------|------------------|
| `= NULL` | Всегда NULL (не TRUE) |
| `IS NULL` | Правильная проверка на NULL |
| `IS DISTINCT FROM` | NULL-safe сравнение |
| `COALESCE(a, b)` | Первое не-NULL |
| `NULLIF(a, b)` | NULL если a = b |
| `COUNT(*)` | Считает все строки |
| `COUNT(column)` | Игнорирует NULL |
| `SUM/AVG/MIN/MAX` | Игнорируют NULL |
| `UNIQUE` | Несколько NULL разрешены |
| `NOT IN` | Опасен с NULL в списке |
| `ORDER BY` | NULL "больше" всех (по умолчанию) |

Правильная обработка NULL критически важна для корректной работы с данными в PostgreSQL.

---
[prev: 06-constraints](./06-constraints.md) | [next: 01-acid](../03-high-level-database-concepts/01-acid.md)