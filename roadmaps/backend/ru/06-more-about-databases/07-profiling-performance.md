# Профилирование производительности

## Зачем нужно профилирование

Профилирование производительности базы данных — это процесс анализа и измерения работы СУБД для выявления узких мест и оптимизации запросов. Без профилирования невозможно понять, почему приложение работает медленно.

**Основные цели профилирования:**
- Выявление медленных запросов
- Обнаружение неэффективного использования индексов
- Анализ потребления ресурсов (CPU, память, I/O)
- Планирование масштабирования
- Предотвращение деградации производительности

---

## Метрики производительности БД

### Ключевые метрики

| Метрика | Описание | Нормальное значение |
|---------|----------|---------------------|
| **Query time** | Время выполнения запроса | < 100ms для OLTP |
| **Throughput** | Количество запросов в секунду (QPS) | Зависит от нагрузки |
| **Connections** | Активные соединения | < max_connections |
| **Cache hit ratio** | Процент попаданий в кэш | > 99% |
| **Lock waits** | Время ожидания блокировок | Минимальное |
| **Disk I/O** | Операции чтения/записи | Зависит от железа |

### Проверка cache hit ratio в PostgreSQL

```sql
-- Проверка эффективности буферного кэша
SELECT
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit) as heap_hit,
    round(sum(heap_blks_hit) /
          (sum(heap_blks_hit) + sum(heap_blks_read))::numeric * 100, 2)
          as cache_hit_ratio
FROM pg_statio_user_tables;
```

### Активные соединения

```sql
-- Количество активных соединений по состоянию
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;

-- Детали активных запросов
SELECT pid, usename, application_name,
       state, query, query_start,
       now() - query_start as duration
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;
```

---

## Инструменты профилирования

### 1. EXPLAIN / EXPLAIN ANALYZE

**EXPLAIN** показывает план выполнения запроса без его выполнения:

```sql
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
```

**EXPLAIN ANALYZE** выполняет запрос и показывает реальное время:

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
```

**Полезные опции:**

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders
WHERE created_at > '2024-01-01'
  AND status = 'completed';
```

- `ANALYZE` — выполняет запрос и показывает реальное время
- `BUFFERS` — показывает использование буферов (shared hit/read)
- `FORMAT` — формат вывода (TEXT, JSON, XML, YAML)

### 2. pg_stat_statements

Расширение для сбора статистики по всем выполненным запросам.

**Включение:**

```sql
-- В postgresql.conf добавить:
-- shared_preload_libraries = 'pg_stat_statements'

-- Создание расширения
CREATE EXTENSION pg_stat_statements;
```

**Анализ медленных запросов:**

```sql
-- Топ-10 запросов по общему времени выполнения
SELECT
    substring(query, 1, 100) as query,
    calls,
    round(total_exec_time::numeric, 2) as total_time_ms,
    round(mean_exec_time::numeric, 2) as avg_time_ms,
    round((100 * total_exec_time /
           sum(total_exec_time) OVER ())::numeric, 2) as percentage
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Запросы с высоким временем на вызов
SELECT
    substring(query, 1, 100) as query,
    calls,
    round(mean_exec_time::numeric, 2) as avg_time_ms,
    rows / calls as avg_rows
FROM pg_stat_statements
WHERE calls > 100
ORDER BY mean_exec_time DESC
LIMIT 10;
```

### 3. Query Logs

**Настройка логирования медленных запросов в postgresql.conf:**

```conf
# Логировать запросы дольше 1 секунды
log_min_duration_statement = 1000

# Логировать все запросы (для отладки)
log_statement = 'all'

# Логировать использование временных файлов
log_temp_files = 0

# Включить статистику в лог
log_duration = on
```

**Анализ логов с помощью pgBadger:**

```bash
# Установка
apt-get install pgbadger

# Генерация отчёта
pgbadger /var/log/postgresql/postgresql-*.log -o report.html
```

### 4. Инструменты мониторинга

**pgAdmin** — встроенная статистика:
- Dashboard с метриками сервера
- Query Tool с EXPLAIN ANALYZE
- Server Activity для анализа активных сессий

**pg_top** — top для PostgreSQL:

```bash
pg_top -U postgres -d mydb
```

**Prometheus + Grafana:**
- `postgres_exporter` для сбора метрик
- Готовые дашборды для визуализации

**Коммерческие решения:**
- Datadog
- New Relic
- pganalyze

---

## Анализ плана выполнения запроса

### Структура вывода EXPLAIN

```sql
EXPLAIN ANALYZE
SELECT u.name, count(o.id) as order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id, u.name;
```

**Пример вывода:**

