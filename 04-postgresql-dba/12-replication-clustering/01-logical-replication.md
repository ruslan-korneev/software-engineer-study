# Логическая репликация PostgreSQL

## Введение

Логическая репликация — это метод репликации данных в PostgreSQL, при котором изменения передаются на уровне строк таблиц (INSERT, UPDATE, DELETE), а не на уровне блоков WAL-файлов. Это позволяет реплицировать данные между серверами PostgreSQL разных версий, выборочно реплицировать отдельные таблицы и выполнять трансформации данных на лету.

## Архитектура логической репликации

### Основные компоненты

```
┌─────────────────┐          ┌─────────────────┐
│   Publisher     │          │   Subscriber    │
│   (Primary)     │   WAL    │   (Replica)     │
│                 │ ──────►  │                 │
│ ┌─────────────┐ │          │ ┌─────────────┐ │
│ │ Publication │ │          │ │Subscription │ │
│ └─────────────┘ │          │ └─────────────┘ │
│        │        │          │        │        │
│        ▼        │          │        ▼        │
│   WAL Sender    │◄────────►│   Logical       │
│                 │          │   Worker        │
└─────────────────┘          └─────────────────┘
```

### Ключевые понятия

- **Publication** (Публикация) — набор таблиц, изменения которых будут реплицироваться
- **Subscription** (Подписка) — определяет подключение к публикации и какие данные получать
- **Replication Slot** (Слот репликации) — гарантирует, что данные не будут удалены до их получения подписчиком
- **Logical Decoding** — преобразование WAL-записей в логические изменения

## Настройка логической репликации

### Шаг 1: Конфигурация Publisher (мастер)

```bash
# postgresql.conf на publisher
wal_level = logical              # Обязательно для логической репликации
max_replication_slots = 10       # Количество слотов репликации
max_wal_senders = 10             # Количество процессов отправки WAL
```

```bash
# pg_hba.conf — разрешить подключение для репликации
host    replication     replicator    192.168.1.0/24    scram-sha-256
host    mydb            replicator    192.168.1.0/24    scram-sha-256
```

### Шаг 2: Создание пользователя для репликации

```sql
-- На publisher
CREATE USER replicator WITH REPLICATION PASSWORD 'secure_password';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replicator;

-- Или дать права на конкретные таблицы
GRANT SELECT ON TABLE users, orders, products TO replicator;
```

### Шаг 3: Создание публикации

```sql
-- Публикация всех таблиц в базе данных
CREATE PUBLICATION my_publication FOR ALL TABLES;

-- Публикация конкретных таблиц
CREATE PUBLICATION my_publication FOR TABLE users, orders, products;

-- Публикация с фильтрацией операций
CREATE PUBLICATION inserts_only FOR TABLE logs
    WITH (publish = 'insert');  -- Только INSERT

-- Публикация с фильтрацией строк (PostgreSQL 15+)
CREATE PUBLICATION active_users FOR TABLE users
    WHERE (status = 'active');
```

### Шаг 4: Настройка Subscriber (реплика)

```bash
# postgresql.conf на subscriber
max_replication_slots = 10
max_logical_replication_workers = 10
max_worker_processes = 20
```

### Шаг 5: Создание структуры таблиц на subscriber

```sql
-- Таблицы должны существовать до создания подписки
-- Структура должна быть совместимой

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    total DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Шаг 6: Создание подписки

```sql
-- Базовая подписка
CREATE SUBSCRIPTION my_subscription
    CONNECTION 'host=publisher_host port=5432 dbname=mydb user=replicator password=secure_password'
    PUBLICATION my_publication;

-- Подписка с начальной синхронизацией данных
CREATE SUBSCRIPTION my_subscription
    CONNECTION 'host=publisher_host port=5432 dbname=mydb user=replicator password=secure_password'
    PUBLICATION my_publication
    WITH (copy_data = true);  -- По умолчанию true

-- Подписка без начальной синхронизации
CREATE SUBSCRIPTION my_subscription
    CONNECTION 'host=publisher_host port=5432 dbname=mydb user=replicator password=secure_password'
    PUBLICATION my_publication
    WITH (copy_data = false, create_slot = true);

-- Подписка на несколько публикаций
CREATE SUBSCRIPTION multi_sub
    CONNECTION 'host=publisher_host port=5432 dbname=mydb user=replicator password=secure_password'
    PUBLICATION pub1, pub2, pub3;
```

## Управление репликацией

### Мониторинг состояния

```sql
-- На publisher: просмотр слотов репликации
SELECT slot_name, plugin, slot_type, active, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots;

