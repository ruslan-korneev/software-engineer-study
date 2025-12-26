# Паттерны написания оптимальных запросов

## Введение

Оптимизация SQL-запросов — ключевой навык для работы с PostgreSQL. Правильно написанный запрос может работать в сотни раз быстрее неоптимального. В этом разделе рассмотрим основные паттерны и антипаттерны.

---

## 1. Common Table Expressions (CTE)

### Основы CTE

CTE (WITH-запросы) улучшают читаемость и позволяют переиспользовать подзапросы:

```sql
-- Без CTE (сложно читать)
SELECT * FROM (
    SELECT customer_id, SUM(amount) as total
    FROM orders
    WHERE created_at > '2024-01-01'
    GROUP BY customer_id
) sub
WHERE total > 1000;

-- С CTE (читаемо)
WITH customer_totals AS (
    SELECT
        customer_id,
        SUM(amount) as total
    FROM orders
    WHERE created_at > '2024-01-01'
    GROUP BY customer_id
)
SELECT * FROM customer_totals WHERE total > 1000;
```

### Множественные CTE

```sql
WITH
-- Первый CTE
active_users AS (
    SELECT id, name, email
    FROM users
    WHERE status = 'active'
),
-- Второй CTE использует первый
user_orders AS (
    SELECT
        u.id,
        u.name,
        COUNT(o.id) as order_count,
        SUM(o.amount) as total_spent
    FROM active_users u
    LEFT JOIN orders o ON o.user_id = u.id
    GROUP BY u.id, u.name
)
SELECT *
FROM user_orders
WHERE order_count > 5
ORDER BY total_spent DESC;
```

### CTE MATERIALIZED vs NOT MATERIALIZED

PostgreSQL 12+ позволяет контролировать материализацию CTE:

```sql
-- Принудительная материализация (сначала выполняется CTE)
WITH expensive_calc AS MATERIALIZED (
    SELECT id, complex_function(data) as result
    FROM big_table
)
SELECT * FROM expensive_calc WHERE result > 100;

-- Без материализации (оптимизатор может inline CTE)
WITH simple_filter AS NOT MATERIALIZED (
    SELECT * FROM users WHERE status = 'active'
)
SELECT * FROM simple_filter WHERE id = 123;
```

**Когда использовать MATERIALIZED:**
- CTE используется несколько раз
- Нужно изолировать сложные вычисления
- Хотите предотвратить inline для отладки

**Когда использовать NOT MATERIALIZED:**
- CTE используется один раз
- Важна оптимизация с push-down фильтров

### Рекурсивные CTE

```sql
-- Обход иерархии (категории с подкатегориями)
WITH RECURSIVE category_tree AS (
    -- Базовый случай
    SELECT id, name, parent_id, 1 as level, ARRAY[id] as path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- Рекурсивный случай
    SELECT
        c.id,
        c.name,
        c.parent_id,
        ct.level + 1,
        ct.path || c.id
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
    WHERE ct.level < 10  -- Защита от бесконечной рекурсии
)
SELECT * FROM category_tree ORDER BY path;

-- Пример вывода:
-- id | name           | level | path
-- 1  | Electronics    | 1     | {1}
-- 2  | Phones         | 2     | {1,2}
-- 3  | Smartphones    | 3     | {1,2,3}
-- 4  | Computers      | 2     | {1,4}
```

---

## 2. Оконные функции

### Базовые оконные функции

```sql
-- ROW_NUMBER - уникальный номер строки
SELECT
    name,
    category,
    price,
    ROW_NUMBER() OVER (ORDER BY price DESC) as rank
FROM products;

-- RANK - ранг с пропусками при равенстве
SELECT
    name,
    score,
    RANK() OVER (ORDER BY score DESC) as rank
FROM players;
-- score: 100, 100, 90 -> rank: 1, 1, 3

-- DENSE_RANK - ранг без пропусков
SELECT
    name,
    score,
    DENSE_RANK() OVER (ORDER BY score DESC) as rank
FROM players;
-- score: 100, 100, 90 -> rank: 1, 1, 2
```

### Партиционирование окон

```sql
-- Топ-3 продукта в каждой категории
SELECT * FROM (
    SELECT
        name,
        category,
        price,
        ROW_NUMBER() OVER (
            PARTITION BY category
            ORDER BY price DESC
        ) as rank_in_category
    FROM products
) ranked
WHERE rank_in_category <= 3;
```

### Агрегатные оконные функции

```sql
SELECT
    order_date,
    amount,
    -- Накопительная сумма
    SUM(amount) OVER (ORDER BY order_date) as running_total,
    -- Скользящее среднее за 7 дней
    AVG(amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as moving_avg_7d,
    -- Процент от общей суммы
    amount * 100.0 / SUM(amount) OVER () as pct_of_total
FROM daily_sales;
```

### LAG и LEAD

