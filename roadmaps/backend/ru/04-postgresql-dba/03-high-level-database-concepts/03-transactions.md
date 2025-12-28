# Транзакции в PostgreSQL

[prev: 02-mvcc](./02-mvcc.md) | [next: 04-write-ahead-log](./04-write-ahead-log.md)
---

## Введение

**Транзакция** — это логическая единица работы с базой данных, состоящая из одной или нескольких операций, которые выполняются как единое целое. Транзакции обеспечивают ACID-свойства и являются основой надёжности реляционных баз данных.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Жизненный цикл транзакции                    │
│                                                                 │
│   BEGIN ──→ Операции ──→ COMMIT (успех)                        │
│                │                                                │
│                └──→ ROLLBACK (откат при ошибке)                │
└─────────────────────────────────────────────────────────────────┘
```

## Основы работы с транзакциями

### Явные транзакции

```sql
-- Начало транзакции
BEGIN;
-- или
BEGIN TRANSACTION;
-- или
START TRANSACTION;

-- Операции внутри транзакции
INSERT INTO orders (customer_id, total) VALUES (1, 100.00);
UPDATE customers SET last_order = NOW() WHERE id = 1;

-- Фиксация изменений
COMMIT;
-- или
END;
```

### Неявные транзакции (Autocommit)

```sql
-- В PostgreSQL каждая отдельная команда — неявная транзакция
INSERT INTO logs (message) VALUES ('Action performed');
-- Автоматически выполняется COMMIT

-- Отключение autocommit (в psql)
\set AUTOCOMMIT off
```

### Откат транзакции

```sql
BEGIN;
    DELETE FROM important_data WHERE condition = true;
    -- Ой, удалили не то!
ROLLBACK;  -- Все изменения отменены

-- Проверяем — данные на месте
SELECT COUNT(*) FROM important_data;
```

## Savepoints (Точки сохранения)

Savepoints позволяют откатить часть транзакции, не отменяя всю транзакцию целиком.

```sql
BEGIN;
    INSERT INTO users (name) VALUES ('Alice');

    SAVEPOINT sp1;  -- Создаём точку сохранения

    INSERT INTO users (name) VALUES ('Bob');
    INSERT INTO users (name) VALUES ('Charlie');

    -- Что-то пошло не так с Bob и Charlie
    ROLLBACK TO SAVEPOINT sp1;  -- Откат до sp1

    -- Alice всё ещё будет добавлена
    INSERT INTO users (name) VALUES ('David');

COMMIT;
-- Результат: добавлены Alice и David
```

### Вложенные savepoints

```sql
BEGIN;
    SAVEPOINT level1;
        INSERT INTO t1 VALUES (1);

        SAVEPOINT level2;
            INSERT INTO t2 VALUES (2);

            SAVEPOINT level3;
                INSERT INTO t3 VALUES (3);
            ROLLBACK TO level3;  -- Отменяет только t3

        ROLLBACK TO level2;  -- Отменяет t2 и всё после level2

    -- level1 всё ещё активен
    INSERT INTO t4 VALUES (4);

COMMIT;
-- Результат: t1 с (1) и t4 с (4)
```

### Освобождение savepoint

```sql
BEGIN;
    SAVEPOINT sp1;
    INSERT INTO data VALUES (1);

    RELEASE SAVEPOINT sp1;  -- Освобождаем savepoint

    -- Теперь ROLLBACK TO sp1 невозможен
    -- ROLLBACK TO sp1; -- Ошибка!

COMMIT;
```

## Уровни изоляции транзакций

### Установка уровня изоляции

```sql
-- Для текущей транзакции
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Для сессии
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Глобально (postgresql.conf)
-- default_transaction_isolation = 'read committed'

-- Проверка текущего уровня
SHOW transaction_isolation;
```

### Read Committed (по умолчанию)

```sql
-- Сессия 1                         -- Сессия 2
BEGIN;                              BEGIN;
SELECT balance FROM accounts
WHERE id = 1;  -- 1000              UPDATE accounts
                                    SET balance = 500
                                    WHERE id = 1;
                                    COMMIT;