-- На publisher: просмотр статистики отправки WAL
SELECT pid, application_name, client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn
FROM pg_stat_replication;

-- На subscriber: просмотр подписок
SELECT subname, subenabled, subconninfo, subslotname, subsynccommit
FROM pg_subscription;

-- На subscriber: статус подписки
SELECT srsubid, srrelid, srsublsn, srsubstate
FROM pg_subscription_rel;

-- Статистика worker'ов подписки
SELECT subname, pid, received_lsn, last_msg_send_time, last_msg_receipt_time
FROM pg_stat_subscription;
```

### Управление публикациями

```sql
-- Добавить таблицу в публикацию
ALTER PUBLICATION my_publication ADD TABLE new_table;

-- Удалить таблицу из публикации
ALTER PUBLICATION my_publication DROP TABLE old_table;

-- Изменить набор операций
ALTER PUBLICATION my_publication SET (publish = 'insert, update');

-- Удалить публикацию
DROP PUBLICATION my_publication;

-- Просмотр публикаций
SELECT * FROM pg_publication;

-- Просмотр таблиц в публикации
SELECT * FROM pg_publication_tables;
```

### Управление подписками

```sql
-- Приостановить подписку
ALTER SUBSCRIPTION my_subscription DISABLE;

-- Возобновить подписку
ALTER SUBSCRIPTION my_subscription ENABLE;

-- Обновить параметры подключения
ALTER SUBSCRIPTION my_subscription
    CONNECTION 'host=new_host port=5432 dbname=mydb user=replicator password=new_password';

-- Обновить подписку (получить новые таблицы)
ALTER SUBSCRIPTION my_subscription REFRESH PUBLICATION;

-- Обновить без копирования данных
ALTER SUBSCRIPTION my_subscription REFRESH PUBLICATION WITH (copy_data = false);

-- Удалить подписку
DROP SUBSCRIPTION my_subscription;
```

## Практические сценарии использования

### Сценарий 1: Миграция между версиями PostgreSQL

```sql
-- На старом сервере (PostgreSQL 12)
CREATE PUBLICATION migration_pub FOR ALL TABLES;

-- На новом сервере (PostgreSQL 16)
-- Создаем структуру через pg_dump --schema-only
CREATE SUBSCRIPTION migration_sub
    CONNECTION 'host=old_server dbname=mydb user=replicator password=pass'
    PUBLICATION migration_pub;

-- После синхронизации переключаем приложение на новый сервер
-- и удаляем подписку
DROP SUBSCRIPTION migration_sub;
```

### Сценарий 2: Репликация подмножества данных

```sql
-- Publisher: публикация только активных заказов
CREATE PUBLICATION active_orders FOR TABLE orders
    WHERE (status IN ('pending', 'processing', 'shipped'));

-- Subscriber: получает только активные заказы для обработки
CREATE SUBSCRIPTION order_processor
    CONNECTION 'host=main_db dbname=shop user=replicator password=pass'
    PUBLICATION active_orders;
```

### Сценарий 3: Агрегация данных из нескольких источников

```sql
-- На центральном сервере (Subscriber)
-- Подписки на разные региональные серверы
CREATE SUBSCRIPTION region_eu
    CONNECTION 'host=eu_server dbname=sales user=rep password=pass'
    PUBLICATION sales_data;

CREATE SUBSCRIPTION region_us
    CONNECTION 'host=us_server dbname=sales user=rep password=pass'
    PUBLICATION sales_data;

CREATE SUBSCRIPTION region_asia
    CONNECTION 'host=asia_server dbname=sales user=rep password=pass'
    PUBLICATION sales_data;
```

### Сценарий 4: Двунаправленная репликация (Bi-Directional)

```sql
-- Сервер A
CREATE PUBLICATION pub_a FOR TABLE shared_data;
CREATE SUBSCRIPTION sub_from_b
    CONNECTION 'host=server_b dbname=mydb user=rep password=pass'
    PUBLICATION pub_b
    WITH (origin = none);  -- Избежать циклов

-- Сервер B
CREATE PUBLICATION pub_b FOR TABLE shared_data;
CREATE SUBSCRIPTION sub_from_a
    CONNECTION 'host=server_a dbname=mydb user=rep password=pass'
    PUBLICATION pub_a
    WITH (origin = none);  -- Избежать циклов
```

## Конфликты и их разрешение

### Типы конфликтов

1. **Конфликт уникальности** — попытка вставить дублирующий ключ
2. **Конфликт внешнего ключа** — ссылка на несуществующую запись
3. **Конфликт NOT NULL** — попытка вставить NULL в NOT NULL колонку

### Обработка конфликтов

```sql
-- Просмотр ошибок репликации в логах
-- postgresql-*.log:
-- ERROR:  duplicate key value violates unique constraint "users_pkey"

