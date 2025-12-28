# GRANT и REVOKE в PostgreSQL

[prev: 01-object-privileges](./01-object-privileges.md) | [next: 03-default-privileges](./03-default-privileges.md)

---

## Введение

`GRANT` и `REVOKE` - это основные SQL-команды для управления привилегиями в PostgreSQL. `GRANT` выдаёт привилегии пользователям и ролям, а `REVOKE` отзывает их. Эти команды являются частью стандарта SQL и реализуют дискреционный контроль доступа (DAC).

## Синтаксис GRANT

### Для таблиц

```sql
GRANT { { SELECT | INSERT | UPDATE | DELETE | TRUNCATE | REFERENCES | TRIGGER }
    [, ...] | ALL [ PRIVILEGES ] }
    ON { [ TABLE ] table_name [, ...]
         | ALL TABLES IN SCHEMA schema_name [, ...] }
    TO role_specification [, ...] [ WITH GRANT OPTION ]
    [ GRANTED BY role_specification ]
```

### Для столбцов

```sql
GRANT { { SELECT | INSERT | UPDATE | REFERENCES } ( column_name [, ...] )
    [, ...] | ALL [ PRIVILEGES ] ( column_name [, ...] ) }
    ON [ TABLE ] table_name [, ...]
    TO role_specification [, ...] [ WITH GRANT OPTION ]
```

### Для последовательностей

```sql
GRANT { { USAGE | SELECT | UPDATE }
    [, ...] | ALL [ PRIVILEGES ] }
    ON { SEQUENCE sequence_name [, ...]
         | ALL SEQUENCES IN SCHEMA schema_name [, ...] }
    TO role_specification [, ...]
```

### Для баз данных

```sql
GRANT { { CREATE | CONNECT | TEMPORARY | TEMP } [, ...] | ALL [ PRIVILEGES ] }
    ON DATABASE database_name [, ...]
    TO role_specification [, ...]
```

### Для схем

```sql
GRANT { { CREATE | USAGE } [, ...] | ALL [ PRIVILEGES ] }
    ON SCHEMA schema_name [, ...]
    TO role_specification [, ...]
```

### Для функций

```sql
GRANT { EXECUTE | ALL [ PRIVILEGES ] }
    ON { FUNCTION function_name [ ( [ [ argmode ] [ arg_name ] arg_type [, ...] ] ) ] [, ...]
         | ALL FUNCTIONS IN SCHEMA schema_name [, ...] }
    TO role_specification [, ...]
```

### Членство в ролях

```sql
GRANT role_name [, ...] TO role_specification [, ...]
    [ WITH { ADMIN | INHERIT | SET } { OPTION | TRUE | FALSE } ]
    [ GRANTED BY role_specification ]
```

## Синтаксис REVOKE

### Для таблиц

```sql
REVOKE [ GRANT OPTION FOR ]
    { { SELECT | INSERT | UPDATE | DELETE | TRUNCATE | REFERENCES | TRIGGER }
    [, ...] | ALL [ PRIVILEGES ] }
    ON { [ TABLE ] table_name [, ...]
         | ALL TABLES IN SCHEMA schema_name [, ...] }
    FROM role_specification [, ...]
    [ GRANTED BY role_specification ]
    [ CASCADE | RESTRICT ]
```

### Членство в ролях

```sql
REVOKE [ { ADMIN | INHERIT | SET } OPTION FOR ]
    role_name [, ...] FROM role_specification [, ...]
    [ GRANTED BY role_specification ]
    [ CASCADE | RESTRICT ]
```

## Практические примеры

### Базовые операции с таблицами

```sql
-- Создание таблицы и ролей для примеров
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_name VARCHAR(100),
    amount NUMERIC(10, 2),
    status VARCHAR(20),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE ROLE sales_team;
CREATE ROLE finance_team;
CREATE ROLE support_team;

-- Выдача отдельных привилегий
GRANT SELECT ON orders TO sales_team;
GRANT SELECT, UPDATE ON orders TO finance_team;
GRANT SELECT, INSERT, UPDATE, DELETE ON orders TO support_team;

-- Выдача всех привилегий
GRANT ALL PRIVILEGES ON orders TO admin_role;

-- Отзыв привилегий
REVOKE DELETE ON orders FROM support_team;
REVOKE ALL PRIVILEGES ON orders FROM admin_role;
```

### Привилегии на уровне столбцов

```sql
CREATE TABLE employee_salaries (
    id SERIAL PRIMARY KEY,
    employee_name VARCHAR(100),
    department VARCHAR(50),
    salary NUMERIC(10, 2),
    bonus NUMERIC(10, 2)
);

-- Разрешить HR видеть всё, а менеджерам - только имена и отделы
GRANT SELECT ON employee_salaries TO hr_team;
GRANT SELECT (id, employee_name, department) ON employee_salaries TO managers;

-- Разрешить обновлять только бонусы
GRANT UPDATE (bonus) ON employee_salaries TO managers;
```

