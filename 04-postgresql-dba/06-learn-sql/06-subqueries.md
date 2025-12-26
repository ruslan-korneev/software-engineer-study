# Подзапросы (Subqueries)

## Что такое подзапрос?

**Подзапрос** — это запрос, вложенный в другой SQL-запрос. Подзапрос заключается в круглые скобки и может использоваться в различных частях основного запроса.

### Классификация подзапросов

По месту использования:
- В `SELECT` — скалярный подзапрос
- В `FROM` — производная таблица
- В `WHERE` — условный подзапрос
- В `HAVING` — условие для групп

По типу результата:
- **Скалярный** — возвращает одно значение
- **Строковый** — возвращает одну строку
- **Табличный** — возвращает набор строк
- **Коррелированный** — зависит от внешнего запроса

---

## Скалярные подзапросы

Возвращают одно значение и могут использоваться везде, где ожидается одно значение:

```sql
-- В SELECT
SELECT
    name,
    price,
    (SELECT AVG(price) FROM products) AS avg_price,
    price - (SELECT AVG(price) FROM products) AS diff_from_avg
FROM products;

-- В WHERE
SELECT * FROM products
WHERE price > (SELECT AVG(price) FROM products);

-- В UPDATE
UPDATE products
SET price = (SELECT MAX(price) FROM products WHERE category = 'premium')
WHERE id = 1;

-- Пример: найти последний заказ пользователя
SELECT
    u.name,
    (SELECT MAX(order_date) FROM orders WHERE user_id = u.id) AS last_order
FROM users u;
```

### Ограничения скалярных подзапросов

```sql
-- Ошибка: подзапрос возвращает более одной строки
SELECT * FROM products
WHERE price = (SELECT price FROM products WHERE category = 'electronics');
-- ERROR: more than one row returned by a subquery

-- Решение 1: добавить LIMIT
SELECT * FROM products
WHERE price = (SELECT price FROM products WHERE category = 'electronics' LIMIT 1);

-- Решение 2: использовать агрегатную функцию
SELECT * FROM products
WHERE price = (SELECT MIN(price) FROM products WHERE category = 'electronics');

-- Решение 3: использовать IN
SELECT * FROM products
WHERE price IN (SELECT price FROM products WHERE category = 'electronics');
```

---

## Подзапросы в FROM (Производные таблицы)

Подзапрос создает временную таблицу для использования в основном запросе:

```sql
-- Статистика по категориям
SELECT
    category,
    product_count,
    avg_price
FROM (
    SELECT
        category,
        COUNT(*) AS product_count,
        AVG(price) AS avg_price
    FROM products
    GROUP BY category
) AS category_stats
WHERE product_count > 10;

-- Ранжирование с дополнительными вычислениями
SELECT
    user_id,
    total_orders,
    total_amount,
    CASE
        WHEN total_amount > 10000 THEN 'VIP'
        WHEN total_amount > 1000 THEN 'Regular'
        ELSE 'New'
    END AS customer_type
FROM (
    SELECT
        user_id,
        COUNT(*) AS total_orders,
        SUM(amount) AS total_amount
    FROM orders
    GROUP BY user_id
) AS user_stats;
```

### Обязательный псевдоним

```sql
-- Ошибка: подзапрос в FROM требует псевдоним
SELECT * FROM (SELECT * FROM users);
-- ERROR: subquery in FROM must have an alias

-- Правильно
SELECT * FROM (SELECT * FROM users) AS u;
```

---

## Подзапросы в WHERE

### IN / NOT IN

```sql
-- Пользователи, сделавшие заказы
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders);

-- Продукты без заказов
SELECT * FROM products
WHERE id NOT IN (SELECT product_id FROM order_items WHERE product_id IS NOT NULL);

-- ВНИМАНИЕ: NOT IN с NULL дает неожиданный результат
SELECT * FROM users
WHERE id NOT IN (1, 2, NULL);  -- Вернет 0 строк!

-- Безопасный вариант
SELECT * FROM users
WHERE id NOT IN (SELECT user_id FROM orders WHERE user_id IS NOT NULL);
```

### EXISTS / NOT EXISTS

Проверяет существование строк в подзапросе:

```sql
-- Пользователи с заказами (лучше чем IN)
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- Пользователи без заказов
SELECT * FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- EXISTS обычно эффективнее IN для больших наборов данных
-- PostgreSQL может остановиться после первого совпадения
```

