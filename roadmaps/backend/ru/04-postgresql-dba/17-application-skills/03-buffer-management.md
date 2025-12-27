# Управление буферами в PostgreSQL

## Обзор Buffer Management

**Buffer Manager** — компонент PostgreSQL, управляющий кэшированием страниц данных в памяти. Он минимизирует дисковый ввод-вывод, сохраняя часто используемые данные в оперативной памяти.

## Shared Buffers

### Концепция

**Shared Buffer Pool** — область разделяемой памяти, содержащая копии страниц данных с диска.

```sql
-- Текущий размер shared buffers
SHOW shared_buffers;

-- Рекомендуемое значение: 25% RAM (но не более 8GB для большинства случаев)
ALTER SYSTEM SET shared_buffers = '4GB';
```

### Структура буфера

Каждый буфер содержит:
- **Buffer Tag** — идентификатор страницы (файл, блок)
- **Buffer Descriptor** — метаданные (грязный ли, закреплен ли, счетчик использования)
- **Buffer Content** — данные страницы (8KB)

```
┌─────────────────────────────────────────────────┐
│              Shared Buffer Pool                  │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐   │
│  │ Buffer │ │ Buffer │ │ Buffer │ │ Buffer │   │
│  │   0    │ │   1    │ │   2    │ │  ...   │   │
│  │ 8KB    │ │ 8KB    │ │ 8KB    │ │ 8KB    │   │
│  │        │ │ dirty  │ │ pinned │ │        │   │
│  └────────┘ └────────┘ └────────┘ └────────┘   │
│                                                  │
│  Buffer Hash Table: Tag → Buffer ID              │
└─────────────────────────────────────────────────┘
```

## Алгоритм Clock Sweep

PostgreSQL использует **Clock Sweep** (вариант LRU) для выбора буфера на вытеснение:

1. Каждый буфер имеет **usage_count** (0-5)
2. При обращении к буферу usage_count увеличивается
3. При поиске свободного буфера алгоритм "обходит" буферы по кругу
4. При обходе уменьшает usage_count на 1
5. Выбирает буфер с usage_count = 0

```
   ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐
   │ u=3 │ ←─ │ u=1 │ ←─ │ u=0 │ ←─ │ u=2 │
   └─────┘    └─────┘    └─────┘    └─────┘
                            ↑
                      Кандидат на вытеснение
```

## Buffer States

### Dirty Buffer (грязный буфер)

Буфер, содержимое которого изменилось, но еще не записано на диск.

```sql
-- Количество грязных буферов
SELECT count(*) as dirty_buffers
FROM pg_buffercache
WHERE isdirty = true;
```

### Pinned Buffer (закрепленный буфер)

Буфер, который сейчас используется каким-то процессом и не может быть вытеснен.

```sql
-- Закрепленные буферы
SELECT count(*) as pinned_buffers
FROM pg_buffercache
WHERE pinning_backends > 0;
```

## Анализ Buffer Cache

Расширение **pg_buffercache** позволяет исследовать содержимое буферного кэша:

```sql
-- Установка расширения
CREATE EXTENSION IF NOT EXISTS pg_buffercache;

-- Топ таблиц по использованию буферов
SELECT
    c.relname,
    count(*) as buffers,
    pg_size_pretty(count(*) * 8192) as buffer_size,
    round(100.0 * count(*) / (SELECT count(*) FROM pg_buffercache), 2) as pct
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
AND b.reldatabase IN (0, (SELECT oid FROM pg_database WHERE datname = current_database()))
GROUP BY c.relname
ORDER BY 2 DESC
LIMIT 10;

-- Использование буферов по типам объектов
SELECT
    CASE
        WHEN c.relkind = 'r' THEN 'table'
        WHEN c.relkind = 'i' THEN 'index'
        WHEN c.relkind = 't' THEN 'toast'
        ELSE 'other'
    END as object_type,
    count(*) as buffers,
    pg_size_pretty(count(*) * 8192) as size
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
GROUP BY 1
ORDER BY 2 DESC;
```

## Buffer Hit Ratio

**Buffer Hit Ratio** — ключевая метрика эффективности кэша:

