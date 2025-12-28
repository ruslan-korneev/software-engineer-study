# Query Processing — Обработка запросов в PostgreSQL

[prev: 04-write-ahead-log](./04-write-ahead-log.md) | [next: 01-using-docker](../04-installation-and-setup/01-using-docker.md)
---

## Введение

Когда вы выполняете SQL-запрос, PostgreSQL проходит через несколько этапов обработки, прежде чем вернуть результат. Понимание этих этапов критически важно для оптимизации запросов и диагностики проблем производительности.

```
┌─────────────────────────────────────────────────────────────────┐
│                 Путь SQL-запроса в PostgreSQL                   │
│                                                                 │
│  SQL Query                                                      │
│      ↓                                                          │
│  ┌─────────────┐                                                │
│  │   Parser    │ → Синтаксический разбор → Parse Tree          │
│  └─────────────┘                                                │
│      ↓                                                          │
│  ┌─────────────┐                                                │
│  │  Analyzer   │ → Семантический анализ → Query Tree           │
│  └─────────────┘                                                │
│      ↓                                                          │
│  ┌─────────────┐                                                │
│  │  Rewriter   │ → Применение правил → Rewritten Query Tree    │
│  └─────────────┘                                                │
│      ↓                                                          │
│  ┌─────────────┐                                                │
│  │  Planner    │ → Оптимизация → Query Plan                    │
│  └─────────────┘                                                │
│      ↓                                                          │
│  ┌─────────────┐                                                │
│  │  Executor   │ → Выполнение → Result                         │
│  └─────────────┘                                                │
└─────────────────────────────────────────────────────────────────┘
```

## 1. Parser (Парсер)

### Лексический и синтаксический анализ

Парсер разбирает SQL-текст и создаёт **Parse Tree** — дерево разбора.

```sql
-- Исходный запрос
SELECT name, email FROM users WHERE age > 18;

-- Parse Tree (концептуально):
SelectStmt
├── targetList: [name, email]
├── fromClause: [users]
└── whereClause: (age > 18)
```

### Ошибки на этапе парсинга

```sql
-- Синтаксическая ошибка — обнаруживается парсером
SELEC name FROM users;
-- ERROR: syntax error at or near "SELEC"

-- Несуществующий оператор — тоже ошибка парсинга
SELECT * FROM users WHER id = 1;
-- ERROR: syntax error at or near "WHER"
```

## 2. Analyzer (Анализатор)

### Семантический анализ

Анализатор проверяет, что объекты существуют и типы совместимы, создавая **Query Tree**.

```sql
-- Анализатор проверяет:
-- 1. Существует ли таблица users?
-- 2. Есть ли столбцы name, email, age?
-- 3. Можно ли сравнивать age с числом 18?

SELECT name, email FROM users WHERE age > 18;
```

### Ошибки на этапе анализа

```sql
-- Несуществующая таблица
SELECT * FROM nonexistent_table;
-- ERROR: relation "nonexistent_table" does not exist

-- Несуществующий столбец
SELECT unknown_column FROM users;
-- ERROR: column "unknown_column" does not exist

-- Несовместимые типы
SELECT * FROM users WHERE name > 100;
-- Может работать (неявное приведение) или дать ошибку
```

## 3. Rewriter (Переписывание)

### Применение правил и представлений

Rewriter раскрывает представления (views) и применяет правила (rules).

```sql
-- Представление
CREATE VIEW active_users AS
SELECT * FROM users WHERE status = 'active';

-- Запрос к представлению
SELECT name FROM active_users WHERE age > 18;

-- После переписывания (концептуально):
SELECT name FROM users WHERE status = 'active' AND age > 18;
```

### Rules (Правила)

```sql
-- Создание правила для аудита
CREATE RULE log_user_updates AS
ON UPDATE TO users
DO ALSO
INSERT INTO audit_log (table_name, action, timestamp)
VALUES ('users', 'UPDATE', NOW());

-- При UPDATE users автоматически добавится INSERT в audit_log
```

## 4. Planner/Optimizer (Планировщик)

### Создание плана выполнения

Планировщик — самый сложный компонент. Он выбирает оптимальный способ выполнения запроса.

