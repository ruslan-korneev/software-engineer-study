# Операции над множествами (Set Operations)

## Обзор операций над множествами

Операции над множествами позволяют комбинировать результаты нескольких SELECT-запросов. PostgreSQL поддерживает три основные операции:

| Операция | Описание |
|----------|----------|
| `UNION` | Объединение (все уникальные строки из обоих запросов) |
| `INTERSECT` | Пересечение (только общие строки) |
| `EXCEPT` | Разность (строки из первого запроса, которых нет во втором) |

---

## Правила использования

### Требования к запросам

1. **Одинаковое количество столбцов** в каждом SELECT
2. **Совместимые типы данных** в соответствующих столбцах
3. **Имена столбцов** берутся из первого запроса

```sql
-- Правильно: одинаковое количество столбцов и совместимые типы
SELECT id, name FROM customers
UNION
SELECT id, company_name FROM suppliers;

-- Ошибка: разное количество столбцов
SELECT id, name, email FROM customers
UNION
SELECT id, name FROM suppliers;
-- ERROR: each UNION query must have the same number of columns

-- Ошибка: несовместимые типы
SELECT id, created_at FROM orders  -- INTEGER, TIMESTAMP
UNION
SELECT name, email FROM users;      -- TEXT, TEXT
-- ERROR: UNION types integer and text cannot be matched
```

---

## UNION — Объединение

### UNION (без дубликатов)

Возвращает уникальные строки из обоих запросов:

```sql
-- Все email из пользователей и подписчиков (без дубликатов)
SELECT email FROM users
UNION
SELECT email FROM subscribers;

-- Результат:
-- email
-- -----
-- alice@example.com
-- bob@example.com
-- charlie@example.com
```

### UNION ALL (с дубликатами)

Возвращает все строки, включая дубликаты (быстрее, так как не требует сортировки):

```sql
SELECT email FROM users
UNION ALL
SELECT email FROM subscribers;

-- Результат (если bob есть в обеих таблицах):
-- email
-- -----
-- alice@example.com
-- bob@example.com
-- bob@example.com  <-- дубликат
-- charlie@example.com
```

### Когда использовать UNION ALL

```sql
-- Объединение данных из партиций (дубликаты невозможны)
SELECT * FROM orders_2023
UNION ALL
SELECT * FROM orders_2024;

-- Объединение с разными источниками (нужно сохранить все)
SELECT 'user' AS source, email FROM users
UNION ALL
SELECT 'subscriber' AS source, email FROM subscribers;

-- Подсчет общего количества
SELECT COUNT(*) FROM (
    SELECT id FROM active_users
    UNION ALL
    SELECT id FROM inactive_users
) all_users;
```

### Производительность

```sql
-- UNION: требует сортировку и удаление дубликатов
EXPLAIN SELECT email FROM users UNION SELECT email FROM subscribers;
-- HashAggregate или Sort + Unique

-- UNION ALL: просто добавляет результаты
EXPLAIN SELECT email FROM users UNION ALL SELECT email FROM subscribers;
-- Append (быстрее)
```

---

## INTERSECT — Пересечение

### INTERSECT (без дубликатов)

Возвращает строки, присутствующие в обоих запросах:

```sql
-- Email, которые есть и в users, и в subscribers
SELECT email FROM users
INTERSECT
SELECT email FROM subscribers;

-- Эквивалентно:
SELECT DISTINCT u.email
FROM users u
WHERE EXISTS (SELECT 1 FROM subscribers s WHERE s.email = u.email);
```

### INTERSECT ALL (с учетом количества)

Сохраняет дубликаты в количестве минимального вхождения:

```sql
-- Если в users email встречается 3 раза, а в subscribers 2 раза,
-- в результате будет 2 раза (минимум)

SELECT email FROM users
INTERSECT ALL
SELECT email FROM subscribers;
```

### Практические примеры

```sql
-- Клиенты, купившие и товар A, и товар B
SELECT customer_id FROM orders WHERE product = 'A'
INTERSECT
SELECT customer_id FROM orders WHERE product = 'B';

-- Пользователи с доступом к обоим ресурсам
SELECT user_id FROM resource_access WHERE resource = 'admin'
INTERSECT
SELECT user_id FROM resource_access WHERE resource = 'reports';
```

---

## EXCEPT — Разность

### EXCEPT (без дубликатов)

Возвращает строки из первого запроса, которых нет во втором:

```sql
-- Пользователи, которые НЕ являются подписчиками
SELECT email FROM users
EXCEPT
SELECT email FROM subscribers;

-- Эквивалентно:
SELECT DISTINCT email
FROM users
WHERE email NOT IN (SELECT email FROM subscribers);

-- Или с NOT EXISTS:
SELECT DISTINCT u.email
FROM users u
WHERE NOT EXISTS (SELECT 1 FROM subscribers s WHERE s.email = u.email);
```

### EXCEPT ALL (с учетом количества)