### Работа со схемами

```sql
-- Создание схемы
CREATE SCHEMA analytics;

-- Минимальные права для работы со схемой
GRANT USAGE ON SCHEMA analytics TO analyst_role;

-- Права на создание объектов в схеме
GRANT CREATE ON SCHEMA analytics TO lead_analyst;

-- Права на все таблицы в схеме
GRANT SELECT ON ALL TABLES IN SCHEMA analytics TO analyst_role;
```

### Работа с последовательностями

```sql
CREATE SEQUENCE invoice_seq;

-- Для вставки данных часто нужен доступ к последовательности
GRANT USAGE ON SEQUENCE invoice_seq TO app_user;

-- Или для всех последовательностей в схеме
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_user;
```

### Работа с функциями

```sql
CREATE FUNCTION calculate_discount(price NUMERIC, discount_pct NUMERIC)
RETURNS NUMERIC AS $$
BEGIN
    RETURN price * (1 - discount_pct / 100);
END;
$$ LANGUAGE plpgsql;

-- По умолчанию функции доступны PUBLIC, отзываем
REVOKE EXECUTE ON FUNCTION calculate_discount(NUMERIC, NUMERIC) FROM PUBLIC;

-- Выдаём только нужным ролям
GRANT EXECUTE ON FUNCTION calculate_discount(NUMERIC, NUMERIC) TO sales_team;
```

## WITH GRANT OPTION

`WITH GRANT OPTION` позволяет получателю привилегии передавать её дальше:

```sql
-- Менеджер может передавать право SELECT своей команде
GRANT SELECT ON orders TO manager WITH GRANT OPTION;

-- Теперь менеджер может выполнить:
-- GRANT SELECT ON orders TO team_member;

-- Отзыв только права передачи, не самой привилегии
REVOKE GRANT OPTION FOR SELECT ON orders FROM manager;

-- Отзыв привилегии с каскадным удалением переданных прав
REVOKE SELECT ON orders FROM manager CASCADE;
```

### Пример каскадного отзыва

```sql
-- Сценарий:
-- 1. admin выдаёт права alice с WITH GRANT OPTION
GRANT SELECT ON products TO alice WITH GRANT OPTION;

-- 2. alice передаёт права bob
-- (выполняется от имени alice)
GRANT SELECT ON products TO bob;

-- 3. Если отозвать права у alice с CASCADE:
REVOKE SELECT ON products FROM alice CASCADE;
-- bob также потеряет права!

-- 4. Если использовать RESTRICT:
REVOKE SELECT ON products FROM alice RESTRICT;
-- Ошибка! Нельзя отозвать, пока bob имеет переданные права
```

## Членство в ролях

```sql
-- Создание ролей
CREATE ROLE developers;
CREATE ROLE senior_developers;
CREATE ROLE tech_leads;

-- Наследование ролей
GRANT developers TO senior_developers;
GRANT senior_developers TO tech_leads;

-- Добавление пользователя в роль
GRANT developers TO alice;

-- С правом администрирования роли
GRANT developers TO bob WITH ADMIN OPTION;
-- bob теперь может добавлять/удалять членов из developers

-- Контроль наследования привилегий (PostgreSQL 16+)
GRANT developers TO charlie WITH INHERIT FALSE;
-- charlie должен явно использовать SET ROLE developers

-- Отзыв членства
REVOKE developers FROM alice;
```

## Массовые операции

### Все объекты в схеме

```sql
-- Все текущие таблицы
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_role;

-- Все текущие последовательности
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_role;

-- Все текущие функции
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA public TO app_role;
```

### Несколько объектов

```sql
-- Несколько таблиц
GRANT SELECT ON orders, products, customers TO analyst;

-- Несколько схем
GRANT USAGE ON SCHEMA sales, marketing, analytics TO bi_tool;

-- Несколько баз данных
GRANT CONNECT ON DATABASE prod_db, staging_db TO developer;
```

## Привилегии для типов объектов

### Типы данных

```sql
CREATE TYPE mood AS ENUM ('happy', 'sad', 'neutral');

-- Право использовать тип
GRANT USAGE ON TYPE mood TO app_user;
```

### Домены

```sql
CREATE DOMAIN email AS VARCHAR(255)
    CHECK (VALUE ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

GRANT USAGE ON DOMAIN email TO app_user;
```

### Foreign Data Wrapper и Foreign Server

```sql
-- Права на FDW
GRANT USAGE ON FOREIGN DATA WRAPPER postgres_fdw TO analyst;

-- Права на внешний сервер
GRANT USAGE ON FOREIGN SERVER remote_db TO analyst;
```