```
HashAggregate  (cost=245.50..247.50 rows=200 width=44)
               (actual time=5.234..5.456 rows=150 loops=1)
   Group Key: u.id, u.name
   ->  Hash Right Join  (cost=12.50..240.00 rows=1100 width=36)
                        (actual time=0.456..4.123 rows=1200 loops=1)
         Hash Cond: (o.user_id = u.id)
         ->  Seq Scan on orders o  (cost=0.00..185.00 rows=10000 width=8)
                                   (actual time=0.012..1.234 rows=10000 loops=1)
         ->  Hash  (cost=10.00..10.00 rows=200 width=36)
                   (actual time=0.345..0.345 rows=150 loops=1)
               Buckets: 1024  Batches: 1  Memory Usage: 15kB
               ->  Index Scan using idx_users_created_at on users u
                   (cost=0.29..10.00 rows=200 width=36)
                   (actual time=0.023..0.234 rows=150 loops=1)
                     Index Cond: (created_at > '2024-01-01'::date)
Planning Time: 0.234 ms
Execution Time: 5.567 ms
```

### Как читать план

**Читаем снизу вверх и изнутри наружу:**

1. **Самый глубокий уровень** — начальные операции
2. **Стоимость (cost)** — `startup..total` в условных единицах
3. **Rows** — ожидаемое vs реальное количество строк
4. **Width** — средний размер строки в байтах
5. **Actual time** — реальное время выполнения в мс
6. **Loops** — количество повторений операции

### Типы сканирования

| Тип | Описание | Когда используется |
|-----|----------|-------------------|
| **Seq Scan** | Последовательное чтение всей таблицы | Нет подходящего индекса или выбирается много строк |
| **Index Scan** | Чтение через индекс + обращение к таблице | Есть подходящий индекс |
| **Index Only Scan** | Чтение только из индекса | Все нужные столбцы есть в индексе |
| **Bitmap Scan** | Построение bitmap + чтение | Средняя селективность |

### Типы соединений

| Тип | Описание | Лучше всего для |
|-----|----------|-----------------|
| **Nested Loop** | Для каждой строки внешней таблицы сканируем внутреннюю | Маленькие таблицы, хорошие индексы |
| **Hash Join** | Строим хэш-таблицу, затем сканируем | Большие таблицы без индексов |
| **Merge Join** | Сортируем обе таблицы, затем сливаем | Отсортированные данные |

---

## Типичные проблемы производительности

### 1. Медленные запросы

**Признаки:**
- Высокое значение `total_exec_time` в pg_stat_statements
- Запросы в логе с `duration > 1s`

**Пример проблемного запроса:**

```sql
-- Без индекса — Seq Scan
SELECT * FROM orders WHERE customer_email = 'user@example.com';
```

```
Seq Scan on orders  (cost=0.00..25000.00 rows=1 width=200)
                    (actual time=450.123..890.456 rows=1 loops=1)
  Filter: (customer_email = 'user@example.com'::text)
  Rows Removed by Filter: 999999
```

### 2. Отсутствие индексов

**Выявление неиспользуемых индексов:**

```sql
-- Индексы, которые не используются
SELECT
    schemaname, tablename, indexname,
    idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

**Выявление таблиц без индексов:**

```sql
-- Таблицы с высоким количеством Seq Scan
SELECT
    schemaname, relname,
    seq_scan, seq_tup_read,
    idx_scan, idx_tup_fetch,
    round(seq_tup_read::numeric / nullif(seq_scan, 0), 0) as avg_seq_rows
FROM pg_stat_user_tables
WHERE seq_scan > 100
ORDER BY seq_tup_read DESC
LIMIT 20;
```

### 3. Full Table Scans

**Проблема:** PostgreSQL читает всю таблицу вместо использования индекса.

**Причины:**
- Нет подходящего индекса
- Статистика устарела
- Запрос возвращает большую часть таблицы
- Неправильный тип данных в условии

```sql
-- Обновление статистики
ANALYZE orders;

-- Обновление статистики всей базы
VACUUM ANALYZE;
```

### 4. Неоптимальные JOIN

**Проблема N+1:**

```sql
-- Плохо: запрос в цикле
FOR user IN SELECT * FROM users LOOP
    SELECT * FROM orders WHERE user_id = user.id;  -- Выполняется N раз!
END LOOP;

-- Хорошо: один запрос с JOIN
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;
```

**Cartesian Product (декартово произведение):**

```sql
-- Плохо: забыли условие JOIN
SELECT * FROM users, orders;  -- users * orders строк!

-- Хорошо: явный JOIN
SELECT * FROM users u
JOIN orders o ON o.user_id = u.id;
```

---

## Оптимизация запросов — примеры

### Пример 1: Добавление индекса

**До оптимизации:**

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'pending' AND created_at > '2024-01-01';
```

