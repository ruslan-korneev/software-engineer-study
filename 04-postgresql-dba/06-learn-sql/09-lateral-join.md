# LATERAL JOIN

## Что такое LATERAL?

**LATERAL** — это модификатор подзапроса в FROM, который позволяет подзапросу ссылаться на столбцы из предшествующих таблиц в том же FROM. По сути, LATERAL создает коррелированный подзапрос, который выполняется для каждой строки внешней таблицы.

### Синтаксис

```sql
SELECT ...
FROM table1
JOIN LATERAL (
    SELECT ...
    FROM table2
    WHERE table2.column = table1.column  -- ссылка на внешнюю таблицу
) alias ON true;
```

---

## Сравнение с обычным JOIN

### Обычный JOIN

```sql
-- Обычный подзапрос не может ссылаться на внешнюю таблицу
SELECT u.name, o.total
FROM users u
JOIN (
    SELECT user_id, SUM(amount) AS total
    FROM orders
    WHERE user_id = u.id  -- ОШИБКА! u.id недоступен
    GROUP BY user_id
) o ON u.id = o.user_id;
-- ERROR: invalid reference to FROM-clause entry for table "u"
```

### LATERAL JOIN

```sql
-- LATERAL позволяет ссылаться на предыдущие таблицы
SELECT u.name, o.total
FROM users u
JOIN LATERAL (
    SELECT SUM(amount) AS total
    FROM orders
    WHERE user_id = u.id  -- u.id доступен благодаря LATERAL
) o ON true;
```

---

## Основные варианты использования

### 1. Top-N в каждой группе

```sql
-- Топ-3 заказа каждого пользователя
SELECT u.name, o.*
FROM users u
CROSS JOIN LATERAL (
    SELECT id, amount, order_date
    FROM orders
    WHERE user_id = u.id
    ORDER BY amount DESC
    LIMIT 3
) o;

-- Эквивалентно, но с LEFT JOIN (включая пользователей без заказов)
SELECT u.name, o.*
FROM users u
LEFT JOIN LATERAL (
    SELECT id, amount, order_date
    FROM orders
    WHERE user_id = u.id
    ORDER BY amount DESC
    LIMIT 3
) o ON true;
```

### 2. Вызов табличных функций

```sql
-- Функция, возвращающая таблицу
CREATE FUNCTION get_user_orders(uid INTEGER)
RETURNS TABLE (order_id INTEGER, amount NUMERIC) AS $$
    SELECT id, amount FROM orders WHERE user_id = uid
$$ LANGUAGE SQL;

-- Вызов для каждого пользователя
SELECT u.name, o.*
FROM users u
CROSS JOIN LATERAL get_user_orders(u.id) o;

-- LATERAL обязателен, но может быть неявным для функций:
SELECT u.name, o.*
FROM users u, get_user_orders(u.id) o;
-- Эквивалентно CROSS JOIN LATERAL
```

### 3. Развертывание массивов и JSON

```sql
-- Развертывание массива
SELECT u.name, tag
FROM users u
CROSS JOIN LATERAL unnest(u.tags) AS tag;

-- Развертывание JSON массива
SELECT u.name, item->>'product' AS product
FROM users u
CROSS JOIN LATERAL jsonb_array_elements(u.purchase_history) AS item;
```

### 4. Вычисления с промежуточными значениями

```sql
-- Использование вычисленного значения несколько раз
SELECT
    p.name,
    calc.discounted_price,
    calc.discounted_price * 0.2 AS tax
FROM products p
CROSS JOIN LATERAL (
    SELECT p.price * (1 - p.discount_rate) AS discounted_price
) calc;

-- Без LATERAL пришлось бы повторять вычисление:
SELECT
    name,
    price * (1 - discount_rate) AS discounted_price,
    price * (1 - discount_rate) * 0.2 AS tax
FROM products;
```

---

## LATERAL с разными типами JOIN

### CROSS JOIN LATERAL

Эквивалентно запятой — возвращает только пары, где подзапрос дал результат:

```sql
-- Только пользователи с заказами
SELECT u.name, o.amount
FROM users u
CROSS JOIN LATERAL (
    SELECT amount FROM orders WHERE user_id = u.id LIMIT 1
) o;

-- Тот же результат с запятой (неявный LATERAL для функций)
SELECT u.name, o.amount
FROM users u,
LATERAL (SELECT amount FROM orders WHERE user_id = u.id LIMIT 1) o;
```

### LEFT JOIN LATERAL

Сохраняет строки из левой таблицы, даже если подзапрос пуст:

```sql
-- Все пользователи, даже без заказов
SELECT u.name, o.amount
FROM users u
LEFT JOIN LATERAL (
    SELECT amount FROM orders WHERE user_id = u.id LIMIT 1
) o ON true;
-- o.amount будет NULL для пользователей без заказов
```

