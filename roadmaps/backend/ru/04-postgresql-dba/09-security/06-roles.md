# Роли в PostgreSQL

[prev: 05-authentication-models](./05-authentication-models.md) | [next: 07-pg_hba-conf](./07-pg_hba-conf.md)

---

## Введение

В PostgreSQL концепция ролей объединяет понятия пользователей и групп. Роль может быть пользователем (может подключаться к БД), группой (содержит другие роли) или и тем, и другим одновременно. Это обеспечивает гибкую систему управления доступом.

## Создание ролей

### Базовый синтаксис

```sql
CREATE ROLE role_name [ [ WITH ] option [ ... ] ]
```

### Атрибуты ролей

| Атрибут | Описание |
|---------|----------|
| `SUPERUSER` / `NOSUPERUSER` | Суперпользователь (обходит все проверки доступа) |
| `CREATEDB` / `NOCREATEDB` | Может создавать базы данных |
| `CREATEROLE` / `NOCREATEROLE` | Может создавать/изменять/удалять роли |
| `INHERIT` / `NOINHERIT` | Автоматически наследует привилегии ролей-членов |
| `LOGIN` / `NOLOGIN` | Может подключаться к базе данных |
| `REPLICATION` / `NOREPLICATION` | Может инициировать репликацию |
| `BYPASSRLS` / `NOBYPASSRLS` | Обходит Row Level Security |
| `CONNECTION LIMIT` | Лимит одновременных подключений |
| `PASSWORD` | Установка пароля |
| `VALID UNTIL` | Срок действия пароля |
| `IN ROLE` | Членство в других ролях при создании |
| `IN GROUP` | То же, что IN ROLE (устаревший синтаксис) |
| `ROLE` | Роли, которые становятся членами новой роли |
| `ADMIN` | Роли с правом администрирования новой роли |

### Примеры создания ролей

```sql
-- Обычный пользователь с паролем
CREATE ROLE app_user WITH
    LOGIN
    PASSWORD 'secure_password'
    CONNECTION LIMIT 10;

-- Пользователь с временным паролем
CREATE ROLE temp_user WITH
    LOGIN
    PASSWORD 'temp_pass'
    VALID UNTIL '2024-12-31 23:59:59';

-- Роль-группа (без LOGIN)
CREATE ROLE developers;

-- Администратор базы данных
CREATE ROLE dba WITH
    LOGIN
    CREATEDB
    CREATEROLE
    PASSWORD 'dba_password';

-- Суперпользователь (использовать осторожно!)
CREATE ROLE admin WITH
    LOGIN
    SUPERUSER
    PASSWORD 'admin_password';

-- Роль для репликации
CREATE ROLE replicator WITH
    LOGIN
    REPLICATION
    PASSWORD 'repl_password';
```

### CREATE USER vs CREATE ROLE

```sql
-- CREATE USER = CREATE ROLE ... LOGIN
CREATE USER alice WITH PASSWORD 'pass';
-- Эквивалентно:
CREATE ROLE alice WITH LOGIN PASSWORD 'pass';
```

## Изменение ролей

### ALTER ROLE

```sql
-- Изменение пароля
ALTER ROLE app_user WITH PASSWORD 'new_password';

-- Добавление атрибутов
ALTER ROLE app_user WITH CREATEDB;

-- Удаление атрибутов
ALTER ROLE app_user WITH NOCREATEDB;

-- Изменение лимита подключений
ALTER ROLE app_user WITH CONNECTION LIMIT 20;

-- Продление срока действия
ALTER ROLE temp_user WITH VALID UNTIL '2025-12-31';

-- Переименование
ALTER ROLE old_name RENAME TO new_name;
```

### Настройки конфигурации для роли

