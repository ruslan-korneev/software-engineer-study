# Рекурсивные CTE в PostgreSQL

[prev: 02-triggers](./02-triggers.md) | [next: 04-aggregate-window-functions](./04-aggregate-window-functions.md)

---

## Введение

CTE (Common Table Expression) — это именованный временный результирующий набор, существующий в пределах одного запроса. Рекурсивные CTE позволяют выполнять запросы, которые ссылаются сами на себя, что идеально подходит для работы с иерархическими и графовыми структурами данных.

## Синтаксис рекурсивного CTE

```sql
WITH RECURSIVE cte_name AS (
    -- Базовая часть (anchor member)
    -- Начальный набор данных
    SELECT ...

    UNION [ALL]

    -- Рекурсивная часть (recursive member)
    -- Ссылается на cte_name
    SELECT ...
    FROM cte_name
    WHERE ...  -- Условие остановки
)
SELECT * FROM cte_name;
```

### Как это работает

1. Выполняется базовая часть — получаем начальный набор строк
2. Выполняется рекурсивная часть с текущими строками
3. Если рекурсивная часть вернула строки, они добавляются к результату
4. Шаги 2-3 повторяются пока рекурсивная часть возвращает строки
5. Возвращается объединённый результат

## Базовые примеры

### Генерация последовательности чисел

```sql
WITH RECURSIVE numbers AS (
    -- Базовая часть: начинаем с 1
    SELECT 1 AS n

    UNION ALL

    -- Рекурсивная часть: добавляем 1
    SELECT n + 1
    FROM numbers
    WHERE n < 10  -- Условие остановки
)
SELECT n FROM numbers;

-- Результат: 1, 2, 3, 4, 5, 6, 7, 8, 9, 10
```

### Генерация дат

```sql
WITH RECURSIVE date_series AS (
    SELECT DATE '2024-01-01' AS date

    UNION ALL

    SELECT date + INTERVAL '1 day'
    FROM date_series
    WHERE date < '2024-01-31'
)
SELECT date FROM date_series;
```

### Факториал

```sql
WITH RECURSIVE factorial AS (
    SELECT 1 AS n, 1::BIGINT AS fact

    UNION ALL

    SELECT n + 1, fact * (n + 1)
    FROM factorial
    WHERE n < 20
)
SELECT n, fact FROM factorial;
```

### Числа Фибоначчи

```sql
WITH RECURSIVE fibonacci AS (
    SELECT 1 AS n, 0::BIGINT AS fib, 1::BIGINT AS next_fib

    UNION ALL

    SELECT n + 1, next_fib, fib + next_fib
    FROM fibonacci
    WHERE n < 50
)
SELECT n, fib FROM fibonacci;
```

## Работа с иерархическими данными

### Структура таблицы сотрудников

```sql
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    manager_id INTEGER REFERENCES employees(id),
    department VARCHAR(50),
    salary NUMERIC
);

INSERT INTO employees (id, name, manager_id, department, salary) VALUES
(1, 'CEO', NULL, 'Executive', 500000),
(2, 'CTO', 1, 'Technology', 300000),
(3, 'CFO', 1, 'Finance', 300000),
(4, 'Dev Manager', 2, 'Technology', 150000),
(5, 'QA Manager', 2, 'Technology', 140000),
(6, 'Senior Dev', 4, 'Technology', 100000),
(7, 'Junior Dev', 4, 'Technology', 60000),
(8, 'QA Engineer', 5, 'Technology', 70000),
(9, 'Accountant', 3, 'Finance', 80000);
```

### Получение всех подчинённых

```sql
WITH RECURSIVE subordinates AS (
    -- Базовая часть: находим начального сотрудника
    SELECT id, name, manager_id, 1 AS level
    FROM employees
    WHERE id = 2  -- CTO

    UNION ALL

    -- Рекурсивная часть: находим подчинённых
    SELECT e.id, e.name, e.manager_id, s.level + 1
    FROM employees e
    INNER JOIN subordinates s ON e.manager_id = s.id
)
SELECT id, name, level
FROM subordinates
ORDER BY level, name;

-- Результат:
-- 2  | CTO         | 1
-- 4  | Dev Manager | 2
-- 5  | QA Manager  | 2
-- 6  | Senior Dev  | 3
-- 7  | Junior Dev  | 3
-- 8  | QA Engineer | 3
```

### Построение полного пути в иерархии