```sql
-- Сравнение с предыдущим значением
SELECT
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) as prev_month_revenue,
    revenue - LAG(revenue) OVER (ORDER BY month) as change,
    LEAD(revenue) OVER (ORDER BY month) as next_month_revenue
FROM monthly_stats;

-- Пример вывода:
-- month   | revenue | prev_month | change | next_month
-- 2024-01 | 10000   | NULL       | NULL   | 12000
-- 2024-02 | 12000   | 10000      | 2000   | 11000
-- 2024-03 | 11000   | 12000      | -1000  | NULL
```

### FIRST_VALUE и LAST_VALUE

```sql
SELECT
    department,
    employee_name,
    salary,
    FIRST_VALUE(employee_name) OVER (
        PARTITION BY department
        ORDER BY salary DESC
    ) as highest_paid,
    LAST_VALUE(employee_name) OVER (
        PARTITION BY department
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as lowest_paid
FROM employees;
```

---

## 3. LATERAL JOIN

### Что такое LATERAL

LATERAL позволяет подзапросу ссылаться на столбцы из предшествующих таблиц:

```sql
-- Получить последние 3 заказа каждого клиента
SELECT
    c.id,
    c.name,
    recent_orders.*
FROM customers c
CROSS JOIN LATERAL (
    SELECT id, amount, created_at
    FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY created_at DESC
    LIMIT 3
) recent_orders;
```

### LATERAL vs обычный подзапрос

```sql
-- Без LATERAL (не работает - нельзя ссылаться на c.id)
SELECT c.*, (
    SELECT * FROM orders
    WHERE customer_id = c.id
    LIMIT 3
) FROM customers c;
-- ERROR!

-- С LATERAL (работает)
SELECT c.*, o.*
FROM customers c
CROSS JOIN LATERAL (
    SELECT * FROM orders
    WHERE customer_id = c.id
    LIMIT 3
) o;
```

### Практические примеры LATERAL

```sql
-- Top-N per group эффективнее чем ROW_NUMBER
SELECT d.name, top_employees.*
FROM departments d
CROSS JOIN LATERAL (
    SELECT e.name, e.salary
    FROM employees e
    WHERE e.department_id = d.id
    ORDER BY e.salary DESC
    LIMIT 5
) top_employees;

-- Развёртка массива с номером элемента
SELECT
    id,
    tag,
    ordinality
FROM articles,
LATERAL unnest(tags) WITH ORDINALITY AS t(tag, ordinality);
```

---

## 4. Правильные JOIN

### Типы JOIN и производительность

```sql
-- INNER JOIN - только совпадающие записи
SELECT u.name, o.id
FROM users u
INNER JOIN orders o ON o.user_id = u.id;

-- LEFT JOIN - все записи из левой таблицы
SELECT u.name, o.id
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;

-- Используйте EXISTS вместо JOIN для проверки существования
-- Плохо
SELECT DISTINCT u.*
FROM users u
JOIN orders o ON o.user_id = u.id;

-- Хорошо
SELECT u.*
FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

### Порядок JOIN

```sql
-- PostgreSQL сам оптимизирует порядок JOIN (до join_collapse_limit)
-- Но для очень сложных запросов порядок может иметь значение

-- Сначала маленькие таблицы, потом большие (для ручной оптимизации)
SELECT *
FROM small_lookup_table s
JOIN large_fact_table l ON l.category_id = s.id
JOIN another_large_table a ON a.fact_id = l.id;
```

### Предикаты в JOIN vs WHERE

```sql
-- Для INNER JOIN - эквивалентно
SELECT * FROM a JOIN b ON a.id = b.a_id WHERE b.status = 'active';
SELECT * FROM a JOIN b ON a.id = b.a_id AND b.status = 'active';

-- Для LEFT JOIN - РАЗНОЕ поведение!
-- Это оставит все записи из a, даже без совпадений в b
SELECT * FROM a LEFT JOIN b ON a.id = b.a_id WHERE b.status = 'active';
-- Результат: только записи где b.status = 'active' (NULL отсеется)

-- Это фильтрует b ДО join
SELECT * FROM a LEFT JOIN b ON a.id = b.a_id AND b.status = 'active';
-- Результат: все записи из a, b только где status = 'active'
```

---

## 5. Антипаттерны и их решения

### 5.1. SELECT *

```sql
-- Плохо
SELECT * FROM users WHERE id = 1;

-- Хорошо
SELECT id, name, email FROM users WHERE id = 1;
```

**Почему плохо:**
- Тянет ненужные данные
- Нельзя использовать Index Only Scan
- Проблемы при изменении схемы

### 5.2. Функции на индексированных столбцах

```sql
-- Плохо (индекс на created_at не используется)
SELECT * FROM orders WHERE DATE(created_at) = '2024-01-15';
SELECT * FROM users WHERE LOWER(email) = 'test@example.com';

-- Хорошо (индекс используется)
SELECT * FROM orders
WHERE created_at >= '2024-01-15' AND created_at < '2024-01-16';