```sql
-- Установить значение по умолчанию для роли
ALTER ROLE analyst SET search_path = analytics, public;
ALTER ROLE app_user SET statement_timeout = '30s';
ALTER ROLE reader SET default_transaction_read_only = on;

-- Для конкретной базы данных
ALTER ROLE app_user IN DATABASE myapp SET log_statement = 'all';

-- Сброс настройки
ALTER ROLE app_user RESET search_path;
ALTER ROLE app_user IN DATABASE myapp RESET ALL;
```

## Удаление ролей

```sql
-- Простое удаление
DROP ROLE role_name;

-- Проверка существования
DROP ROLE IF EXISTS role_name;

-- Перед удалением нужно:
-- 1. Удалить владение объектами
-- 2. Отозвать все привилегии
-- 3. Удалить членство в других ролях

-- Переназначение объектов и удаление привилегий
REASSIGN OWNED BY old_role TO new_role;
DROP OWNED BY old_role;
DROP ROLE old_role;
```

## Членство в ролях (Role Membership)

### GRANT роли другой роли

```sql
-- Создание группы
CREATE ROLE developers;
CREATE ROLE senior_developers;

-- Добавление пользователей в группу
GRANT developers TO alice, bob, charlie;

-- Иерархия групп
GRANT developers TO senior_developers;
GRANT senior_developers TO tech_lead;

-- С правом администрирования (может добавлять/удалять членов)
GRANT developers TO team_lead WITH ADMIN OPTION;

-- Контроль наследования (PostgreSQL 16+)
GRANT developers TO alice WITH INHERIT TRUE;   -- Наследует привилегии автоматически
GRANT developers TO bob WITH INHERIT FALSE;    -- Должен использовать SET ROLE
GRANT developers TO charlie WITH SET TRUE;     -- Может использовать SET ROLE
```

### REVOKE членства

```sql
-- Удаление из группы
REVOKE developers FROM alice;

-- Только право администрирования
REVOKE ADMIN OPTION FOR developers FROM team_lead;
```

## Наследование привилегий

### INHERIT vs NOINHERIT

```sql
-- Роль с наследованием (по умолчанию)
CREATE ROLE user1 WITH LOGIN INHERIT;

-- Роль без наследования
CREATE ROLE user2 WITH LOGIN NOINHERIT;

-- Привилегии группы
CREATE ROLE readers;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readers;

-- user1 автоматически получает SELECT (INHERIT)
GRANT readers TO user1;

-- user2 должен явно переключиться на роль readers
GRANT readers TO user2;
-- SET ROLE readers; -- тогда получит привилегии
```

### SET ROLE

```sql
-- Переключение на другую роль
SET ROLE developers;

-- Проверка текущей роли
SELECT current_user, session_user;

-- Возврат к исходной роли
RESET ROLE;
-- или
SET ROLE NONE;

-- Переключение с правами суперпользователя
SET ROLE postgres;  -- Требует членства или SUPERUSER
```

## Предопределённые роли

PostgreSQL включает набор предопределённых ролей для типичных задач:

| Роль | Описание |
|------|----------|
| `pg_read_all_data` | SELECT на все таблицы, представления, последовательности |
| `pg_write_all_data` | INSERT, UPDATE, DELETE на все таблицы |
| `pg_read_all_settings` | Чтение всех параметров конфигурации |
| `pg_read_all_stats` | Чтение всех статистических представлений |
| `pg_stat_scan_tables` | Выполнение функций мониторинга |
| `pg_monitor` | Чтение статистики и настроек |
| `pg_database_owner` | Владелец текущей базы данных |
| `pg_signal_backend` | Отправка сигналов другим backend процессам |
| `pg_read_server_files` | Чтение файлов на сервере (COPY и др.) |
| `pg_write_server_files` | Запись файлов на сервере |
| `pg_execute_server_program` | Выполнение программ на сервере |

```sql
-- Использование предопределённых ролей
GRANT pg_read_all_data TO analyst;
GRANT pg_monitor TO monitoring_user;

-- Комбинирование
GRANT pg_read_all_data, pg_read_all_stats TO bi_tool;
```

