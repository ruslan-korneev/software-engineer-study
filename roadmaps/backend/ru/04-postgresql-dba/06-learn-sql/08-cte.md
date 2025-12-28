# CTE (Common Table Expressions)

[prev: 07-advanced-topics](./07-advanced-topics.md) | [next: 09-lateral-join](./09-lateral-join.md)

---

## Что такое CTE?

**CTE (Common Table Expression)** — это именованный временный результат запроса, который можно использовать в основном запросе. CTE определяется с помощью ключевого слова `WITH` и существует только в рамках одного запроса.

### Преимущества CTE:
- Улучшение читаемости сложных запросов
- Возможность повторного использования подзапроса
- Рекурсивные запросы
- Упрощение отладки

---

## Базовый синтаксис

```sql
WITH cte_name AS (
    -- Определение CTE
    SELECT column1, column2
    FROM table_name
    WHERE condition
)
-- Основной запрос
SELECT * FROM cte_name;
```

### Простой пример

```sql
-- Без CTE
SELECT *
FROM (
    SELECT user_id, SUM(amount) AS total
    FROM orders
    GROUP BY user_id
) subquery
WHERE total > 1000;

-- С CTE — более читаемо
WITH user_totals AS (
    SELECT user_id, SUM(amount) AS total
    FROM orders
    GROUP BY user_id
)
SELECT * FROM user_totals WHERE total > 1000;
```

---

## Множественные CTE

Можно определить несколько CTE в одном запросе:

```sql
WITH
-- Первый CTE
active_users AS (
    SELECT id, name, email
    FROM users
    WHERE status = 'active'
),
-- Второй CTE (может ссылаться на первый)
user_orders AS (
    SELECT
        au.id AS user_id,
        au.name,
        COUNT(o.id) AS order_count,
        SUM(o.amount) AS total_amount
    FROM active_users au
    LEFT JOIN orders o ON au.id = o.user_id
    GROUP BY au.id, au.name
),
-- Третий CTE
vip_users AS (
    SELECT *
    FROM user_orders
    WHERE total_amount > 10000
)
-- Основной запрос
SELECT
    name,
    order_count,
    total_amount,
    CASE
        WHEN total_amount > 50000 THEN 'Platinum'
        WHEN total_amount > 20000 THEN 'Gold'
        ELSE 'Silver'
    END AS tier
FROM vip_users
ORDER BY total_amount DESC;
```

---

## CTE в разных операциях

### CTE с INSERT

```sql
WITH new_order AS (
    INSERT INTO orders (user_id, amount, order_date)
    VALUES (1, 150.00, CURRENT_DATE)
    RETURNING *
)
INSERT INTO order_history (order_id, status, changed_at)
SELECT id, 'created', NOW()
FROM new_order;
```

### CTE с UPDATE

```sql
WITH updated_products AS (
    UPDATE products
    SET price = price * 1.1
    WHERE category = 'electronics'
    RETURNING id, name, price
)
INSERT INTO price_changes (product_id, new_price, changed_at)
SELECT id, price, NOW()
FROM updated_products;
```

### CTE с DELETE

```sql
WITH deleted_orders AS (
    DELETE FROM orders
    WHERE order_date < '2020-01-01'
    RETURNING *
)
INSERT INTO archived_orders
SELECT * FROM deleted_orders;
```

### Цепочка модификаций

```sql
WITH
-- Удаляем старые записи
deleted AS (
    DELETE FROM events
    WHERE created_at < NOW() - INTERVAL '1 year'
    RETURNING *
),
-- Архивируем удаленные
archived AS (
    INSERT INTO events_archive
    SELECT * FROM deleted
    RETURNING id
)
-- Возвращаем количество
SELECT COUNT(*) AS archived_count FROM archived;
```

---

## Рекурсивные CTE

### Синтаксис рекурсивного CTE

```sql
WITH RECURSIVE cte_name AS (
    -- Базовый случай (anchor member)
    SELECT ...

    UNION [ALL]

    -- Рекурсивная часть (recursive member)
    SELECT ...
    FROM cte_name
    WHERE ...  -- условие завершения
)
SELECT * FROM cte_name;
```

### Пример: Числовая последовательность

```sql
WITH RECURSIVE numbers AS (
    -- Базовый случай: начинаем с 1
    SELECT 1 AS n

    UNION ALL

    -- Рекурсивная часть: увеличиваем на 1
    SELECT n + 1
    FROM numbers
    WHERE n < 10  -- условие завершения
)
SELECT * FROM numbers;

-- Результат: 1, 2, 3, 4, 5, 6, 7, 8, 9, 10
```

### Пример: Иерархия сотрудников

