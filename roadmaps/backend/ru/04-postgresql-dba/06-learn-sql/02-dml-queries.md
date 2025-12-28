# DML Queries (Data Manipulation Language)

[prev: 01-ddl-queries](./01-ddl-queries.md) | [next: 03-data-types](./03-data-types.md)

---

## Что такое DML?

**DML (Data Manipulation Language)** — это подмножество SQL для работы с данными в таблицах. DML-команды позволяют добавлять, изменять, удалять и извлекать данные.

### Основные DML-команды:
- `SELECT` — выборка данных
- `INSERT` — добавление данных
- `UPDATE` — изменение данных
- `DELETE` — удаление данных

---

## SELECT — Выборка данных

### Базовый синтаксис

```sql
SELECT column1, column2, ...
FROM table_name
WHERE condition
ORDER BY column
LIMIT n OFFSET m;
```

### Примеры SELECT

```sql
-- Выбрать все столбцы
SELECT * FROM users;

-- Выбрать конкретные столбцы
SELECT id, username, email FROM users;

-- Выбрать с псевдонимами
SELECT
    id AS user_id,
    username AS login,
    email AS "E-mail адрес"
FROM users;

-- Выбрать уникальные значения
SELECT DISTINCT country FROM users;

-- Выбрать с вычисляемыми столбцами
SELECT
    product_name,
    price,
    quantity,
    price * quantity AS total
FROM order_items;
```

### WHERE — Условия фильтрации

```sql
-- Сравнение
SELECT * FROM users WHERE age >= 18;
SELECT * FROM users WHERE status = 'active';
SELECT * FROM users WHERE status <> 'deleted';  -- или !=

-- Логические операторы
SELECT * FROM users WHERE age >= 18 AND status = 'active';
SELECT * FROM users WHERE country = 'RU' OR country = 'BY';
SELECT * FROM users WHERE NOT is_banned;

-- BETWEEN — диапазон
SELECT * FROM orders WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31';

-- IN — список значений
SELECT * FROM users WHERE country IN ('RU', 'BY', 'KZ');

-- NOT IN
SELECT * FROM users WHERE status NOT IN ('deleted', 'banned');

-- LIKE — поиск по шаблону
SELECT * FROM users WHERE email LIKE '%@gmail.com';
SELECT * FROM users WHERE username LIKE 'admin%';
SELECT * FROM users WHERE phone LIKE '+7___123____';  -- _ = один символ

-- ILIKE — регистронезависимый поиск (PostgreSQL)
SELECT * FROM users WHERE email ILIKE '%@GMAIL.COM';

-- IS NULL / IS NOT NULL
SELECT * FROM users WHERE deleted_at IS NULL;
SELECT * FROM users WHERE phone IS NOT NULL;

-- Подзапрос в WHERE
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE total > 1000);
```

### ORDER BY — Сортировка

```sql
-- Сортировка по возрастанию (по умолчанию)
SELECT * FROM users ORDER BY created_at;

-- Сортировка по убыванию
SELECT * FROM users ORDER BY created_at DESC;

-- Многоуровневая сортировка
SELECT * FROM users ORDER BY country ASC, created_at DESC;

-- Сортировка по номеру столбца
SELECT id, username, email FROM users ORDER BY 2;

-- Сортировка с NULL
SELECT * FROM users ORDER BY phone NULLS FIRST;
SELECT * FROM users ORDER BY phone NULLS LAST;

-- Сортировка по выражению
SELECT * FROM products ORDER BY price * discount_percent;
```

### LIMIT и OFFSET — Пагинация

```sql
-- Первые 10 записей
SELECT * FROM users LIMIT 10;

-- Пропустить первые 20 и взять 10
SELECT * FROM users LIMIT 10 OFFSET 20;

-- Пагинация: страница 3 по 10 записей
SELECT * FROM users LIMIT 10 OFFSET 20;  -- (page - 1) * limit

-- FETCH (стандарт SQL)
SELECT * FROM users
OFFSET 20 ROWS
FETCH FIRST 10 ROWS ONLY;
```

