# Привилегии объектов в PostgreSQL

[prev: 08-postgresql-conf](../08-configuring-postgresql/08-postgresql-conf.md) | [next: 02-grant-revoke](./02-grant-revoke.md)

---

## Введение

Привилегии объектов (Object Privileges) в PostgreSQL определяют, какие действия пользователи или роли могут выполнять над конкретными объектами базы данных. Это фундаментальный механизм контроля доступа, обеспечивающий безопасность данных.

## Типы привилегий

### Привилегии для таблиц

| Привилегия | Описание |
|------------|----------|
| `SELECT` | Чтение данных из таблицы |
| `INSERT` | Добавление новых строк |
| `UPDATE` | Изменение существующих строк |
| `DELETE` | Удаление строк |
| `TRUNCATE` | Очистка таблицы (TRUNCATE TABLE) |
| `REFERENCES` | Создание внешних ключей, ссылающихся на таблицу |
| `TRIGGER` | Создание триггеров на таблице |

### Привилегии для последовательностей (SEQUENCE)

| Привилегия | Описание |
|------------|----------|
| `USAGE` | Использование currval и nextval |
| `SELECT` | Чтение текущего значения (currval) |
| `UPDATE` | Изменение значения (setval, nextval) |

### Привилегии для функций и процедур

| Привилегия | Описание |
|------------|----------|
| `EXECUTE` | Право вызывать функцию или процедуру |

### Привилегии для схем

| Привилегия | Описание |
|------------|----------|
| `CREATE` | Создание объектов в схеме |
| `USAGE` | Доступ к объектам в схеме |

### Привилегии для баз данных

| Привилегия | Описание |
|------------|----------|
| `CREATE` | Создание схем в базе данных |
| `CONNECT` | Право подключаться к базе данных |
| `TEMPORARY` | Создание временных таблиц |

### Привилегии для табличных пространств

| Привилегия | Описание |
|------------|----------|
| `CREATE` | Создание таблиц, индексов в табличном пространстве |

## Просмотр привилегий

### Просмотр привилегий таблицы

```sql
-- Через psql команду
\dp имя_таблицы

-- Через системные каталоги
SELECT
    grantee,
    privilege_type,
    is_grantable
FROM information_schema.table_privileges
WHERE table_name = 'employees';
```

### Расшифровка ACL (Access Control List)

PostgreSQL использует компактный формат для отображения привилегий:

```
grantee=privileges/grantor
```

Коды привилегий:
- `r` - SELECT (read)
- `w` - UPDATE (write)
- `a` - INSERT (append)
- `d` - DELETE
- `D` - TRUNCATE
- `x` - REFERENCES
- `t` - TRIGGER
- `X` - EXECUTE
- `U` - USAGE
- `C` - CREATE
- `c` - CONNECT
- `T` - TEMPORARY

Пример:
```sql
-- Вывод: {alice=arwd/postgres,bob=r/postgres}
-- alice имеет INSERT, SELECT, UPDATE, DELETE от postgres
-- bob имеет только SELECT от postgres
```

### Просмотр привилегий схемы

```sql
SELECT
    n.nspname AS schema_name,
    pg_catalog.pg_get_userbyid(n.nspowner) AS owner,
    n.nspacl AS access_privileges
FROM pg_catalog.pg_namespace n
WHERE n.nspname NOT LIKE 'pg_%'
  AND n.nspname != 'information_schema';
```

## Примеры использования

### Базовые операции

```sql
-- Создание тестовой таблицы
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    salary NUMERIC(10, 2),
    department VARCHAR(50)
);

-- Создание роли
CREATE ROLE hr_manager;

-- Выдача привилегии SELECT
GRANT SELECT ON employees TO hr_manager;

-- Проверка привилегий
\dp employees
```

### Привилегии на уровне столбцов

```sql
-- Разрешить чтение только определённых столбцов
GRANT SELECT (id, name, department) ON employees TO public_viewer;

-- Разрешить обновление только определённых столбцов
GRANT UPDATE (department) ON employees TO team_lead;

-- Проверка привилегий на столбцы
SELECT
    column_name,
    privilege_type
FROM information_schema.column_privileges
WHERE table_name = 'employees'
  AND grantee = 'team_lead';
```

### Работа с последовательностями

```sql
-- Создание последовательности
CREATE SEQUENCE order_seq;

-- Выдача права использовать последовательность
GRANT USAGE ON SEQUENCE order_seq TO app_user;

-- Выдача всех прав на последовательность
GRANT ALL ON SEQUENCE order_seq TO admin_user;
```

### Привилегии на функции