Вычитает с учетом количества вхождений:

```sql
-- Если email в users 3 раза, а в subscribers 1 раз,
-- в результате будет 2 раза (3 - 1)

SELECT email FROM users
EXCEPT ALL
SELECT email FROM subscribers;
```

### Практические примеры

```sql
-- Товары без продаж за последний месяц
SELECT product_id FROM products
EXCEPT
SELECT product_id FROM orders WHERE order_date >= NOW() - INTERVAL '1 month';

-- Пользователи без заказов
SELECT id FROM users
EXCEPT
SELECT DISTINCT user_id FROM orders;
```

---

## Порядок операций и скобки

### Приоритет операций

По умолчанию все операции имеют одинаковый приоритет и выполняются слева направо:

```sql
-- Выполняется как: (A UNION B) INTERSECT C
SELECT * FROM A
UNION
SELECT * FROM B
INTERSECT
SELECT * FROM C;

-- Используйте скобки для явного порядка
(SELECT * FROM A UNION SELECT * FROM B)
INTERSECT
SELECT * FROM C;

-- Или
SELECT * FROM A
UNION
(SELECT * FROM B INTERSECT SELECT * FROM C);
```

### Комбинирование операций

```sql
-- Все активные пользователи, кроме заблокированных
(
    SELECT id, email FROM users WHERE status = 'active'
    UNION
    SELECT id, email FROM admins WHERE status = 'active'
)
EXCEPT
SELECT id, email FROM blocked_users;
```

---

## ORDER BY и LIMIT

### ORDER BY применяется к результату

```sql
-- Сортировка всего результата
SELECT name, 'user' AS type FROM users
UNION
SELECT name, 'admin' AS type FROM admins
ORDER BY name;  -- Применяется после UNION

-- Ошибка: нельзя сортировать отдельные части
SELECT name FROM users ORDER BY name  -- ERROR
UNION
SELECT name FROM admins;

-- Правильно: сортировка в подзапросе с LIMIT
(SELECT name FROM users ORDER BY created_at DESC LIMIT 5)
UNION ALL
(SELECT name FROM admins ORDER BY created_at DESC LIMIT 5)
ORDER BY name;
```

### LIMIT и OFFSET

```sql
-- Ограничение общего результата
SELECT email FROM users
UNION
SELECT email FROM subscribers
LIMIT 10 OFFSET 20;

-- Разное ограничение для частей
(SELECT email FROM users ORDER BY created_at DESC LIMIT 5)
UNION ALL
(SELECT email FROM subscribers ORDER BY created_at DESC LIMIT 5);
```

---

## Практические примеры

### Объединение данных из разных источников

```sql
-- Все контакты из разных таблиц
SELECT
    'customer' AS source,
    name,
    email,
    phone
FROM customers
UNION ALL
SELECT
    'supplier' AS source,
    contact_name,
    contact_email,
    contact_phone
FROM suppliers
UNION ALL
SELECT
    'employee' AS source,
    full_name,
    work_email,
    work_phone
FROM employees
ORDER BY name;
```

### Сравнение таблиц

```sql
-- Найти различия между таблицами
-- Строки только в table1
SELECT *, 'only in table1' AS status
FROM table1
EXCEPT
SELECT *, 'only in table1'
FROM table2

UNION ALL

-- Строки только в table2
SELECT *, 'only in table2'
FROM table2
EXCEPT
SELECT *, 'only in table2'
FROM table1;
```

### Создание сводного отчета

```sql
-- Отчет с промежуточными итогами
SELECT
    category,
    product_name,
    SUM(sales) AS total_sales
FROM sales_data
GROUP BY category, product_name

UNION ALL

SELECT
    category,
    'CATEGORY TOTAL' AS product_name,
    SUM(sales)
FROM sales_data
GROUP BY category

UNION ALL

SELECT
    'GRAND TOTAL' AS category,
    NULL AS product_name,
    SUM(sales)
FROM sales_data

ORDER BY
    category NULLS LAST,
    CASE WHEN product_name = 'CATEGORY TOTAL' THEN 1
         WHEN product_name IS NULL THEN 2
         ELSE 0 END,
    product_name;
```

### Пагинация по нескольким таблицам

```sql
-- Объединенный список событий с пагинацией
WITH combined_events AS (
    SELECT id, 'order' AS type, created_at, description
    FROM orders
    UNION ALL
    SELECT id, 'payment' AS type, created_at, description
    FROM payments
    UNION ALL
    SELECT id, 'shipment' AS type, created_at, description
    FROM shipments
)
SELECT *
FROM combined_events
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;
```

### Временные ряды с разными источниками

