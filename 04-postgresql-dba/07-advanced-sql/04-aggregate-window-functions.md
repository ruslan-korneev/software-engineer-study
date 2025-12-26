# Агрегатные и Оконные Функции в PostgreSQL

## Введение

Агрегатные функции вычисляют одно значение на основе набора строк (например, сумму или среднее). Оконные (window) функции выполняют вычисления по набору строк, связанных с текущей строкой, но в отличие от агрегатных функций, не группируют строки — каждая строка сохраняется в результате.

## Агрегатные функции

### Основные агрегатные функции

| Функция | Описание |
|---------|----------|
| COUNT(*) | Количество строк |
| COUNT(column) | Количество не-NULL значений |
| COUNT(DISTINCT column) | Количество уникальных значений |
| SUM(column) | Сумма |
| AVG(column) | Среднее арифметическое |
| MIN(column) | Минимальное значение |
| MAX(column) | Максимальное значение |
| ARRAY_AGG(column) | Собрать значения в массив |
| STRING_AGG(column, delimiter) | Объединить строки |
| BOOL_AND(column) | Логическое И для всех значений |
| BOOL_OR(column) | Логическое ИЛИ для всех значений |

### Примеры агрегатных функций

```sql
-- Тестовая таблица
CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    product VARCHAR(50),
    category VARCHAR(50),
    amount NUMERIC,
    quantity INTEGER,
    sale_date DATE,
    salesperson VARCHAR(50)
);

-- Базовые агрегаты
SELECT
    COUNT(*) AS total_sales,
    COUNT(DISTINCT product) AS unique_products,
    SUM(amount) AS total_revenue,
    AVG(amount) AS avg_sale,
    MIN(amount) AS min_sale,
    MAX(amount) AS max_sale
FROM sales;

-- Группировка
SELECT
    category,
    COUNT(*) AS sales_count,
    SUM(amount) AS total,
    ROUND(AVG(amount), 2) AS average
FROM sales
GROUP BY category
ORDER BY total DESC;

-- STRING_AGG — объединение строк
SELECT
    category,
    STRING_AGG(DISTINCT product, ', ' ORDER BY product) AS products
FROM sales
GROUP BY category;

-- ARRAY_AGG — создание массива
SELECT
    salesperson,
    ARRAY_AGG(DISTINCT category ORDER BY category) AS categories_sold
FROM sales
GROUP BY salesperson;
```

### Агрегаты с FILTER

```sql
-- Условные агрегаты (PostgreSQL 9.4+)
SELECT
    salesperson,
    COUNT(*) AS total_sales,
    COUNT(*) FILTER (WHERE amount > 1000) AS big_sales,
    SUM(amount) FILTER (WHERE category = 'Electronics') AS electronics_revenue,
    AVG(amount) FILTER (WHERE sale_date >= CURRENT_DATE - INTERVAL '30 days') AS recent_avg
FROM sales
GROUP BY salesperson;

-- Эквивалент без FILTER (старый стиль)
SELECT
    salesperson,
    SUM(CASE WHEN amount > 1000 THEN 1 ELSE 0 END) AS big_sales,
    SUM(CASE WHEN category = 'Electronics' THEN amount ELSE 0 END) AS electronics_revenue
FROM sales
GROUP BY salesperson;
```

### Статистические агрегаты

```sql
SELECT
    category,
    -- Статистические функции
    STDDEV(amount) AS std_deviation,           -- Стандартное отклонение
    VARIANCE(amount) AS variance,              -- Дисперсия
    STDDEV_POP(amount) AS population_stddev,   -- Стандартное отклонение (генеральная совокупность)
    STDDEV_SAMP(amount) AS sample_stddev,      -- Стандартное отклонение (выборка)

    -- Корреляция и регрессия
    CORR(amount, quantity) AS correlation,     -- Корреляция
    COVAR_POP(amount, quantity) AS covariance, -- Ковариация

    -- Процентили
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY amount) AS median,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY amount) AS p95,
    PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY amount) AS median_discrete,

    -- Режим (мода)
    MODE() WITHIN GROUP (ORDER BY amount) AS mode_value
FROM sales
GROUP BY category;
```

### GROUPING SETS, CUBE, ROLLUP