```sql
-- Таблица сотрудников
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    manager_id INTEGER REFERENCES employees(id)
);

-- Иерархия снизу вверх (от сотрудника к CEO)
WITH RECURSIVE management_chain AS (
    -- Базовый случай: начинаем с конкретного сотрудника
    SELECT id, name, manager_id, 1 AS level
    FROM employees
    WHERE id = 10  -- ID сотрудника

    UNION ALL

    -- Рекурсивно: находим менеджера
    SELECT e.id, e.name, e.manager_id, mc.level + 1
    FROM employees e
    JOIN management_chain mc ON e.id = mc.manager_id
)
SELECT * FROM management_chain;

-- Иерархия сверху вниз (все подчиненные)
WITH RECURSIVE subordinates AS (
    -- Базовый случай: начинаем с менеджера
    SELECT id, name, manager_id, 0 AS level, ARRAY[name] AS path
    FROM employees
    WHERE id = 1  -- ID менеджера

    UNION ALL

    -- Рекурсивно: находим подчиненных
    SELECT e.id, e.name, e.manager_id, s.level + 1, s.path || e.name
    FROM employees e
    JOIN subordinates s ON e.manager_id = s.id
)
SELECT
    REPEAT('  ', level) || name AS tree,
    level,
    array_to_string(path, ' -> ') AS full_path
FROM subordinates
ORDER BY path;
```

### Пример: Категории с вложенностью

```sql
WITH RECURSIVE category_tree AS (
    -- Корневые категории
    SELECT id, name, parent_id, 0 AS depth, name::TEXT AS path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- Подкатегории
    SELECT c.id, c.name, c.parent_id, ct.depth + 1, ct.path || ' > ' || c.name
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY path;
```

### Пример: Граф связей

```sql
-- Поиск всех связанных узлов
WITH RECURSIVE connected AS (
    -- Начальный узел
    SELECT id, ARRAY[id] AS visited
    FROM nodes
    WHERE id = 1

    UNION

    -- Связанные узлы (без циклов)
    SELECT e.to_node, c.visited || e.to_node
    FROM edges e
    JOIN connected c ON e.from_node = c.id
    WHERE NOT e.to_node = ANY(c.visited)
)
SELECT DISTINCT id FROM connected;
```

### Пример: Fibonacci

```sql
WITH RECURSIVE fib AS (
    SELECT 0 AS n, 0::BIGINT AS fib_n, 1::BIGINT AS fib_n1

    UNION ALL

    SELECT n + 1, fib_n1, fib_n + fib_n1
    FROM fib
    WHERE n < 50
)
SELECT n, fib_n AS fibonacci FROM fib;
```

---

## MATERIALIZED и NOT MATERIALIZED

### Материализация CTE

По умолчанию CTE материализуется (результат вычисляется один раз и сохраняется):

```sql
-- PostgreSQL 12+ : явное управление
WITH user_stats AS MATERIALIZED (
    SELECT user_id, COUNT(*) AS cnt
    FROM orders
    GROUP BY user_id
)
SELECT * FROM user_stats WHERE cnt > 10;
```

### Встраивание CTE

Указание `NOT MATERIALIZED` позволяет оптимизатору встроить CTE в основной запрос:

```sql
WITH user_stats AS NOT MATERIALIZED (
    SELECT user_id, COUNT(*) AS cnt
    FROM orders
    GROUP BY user_id
)
SELECT * FROM user_stats WHERE cnt > 10;
-- Оптимизатор может переписать как подзапрос
```

### Когда что использовать

| Ситуация | Рекомендация |
|----------|--------------|
| CTE используется несколько раз | MATERIALIZED |
| CTE с SELECT FROM одной таблицы | NOT MATERIALIZED |
| Рекурсивный CTE | Всегда материализуется |
| Нужен барьер оптимизации | MATERIALIZED |

---

## Практические примеры

### Отчет с промежуточными вычислениями

```sql
WITH
-- Агрегация по месяцам
monthly_sales AS (
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        SUM(amount) AS total
    FROM orders
    WHERE order_date >= '2024-01-01'
    GROUP BY DATE_TRUNC('month', order_date)
),
-- Накопительный итог
running_totals AS (
    SELECT
        month,
        total,
        SUM(total) OVER (ORDER BY month) AS running_total
    FROM monthly_sales
),
-- Изменение к предыдущему месяцу
with_changes AS (
    SELECT
        month,
        total,
        running_total,
        total - LAG(total) OVER (ORDER BY month) AS change,
        ROUND(
            (total - LAG(total) OVER (ORDER BY month)) * 100.0 /
            NULLIF(LAG(total) OVER (ORDER BY month), 0),
            2
        ) AS change_pct
    FROM running_totals
)
SELECT * FROM with_changes ORDER BY month;
```

### Удаление дубликатов

```sql
WITH duplicates AS (
    SELECT id, ROW_NUMBER() OVER (
        PARTITION BY email
        ORDER BY created_at DESC
    ) AS rn
    FROM users
)
DELETE FROM users
WHERE id IN (
    SELECT id FROM duplicates WHERE rn > 1
);
```

### Обновление с использованием оконных функций

```sql
WITH ranked AS (
    SELECT
        id,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) AS rank
    FROM products
)
UPDATE products p
SET is_bestseller = (r.rank = 1)
FROM ranked r
WHERE p.id = r.id;
```

### Пагинация с общим количеством