```sql
WITH RECURSIVE hierarchy AS (
    SELECT
        id,
        name,
        manager_id,
        name AS path,
        ARRAY[id] AS path_ids,
        1 AS depth
    FROM employees
    WHERE manager_id IS NULL  -- Начинаем с корня

    UNION ALL

    SELECT
        e.id,
        e.name,
        e.manager_id,
        h.path || ' > ' || e.name,
        h.path_ids || e.id,
        h.depth + 1
    FROM employees e
    INNER JOIN hierarchy h ON e.manager_id = h.id
)
SELECT id, name, path, depth
FROM hierarchy
ORDER BY path;

-- Результат:
-- 1 | CEO         | CEO                                    | 1
-- 3 | CFO         | CEO > CFO                              | 2
-- 9 | Accountant  | CEO > CFO > Accountant                 | 3
-- 2 | CTO         | CEO > CTO                              | 2
-- 4 | Dev Manager | CEO > CTO > Dev Manager                | 3
-- 6 | Senior Dev  | CEO > CTO > Dev Manager > Senior Dev   | 4
-- 7 | Junior Dev  | CEO > CTO > Dev Manager > Junior Dev   | 4
-- 5 | QA Manager  | CEO > CTO > QA Manager                 | 3
-- 8 | QA Engineer | CEO > CTO > QA Manager > QA Engineer   | 4
```

### Поиск всех предков (путь к корню)

```sql
WITH RECURSIVE ancestors AS (
    -- Начинаем с целевого сотрудника
    SELECT id, name, manager_id, 0 AS level
    FROM employees
    WHERE id = 7  -- Junior Dev

    UNION ALL

    -- Поднимаемся к руководителям
    SELECT e.id, e.name, e.manager_id, a.level + 1
    FROM employees e
    INNER JOIN ancestors a ON e.id = a.manager_id
)
SELECT id, name, level
FROM ancestors
ORDER BY level DESC;

-- Результат (от корня к листу):
-- 1 | CEO         | 3
-- 2 | CTO         | 2
-- 4 | Dev Manager | 1
-- 7 | Junior Dev  | 0
```

### Подсчёт подчинённых и суммы зарплат

```sql
WITH RECURSIVE team_stats AS (
    SELECT
        id,
        name,
        manager_id,
        salary,
        1 AS team_size,
        salary AS team_salary
    FROM employees

    UNION ALL

    SELECT
        e.id,
        e.name,
        e.manager_id,
        e.salary,
        ts.team_size + 1,
        ts.team_salary + e.salary
    FROM employees e
    INNER JOIN team_stats ts ON ts.manager_id = e.id
)
SELECT
    id,
    name,
    MAX(team_size) - 1 AS subordinates_count,  -- -1 чтобы исключить самого себя
    MAX(team_salary) - salary AS subordinates_salary
FROM team_stats
JOIN employees USING (id)
GROUP BY id, name, salary
ORDER BY subordinates_count DESC;
```

## Работа с графами

### Структура графа

```sql
CREATE TABLE graph_edges (
    source INTEGER,
    target INTEGER,
    weight NUMERIC DEFAULT 1,
    PRIMARY KEY (source, target)
);

INSERT INTO graph_edges VALUES
(1, 2, 10), (1, 3, 5),
(2, 3, 2), (2, 4, 1),
(3, 2, 3), (3, 4, 9), (3, 5, 2),
(4, 5, 4),
(5, 4, 6), (5, 1, 7);
```

### Поиск всех достижимых вершин

```sql
WITH RECURSIVE reachable AS (
    SELECT
        target AS node,
        ARRAY[source, target] AS path,
        weight AS total_weight
    FROM graph_edges
    WHERE source = 1

    UNION

    SELECT
        e.target,
        r.path || e.target,
        r.total_weight + e.weight
    FROM graph_edges e
    INNER JOIN reachable r ON e.source = r.node
    WHERE NOT e.target = ANY(r.path)  -- Избегаем циклов
)
SELECT DISTINCT node, path, total_weight
FROM reachable
ORDER BY total_weight;
```

### Поиск кратчайшего пути (Упрощённый Dijkstra)

```sql
WITH RECURSIVE shortest_paths AS (
    -- Начальная точка
    SELECT
        1 AS node,
        0::NUMERIC AS distance,
        ARRAY[1] AS path

    UNION ALL

    SELECT DISTINCT ON (e.target)
        e.target,
        sp.distance + e.weight,
        sp.path || e.target
    FROM shortest_paths sp
    JOIN graph_edges e ON e.source = sp.node
    WHERE NOT e.target = ANY(sp.path)
    ORDER BY e.target, sp.distance + e.weight
)
SELECT DISTINCT ON (node)
    node, distance, path
FROM shortest_paths
ORDER BY node, distance;
```

### Обнаружение циклов

```sql
WITH RECURSIVE cycle_detection AS (
    SELECT
        source,
        target,
        ARRAY[source] AS path,
        FALSE AS has_cycle
    FROM graph_edges
    WHERE source = 1

    UNION ALL

    SELECT
        cd.source,
        e.target,
        cd.path || e.target,
        e.target = ANY(cd.path)
    FROM graph_edges e
    INNER JOIN cycle_detection cd ON e.source = cd.target
    WHERE NOT cd.has_cycle
)
SELECT path, has_cycle
FROM cycle_detection
WHERE has_cycle;
```

