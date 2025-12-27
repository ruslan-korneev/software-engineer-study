# Query Planner (Планировщик запросов)

Планировщик запросов (Query Planner или Optimizer) PostgreSQL анализирует SQL-запросы и выбирает наиболее эффективный план выполнения на основе статистики и конфигурационных параметров.

## Как работает планировщик

### Этапы обработки запроса

1. **Парсинг** — преобразование SQL в дерево разбора
2. **Анализ** — проверка синтаксиса и семантики
3. **Переписывание** — применение правил (rules, views)
4. **Планирование** — выбор оптимального плана
5. **Выполнение** — исполнение плана

### Факторы, влияющие на план

- Статистика таблиц (pg_statistic)
- Конфигурационные параметры стоимости
- Доступные индексы
- Размеры таблиц
- Настройки планировщика

## Параметры стоимости операций

### Стоимость доступа к диску

```ini
# postgresql.conf

# Стоимость последовательного чтения страницы
seq_page_cost = 1.0

# Стоимость случайного чтения (важно для индексов)
random_page_cost = 4.0    # HDD
random_page_cost = 1.1    # SSD

# Стоимость обработки кортежа
cpu_tuple_cost = 0.01

# Стоимость обработки индексной записи
cpu_index_tuple_cost = 0.005

# Стоимость вычисления оператора
cpu_operator_cost = 0.0025
```

### Стоимость параллельных операций

```ini
# Стоимость запуска параллельного воркера
parallel_setup_cost = 1000.0

# Стоимость передачи кортежа между процессами
parallel_tuple_cost = 0.1
```

### Настройка для SSD

```ini
# Оптимизация для SSD/NVMe
random_page_cost = 1.1        # близко к seq_page_cost
effective_io_concurrency = 200
```

## Оценка размера результата

### effective_cache_size

Помогает планировщику оценить, сколько данных поместится в кэш.

```ini
# Обычно 50-75% от общей RAM
effective_cache_size = 12GB
```

```sql
-- Влияние на выбор плана
SET effective_cache_size = '256MB';
EXPLAIN SELECT * FROM large_table WHERE indexed_column = 100;

SET effective_cache_size = '8GB';
EXPLAIN SELECT * FROM large_table WHERE indexed_column = 100;
-- При большем кэше планировщик охотнее использует индексы
```

### default_statistics_target

Детализация статистики для оценки селективности.

```sql
-- Глобальная настройка (по умолчанию 100)
SET default_statistics_target = 100;

-- Увеличение для конкретного столбца
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;

-- После изменения нужен ANALYZE
ANALYZE orders;
```

## Управление методами соединения

### Типы соединений

```sql
-- Nested Loop Join — для малых таблиц или индексного доступа
enable_nestloop = on

-- Hash Join — для больших таблиц без индексов
enable_hashjoin = on

-- Merge Join — для отсортированных данных
enable_mergejoin = on
```

### Отключение методов для отладки

```sql
-- Временное отключение для анализа
SET enable_hashjoin = off;
EXPLAIN ANALYZE SELECT * FROM a JOIN b ON a.id = b.a_id;

-- Сброс настроек
RESET enable_hashjoin;
```

## Управление методами сканирования

```sql
-- Последовательное сканирование
enable_seqscan = on

-- Сканирование по индексу
enable_indexscan = on

-- Сканирование только по индексу
enable_indexonlyscan = on

-- Bitmap сканирование
enable_bitmapscan = on

-- TID сканирование
enable_tidscan = on
```

```sql
-- Пример: заставить использовать индекс
SET enable_seqscan = off;
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
```

## Параллельное выполнение запросов

### Параметры параллелизма

```ini
# postgresql.conf

# Максимум воркеров на запрос
max_parallel_workers_per_gather = 2

# Общий лимит параллельных воркеров
max_parallel_workers = 8

# Минимальный размер таблицы для параллелизма
min_parallel_table_scan_size = 8MB

# Минимальный размер индекса для параллелизма
min_parallel_index_scan_size = 512kB
```

### Управление параллелизмом для таблиц

```sql
-- Установка параллельных воркеров для таблицы
ALTER TABLE large_table SET (parallel_workers = 4);

-- Отключение параллелизма для таблицы
ALTER TABLE small_table SET (parallel_workers = 0);
```

```sql
-- Пример параллельного плана
EXPLAIN ANALYZE
SELECT count(*) FROM large_table WHERE status = 'active';

-- Gather
--   Workers Planned: 2
--   Workers Launched: 2
--   ->  Parallel Seq Scan on large_table
```

## Статистика и ANALYZE

### Сбор статистики

```sql
-- ANALYZE конкретной таблицы
ANALYZE users;

-- ANALYZE конкретных столбцов
ANALYZE users (email, created_at);

-- ANALYZE всей базы
ANALYZE;

-- ANALYZE с выводом информации
ANALYZE VERBOSE users;
```