SELECT balance FROM accounts
WHERE id = 1;  -- 500 (видит изменения!)
COMMIT;
```

### Repeatable Read

```sql
-- Сессия 1                         -- Сессия 2
BEGIN ISOLATION LEVEL               BEGIN;
REPEATABLE READ;

SELECT balance FROM accounts
WHERE id = 1;  -- 1000              UPDATE accounts
                                    SET balance = 500
                                    WHERE id = 1;
                                    COMMIT;

SELECT balance FROM accounts
WHERE id = 1;  -- 1000 (НЕ видит изменения!)
COMMIT;
```

### Serializable

```sql
-- Сессия 1                         -- Сессия 2
BEGIN ISOLATION LEVEL               BEGIN ISOLATION LEVEL
SERIALIZABLE;                       SERIALIZABLE;

SELECT SUM(balance)                 SELECT SUM(balance)
FROM accounts;  -- 10000            FROM accounts;  -- 10000

UPDATE accounts                     UPDATE accounts
SET balance = balance + 100         SET balance = balance + 100
WHERE id = 1;                       WHERE id = 2;

COMMIT;  -- Успех                   COMMIT;  -- ОШИБКА!
                                    -- serialization_failure
```

## Блокировки в транзакциях

### Явные блокировки строк

```sql
BEGIN;
    -- Блокировка для обновления (эксклюзивная)
    SELECT * FROM accounts WHERE id = 1 FOR UPDATE;

    -- Другие транзакции ждут
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

### Типы блокировок строк

```sql
-- FOR UPDATE — эксклюзивная блокировка
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;

-- FOR NO KEY UPDATE — блокирует, но разрешает SELECT ... FOR KEY SHARE
SELECT * FROM accounts WHERE id = 1 FOR NO KEY UPDATE;

-- FOR SHARE — разделяемая блокировка (чтение)
SELECT * FROM accounts WHERE id = 1 FOR SHARE;

-- FOR KEY SHARE — блокировка ключа
SELECT * FROM accounts WHERE id = 1 FOR KEY SHARE;

-- NOWAIT — не ждать блокировку
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;
-- Вернёт ошибку, если строка заблокирована

-- SKIP LOCKED — пропустить заблокированные строки
SELECT * FROM tasks WHERE status = 'pending'
FOR UPDATE SKIP LOCKED LIMIT 1;
-- Отлично для очередей задач!
```

### Блокировки таблиц

```sql
-- Блокировка таблицы целиком
LOCK TABLE accounts IN ACCESS EXCLUSIVE MODE;

-- Режимы блокировки (от слабого к сильному):
-- ACCESS SHARE        (SELECT)
-- ROW SHARE          (SELECT FOR UPDATE/SHARE)
-- ROW EXCLUSIVE      (UPDATE, DELETE, INSERT)
-- SHARE UPDATE EXCLUSIVE (VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY)
-- SHARE              (CREATE INDEX)
-- SHARE ROW EXCLUSIVE
-- EXCLUSIVE          (блокирует чтение с FOR UPDATE)
-- ACCESS EXCLUSIVE   (ALTER TABLE, DROP, TRUNCATE, REINDEX, VACUUM FULL)
```

## Deadlock (Взаимная блокировка)

### Как возникает deadlock

```
┌─────────────────────────────────────────────────────────────────┐
│                       Deadlock                                  │
│                                                                 │
│   Транзакция 1                  Транзакция 2                   │
│       │                              │                          │
│   Блокирует A ◄─────────────────── Ждёт A                      │
│       │                              │                          │
│   Ждёт B ───────────────────────► Блокирует B                  │
│                                                                 │
│              Обе ждут друг друга — DEADLOCK!                   │
└─────────────────────────────────────────────────────────────────┘
```

### Пример deadlock

