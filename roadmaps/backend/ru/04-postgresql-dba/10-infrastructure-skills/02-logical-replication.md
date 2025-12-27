# Логическая репликация в PostgreSQL

## Что такое логическая репликация

**Логическая репликация** — это метод репликации данных в PostgreSQL, который передает изменения на уровне строк (INSERT, UPDATE, DELETE) между серверами. В отличие от физической репликации, логическая репликация позволяет:

- Реплицировать отдельные таблицы, а не весь кластер
- Реплицировать между разными мажорными версиями PostgreSQL
- Реплицировать в другие системы баз данных
- Выполнять трансформацию данных при репликации

## Архитектура логической репликации

### Основные компоненты

```
┌─────────────────┐                    ┌─────────────────┐
│   Publisher     │                    │   Subscriber    │
│   (Источник)    │                    │   (Приемник)    │
│                 │                    │                 │
│  ┌───────────┐  │   WAL-сообщения   │  ┌───────────┐  │
│  │ Publication│ ├──────────────────▶│  │Subscription│  │
│  └───────────┘  │                    │  └───────────┘  │
│                 │                    │                 │
│  ┌───────────┐  │                    │  ┌───────────┐  │
│  │  Tables   │  │                    │  │  Tables   │  │
│  └───────────┘  │                    │  └───────────┘  │
└─────────────────┘                    └─────────────────┘
```

### Ключевые понятия

- **Publication** — набор таблиц на источнике, изменения которых публикуются
- **Subscription** — подписка на приемнике для получения изменений
- **Replication Slot** — механизм отслеживания прогресса репликации
- **Logical Decoding** — процесс преобразования WAL в логические изменения

## Настройка Publisher (источник)

### 1. Конфигурация postgresql.conf

```ini
# Включить логическую репликацию
wal_level = logical

# Максимальное количество слотов репликации
max_replication_slots = 10

# Максимальное количество отправителей WAL
max_wal_senders = 10

# Сохранение WAL сегментов
wal_keep_size = 1GB
```

### 2. Настройка pg_hba.conf

```conf
# Разрешить подключение для репликации
host    replication     replication_user    10.0.0.0/24    scram-sha-256
host    mydb            replication_user    10.0.0.0/24    scram-sha-256
```

### 3. Создание пользователя для репликации

```sql
-- Создание пользователя с правами репликации
CREATE USER replication_user WITH REPLICATION LOGIN PASSWORD 'secure_password';

-- Предоставление прав на чтение таблиц
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replication_user;
```

### 4. Создание публикации

```sql
-- Публикация всех таблиц
CREATE PUBLICATION my_publication FOR ALL TABLES;

-- Публикация конкретных таблиц
CREATE PUBLICATION my_publication FOR TABLE users, orders, products;

-- Публикация с фильтром (PostgreSQL 15+)
CREATE PUBLICATION my_publication FOR TABLE orders WHERE (status = 'completed');

-- Публикация только определенных операций
CREATE PUBLICATION my_publication FOR TABLE users
    WITH (publish = 'insert, update');
```

## Настройка Subscriber (приемник)

### 1. Создание таблиц (должны существовать)

```sql
-- Таблицы должны иметь идентичную структуру
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT,
    total DECIMAL(10,2),
    status VARCHAR(50)
);
```

### 2. Создание подписки

```sql
-- Базовая подписка
CREATE SUBSCRIPTION my_subscription
    CONNECTION 'host=publisher_host port=5432 dbname=mydb user=replication_user password=secure_password'
    PUBLICATION my_publication;

-- Подписка с параметрами
CREATE SUBSCRIPTION my_subscription
    CONNECTION 'host=publisher_host port=5432 dbname=mydb user=replication_user password=secure_password'
    PUBLICATION my_publication
    WITH (
        copy_data = true,           -- Копировать существующие данные
        create_slot = true,         -- Создать слот репликации
        enabled = true,             -- Включить сразу
        synchronous_commit = 'off', -- Асинхронная запись
        streaming = true            -- Потоковая передача транзакций
    );
```

## Управление репликацией

### Мониторинг на Publisher