```sql
-- Общий hit ratio
SELECT
    sum(blks_hit) as hits,
    sum(blks_read) as reads,
    round(100.0 * sum(blks_hit) / nullif(sum(blks_hit) + sum(blks_read), 0), 2) as hit_ratio
FROM pg_stat_database;

-- Hit ratio по таблицам
SELECT
    relname,
    heap_blks_hit,
    heap_blks_read,
    round(100.0 * heap_blks_hit / nullif(heap_blks_hit + heap_blks_read, 0), 2) as hit_ratio
FROM pg_statio_user_tables
WHERE heap_blks_hit + heap_blks_read > 0
ORDER BY heap_blks_read DESC
LIMIT 10;

-- Hit ratio по индексам
SELECT
    indexrelname,
    idx_blks_hit,
    idx_blks_read,
    round(100.0 * idx_blks_hit / nullif(idx_blks_hit + idx_blks_read, 0), 2) as hit_ratio
FROM pg_statio_user_indexes
WHERE idx_blks_hit + idx_blks_read > 0
ORDER BY idx_blks_read DESC
LIMIT 10;
```

**Целевые значения:**
- > 99% — отлично
- 95-99% — хорошо
- < 95% — требует внимания (возможно, нужно увеличить shared_buffers)

## Background Writer

**Background Writer** периодически записывает грязные буферы на диск для:
- Снижения нагрузки на checkpoint
- Поддержания пула чистых буферов

```sql
-- Настройки bgwriter
SHOW bgwriter_delay;        -- интервал (200ms)
SHOW bgwriter_lru_maxpages; -- макс. страниц за раунд (100)
SHOW bgwriter_lru_multiplier; -- множитель для оценки потребности (2.0)

-- Статистика bgwriter
SELECT
    checkpoints_timed,
    checkpoints_req,
    buffers_checkpoint,
    buffers_clean,        -- записано bgwriter
    buffers_backend,      -- записано backend процессами (плохо если много)
    buffers_alloc
FROM pg_stat_bgwriter;
```

**Важно:** высокое значение `buffers_backend` указывает на то, что backend процессам приходится самим записывать грязные буферы — это снижает производительность.

## Ring Buffer

Для больших последовательных операций (seq scan, bulk insert, VACUUM) PostgreSQL использует **Ring Buffer** — небольшой изолированный буфер, чтобы не вытеснять полезные данные из основного кэша.

```sql
-- Размеры ring buffers
-- Bulk read (seq scan больших таблиц): 256KB
-- Bulk write (COPY, CREATE TABLE AS): 16MB
-- Vacuum: 256KB
```

## Huge Pages

Использование **Huge Pages** (2MB вместо 4KB) уменьшает накладные расходы на управление памятью:

```sql
-- Включение huge pages
ALTER SYSTEM SET huge_pages = 'try';  -- try, on, off

-- Проверка использования
SHOW huge_pages;
```

```bash
# Настройка на уровне ОС (Linux)
# Подсчет необходимых huge pages
head -1 /proc/meminfo | grep HugePages

# Установка (пример для 8GB shared_buffers)
# 8GB / 2MB = 4096 huge pages
sysctl -w vm.nr_hugepages=4200  # с запасом
```

## Оптимизация Buffer Management

### Настройка shared_buffers

```sql
-- Для сервера с 64GB RAM
ALTER SYSTEM SET shared_buffers = '16GB';      -- 25%
ALTER SYSTEM SET effective_cache_size = '48GB'; -- 75% (для планировщика)
```

### Мониторинг эффективности

```sql
-- Создание функции для анализа
CREATE OR REPLACE FUNCTION buffer_usage_report()
RETURNS TABLE (
    table_name text,
    buffer_count bigint,
    buffer_size text,
    dirty_count bigint,
    pct_of_relation numeric
) AS $$
SELECT
    c.relname::text,
    count(*),
    pg_size_pretty(count(*) * 8192),
    count(*) FILTER (WHERE b.isdirty),
    round(100.0 * count(*) * 8192 / pg_relation_size(c.oid), 2)
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
AND c.relkind = 'r'
GROUP BY c.oid, c.relname
ORDER BY 2 DESC
LIMIT 20;
$$ LANGUAGE sql;
```

## Best Practices

1. **shared_buffers**: 25% RAM для большинства систем
2. **Мониторьте hit ratio**: цель > 99%
3. **Следите за buffers_backend**: должно быть минимальным
4. **Используйте huge pages**: для больших shared_buffers
5. **effective_cache_size**: установите 50-75% RAM
6. **Регулярно анализируйте** использование буферов через pg_buffercache