```sql
-- Сессия 1                         -- Сессия 2
BEGIN;                              BEGIN;
UPDATE accounts SET balance = 100
WHERE id = 1;  -- Блокирует id=1    UPDATE accounts SET balance = 200
                                    WHERE id = 2;  -- Блокирует id=2

UPDATE accounts SET balance = 100
WHERE id = 2;  -- Ждёт id=2         UPDATE accounts SET balance = 200
                                    WHERE id = 1;  -- Ждёт id=1

-- PostgreSQL обнаружит deadlock и отменит одну из транзакций
-- ERROR: deadlock detected
```

### Избежание deadlock

```sql
-- Правило: всегда блокируйте ресурсы в одинаковом порядке
BEGIN;
    -- Сортируем ID и блокируем по порядку
    SELECT * FROM accounts
    WHERE id IN (1, 2)
    ORDER BY id
    FOR UPDATE;

    UPDATE accounts SET balance = 100 WHERE id = 1;
    UPDATE accounts SET balance = 200 WHERE id = 2;
COMMIT;

-- Или используйте advisory locks
BEGIN;
    SELECT pg_advisory_xact_lock(1);  -- Получаем "глобальную" блокировку
    UPDATE accounts SET balance = 100 WHERE id = 1;
    UPDATE accounts SET balance = 200 WHERE id = 2;
COMMIT;  -- Автоматически освобождается
```

## Advisory Locks (Рекомендательные блокировки)

### Сессионные advisory locks

```sql
-- Получение блокировки (ждёт если занята)
SELECT pg_advisory_lock(12345);

-- Попытка получить без ожидания
SELECT pg_try_advisory_lock(12345);  -- true/false

-- Освобождение
SELECT pg_advisory_unlock(12345);

-- С двумя ключами
SELECT pg_advisory_lock(1, 100);  -- Категория + ID
```

### Транзакционные advisory locks

```sql
BEGIN;
    -- Автоматически освобождается при COMMIT/ROLLBACK
    SELECT pg_advisory_xact_lock(12345);

    -- Операции...
COMMIT;  -- Блокировка освобождена
```

### Практический пример: защита от дублей

```sql
-- Предотвращение одновременной обработки одного заказа
CREATE OR REPLACE FUNCTION process_order(order_id INTEGER)
RETURNS BOOLEAN AS $$
BEGIN
    -- Пытаемся получить блокировку без ожидания
    IF NOT pg_try_advisory_xact_lock(order_id) THEN
        RAISE NOTICE 'Заказ % уже обрабатывается', order_id;
        RETURN FALSE;
    END IF;

    -- Обработка заказа...
    UPDATE orders SET status = 'processing' WHERE id = order_id;
    PERFORM pg_sleep(5);  -- Имитация работы
    UPDATE orders SET status = 'completed' WHERE id = order_id;

    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;
```

## Мониторинг транзакций

### Активные транзакции

```sql
-- Все активные транзакции
SELECT pid,
       usename,
       application_name,
       client_addr,
       xact_start,
       state,
       query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;

-- Долгие транзакции (> 5 минут)
SELECT pid,
       age(clock_timestamp(), xact_start) as duration,
       query
FROM pg_stat_activity
WHERE xact_start < clock_timestamp() - interval '5 minutes'
  AND state != 'idle';
```

### Блокировки

```sql
-- Кто кого блокирует
SELECT
    blocked.pid AS blocked_pid,
    blocked.usename AS blocked_user,
    blocking.pid AS blocking_pid,
    blocking.usename AS blocking_user,
    blocked.query AS blocked_query,
    blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks blocked_locks ON blocked.pid = blocked_locks.pid
JOIN pg_locks blocking_locks ON blocked_locks.locktype = blocking_locks.locktype
    AND blocked_locks.relation = blocking_locks.relation
    AND blocked_locks.pid != blocking_locks.pid
JOIN pg_stat_activity blocking ON blocking_locks.pid = blocking.pid
WHERE NOT blocked_locks.granted;
```

### Принудительное завершение

