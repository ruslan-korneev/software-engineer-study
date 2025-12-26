# Соединение таблиц (JOIN)

## Что такое JOIN?

**JOIN** — это операция объединения строк из двух или более таблиц на основе связанных столбцов. JOIN позволяет получать данные из нескольких таблиц в одном запросе.

---

## Подготовка тестовых данных

```sql
-- Таблица пользователей
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    department_id INTEGER
);

-- Таблица отделов
CREATE TABLE departments (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

-- Таблица заказов
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    amount NUMERIC(10, 2),
    order_date DATE
);

-- Наполнение данными
INSERT INTO departments (id, name) VALUES
    (1, 'Engineering'),
    (2, 'Sales'),
    (3, 'Marketing'),
    (4, 'HR');  -- отдел без сотрудников

INSERT INTO users (id, name, department_id) VALUES
    (1, 'Alice', 1),
    (2, 'Bob', 1),
    (3, 'Charlie', 2),
    (4, 'Diana', NULL),  -- пользователь без отдела
    (5, 'Eve', 3);

INSERT INTO orders (user_id, amount, order_date) VALUES
    (1, 100.00, '2024-01-15'),
    (1, 250.00, '2024-01-20'),
    (2, 75.50, '2024-01-18'),
    (3, 500.00, '2024-01-22');
    -- Eve (id=5) не имеет заказов
```

---

## Типы JOIN

### INNER JOIN (Внутреннее соединение)

Возвращает только те строки, где есть совпадение в обеих таблицах.

```sql
-- Пользователи с их отделами (только те, у кого есть отдел)
SELECT u.name, d.name AS department
FROM users u
INNER JOIN departments d ON u.department_id = d.id;

-- Результат:
-- name    | department
-- --------|------------
-- Alice   | Engineering
-- Bob     | Engineering
-- Charlie | Sales
-- Eve     | Marketing
```

```
users                departments
+----+-------+----+   +----+------------+
| id | name  |dep |   | id | name       |
+----+-------+----+   +----+------------+
| 1  | Alice | 1  |-->| 1  | Engineering|
| 2  | Bob   | 1  |-->| 1  | Engineering|
| 3  | Charlie| 2 |-->| 2  | Sales      |
| 4  | Diana |NULL|   | 3  | Marketing  |
| 5  | Eve   | 3  |-->| 3  | Marketing  |
+----+-------+----+   | 4  | HR         |
                      +----+------------+
```

### LEFT JOIN (LEFT OUTER JOIN)

Возвращает все строки из левой таблицы и совпадающие строки из правой. Если совпадения нет — NULL.

```sql
-- Все пользователи с их отделами (включая тех, у кого отдела нет)
SELECT u.name, d.name AS department
FROM users u
LEFT JOIN departments d ON u.department_id = d.id;

-- Результат:
-- name    | department
-- --------|------------
-- Alice   | Engineering
-- Bob     | Engineering
-- Charlie | Sales
-- Diana   | NULL        <-- пользователь без отдела
-- Eve     | Marketing
```

### RIGHT JOIN (RIGHT OUTER JOIN)

Возвращает все строки из правой таблицы и совпадающие строки из левой.

```sql
-- Все отделы с их сотрудниками (включая пустые отделы)
SELECT u.name, d.name AS department
FROM users u
RIGHT JOIN departments d ON u.department_id = d.id;

-- Результат:
-- name    | department
-- --------|------------
-- Alice   | Engineering
-- Bob     | Engineering
-- Charlie | Sales
-- Eve     | Marketing
-- NULL    | HR          <-- отдел без сотрудников
```

### FULL OUTER JOIN

Возвращает все строки из обеих таблиц. NULL там, где нет совпадения.

```sql
SELECT u.name, d.name AS department
FROM users u
FULL OUTER JOIN departments d ON u.department_id = d.id;

-- Результат:
-- name    | department
-- --------|------------
-- Alice   | Engineering
-- Bob     | Engineering
-- Charlie | Sales
-- Eve     | Marketing
-- Diana   | NULL        <-- пользователь без отдела
-- NULL    | HR          <-- отдел без сотрудников
```

### CROSS JOIN (Декартово произведение)

Каждая строка из первой таблицы соединяется с каждой строкой из второй.

```sql
SELECT u.name, d.name AS department
FROM users u
CROSS JOIN departments d;

-- Результат: 5 users × 4 departments = 20 строк
```

```sql
-- Практическое применение: генерация всех комбинаций
SELECT
    p.name AS product,
    c.name AS color
FROM products p
CROSS JOIN colors c;
```