### Агрегатные функции

```sql
-- COUNT — подсчет
SELECT COUNT(*) FROM users;
SELECT COUNT(DISTINCT country) FROM users;
SELECT COUNT(*) FILTER (WHERE status = 'active') FROM users;

-- SUM — сумма
SELECT SUM(amount) FROM payments;

-- AVG — среднее
SELECT AVG(price) FROM products;

-- MIN / MAX
SELECT MIN(created_at), MAX(created_at) FROM users;

-- STRING_AGG — объединение строк
SELECT STRING_AGG(username, ', ') FROM users WHERE country = 'RU';

-- ARRAY_AGG — объединение в массив
SELECT ARRAY_AGG(tag) FROM article_tags WHERE article_id = 1;
```

### GROUP BY — Группировка

```sql
-- Количество пользователей по странам
SELECT country, COUNT(*) as user_count
FROM users
GROUP BY country;

-- Сумма продаж по месяцам
SELECT
    DATE_TRUNC('month', order_date) AS month,
    SUM(total_amount) AS total_sales
FROM orders
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY month;

-- Группировка по нескольким столбцам
SELECT country, status, COUNT(*)
FROM users
GROUP BY country, status;

-- ROLLUP — иерархические итоги
SELECT country, city, COUNT(*)
FROM users
GROUP BY ROLLUP (country, city);

-- CUBE — все комбинации итогов
SELECT country, status, COUNT(*)
FROM users
GROUP BY CUBE (country, status);

-- GROUPING SETS — произвольные группировки
SELECT country, status, COUNT(*)
FROM users
GROUP BY GROUPING SETS ((country), (status), ());
```

### HAVING — Фильтрация групп

```sql
-- Страны с более чем 100 пользователями
SELECT country, COUNT(*) as cnt
FROM users
GROUP BY country
HAVING COUNT(*) > 100;

-- Товары со средней оценкой выше 4
SELECT product_id, AVG(rating) as avg_rating
FROM reviews
GROUP BY product_id
HAVING AVG(rating) > 4;
```

---

## INSERT — Добавление данных

### Базовый INSERT

```sql
-- Вставка одной записи
INSERT INTO users (username, email, created_at)
VALUES ('john_doe', 'john@example.com', NOW());

-- Вставка с указанием всех столбцов
INSERT INTO users VALUES (1, 'john_doe', 'john@example.com', NOW());

-- Вставка нескольких записей
INSERT INTO users (username, email) VALUES
    ('user1', 'user1@example.com'),
    ('user2', 'user2@example.com'),
    ('user3', 'user3@example.com');
```

### INSERT с RETURNING

```sql
-- Получить ID вставленной записи
INSERT INTO users (username, email)
VALUES ('new_user', 'new@example.com')
RETURNING id;

-- Получить всю вставленную запись
INSERT INTO users (username, email)
VALUES ('new_user', 'new@example.com')
RETURNING *;

-- Получить несколько столбцов
INSERT INTO users (username, email)
VALUES ('new_user', 'new@example.com')
RETURNING id, created_at;
```

### INSERT из SELECT

```sql
-- Копирование данных из другой таблицы
INSERT INTO users_archive (id, username, email)
SELECT id, username, email
FROM users
WHERE deleted_at IS NOT NULL;

-- Вставка с преобразованием
INSERT INTO monthly_stats (month, total_orders, total_amount)
SELECT
    DATE_TRUNC('month', order_date),
    COUNT(*),
    SUM(total_amount)
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY DATE_TRUNC('month', order_date);
```

### INSERT с ON CONFLICT (UPSERT)

