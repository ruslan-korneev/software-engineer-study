# Схемы и базы данных в PostgreSQL

[prev: 03-tables-rows-columns](./03-tables-rows-columns.md) | [next: 05-data-types](./05-data-types.md)
---

## Иерархия объектов PostgreSQL

PostgreSQL организует данные в трёхуровневую иерархию:

```
Cluster (Кластер PostgreSQL)
├── Database: postgres (системная)
├── Database: template0 (шаблон, неизменяемый)
├── Database: template1 (шаблон, настраиваемый)
└── Database: myapp (пользовательская)
    ├── Schema: public (по умолчанию)
    │   ├── Table: users
    │   ├── Table: orders
    │   └── ...
    ├── Schema: auth
    │   ├── Table: sessions
    │   └── Function: authenticate()
    └── Schema: reports
        ├── View: monthly_sales
        └── Materialized View: yearly_summary
```

## Базы данных (Databases)

### Что такое база данных в PostgreSQL?

**База данных** — это изолированный контейнер для хранения схем, таблиц и других объектов. Ключевая особенность: **нельзя выполнять запросы между разными базами данных** в одном SQL-запросе (без расширений).

### Системные базы данных

```sql
-- Просмотр всех баз данных
SELECT datname, datistemplate, datallowconn
FROM pg_database;

-- datname      | datistemplate | datallowconn
-- -------------|---------------|-------------
-- postgres     | f             | t           -- для администрирования
-- template1    | t             | t           -- шаблон (можно менять)
-- template0    | t             | f           -- чистый шаблон (read-only)
-- myapp        | f             | t           -- пользовательская
```

- **postgres** — база по умолчанию для подключения, используется для администрирования
- **template1** — шаблон для новых баз данных (изменения применяются к новым БД)
- **template0** — чистый неизменяемый шаблон для восстановления

### Создание базы данных

```sql
-- Простое создание
CREATE DATABASE myapp;

-- С параметрами
CREATE DATABASE myapp
    WITH
    OWNER = myuser                    -- владелец
    ENCODING = 'UTF8'                 -- кодировка
    LC_COLLATE = 'ru_RU.UTF-8'       -- правила сортировки
    LC_CTYPE = 'ru_RU.UTF-8'         -- классификация символов
    TABLESPACE = pg_default           -- табличное пространство
    CONNECTION LIMIT = 100            -- лимит подключений
    TEMPLATE = template0;             -- шаблон

-- Создание из шаблона с расширениями (template1)
-- Сначала установите расширения в template1:
-- \c template1
-- CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE DATABASE newapp TEMPLATE template1;
```

### Управление базами данных

```sql
-- Переименование
ALTER DATABASE myapp RENAME TO myapp_v2;

-- Изменение владельца
ALTER DATABASE myapp OWNER TO newowner;

-- Изменение параметров
ALTER DATABASE myapp SET timezone TO 'Europe/Moscow';
ALTER DATABASE myapp SET work_mem TO '256MB';

-- Просмотр настроек базы данных
SELECT name, setting
FROM pg_settings
WHERE name IN ('timezone', 'work_mem');

-- Удаление базы данных
DROP DATABASE myapp;

-- Удаление с принудительным отключением пользователей (PostgreSQL 13+)
DROP DATABASE myapp WITH (FORCE);

-- Проверка активных подключений перед удалением
SELECT pid, usename, application_name
FROM pg_stat_activity
WHERE datname = 'myapp';

-- Завершение подключений вручную
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'myapp' AND pid <> pg_backend_pid();
```

### Подключение к базе данных

```bash
# Из командной строки
psql -h localhost -U username -d myapp

# Или с использованием connection string
psql "postgresql://username:password@localhost:5432/myapp"
```

```sql
-- Внутри psql переключение между базами
\c myapp
-- You are now connected to database "myapp"

-- Текущая база данных
SELECT current_database();
```

## Схемы (Schemas)

### Что такое схема?

**Схема** — это пространство имён (namespace) внутри базы данных. Она позволяет:
- Логически группировать объекты
- Избежать конфликтов имён
- Управлять правами доступа на уровне группы объектов