```
Seq Scan on orders  (cost=0.00..35000.00 rows=5000 width=200)
                    (actual time=0.045..890.123 rows=4532 loops=1)
  Filter: ((status = 'pending') AND (created_at > '2024-01-01'))
  Rows Removed by Filter: 995468
Execution Time: 892.456 ms
```

**Создание индекса:**

```sql
CREATE INDEX idx_orders_status_created
ON orders(status, created_at);
```

**После оптимизации:**

```
Index Scan using idx_orders_status_created on orders
    (cost=0.43..156.78 rows=5000 width=200)
    (actual time=0.034..12.345 rows=4532 loops=1)
  Index Cond: ((status = 'pending') AND (created_at > '2024-01-01'))
Execution Time: 14.567 ms
```

**Улучшение:** с 892ms до 14ms (в 64 раза быстрее)

### Пример 2: Покрывающий индекс

**До:** Index Scan + Heap Fetch

```sql
-- Запрос
SELECT id, email FROM users WHERE email LIKE 'admin%';

-- Создаём покрывающий индекс
CREATE INDEX idx_users_email_id ON users(email) INCLUDE (id);
```

**После:** Index Only Scan (без обращения к таблице)

```
Index Only Scan using idx_users_email_id on users
    (cost=0.43..45.67 rows=100 width=40)
    (actual time=0.023..0.234 rows=87 loops=1)
  Index Cond: ((email >= 'admin') AND (email < 'admio'))
  Heap Fetches: 0
```

### Пример 3: Оптимизация LIKE

```sql
-- Плохо: LIKE с % в начале — не использует B-tree индекс
SELECT * FROM products WHERE name LIKE '%phone%';

-- Решение 1: pg_trgm для полнотекстового поиска
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_products_name_trgm ON products
USING gin (name gin_trgm_ops);

-- Решение 2: Полнотекстовый поиск
ALTER TABLE products ADD COLUMN name_tsv tsvector;
UPDATE products SET name_tsv = to_tsvector('russian', name);
CREATE INDEX idx_products_name_fts ON products USING gin(name_tsv);

SELECT * FROM products
WHERE name_tsv @@ to_tsquery('russian', 'телефон');
```

### Пример 4: Оптимизация подзапросов

```sql
-- Плохо: подзапрос выполняется для каждой строки
SELECT u.name,
       (SELECT count(*) FROM orders WHERE user_id = u.id) as order_count
FROM users u;

-- Хорошо: один запрос с GROUP BY
SELECT u.name, count(o.id) as order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name;
```

---

## Индексы для ускорения

### Типы индексов в PostgreSQL

| Тип | Применение | Операторы |
|-----|------------|-----------|
| **B-tree** | По умолчанию, для сравнений | `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `IN` |
| **Hash** | Только равенство | `=` |
| **GIN** | Полнотекстовый поиск, массивы, JSONB | `@>`, `<@`, `&&`, `@@` |
| **GiST** | Геометрия, диапазоны, полнотекст | `<<`, `>>`, `@>`, `<@`, `&&` |
| **BRIN** | Очень большие таблицы с коррелированными данными | `=`, `<`, `>` |

### Примеры создания индексов

```sql
-- B-tree (по умолчанию)
CREATE INDEX idx_users_email ON users(email);

-- Составной индекс
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);

-- Частичный индекс
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';

-- GIN для JSONB
CREATE INDEX idx_products_attrs ON products USING gin(attributes);

-- GIN для массивов
CREATE INDEX idx_posts_tags ON posts USING gin(tags);

-- GiST для геометрии (PostGIS)
CREATE INDEX idx_locations_geom ON locations USING gist(geom);

