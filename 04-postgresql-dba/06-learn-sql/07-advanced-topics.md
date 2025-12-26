# Продвинутые темы SQL

## Оконные функции (Window Functions)

Оконные функции выполняют вычисления над набором строк, связанных с текущей строкой, без группировки результата.

### Синтаксис

```sql
function_name() OVER (
    [PARTITION BY column1, column2, ...]
    [ORDER BY column3, column4, ...]
    [frame_clause]
)
```

### Ранжирующие функции

```sql
SELECT
    name,
    department,
    salary,
    -- Ранг с пропусками при равных значениях
    RANK() OVER (ORDER BY salary DESC) AS rank,
    -- Ранг без пропусков
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank,
    -- Номер строки
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num,
    -- Ранг внутри отдела
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees;

-- Результат:
-- name    | department | salary | rank | dense_rank | row_num | dept_rank
-- --------|------------|--------|------|------------|---------|----------
-- Alice   | IT         | 8000   | 1    | 1          | 1       | 1
-- Bob     | IT         | 8000   | 1    | 1          | 2       | 1
-- Charlie | IT         | 6000   | 3    | 2          | 3       | 2
```

### Агрегатные функции как оконные

```sql
SELECT
    order_date,
    amount,
    -- Сумма по всем строкам
    SUM(amount) OVER () AS total,
    -- Накопительная сумма
    SUM(amount) OVER (ORDER BY order_date) AS running_total,
    -- Скользящее среднее за 3 дня
    AVG(amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg,
    -- Среднее по месяцу
    AVG(amount) OVER (PARTITION BY DATE_TRUNC('month', order_date)) AS monthly_avg
FROM orders;
```

### LAG и LEAD

```sql
SELECT
    order_date,
    amount,
    -- Предыдущее значение
    LAG(amount) OVER (ORDER BY order_date) AS prev_amount,
    -- Следующее значение
    LEAD(amount) OVER (ORDER BY order_date) AS next_amount,
    -- Предыдущее со смещением 2
    LAG(amount, 2) OVER (ORDER BY order_date) AS two_before,
    -- Со значением по умолчанию
    LAG(amount, 1, 0) OVER (ORDER BY order_date) AS prev_or_zero,
    -- Изменение
    amount - LAG(amount) OVER (ORDER BY order_date) AS change
FROM orders;
```

### FIRST_VALUE, LAST_VALUE, NTH_VALUE

```sql
SELECT
    name,
    department,
    salary,
    -- Первое значение в окне
    FIRST_VALUE(name) OVER (
        PARTITION BY department ORDER BY salary DESC
    ) AS top_earner,
    -- Последнее (осторожно с рамкой!)
    LAST_VALUE(name) OVER (
        PARTITION BY department ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS lowest_earner,
    -- N-е значение
    NTH_VALUE(name, 2) OVER (
        PARTITION BY department ORDER BY salary DESC
    ) AS second_earner
FROM employees;
```

### NTILE — разбиение на группы

```sql
-- Разбить сотрудников на 4 квартиля по зарплате
SELECT
    name,
    salary,
    NTILE(4) OVER (ORDER BY salary) AS quartile
FROM employees;

-- Разбить на 3 группы
SELECT
    name,
    score,
    CASE NTILE(3) OVER (ORDER BY score DESC)
        WHEN 1 THEN 'Top'
        WHEN 2 THEN 'Middle'
        WHEN 3 THEN 'Bottom'
    END AS tier
FROM students;
```

### Рамки окна (Frame Clause)

```sql
-- ROWS — физические строки
SELECT
    order_date,
    amount,
    -- 3 предыдущих строки и текущая
    SUM(amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN 3 PRECEDING AND CURRENT ROW
    ) AS sum_last_4,
    -- Все предыдущие и текущая
    SUM(amount) OVER (
        ORDER BY order_date
        ROWS UNBOUNDED PRECEDING
    ) AS running_total,
    -- Все строки
    SUM(amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS total
FROM orders;

-- RANGE — логический диапазон
SELECT
    order_date,
    amount,
    -- Все заказы за последние 7 дней
    SUM(amount) OVER (
        ORDER BY order_date
        RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW
    ) AS week_sum
FROM orders;

-- GROUPS — группы одинаковых значений (PostgreSQL 11+)
SELECT
    category,
    price,
    AVG(price) OVER (
        ORDER BY price
        GROUPS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS avg_nearby
FROM products;
```

---

## CASE Выражения

### Простой CASE

```sql
SELECT
    name,
    status,
    CASE status
        WHEN 'active' THEN 'Активен'
        WHEN 'inactive' THEN 'Неактивен'
        WHEN 'pending' THEN 'Ожидает'
        ELSE 'Неизвестно'
    END AS status_ru
FROM users;
```

### Поисковый CASE

```sql
SELECT
    name,
    price,
    CASE
        WHEN price < 10 THEN 'Дешево'
        WHEN price < 100 THEN 'Средне'
        WHEN price < 1000 THEN 'Дорого'
        ELSE 'Премиум'
    END AS price_category
FROM products;

-- В WHERE
SELECT * FROM products
WHERE CASE
    WHEN category = 'food' THEN price < 50
    WHEN category = 'electronics' THEN price < 500
    ELSE price < 100
END;

-- В ORDER BY
SELECT * FROM tasks
ORDER BY CASE priority
    WHEN 'high' THEN 1
    WHEN 'medium' THEN 2
    WHEN 'low' THEN 3
END;
```