-- Или создать функциональный индекс
CREATE INDEX idx_users_email_lower ON users (LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'test@example.com';
```

### 5.3. OR вместо IN/ANY

```sql
-- Плохо (может не использовать индекс эффективно)
SELECT * FROM users WHERE status = 'active' OR status = 'pending';

-- Хорошо
SELECT * FROM users WHERE status IN ('active', 'pending');
SELECT * FROM users WHERE status = ANY(ARRAY['active', 'pending']);
```

### 5.4. NOT IN с NULL

```sql
-- Опасно! Если в subquery есть NULL, результат пустой
SELECT * FROM users
WHERE id NOT IN (SELECT user_id FROM banned_users);

-- Безопасно
SELECT * FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM banned_users b WHERE b.user_id = u.id
);
```

### 5.5. OFFSET для пагинации

```sql
-- Плохо (сканирует все пропущенные строки)
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 10000;

-- Хорошо (keyset pagination)
SELECT * FROM products
WHERE id > 10000  -- last_seen_id
ORDER BY id
LIMIT 20;
```

### 5.6. SELECT DISTINCT без необходимости

```sql
-- Часто DISTINCT скрывает проблему с JOIN
SELECT DISTINCT u.id, u.name
FROM users u
JOIN orders o ON o.user_id = u.id;

-- Лучше переосмыслить запрос
SELECT u.id, u.name
FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

### 5.7. Коррелированные подзапросы в SELECT

```sql
-- Плохо (выполняется для каждой строки)
SELECT
    u.name,
    (SELECT COUNT(*) FROM orders WHERE user_id = u.id) as order_count
FROM users u;

-- Хорошо
SELECT
    u.name,
    COALESCE(o.order_count, 0) as order_count
FROM users u
LEFT JOIN (
    SELECT user_id, COUNT(*) as order_count
    FROM orders
    GROUP BY user_id
) o ON o.user_id = u.id;
```

---

## 6. Полезные техники

### 6.1. COALESCE и NULLIF

```sql
-- Замена NULL на значение по умолчанию
SELECT COALESCE(nickname, username, 'Anonymous') as display_name
FROM users;

-- Защита от деления на ноль
SELECT total / NULLIF(count, 0) as average
FROM stats;
```

### 6.2. Условная агрегация

```sql
-- Вместо нескольких запросов
SELECT
    COUNT(*) FILTER (WHERE status = 'active') as active_count,
    COUNT(*) FILTER (WHERE status = 'pending') as pending_count,
    COUNT(*) FILTER (WHERE status = 'closed') as closed_count,
    SUM(amount) FILTER (WHERE status = 'active') as active_amount
FROM orders;
```

### 6.3. DISTINCT ON

```sql
-- Уникальная PostgreSQL-фича: первая запись в каждой группе
SELECT DISTINCT ON (customer_id)
    customer_id,
    id as last_order_id,
    amount,
    created_at
FROM orders
ORDER BY customer_id, created_at DESC;
```

### 6.4. Генерация серий

```sql
-- Заполнение пропусков в данных
SELECT
    d.date,
    COALESCE(o.count, 0) as orders_count
FROM generate_series(
    '2024-01-01'::date,
    '2024-01-31'::date,
    '1 day'
) as d(date)
LEFT JOIN (
    SELECT DATE(created_at) as date, COUNT(*) as count
    FROM orders
    GROUP BY DATE(created_at)
) o ON o.date = d.date;
```

### 6.5. UPSERT (INSERT ON CONFLICT)

```sql
-- Вставка или обновление
INSERT INTO user_stats (user_id, login_count, last_login)
VALUES (1, 1, NOW())
ON CONFLICT (user_id) DO UPDATE SET
    login_count = user_stats.login_count + 1,
    last_login = EXCLUDED.last_login;
```

---

## Best Practices

1. **Всегда проверяйте план выполнения** с `EXPLAIN ANALYZE`
2. **Используйте подготовленные запросы** для часто выполняемых операций
3. **Ограничивайте результат** с LIMIT даже для отладки
4. **Предпочитайте JOIN подзапросам** где возможно
5. **Используйте транзакции** для связанных операций
6. **Избегайте SELECT *** в production коде
7. **Проверяйте кардинальность** — понимайте сколько строк вернёт каждый шаг

---

## Шпаргалка по выбору подхода

| Задача | Решение |
|--------|---------|
| Top-N в группе | `ROW_NUMBER() OVER` или `LATERAL` |
| Проверка существования | `EXISTS`, не `JOIN` |
| Накопительная сумма | `SUM() OVER (ORDER BY ...)` |
| Сравнение с предыдущим | `LAG()` |
| Первая/последняя запись | `DISTINCT ON` или `FIRST_VALUE` |
| Иерархия | Recursive CTE |
| Пагинация | Keyset (WHERE id > last), не OFFSET |
| Условные агрегаты | `COUNT() FILTER (WHERE ...)` |