```sql
-- GROUPING SETS — несколько уровней группировки в одном запросе
SELECT
    category,
    salesperson,
    SUM(amount) AS total
FROM sales
GROUP BY GROUPING SETS (
    (category, salesperson),  -- По категории и продавцу
    (category),               -- Только по категории
    (salesperson),            -- Только по продавцу
    ()                        -- Общий итог
)
ORDER BY category NULLS LAST, salesperson NULLS LAST;

-- ROLLUP — иерархические итоги
SELECT
    category,
    product,
    SUM(amount) AS total
FROM sales
GROUP BY ROLLUP (category, product)
ORDER BY category NULLS LAST, product NULLS LAST;
-- Результат: category+product, category, grand total

-- CUBE — все комбинации
SELECT
    category,
    salesperson,
    SUM(amount) AS total
FROM sales
GROUP BY CUBE (category, salesperson)
ORDER BY category NULLS LAST, salesperson NULLS LAST;
-- Результат: все возможные комбинации группировок

-- GROUPING() — определить итоговые строки
SELECT
    CASE WHEN GROUPING(category) = 1 THEN 'Все категории' ELSE category END AS category,
    CASE WHEN GROUPING(salesperson) = 1 THEN 'Все продавцы' ELSE salesperson END AS salesperson,
    SUM(amount) AS total,
    GROUPING(category, salesperson) AS grouping_level
FROM sales
GROUP BY ROLLUP (category, salesperson);
```

## Оконные функции

### Синтаксис оконных функций

```sql
function_name(args) OVER (
    [PARTITION BY partition_expression]
    [ORDER BY sort_expression [ASC|DESC] [NULLS {FIRST|LAST}]]
    [frame_clause]
)
```

### Ранжирующие функции

```sql
SELECT
    salesperson,
    category,
    amount,

    -- ROW_NUMBER: уникальный номер строки
    ROW_NUMBER() OVER (ORDER BY amount DESC) AS row_num,

    -- RANK: ранг с пропусками при одинаковых значениях
    RANK() OVER (ORDER BY amount DESC) AS rank,

    -- DENSE_RANK: ранг без пропусков
    DENSE_RANK() OVER (ORDER BY amount DESC) AS dense_rank,

    -- NTILE: разбиение на N равных групп
    NTILE(4) OVER (ORDER BY amount DESC) AS quartile,

    -- PERCENT_RANK: процентный ранг (0-1)
    ROUND(PERCENT_RANK() OVER (ORDER BY amount)::NUMERIC, 2) AS pct_rank,

    -- CUME_DIST: кумулятивное распределение
    ROUND(CUME_DIST() OVER (ORDER BY amount)::NUMERIC, 2) AS cume_dist

FROM sales;

-- Ранжирование внутри групп
SELECT
    category,
    product,
    amount,
    ROW_NUMBER() OVER (PARTITION BY category ORDER BY amount DESC) AS rank_in_category,
    RANK() OVER (PARTITION BY category ORDER BY amount DESC) AS rank_with_ties
FROM sales;

-- Топ-3 продажи в каждой категории
SELECT * FROM (
    SELECT
        category,
        product,
        amount,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY amount DESC) AS rn
    FROM sales
) ranked
WHERE rn <= 3;
```

### Агрегатные оконные функции

```sql
SELECT
    sale_date,
    category,
    amount,

    -- Суммы
    SUM(amount) OVER () AS grand_total,
    SUM(amount) OVER (PARTITION BY category) AS category_total,
    SUM(amount) OVER (ORDER BY sale_date) AS running_total,
    SUM(amount) OVER (PARTITION BY category ORDER BY sale_date) AS category_running_total,

    -- Средние
    AVG(amount) OVER (PARTITION BY category) AS category_avg,

    -- Подсчёт
    COUNT(*) OVER (PARTITION BY category) AS sales_in_category,

    -- Min/Max
    MIN(amount) OVER (PARTITION BY category) AS category_min,
    MAX(amount) OVER (PARTITION BY category) AS category_max

FROM sales
ORDER BY category, sale_date;

-- Процент от общего
SELECT
    category,
    product,
    amount,
    ROUND(amount / SUM(amount) OVER () * 100, 2) AS pct_of_total,
    ROUND(amount / SUM(amount) OVER (PARTITION BY category) * 100, 2) AS pct_of_category
FROM sales;

-- Отклонение от среднего
SELECT
    product,
    amount,
    AVG(amount) OVER () AS overall_avg,
    amount - AVG(amount) OVER () AS deviation,
    ROUND((amount / AVG(amount) OVER () - 1) * 100, 2) AS pct_deviation
FROM sales;
```