## Практические примеры

### Категории и подкатегории

```sql
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    parent_id INTEGER REFERENCES categories(id)
);

-- Получить полную иерархию с отступами
WITH RECURSIVE category_tree AS (
    SELECT
        id,
        name,
        parent_id,
        0 AS depth,
        name::TEXT AS full_path,
        ARRAY[id] AS path_array
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    SELECT
        c.id,
        c.name,
        c.parent_id,
        ct.depth + 1,
        ct.full_path || ' / ' || c.name,
        ct.path_array || c.id
    FROM categories c
    INNER JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT
    REPEAT('  ', depth) || name AS indented_name,
    full_path,
    depth
FROM category_tree
ORDER BY path_array;
```

### Комментарии с ответами

```sql
CREATE TABLE comments (
    id SERIAL PRIMARY KEY,
    post_id INTEGER,
    parent_id INTEGER REFERENCES comments(id),
    author VARCHAR(100),
    content TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Получить дерево комментариев
WITH RECURSIVE comment_tree AS (
    SELECT
        id,
        post_id,
        parent_id,
        author,
        content,
        created_at,
        0 AS depth,
        ARRAY[id] AS path
    FROM comments
    WHERE parent_id IS NULL AND post_id = 1

    UNION ALL

    SELECT
        c.id,
        c.post_id,
        c.parent_id,
        c.author,
        c.content,
        c.created_at,
        ct.depth + 1,
        ct.path || c.id
    FROM comments c
    INNER JOIN comment_tree ct ON c.parent_id = ct.id
)
SELECT
    REPEAT('  ', depth) || author || ': ' ||
        LEFT(content, 50) AS formatted_comment,
    depth,
    created_at
FROM comment_tree
ORDER BY path;
```

### Bill of Materials (Структура изделия)

```sql
CREATE TABLE parts (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    cost NUMERIC
);

CREATE TABLE assemblies (
    parent_id INTEGER REFERENCES parts(id),
    child_id INTEGER REFERENCES parts(id),
    quantity INTEGER,
    PRIMARY KEY (parent_id, child_id)
);

-- Рассчитать полную стоимость изделия
WITH RECURSIVE bom AS (
    -- Базовая часть: корневое изделие
    SELECT
        p.id,
        p.name,
        p.cost,
        1 AS quantity,
        0 AS level,
        ARRAY[p.id] AS path
    FROM parts p
    WHERE p.id = 1  -- ID главного изделия

    UNION ALL

    -- Рекурсивная часть: компоненты
    SELECT
        p.id,
        p.name,
        p.cost,
        a.quantity * b.quantity,  -- Умножаем количество
        b.level + 1,
        b.path || p.id
    FROM assemblies a
    JOIN parts p ON p.id = a.child_id
    JOIN bom b ON b.id = a.parent_id
    WHERE NOT p.id = ANY(b.path)  -- Защита от циклов
)
SELECT
    REPEAT('  ', level) || name AS component,
    quantity,
    cost,
    cost * quantity AS total_cost
FROM bom
ORDER BY path;

-- Общая стоимость изделия
SELECT SUM(cost * quantity) AS total_product_cost
FROM bom
WHERE level > 0;  -- Исключаем само изделие
```

### Разбор JSON-дерева

```sql
WITH RECURSIVE json_tree AS (
    SELECT
        key,
        value,
        0 AS depth,
        key AS path
    FROM jsonb_each('{"a": {"b": {"c": 1}}, "d": [1, 2, 3]}'::JSONB)

    UNION ALL

    SELECT
        COALESCE(kv.key, elem.idx::TEXT),
        COALESCE(kv.value, elem.value),
        jt.depth + 1,
        jt.path || '.' || COALESCE(kv.key, elem.idx::TEXT)
    FROM json_tree jt
    LEFT JOIN LATERAL jsonb_each(
        CASE WHEN jsonb_typeof(jt.value) = 'object' THEN jt.value END
    ) kv ON TRUE
    LEFT JOIN LATERAL jsonb_array_elements(
        CASE WHEN jsonb_typeof(jt.value) = 'array' THEN jt.value END
    ) WITH ORDINALITY AS elem(value, idx) ON TRUE
    WHERE kv.key IS NOT NULL OR elem.value IS NOT NULL
)
SELECT path, value, depth
FROM json_tree
ORDER BY path;
```

## UNION vs UNION ALL