### Схема public

По умолчанию все объекты создаются в схеме `public`:

```sql
-- Эти два запроса эквивалентны
CREATE TABLE users (id SERIAL PRIMARY KEY);
CREATE TABLE public.users (id SERIAL PRIMARY KEY);

-- Полное имя объекта: schema.table
SELECT * FROM public.users;
```

### Создание и управление схемами

```sql
-- Создание схемы
CREATE SCHEMA sales;

-- Создание с владельцем
CREATE SCHEMA hr AUTHORIZATION hr_admin;

-- Создание схемы с объектами
CREATE SCHEMA inventory
    CREATE TABLE products (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100)
    )
    CREATE TABLE warehouses (
        id SERIAL PRIMARY KEY,
        location VARCHAR(200)
    );

-- Переименование
ALTER SCHEMA sales RENAME TO commerce;

-- Изменение владельца
ALTER SCHEMA hr OWNER TO new_hr_admin;

-- Удаление пустой схемы
DROP SCHEMA sales;

-- Удаление с содержимым
DROP SCHEMA sales CASCADE;
```

### Search Path

**Search path** определяет порядок поиска объектов в схемах:

```sql
-- Просмотр текущего search_path
SHOW search_path;
-- "$user", public

-- PostgreSQL ищет объекты в следующем порядке:
-- 1. Схема с именем текущего пользователя (если существует)
-- 2. Схема public

-- Изменение search_path для сессии
SET search_path TO sales, public;

-- Изменение search_path для пользователя (постоянно)
ALTER ROLE myuser SET search_path TO myschema, public;

-- Изменение search_path для базы данных
ALTER DATABASE myapp SET search_path TO app, public;

-- Пример использования
SET search_path TO hr, sales, public;

-- Теперь можно без указания схемы
SELECT * FROM employees;  -- ищет в hr, затем sales, затем public

-- Явное указание схемы всегда работает
SELECT * FROM sales.orders;
```

### Практический пример: мультитенантность

```sql
-- Создание схем для разных клиентов (арендаторов)
CREATE SCHEMA tenant_acme;
CREATE SCHEMA tenant_globex;
CREATE SCHEMA shared;

-- Общие таблицы
CREATE TABLE shared.products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    price NUMERIC(10, 2)
);

-- Данные арендатора ACME
CREATE TABLE tenant_acme.orders (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES shared.products(id),
    quantity INTEGER
);

-- Данные арендатора Globex
CREATE TABLE tenant_globex.orders (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES shared.products(id),
    quantity INTEGER
);

-- При подключении арендатора устанавливаем search_path
-- Приложение динамически устанавливает схему
SET search_path TO tenant_acme, shared;
SELECT * FROM orders;  -- видит только свои заказы
```

### Организация по функциональности

```sql
-- Разделение по функциональным областям
CREATE SCHEMA core;       -- основные бизнес-сущности
CREATE SCHEMA auth;       -- аутентификация
CREATE SCHEMA audit;      -- логирование изменений
CREATE SCHEMA reports;    -- аналитические представления
CREATE SCHEMA api;        -- объекты для API

-- Пример структуры
CREATE TABLE core.users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL
);

CREATE TABLE auth.sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id INTEGER REFERENCES core.users(id),
    expires_at TIMESTAMP
);

CREATE TABLE audit.user_changes (
    id SERIAL PRIMARY KEY,
    user_id INTEGER,
    changed_at TIMESTAMP DEFAULT NOW(),
    old_data JSONB,
    new_data JSONB
);

CREATE VIEW reports.active_users AS
SELECT u.*, COUNT(s.id) AS session_count
FROM core.users u
LEFT JOIN auth.sessions s ON s.user_id = u.id
WHERE s.expires_at > NOW()
GROUP BY u.id;
```

## Права доступа

### Права на базу данных