### INNER JOIN LATERAL

То же, что CROSS JOIN LATERAL, но с явным условием:

```sql
SELECT u.name, o.amount
FROM users u
INNER JOIN LATERAL (
    SELECT amount FROM orders WHERE user_id = u.id
) o ON o.amount > 100;  -- дополнительное условие
```

---

## Практические примеры

### Последняя запись в каждой группе

```sql
-- Последний заказ каждого пользователя (эффективнее чем подзапрос)
SELECT u.name, lo.*
FROM users u
LEFT JOIN LATERAL (
    SELECT id, amount, order_date
    FROM orders
    WHERE user_id = u.id
    ORDER BY order_date DESC
    LIMIT 1
) lo ON true;
```

### Сравнение: LATERAL vs оконные функции

```sql
-- Способ 1: С оконной функцией
SELECT u.name, o.id, o.amount, o.order_date
FROM users u
LEFT JOIN (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_date DESC) AS rn
    FROM orders
) o ON u.id = o.user_id AND o.rn = 1;

-- Способ 2: С LATERAL (часто эффективнее при наличии индекса)
SELECT u.name, o.id, o.amount, o.order_date
FROM users u
LEFT JOIN LATERAL (
    SELECT id, amount, order_date
    FROM orders
    WHERE user_id = u.id
    ORDER BY order_date DESC
    LIMIT 1
) o ON true;
```

### Ближайшие записи

```sql
-- Найти ближайший магазин к каждому пользователю
SELECT u.name, s.store_name, s.distance
FROM users u
CROSS JOIN LATERAL (
    SELECT
        store_name,
        ST_Distance(u.location, s.location) AS distance
    FROM stores s
    ORDER BY u.location <-> s.location  -- оператор расстояния PostGIS
    LIMIT 1
) s;
```

### Агрегация с условием

```sql
-- Статистика заказов за разные периоды для каждого пользователя
SELECT
    u.name,
    stats.orders_30d,
    stats.total_30d,
    stats.orders_all,
    stats.total_all
FROM users u
CROSS JOIN LATERAL (
    SELECT
        COUNT(*) FILTER (WHERE order_date >= NOW() - INTERVAL '30 days') AS orders_30d,
        COALESCE(SUM(amount) FILTER (WHERE order_date >= NOW() - INTERVAL '30 days'), 0) AS total_30d,
        COUNT(*) AS orders_all,
        COALESCE(SUM(amount), 0) AS total_all
    FROM orders
    WHERE user_id = u.id
) stats;
```

### Парсинг JSON

```sql
-- Извлечение данных из JSON массива с доступом к родительским данным
SELECT
    o.id AS order_id,
    o.order_date,
    item->>'product_id' AS product_id,
    (item->>'quantity')::INTEGER AS quantity,
    (item->>'price')::NUMERIC AS price
FROM orders o
CROSS JOIN LATERAL jsonb_array_elements(o.items) AS item;
```

### Генерация строк на основе данных

```sql
-- Развернуть диапазон дат для каждого события
SELECT
    e.name,
    d.event_date
FROM events e
CROSS JOIN LATERAL generate_series(
    e.start_date,
    e.end_date,
    '1 day'::INTERVAL
) AS d(event_date);

-- Сгенерировать записи на основе количества
SELECT p.name, n
FROM products p
CROSS JOIN LATERAL generate_series(1, p.stock_quantity) AS n;
```

---

## Производительность LATERAL

### Когда LATERAL эффективнее

```sql
-- Индекс критически важен!
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date DESC);

-- LATERAL использует индекс для каждого пользователя
EXPLAIN ANALYZE
SELECT u.name, o.*
FROM users u
LEFT JOIN LATERAL (
    SELECT * FROM orders
    WHERE user_id = u.id
    ORDER BY order_date DESC
    LIMIT 3
) o ON true;

-- Покажет Index Scan для каждой итерации
```

### Когда LATERAL менее эффективнее

```sql
-- Если нет подходящего индекса, может быть медленнее чем JOIN
-- Для каждой строки внешней таблицы выполняется подзапрос

-- Альтернатива без LATERAL (может быть быстрее на больших данных):
WITH ranked AS (
    SELECT
        *,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_date DESC) AS rn
    FROM orders
)
SELECT u.name, r.*
FROM users u
LEFT JOIN ranked r ON u.id = r.user_id AND r.rn <= 3;
```

### Советы по оптимизации

1. **Создавайте индексы** на столбцах, используемых в WHERE и ORDER BY внутри LATERAL
2. **Используйте LIMIT** когда нужно ограниченное количество строк
3. **Анализируйте EXPLAIN** чтобы понять план выполнения
4. **Сравнивайте с альтернативами** — оконные функции, CTE, обычные JOIN

---