---

## Синтаксис JOIN

### Явный синтаксис (рекомендуется)

```sql
SELECT *
FROM table1
JOIN table2 ON table1.column = table2.column;
```

### Неявный синтаксис (устаревший)

```sql
-- Старый стиль (не рекомендуется)
SELECT *
FROM table1, table2
WHERE table1.column = table2.column;

-- Эквивалентно CROSS JOIN если забыть WHERE!
SELECT *
FROM table1, table2;  -- Декартово произведение!
```

### USING — упрощенный синтаксис

Когда столбцы имеют одинаковое имя в обеих таблицах:

```sql
-- Вместо
SELECT * FROM orders o
JOIN users u ON o.user_id = u.id;

-- Если столбец называется одинаково
SELECT * FROM orders
JOIN users USING (user_id);  -- требует user_id в обеих таблицах
```

### NATURAL JOIN

Автоматически соединяет по всем одноименным столбцам (не рекомендуется):

```sql
-- Опасно: может соединить по неожиданным столбцам
SELECT * FROM orders NATURAL JOIN users;
```

---

## Соединение нескольких таблиц

```sql
-- Пользователи, их отделы и заказы
SELECT
    u.name AS user_name,
    d.name AS department,
    o.amount,
    o.order_date
FROM users u
LEFT JOIN departments d ON u.department_id = d.id
LEFT JOIN orders o ON u.id = o.user_id
ORDER BY u.name, o.order_date;

-- Результат:
-- user_name | department  | amount  | order_date
-- ----------|-------------|---------|------------
-- Alice     | Engineering | 100.00  | 2024-01-15
-- Alice     | Engineering | 250.00  | 2024-01-20
-- Bob       | Engineering | 75.50   | 2024-01-18
-- Charlie   | Sales       | 500.00  | 2024-01-22
-- Diana     | NULL        | NULL    | NULL
-- Eve       | Marketing   | NULL    | NULL
```

---

## Self JOIN (Самосоединение)

Соединение таблицы с самой собой:

```sql
-- Таблица сотрудников с менеджерами
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    manager_id INTEGER REFERENCES employees(id)
);

INSERT INTO employees (id, name, manager_id) VALUES
    (1, 'CEO', NULL),
    (2, 'CTO', 1),
    (3, 'Developer', 2),
    (4, 'Designer', 2);

-- Сотрудники и их менеджеры
SELECT
    e.name AS employee,
    m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- Результат:
-- employee  | manager
-- ----------|---------
-- CEO       | NULL
-- CTO       | CEO
-- Developer | CTO
-- Designer  | CTO
```

---

## Условия в JOIN

### Множественные условия

```sql
-- Соединение по нескольким столбцам
SELECT *
FROM order_items oi
JOIN products p ON oi.product_id = p.id
                AND oi.warehouse_id = p.warehouse_id;
```

### Фильтрация в ON vs WHERE

```sql
-- Разница важна для OUTER JOIN!

-- Фильтр в ON: применяется ДО соединения
SELECT u.name, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id AND o.amount > 100;
-- Вернет всех пользователей, но заказы только > 100

-- Фильтр в WHERE: применяется ПОСЛЕ соединения
SELECT u.name, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.amount > 100;
-- Отфильтрует пользователей без заказов > 100!
```

### Неравенства в JOIN

```sql
-- Диапазонное соединение
SELECT e.name, s.grade
FROM employees e
JOIN salary_grades s ON e.salary BETWEEN s.min_salary AND s.max_salary;

-- Временные интервалы
SELECT *
FROM events e1
JOIN events e2 ON e1.id < e2.id  -- не дублировать пары
               AND e1.end_time > e2.start_time;  -- пересечение
```

---

## Anti-Join и Semi-Join

### Anti-Join (записи без совпадений)

```sql
-- Пользователи без заказов

-- Способ 1: LEFT JOIN + WHERE NULL
SELECT u.*
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.id IS NULL;

-- Способ 2: NOT EXISTS
SELECT *
FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- Способ 3: NOT IN (осторожно с NULL!)
SELECT *
FROM users
WHERE id NOT IN (SELECT user_id FROM orders WHERE user_id IS NOT NULL);
```

### Semi-Join (записи с совпадениями)