## Просмотр выданных привилегий

```sql
-- Привилегии на таблицы
SELECT
    grantor,
    grantee,
    table_schema,
    table_name,
    privilege_type,
    is_grantable
FROM information_schema.table_privileges
WHERE table_schema = 'public'
ORDER BY table_name, grantee;

-- Привилегии на столбцы
SELECT
    grantor,
    grantee,
    table_name,
    column_name,
    privilege_type
FROM information_schema.column_privileges
WHERE table_name = 'employee_salaries';

-- Привилегии на схемы
SELECT
    n.nspname AS schema_name,
    pg_catalog.pg_get_userbyid(n.nspowner) AS owner,
    n.nspacl AS privileges
FROM pg_catalog.pg_namespace n
WHERE n.nspname NOT LIKE 'pg_%';

-- Через psql
\dp table_name     -- привилегии на таблицу
\dn+ schema_name   -- привилегии на схему
```

## Best Practices

### 1. Используйте роли, не отдельных пользователей

```sql
-- Плохо
GRANT SELECT ON orders TO alice;
GRANT SELECT ON orders TO bob;
GRANT SELECT ON orders TO charlie;

-- Хорошо
CREATE ROLE readers;
GRANT SELECT ON orders TO readers;
GRANT readers TO alice, bob, charlie;
```

### 2. Принцип минимальных привилегий

```sql
-- Плохо
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO app_user;

-- Хорошо
GRANT SELECT ON customers TO app_user;
GRANT SELECT, INSERT ON orders TO app_user;
GRANT UPDATE (status) ON orders TO app_user;
```

### 3. Отзывайте привилегии PUBLIC на функциях

```sql
-- Функции по умолчанию доступны всем
CREATE FUNCTION sensitive_operation() RETURNS void AS $$ ... $$;

-- Сразу отзываем публичный доступ
REVOKE EXECUTE ON FUNCTION sensitive_operation() FROM PUBLIC;
GRANT EXECUTE ON FUNCTION sensitive_operation() TO authorized_role;
```

### 4. Документируйте привилегии

```sql
-- Используйте комментарии
COMMENT ON ROLE sales_team IS 'Доступ на чтение к данным продаж';
COMMENT ON ROLE finance_team IS 'Полный доступ к финансовым отчётам';
```

### 5. Регулярно проводите аудит

```sql
-- Скрипт аудита привилегий
SELECT
    r.rolname AS role_name,
    t.schemaname,
    t.tablename,
    array_agg(p.privilege_type) AS privileges
FROM pg_roles r
JOIN information_schema.table_privileges p ON r.rolname = p.grantee
JOIN pg_tables t ON p.table_name = t.tablename AND p.table_schema = t.schemaname
WHERE r.rolname NOT LIKE 'pg_%'
GROUP BY r.rolname, t.schemaname, t.tablename
ORDER BY r.rolname, t.schemaname, t.tablename;
```

## Типичные ошибки

### 1. Забыли USAGE на схеме

```sql
-- Ошибка: пользователь не видит таблицу, хотя SELECT выдан
GRANT SELECT ON analytics.reports TO analyst;

-- Правильно: сначала USAGE на схему
GRANT USAGE ON SCHEMA analytics TO analyst;
GRANT SELECT ON analytics.reports TO analyst;
```

### 2. Забыли про последовательности

```sql
-- Ошибка: INSERT не работает из-за отсутствия доступа к serial
GRANT INSERT ON users TO app;
-- ERROR: permission denied for sequence users_id_seq

-- Правильно
GRANT INSERT ON users TO app;
GRANT USAGE ON SEQUENCE users_id_seq TO app;

-- Или для всех последовательностей
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app;
```

### 3. Путаница с CASCADE/RESTRICT

```sql
-- RESTRICT (по умолчанию) выдаст ошибку если есть зависимые привилегии
REVOKE SELECT ON orders FROM manager RESTRICT;
-- ERROR: dependent privileges exist

-- CASCADE отзовёт все зависимые привилегии
REVOKE SELECT ON orders FROM manager CASCADE;
```

### 4. Не обновили привилегии после смены владельца

```sql
-- После смены владельца старые GRANT'ы могут не работать как ожидалось
ALTER TABLE orders OWNER TO new_owner;
-- Проверьте и обновите привилегии!
```

## Связанные темы

- [Привилегии объектов](./01-object-privileges.md) - типы привилегий
- [Привилегии по умолчанию](./03-default-privileges.md) - автоматическое назначение
- [Роли](./06-roles.md) - создание и управление ролями

---

[prev: 01-object-privileges](./01-object-privileges.md) | [next: 03-default-privileges](./03-default-privileges.md)
