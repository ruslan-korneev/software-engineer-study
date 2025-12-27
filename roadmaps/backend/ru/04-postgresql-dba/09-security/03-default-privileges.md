# Привилегии по умолчанию (Default Privileges)

## Введение

Default Privileges (привилегии по умолчанию) позволяют автоматически назначать привилегии на объекты, которые будут созданы в будущем. Это решает важную проблему: обычные команды `GRANT` действуют только на существующие объекты, а новые объекты создаются без автоматических прав для других пользователей.

## Проблема, которую решают Default Privileges

```sql
-- Выдаём права на все существующие таблицы
GRANT SELECT ON ALL TABLES IN SCHEMA public TO analyst;

-- Создаём новую таблицу
CREATE TABLE new_reports (id SERIAL, data TEXT);

-- analyst НЕ имеет доступа к new_reports!
-- Нужно снова выполнить GRANT
```

С Default Privileges:

```sql
-- Настраиваем права для будущих таблиц
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO analyst;

-- Теперь все новые таблицы автоматически получат права для analyst
CREATE TABLE new_reports (id SERIAL, data TEXT);
-- analyst уже имеет SELECT на new_reports!
```

## Синтаксис ALTER DEFAULT PRIVILEGES

### Основной синтаксис

```sql
ALTER DEFAULT PRIVILEGES
    [ FOR { ROLE | USER } target_role [, ...] ]
    [ IN SCHEMA schema_name [, ...] ]
    grant_or_revoke
```

Где `grant_or_revoke`:

```sql
GRANT { { SELECT | INSERT | UPDATE | DELETE | TRUNCATE | REFERENCES | TRIGGER }
    [, ...] | ALL [ PRIVILEGES ] }
    ON TABLES
    TO { [ GROUP ] role_name | PUBLIC } [, ...] [ WITH GRANT OPTION ]

GRANT { { USAGE | SELECT | UPDATE }
    [, ...] | ALL [ PRIVILEGES ] }
    ON SEQUENCES
    TO { [ GROUP ] role_name | PUBLIC } [, ...] [ WITH GRANT OPTION ]

GRANT { EXECUTE | ALL [ PRIVILEGES ] }
    ON { FUNCTIONS | ROUTINES }
    TO { [ GROUP ] role_name | PUBLIC } [, ...] [ WITH GRANT OPTION ]

GRANT { USAGE | ALL [ PRIVILEGES ] }
    ON TYPES
    TO { [ GROUP ] role_name | PUBLIC } [, ...] [ WITH GRANT OPTION ]

GRANT { USAGE | CREATE | ALL [ PRIVILEGES ] }
    ON SCHEMAS
    TO { [ GROUP ] role_name | PUBLIC } [, ...] [ WITH GRANT OPTION ]
```

## Практические примеры

### Базовая настройка для приложения

```sql
-- Создаём роли
CREATE ROLE app_user;
CREATE ROLE readonly_user;
CREATE ROLE app_admin;

-- Настраиваем default privileges для схемы public
-- Все новые таблицы будут доступны для чтения readonly_user
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO readonly_user;

-- app_user получает полный CRUD на новые таблицы
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_user;

-- app_user также нужны последовательности для INSERT
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT USAGE ON SEQUENCES TO app_user;

-- app_admin получает всё
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT ALL PRIVILEGES ON TABLES TO app_admin;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT ALL PRIVILEGES ON SEQUENCES TO app_admin;
```

### FOR ROLE - права для объектов определённого создателя

```sql
-- Когда developer создаёт таблицы, app_user автоматически получит SELECT
ALTER DEFAULT PRIVILEGES FOR ROLE developer IN SCHEMA public
GRANT SELECT ON TABLES TO app_user;

-- Когда dba создаёт таблицы, все получают SELECT
ALTER DEFAULT PRIVILEGES FOR ROLE dba IN SCHEMA public
GRANT SELECT ON TABLES TO PUBLIC;
```

### Настройка для нескольких схем

```sql
-- Создаём схемы
CREATE SCHEMA sales;
CREATE SCHEMA inventory;
CREATE SCHEMA analytics;

-- Настраиваем права в каждой схеме
ALTER DEFAULT PRIVILEGES IN SCHEMA sales, inventory
GRANT SELECT, INSERT, UPDATE ON TABLES TO operations_team;

ALTER DEFAULT PRIVILEGES IN SCHEMA analytics
GRANT SELECT ON TABLES TO bi_team;

-- Не забываем про USAGE на схемы!
GRANT USAGE ON SCHEMA sales, inventory TO operations_team;
GRANT USAGE ON SCHEMA analytics TO bi_team;
```

### Полная настройка для production окружения