-- Пропустить конфликтующую транзакцию
-- 1. Найти LSN проблемной транзакции
SELECT * FROM pg_stat_subscription;

-- 2. Пропустить до следующей LSN
ALTER SUBSCRIPTION my_subscription
    SKIP (lsn = '0/XXXXXXX');

-- 3. Либо разрешить конфликт вручную
DELETE FROM users WHERE id = 123;  -- Удалить конфликтующую запись
ALTER SUBSCRIPTION my_subscription ENABLE;
```

### Предотвращение конфликтов

```sql
-- Использовать uuid вместо serial для первичных ключей
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100)
);

-- Разделять диапазоны ID между серверами
-- Сервер A: id начинается с 1, шаг 2 (1, 3, 5, 7...)
-- Сервер B: id начинается с 2, шаг 2 (2, 4, 6, 8...)
```

## Ограничения логической репликации

### Что НЕ реплицируется

- **DDL-операции** (CREATE TABLE, ALTER TABLE, DROP TABLE)
- **Последовательности** (sequence values)
- **Large Objects** (lo_*)
- **Truncate** (до PostgreSQL 11, с 11 версии поддерживается)
- **Материализованные представления**
- **Партиционированные таблицы** (до PostgreSQL 13)

### Требования к таблицам

```sql
-- Таблицы ДОЛЖНЫ иметь REPLICA IDENTITY
-- По умолчанию используется PRIMARY KEY

-- Для таблиц без PK нужно явно указать
ALTER TABLE mytable REPLICA IDENTITY FULL;

-- Или использовать уникальный индекс
CREATE UNIQUE INDEX mytable_unique ON mytable(col1, col2);
ALTER TABLE mytable REPLICA IDENTITY USING INDEX mytable_unique;
```

## Best Practices

### 1. Настройка безопасности

```sql
-- Создать отдельного пользователя с минимальными правами
CREATE USER replicator WITH REPLICATION PASSWORD 'strong_password';
GRANT USAGE ON SCHEMA public TO replicator;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replicator;
REVOKE ALL ON SCHEMA pg_catalog FROM replicator;
```

### 2. Мониторинг задержки репликации

```sql
-- Проверка lag на subscriber
SELECT
    subname,
    pg_wal_lsn_diff(
        pg_current_wal_lsn(),
        confirmed_flush_lsn
    ) AS replication_lag_bytes
FROM pg_subscription s
JOIN pg_replication_slots r ON s.subslotname = r.slot_name;
```

### 3. Управление слотами репликации

```sql
-- Удалить неиспользуемые слоты (освободить WAL)
SELECT pg_drop_replication_slot('unused_slot');

-- Установить максимальный размер WAL для слота (PostgreSQL 13+)
ALTER SYSTEM SET max_slot_wal_keep_size = '10GB';
SELECT pg_reload_conf();
```

### 4. Начальная синхронизация больших таблиц

```sql
-- Для больших таблиц лучше делать начальную синхронизацию через pg_dump
-- 1. Создать подписку без copy_data
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=pub dbname=mydb user=rep password=pass'
    PUBLICATION my_pub
    WITH (copy_data = false, enabled = false);

-- 2. Сделать pg_dump с мастера и восстановить
pg_dump -h publisher -t table1 -t table2 mydb | psql -h subscriber mydb

-- 3. Включить подписку
ALTER SUBSCRIPTION my_sub ENABLE;
```

## Типичные ошибки и их решения

### Ошибка: "relation does not exist"

```bash
# Причина: таблица не существует на subscriber
# Решение: создать таблицу перед созданием подписки
psql -h subscriber -d mydb -c "CREATE TABLE missing_table (...)"
```

### Ошибка: "replication slot already exists"

```sql
-- Причина: слот не был удален после DROP SUBSCRIPTION
-- Решение: удалить слот вручную на publisher
SELECT pg_drop_replication_slot('my_subscription');
```

### Ошибка: "could not connect to the publisher"

```bash
# Проверить pg_hba.conf на publisher
# Проверить доступность порта
nc -zv publisher_host 5432

# Проверить firewall
iptables -L -n | grep 5432
```

## Заключение

Логическая репликация PostgreSQL предоставляет гибкий механизм для:
- Миграции между версиями PostgreSQL
- Выборочной репликации данных
- Построения распределенных систем
- Интеграции данных из разных источников

При правильной настройке и мониторинге логическая репликация обеспечивает надежную и эффективную синхронизацию данных между серверами PostgreSQL.
