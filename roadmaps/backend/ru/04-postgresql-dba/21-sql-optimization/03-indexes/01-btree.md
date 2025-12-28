# B-Tree индексы

[prev: 02-query-patterns](../02-query-patterns.md) | [next: 02-hash](./02-hash.md)

---

## Введение

B-Tree (Balanced Tree) — это **тип индекса по умолчанию** в PostgreSQL. Если вы создаёте индекс без указания типа, будет создан именно B-Tree индекс. Это самый универсальный и часто используемый тип индекса.

---

## Структура B-Tree

### Как устроен B-Tree

B-Tree — это сбалансированное дерево, где:
- Все листья находятся на одном уровне
- Каждый узел содержит отсортированные ключи
- Между узлами есть указатели для навигации

```
                    [50]
                   /    \
            [20, 35]    [70, 85]
           /   |   \    /   |   \
        [10] [25] [40] [60] [75] [90]
          ↓    ↓    ↓    ↓    ↓    ↓
        данные  ...  ...  ...  ...  данные
```

### Характеристики

- **Время поиска**: O(log n)
- **Время вставки**: O(log n)
- **Размер**: обычно 2-3% от размера таблицы
- **Сортировка**: данные в индексе отсортированы

---

## Создание B-Tree индекса

### Базовый синтаксис

```sql
-- Простой индекс
CREATE INDEX idx_users_email ON users (email);

-- Явное указание типа (необязательно для B-Tree)
CREATE INDEX idx_users_email ON users USING btree (email);

-- Уникальный индекс
CREATE UNIQUE INDEX idx_users_email ON users (email);

-- Индекс с условием (частичный индекс)
CREATE INDEX idx_active_users ON users (created_at)
WHERE status = 'active';

-- Индекс с указанием порядка сортировки
CREATE INDEX idx_orders_date ON orders (created_at DESC NULLS LAST);
```

### Многоколоночные индексы

```sql
-- Составной индекс
CREATE INDEX idx_orders_customer_date ON orders (customer_id, created_at);
```

**Важно понимать порядок колонок:**

Индекс `(customer_id, created_at)` эффективен для:
```sql
-- Использует индекс полностью
WHERE customer_id = 1 AND created_at > '2024-01-01'

-- Использует только первую колонку
WHERE customer_id = 1

-- Использует индекс (Index Skip Scan в PG 13+)
WHERE created_at > '2024-01-01'
```

Но **НЕ эффективен** для:
```sql
-- В старых версиях PostgreSQL не использовал индекс
WHERE created_at > '2024-01-01'  -- без customer_id
```

### Правило левого префикса

Для индекса `(a, b, c)`:
- `WHERE a = ?` — использует индекс
- `WHERE a = ? AND b = ?` — использует индекс
- `WHERE a = ? AND b = ? AND c = ?` — использует индекс полностью
- `WHERE b = ?` — **не использует** индекс (в большинстве случаев)
- `WHERE a = ? AND c = ?` — использует только колонку `a`

---

## Операторы, поддерживаемые B-Tree

B-Tree поддерживает операторы сравнения:

| Оператор | Пример | Описание |
|----------|--------|----------|
| `=` | `WHERE id = 5` | Равенство |
| `<` | `WHERE price < 100` | Меньше |
| `<=` | `WHERE price <= 100` | Меньше или равно |
| `>` | `WHERE date > '2024-01-01'` | Больше |
| `>=` | `WHERE date >= '2024-01-01'` | Больше или равно |
| `BETWEEN` | `WHERE price BETWEEN 10 AND 100` | Диапазон |
| `IN` | `WHERE status IN ('a', 'b')` | Множество значений |
| `IS NULL` | `WHERE deleted_at IS NULL` | Проверка на NULL |
| `LIKE` | `WHERE name LIKE 'John%'` | Префиксный поиск |

### LIKE и B-Tree