```sql
-- Список публикаций
SELECT * FROM pg_publication;

-- Таблицы в публикации
SELECT * FROM pg_publication_tables;

-- Статус слотов репликации
SELECT
    slot_name,
    plugin,
    slot_type,
    active,
    restart_lsn,
    confirmed_flush_lsn
FROM pg_replication_slots;

-- Статус отправки WAL
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    sync_state
FROM pg_stat_replication;
```

### Мониторинг на Subscriber

```sql
-- Список подписок
SELECT * FROM pg_subscription;

-- Статус подписки
SELECT
    subname,
    received_lsn,
    latest_end_lsn,
    latest_end_time
FROM pg_stat_subscription;

-- Статус синхронизации таблиц
SELECT
    srsubid,
    srrelid::regclass,
    srsubstate,
    srsublsn
FROM pg_subscription_rel;
```

### Управление подписками

```sql
-- Отключить подписку
ALTER SUBSCRIPTION my_subscription DISABLE;

-- Включить подписку
ALTER SUBSCRIPTION my_subscription ENABLE;

-- Обновить параметры
ALTER SUBSCRIPTION my_subscription SET (synchronous_commit = 'on');

-- Обновить строку подключения
ALTER SUBSCRIPTION my_subscription
    CONNECTION 'host=new_host port=5432 dbname=mydb user=replication_user password=new_password';

-- Добавить публикацию
ALTER SUBSCRIPTION my_subscription ADD PUBLICATION another_publication;

-- Обновить данные (пересинхронизация)
ALTER SUBSCRIPTION my_subscription REFRESH PUBLICATION;

-- Удалить подписку
DROP SUBSCRIPTION my_subscription;
```

## Конфликты и их разрешение

### Типы конфликтов

1. **Нарушение PRIMARY KEY** — строка уже существует
2. **Нарушение UNIQUE** — дублирование уникального значения
3. **Нарушение FOREIGN KEY** — связанная запись не существует
4. **Нарушение NOT NULL** — отсутствие обязательного значения

### Обнаружение конфликтов

```sql
-- Проверка логов сервера
-- В postgresql.conf:
log_min_messages = warning

-- В логах будут сообщения вида:
-- ERROR: duplicate key value violates unique constraint
```

### Разрешение конфликтов

```sql
-- 1. Удалить конфликтующую строку на приемнике
DELETE FROM users WHERE id = 123;

-- 2. Пропустить транзакцию (осторожно!)
-- Найти LSN проблемной транзакции и переместить
ALTER SUBSCRIPTION my_subscription
    SKIP (lsn = '0/123456');  -- PostgreSQL 15+

-- 3. Временно отключить репликацию таблицы
ALTER SUBSCRIPTION my_subscription
    SET PUBLICATION my_publication
    WITH (refresh = false);
```

## Двунаправленная репликация

### Настройка (PostgreSQL 16+)

```sql
-- На сервере A
CREATE PUBLICATION pub_a FOR ALL TABLES;

CREATE SUBSCRIPTION sub_from_b
    CONNECTION 'host=server_b ...'
    PUBLICATION pub_b
    WITH (origin = none);  -- Не реплицировать изменения от других источников

-- На сервере B
CREATE PUBLICATION pub_b FOR ALL TABLES;

CREATE SUBSCRIPTION sub_from_a
    CONNECTION 'host=server_a ...'
    PUBLICATION pub_a
    WITH (origin = none);
```

### Предотвращение циклов

```sql
-- Использование триггеров для отслеживания источника
CREATE OR REPLACE FUNCTION prevent_replication_loop()
RETURNS TRIGGER AS $$
BEGIN
    IF current_setting('session_replication_role') = 'replica' THEN
        RETURN NULL;  -- Не обрабатывать реплицированные изменения
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

## Миграция с минимальным простоем

### Пошаговый план

```sql
-- 1. На источнике: создать публикацию
CREATE PUBLICATION migration_pub FOR ALL TABLES;

-- 2. На приемнике: создать структуру БД
pg_dump --schema-only source_db | psql target_db

-- 3. На приемнике: создать подписку
CREATE SUBSCRIPTION migration_sub
    CONNECTION 'host=source_host ...'
    PUBLICATION migration_pub;

-- 4. Дождаться синхронизации
SELECT * FROM pg_stat_subscription;

