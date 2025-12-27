# Физическое хранение данных в PostgreSQL

## Структура каталогов

При инициализации PostgreSQL создает директорию данных (PGDATA):

```bash
$PGDATA/
├── base/                   # Базы данных
│   ├── 1/                  # template1
│   ├── 13067/             # postgres
│   └── 16384/             # пользовательская БД (OID)
├── global/                 # Общие объекты кластера
├── pg_wal/                 # WAL файлы
├── pg_xact/                # Статусы транзакций (CLOG)
├── pg_tblspc/              # Символические ссылки на tablespaces
├── pg_stat/                # Статистика
├── pg_stat_tmp/            # Временная статистика
├── pg_subtrans/            # Подтранзакции
├── pg_commit_ts/           # Временные метки коммитов
├── pg_multixact/           # Мультитранзакции
├── pg_notify/              # LISTEN/NOTIFY
├── pg_replslot/            # Слоты репликации
├── pg_snapshots/           # Снимки
├── pg_twophase/            # Подготовленные транзакции
├── postgresql.conf         # Конфигурация
├── pg_hba.conf             # Аутентификация
├── pg_ident.conf           # Маппинг пользователей
├── PG_VERSION              # Версия
├── postmaster.pid          # PID главного процесса
└── postmaster.opts         # Опции запуска
```

## Файлы данных

### Связь объектов и файлов

```sql
-- OID базы данных
SELECT oid, datname FROM pg_database;

-- OID таблицы
SELECT oid, relname, relfilenode FROM pg_class WHERE relname = 'users';

-- Путь к файлу таблицы
SELECT pg_relation_filepath('users');
-- Результат: base/16384/24576
```

### Структура файлов таблицы

Для каждой таблицы PostgreSQL создает несколько файлов:

```bash
base/16384/
├── 24576           # Основной файл данных (heap)
├── 24576_fsm       # Free Space Map
├── 24576_vm        # Visibility Map
└── 24576.1         # Следующий сегмент (при размере > 1GB)
```

```sql
-- Размеры компонентов
SELECT
    pg_relation_filepath('users') as main_file,
    pg_size_pretty(pg_relation_size('users', 'main')) as heap_size,
    pg_size_pretty(pg_relation_size('users', 'fsm')) as fsm_size,
    pg_size_pretty(pg_relation_size('users', 'vm')) as vm_size,
    pg_size_pretty(pg_total_relation_size('users')) as total_size;
```

## Страницы (Pages)

Все данные хранятся в **страницах** размером 8KB (по умолчанию).

### Структура страницы

```
┌─────────────────────────────────────────────────┐ 0
│              Page Header (24 bytes)             │
├─────────────────────────────────────────────────┤ 24
│           Item Pointers (Line Pointers)         │
│  [lp1] [lp2] [lp3] ... [lpN]                   │
├─────────────────────────────────────────────────┤
│                                                 │
│              Free Space                         │
│                                                 │
├─────────────────────────────────────────────────┤
│                                                 │
│     Tuples (rows) - растут снизу вверх         │
│  [Tuple N] [Tuple 3] [Tuple 2] [Tuple 1]       │
│                                                 │
├─────────────────────────────────────────────────┤
│            Special Space (для индексов)         │
└─────────────────────────────────────────────────┘ 8192
```

### Анализ страниц

```sql
-- Расширение для анализа страниц
CREATE EXTENSION pageinspect;

-- Заголовок страницы
SELECT * FROM page_header(get_raw_page('users', 0));

-- Item pointers
SELECT * FROM heap_page_items(get_raw_page('users', 0));

-- Детали кортежа
SELECT
    lp,                    -- номер указателя
    lp_off,               -- смещение
    lp_len,               -- длина
    t_xmin,               -- создавшая транзакция
    t_xmax,               -- удалившая транзакция
    t_ctid,               -- физическое расположение
    t_infomask::bit(16)   -- флаги
FROM heap_page_items(get_raw_page('users', 0))
WHERE lp_len > 0;
```

## Tuple (кортеж/строка)

### Структура Tuple

```
┌────────────────────────────────────────┐
│         Tuple Header (23 bytes)        │
│  - t_xmin (4): создавшая транзакция   │
│  - t_xmax (4): удалившая транзакция   │
│  - t_cid (4): command ID               │
│  - t_ctid (6): (block, offset)         │
│  - t_infomask (2): флаги              │
│  - t_infomask2 (2): флаги             │
│  - t_hoff (1): смещение данных         │
├────────────────────────────────────────┤
│            Null Bitmap                  │
│    (если есть NULL значения)           │
├────────────────────────────────────────┤
│            User Data                    │
│    (значения полей строки)             │
└────────────────────────────────────────┘
```

```sql
-- Оценка размера строки
SELECT
    pg_column_size(t.*) as tuple_size,
    pg_column_size(t.id) as id_size,
    pg_column_size(t.name) as name_size
FROM users t
LIMIT 5;
```

## TOAST (The Oversized-Attribute Storage Technique)