### Просмотр статистики

```sql
-- Статистика таблицы
SELECT
    relname,
    reltuples AS row_estimate,
    relpages AS pages,
    pg_size_pretty(pg_relation_size(oid)) AS size
FROM pg_class
WHERE relname = 'users';

-- Детальная статистика столбца
SELECT
    attname,
    null_frac,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    histogram_bounds
FROM pg_stats
WHERE tablename = 'users' AND attname = 'status';
```

### Расширенная статистика

```sql
-- Создание расширенной статистики для коррелированных столбцов
CREATE STATISTICS stats_city_country ON city, country FROM addresses;

-- Типы статистики
CREATE STATISTICS stats_combo (dependencies, ndistinct, mcv)
ON column1, column2 FROM table_name;

-- Обновление расширенной статистики
ANALYZE addresses;

-- Просмотр
SELECT * FROM pg_statistic_ext;
```

## EXPLAIN и анализ планов

### Базовый EXPLAIN

```sql
-- Показать план без выполнения
EXPLAIN SELECT * FROM users WHERE id = 1;

-- Показать план с выполнением и реальным временем
EXPLAIN ANALYZE SELECT * FROM users WHERE id = 1;

-- Подробный вывод
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM users WHERE id = 1;
```

### Форматы вывода

```sql
-- JSON формат (удобно для парсинга)
EXPLAIN (FORMAT JSON) SELECT * FROM users;

-- YAML формат
EXPLAIN (FORMAT YAML) SELECT * FROM users;

-- XML формат
EXPLAIN (FORMAT XML) SELECT * FROM users;
```

### Анализ с буферами и временем

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING, COSTS)
SELECT u.*, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.created_at > '2024-01-01';

-- Вывод покажет:
-- Planning Time: 0.5 ms
-- Execution Time: 10.2 ms
-- Buffers: shared hit=100 read=20
```

## Генетический оптимизатор (GEQO)

Для запросов с большим количеством JOIN используется генетический алгоритм.

```ini
# postgresql.conf

# Включение GEQO
geqo = on

# Порог количества таблиц для включения GEQO
geqo_threshold = 12

# Параметры генетического алгоритма
geqo_effort = 5           # 1-10, баланс скорость/качество
geqo_pool_size = 0        # размер популяции (0 = авто)
geqo_generations = 0      # количество поколений (0 = авто)
```

## Параметры JIT-компиляции

```ini
# Включение JIT
jit = on

# Пороги для использования JIT
jit_above_cost = 100000           # минимальная стоимость для JIT
jit_inline_above_cost = 500000    # порог для инлайнинга
jit_optimize_above_cost = 500000  # порог для оптимизации
```

```sql
-- Проверка JIT в плане
EXPLAIN (ANALYZE, BUFFERS)
SELECT sum(amount) FROM large_transactions;

-- JIT:
--   Functions: 4
--   Options: Inlining true, Optimization true, Expressions true
--   Timing: Generation 1.2 ms, Inlining 3.4 ms, Optimization 20.5 ms
```

## Подсказки планировщику (через CTE и подзапросы)

PostgreSQL не поддерживает хинты напрямую, но есть обходные пути:

```sql
-- Материализация CTE (PostgreSQL 12+)
WITH orders_cte AS MATERIALIZED (
    SELECT * FROM orders WHERE status = 'pending'
)
SELECT * FROM orders_cte WHERE total > 100;

-- Без материализации
WITH orders_cte AS NOT MATERIALIZED (
    SELECT * FROM orders WHERE status = 'pending'
)
SELECT * FROM orders_cte WHERE total > 100;
```

## Типичные ошибки

1. **Устаревшая статистика** — планировщик использует неверные оценки
2. **random_page_cost = 4 на SSD** — избегание индексов
3. **Слишком низкий work_mem** — переход на дисковые сортировки
4. **Игнорирование EXPLAIN ANALYZE** — оптимизация вслепую
5. **Неоптимальный default_statistics_target** — плохие оценки селективности

## Best Practices

1. **Регулярно запускайте ANALYZE** — autovacuum делает это, но после массовых изменений нужен ручной запуск
2. **Настройте random_page_cost** для вашего типа хранилища
3. **Используйте EXPLAIN ANALYZE BUFFERS** для анализа проблемных запросов
4. **Увеличьте statistics_target** для столбцов с неравномерным распределением
5. **Создавайте расширенную статистику** для коррелированных столбцов
6. **Не отключайте методы сканирования** в production без веских причин
7. **Мониторьте время планирования** для сложных запросов

```sql
-- Полезный запрос для анализа медленных запросов
SELECT
    query,
    calls,
    mean_exec_time,
    mean_plan_time,
    rows / calls as avg_rows
FROM pg_stat_statements
WHERE mean_exec_time > 1000  -- более 1 секунды
ORDER BY mean_exec_time DESC
LIMIT 20;
```