```sql
-- Просмотр плана выполнения
EXPLAIN SELECT * FROM users WHERE id = 100;

-- Результат:
Index Scan using users_pkey on users  (cost=0.29..8.30 rows=1 width=64)
  Index Cond: (id = 100)
```

### Cost-based optimization

PostgreSQL использует **стоимостную модель** для выбора плана:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Составляющие стоимости                       │
│                                                                 │
│  seq_page_cost = 1.0        # Последовательное чтение страницы │
│  random_page_cost = 4.0     # Случайное чтение страницы        │
│  cpu_tuple_cost = 0.01      # Обработка одной строки           │
│  cpu_index_tuple_cost = 0.005  # Обработка записи индекса      │
│  cpu_operator_cost = 0.0025    # Выполнение оператора          │
└─────────────────────────────────────────────────────────────────┘
```

### Статистика для планировщика

```sql
-- Просмотр статистики таблицы
SELECT relname, reltuples, relpages
FROM pg_class
WHERE relname = 'users';

-- Детальная статистика по столбцам
SELECT attname, n_distinct, most_common_vals, histogram_bounds
FROM pg_stats
WHERE tablename = 'users' AND attname = 'status';

-- Обновление статистики
ANALYZE users;
```

## Методы доступа к данным

### Sequential Scan

```sql
-- Полный просмотр таблицы
EXPLAIN SELECT * FROM users WHERE status = 'active';

-- Seq Scan on users  (cost=0.00..1834.00 rows=50000 width=64)
--   Filter: (status = 'active')
```

### Index Scan

```sql
-- Использование индекса с доступом к таблице
CREATE INDEX idx_users_id ON users(id);

EXPLAIN SELECT * FROM users WHERE id = 100;

-- Index Scan using idx_users_id on users  (cost=0.29..8.30 rows=1 width=64)
--   Index Cond: (id = 100)
```

### Index Only Scan

```sql
-- Все данные берутся из индекса
CREATE INDEX idx_users_email ON users(email) INCLUDE (name);

EXPLAIN SELECT email, name FROM users WHERE email = 'test@example.com';

-- Index Only Scan using idx_users_email on users  (cost=0.29..4.30 rows=1 width=64)
--   Index Cond: (email = 'test@example.com')
```

### Bitmap Scan

```sql
-- Двухфазное сканирование для множества строк
EXPLAIN SELECT * FROM orders WHERE status IN ('pending', 'processing');

-- Bitmap Heap Scan on orders  (cost=89.69..4295.03 rows=4576 width=44)
--   Recheck Cond: (status = ANY ('{pending,processing}'))
--   ->  Bitmap Index Scan on idx_orders_status  (cost=0.00..88.55 rows=4576 width=0)
--         Index Cond: (status = ANY ('{pending,processing}'))
```

## Методы соединения (JOIN)

### Nested Loop

```sql
-- Для каждой строки внешней таблицы сканируется внутренняя
EXPLAIN SELECT u.name, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.id = 100;

-- Nested Loop  (cost=0.57..16.62 rows=1 width=72)
--   ->  Index Scan using users_pkey on users u  (cost=0.29..8.30 rows=1 width=36)
--         Index Cond: (id = 100)
--   ->  Index Scan using idx_orders_user_id on orders o  (cost=0.29..8.30 rows=1 width=40)
--         Index Cond: (user_id = 100)
```

### Hash Join

```sql
-- Строит хеш-таблицу по одной таблице, проверяет другую
EXPLAIN SELECT u.name, o.total
FROM users u
JOIN orders o ON u.id = o.user_id;

-- Hash Join  (cost=225.00..1250.00 rows=10000 width=72)
--   Hash Cond: (o.user_id = u.id)
--   ->  Seq Scan on orders o  (cost=0.00..580.00 rows=10000 width=40)
--   ->  Hash  (cost=150.00..150.00 rows=5000 width=36)
--         ->  Seq Scan on users u  (cost=0.00..150.00 rows=5000 width=36)
```

### Merge Join

```sql
-- Требует отсортированные входные данные
EXPLAIN SELECT u.name, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
ORDER BY u.id;