Для хранения больших значений (> 2KB) PostgreSQL использует **TOAST**:

### Стратегии TOAST

| Стратегия | Описание |
|-----------|----------|
| PLAIN | Не использовать TOAST (для мелких типов) |
| EXTENDED | Сжатие + вынос в TOAST таблицу |
| EXTERNAL | Без сжатия, вынос в TOAST таблицу |
| MAIN | Сжатие, вынос только если не помещается |

```sql
-- Просмотр стратегий столбцов
SELECT
    attname,
    atttypid::regtype,
    CASE attstorage
        WHEN 'p' THEN 'PLAIN'
        WHEN 'e' THEN 'EXTERNAL'
        WHEN 'm' THEN 'MAIN'
        WHEN 'x' THEN 'EXTENDED'
    END as storage
FROM pg_attribute
WHERE attrelid = 'users'::regclass
AND attnum > 0;

-- Изменение стратегии
ALTER TABLE users ALTER COLUMN large_text SET STORAGE EXTERNAL;
```

### TOAST таблица

```sql
-- Найти TOAST таблицу
SELECT
    c.relname as table_name,
    t.relname as toast_table,
    pg_size_pretty(pg_relation_size(t.oid)) as toast_size
FROM pg_class c
JOIN pg_class t ON c.reltoastrelid = t.oid
WHERE c.relname = 'users';

-- Размер TOAST данных
SELECT
    relname,
    pg_size_pretty(pg_relation_size(reltoastrelid)) as toast_size
FROM pg_class
WHERE reltoastrelid != 0
ORDER BY pg_relation_size(reltoastrelid) DESC
LIMIT 10;
```

## Free Space Map (FSM)

**FSM** отслеживает свободное место на каждой странице:

```sql
-- Расширение для анализа FSM
CREATE EXTENSION pg_freespacemap;

-- Свободное место по страницам
SELECT
    blkno,
    avail as free_bytes,
    round(100.0 * avail / 8192, 2) as free_pct
FROM pg_freespace('users')
ORDER BY blkno
LIMIT 20;

-- Общая статистика
SELECT
    count(*) as total_pages,
    sum(avail) as total_free_bytes,
    pg_size_pretty(sum(avail)) as total_free
FROM pg_freespace('users');
```

## Visibility Map (VM)

**VM** отслеживает страницы, где все кортежи видимы всем транзакциям:

```sql
-- Расширение для анализа VM
CREATE EXTENSION pg_visibility;

-- Состояние visibility map
SELECT * FROM pg_visibility_map('users') LIMIT 10;

-- Статистика
SELECT
    all_visible,
    all_frozen,
    pd_all_visible
FROM pg_visibility_map_summary('users');
```

## Tablespaces

**Tablespaces** позволяют размещать данные на разных дисках:

```sql
-- Создание tablespace
CREATE TABLESPACE fast_storage LOCATION '/mnt/ssd/postgres';
CREATE TABLESPACE archive_storage LOCATION '/mnt/hdd/postgres';

-- Создание таблицы в tablespace
CREATE TABLE hot_data (id serial, data text) TABLESPACE fast_storage;

-- Перемещение таблицы
ALTER TABLE cold_data SET TABLESPACE archive_storage;

-- Перемещение индекса
ALTER INDEX idx_hot_data SET TABLESPACE fast_storage;

-- Tablespace по умолчанию для БД
ALTER DATABASE mydb SET TABLESPACE fast_storage;

-- Просмотр tablespaces
SELECT
    spcname,
    pg_tablespace_location(oid) as location,
    pg_size_pretty(pg_tablespace_size(oid)) as size
FROM pg_tablespace;
```

## Сегменты файлов

При достижении 1GB файл разбивается на сегменты:

```bash
# Для таблицы размером 2.5GB
24576       # 0-1GB
24576.1     # 1-2GB
24576.2     # 2-2.5GB
```

```sql
-- Изменение размера сегмента (только при сборке)
-- ./configure --with-segsize=2  # 2GB сегменты
```

## Инструменты анализа

```sql
-- Общий размер объектов
SELECT
    relname,
    pg_size_pretty(pg_relation_size(oid)) as table_size,
    pg_size_pretty(pg_indexes_size(oid)) as indexes_size,
    pg_size_pretty(pg_total_relation_size(oid)) as total_size
FROM pg_class
WHERE relkind = 'r' AND relnamespace = 'public'::regnamespace
ORDER BY pg_total_relation_size(oid) DESC
LIMIT 10;

-- Физические пути
SELECT
    tablename,
    pg_relation_filepath(tablename::regclass) as filepath
FROM pg_tables
WHERE schemaname = 'public';
```

## Best Practices

1. **Мониторинг bloat**: регулярно проверяйте FSM и VM
2. **TOAST**: используйте EXTERNAL для больших бинарных данных, которые не сжимаются
3. **Tablespaces**: размещайте индексы и "горячие" таблицы на SSD
4. **Размер страницы**: не меняйте без веской причины (требует пересборки)
5. **pageinspect**: используйте для диагностики проблем на уровне страниц