### Сравнение с ANY / ALL

```sql
-- ANY (SOME) - хотя бы одно значение
SELECT * FROM products
WHERE price > ANY (SELECT price FROM products WHERE category = 'budget');
-- Эквивалентно: price > MIN(...)

-- ALL - все значения
SELECT * FROM products
WHERE price > ALL (SELECT price FROM products WHERE category = 'budget');
-- Эквивалентно: price > MAX(...)

-- Другие операторы
SELECT * FROM products
WHERE price = ANY (SELECT price FROM products WHERE category = 'sale');
-- Эквивалентно: price IN (...)
```

---

## Коррелированные подзапросы

Зависят от значений внешнего запроса и выполняются для каждой строки:

```sql
-- Продукты дороже средней цены в своей категории
SELECT p.*
FROM products p
WHERE price > (
    SELECT AVG(price)
    FROM products
    WHERE category = p.category  -- ссылка на внешний запрос
);

-- Последний заказ каждого пользователя
SELECT * FROM orders o1
WHERE order_date = (
    SELECT MAX(order_date)
    FROM orders o2
    WHERE o2.user_id = o1.user_id
);

-- Количество заказов для каждого пользователя
SELECT
    u.name,
    (SELECT COUNT(*) FROM orders WHERE user_id = u.id) AS order_count
FROM users u;
```

### Производительность коррелированных подзапросов

```sql
-- Медленно: подзапрос выполняется для каждой строки
SELECT
    p.name,
    (SELECT c.name FROM categories c WHERE c.id = p.category_id) AS category
FROM products p;

-- Быстрее: использовать JOIN
SELECT p.name, c.name AS category
FROM products p
LEFT JOIN categories c ON p.category_id = c.id;
```

---

## Подзапросы в HAVING

```sql
-- Категории с количеством товаров выше среднего
SELECT category, COUNT(*) AS product_count
FROM products
GROUP BY category
HAVING COUNT(*) > (
    SELECT AVG(cnt) FROM (
        SELECT COUNT(*) AS cnt
        FROM products
        GROUP BY category
    ) AS cat_counts
);

-- Пользователи с суммой заказов выше средней
SELECT
    user_id,
    SUM(amount) AS total
FROM orders
GROUP BY user_id
HAVING SUM(amount) > (SELECT AVG(amount) * 5 FROM orders);
```

---

## Вложенные подзапросы

```sql
-- Многоуровневые подзапросы
SELECT * FROM products
WHERE category_id IN (
    SELECT id FROM categories
    WHERE parent_id IN (
        SELECT id FROM categories
        WHERE name = 'Electronics'
    )
);

-- Тот же результат через рекурсивный CTE (лучше)
WITH RECURSIVE subcategories AS (
    SELECT id FROM categories WHERE name = 'Electronics'
    UNION ALL
    SELECT c.id FROM categories c
    JOIN subcategories s ON c.parent_id = s.id
)
SELECT * FROM products WHERE category_id IN (SELECT id FROM subcategories);
```

---

## Практические примеры

### Топ-N в каждой группе

```sql
-- Топ-3 самых дорогих товара в каждой категории
SELECT * FROM (
    SELECT
        *,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY price DESC) AS rn
    FROM products
) ranked
WHERE rn <= 3;

-- Альтернатива с LATERAL (часто эффективнее)
SELECT c.name, p.*
FROM categories c
CROSS JOIN LATERAL (
    SELECT * FROM products
    WHERE category_id = c.id
    ORDER BY price DESC
    LIMIT 3
) p;
```

### Процент от общего

```sql
-- Доля продаж каждого продукта
SELECT
    product_name,
    total_sales,
    ROUND(
        total_sales * 100.0 / (SELECT SUM(amount) FROM orders),
        2
    ) AS percentage
FROM (
    SELECT
        p.name AS product_name,
        SUM(oi.quantity * oi.price) AS total_sales
    FROM order_items oi
    JOIN products p ON oi.product_id = p.id
    GROUP BY p.id, p.name
) product_sales
ORDER BY percentage DESC;

-- Эффективнее с оконными функциями
SELECT
    p.name,
    SUM(oi.quantity * oi.price) AS total_sales,
    ROUND(
        SUM(oi.quantity * oi.price) * 100.0 /
        SUM(SUM(oi.quantity * oi.price)) OVER (),
        2
    ) AS percentage
FROM order_items oi
JOIN products p ON oi.product_id = p.id
GROUP BY p.id, p.name;
```