```sql
-- Использует индекс (поиск по префиксу)
SELECT * FROM users WHERE name LIKE 'John%';

-- НЕ использует индекс (подстрока в середине)
SELECT * FROM users WHERE name LIKE '%John%';

-- НЕ использует индекс (подстрока в конце)
SELECT * FROM users WHERE name LIKE '%John';
```

Для суффиксного поиска можно создать индекс на обратную строку:
```sql
CREATE INDEX idx_users_name_reverse ON users (reverse(name));
SELECT * FROM users WHERE reverse(name) LIKE reverse('%son');
```

---

## Типы сканирования с B-Tree

### Index Scan

```sql
EXPLAIN SELECT * FROM users WHERE id = 100;

-- Index Scan using users_pkey on users
--   Index Cond: (id = 100)
```
Сначала поиск в индексе, потом обращение к таблице за данными.

### Index Only Scan

```sql
-- Индекс содержит все нужные колонки
CREATE INDEX idx_users_email_name ON users (email, name);

EXPLAIN SELECT email, name FROM users WHERE email = 'test@example.com';

-- Index Only Scan using idx_users_email_name on users
--   Index Cond: (email = 'test@example.com')
```
Данные берутся только из индекса, без обращения к таблице.

**Условия для Index Only Scan:**
- Все возвращаемые колонки есть в индексе
- Visibility map показывает, что страницы "all-visible"

### Bitmap Index Scan

```sql
EXPLAIN SELECT * FROM orders WHERE status = 'pending' AND amount > 1000;

-- Bitmap Heap Scan on orders
--   Recheck Cond: ((status = 'pending') AND (amount > 1000))
--   ->  BitmapAnd
--         ->  Bitmap Index Scan on idx_orders_status
--         ->  Bitmap Index Scan on idx_orders_amount
```
Используется при большом количестве результатов или объединении нескольких индексов.

---

## Практические примеры

### Пример 1: Оптимизация поиска по email

```sql
-- Создаём таблицу
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email TEXT NOT NULL,
    name TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Вставляем тестовые данные
INSERT INTO users (email, name)
SELECT
    'user' || i || '@example.com',
    'User ' || i
FROM generate_series(1, 1000000) i;

-- Без индекса
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user500000@example.com';
-- Seq Scan on users  (cost=0.00..19853.00 rows=1 width=52)
--   Filter: (email = 'user500000@example.com')
-- Planning Time: 0.123 ms
-- Execution Time: 98.456 ms

-- Создаём индекс
CREATE INDEX idx_users_email ON users (email);

-- С индексом
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user500000@example.com';
-- Index Scan using idx_users_email on users  (cost=0.42..8.44 rows=1 width=52)
--   Index Cond: (email = 'user500000@example.com')
-- Planning Time: 0.156 ms
-- Execution Time: 0.034 ms
```

### Пример 2: Составной индекс для заказов

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    status TEXT NOT NULL,
    amount NUMERIC(10,2),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Типичный запрос: заказы клиента за период
SELECT * FROM orders
WHERE customer_id = 123
  AND created_at BETWEEN '2024-01-01' AND '2024-12-31'
ORDER BY created_at DESC;