```sql
-- Игнорировать конфликт
INSERT INTO users (id, username, email)
VALUES (1, 'john', 'john@example.com')
ON CONFLICT (id) DO NOTHING;

-- Обновить при конфликте
INSERT INTO users (id, username, email)
VALUES (1, 'john', 'john@example.com')
ON CONFLICT (id) DO UPDATE SET
    username = EXCLUDED.username,
    email = EXCLUDED.email,
    updated_at = NOW();

-- Конфликт по уникальному ограничению
INSERT INTO products (sku, name, price)
VALUES ('ABC123', 'Product', 99.99)
ON CONFLICT ON CONSTRAINT products_sku_key DO UPDATE SET
    price = EXCLUDED.price;

-- Условный UPDATE при конфликте
INSERT INTO inventory (product_id, quantity)
VALUES (1, 10)
ON CONFLICT (product_id) DO UPDATE SET
    quantity = inventory.quantity + EXCLUDED.quantity
WHERE inventory.quantity + EXCLUDED.quantity >= 0;
```

### INSERT с DEFAULT

```sql
-- Использовать значение по умолчанию
INSERT INTO users (username, email, status)
VALUES ('john', 'john@example.com', DEFAULT);

-- Все значения по умолчанию
INSERT INTO logs DEFAULT VALUES;
```

---

## UPDATE — Изменение данных

### Базовый UPDATE

```sql
-- Обновить одно поле
UPDATE users SET status = 'inactive' WHERE id = 1;

-- Обновить несколько полей
UPDATE users SET
    status = 'active',
    updated_at = NOW()
WHERE id = 1;

-- Обновить все записи (осторожно!)
UPDATE products SET price = price * 1.1;
```

### UPDATE с RETURNING

```sql
-- Получить обновленные данные
UPDATE users
SET status = 'active'
WHERE id = 1
RETURNING *;

-- Получить только измененные столбцы
UPDATE products
SET price = price * 1.1
WHERE category = 'electronics'
RETURNING id, name, price;
```

### UPDATE с подзапросом

```sql
-- Обновление на основе данных из другой таблицы
UPDATE orders
SET status = 'shipped'
WHERE id IN (
    SELECT order_id
    FROM shipments
    WHERE shipped_at IS NOT NULL
);

-- UPDATE с FROM
UPDATE orders o
SET total_amount = s.calculated_total
FROM (
    SELECT order_id, SUM(price * quantity) as calculated_total
    FROM order_items
    GROUP BY order_id
) s
WHERE o.id = s.order_id;

-- UPDATE с JOIN
UPDATE products p
SET category_name = c.name
FROM categories c
WHERE p.category_id = c.id;
```

### UPDATE с вычислениями

```sql
-- Увеличить значение
UPDATE products SET stock = stock - 1 WHERE id = 1;

-- Условное обновление с CASE
UPDATE users SET
    status = CASE
        WHEN last_login < NOW() - INTERVAL '1 year' THEN 'inactive'
        WHEN last_login < NOW() - INTERVAL '1 month' THEN 'dormant'
        ELSE 'active'
    END;

-- Обновление JSON
UPDATE users
SET settings = settings || '{"theme": "dark"}'
WHERE id = 1;
```

---

## DELETE — Удаление данных

### Базовый DELETE

```sql
-- Удалить одну запись
DELETE FROM users WHERE id = 1;

-- Удалить по условию
DELETE FROM users WHERE status = 'deleted';

-- Удалить все записи (осторожно!)
DELETE FROM temp_data;
```

### DELETE с RETURNING

```sql
-- Получить удаленные данные
DELETE FROM users
WHERE status = 'deleted'
RETURNING *;

-- Получить количество удаленных
DELETE FROM old_logs
WHERE created_at < NOW() - INTERVAL '1 year'
RETURNING id;
```

### DELETE с подзапросом

```sql
-- Удаление по связанным данным
DELETE FROM order_items
WHERE order_id IN (
    SELECT id FROM orders WHERE status = 'cancelled'
);

-- DELETE с USING (PostgreSQL)
DELETE FROM order_items oi
USING orders o
WHERE oi.order_id = o.id AND o.status = 'cancelled';
```

### Soft Delete — Мягкое удаление

