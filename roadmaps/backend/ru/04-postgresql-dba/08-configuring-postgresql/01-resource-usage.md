# Resource Usage (Использование ресурсов)

[prev: 04-aggregate-window-functions](../07-advanced-sql/04-aggregate-window-functions.md) | [next: 02-vacuums](./02-vacuums.md)

---

Конфигурация использования ресурсов в PostgreSQL определяет, как сервер распределяет память, CPU и дисковые операции. Правильная настройка этих параметров критически важна для производительности базы данных.

## Управление памятью

### shared_buffers

Основной параметр, определяющий объём памяти для кэширования данных.

```sql
-- Рекомендуемые значения
shared_buffers = 256MB  -- минимум для продакшена
shared_buffers = 4GB    -- для сервера с 16GB RAM
shared_buffers = 8GB    -- для сервера с 32GB+ RAM
```

**Рекомендация**: 25% от общей RAM, но не более 8GB на большинстве систем.

```sql
-- Проверка текущего значения
SHOW shared_buffers;

-- Просмотр использования буферов
SELECT
    c.relname,
    pg_size_pretty(count(*) * 8192) as buffered,
    round(100.0 * count(*) / (SELECT setting FROM pg_settings WHERE name='shared_buffers')::integer, 1) AS buffers_percent
FROM pg_buffercache b
INNER JOIN pg_class c ON b.relfilenode = c.relfilenode
GROUP BY c.relname
ORDER BY count(*) DESC
LIMIT 10;
```

### work_mem

Память для операций сортировки и хэширования в рамках одного запроса.

```sql
-- Значения по умолчанию и рекомендации
work_mem = 4MB      -- по умолчанию
work_mem = 64MB     -- для аналитических запросов
work_mem = 256MB    -- для сложных OLAP систем
```

**Важно**: Память выделяется для каждой операции отдельно. Сложный запрос может использовать `work_mem * количество_операций`.

```sql
-- Временное увеличение для сессии
SET work_mem = '256MB';

-- Проверка влияния на план запроса
EXPLAIN ANALYZE SELECT * FROM large_table ORDER BY column;
```

### maintenance_work_mem

Память для операций обслуживания: VACUUM, CREATE INDEX, ALTER TABLE.

```sql
-- Рекомендуемые значения
maintenance_work_mem = 64MB     -- по умолчанию
maintenance_work_mem = 512MB    -- для среднего сервера
maintenance_work_mem = 2GB      -- для больших баз данных
```

```sql
-- Увеличение для создания индекса
SET maintenance_work_mem = '1GB';
CREATE INDEX CONCURRENTLY idx_large ON large_table(column);
RESET maintenance_work_mem;
```

### effective_cache_size

Оценка памяти, доступной для кэширования (shared_buffers + OS cache).

```sql
-- Обычно 50-75% от общей RAM
effective_cache_size = 12GB  -- для сервера с 16GB RAM
```

Этот параметр влияет только на планировщик запросов, помогая ему принимать решения об использовании индексов.

## Управление соединениями

### max_connections

Максимальное количество одновременных подключений.

```sql
max_connections = 100       -- по умолчанию
max_connections = 200       -- типичное значение
max_connections = 1000      -- требует connection pooler
```

**Формула расчёта памяти**:
```
Память на соединение ≈ work_mem + temp_buffers + другие буферы
Максимальная память = max_connections * память_на_соединение
```

```sql
-- Мониторинг соединений
SELECT
    count(*) as total_connections,
    count(*) FILTER (WHERE state = 'active') as active,
    count(*) FILTER (WHERE state = 'idle') as idle,
    count(*) FILTER (WHERE state = 'idle in transaction') as idle_in_transaction
FROM pg_stat_activity;
```

### superuser_reserved_connections

Резервные соединения для суперпользователей.

```sql
superuser_reserved_connections = 3  -- по умолчанию
```

## Управление дисковыми операциями

### temp_buffers

Память для временных таблиц в каждой сессии.

```sql
temp_buffers = 8MB      -- по умолчанию
temp_buffers = 64MB     -- для сессий с большими временными таблицами
```

### random_page_cost и seq_page_cost

Стоимость доступа к случайной и последовательной странице диска.

```sql
-- Для HDD
random_page_cost = 4.0
seq_page_cost = 1.0

-- Для SSD
random_page_cost = 1.1
seq_page_cost = 1.0

-- Для NVMe
random_page_cost = 1.0
seq_page_cost = 1.0
```

### effective_io_concurrency

Количество параллельных I/O операций.

```sql
effective_io_concurrency = 1      -- для HDD
effective_io_concurrency = 200    -- для SSD
```

## Параллельная обработка

### max_worker_processes

Максимальное количество фоновых процессов.

```sql
max_worker_processes = 8    -- по умолчанию
```

### max_parallel_workers_per_gather

Параллельные воркеры для одного запроса.

```sql
max_parallel_workers_per_gather = 2     -- по умолчанию
max_parallel_workers_per_gather = 4     -- для мощных серверов
```

### max_parallel_workers

Общий лимит параллельных воркеров.

```sql
max_parallel_workers = 8    -- по умолчанию
```

```sql
-- Пример параллельного запроса
EXPLAIN ANALYZE
SELECT count(*) FROM large_table WHERE column > 1000;

-- Результат покажет Parallel Seq Scan, если настроено
```

## Пример конфигурации для разных сценариев

### Веб-приложение (16GB RAM, SSD)

```ini
# postgresql.conf
shared_buffers = 4GB
effective_cache_size = 12GB
work_mem = 32MB
maintenance_work_mem = 512MB
max_connections = 200

random_page_cost = 1.1
effective_io_concurrency = 200

max_worker_processes = 8
max_parallel_workers_per_gather = 2
max_parallel_workers = 8
```

### Аналитическая система (64GB RAM, NVMe)

```ini
# postgresql.conf
shared_buffers = 16GB
effective_cache_size = 48GB
work_mem = 256MB
maintenance_work_mem = 2GB
max_connections = 50

random_page_cost = 1.0
effective_io_concurrency = 200

max_worker_processes = 16
max_parallel_workers_per_gather = 8
max_parallel_workers = 16
```

## Диагностика и мониторинг

```sql
-- Проверка всех параметров памяти
SELECT name, setting, unit, context, short_desc
FROM pg_settings
WHERE category LIKE '%Resource%' OR category LIKE '%Memory%'
ORDER BY name;

-- Мониторинг использования памяти
SELECT
    pg_size_pretty(pg_database_size(current_database())) as db_size,
    (SELECT setting::bigint * 8192 FROM pg_settings WHERE name = 'shared_buffers') as shared_buffers_bytes;
```

## Типичные ошибки

1. **Слишком большой shared_buffers** — оставляйте память для OS cache
2. **Высокий work_mem при большом max_connections** — может исчерпать память
3. **Игнорирование effective_cache_size** — планировщик будет принимать неоптимальные решения
4. **random_page_cost = 4 на SSD** — база будет избегать индексов

## Best Practices

1. **Мониторьте использование памяти** через pg_stat_activity и системные утилиты
2. **Тестируйте изменения** на реальной нагрузке перед применением в продакшене
3. **Используйте connection pooler** (PgBouncer) при большом количестве соединений
4. **Настраивайте параметры под тип нагрузки** (OLTP vs OLAP)
5. **Регулярно пересматривайте настройки** при изменении данных или нагрузки

---

[prev: 04-aggregate-window-functions](../07-advanced-sql/04-aggregate-window-functions.md) | [next: 02-vacuums](./02-vacuums.md)
