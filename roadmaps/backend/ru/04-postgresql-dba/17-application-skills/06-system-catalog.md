# Системный каталог PostgreSQL

[prev: 05-physical-storage](./05-physical-storage.md) | [next: 07-workload-tuning](./07-workload-tuning.md)

---

## Обзор

**Системный каталог** — набор таблиц и представлений, содержащих метаданные о всех объектах базы данных. PostgreSQL хранит всю информацию о структуре БД в этих таблицах.

## pg_catalog — основная схема

Схема **pg_catalog** содержит системные таблицы и функции:

```sql
-- Список объектов в pg_catalog
SELECT relname, relkind
FROM pg_class
WHERE relnamespace = 'pg_catalog'::regnamespace
ORDER BY relname;

-- relkind: r=таблица, v=view, i=индекс, S=sequence
```

## Основные системные таблицы

### pg_class — все объекты

```sql
-- Все таблицы, индексы, представления и т.д.
SELECT
    relname,
    relkind,           -- r=table, i=index, v=view, S=sequence, t=toast
    reltuples,         -- оценка количества строк
    relpages,          -- количество страниц
    relhasindex,
    relowner::regrole as owner
FROM pg_class
WHERE relnamespace = 'public'::regnamespace
AND relkind = 'r';  -- только таблицы
```

### pg_attribute — столбцы

```sql
-- Столбцы таблицы
SELECT
    attname,
    atttypid::regtype as data_type,
    attnum,            -- порядковый номер
    attnotnull,        -- NOT NULL?
    atthasdef,         -- есть DEFAULT?
    attstorage         -- стратегия TOAST
FROM pg_attribute
WHERE attrelid = 'users'::regclass
AND attnum > 0         -- системные столбцы имеют отрицательные номера
AND NOT attisdropped;
```

### pg_index — индексы

```sql
-- Информация об индексах
SELECT
    i.indexrelid::regclass as index_name,
    i.indrelid::regclass as table_name,
    i.indisunique,
    i.indisprimary,
    pg_get_indexdef(i.indexrelid) as definition
FROM pg_index i
WHERE i.indrelid = 'users'::regclass;
```

### pg_constraint — ограничения

```sql
-- Ограничения таблицы
SELECT
    conname,
    contype,           -- p=PK, f=FK, u=unique, c=check
    pg_get_constraintdef(oid) as definition
FROM pg_constraint
WHERE conrelid = 'users'::regclass;
```

### pg_namespace — схемы

```sql
-- Все схемы
SELECT nspname, nspowner::regrole as owner
FROM pg_namespace
WHERE nspname NOT LIKE 'pg_%';
```

### pg_database — базы данных

```sql
-- Все базы данных
SELECT
    datname,
    datdba::regrole as owner,
    encoding,
    pg_encoding_to_char(encoding) as encoding_name,
    datcollate,
    pg_size_pretty(pg_database_size(oid)) as size
FROM pg_database;
```

### pg_roles / pg_authid — роли

```sql
-- Роли и их права
SELECT
    rolname,
    rolsuper,
    rolcreaterole,
    rolcreatedb,
    rolcanlogin,
    rolreplication
FROM pg_roles;

-- pg_authid содержит пароли (доступен только суперпользователям)
```

### pg_proc — функции и процедуры

```sql
-- Пользовательские функции
SELECT
    proname,
    pronargs,          -- количество аргументов
    prorettype::regtype as return_type,
    prosrc              -- исходный код (для PL/pgSQL)
FROM pg_proc
WHERE pronamespace = 'public'::regnamespace;

-- Определение функции
SELECT pg_get_functiondef('my_function'::regproc);
```

### pg_trigger — триггеры

```sql
-- Триггеры
SELECT
    tgname,
    tgrelid::regclass as table_name,
    tgfoid::regproc as function_name,
    tgtype,
    tgenabled
FROM pg_trigger
WHERE NOT tgisinternal;
```

## Information Schema

**information_schema** — SQL-стандартный способ получения метаданных:

```sql
-- Таблицы
SELECT table_schema, table_name, table_type
FROM information_schema.tables
WHERE table_schema = 'public';

-- Столбцы
SELECT
    column_name,
    data_type,
    character_maximum_length,
    is_nullable,
    column_default
FROM information_schema.columns
WHERE table_name = 'users';

-- Ограничения
SELECT
    constraint_name,
    constraint_type
FROM information_schema.table_constraints
WHERE table_name = 'users';
```

### Сравнение pg_catalog и information_schema