### Кумулятивные вычисления

```sql
-- Накопительная сумма заказов
SELECT
    order_date,
    daily_total,
    (
        SELECT SUM(amount)
        FROM orders o2
        WHERE o2.order_date <= o1.order_date
    ) AS running_total
FROM (
    SELECT
        order_date,
        SUM(amount) AS daily_total
    FROM orders o1
    GROUP BY order_date
) o1
ORDER BY order_date;

-- Лучше с оконной функцией
SELECT
    order_date,
    SUM(amount) AS daily_total,
    SUM(SUM(amount)) OVER (ORDER BY order_date) AS running_total
FROM orders
GROUP BY order_date;
```

### Разница с предыдущей записью

```sql
-- Изменение продаж по сравнению с предыдущим месяцем
SELECT
    month,
    total_sales,
    total_sales - (
        SELECT total_sales
        FROM monthly_sales ms2
        WHERE ms2.month = ms1.month - INTERVAL '1 month'
    ) AS change
FROM monthly_sales ms1;

-- С оконной функцией LAG
SELECT
    month,
    total_sales,
    total_sales - LAG(total_sales) OVER (ORDER BY month) AS change
FROM monthly_sales;
```

---

## EXISTS vs IN vs JOIN

### Сравнение производительности

```sql
-- EXISTS (обычно лучше для больших подзапросов)
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders WHERE user_id = u.id);

-- IN (хорошо для маленьких списков)
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders);

-- JOIN (когда нужны данные из обеих таблиц)
SELECT DISTINCT u.*
FROM users u
JOIN orders o ON u.id = o.user_id;
```

### Когда что использовать

| Ситуация | Рекомендация |
|----------|--------------|
| Проверка существования | EXISTS |
| Маленький статичный список | IN |
| Нужны данные из подзапроса | JOIN |
| NOT IN с возможными NULL | NOT EXISTS |
| Большой подзапрос | EXISTS или JOIN |

---

## Оптимизация подзапросов

### Избегайте коррелированных подзапросов в SELECT

```sql
-- Плохо: N+1 запросов
SELECT
    u.name,
    (SELECT COUNT(*) FROM orders WHERE user_id = u.id) AS order_count
FROM users u;

-- Хорошо: один запрос с JOIN
SELECT
    u.name,
    COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;
```

### Материализация подзапросов

```sql
-- PostgreSQL может материализовать подзапрос
SELECT * FROM products
WHERE category_id IN (
    SELECT id FROM categories WHERE active = true
);

-- Принудительная материализация с CTE
WITH active_categories AS MATERIALIZED (
    SELECT id FROM categories WHERE active = true
)
SELECT * FROM products
WHERE category_id IN (SELECT id FROM active_categories);
```

### Использование EXPLAIN

```sql
EXPLAIN ANALYZE
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- Смотрите на:
-- - SubPlan vs HashJoin
-- - Количество выполнений подзапроса
-- - Использование индексов
```

---

## Типичные ошибки

### 1. NULL в NOT IN

```sql
-- Вернет 0 строк если в подзапросе есть NULL
SELECT * FROM users WHERE id NOT IN (1, 2, NULL);

-- Решение
SELECT * FROM users WHERE id NOT IN (
    SELECT user_id FROM orders WHERE user_id IS NOT NULL
);
-- Или использовать NOT EXISTS
```

### 2. Многократное выполнение одного подзапроса

```sql
-- Плохо: подзапрос выполняется дважды
SELECT
    (SELECT AVG(price) FROM products) AS avg_price,
    (SELECT COUNT(*) FROM products WHERE price > (SELECT AVG(price) FROM products)) AS above_avg
;

-- Хорошо: использовать CTE
WITH stats AS (
    SELECT AVG(price) AS avg_price FROM products
)
SELECT
    avg_price,
    (SELECT COUNT(*) FROM products WHERE price > stats.avg_price) AS above_avg
FROM stats;
```

### 3. Подзапрос вместо JOIN

```sql
-- Медленно
SELECT
    u.name,
    (SELECT name FROM departments d WHERE d.id = u.department_id) AS dept_name
FROM users u;

-- Быстро
SELECT u.name, d.name AS dept_name
FROM users u
LEFT JOIN departments d ON u.department_id = d.id;
```