### Функции навигации

```sql
SELECT
    sale_date,
    product,
    amount,

    -- LAG: предыдущее значение
    LAG(amount) OVER (ORDER BY sale_date) AS prev_amount,
    LAG(amount, 2) OVER (ORDER BY sale_date) AS prev_2_amount,
    LAG(amount, 1, 0) OVER (ORDER BY sale_date) AS prev_or_zero,

    -- LEAD: следующее значение
    LEAD(amount) OVER (ORDER BY sale_date) AS next_amount,
    LEAD(amount, 2) OVER (ORDER BY sale_date) AS next_2_amount,

    -- FIRST_VALUE: первое значение в окне
    FIRST_VALUE(amount) OVER (ORDER BY sale_date) AS first_sale,

    -- LAST_VALUE: последнее значение в окне
    LAST_VALUE(amount) OVER (
        ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_sale,

    -- NTH_VALUE: N-ное значение
    NTH_VALUE(amount, 3) OVER (ORDER BY sale_date) AS third_sale

FROM sales;

-- Изменение по сравнению с предыдущим
SELECT
    sale_date,
    amount,
    LAG(amount) OVER (ORDER BY sale_date) AS prev_amount,
    amount - LAG(amount) OVER (ORDER BY sale_date) AS change,
    ROUND(
        (amount - LAG(amount) OVER (ORDER BY sale_date)) /
        NULLIF(LAG(amount) OVER (ORDER BY sale_date), 0) * 100,
    2) AS pct_change
FROM sales;

-- Навигация внутри групп
SELECT
    category,
    sale_date,
    amount,
    LAG(amount) OVER (PARTITION BY category ORDER BY sale_date) AS prev_in_category,
    FIRST_VALUE(amount) OVER (PARTITION BY category ORDER BY sale_date) AS first_in_category
FROM sales;
```

### Frame Clause (Рамка окна)

```sql
-- Синтаксис рамки
{ROWS | RANGE | GROUPS}
BETWEEN frame_start AND frame_end

-- frame_start / frame_end:
-- UNBOUNDED PRECEDING - от начала партиции
-- n PRECEDING - n строк/значений назад
-- CURRENT ROW - текущая строка
-- n FOLLOWING - n строк/значений вперёд
-- UNBOUNDED FOLLOWING - до конца партиции

SELECT
    sale_date,
    amount,

    -- По умолчанию: RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    SUM(amount) OVER (ORDER BY sale_date) AS default_running,

    -- Явная рамка ROWS
    SUM(amount) OVER (
        ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS rows_running,

    -- Скользящее среднее за 3 строки
    AVG(amount) OVER (
        ORDER BY sale_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3,

    -- Скользящее среднее 7 дней (по значению)
    AVG(amount) OVER (
        ORDER BY sale_date
        RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW
    ) AS moving_avg_7days,

    -- Центрированное скользящее среднее
    AVG(amount) OVER (
        ORDER BY sale_date
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS centered_avg

FROM sales
ORDER BY sale_date;

-- Пример с GROUPS (PostgreSQL 11+)
SELECT
    sale_date,
    amount,
    SUM(amount) OVER (
        ORDER BY sale_date
        GROUPS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS groups_sum
FROM sales;
```

### EXCLUDE в рамке окна (PostgreSQL 11+)

```sql
SELECT
    amount,
    -- Сумма без текущей строки
    SUM(amount) OVER (
        ORDER BY amount
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        EXCLUDE CURRENT ROW
    ) AS sum_excluding_current,

    -- EXCLUDE GROUP - исключить текущую группу (одинаковые значения)
    -- EXCLUDE TIES - исключить похожие, но оставить текущую
    -- EXCLUDE NO OTHERS - ничего не исключать (по умолчанию)

    SUM(amount) OVER (
        ORDER BY amount
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        EXCLUDE TIES
    ) AS sum_excluding_ties
FROM sales;
```

### Именованные окна (WINDOW clause)