```sql
-- Пользователи с хотя бы одним заказом

-- Способ 1: EXISTS
SELECT *
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- Способ 2: IN
SELECT *
FROM users
WHERE id IN (SELECT user_id FROM orders);

-- Способ 3: JOIN + DISTINCT
SELECT DISTINCT u.*
FROM users u
JOIN orders o ON u.id = o.user_id;
```

---

## Производительность JOIN

### Использование индексов

```sql
-- Индексы на столбцах соединения критически важны!
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_users_department_id ON users(department_id);
```

### Порядок таблиц

PostgreSQL оптимизатор обычно сам выбирает оптимальный порядок, но:

```sql
-- Принудительный порядок соединения
SET join_collapse_limit = 1;

SELECT *
FROM small_table s
JOIN large_table l ON s.id = l.small_id;
```

### EXPLAIN для анализа

```sql
EXPLAIN ANALYZE
SELECT u.name, d.name
FROM users u
JOIN departments d ON u.department_id = d.id;

-- Смотрите на:
-- - Тип соединения (Nested Loop, Hash Join, Merge Join)
-- - Использование индексов
-- - Estimated vs Actual rows
```

---

## Практические примеры

### Агрегация с JOIN

```sql
-- Сумма заказов по отделам
SELECT
    d.name AS department,
    COUNT(o.id) AS order_count,
    COALESCE(SUM(o.amount), 0) AS total_amount
FROM departments d
LEFT JOIN users u ON d.id = u.department_id
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY d.id, d.name
ORDER BY total_amount DESC;
```

### Последний заказ каждого пользователя

```sql
-- С использованием подзапроса
SELECT u.name, o.amount, o.order_date
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.order_date = (
    SELECT MAX(order_date)
    FROM orders
    WHERE user_id = u.id
);

-- С использованием LATERAL (лучше)
SELECT u.name, lo.amount, lo.order_date
FROM users u
JOIN LATERAL (
    SELECT amount, order_date
    FROM orders
    WHERE user_id = u.id
    ORDER BY order_date DESC
    LIMIT 1
) lo ON true;

-- С использованием DISTINCT ON (PostgreSQL)
SELECT DISTINCT ON (u.id)
    u.name, o.amount, o.order_date
FROM users u
JOIN orders o ON u.id = o.user_id
ORDER BY u.id, o.order_date DESC;
```

### Иерархические данные (рекурсивный CTE)

```sql
-- Все подчиненные менеджера
WITH RECURSIVE subordinates AS (
    -- Базовый случай
    SELECT id, name, manager_id, 1 AS level
    FROM employees
    WHERE id = 2  -- CTO

    UNION ALL

    -- Рекурсивная часть
    SELECT e.id, e.name, e.manager_id, s.level + 1
    FROM employees e
    JOIN subordinates s ON e.manager_id = s.id
)
SELECT * FROM subordinates;
```

---

## Типичные ошибки

### 1. Забыть условие соединения

```sql
-- Создает декартово произведение!
SELECT * FROM users, orders;  -- users × orders строк

-- Правильно
SELECT * FROM users u JOIN orders o ON u.id = o.user_id;
```

### 2. Неправильный тип JOIN

```sql
-- Теряем пользователей без заказов
SELECT u.name, COUNT(o.id)
FROM users u
INNER JOIN orders o ON u.id = o.user_id
GROUP BY u.id;

-- Правильно: LEFT JOIN
SELECT u.name, COUNT(o.id)
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id;
```

### 3. Дублирование строк

```sql
-- Если у пользователя несколько заказов, строка дублируется
SELECT u.name, u.email, o.order_date
FROM users u
JOIN orders o ON u.id = o.user_id;

-- Решение: агрегация или DISTINCT
SELECT DISTINCT u.name, u.email
FROM users u
JOIN orders o ON u.id = o.user_id;
```

### 4. NULL в условии соединения

```sql
-- NULL != NULL, поэтому строки с NULL не соединятся
SELECT *
FROM table1 t1
JOIN table2 t2 ON t1.nullable_col = t2.nullable_col;

-- Если нужно соединять NULL
SELECT *
FROM table1 t1
JOIN table2 t2 ON t1.nullable_col IS NOT DISTINCT FROM t2.nullable_col;
```

---

## Лучшие практики

1. **Всегда используйте псевдонимы** для таблиц
2. **Указывайте схему** для столбцов (`u.name`, не `name`)
3. **Создавайте индексы** на столбцах соединения
4. **Используйте EXPLAIN** для анализа производительности
5. **Выбирайте правильный тип JOIN** в зависимости от задачи
6. **Фильтруйте раньше** — используйте WHERE до JOIN где возможно