-- Merge Join  (cost=0.57..1015.00 rows=10000 width=72)
--   Merge Cond: (u.id = o.user_id)
--   ->  Index Scan using users_pkey on users u  (cost=0.29..350.00 rows=5000 width=36)
--   ->  Index Scan using idx_orders_user_id on orders o  (cost=0.29..500.00 rows=10000 width=40)
```

## EXPLAIN — Анализ плана

### Базовое использование

```sql
-- Только план (без выполнения)
EXPLAIN SELECT * FROM users WHERE id = 100;

-- С выполнением и реальной статистикой
EXPLAIN ANALYZE SELECT * FROM users WHERE id = 100;

-- С дополнительной информацией
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM users WHERE id = 100;
```

### Чтение EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending' AND total > 1000;

-- Seq Scan on orders  (cost=0.00..2150.00 rows=125 width=44)
--                      (actual time=0.015..15.234 rows=150 loops=1)
--   Filter: ((status = 'pending') AND (total > 1000))
--   Rows Removed by Filter: 9850
-- Planning Time: 0.123 ms
-- Execution Time: 15.456 ms
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    Расшифровка EXPLAIN                          │
│                                                                 │
│  cost=0.00..2150.00                                            │
│       │     │                                                   │
│       │     └── Общая стоимость                                │
│       └────── Стоимость до первой строки                       │
│                                                                 │
│  rows=125      ← Ожидаемое количество строк                    │
│  actual rows=150  ← Реальное количество (ANALYZE)              │
│                                                                 │
│  loops=1       ← Сколько раз выполнялся узел                   │
└─────────────────────────────────────────────────────────────────┘
```

### EXPLAIN BUFFERS

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE email = 'test@example.com';

-- Index Scan using idx_users_email on users
--   (actual time=0.025..0.027 rows=1 loops=1)
--   Index Cond: (email = 'test@example.com')
--   Buffers: shared hit=4
-- Planning Time: 0.095 ms
-- Execution Time: 0.045 ms
```

```
Buffers:
  shared hit=4     ← Страницы из кэша (хорошо!)
  shared read=10   ← Страницы с диска (I/O)
  shared written=2 ← Записано страниц
  temp read/written ← Использование временных файлов (плохо для JOIN)
```

## 5. Executor (Исполнитель)

### Volcano/Iterator модель

PostgreSQL использует модель итератора: каждый узел плана предоставляет функции `init()`, `next()`, `close()`.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Volcano Model                                │
│                                                                 │
│   ┌──────────┐                                                 │
│   │  Output  │ ← next() возвращает по одной строке             │
│   └────┬─────┘                                                 │
│        │                                                        │
│   ┌────┴─────┐                                                 │
│   │   Sort   │                                                 │
│   └────┬─────┘                                                 │
│        │                                                        │
│   ┌────┴─────┐                                                 │
│   │  Filter  │                                                 │
│   └────┬─────┘                                                 │
│        │                                                        │
│   ┌────┴─────┐                                                 │
│   │  Scan    │ ← Читает данные из таблицы                      │
│   └──────────┘                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### JIT Compilation (PostgreSQL 11+)

```sql
-- Включение JIT
SET jit = on;

EXPLAIN ANALYZE SELECT SUM(total) FROM orders WHERE status = 'completed';

-- ...
-- JIT:
--   Functions: 4
--   Options: Inlining true, Optimization true, Expressions true, Deforming true
--   Timing: Generation 0.521 ms, Inlining 2.152 ms, Optimization 8.234 ms,
--           Emission 6.123 ms, Total 17.030 ms
```

## Оптимизация запросов

### Распространённые проблемы

#### 1. Неточная статистика

```sql
-- Проблема: rows=1, actual rows=50000
EXPLAIN ANALYZE SELECT * FROM users WHERE status = 'active';

-- Решение: обновить статистику
ANALYZE users;

-- Или увеличить детализацию
ALTER TABLE users ALTER COLUMN status SET STATISTICS 1000;
ANALYZE users;
```

#### 2. Seq Scan вместо Index Scan

```sql
-- Проверить наличие индекса
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'users';

-- Создать индекс при необходимости
CREATE INDEX idx_users_status ON users(status);

-- Иногда Seq Scan оптимальнее!
-- При выборке > 10-20% таблицы индекс не помогает
```

#### 3. Плохой JOIN порядок

```sql
-- Подсказка планировщику (использовать осторожно!)
SET join_collapse_limit = 1;  -- Сохранить порядок JOIN
SET from_collapse_limit = 1;  -- Не менять подзапросы