```sql
-- Объединение метрик из разных таблиц
SELECT date, 'revenue' AS metric, SUM(amount) AS value
FROM orders
GROUP BY date
UNION ALL
SELECT date, 'expenses' AS metric, SUM(amount) AS value
FROM expenses
GROUP BY date
UNION ALL
SELECT date, 'profit' AS metric, revenue - expenses AS value
FROM (
    SELECT
        COALESCE(o.date, e.date) AS date,
        COALESCE(SUM(o.amount), 0) AS revenue,
        COALESCE(SUM(e.amount), 0) AS expenses
    FROM orders o
    FULL OUTER JOIN expenses e ON o.date = e.date
    GROUP BY COALESCE(o.date, e.date)
) calculated
ORDER BY date, metric;
```

---

## Сравнение с другими подходами

### UNION vs OR в WHERE

```sql
-- С OR (может быть эффективнее с индексом)
SELECT * FROM users
WHERE department = 'Sales' OR department = 'Marketing';

-- С UNION (может использовать разные индексы)
SELECT * FROM users WHERE department = 'Sales'
UNION
SELECT * FROM users WHERE department = 'Marketing';

-- IN часто лучше
SELECT * FROM users WHERE department IN ('Sales', 'Marketing');
```

### INTERSECT vs JOIN

```sql
-- INTERSECT: только общие значения
SELECT product_id FROM inventory
INTERSECT
SELECT product_id FROM orders;

-- JOIN: можно получить дополнительные данные
SELECT DISTINCT i.product_id, i.quantity
FROM inventory i
JOIN orders o ON i.product_id = o.product_id;

-- EXISTS: часто эффективнее
SELECT product_id FROM inventory i
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.product_id = i.product_id);
```

### EXCEPT vs NOT EXISTS

```sql
-- EXCEPT
SELECT customer_id FROM customers
EXCEPT
SELECT customer_id FROM orders;

-- NOT EXISTS (более гибкий)
SELECT customer_id
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);

-- NOT IN (осторожно с NULL!)
SELECT customer_id FROM customers
WHERE customer_id NOT IN (
    SELECT customer_id FROM orders WHERE customer_id IS NOT NULL
);
```

---

## Оптимизация

### Использование UNION ALL когда возможно

```sql
-- Если дубликаты невозможны или допустимы
-- UNION ALL всегда быстрее

-- Партиционированные данные (дубликаты невозможны)
SELECT * FROM sales_2023 WHERE amount > 1000
UNION ALL
SELECT * FROM sales_2024 WHERE amount > 1000;
```

### Фильтрация перед операцией

```sql
-- Медленнее: фильтрация после объединения
SELECT * FROM (
    SELECT * FROM table1
    UNION ALL
    SELECT * FROM table2
) combined
WHERE status = 'active';

-- Быстрее: фильтрация до объединения
SELECT * FROM table1 WHERE status = 'active'
UNION ALL
SELECT * FROM table2 WHERE status = 'active';
```

### Использование LIMIT в подзапросах

```sql
-- Для top-N из разных источников
(SELECT * FROM orders ORDER BY created_at DESC LIMIT 100)
UNION ALL
(SELECT * FROM archived_orders ORDER BY created_at DESC LIMIT 100)
ORDER BY created_at DESC
LIMIT 100;
```

---

## Типичные ошибки

### 1. Неправильный порядок столбцов

```sql
-- Столбцы не соответствуют по смыслу
SELECT id, name FROM customers
UNION
SELECT name, id FROM suppliers;  -- Перепутаны местами!

-- Правильно
SELECT id, name FROM customers
UNION
SELECT id, company_name FROM suppliers;
```

### 2. Забыть про NULL в EXCEPT

```sql
-- NULL может создать неожиданные результаты
SELECT email FROM users
EXCEPT
SELECT email FROM verified_users;
-- NULL != NULL, поэтому NULL попадет в результат

-- Учитывайте NULL
SELECT email FROM users WHERE email IS NOT NULL
EXCEPT
SELECT email FROM verified_users WHERE email IS NOT NULL;
```

### 3. Использование UNION вместо UNION ALL

```sql
-- Теряем производительность
SELECT * FROM partition_2023
UNION  -- Зачем удалять дубликаты между партициями?
SELECT * FROM partition_2024;

-- Правильно
SELECT * FROM partition_2023
UNION ALL
SELECT * FROM partition_2024;
```

### 4. Сортировка в подзапросах без LIMIT

```sql
-- ORDER BY без LIMIT в подзапросе игнорируется!
SELECT name FROM users ORDER BY created_at  -- Бесполезно
UNION
SELECT name FROM admins;

-- Если нужна сортировка - в конце или с LIMIT
SELECT name FROM users
UNION
SELECT name FROM admins
ORDER BY name;
```

---

## Лучшие практики

1. **Используйте UNION ALL** когда дубликаты невозможны или допустимы
2. **Фильтруйте до операции**, а не после
3. **Давайте понятные псевдонимы** столбцам в первом запросе
4. **Используйте скобки** для явного порядка операций
5. **Проверяйте типы данных** соответствующих столбцов
6. **Сортировку и LIMIT** размещайте в конце или в подзапросах с LIMIT