```sql
-- Создание функции
CREATE FUNCTION get_salary(emp_id INT)
RETURNS NUMERIC AS $$
BEGIN
    RETURN (SELECT salary FROM employees WHERE id = emp_id);
END;
$$ LANGUAGE plpgsql;

-- Отзыв публичного доступа (по умолчанию все могут выполнять)
REVOKE EXECUTE ON FUNCTION get_salary(INT) FROM PUBLIC;

-- Выдача права конкретной роли
GRANT EXECUTE ON FUNCTION get_salary(INT) TO hr_manager;
```

## Владелец объекта

Владелец объекта автоматически получает все привилегии на объект и может передавать их другим:

```sql
-- Смена владельца таблицы
ALTER TABLE employees OWNER TO new_owner;

-- Смена владельца схемы
ALTER SCHEMA hr OWNER TO hr_admin;

-- Просмотр владельца
SELECT
    tablename,
    tableowner
FROM pg_tables
WHERE schemaname = 'public';
```

## WITH GRANT OPTION

Позволяет получателю привилегии передавать её другим:

```sql
-- Выдача привилегии с правом передачи
GRANT SELECT ON employees TO manager WITH GRANT OPTION;

-- Теперь manager может делать:
GRANT SELECT ON employees TO subordinate;

-- Просмотр информации о праве передачи
SELECT
    grantee,
    privilege_type,
    is_grantable
FROM information_schema.table_privileges
WHERE table_name = 'employees';
```

## Специальные роли PUBLIC и ALL

### PUBLIC

`PUBLIC` - это псевдороль, представляющая всех пользователей:

```sql
-- Выдать всем право на чтение
GRANT SELECT ON public_data TO PUBLIC;

-- Отозвать у всех право на подключение
REVOKE CONNECT ON DATABASE mydb FROM PUBLIC;
```

### ALL PRIVILEGES

```sql
-- Выдать все привилегии на таблицу
GRANT ALL PRIVILEGES ON employees TO admin;

-- Выдать все привилегии на все таблицы в схеме
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO admin;
```

## Best Practices

### 1. Принцип минимальных привилегий

```sql
-- Плохо: выдавать все права
GRANT ALL ON employees TO app_user;

-- Хорошо: выдавать только необходимые
GRANT SELECT, INSERT ON employees TO app_user;
```

### 2. Использование ролей-групп

```sql
-- Создание группы
CREATE ROLE readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;

-- Назначение роли пользователям
GRANT readonly TO alice, bob, charlie;
```

### 3. Ограничение доступа к чувствительным данным

```sql
-- Создание представления без чувствительных данных
CREATE VIEW employees_public AS
SELECT id, name, department
FROM employees;

GRANT SELECT ON employees_public TO PUBLIC;
REVOKE SELECT ON employees FROM PUBLIC;
```

### 4. Регулярный аудит привилегий

```sql
-- Просмотр всех привилегий пользователя
SELECT
    table_schema,
    table_name,
    privilege_type
FROM information_schema.table_privileges
WHERE grantee = 'username';
```

## Типичные ошибки

### 1. Забытые привилегии PUBLIC

```sql
-- Проблема: функции по умолчанию доступны PUBLIC
CREATE FUNCTION sensitive_operation() RETURNS void AS $$ ... $$;

-- Решение: отзывать привилегии сразу
REVOKE EXECUTE ON FUNCTION sensitive_operation() FROM PUBLIC;
```

### 2. Отсутствие USAGE на схеме

```sql
-- Ошибка: есть SELECT на таблицу, но нет USAGE на схему
GRANT SELECT ON hr.employees TO reader;
-- Пользователь всё равно не увидит таблицу!

-- Правильно:
GRANT USAGE ON SCHEMA hr TO reader;
GRANT SELECT ON hr.employees TO reader;
```

### 3. Каскадное удаление привилегий

```sql
-- Если отозвать привилегию у пользователя с GRANT OPTION,
-- все переданные им привилегии также будут отозваны
REVOKE SELECT ON employees FROM manager CASCADE;
```

## Связанные темы

- [GRANT и REVOKE](./02-grant-revoke.md) - команды управления привилегиями
- [Привилегии по умолчанию](./03-default-privileges.md) - настройка привилегий для новых объектов
- [Row Level Security](./04-row-level-security.md) - контроль доступа на уровне строк
- [Роли](./06-roles.md) - система ролей PostgreSQL

---

[prev: 08-postgresql-conf](../08-configuring-postgresql/08-postgresql-conf.md) | [next: 02-grant-revoke](./02-grant-revoke.md)