```sql
-- Создание ролей
CREATE ROLE db_admin;        -- Полные права
CREATE ROLE app_backend;     -- Backend приложения
CREATE ROLE app_readonly;    -- Только чтение
CREATE ROLE etl_service;     -- ETL процессы

-- Настройка default privileges
-- Для таблиц
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT ALL ON TABLES TO db_admin;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_backend;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO app_readonly;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT, INSERT ON TABLES TO etl_service;

-- Для последовательностей
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT ALL ON SEQUENCES TO db_admin;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT USAGE ON SEQUENCES TO app_backend;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT USAGE ON SEQUENCES TO etl_service;

-- Для функций (отзываем PUBLIC и выдаём только нужным ролям)
ALTER DEFAULT PRIVILEGES IN SCHEMA public
REVOKE EXECUTE ON FUNCTIONS FROM PUBLIC;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT EXECUTE ON FUNCTIONS TO db_admin;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT EXECUTE ON FUNCTIONS TO app_backend;
```

## Отзыв Default Privileges

```sql
-- Отзыв привилегий по умолчанию
ALTER DEFAULT PRIVILEGES IN SCHEMA public
REVOKE SELECT ON TABLES FROM readonly_user;

-- Отзыв для конкретного создателя
ALTER DEFAULT PRIVILEGES FOR ROLE developer IN SCHEMA public
REVOKE ALL ON TABLES FROM app_user;
```

## Просмотр Default Privileges

### Через psql

```sql
-- Команда \ddp показывает default privileges
\ddp
```

### Через системные каталоги

```sql
-- Просмотр всех default privileges
SELECT
    pg_get_userbyid(d.defaclrole) AS owner,
    n.nspname AS schema,
    CASE d.defaclobjtype
        WHEN 'r' THEN 'tables'
        WHEN 'S' THEN 'sequences'
        WHEN 'f' THEN 'functions'
        WHEN 'T' THEN 'types'
        WHEN 'n' THEN 'schemas'
    END AS object_type,
    pg_catalog.array_to_string(d.defaclacl, E'\n') AS default_privileges
FROM pg_default_acl d
LEFT JOIN pg_namespace n ON n.oid = d.defaclnamespace
ORDER BY owner, schema, object_type;
```

### Детальный просмотр

```sql
-- Расшифровка ACL
SELECT
    pg_get_userbyid(d.defaclrole) AS owner,
    n.nspname AS schema,
    CASE d.defaclobjtype
        WHEN 'r' THEN 'TABLE'
        WHEN 'S' THEN 'SEQUENCE'
        WHEN 'f' THEN 'FUNCTION'
        WHEN 'T' THEN 'TYPE'
    END AS object_type,
    (aclexplode(d.defaclacl)).grantee::regrole AS grantee,
    (aclexplode(d.defaclacl)).privilege_type AS privilege,
    (aclexplode(d.defaclacl)).is_grantable AS grantable
FROM pg_default_acl d
LEFT JOIN pg_namespace n ON n.oid = d.defaclnamespace;
```

## Важные особенности

### 1. Default Privileges привязаны к создателю

```sql
-- Если alice настроила default privileges
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO reader;

-- Только таблицы, созданные alice, будут иметь эти права
-- Таблицы, созданные bob, НЕ получат эти права автоматически

-- Решение: настроить для конкретного создателя или от имени создателя
ALTER DEFAULT PRIVILEGES FOR ROLE bob IN SCHEMA public
GRANT SELECT ON TABLES TO reader;
```

### 2. Схемы без указания IN SCHEMA

```sql
-- Без IN SCHEMA - глобальные default privileges
ALTER DEFAULT PRIVILEGES
GRANT SELECT ON TABLES TO reader;

-- Применяется ко ВСЕМ схемам, где пользователь создаёт таблицы
```

### 3. Сочетание с обычным GRANT

```sql
-- Сначала права на существующие объекты
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;

-- Затем права на будущие объекты
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO readonly_user;
```

## Best Practices

### 1. Настраивайте Default Privileges при создании схемы

```sql
-- Создание схемы с полной настройкой прав
CREATE SCHEMA reporting;

-- Права на схему
GRANT USAGE ON SCHEMA reporting TO bi_team;
GRANT USAGE, CREATE ON SCHEMA reporting TO etl_service;

-- Права на существующие объекты (если есть)
GRANT SELECT ON ALL TABLES IN SCHEMA reporting TO bi_team;
GRANT ALL ON ALL TABLES IN SCHEMA reporting TO etl_service;

-- Права на будущие объекты
ALTER DEFAULT PRIVILEGES IN SCHEMA reporting
GRANT SELECT ON TABLES TO bi_team;

ALTER DEFAULT PRIVILEGES IN SCHEMA reporting
GRANT ALL ON TABLES TO etl_service;

ALTER DEFAULT PRIVILEGES IN SCHEMA reporting
GRANT USAGE ON SEQUENCES TO etl_service;
```

