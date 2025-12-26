# Миграции схемы базы данных PostgreSQL

## Введение

Миграции базы данных (Database Migrations) - это способ управления изменениями структуры базы данных версионированным и воспроизводимым образом. Миграции позволяют отслеживать историю изменений схемы, откатывать изменения и синхронизировать структуру БД между различными окружениями.

## Зачем нужны миграции

### Проблемы без миграций

```sql
-- Разработчик 1 добавил колонку
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Разработчик 2 не знает об этом и получает ошибку
-- Как синхронизировать staging и production?
-- Как откатить изменения?
-- Как узнать текущую версию схемы?
```

### Преимущества миграций

1. **Версионирование** - каждое изменение имеет версию
2. **Воспроизводимость** - одинаковая схема на всех окружениях
3. **Откат** - возможность отката изменений
4. **История** - полная история изменений схемы
5. **Командная работа** - координация изменений между разработчиками
6. **CI/CD** - автоматическое применение миграций при деплое

## Типы миграций

### State-based (Declarative)

```sql
-- Описываем желаемое состояние
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Инструмент сам вычисляет diff и генерирует ALTER
```

**Примеры инструментов:** pgModeler, dbForge, некоторые ORM

### Change-based (Imperative)

```sql
-- Описываем конкретные изменения
-- V1__create_users_table.sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL
);

-- V2__add_name_to_users.sql
ALTER TABLE users ADD COLUMN name VARCHAR(100);
```

**Примеры инструментов:** Flyway, Liquibase, Alembic, Django Migrations

## Структура миграций

### Именование файлов

```
migrations/
├── V001__create_users_table.sql
├── V002__create_orders_table.sql
├── V003__add_phone_to_users.sql
├── V004__create_products_table.sql
└── V005__add_indexes.sql
```

**Соглашения об именовании:**
- `V{version}__{description}.sql` - Flyway
- `{timestamp}_{description}.sql` - Rails/Django
- `{version}_{description}.up.sql` / `.down.sql` - golang-migrate

### Структура файла миграции

```sql
-- migrations/V003__add_phone_to_users.sql

-- Описание изменения
-- Добавляет поле телефона в таблицу пользователей
-- Автор: developer@company.com
-- Дата: 2024-01-15

-- UP Migration
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Создание индекса для поиска по телефону
CREATE INDEX CONCURRENTLY idx_users_phone ON users(phone);

-- DOWN Migration (в отдельном файле или в комментариях)
-- DROP INDEX CONCURRENTLY idx_users_phone;
-- ALTER TABLE users DROP COLUMN phone;
```

## Паттерны миграций

### Добавление колонки (безопасно)

```sql
-- V010__add_status_to_orders.sql

-- Добавление колонки с NULL (не блокирует таблицу)
ALTER TABLE orders ADD COLUMN status VARCHAR(20);

-- Установка значения по умолчанию для новых записей
ALTER TABLE orders ALTER COLUMN status SET DEFAULT 'pending';

-- Заполнение существующих записей (батчами для больших таблиц)
UPDATE orders SET status = 'completed' WHERE status IS NULL;

-- Добавление NOT NULL constraint после заполнения
ALTER TABLE orders ALTER COLUMN status SET NOT NULL;
```

### Добавление колонки с NOT NULL и DEFAULT (PostgreSQL 11+)

```sql
-- В PostgreSQL 11+ это операция только с метаданными (мгновенная)
ALTER TABLE orders
ADD COLUMN status VARCHAR(20) NOT NULL DEFAULT 'pending';
```

### Удаление колонки (безопасно)

```sql
-- V011__remove_legacy_field.sql

-- Шаг 1: Убедиться что колонка не используется в коде
-- Шаг 2: Удалить зависимые индексы
DROP INDEX IF EXISTS idx_users_legacy_field;

-- Шаг 3: Удалить колонку
ALTER TABLE users DROP COLUMN legacy_field;
```

### Переименование колонки

```sql
-- V012__rename_user_name_to_full_name.sql

-- Вариант 1: Прямое переименование (блокирует на короткое время)
ALTER TABLE users RENAME COLUMN name TO full_name;

-- Вариант 2: Безопасное переименование для больших таблиц
-- Шаг 1: Добавить новую колонку
ALTER TABLE users ADD COLUMN full_name VARCHAR(100);

-- Шаг 2: Копировать данные (в фоне)
UPDATE users SET full_name = name WHERE full_name IS NULL;

-- Шаг 3: Обновить приложение для записи в обе колонки
-- Шаг 4: Удалить старую колонку (в следующей миграции)
```

### Изменение типа колонки