## Просмотр информации о ролях

### Через psql

```sql
-- Список всех ролей
\du

-- Детальная информация
\du+

-- Информация о конкретной роли
\du role_name
```

### Через системные каталоги

```sql
-- Все роли
SELECT
    rolname,
    rolsuper,
    rolinherit,
    rolcreaterole,
    rolcreatedb,
    rolcanlogin,
    rolreplication,
    rolbypassrls,
    rolconnlimit,
    rolvaliduntil
FROM pg_roles
ORDER BY rolname;

-- Членство в ролях
SELECT
    r.rolname AS role,
    m.rolname AS member,
    g.rolname AS grantor,
    am.admin_option
FROM pg_auth_members am
JOIN pg_roles r ON am.roleid = r.oid
JOIN pg_roles m ON am.member = m.oid
JOIN pg_roles g ON am.grantor = g.oid
ORDER BY role, member;

-- Привилегии роли
SELECT
    grantee,
    table_schema,
    table_name,
    privilege_type
FROM information_schema.table_privileges
WHERE grantee = 'role_name';
```

## Практические сценарии

### Типичная структура ролей для приложения

```sql
-- 1. Создание ролей-групп
CREATE ROLE app_readonly;    -- Только чтение
CREATE ROLE app_readwrite;   -- Чтение и запись
CREATE ROLE app_admin;       -- Администратор приложения

-- 2. Настройка привилегий групп
-- Readonly
GRANT USAGE ON SCHEMA public TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO app_readonly;

-- Readwrite = readonly + write
GRANT app_readonly TO app_readwrite;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_readwrite;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT INSERT, UPDATE, DELETE ON TABLES TO app_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT USAGE ON SEQUENCES TO app_readwrite;

-- Admin = readwrite + DDL
GRANT app_readwrite TO app_admin;
GRANT CREATE ON SCHEMA public TO app_admin;

-- 3. Создание пользователей
CREATE ROLE web_app WITH LOGIN PASSWORD 'web_pass';
CREATE ROLE api_service WITH LOGIN PASSWORD 'api_pass';
CREATE ROLE dba_user WITH LOGIN PASSWORD 'dba_pass';

-- 4. Назначение ролей
GRANT app_readwrite TO web_app;
GRANT app_readwrite TO api_service;
GRANT app_admin TO dba_user;
```

### Разделение по отделам

```sql
-- Группы по отделам
CREATE ROLE sales_team;
CREATE ROLE finance_team;
CREATE ROLE hr_team;

-- Схемы для каждого отдела
CREATE SCHEMA sales;
CREATE SCHEMA finance;
CREATE SCHEMA hr;

-- Привилегии на схемы
GRANT USAGE ON SCHEMA sales TO sales_team;
GRANT ALL ON ALL TABLES IN SCHEMA sales TO sales_team;

GRANT USAGE ON SCHEMA finance TO finance_team;
GRANT ALL ON ALL TABLES IN SCHEMA finance TO finance_team;

-- HR видит всех сотрудников (включая таблицы в других схемах)
GRANT USAGE ON SCHEMA hr, sales, finance TO hr_team;
GRANT SELECT ON hr.employees TO hr_team;
GRANT SELECT (id, name, department) ON sales.staff TO hr_team;
GRANT SELECT (id, name, department) ON finance.staff TO hr_team;
```

### Временный доступ

```sql
-- Временный пользователь для подрядчика
CREATE ROLE contractor WITH
    LOGIN
    PASSWORD 'temp_password'
    VALID UNTIL '2024-03-31 23:59:59'
    CONNECTION LIMIT 1;

GRANT app_readonly TO contractor;

-- Ограничение доступа только к определённым таблицам
REVOKE SELECT ON sensitive_data FROM app_readonly;
-- или использовать Row Level Security
```

### Сервисные аккаунты