```sql
WITH filtered AS (
    SELECT *
    FROM products
    WHERE category = 'electronics'
),
counted AS (
    SELECT COUNT(*) AS total FROM filtered
)
SELECT
    f.*,
    c.total
FROM filtered f, counted c
ORDER BY f.price
LIMIT 10 OFFSET 20;
```

---

## CTE vs Подзапросы vs Временные таблицы

### Сравнение

| Характеристика | CTE | Подзапрос | Временная таблица |
|---------------|-----|-----------|-------------------|
| Читаемость | Высокая | Низкая | Средняя |
| Повторное использование | В одном запросе | Нет | В сессии |
| Рекурсия | Да | Нет | Нет |
| Индексирование | Нет | Нет | Да |
| Материализация | Автоматическая | Зависит | Всегда |

### Когда использовать CTE

```sql
-- 1. Сложные запросы с логическими шагами
-- 2. Рекурсивные операции
-- 3. Модифицирующие CTE (INSERT/UPDATE/DELETE)
-- 4. Когда подзапрос используется несколько раз

-- Пример: один подзапрос используется дважды
WITH avg_price AS (
    SELECT AVG(price) AS avg FROM products
)
SELECT
    (SELECT avg FROM avg_price) AS average,
    COUNT(*) FILTER (WHERE price > (SELECT avg FROM avg_price)) AS above_avg
FROM products;
```

### Когда использовать временную таблицу

```sql
-- Для больших промежуточных результатов с индексами
CREATE TEMP TABLE temp_stats AS
SELECT user_id, SUM(amount) AS total
FROM orders
GROUP BY user_id;

CREATE INDEX idx_temp_stats_total ON temp_stats(total);

-- Использование в нескольких запросах
SELECT * FROM temp_stats WHERE total > 1000;
SELECT AVG(total) FROM temp_stats;

-- Очистка
DROP TABLE temp_stats;
```

---

## Типичные ошибки

### 1. Забыть RECURSIVE для рекурсии

```sql
-- Ошибка: нужно WITH RECURSIVE
WITH tree AS (
    SELECT id FROM nodes WHERE id = 1
    UNION ALL
    SELECT n.id FROM nodes n JOIN tree t ON n.parent_id = t.id
)
SELECT * FROM tree;
-- ERROR: recursive reference to query "tree" must not appear within its non-recursive term

-- Правильно
WITH RECURSIVE tree AS (...)
```

### 2. Бесконечная рекурсия

```sql
-- Опасно: нет условия выхода!
WITH RECURSIVE nums AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM nums  -- бесконечный цикл!
)
SELECT * FROM nums;

-- Решение: добавить WHERE
WITH RECURSIVE nums AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM nums WHERE n < 100
)
SELECT * FROM nums;

-- Или LIMIT в основном запросе
WITH RECURSIVE nums AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM nums
)
SELECT * FROM nums LIMIT 100;
```

### 3. Циклы в графе

```sql
-- Бесконечный цикл при наличии циклов в данных
WITH RECURSIVE paths AS (
    SELECT id, ARRAY[id] AS path
    FROM nodes WHERE id = 1
    UNION ALL
    SELECT n.id, p.path || n.id
    FROM nodes n
    JOIN paths p ON n.parent_id = p.id
    -- Нет проверки на цикл!
)
SELECT * FROM paths;

-- Решение: отслеживать посещенные узлы
WITH RECURSIVE paths AS (
    SELECT id, ARRAY[id] AS path
    FROM nodes WHERE id = 1
    UNION ALL
    SELECT n.id, p.path || n.id
    FROM nodes n
    JOIN paths p ON n.parent_id = p.id
    WHERE NOT n.id = ANY(p.path)  -- предотвращение циклов
)
SELECT * FROM paths;
```

### 4. UNION вместо UNION ALL в рекурсии

```sql
-- UNION удаляет дубликаты (медленнее)
WITH RECURSIVE tree AS (
    SELECT id FROM nodes WHERE id = 1
    UNION  -- удаление дубликатов на каждом шаге
    SELECT n.id FROM nodes n JOIN tree t ON n.parent_id = t.id
)
SELECT * FROM tree;

-- UNION ALL быстрее (если дубликаты не ожидаются)
WITH RECURSIVE tree AS (
    SELECT id FROM nodes WHERE id = 1
    UNION ALL
    SELECT n.id FROM nodes n JOIN tree t ON n.parent_id = t.id
    WHERE NOT n.id = ANY(...)  -- явный контроль дубликатов
)
SELECT * FROM tree;
```

---

## Лучшие практики

1. **Давайте CTE понятные имена** — `user_orders`, а не `t1`
2. **Используйте CTE для логического разделения** сложных запросов
3. **Не злоупотребляйте CTE** — простые подзапросы могут быть эффективнее
4. **В рекурсивных CTE всегда предусматривайте выход** из рекурсии
5. **Используйте MATERIALIZED/NOT MATERIALIZED** осознанно в PostgreSQL 12+

---

[prev: 07-advanced-topics](./07-advanced-topics.md) | [next: 09-lateral-join](./09-lateral-join.md)