```sql
SELECT
    category,
    sale_date,
    amount,

    -- Используем именованные окна
    SUM(amount) OVER category_window AS category_total,
    AVG(amount) OVER category_window AS category_avg,
    ROW_NUMBER() OVER date_window AS date_order,
    LAG(amount) OVER date_window AS prev_amount

FROM sales
WINDOW
    category_window AS (PARTITION BY category),
    date_window AS (ORDER BY sale_date);

-- Наследование окон
SELECT
    category,
    sale_date,
    amount,
    SUM(amount) OVER base_window AS running_total,
    AVG(amount) OVER (base_window ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg
FROM sales
WINDOW base_window AS (PARTITION BY category ORDER BY sale_date);
```

## Практические примеры

### Анализ продаж

```sql
-- Сравнение с прошлым периодом
WITH monthly_sales AS (
    SELECT
        DATE_TRUNC('month', sale_date) AS month,
        SUM(amount) AS total
    FROM sales
    GROUP BY DATE_TRUNC('month', sale_date)
)
SELECT
    month,
    total,
    LAG(total) OVER (ORDER BY month) AS prev_month,
    total - LAG(total) OVER (ORDER BY month) AS mom_change,
    ROUND(
        (total - LAG(total) OVER (ORDER BY month)) /
        NULLIF(LAG(total) OVER (ORDER BY month), 0) * 100,
    2) AS mom_pct_change,
    SUM(total) OVER (ORDER BY month) AS ytd_total
FROM monthly_sales;
```

### Расчёт бегущего среднего

```sql
SELECT
    sale_date,
    amount,
    AVG(amount) OVER (
        ORDER BY sale_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7,
    AVG(amount) OVER (
        ORDER BY sale_date
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS moving_avg_30
FROM sales
ORDER BY sale_date;
```

### ABC-анализ

```sql
WITH product_revenue AS (
    SELECT
        product,
        SUM(amount) AS total_revenue
    FROM sales
    GROUP BY product
),
ranked AS (
    SELECT
        product,
        total_revenue,
        SUM(total_revenue) OVER (ORDER BY total_revenue DESC) AS cumulative_revenue,
        SUM(total_revenue) OVER () AS grand_total
    FROM product_revenue
)
SELECT
    product,
    total_revenue,
    ROUND(cumulative_revenue / grand_total * 100, 2) AS cumulative_pct,
    CASE
        WHEN cumulative_revenue / grand_total <= 0.80 THEN 'A'
        WHEN cumulative_revenue / grand_total <= 0.95 THEN 'B'
        ELSE 'C'
    END AS abc_category
FROM ranked
ORDER BY total_revenue DESC;
```

### Когортный анализ

```sql
WITH user_cohorts AS (
    SELECT
        user_id,
        DATE_TRUNC('month', MIN(order_date)) AS cohort_month
    FROM orders
    GROUP BY user_id
),
cohort_activity AS (
    SELECT
        uc.cohort_month,
        DATE_TRUNC('month', o.order_date) AS activity_month,
        COUNT(DISTINCT o.user_id) AS active_users
    FROM orders o
    JOIN user_cohorts uc ON o.user_id = uc.user_id
    GROUP BY uc.cohort_month, DATE_TRUNC('month', o.order_date)
)
SELECT
    cohort_month,
    activity_month,
    active_users,
    FIRST_VALUE(active_users) OVER (
        PARTITION BY cohort_month
        ORDER BY activity_month
    ) AS initial_users,
    ROUND(
        active_users::NUMERIC /
        FIRST_VALUE(active_users) OVER (
            PARTITION BY cohort_month
            ORDER BY activity_month
        ) * 100,
    2) AS retention_rate
FROM cohort_activity
ORDER BY cohort_month, activity_month;
```

### Обнаружение аномалий

```sql
WITH stats AS (
    SELECT
        sale_date,
        amount,
        AVG(amount) OVER () AS mean,
        STDDEV(amount) OVER () AS stddev
    FROM sales
)
SELECT
    sale_date,
    amount,
    mean,
    stddev,
    (amount - mean) / NULLIF(stddev, 0) AS z_score,
    CASE
        WHEN ABS((amount - mean) / NULLIF(stddev, 0)) > 3 THEN 'Anomaly'
        WHEN ABS((amount - mean) / NULLIF(stddev, 0)) > 2 THEN 'Suspicious'
        ELSE 'Normal'
    END AS status
FROM stats
ORDER BY ABS((amount - mean) / NULLIF(stddev, 0)) DESC;
```