```sql
-- Отмена текущего запроса (мягко)
SELECT pg_cancel_backend(12345);

-- Завершение сессии (жёстко)
SELECT pg_terminate_backend(12345);
```

## Read-Only транзакции

```sql
-- Транзакция только для чтения
BEGIN READ ONLY;
    SELECT * FROM accounts;
    -- UPDATE accounts SET ... ; -- Ошибка!
COMMIT;

-- Deferrable read-only (для Serializable)
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
    -- Ждёт безопасный момент для создания snapshot
    -- Гарантирует отсутствие serialization failures
    SELECT * FROM big_report_view;
COMMIT;
```

## Two-Phase Commit (2PC)

Для распределённых транзакций между несколькими базами:

```sql
-- Подготовка транзакции
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
PREPARE TRANSACTION 'transfer_001';

-- Транзакция теперь "подвешена"
-- Можно проверить статус
SELECT * FROM pg_prepared_xacts;

-- На другой БД тоже подготавливаем...

-- Если всё OK — фиксируем
COMMIT PREPARED 'transfer_001';

-- Если проблемы — откатываем
ROLLBACK PREPARED 'transfer_001';
```

## Best Practices

### 1. Держите транзакции короткими

```sql
-- Плохо: долгая транзакция
BEGIN;
    SELECT * FROM large_table;  -- Обработка 1 млн строк в приложении
    -- ... 10 минут работы ...
    UPDATE small_table SET processed = true;
COMMIT;

-- Хорошо: batch processing
DO $$
DECLARE
    batch_size INTEGER := 1000;
    processed INTEGER := 0;
BEGIN
    LOOP
        WITH batch AS (
            SELECT id FROM large_table
            WHERE processed = false
            LIMIT batch_size
            FOR UPDATE SKIP LOCKED
        )
        UPDATE large_table SET processed = true
        WHERE id IN (SELECT id FROM batch);

        GET DIAGNOSTICS processed = ROW_COUNT;
        EXIT WHEN processed = 0;

        COMMIT;  -- Короткие транзакции
    END LOOP;
END $$;
```

### 2. Обрабатывайте ошибки правильно

```sql
-- Использование блока исключений
DO $$
BEGIN
    BEGIN
        INSERT INTO users (email) VALUES ('test@example.com');
    EXCEPTION
        WHEN unique_violation THEN
            RAISE NOTICE 'Email уже существует';
        WHEN OTHERS THEN
            RAISE NOTICE 'Ошибка: %', SQLERRM;
            RAISE;  -- Пробрасываем дальше
    END;
END $$;
```

### 3. Используйте правильный уровень изоляции

```sql
-- Для большинства OLTP: Read Committed (по умолчанию)
-- Для отчётов с согласованностью: Repeatable Read
-- Для критичных финансовых операций: Serializable

-- Пример: генерация отчёта
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ READ ONLY;
    SELECT * FROM monthly_sales_report();
    SELECT * FROM monthly_expenses_report();
    -- Оба отчёта видят одинаковый snapshot данных
COMMIT;
```

### 4. Реализуйте retry-логику для Serializable

```python
# Python псевдокод
def execute_serializable_transaction(conn, action):
    max_retries = 3
    for attempt in range(max_retries):
        try:
            with conn.transaction():
                conn.execute("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE")
                action(conn)
                return  # Успех
        except SerializationFailure:
            if attempt == max_retries - 1:
                raise
            time.sleep(0.1 * (2 ** attempt))  # Exponential backoff
```

## Заключение

Транзакции — это мощный инструмент для обеспечения целостности данных. Ключевые моменты:

1. Используйте явные транзакции для связанных операций
2. Savepoints помогают управлять частичным откатом
3. Выбирайте уровень изоляции под задачу
4. Избегайте deadlock через упорядоченный доступ к ресурсам
5. Держите транзакции короткими
6. Мониторьте долгие транзакции и блокировки

---
[prev: 02-mvcc](./02-mvcc.md) | [next: 04-write-ahead-log](./04-write-ahead-log.md)