-- Оптимальный индекс для этого запроса
CREATE INDEX idx_orders_customer_date ON orders (customer_id, created_at DESC);
```

### Пример 3: Частичный индекс

```sql
-- Таблица с миллионами записей, но активных только 1%
CREATE TABLE tasks (
    id SERIAL PRIMARY KEY,
    title TEXT,
    status TEXT DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Вместо полного индекса
CREATE INDEX idx_tasks_status ON tasks (status);  -- Большой индекс

-- Создаём частичный индекс только для активных задач
CREATE INDEX idx_tasks_pending ON tasks (created_at)
WHERE status = 'pending';

-- Размер значительно меньше, запросы быстрее
SELECT * FROM tasks WHERE status = 'pending' ORDER BY created_at;
```

---

## Когда использовать B-Tree

### Идеально подходит для:
- Поиск по равенству (`=`)
- Диапазонные запросы (`<`, `>`, `BETWEEN`)
- Сортировка (`ORDER BY`)
- Уникальные ограничения (`UNIQUE`)
- Первичные ключи (`PRIMARY KEY`)
- Префиксный поиск (`LIKE 'prefix%'`)

### Не подходит для:
- Полнотекстовый поиск (используйте GIN/GiST)
- Геометрические данные (используйте GiST)
- Поиск по массивам (используйте GIN)
- Поиск подстроки `LIKE '%text%'` (используйте GIN с pg_trgm)
- Очень большие таблицы с последовательными данными (рассмотрите BRIN)

---

## Оптимизация B-Tree индексов

### INCLUDE для Index Only Scan (PostgreSQL 11+)

```sql
-- Обычный индекс — нужен доступ к таблице для получения name
CREATE INDEX idx_users_email ON users (email);
SELECT email, name FROM users WHERE email = 'test@example.com';

-- Индекс с INCLUDE — Index Only Scan
CREATE INDEX idx_users_email ON users (email) INCLUDE (name);
SELECT email, name FROM users WHERE email = 'test@example.com';
```

### Параллельное создание индекса

```sql
-- Не блокирует таблицу на запись
CREATE INDEX CONCURRENTLY idx_users_email ON users (email);
```

### Анализ использования индексов

```sql
-- Статистика использования индексов
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,      -- Количество использований
    idx_tup_read,  -- Прочитано строк из индекса
    idx_tup_fetch  -- Получено строк из таблицы
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan DESC;

-- Неиспользуемые индексы
SELECT
    indexname,
    idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexname NOT LIKE '%_pkey';
```

### Обслуживание индексов

```sql
-- Перестроение индекса (дефрагментация)
REINDEX INDEX idx_users_email;
REINDEX INDEX CONCURRENTLY idx_users_email;  -- PostgreSQL 12+

-- Проверка размера индекса
SELECT pg_size_pretty(pg_relation_size('idx_users_email'));

-- Bloat индекса (примерная оценка)
SELECT
    indexrelname,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
WHERE schemaname = 'public';
```

---

## Типичные ошибки

1. **Создание индекса на каждую колонку**
   - Индексы занимают место и замедляют INSERT/UPDATE
   - Создавайте только нужные индексы

2. **Неправильный порядок колонок в составном индексе**
   - Ставьте сначала колонки для равенства, потом для диапазонов
   - `(status, created_at)` лучше чем `(created_at, status)` для `WHERE status = 'active' AND created_at > ...`

3. **Индекс на низкоселективные колонки**
   ```sql
   -- Плохо: только 3 уникальных значения
   CREATE INDEX idx_users_gender ON users (gender);
   -- PostgreSQL предпочтёт Seq Scan
   ```

4. **Забывают про NULL**
   ```sql
   -- NULL-значения индексируются в B-Tree
   CREATE INDEX idx_users_deleted ON users (deleted_at);
   -- Но для WHERE deleted_at IS NULL частичный индекс эффективнее:
   CREATE INDEX idx_users_active ON users (id) WHERE deleted_at IS NULL;
   ```

5. **Игнорирование collation**
   ```sql
   -- Для регистронезависимого поиска нужен соответствующий индекс
   CREATE INDEX idx_users_name_ci ON users (name COLLATE "und-x-icu");
   -- Или функциональный индекс
   CREATE INDEX idx_users_name_lower ON users (LOWER(name));
   ```

---

## Best Practices

1. Используйте `EXPLAIN ANALYZE` для проверки использования индексов
2. Создавайте индексы на внешние ключи (FK)
3. Используйте `CONCURRENTLY` для создания индексов на production
4. Регулярно проверяйте неиспользуемые индексы
5. Используйте частичные индексы для экономии места
6. Применяйте `INCLUDE` для Index Only Scan
7. Мониторьте bloat индексов и делайте REINDEX при необходимости

---

[prev: 02-query-patterns](../02-query-patterns.md) | [next: 02-hash](./02-hash.md)