```sql
-- V013__change_price_type.sql

-- Безопасное изменение типа (если совместимо)
ALTER TABLE products ALTER COLUMN price TYPE NUMERIC(12,2);

-- Изменение с конвертацией данных
ALTER TABLE products
ALTER COLUMN price TYPE NUMERIC(12,2)
USING price::NUMERIC(12,2);

-- Для несовместимых типов - создание новой колонки
ALTER TABLE products ADD COLUMN price_new NUMERIC(12,2);
UPDATE products SET price_new = price::NUMERIC(12,2);
ALTER TABLE products DROP COLUMN price;
ALTER TABLE products RENAME COLUMN price_new TO price;
```

### Создание индекса (безопасно)

```sql
-- V014__add_index_on_email.sql

-- CONCURRENTLY не блокирует таблицу для записи
-- НО: нельзя использовать в транзакции
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);

-- Для уникального индекса
CREATE UNIQUE INDEX CONCURRENTLY idx_users_email_unique ON users(email);
```

### Разделение таблицы

```sql
-- V015__split_addresses_table.sql

-- Создание новой таблицы
CREATE TABLE addresses (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    street VARCHAR(255),
    city VARCHAR(100),
    country VARCHAR(100),
    postal_code VARCHAR(20)
);

-- Миграция данных
INSERT INTO addresses (user_id, street, city, country, postal_code)
SELECT id, address_street, address_city, address_country, address_postal
FROM users
WHERE address_street IS NOT NULL;

-- Удаление старых колонок (в отдельной миграции после обновления кода)
-- ALTER TABLE users DROP COLUMN address_street;
-- ALTER TABLE users DROP COLUMN address_city;
-- ...
```

## Миграции для больших таблиц

### Батчевое обновление данных

```sql
-- V020__backfill_user_status.sql

-- Функция для батчевого обновления
DO $$
DECLARE
    batch_size INTEGER := 10000;
    affected_rows INTEGER;
BEGIN
    LOOP
        UPDATE users
        SET status = 'active'
        WHERE id IN (
            SELECT id FROM users
            WHERE status IS NULL
            LIMIT batch_size
            FOR UPDATE SKIP LOCKED
        );

        GET DIAGNOSTICS affected_rows = ROW_COUNT;

        IF affected_rows = 0 THEN
            EXIT;
        END IF;

        -- Пауза для снижения нагрузки
        PERFORM pg_sleep(0.1);

        COMMIT;
    END LOOP;
END $$;
```

### Использование pg_repack для перестройки таблиц

```bash
# Установка pg_repack
sudo apt install postgresql-15-repack

# Перестройка таблицы без блокировки
pg_repack -d mydb -t large_table
```

### Партиционирование существующей таблицы

```sql
-- V025__partition_orders_table.sql

-- Шаг 1: Создание партиционированной таблицы
CREATE TABLE orders_partitioned (
    id SERIAL,
    user_id INTEGER NOT NULL,
    created_at TIMESTAMP NOT NULL,
    total NUMERIC(10,2),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- Шаг 2: Создание партиций
CREATE TABLE orders_2024_q1 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE orders_2024_q2 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- Шаг 3: Миграция данных (батчами)
INSERT INTO orders_partitioned
SELECT * FROM orders
WHERE created_at >= '2024-01-01' AND created_at < '2024-04-01';

-- Шаг 4: Переключение (в maintenance window)
BEGIN;
ALTER TABLE orders RENAME TO orders_old;
ALTER TABLE orders_partitioned RENAME TO orders;
COMMIT;

-- Шаг 5: Удаление старой таблицы (после проверки)
-- DROP TABLE orders_old;
```

## Откат миграций (Rollback)

### Структура down-миграций

```sql
-- migrations/V003__add_phone_to_users.up.sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
CREATE INDEX CONCURRENTLY idx_users_phone ON users(phone);

-- migrations/V003__add_phone_to_users.down.sql
DROP INDEX CONCURRENTLY IF EXISTS idx_users_phone;
ALTER TABLE users DROP COLUMN IF EXISTS phone;
```

### Необратимые миграции

```sql
-- V030__drop_legacy_data.sql

-- ВНИМАНИЕ: Эта миграция необратима!
-- Убедитесь что данные были перенесены

-- Удаление таблицы (нельзя откатить без бэкапа)
DROP TABLE legacy_users;

-- DOWN: NOT REVERSIBLE
-- Для отката необходимо восстановление из бэкапа
```

### Стратегии отката

```sql
-- Вариант 1: Полный откат
flyway undo  -- откатывает последнюю миграцию

-- Вариант 2: Компенсирующая миграция
-- V031__revert_v030.sql
CREATE TABLE legacy_users AS SELECT * FROM backup_legacy_users;

-- Вариант 3: Восстановление из бэкапа
pg_restore -d mydb -t legacy_users backup.dump
```

## Тестирование миграций

### Локальное тестирование

