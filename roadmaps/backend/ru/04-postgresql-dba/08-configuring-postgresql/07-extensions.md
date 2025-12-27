# Extensions (Расширения PostgreSQL)

Расширения — это модули, добавляющие новую функциональность в PostgreSQL: типы данных, функции, операторы, индексы и другие возможности.

## Управление расширениями

### Основные команды

```sql
-- Список доступных расширений
SELECT * FROM pg_available_extensions ORDER BY name;

-- Список установленных расширений
SELECT * FROM pg_extension;

-- Детальная информация о расширении
\dx+ extension_name

-- Установка расширения
CREATE EXTENSION extension_name;

-- Установка в конкретную схему
CREATE EXTENSION extension_name SCHEMA schema_name;

-- Обновление расширения
ALTER EXTENSION extension_name UPDATE TO '2.0';

-- Удаление расширения
DROP EXTENSION extension_name;

-- Удаление с зависимостями
DROP EXTENSION extension_name CASCADE;
```

### Расширения, требующие preload

Некоторые расширения должны быть загружены при старте сервера:

```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements, auto_explain, pg_cron'
```

После изменения требуется перезапуск PostgreSQL.

## Популярные расширения

### pg_stat_statements

Сбор статистики выполнения запросов.

```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 10000
pg_stat_statements.track = all
```

```sql
CREATE EXTENSION pg_stat_statements;

-- Топ запросов по времени
SELECT
    query,
    calls,
    mean_exec_time,
    total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

### pg_trgm (триграммы)

Поиск по подстроке и нечёткий поиск.

```sql
CREATE EXTENSION pg_trgm;

-- Создание GiST индекса для LIKE/ILIKE
CREATE INDEX idx_name_trgm ON users USING GiST (name gist_trgm_ops);

-- Создание GIN индекса (быстрее для поиска)
CREATE INDEX idx_name_trgm ON users USING GIN (name gin_trgm_ops);

-- Поиск по подстроке
SELECT * FROM users WHERE name LIKE '%john%';

-- Нечёткий поиск с порогом сходства
SELECT *, similarity(name, 'jon') as sim
FROM users
WHERE similarity(name, 'jon') > 0.3
ORDER BY sim DESC;

-- Установка порога
SET pg_trgm.similarity_threshold = 0.3;
SELECT * FROM users WHERE name % 'jon';
```

### uuid-ossp

Генерация UUID.

```sql
CREATE EXTENSION "uuid-ossp";

-- Генерация UUID v4 (случайный)
SELECT uuid_generate_v4();

-- Генерация UUID v1 (на основе времени и MAC)
SELECT uuid_generate_v1();

-- Использование в таблице
CREATE TABLE items (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT
);
```

Альтернатива в PostgreSQL 13+:
```sql
-- Встроенная функция, не требует расширения
SELECT gen_random_uuid();
```

### pgcrypto

Криптографические функции.

```sql
CREATE EXTENSION pgcrypto;

-- Хеширование паролей
SELECT crypt('password', gen_salt('bf'));

-- Проверка пароля
SELECT crypt('password', stored_hash) = stored_hash;

-- Шифрование данных
SELECT pgp_sym_encrypt('secret data', 'encryption_key');

-- Расшифровка
SELECT pgp_sym_decrypt(encrypted_data, 'encryption_key');

-- Генерация случайных байт
SELECT gen_random_bytes(16);
```

### hstore

Хранение пар ключ-значение.

```sql
CREATE EXTENSION hstore;

-- Создание таблицы с hstore
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    attributes HSTORE
);

-- Вставка данных
INSERT INTO products (name, attributes) VALUES
    ('Laptop', 'brand => "Dell", ram => "16GB", ssd => "512GB"');

-- Запросы
SELECT * FROM products WHERE attributes -> 'brand' = 'Dell';
SELECT * FROM products WHERE attributes ? 'ram';
SELECT * FROM products WHERE attributes @> 'ram => "16GB"';

-- Индексирование
CREATE INDEX idx_attributes ON products USING GIN (attributes);
```

### ltree

Иерархические данные.

```sql
CREATE EXTENSION ltree;

-- Хранение иерархии категорий
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name TEXT,
    path LTREE
);

INSERT INTO categories (name, path) VALUES
    ('Electronics', 'Electronics'),
    ('Computers', 'Electronics.Computers'),
    ('Laptops', 'Electronics.Computers.Laptops'),
    ('Phones', 'Electronics.Phones');

-- Поиск всех потомков
SELECT * FROM categories WHERE path <@ 'Electronics.Computers';

-- Поиск всех предков
SELECT * FROM categories WHERE path @> 'Electronics.Computers.Laptops';

-- Индексирование
CREATE INDEX idx_path ON categories USING GIST (path);
```

### PostGIS

Геопространственные данные.

```sql
CREATE EXTENSION postgis;

-- Создание таблицы с геометрией
CREATE TABLE places (
    id SERIAL PRIMARY KEY,
    name TEXT,
    location GEOGRAPHY(Point, 4326)
);

-- Вставка точки
INSERT INTO places (name, location) VALUES
    ('Moscow', ST_Point(37.6173, 55.7558));