```sql
-- Вместо DELETE используем UPDATE
UPDATE users
SET deleted_at = NOW(), status = 'deleted'
WHERE id = 1;

-- Создание представления для "живых" записей
CREATE VIEW active_users AS
SELECT * FROM users WHERE deleted_at IS NULL;
```

---

## Лучшие практики DML

### 1. Всегда используйте WHERE в UPDATE/DELETE

```sql
-- ОПАСНО: обновит ВСЕ записи!
UPDATE users SET status = 'inactive';

-- БЕЗОПАСНО: только конкретные записи
UPDATE users SET status = 'inactive' WHERE id = 1;

-- Для массовых операций — явно указывайте WHERE TRUE
UPDATE users SET status = 'inactive' WHERE TRUE;  -- осознанное решение
```

### 2. Используйте транзакции

```sql
BEGIN;

DELETE FROM order_items WHERE order_id = 1;
DELETE FROM orders WHERE id = 1;

-- Проверяем результат перед COMMIT
SELECT * FROM orders WHERE id = 1;

COMMIT;
-- или ROLLBACK; если что-то пошло не так
```

### 3. Ограничивайте DELETE/UPDATE

```sql
-- Удаление партиями
DELETE FROM logs
WHERE created_at < '2020-01-01'
LIMIT 1000;

-- В цикле (псевдокод)
-- WHILE rows_affected > 0:
--     DELETE ... LIMIT 1000;
```

### 4. Используйте EXPLAIN для оптимизации

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'test@example.com';
```

---

## Типичные ошибки

### 1. Забыть WHERE в UPDATE/DELETE

```sql
-- Ошибка: обновит все записи
UPDATE users SET password = 'reset';

-- Правильно
UPDATE users SET password = 'reset' WHERE id = 1;
```

### 2. Неэффективные подзапросы

```sql
-- Медленно: подзапрос выполняется для каждой строки
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders);

-- Быстрее: использовать EXISTS
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- Или JOIN
SELECT DISTINCT u.*
FROM users u
JOIN orders o ON u.id = o.user_id;
```

### 3. SELECT * в продакшене

```sql
-- Плохо: передает лишние данные
SELECT * FROM users;

-- Хорошо: только нужные столбцы
SELECT id, username, email FROM users;
```

### 4. Пагинация через OFFSET на больших таблицах

```sql
-- Медленно для больших OFFSET
SELECT * FROM logs ORDER BY id LIMIT 10 OFFSET 1000000;

-- Быстрее: keyset pagination
SELECT * FROM logs
WHERE id > 1000000
ORDER BY id
LIMIT 10;
```

### 5. Игнорирование NULL

```sql
-- НЕ РАБОТАЕТ: NULL не равен ничему
SELECT * FROM users WHERE phone = NULL;

-- Правильно
SELECT * FROM users WHERE phone IS NULL;

-- НЕ РАБОТАЕТ с NOT IN
SELECT * FROM users WHERE id NOT IN (1, 2, NULL);  -- вернет 0 строк!

-- Правильно
SELECT * FROM users WHERE id NOT IN (SELECT id FROM ... WHERE ... IS NOT NULL);
```

---

## Полезные паттерны

### Batch INSERT с генерацией данных

```sql
INSERT INTO test_data (value)
SELECT random() FROM generate_series(1, 1000000);
```

### Обновление с лимитом

```sql
WITH to_update AS (
    SELECT id FROM users
    WHERE status = 'pending'
    LIMIT 100
    FOR UPDATE SKIP LOCKED
)
UPDATE users
SET status = 'processing'
WHERE id IN (SELECT id FROM to_update)
RETURNING id;
```

### Атомарный UPSERT с подсчетом

```sql
INSERT INTO page_views (page_id, view_count)
VALUES (1, 1)
ON CONFLICT (page_id) DO UPDATE SET
    view_count = page_views.view_count + 1
RETURNING view_count;
```

---

[prev: 01-ddl-queries](./01-ddl-queries.md) | [next: 03-data-types](./03-data-types.md)