```sql
-- Аккаунт для бэкапов
CREATE ROLE backup_user WITH
    LOGIN
    REPLICATION
    PASSWORD 'backup_pass';

GRANT pg_read_all_data TO backup_user;

-- Аккаунт для мониторинга
CREATE ROLE monitoring WITH
    LOGIN
    PASSWORD 'monitor_pass'
    CONNECTION LIMIT 3;

GRANT pg_monitor TO monitoring;

-- Аккаунт для ETL
CREATE ROLE etl_service WITH
    LOGIN
    PASSWORD 'etl_pass';

GRANT app_readwrite TO etl_service;
GRANT TRUNCATE ON staging_tables TO etl_service;
```

## Best Practices

### 1. Используйте роли-группы

```sql
-- Плохо: назначать привилегии напрямую пользователям
GRANT SELECT ON orders TO alice;
GRANT SELECT ON orders TO bob;

-- Хорошо: через группы
CREATE ROLE order_readers;
GRANT SELECT ON orders TO order_readers;
GRANT order_readers TO alice, bob;
```

### 2. Принцип минимальных привилегий

```sql
-- Плохо
CREATE ROLE app_user WITH SUPERUSER;

-- Хорошо: только необходимые права
CREATE ROLE app_user WITH LOGIN;
GRANT SELECT, INSERT ON app_tables TO app_user;
```

### 3. Используйте NOINHERIT для чувствительных ролей

```sql
-- Пользователь должен явно переключаться для получения admin привилегий
CREATE ROLE alice WITH LOGIN NOINHERIT;
GRANT admin_role TO alice;
-- alice должна выполнить SET ROLE admin_role для получения прав
```

### 4. Установите срок действия паролей

```sql
-- Политика: пароли истекают через 90 дней
CREATE ROLE user1 WITH
    LOGIN
    PASSWORD 'pass'
    VALID UNTIL (CURRENT_DATE + INTERVAL '90 days');
```

### 5. Ограничивайте подключения

```sql
-- Один пользователь - одно подключение
CREATE ROLE service_account WITH
    LOGIN
    CONNECTION LIMIT 1;
```

### 6. Документируйте роли

```sql
COMMENT ON ROLE app_readonly IS 'Read-only access to application tables';
COMMENT ON ROLE dba_user IS 'Database administrator for app_db';
```

## Типичные ошибки

### 1. Забыли LOGIN

```sql
-- Роль без LOGIN не может подключиться
CREATE ROLE app_user WITH PASSWORD 'pass';
-- ERROR: role "app_user" is not permitted to log in

-- Правильно
CREATE ROLE app_user WITH LOGIN PASSWORD 'pass';
```

### 2. Путаница с наследованием

```sql
-- NOINHERIT роль не получает привилегии автоматически
CREATE ROLE user1 WITH LOGIN NOINHERIT;
GRANT readers TO user1;
-- user1 НЕ имеет привилегий readers, пока не выполнит SET ROLE
```

### 3. Удаление роли с объектами

```sql
-- Ошибка: роль владеет объектами
DROP ROLE owner_role;
-- ERROR: role "owner_role" cannot be dropped because some objects depend on it

-- Правильно: сначала переназначить/удалить объекты
REASSIGN OWNED BY owner_role TO new_owner;
DROP OWNED BY owner_role;
DROP ROLE owner_role;
```

### 4. Суперпользователи для приложений

```sql
-- НИКОГДА не делайте это!
CREATE ROLE app_user WITH SUPERUSER;
-- Используйте только необходимые привилегии
```

## Связанные темы

- [Привилегии объектов](./01-object-privileges.md) - типы привилегий
- [GRANT и REVOKE](./02-grant-revoke.md) - управление привилегиями
- [pg_hba.conf](./07-pg_hba-conf.md) - настройка аутентификации
- [Модели аутентификации](./05-authentication-models.md) - методы аутентификации

---

[prev: 05-authentication-models](./05-authentication-models.md) | [next: 07-pg_hba-conf](./07-pg_hba-conf.md)