## LATERAL с функциями

### Табличные функции (Set-Returning Functions)

```sql
-- Функции, возвращающие множество строк, неявно используют LATERAL
SELECT u.name, n
FROM users u, generate_series(1, u.order_count) AS n;

-- Эквивалентно:
SELECT u.name, n
FROM users u
CROSS JOIN LATERAL generate_series(1, u.order_count) AS n;

-- unnest тоже работает с LATERAL
SELECT u.name, tag
FROM users u, unnest(u.tags) AS tag;
```

### Функции с несколькими выходными столбцами

```sql
-- regexp_matches возвращает массив
SELECT u.email, matches[1] AS domain
FROM users u
CROSS JOIN LATERAL regexp_matches(u.email, '@(.+)$') AS matches;

-- string_to_table (PostgreSQL 14+)
SELECT u.name, word
FROM users u
CROSS JOIN LATERAL string_to_table(u.bio, ' ') AS word;
```

---

## Сложные примеры

### Временные ряды с заполнением пропусков

```sql
SELECT
    u.name,
    d.date,
    COALESCE(stats.order_count, 0) AS order_count
FROM users u
CROSS JOIN LATERAL generate_series(
    '2024-01-01'::DATE,
    '2024-01-31'::DATE,
    '1 day'::INTERVAL
) AS d(date)
LEFT JOIN LATERAL (
    SELECT COUNT(*) AS order_count
    FROM orders
    WHERE user_id = u.id
      AND order_date = d.date
) stats ON true;
```

### Рекурсивный поиск с LATERAL

```sql
-- Все пути в графе с ограничением глубины
WITH RECURSIVE paths AS (
    SELECT
        id AS start_id,
        id AS current_id,
        ARRAY[id] AS path,
        0 AS depth
    FROM nodes
    WHERE id = 1

    UNION ALL

    SELECT
        p.start_id,
        e.to_id,
        p.path || e.to_id,
        p.depth + 1
    FROM paths p
    CROSS JOIN LATERAL (
        SELECT to_id
        FROM edges
        WHERE from_id = p.current_id
          AND NOT to_id = ANY(p.path)
    ) e
    WHERE p.depth < 10
)
SELECT * FROM paths;
```

### Поиск по нескольким критериям с приоритетом

```sql
-- Найти товар: сначала по точному совпадению SKU, потом по названию
SELECT DISTINCT ON (search_term) *
FROM (VALUES ('ABC123'), ('Widget'), ('Unknown')) AS searches(search_term)
CROSS JOIN LATERAL (
    -- Точное совпадение SKU (приоритет 1)
    SELECT *, 1 AS priority FROM products WHERE sku = search_term
    UNION ALL
    -- Поиск по названию (приоритет 2)
    SELECT *, 2 FROM products WHERE name ILIKE '%' || search_term || '%'
    UNION ALL
    -- Не найдено (приоритет 3)
    SELECT NULL::products.*, 3
) matches
ORDER BY search_term, priority;
```

---

## Типичные ошибки

### 1. Забыть ON true для LEFT JOIN LATERAL

```sql
-- Ошибка синтаксиса
SELECT u.name, o.amount
FROM users u
LEFT JOIN LATERAL (
    SELECT amount FROM orders WHERE user_id = u.id LIMIT 1
) o;  -- ERROR: нужен ON

-- Правильно
SELECT u.name, o.amount
FROM users u
LEFT JOIN LATERAL (...) o ON true;  -- ON true обязателен
```

### 2. Использовать LATERAL без необходимости

```sql
-- Избыточно: подзапрос не ссылается на внешнюю таблицу
SELECT u.*, s.*
FROM users u
CROSS JOIN LATERAL (
    SELECT AVG(amount) AS avg_amount FROM orders  -- не использует u
) s;

-- Лучше без LATERAL
SELECT u.*, s.*
FROM users u
CROSS JOIN (SELECT AVG(amount) AS avg_amount FROM orders) s;
```

### 3. Не учитывать пустой результат подзапроса

```sql
-- CROSS JOIN LATERAL отфильтрует пользователей без заказов
SELECT u.name, o.amount
FROM users u
CROSS JOIN LATERAL (
    SELECT amount FROM orders WHERE user_id = u.id
) o;
-- Пользователи без заказов не попадут в результат

-- Если нужны все пользователи - используйте LEFT JOIN LATERAL
```

---

## Лучшие практики

1. **Всегда создавайте индексы** для столбцов в WHERE внутри LATERAL
2. **Используйте LIMIT** когда нужно ограниченное количество записей
3. **Выбирайте правильный тип JOIN** — CROSS для фильтрации, LEFT для сохранения всех строк
4. **Сравнивайте с альтернативами** через EXPLAIN ANALYZE
5. **Давайте понятные псевдонимы** подзапросам LATERAL