| Аспект | pg_catalog | information_schema |
|--------|------------|-------------------|
| Стандарт | PostgreSQL-специфичный | SQL стандарт |
| Производительность | Быстрее | Медленнее (views) |
| Полнота | Полная информация | Базовая информация |
| Переносимость | Только PostgreSQL | Между СУБД |

## Системные представления

### pg_stat_* — статистика

```sql
-- Статистика таблиц
SELECT * FROM pg_stat_user_tables WHERE relname = 'users';

-- Статистика индексов
SELECT * FROM pg_stat_user_indexes WHERE relname = 'users';

-- Активные сессии
SELECT * FROM pg_stat_activity;

-- Статистика репликации
SELECT * FROM pg_stat_replication;
```

### pg_statio_* — I/O статистика

```sql
-- I/O по таблицам
SELECT
    relname,
    heap_blks_read,
    heap_blks_hit,
    round(100.0 * heap_blks_hit / nullif(heap_blks_hit + heap_blks_read, 0), 2) as hit_ratio
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC;
```

## Полезные запросы к каталогу

### Размеры объектов

```sql
-- Топ таблиц по размеру
SELECT
    schemaname || '.' || relname as table,
    pg_size_pretty(pg_total_relation_size(relid)) as total_size,
    pg_size_pretty(pg_relation_size(relid)) as table_size,
    pg_size_pretty(pg_indexes_size(relid)) as indexes_size
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 10;
```

### Неиспользуемые индексы

```sql
SELECT
    schemaname || '.' || relname as table,
    indexrelname as index,
    pg_size_pretty(pg_relation_size(indexrelid)) as size,
    idx_scan as scans
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND indexrelid NOT IN (
    SELECT indexrelid FROM pg_constraint WHERE contype IN ('p', 'u')
)
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Таблицы без первичного ключа

```sql
SELECT
    n.nspname || '.' || c.relname as table_name
FROM pg_class c
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE c.relkind = 'r'
AND n.nspname NOT IN ('pg_catalog', 'information_schema')
AND NOT EXISTS (
    SELECT 1 FROM pg_constraint con
    WHERE con.conrelid = c.oid AND con.contype = 'p'
);
```

### Внешние ключи

```sql
SELECT
    conname as fk_name,
    conrelid::regclass as table_name,
    a.attname as column_name,
    confrelid::regclass as foreign_table,
    af.attname as foreign_column
FROM pg_constraint c
JOIN pg_attribute a ON a.attrelid = c.conrelid AND a.attnum = ANY(c.conkey)
JOIN pg_attribute af ON af.attrelid = c.confrelid AND af.attnum = ANY(c.confkey)
WHERE c.contype = 'f';
```

### Зависимости между объектами

```sql
-- От чего зависит представление
SELECT
    dependent.relname as view_name,
    source.relname as depends_on
FROM pg_depend d
JOIN pg_rewrite r ON d.objid = r.oid
JOIN pg_class dependent ON r.ev_class = dependent.oid
JOIN pg_class source ON d.refobjid = source.oid
WHERE source.relkind = 'r'
AND dependent.relkind = 'v'
AND dependent.relname = 'my_view';
```

## Модификация каталога

**ВАЖНО**: Прямое изменение системных таблиц опасно и не рекомендуется!

```sql
-- Правильный способ — использовать DDL
ALTER TABLE users RENAME TO customers;
ALTER TABLE users ALTER COLUMN name SET NOT NULL;

-- Неправильный способ (никогда так не делайте)
-- UPDATE pg_class SET relname = 'customers' WHERE relname = 'users';
```

## Расширения для работы с каталогом

```sql
-- pg_catalog_get_defs — получение DDL
SELECT pg_get_tabledef('users'::regclass);  -- если есть расширение

-- Стандартные функции
SELECT pg_get_viewdef('my_view'::regclass);
SELECT pg_get_functiondef('my_func'::regproc);
SELECT pg_get_indexdef('my_index'::regclass);
SELECT pg_get_constraintdef(oid) FROM pg_constraint WHERE conname = 'my_fk';
```

## Best Practices

1. **information_schema** для простых запросов и переносимости
2. **pg_catalog** для полной информации и производительности
3. **Никогда не модифицируйте** системные таблицы напрямую
4. **Кэшируйте** результаты запросов к каталогу — они не меняются часто
5. **Используйте regclass/regproc** для безопасного приведения имен к OID

---

[prev: 05-physical-storage](./05-physical-storage.md) | [next: 07-workload-tuning](./07-workload-tuning.md)