### CASE в агрегации

```sql
-- Подсчет по условию
SELECT
    COUNT(*) AS total,
    COUNT(CASE WHEN status = 'active' THEN 1 END) AS active_count,
    COUNT(CASE WHEN status = 'inactive' THEN 1 END) AS inactive_count,
    SUM(CASE WHEN premium THEN amount ELSE 0 END) AS premium_revenue
FROM users;

-- Pivot-таблица
SELECT
    department,
    SUM(CASE WHEN month = 1 THEN sales ELSE 0 END) AS jan,
    SUM(CASE WHEN month = 2 THEN sales ELSE 0 END) AS feb,
    SUM(CASE WHEN month = 3 THEN sales ELSE 0 END) AS mar
FROM monthly_sales
GROUP BY department;
```

### COALESCE и NULLIF

```sql
-- COALESCE — первое не-NULL значение
SELECT COALESCE(phone, email, 'No contact') AS contact FROM users;

-- NULLIF — NULL если значения равны
SELECT NULLIF(status, 'unknown') AS status FROM data;

-- Избежание деления на ноль
SELECT total / NULLIF(count, 0) AS average FROM stats;
```

---

## Рекурсивные запросы

### Рекурсивный CTE

```sql
-- Иерархия сотрудников
WITH RECURSIVE employee_tree AS (
    -- Базовый случай (корни дерева)
    SELECT id, name, manager_id, 1 AS level, ARRAY[name] AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Рекурсивная часть
    SELECT e.id, e.name, e.manager_id, et.level + 1, et.path || e.name
    FROM employees e
    JOIN employee_tree et ON e.manager_id = et.id
)
SELECT * FROM employee_tree ORDER BY path;
```

### Генерация последовательностей

```sql
-- Числовая последовательность
WITH RECURSIVE nums AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM nums WHERE n < 100
)
SELECT * FROM nums;

-- Даты
WITH RECURSIVE dates AS (
    SELECT '2024-01-01'::DATE AS date
    UNION ALL
    SELECT date + 1 FROM dates WHERE date < '2024-12-31'
)
SELECT * FROM dates;

-- Проще с generate_series
SELECT generate_series(1, 100) AS n;
SELECT generate_series('2024-01-01'::DATE, '2024-12-31', '1 day'::INTERVAL);
```

### Граф и пути

```sql
-- Поиск пути в графе
WITH RECURSIVE paths AS (
    SELECT
        from_node AS start,
        to_node AS current,
        ARRAY[from_node, to_node] AS path,
        weight AS total_weight
    FROM edges
    WHERE from_node = 'A'

    UNION ALL

    SELECT
        p.start,
        e.to_node,
        p.path || e.to_node,
        p.total_weight + e.weight
    FROM paths p
    JOIN edges e ON p.current = e.from_node
    WHERE NOT e.to_node = ANY(p.path)  -- предотвращение циклов
)
SELECT * FROM paths WHERE current = 'Z' ORDER BY total_weight;
```

---

## Полнотекстовый поиск

### Основы

```sql
-- Создание tsvector
SELECT to_tsvector('russian', 'PostgreSQL - это отличная база данных');
-- 'баз':5 'дан':6 'отличн':4 'postgresql':1

-- Создание tsquery
SELECT to_tsquery('russian', 'база & данных');
-- 'баз' & 'дан'

-- Поиск
SELECT * FROM articles
WHERE to_tsvector('russian', content) @@ to_tsquery('russian', 'база & данных');
```

### Индексирование

```sql
-- GIN индекс для полнотекстового поиска
CREATE INDEX idx_articles_fts ON articles
    USING GIN (to_tsvector('russian', title || ' ' || content));

-- Хранимый tsvector
ALTER TABLE articles ADD COLUMN search_vector TSVECTOR;

UPDATE articles
SET search_vector = to_tsvector('russian', title || ' ' || content);

CREATE INDEX idx_articles_search ON articles USING GIN (search_vector);

-- Автообновление через триггер
CREATE FUNCTION update_search_vector() RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector := to_tsvector('russian', NEW.title || ' ' || NEW.content);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER articles_search_update
BEFORE INSERT OR UPDATE ON articles
FOR EACH ROW EXECUTE FUNCTION update_search_vector();
```

### Ранжирование результатов

```sql
SELECT
    title,
    ts_rank(search_vector, query) AS rank,
    ts_rank_cd(search_vector, query) AS rank_cd,
    ts_headline('russian', content, query) AS headline
FROM articles, to_tsquery('russian', 'postgresql') query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

---

## Условная логика и работа с NULL

### GREATEST / LEAST

```sql
SELECT GREATEST(10, 20, 5);  -- 20
SELECT LEAST(10, 20, 5);     -- 5

-- Практическое применение
SELECT
    product_name,
    GREATEST(price, min_price) AS actual_price,
    LEAST(quantity, max_order) AS order_limit