-- 5. Переключить приложение на новый сервер

-- 6. На приемнике: удалить подписку
DROP SUBSCRIPTION migration_sub;
```

## Репликация с фильтрацией

### Фильтрация по строкам (PostgreSQL 15+)

```sql
-- Публикация только активных пользователей
CREATE PUBLICATION active_users_pub
    FOR TABLE users WHERE (status = 'active');

-- Публикация заказов за последний месяц
CREATE PUBLICATION recent_orders_pub
    FOR TABLE orders WHERE (created_at > CURRENT_DATE - INTERVAL '30 days');
```

### Фильтрация по колонкам (PostgreSQL 15+)

```sql
-- Публикация без чувствительных данных
CREATE PUBLICATION public_data_pub
    FOR TABLE users (id, name, created_at);  -- Без email и password
```

### Фильтрация по операциям

```sql
-- Только INSERT операции
CREATE PUBLICATION inserts_only_pub
    FOR TABLE audit_log
    WITH (publish = 'insert');

-- INSERT и UPDATE, без DELETE
CREATE PUBLICATION no_delete_pub
    FOR TABLE products
    WITH (publish = 'insert, update');
```

## Best Practices

### Производительность

1. **Используйте PRIMARY KEY** на всех реплицируемых таблицах
2. **Мониторьте отставание** репликации
3. **Настройте wal_keep_size** для предотвращения потери WAL
4. **Используйте streaming** для больших транзакций

```sql
-- Добавить REPLICA IDENTITY для таблиц без PK
ALTER TABLE my_table REPLICA IDENTITY FULL;

-- Или использовать уникальный индекс
CREATE UNIQUE INDEX ON my_table (column1, column2);
ALTER TABLE my_table REPLICA IDENTITY USING INDEX my_table_column1_column2_idx;
```

### Безопасность

```sql
-- Минимальные права для пользователя репликации
GRANT SELECT ON table1, table2 TO replication_user;
GRANT USAGE ON SCHEMA public TO replication_user;
```

### Мониторинг

```sql
-- Отставание репликации
SELECT
    slot_name,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag_size
FROM pg_replication_slots
WHERE slot_type = 'logical';

-- Алерт при большом отставании
SELECT CASE
    WHEN pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) > 1073741824
    THEN 'CRITICAL: Replication lag > 1GB'
    ELSE 'OK'
END
FROM pg_replication_slots;
```

## Типичные ошибки и решения

### Ошибка: "replication slot ... does not exist"

```sql
-- На источнике создать слот вручную
SELECT pg_create_logical_replication_slot('my_subscription', 'pgoutput');
```

### Ошибка: "table ... is not part of the publication"

```sql
-- Добавить таблицу в публикацию
ALTER PUBLICATION my_publication ADD TABLE new_table;

-- На приемнике обновить подписку
ALTER SUBSCRIPTION my_subscription REFRESH PUBLICATION;
```

### Ошибка: "schema ... does not exist"

```sql
-- Создать схему на приемнике
CREATE SCHEMA IF NOT EXISTS my_schema;

-- Структура таблиц должна быть идентичной
pg_dump --schema-only --schema=my_schema | psql target_db
```

### Накопление WAL

```sql
-- Проверить неактивные слоты
SELECT slot_name, active, restart_lsn
FROM pg_replication_slots
WHERE NOT active;

-- Удалить неиспользуемые слоты
SELECT pg_drop_replication_slot('unused_slot');
```

## Сравнение с физической репликацией

| Характеристика | Логическая | Физическая |
|----------------|------------|------------|
| Уровень репликации | Таблицы/строки | Весь кластер |
| Версии PostgreSQL | Разные | Одинаковые |
| Структура таблиц | Может отличаться | Идентичная |
| Чтение с реплики | Да, и запись | Только чтение |
| Производительность | Выше нагрузка | Ниже нагрузка |
| Начальная синхронизация | Долгая | Быстрая (pg_basebackup) |

## Полезные ссылки

- [Официальная документация по логической репликации](https://www.postgresql.org/docs/current/logical-replication.html)
- [Publication](https://www.postgresql.org/docs/current/sql-createpublication.html)
- [Subscription](https://www.postgresql.org/docs/current/sql-createsubscription.html)