-- Или использовать CTE для контроля
WITH filtered_users AS MATERIALIZED (
    SELECT * FROM users WHERE status = 'active'
)
SELECT * FROM filtered_users JOIN orders ON ...;
```

### Оптимизационные подсказки

```sql
-- Отключение определённых методов для тестирования
SET enable_seqscan = off;
SET enable_indexscan = off;
SET enable_hashjoin = off;
SET enable_mergejoin = off;
SET enable_nestloop = off;

-- Использовать только для диагностики!
-- Не использовать в production!
```

## Prepared Statements

### Кэширование планов

```sql
-- Подготовка запроса
PREPARE user_by_id(integer) AS
SELECT * FROM users WHERE id = $1;

-- Выполнение
EXECUTE user_by_id(100);

-- Просмотр generic плана (после 5 выполнений)
EXPLAIN EXECUTE user_by_id(100);

-- Освобождение
DEALLOCATE user_by_id;
```

### Custom vs Generic план

```sql
-- PostgreSQL решает автоматически:
-- Первые 5 выполнений: custom plan (с конкретными значениями)
-- Затем: если generic план не хуже в среднем, используется он

-- Принудительно custom план
SET plan_cache_mode = force_custom_plan;

-- Принудительно generic план
SET plan_cache_mode = force_generic_plan;
```

## Parallel Query

### Параллельное выполнение

```sql
EXPLAIN ANALYZE SELECT COUNT(*) FROM large_table WHERE status = 'active';

-- Gather  (cost=1000.00..11500.00 rows=1 width=8)
--         (actual time=50.123..150.456 rows=1 loops=1)
--   Workers Planned: 4
--   Workers Launched: 4
--   ->  Partial Aggregate  (cost=0.00..10500.00 rows=1 width=8)
--                          (actual time=45.123..45.234 rows=1 loops=5)
--         ->  Parallel Seq Scan on large_table
--             (cost=0.00..9500.00 rows=250000 width=0)
--             (actual time=0.015..35.456 rows=200000 loops=5)
--               Filter: (status = 'active')
```

### Настройки параллелизма

```sql
-- Максимум worker'ов
SHOW max_parallel_workers_per_gather;  -- default: 2

-- Порог для параллельного сканирования
SHOW min_parallel_table_scan_size;     -- default: 8MB

-- Стоимость запуска worker'а
SHOW parallel_setup_cost;              -- default: 1000
SHOW parallel_tuple_cost;              -- default: 0.1
```

## Best Practices

### 1. Регулярно обновляйте статистику

```sql
-- Автоматически через autovacuum (рекомендуется)
-- Или вручную после больших изменений
ANALYZE;
ANALYZE large_table;
```

### 2. Используйте EXPLAIN ANALYZE

```sql
-- Всегда проверяйте реальное выполнение, не только план
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT ...;
```

### 3. Избегайте функций в WHERE для индексированных столбцов

```sql
-- Плохо: индекс не используется
SELECT * FROM users WHERE LOWER(email) = 'test@example.com';

-- Хорошо: функциональный индекс
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
```

### 4. Используйте покрывающие индексы

```sql
-- Index Only Scan возможен
CREATE INDEX idx_orders_user_total ON orders(user_id) INCLUDE (total, status);

SELECT total, status FROM orders WHERE user_id = 100;
-- Не нужен доступ к таблице!
```

### 5. Ограничивайте результаты

```sql
-- Плохо для огромных результатов
SELECT * FROM logs;

-- Хорошо
SELECT * FROM logs ORDER BY created_at DESC LIMIT 100;
```

## Заключение

Понимание процесса обработки запросов позволяет:

1. **Диагностировать проблемы** производительности с помощью EXPLAIN
2. **Оптимизировать запросы**, понимая, как планировщик принимает решения
3. **Создавать правильные индексы** для ускорения доступа к данным
4. **Настраивать PostgreSQL** для конкретной нагрузки

Ключевые инструменты: `EXPLAIN`, `EXPLAIN ANALYZE`, `EXPLAIN (BUFFERS)` и понимание статистики.

---
[prev: 04-write-ahead-log](./04-write-ahead-log.md) | [next: 01-using-docker](../04-installation-and-setup/01-using-docker.md)