-- Поиск в радиусе
SELECT name, ST_Distance(location, ST_Point(37.6, 55.7)::geography) as distance
FROM places
WHERE ST_DWithin(location, ST_Point(37.6, 55.7)::geography, 10000)
ORDER BY distance;

-- Пространственный индекс
CREATE INDEX idx_location ON places USING GIST (location);
```

### pg_cron

Планировщик задач внутри PostgreSQL.

```ini
# postgresql.conf
shared_preload_libraries = 'pg_cron'
cron.database_name = 'postgres'
```

```sql
CREATE EXTENSION pg_cron;

-- Ежедневная очистка в полночь
SELECT cron.schedule('cleanup', '0 0 * * *',
    $$DELETE FROM logs WHERE created_at < now() - interval '30 days'$$);

-- Каждые 5 минут
SELECT cron.schedule('refresh-stats', '*/5 * * * *',
    $$REFRESH MATERIALIZED VIEW stats_view$$);

-- Просмотр задач
SELECT * FROM cron.job;

-- Удаление задачи
SELECT cron.unschedule('cleanup');
```

### auto_explain

Автоматическое логирование планов запросов.

```ini
# postgresql.conf
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = '1s'
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_format = 'json'
```

### pg_repack

Онлайн-реорганизация таблиц без блокировок.

```bash
# Установка (из пакета или исходников)
sudo apt install postgresql-16-repack

# Использование
pg_repack -d mydb -t bloated_table
```

```sql
CREATE EXTENSION pg_repack;
```

### pg_partman

Автоматическое управление партиционированием.

```sql
CREATE EXTENSION pg_partman;

-- Создание партиционированной таблицы
CREATE TABLE events (
    id SERIAL,
    created_at TIMESTAMP NOT NULL,
    data JSONB
) PARTITION BY RANGE (created_at);

-- Настройка автоматического партиционирования
SELECT partman.create_parent(
    p_parent_table := 'public.events',
    p_control := 'created_at',
    p_type := 'native',
    p_interval := 'daily',
    p_premake := 7
);

-- Автоматическое обслуживание (запускать по cron)
SELECT partman.run_maintenance();
```

### timescaledb

Оптимизированное хранение временных рядов.

```sql
CREATE EXTENSION timescaledb;

-- Создание hypertable
CREATE TABLE metrics (
    time TIMESTAMPTZ NOT NULL,
    device_id INTEGER,
    value DOUBLE PRECISION
);

SELECT create_hypertable('metrics', 'time');

-- Автоматическое сжатие старых данных
ALTER TABLE metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'device_id'
);

SELECT add_compression_policy('metrics', INTERVAL '7 days');
```

## Создание собственных расширений

### Структура расширения

```
my_extension/
├── my_extension.control    # метаданные
├── my_extension--1.0.sql   # SQL скрипт
└── Makefile                # для сборки
```

### control файл

```ini
# my_extension.control
comment = 'My custom extension'
default_version = '1.0'
module_pathname = '$libdir/my_extension'
relocatable = true
```

### SQL файл

```sql
-- my_extension--1.0.sql

-- создание функции
CREATE FUNCTION my_function(text) RETURNS text
AS 'SELECT $1 || '' processed'''
LANGUAGE SQL IMMUTABLE;

-- создание типа, операторов и т.д.
```

## Поиск расширений

### PGXN (PostgreSQL Extension Network)

```bash
# Установка клиента PGXN
pip install pgxnclient

# Поиск расширения
pgxn search keyword

# Установка
pgxn install extension_name
```

### Репозитории пакетов

```bash
# Debian/Ubuntu
apt search postgresql-16-

# RHEL/CentOS
yum search postgresql16-
```

## Типичные ошибки

1. **Забыли перезапустить после shared_preload_libraries** — расширение не загружено
2. **Установка расширения в неверную схему** — конфликты имён
3. **Игнорирование зависимостей** — расширение не работает
4. **Использование неподдерживаемых версий** — несовместимость

## Best Practices

1. **Устанавливайте расширения в отдельную схему** — легче управлять
2. **Документируйте используемые расширения** — для воспроизводимости
3. **Тестируйте обновления расширений** — могут быть breaking changes
4. **Мониторьте влияние на производительность** — особенно для preload расширений
5. **Используйте расширения из официальных источников** — безопасность
6. **Планируйте миграцию** при обновлении PostgreSQL — расширения могут требовать обновления

```sql
-- Полезный запрос: все зависимости расширений
SELECT
    e.extname,
    e.extversion,
    n.nspname as schema,
    e.extrelocatable
FROM pg_extension e
JOIN pg_namespace n ON e.extnamespace = n.oid
ORDER BY e.extname;
```

## Рекомендуемый набор расширений

```sql
-- Для большинства production систем
CREATE EXTENSION pg_stat_statements;  -- статистика запросов
CREATE EXTENSION pg_trgm;             -- полнотекстовый поиск
CREATE EXTENSION "uuid-ossp";         -- UUID (или gen_random_uuid)
CREATE EXTENSION pgcrypto;            -- криптография

-- По необходимости
CREATE EXTENSION hstore;              -- key-value хранение
CREATE EXTENSION ltree;               -- иерархии
CREATE EXTENSION postgis;             -- геоданные
```