FROM products;
```

### Операции с массивами в WHERE

```sql
-- Проверка пересечения
SELECT * FROM products WHERE tags && ARRAY['sale', 'new'];

-- Проверка включения
SELECT * FROM products WHERE tags @> ARRAY['premium'];

-- Проверка ANY/ALL
SELECT * FROM products WHERE 'sale' = ANY(tags);
SELECT * FROM orders WHERE amount > ALL(ARRAY[100, 200, 300]);
```

---

## Агрегации с фильтрацией

### FILTER

```sql
SELECT
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE status = 'active') AS active,
    COUNT(*) FILTER (WHERE status = 'inactive') AS inactive,
    SUM(amount) FILTER (WHERE order_date >= '2024-01-01') AS ytd_sales,
    AVG(rating) FILTER (WHERE verified = true) AS verified_avg_rating
FROM orders;
```

### Условная агрегация

```sql
-- Без FILTER (старый способ)
SELECT
    SUM(CASE WHEN type = 'income' THEN amount ELSE 0 END) AS total_income,
    SUM(CASE WHEN type = 'expense' THEN amount ELSE 0 END) AS total_expense
FROM transactions;

-- С FILTER (PostgreSQL 9.4+)
SELECT
    SUM(amount) FILTER (WHERE type = 'income') AS total_income,
    SUM(amount) FILTER (WHERE type = 'expense') AS total_expense
FROM transactions;
```

---

## Статистические функции

### Базовые статистические функции

```sql
SELECT
    -- Стандартное отклонение (выборочное)
    STDDEV(price) AS stddev_sample,
    -- Стандартное отклонение (генеральная совокупность)
    STDDEV_POP(price) AS stddev_pop,
    -- Дисперсия
    VARIANCE(price) AS variance,
    VAR_POP(price) AS var_pop,
    -- Корреляция
    CORR(price, quantity) AS correlation,
    -- Ковариация
    COVAR_SAMP(price, quantity) AS covariance,
    -- Линейная регрессия
    REGR_SLOPE(quantity, price) AS slope,
    REGR_INTERCEPT(quantity, price) AS intercept
FROM products;
```

### Процентили и медиана

```sql
-- Percentile
SELECT
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median,
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY salary) AS q1,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY salary) AS q3,
    PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY salary) AS median_disc
FROM employees;

-- Процентильный ранг
SELECT
    name,
    salary,
    PERCENT_RANK() OVER (ORDER BY salary) AS pct_rank,
    CUME_DIST() OVER (ORDER BY salary) AS cumulative_dist
FROM employees;

-- MODE — наиболее частое значение
SELECT MODE() WITHIN GROUP (ORDER BY department) FROM employees;
```

---

## Генерация данных

### generate_series

```sql
-- Числа
SELECT generate_series(1, 10);
SELECT generate_series(1, 10, 2);  -- с шагом

-- Даты
SELECT generate_series(
    '2024-01-01'::DATE,
    '2024-12-31'::DATE,
    '1 month'::INTERVAL
);

-- Заполнение пропусков в датах
SELECT
    d.date,
    COALESCE(o.count, 0) AS order_count
FROM generate_series('2024-01-01'::DATE, '2024-01-31', '1 day') d(date)
LEFT JOIN (
    SELECT order_date, COUNT(*) as count
    FROM orders
    GROUP BY order_date
) o ON d.date = o.order_date;
```

### Генерация тестовых данных

```sql
-- Генерация случайных данных
INSERT INTO users (name, email, created_at)
SELECT
    'User ' || i,
    'user' || i || '@example.com',
    NOW() - (random() * INTERVAL '365 days')
FROM generate_series(1, 1000) AS i;

-- Случайный выбор из набора
SELECT (ARRAY['red', 'green', 'blue'])[1 + floor(random() * 3)];
```

---

## Работа с JSON агрегацией

```sql
-- Агрегация в JSON массив
SELECT json_agg(name) FROM users WHERE department = 'IT';
-- ["Alice", "Bob", "Charlie"]

-- Агрегация в JSON объект
SELECT json_object_agg(id, name) FROM users;
-- {"1": "Alice", "2": "Bob"}

-- Построение JSON объекта
SELECT jsonb_build_object(
    'user_id', id,
    'full_name', name,
    'orders', (
        SELECT jsonb_agg(jsonb_build_object('id', o.id, 'amount', o.amount))
        FROM orders o WHERE o.user_id = users.id
    )
) FROM users;
```

---

## Советы по производительности

### 1. Используйте EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM users WHERE email = 'test@example.com';
```

### 2. Индексируйте оконные функции

```sql
-- Индекс для ORDER BY в окне
CREATE INDEX idx_orders_date ON orders(order_date);

-- Индекс для PARTITION BY
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);
```

### 3. Материализуйте сложные подзапросы

```sql
-- Материализованный CTE
WITH stats AS MATERIALIZED (
    SELECT user_id, SUM(amount) as total
    FROM orders
    GROUP BY user_id
)
SELECT u.name, s.total
FROM users u
JOIN stats s ON u.id = s.user_id;
```