### Gaps and Islands

```sql
-- Поиск последовательных групп
WITH numbered AS (
    SELECT
        sale_date,
        status,
        ROW_NUMBER() OVER (ORDER BY sale_date) -
        ROW_NUMBER() OVER (PARTITION BY status ORDER BY sale_date) AS grp
    FROM daily_status
)
SELECT
    status,
    MIN(sale_date) AS period_start,
    MAX(sale_date) AS period_end,
    COUNT(*) AS days_count
FROM numbered
GROUP BY status, grp
ORDER BY period_start;
```

### Дедупликация с сохранением последней записи

```sql
-- Удаление дубликатов, оставляя последнюю запись
DELETE FROM sales
WHERE id IN (
    SELECT id FROM (
        SELECT
            id,
            ROW_NUMBER() OVER (
                PARTITION BY product, sale_date
                ORDER BY id DESC
            ) AS rn
        FROM sales
    ) ranked
    WHERE rn > 1
);

-- Выборка без дубликатов
SELECT * FROM (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY product, sale_date
            ORDER BY id DESC
        ) AS rn
    FROM sales
) t
WHERE rn = 1;
```

## Best Practices

### 1. Используйте именованные окна

```sql
-- Хорошо: именованные окна для читаемости
SELECT
    SUM(amount) OVER w AS running_total,
    AVG(amount) OVER w AS running_avg
FROM sales
WINDOW w AS (PARTITION BY category ORDER BY sale_date);

-- Плохо: дублирование определения окна
SELECT
    SUM(amount) OVER (PARTITION BY category ORDER BY sale_date),
    AVG(amount) OVER (PARTITION BY category ORDER BY sale_date)
FROM sales;
```

### 2. Правильный выбор ROWS vs RANGE

```sql
-- ROWS: фиксированное число строк (быстрее)
AVG(amount) OVER (ORDER BY id ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)

-- RANGE: по значению (для дат с пропусками)
AVG(amount) OVER (ORDER BY sale_date RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW)
```

### 3. Оптимизация производительности

```sql
-- Создавайте индексы для PARTITION BY и ORDER BY
CREATE INDEX idx_sales_category_date ON sales(category, sale_date);

-- Избегайте DISTINCT внутри оконных функций где возможно
-- Используйте подзапросы для предварительной агрегации
```

### 4. Обработка NULL

```sql
-- Явно обрабатывайте NULL значения
COALESCE(LAG(amount) OVER (ORDER BY sale_date), 0) AS prev_or_zero

-- Используйте NULLS FIRST/LAST в ORDER BY
ORDER BY sale_date NULLS LAST
```

## Типичные ошибки

### 1. LAST_VALUE без правильной рамки

```sql
-- Неправильно: вернёт текущую строку
LAST_VALUE(amount) OVER (ORDER BY sale_date)

-- Правильно: указать рамку до конца
LAST_VALUE(amount) OVER (
    ORDER BY sale_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

### 2. Неожиданное поведение RANGE

```sql
-- RANGE с одинаковыми значениями включает все одинаковые
SELECT amount, SUM(amount) OVER (ORDER BY amount RANGE UNBOUNDED PRECEDING)
-- Для одинаковых amount сумма будет одинаковой!

-- Используйте ROWS для построчной обработки
SELECT amount, SUM(amount) OVER (ORDER BY amount ROWS UNBOUNDED PRECEDING)
```

### 3. GROUP BY с оконными функциями

```sql
-- Ошибка: нельзя использовать оконные функции в WHERE
SELECT * FROM sales WHERE ROW_NUMBER() OVER (ORDER BY amount) <= 10;

-- Правильно: подзапрос
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (ORDER BY amount) AS rn FROM sales
) t WHERE rn <= 10;
```

## Заключение

Агрегатные и оконные функции — мощнейший инструмент аналитических запросов в PostgreSQL. Агрегатные функции сворачивают данные в итоги, оконные — вычисляют значения в контексте связанных строк без потери детализации. Ключевые концепции: PARTITION BY для группировки, ORDER BY для сортировки внутри окна, и Frame Clause для определения границ расчёта. Эти инструменты незаменимы для ранжирования, расчёта бегущих итогов, анализа временных рядов и сложных аналитических отчётов.