```sql
-- UNION ALL: включает дубликаты, быстрее
-- Используйте когда уверены в уникальности

-- UNION: удаляет дубликаты, медленнее
-- Используйте для графов с возможными циклами

-- Пример различия:
WITH RECURSIVE test AS (
    SELECT 1 AS n
    UNION ALL  -- С ALL может быть бесконечный цикл!
    SELECT 1 FROM test WHERE n < 5
)
SELECT * FROM test;
-- ВНИМАНИЕ: Бесконечный цикл!

-- Правильно:
WITH RECURSIVE test AS (
    SELECT 1 AS n
    UNION  -- Дубликаты удаляются, рекурсия остановится
    SELECT 1 FROM test WHERE n < 5
)
SELECT * FROM test;
-- Результат: 1 (только одна строка)
```

## Ограничение глубины рекурсии

```sql
-- Через условие в WHERE
WITH RECURSIVE limited AS (
    SELECT 1 AS n, 1 AS depth
    UNION ALL
    SELECT n + 1, depth + 1
    FROM limited
    WHERE depth < 100  -- Ограничение глубины
)
SELECT * FROM limited;

-- Через LIMIT (PostgreSQL 14+)
WITH RECURSIVE nums AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM nums
)
SELECT * FROM nums LIMIT 100;

-- Глобальная настройка
SET max_recursive_iterations = 1000;  -- Не существует в PostgreSQL!
-- PostgreSQL не имеет такой настройки, используйте условия в запросе
```

## Оптимизация рекурсивных CTE

### Использование индексов

```sql
-- Создайте индексы на столбцах, используемых в JOIN
CREATE INDEX idx_employees_manager ON employees(manager_id);
CREATE INDEX idx_graph_source ON graph_edges(source);
```

### MATERIALIZED CTE (PostgreSQL 12+)

```sql
WITH RECURSIVE subtree AS MATERIALIZED (
    SELECT id, manager_id FROM employees WHERE id = 1
    UNION ALL
    SELECT e.id, e.manager_id
    FROM employees e
    JOIN subtree s ON e.manager_id = s.id
)
SELECT * FROM subtree;
```

### Avoid SELECT *

```sql
-- Плохо: выбираем все столбцы
WITH RECURSIVE r AS (
    SELECT * FROM large_table WHERE id = 1
    UNION ALL
    SELECT lt.* FROM large_table lt JOIN r ON lt.parent = r.id
)
SELECT * FROM r;

-- Хорошо: только нужные столбцы
WITH RECURSIVE r AS (
    SELECT id, parent, name FROM large_table WHERE id = 1
    UNION ALL
    SELECT lt.id, lt.parent, lt.name
    FROM large_table lt JOIN r ON lt.parent = r.id
)
SELECT * FROM r;
```

## Типичные ошибки

### 1. Бесконечная рекурсия

```sql
-- Проблема: нет условия остановки
WITH RECURSIVE infinite AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM infinite  -- Будет выполняться вечно!
)
SELECT * FROM infinite;

-- Решение: добавить WHERE
WHERE n < 1000
```

### 2. Неправильное использование UNION

```sql
-- Проблема: UNION ALL в графе с циклами
WITH RECURSIVE traverse AS (
    SELECT 1 AS node, ARRAY[1] AS path
    UNION ALL  -- Цикл 1->2->3->1 вызовет бесконечность
    SELECT target, path || target
    FROM graph_edges g
    JOIN traverse t ON g.source = t.node
)
SELECT * FROM traverse;

-- Решение: проверка на цикл
WHERE NOT target = ANY(path)
```

### 3. Производительность при больших данных

```sql
-- Проблема: рекурсия обрабатывает миллионы строк
-- Решение: используйте LIMIT или раннюю фильтрацию
WITH RECURSIVE r AS (...)
SELECT * FROM r
LIMIT 1000;  -- Ограничиваем результат
```

## Best Practices

1. **Всегда добавляйте условие остановки** в рекурсивной части
2. **Используйте массивы для отслеживания пути** и предотвращения циклов
3. **Добавляйте счётчик глубины** для отладки и ограничения
4. **Создавайте индексы** на столбцах, участвующих в JOIN
5. **Выбирайте только нужные столбцы** для производительности
6. **Тестируйте на небольших данных** перед запуском на production
7. **Используйте EXPLAIN ANALYZE** для анализа плана выполнения

## Заключение

Рекурсивные CTE — мощный инструмент для работы с иерархическими структурами (деревья, графы, организационные структуры). Они позволяют элегантно решать задачи, которые без рекурсии потребовали бы множественных запросов или процедурного кода. Ключевые моменты: всегда предусматривайте условие остановки, защищайтесь от циклов и оптимизируйте для больших объёмов данных.

---

[prev: 02-triggers](./02-triggers.md) | [next: 04-aggregate-window-functions](./04-aggregate-window-functions.md)
