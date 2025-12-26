# Процесс VACUUM в PostgreSQL

## Зачем нужен VACUUM

PostgreSQL использует **MVCC** (Multi-Version Concurrency Control), что означает:
- При UPDATE создается новая версия строки, старая помечается как "мертвая"
- При DELETE строка не удаляется физически, а только помечается
- Эти "мертвые" строки (dead tuples) занимают место и замедляют запросы

**VACUUM** — процесс очистки, который:
- Освобождает пространство, занятое мертвыми строками
- Обновляет карту видимости (visibility map)
- Обновляет карту свободного пространства (free space map)
- Предотвращает "wraparound" transaction ID

## Типы VACUUM

### Обычный VACUUM

```sql
-- Очистка одной таблицы
VACUUM tablename;

-- Очистка всей базы данных
VACUUM;

-- VACUUM с подробной информацией
VACUUM VERBOSE tablename;
```

**Особенности:**
- Не блокирует таблицу для чтения/записи
- Освобождает пространство для повторного использования внутри таблицы
- Не возвращает пространство операционной системе

### VACUUM FULL

```sql
VACUUM FULL tablename;
```

**Особенности:**
- Полностью перестраивает таблицу
- Возвращает пространство операционной системе
- **Блокирует таблицу полностью** (ACCESS EXCLUSIVE lock)
- Требует дополнительного места на диске

### VACUUM ANALYZE

```sql
-- Очистка + обновление статистики
VACUUM ANALYZE tablename;

-- Только обновление статистики
ANALYZE tablename;
```

## Как работает VACUUM

### Фаза 1: Сканирование таблицы

```
┌─────────────┐
│   Page 1    │  Сканируем страницу за страницей
│  ┌───────┐  │  Находим dead tuples
│  │live   │  │
│  │dead   │◄─┤  Помечаем для очистки
│  │live   │  │
│  └───────┘  │
└─────────────┘
```

### Фаза 2: Очистка индексов

Для каждого индекса удаляются указатели на мертвые строки.

### Фаза 3: Очистка heap

```sql
-- Пример: до и после VACUUM
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'my_table';
```

## Autovacuum

PostgreSQL автоматически запускает VACUUM через **autovacuum daemon**.

### Настройки autovacuum

```sql
-- Глобальные настройки
SHOW autovacuum;                        -- on/off
SHOW autovacuum_vacuum_threshold;       -- минимум dead tuples (50)
SHOW autovacuum_vacuum_scale_factor;    -- процент от размера таблицы (0.2 = 20%)
SHOW autovacuum_naptime;                -- интервал проверки (1min)
SHOW autovacuum_max_workers;            -- макс. параллельных workers (3)
```

### Формула запуска autovacuum

```
vacuum_threshold = autovacuum_vacuum_threshold +
                   autovacuum_vacuum_scale_factor * число_строк
```

Пример: таблица с 1,000,000 строк
```
50 + 0.2 * 1,000,000 = 200,050 dead tuples для запуска
```

### Настройка для отдельных таблиц

```sql
-- Агрессивный autovacuum для "горячей" таблицы
ALTER TABLE hot_table SET (
    autovacuum_vacuum_scale_factor = 0.01,    -- 1% вместо 20%
    autovacuum_vacuum_threshold = 100,
    autovacuum_vacuum_cost_limit = 2000       -- быстрее
);

-- Редкий autovacuum для статичной таблицы
ALTER TABLE static_table SET (
    autovacuum_vacuum_scale_factor = 0.5,     -- 50%
    autovacuum_enabled = false                -- или полностью отключить
);
```

## Transaction ID Wraparound

PostgreSQL использует 32-битный transaction ID (около 4 миллиардов значений). При исчерпании ID база "замерзает" для предотвращения потери данных.

### Freezing

```sql
-- Параметры заморозки
SHOW vacuum_freeze_min_age;      -- 50,000,000
SHOW vacuum_freeze_table_age;    -- 150,000,000
SHOW autovacuum_freeze_max_age;  -- 200,000,000
```

```sql
-- Проверка возраста транзакций
SELECT datname,
       age(datfrozenxid) as xid_age,
       current_setting('autovacuum_freeze_max_age')::int - age(datfrozenxid) as xids_remaining
FROM pg_database
ORDER BY 2 DESC;

-- Таблицы, требующие заморозки
SELECT relname,
       age(relfrozenxid) as xid_age
FROM pg_class
WHERE relkind = 'r'
ORDER BY 2 DESC
LIMIT 10;
```

## Мониторинг VACUUM

### Просмотр прогресса

```sql
-- Текущие операции VACUUM
SELECT pid,
       datname,
       relid::regclass as table_name,
       phase,
       heap_blks_scanned,
       heap_blks_total,
       round(100.0 * heap_blks_scanned / nullif(heap_blks_total, 0), 2) as progress_pct
FROM pg_stat_progress_vacuum;
```

### Статистика очистки

```sql
-- История очисток
SELECT schemaname,
       relname,
       n_dead_tup,
       last_vacuum,
       last_autovacuum,
       vacuum_count,
       autovacuum_count
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC;
```

### Bloat (раздувание)

```sql
-- Оценка bloat для таблиц
SELECT
    schemaname || '.' || relname as table_name,
    pg_size_pretty(pg_relation_size(schemaname || '.' || relname)) as table_size,
    n_dead_tup,
    n_live_tup,
    round(100.0 * n_dead_tup / nullif(n_live_tup + n_dead_tup, 0), 2) as dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

## Оптимизация VACUUM

### Cost-based VACUUM

```sql
-- Ограничение нагрузки VACUUM
SHOW vacuum_cost_delay;    -- задержка (0 = выключено)
SHOW vacuum_cost_limit;    -- лимит "стоимости" перед паузой
SHOW vacuum_cost_page_hit; -- стоимость чтения из buffer cache
SHOW vacuum_cost_page_miss;-- стоимость чтения с диска
```

```sql
-- Увеличение скорости autovacuum
ALTER SYSTEM SET autovacuum_vacuum_cost_limit = 2000;  -- по умолчанию 200
ALTER SYSTEM SET autovacuum_vacuum_cost_delay = '2ms'; -- по умолчанию 2ms
```

### Параллельный VACUUM (PostgreSQL 13+)

```sql
-- Параллельная очистка индексов
VACUUM (PARALLEL 4) large_table;
```

## Best Practices

1. **Не отключайте autovacuum** — это критически важный процесс
2. **Мониторьте dead tuples** — показатель необходимости очистки
3. **Настройте autovacuum для больших таблиц** — снизьте scale_factor
4. **Следите за transaction ID age** — предотвращайте wraparound
5. **Избегайте VACUUM FULL** в production — используйте pg_repack
6. **Запускайте ANALYZE** после массовых изменений данных

```sql
-- После bulk load
VACUUM ANALYZE newly_loaded_table;

-- Периодическое обслуживание
VACUUM VERBOSE ANALYZE;
```