```bash
#!/bin/bash

# Создание тестовой базы
createdb migration_test

# Применение всех миграций
flyway -url=jdbc:postgresql://localhost/migration_test migrate

# Проверка схемы
pg_dump -s migration_test > current_schema.sql

# Откат
flyway -url=jdbc:postgresql://localhost/migration_test undo

# Повторное применение
flyway -url=jdbc:postgresql://localhost/migration_test migrate

# Очистка
dropdb migration_test
```

### CI/CD проверки

```yaml
# .github/workflows/migrations.yml
name: Test Migrations

on:
  pull_request:
    paths:
      - 'migrations/**'

jobs:
  test-migrations:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v3

      - name: Run migrations
        run: |
          flyway -url=jdbc:postgresql://localhost/postgres \
                 -user=postgres \
                 -password=postgres \
                 migrate

      - name: Test rollback
        run: |
          flyway -url=jdbc:postgresql://localhost/postgres \
                 -user=postgres \
                 -password=postgres \
                 undo

      - name: Re-apply migrations
        run: |
          flyway -url=jdbc:postgresql://localhost/postgres \
                 -user=postgres \
                 -password=postgres \
                 migrate
```

## Best Practices

### 1. Одна миграция - одно изменение

```sql
-- ПЛОХО: Много изменений в одной миграции
-- V001__initial.sql содержит 500 строк

-- ХОРОШО: Разбиваем на логические части
-- V001__create_users.sql
-- V002__create_orders.sql
-- V003__create_products.sql
-- V004__add_indexes.sql
```

### 2. Избегайте изменения примененных миграций

```sql
-- НИКОГДА не меняйте уже примененную миграцию!
-- Вместо этого создайте новую миграцию с исправлением

-- V010__add_email_index.sql (уже применена с ошибкой)
-- НЕ редактируем!

-- V011__fix_email_index.sql (новая миграция)
DROP INDEX IF EXISTS idx_users_email;
CREATE INDEX CONCURRENTLY idx_users_email ON users(lower(email));
```

### 3. Транзакционность

```sql
-- Каждая миграция должна быть атомарной
BEGIN;

CREATE TABLE new_table (...);
ALTER TABLE existing_table ADD COLUMN ...;
INSERT INTO new_table SELECT ...;

COMMIT;

-- Исключение: CREATE INDEX CONCURRENTLY не работает в транзакции
```

### 4. Обратная совместимость

```sql
-- Применяйте изменения в несколько этапов:

-- Этап 1: Добавляем новое (код работает со старым)
ALTER TABLE users ADD COLUMN new_field VARCHAR(100);

-- Этап 2: Обновляем код для работы с обоими полями
-- Деплой нового кода

-- Этап 3: Мигрируем данные
UPDATE users SET new_field = old_field WHERE new_field IS NULL;

-- Этап 4: Удаляем старое (после проверки)
ALTER TABLE users DROP COLUMN old_field;
```

### 5. Документация миграций

```sql
-- V050__restructure_permissions.sql
--
-- Цель: Переход от role-based к permission-based системе
-- Автор: security-team
-- Задача: JIRA-1234
-- Зависимости: V049
-- Время выполнения: ~5 минут на production
-- Откат: V050_rollback.sql
--
-- Изменения:
-- 1. Создание таблицы permissions
-- 2. Миграция данных из roles
-- 3. Создание связей user_permissions

CREATE TABLE permissions (
    ...
);
```

### 6. Мониторинг миграций

```sql
-- Логирование времени выполнения
CREATE TABLE migration_log (
    version VARCHAR(50) PRIMARY KEY,
    description VARCHAR(255),
    started_at TIMESTAMP,
    finished_at TIMESTAMP,
    execution_time_ms INTEGER,
    success BOOLEAN,
    error_message TEXT
);

-- Проверка статуса
SELECT version, description,
       execution_time_ms / 1000.0 as seconds,
       success
FROM migration_log
ORDER BY started_at DESC
LIMIT 10;
```

## Схема процесса миграций

```
Development          Staging              Production
    |                   |                     |
    v                   v                     v
[Написание] -----> [Тестирование] -----> [Применение]
    |                   |                     |
    |              [Code Review]              |
    |                   |                     |
    +----------- [CI/CD Pipeline] -----------+
                       |
               [Мониторинг и алерты]
```

## Заключение

Миграции базы данных - критически важная часть процесса разработки. Ключевые принципы:

1. **Версионируйте все изменения** - каждое изменение схемы должно быть миграцией
2. **Делайте миграции обратимыми** - всегда пишите down-миграции
3. **Тестируйте миграции** - проверяйте на тестовом окружении перед production
4. **Применяйте постепенно** - разбивайте большие изменения на этапы
5. **Документируйте** - описывайте цель и последствия миграции
6. **Мониторьте выполнение** - отслеживайте время и ошибки

Следующий шаг - изучение конкретных инструментов миграций (Flyway, Liquibase, Alembic и др.).