-- BRIN для временных рядов
CREATE INDEX idx_logs_created ON logs USING brin(created_at);
```

### Когда какой индекс использовать

**B-tree:**
- Первичные ключи
- Внешние ключи
- Условия с `=`, `<`, `>`, `BETWEEN`
- `ORDER BY`

**Hash:**
- Только `=` сравнения
- Редко используется (B-tree обычно не хуже)

**GIN:**
- Полнотекстовый поиск (`tsvector`, `tsquery`)
- JSONB поля с операторами `@>`, `?`, `?&`
- Массивы с операторами `@>`, `&&`
- trigram поиск (`LIKE '%pattern%'`)

**GiST:**
- Геоданные (PostGIS)
- Диапазоны (range types)
- Полнотекстовый поиск (альтернатива GIN)

**BRIN:**
- Очень большие таблицы (миллиарды строк)
- Данные естественно отсортированы (логи по времени)
- Минимальный размер индекса

---

## Connection Pooling

### Зачем нужен пулинг соединений

Создание соединения с PostgreSQL — дорогая операция:
- Fork нового процесса
- Аутентификация
- Выделение памяти (work_mem для каждого соединения)

**Проблемы без пулинга:**
- Лимит `max_connections` быстро достигается
- Высокое потребление памяти
- Задержки на создание соединений

### PgBouncer

Самый популярный connection pooler для PostgreSQL.

**Режимы работы:**

| Режим | Описание | Когда использовать |
|-------|----------|-------------------|
| **Session** | Соединение закрепляется за клиентом на всю сессию | Нужны prepared statements, курсоры |
| **Transaction** | Соединение выделяется на время транзакции | Большинство приложений |
| **Statement** | Соединение выделяется на один запрос | Только простые запросы |

**Пример конфигурации pgbouncer.ini:**

```ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

# Режим пулинга
pool_mode = transaction

# Размеры пулов
default_pool_size = 20
max_client_conn = 1000
min_pool_size = 5

# Таймауты
server_idle_timeout = 600
client_idle_timeout = 0
```

### Мониторинг PgBouncer

```sql
-- Подключение к админ-консоли PgBouncer
psql -p 6432 -U pgbouncer pgbouncer

-- Статистика пулов
SHOW POOLS;

-- Активные клиенты
SHOW CLIENTS;

-- Статистика по базам
SHOW STATS;
```

---

## Best Practices

### 1. Регулярный мониторинг

```sql
-- Еженедельный отчёт по медленным запросам
SELECT
    substring(query, 1, 80) as query,
    calls,
    round(total_exec_time::numeric / 1000, 2) as total_sec,
    round(mean_exec_time::numeric, 2) as avg_ms
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Сброс статистики после анализа
SELECT pg_stat_statements_reset();
```

### 2. Регулярное обслуживание

```sql
-- Обновление статистики
ANALYZE;

-- Очистка мёртвых строк и обновление статистики
VACUUM ANALYZE;

-- Полная перестройка таблицы (блокирующая!)
VACUUM FULL table_name;

-- Перестройка индексов
REINDEX INDEX idx_name;
REINDEX TABLE table_name;
```

### 3. Правила создания индексов

1. **Создавайте индексы для WHERE и JOIN условий**
2. **Порядок столбцов в составном индексе важен** — от более селективного к менее
3. **Не создавайте лишних индексов** — они замедляют INSERT/UPDATE
4. **Используйте частичные индексы** для фильтрации данных
5. **Используйте покрывающие индексы** для избежания Heap Fetch

### 4. Конфигурация PostgreSQL

```conf
# Память
shared_buffers = 25% от RAM (не более 8GB)
effective_cache_size = 75% от RAM
work_mem = 64MB (осторожно, умножается на connections)
maintenance_work_mem = 512MB

# Планировщик
random_page_cost = 1.1 (для SSD)
effective_io_concurrency = 200 (для SSD)

# WAL
wal_buffers = 64MB
checkpoint_completion_target = 0.9

# Параллелизм
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
```

### 5. Чеклист оптимизации

- [ ] Включен pg_stat_statements
- [ ] Настроено логирование медленных запросов
- [ ] Регулярный VACUUM ANALYZE (autovacuum настроен)
- [ ] Проверены неиспользуемые индексы
- [ ] Проверены таблицы с частым Seq Scan
- [ ] Настроен connection pooling
- [ ] Мониторинг cache hit ratio > 99%
- [ ] Мониторинг активных соединений

---

## Краткое резюме

1. **Профилирование** — ключ к пониманию производительности БД

2. **Основные инструменты:**
   - `EXPLAIN ANALYZE` — для анализа отдельных запросов
   - `pg_stat_statements` — для общей картины
   - Query logs + pgBadger — для исторического анализа

3. **Чтение EXPLAIN:**
   - Читаем снизу вверх
   - Сравниваем estimated vs actual rows
   - Ищем Seq Scan на больших таблицах
   - Проверяем тип JOIN

4. **Типичные проблемы:**
   - Отсутствие индексов
   - Устаревшая статистика
   - N+1 запросы
   - Неоптимальные типы данных

5. **Индексы:**
   - B-tree для большинства случаев
   - GIN для JSONB, массивов, полнотекстового поиска
   - BRIN для очень больших таблиц с коррелированными данными

6. **Connection pooling** (PgBouncer) — обязателен для production

7. **Регулярное обслуживание:**
   - `VACUUM ANALYZE`
   - Мониторинг pg_stat_statements
   - Проверка неиспользуемых индексов