### 2. Используйте роли-группы

```sql
-- Создание роли-группы
CREATE ROLE data_readers;
CREATE ROLE data_writers;

-- Настройка default privileges для групп
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO data_readers;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO data_writers;

-- Добавление пользователей в группы
GRANT data_readers TO alice, bob;
GRANT data_writers TO charlie;
```

### 3. Не забывайте про функции

```sql
-- Функции по умолчанию доступны PUBLIC
-- Отключаем это для новых функций
ALTER DEFAULT PRIVILEGES IN SCHEMA public
REVOKE EXECUTE ON FUNCTIONS FROM PUBLIC;

-- Выдаём права только нужным ролям
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT EXECUTE ON FUNCTIONS TO trusted_role;
```

### 4. Документируйте настройки

```sql
-- Создайте скрипт инициализации для каждой среды
-- init_privileges.sql

-- Роли
CREATE ROLE IF NOT EXISTS app_user;
CREATE ROLE IF NOT EXISTS readonly_user;

-- Существующие объекты
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_user;

-- Будущие объекты
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO readonly_user;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT USAGE ON SEQUENCES TO app_user;

-- Логирование
COMMENT ON SCHEMA public IS 'Default privileges configured on YYYY-MM-DD';
```

## Типичные ошибки

### 1. Права настроены только для текущего пользователя

```sql
-- Ошибка: alice настроила права, но таблицы создаёт bob
-- Решение: настроить FOR ROLE bob
ALTER DEFAULT PRIVILEGES FOR ROLE bob IN SCHEMA public
GRANT SELECT ON TABLES TO reader;

-- Или выполнить от имени bob
SET ROLE bob;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO reader;
RESET ROLE;
```

### 2. Забыли про последовательности

```sql
-- Выдали права на таблицы
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT ALL ON TABLES TO app_user;

-- Но app_user не может делать INSERT в таблицы с SERIAL!
-- Нужно добавить:
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT USAGE ON SEQUENCES TO app_user;
```

### 3. Не синхронизировали существующие и будущие объекты

```sql
-- Только future privileges
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO reader;

-- reader не видит существующие таблицы!
-- Добавляем:
GRANT SELECT ON ALL TABLES IN SCHEMA public TO reader;
```

### 4. Путаница с отзывом

```sql
-- Отзыв default privileges НЕ влияет на уже созданные объекты
ALTER DEFAULT PRIVILEGES IN SCHEMA public
REVOKE SELECT ON TABLES FROM reader;

-- reader всё ещё имеет SELECT на ранее созданные таблицы
-- Нужно явно отозвать:
REVOKE SELECT ON ALL TABLES IN SCHEMA public FROM reader;
```

## Полезные скрипты

### Синхронизация прав существующих и будущих объектов

```sql
-- Функция для полной настройки прав роли
CREATE OR REPLACE FUNCTION setup_role_privileges(
    p_schema_name TEXT,
    p_role_name TEXT,
    p_table_privs TEXT[],
    p_seq_privs TEXT[]
) RETURNS void AS $$
DECLARE
    v_table_privs TEXT := array_to_string(p_table_privs, ', ');
    v_seq_privs TEXT := array_to_string(p_seq_privs, ', ');
BEGIN
    -- Права на существующие таблицы
    EXECUTE format('GRANT %s ON ALL TABLES IN SCHEMA %I TO %I',
                   v_table_privs, p_schema_name, p_role_name);

    -- Права на существующие последовательности
    IF array_length(p_seq_privs, 1) > 0 THEN
        EXECUTE format('GRANT %s ON ALL SEQUENCES IN SCHEMA %I TO %I',
                       v_seq_privs, p_schema_name, p_role_name);
    END IF;

    -- Default privileges для таблиц
    EXECUTE format('ALTER DEFAULT PRIVILEGES IN SCHEMA %I GRANT %s ON TABLES TO %I',
                   p_schema_name, v_table_privs, p_role_name);

    -- Default privileges для последовательностей
    IF array_length(p_seq_privs, 1) > 0 THEN
        EXECUTE format('ALTER DEFAULT PRIVILEGES IN SCHEMA %I GRANT %s ON SEQUENCES TO %I',
                       p_schema_name, v_seq_privs, p_role_name);
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Использование
SELECT setup_role_privileges(
    'public',
    'app_user',
    ARRAY['SELECT', 'INSERT', 'UPDATE', 'DELETE'],
    ARRAY['USAGE']
);
```

## Связанные темы

- [Привилегии объектов](./01-object-privileges.md) - типы привилегий
- [GRANT и REVOKE](./02-grant-revoke.md) - команды управления привилегиями
- [Роли](./06-roles.md) - создание и управление ролями