```sql
-- Разрешить подключение к базе данных
GRANT CONNECT ON DATABASE myapp TO myuser;

-- Разрешить создание схем
GRANT CREATE ON DATABASE myapp TO admin_user;

-- Запретить подключение (для обслуживания)
REVOKE CONNECT ON DATABASE myapp FROM PUBLIC;
```

### Права на схему

```sql
-- Разрешить использование схемы (доступ к объектам)
GRANT USAGE ON SCHEMA sales TO sales_team;

-- Разрешить создание объектов в схеме
GRANT CREATE ON SCHEMA sales TO sales_admin;

-- Разрешить всё
GRANT ALL ON SCHEMA sales TO sales_admin;

-- Права по умолчанию для новых объектов
ALTER DEFAULT PRIVILEGES IN SCHEMA sales
    GRANT SELECT ON TABLES TO sales_team;

ALTER DEFAULT PRIVILEGES IN SCHEMA sales
    GRANT EXECUTE ON FUNCTIONS TO sales_team;
```

### Пример: настройка прав для приложения

```sql
-- Создание ролей
CREATE ROLE app_read;    -- только чтение
CREATE ROLE app_write;   -- чтение и запись
CREATE ROLE app_admin;   -- полный доступ

-- Создание пользователя приложения
CREATE USER app_service WITH PASSWORD 'secret';
GRANT app_write TO app_service;

-- Настройка прав
GRANT USAGE ON SCHEMA public TO app_read, app_write;

-- Права на существующие таблицы
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_read;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_write;

-- Права на будущие таблицы
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO app_read;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_write;

-- Права на последовательности (для SERIAL/IDENTITY)
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_write;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT USAGE ON SEQUENCES TO app_write;
```

## Системные схемы

```sql
-- information_schema — стандартные метаданные (SQL стандарт)
SELECT table_name, table_type
FROM information_schema.tables
WHERE table_schema = 'public';

-- pg_catalog — системный каталог PostgreSQL
SELECT relname, relkind
FROM pg_catalog.pg_class
WHERE relnamespace = 'public'::regnamespace;

-- pg_toast — хранение больших значений
-- pg_temp_N — временные таблицы сессии N
```

## Best Practices

### Организация баз данных

```sql
-- 1. Одно приложение = одна база данных
-- Упрощает резервное копирование и миграции

-- 2. Используйте template1 для общих расширений
\c template1
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- 3. Настройте параметры по умолчанию
ALTER DATABASE myapp SET timezone TO 'UTC';
ALTER DATABASE myapp SET statement_timeout TO '30s';
```

### Организация схем

```sql
-- 1. Не используйте public для production-данных
-- Создавайте отдельные схемы

-- 2. Группируйте по функциональности или по арендаторам

-- 3. Отзовите права на public у PUBLIC
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON DATABASE myapp FROM PUBLIC;

-- 4. Устанавливайте search_path явно
ALTER DATABASE myapp SET search_path TO app, public;
```

### Именование

```sql
-- Используйте snake_case
CREATE SCHEMA user_management;  -- хорошо
CREATE SCHEMA UserManagement;   -- плохо (PostgreSQL приведёт к нижнему регистру)

-- Добавляйте префиксы для ясности
CREATE SCHEMA tenant_acme;
CREATE SCHEMA module_billing;
CREATE SCHEMA ext_stripe;  -- интеграция с внешним сервисом
```

## Сравнение: База данных vs Схема

| Аспект | База данных | Схема |
|--------|-------------|-------|
| Изоляция | Полная | Логическая (внутри БД) |
| Cross-запросы | Невозможны (без dblink/FDW) | Возможны |
| Подключение | Требует переподключения | Не требует |
| Резервное копирование | pg_dump по БД | Внутри одного дампа |
| Права | Отдельная настройка | Наследуются от БД |
| Использование | Разные приложения | Модули одного приложения |

Схемы и базы данных — мощные инструменты организации данных в PostgreSQL. Правильное их использование упрощает управление правами, миграции и масштабирование приложений.

---
[prev: 03-tables-rows-columns](./03-tables-rows-columns.md) | [next: 05-data-types](./05-data-types.md)