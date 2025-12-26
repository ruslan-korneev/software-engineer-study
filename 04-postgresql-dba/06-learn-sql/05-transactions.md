# Транзакции в PostgreSQL

## Что такое транзакция?

**Транзакция** — это группа операций с базой данных, которые выполняются как единое целое. Либо все операции выполняются успешно, либо ни одна из них не применяется.

### Свойства ACID

| Свойство | Описание |
|----------|----------|
| **A**tomicity (Атомарность) | Транзакция выполняется полностью или не выполняется вовсе |
| **C**onsistency (Согласованность) | База данных переходит из одного согласованного состояния в другое |
| **I**solation (Изолированность) | Параллельные транзакции не влияют друг на друга |
| **D**urability (Долговечность) | После фиксации данные сохраняются даже при сбое |

---

## Основные команды

### BEGIN / COMMIT / ROLLBACK

```sql
-- Начало транзакции
BEGIN;
-- или
BEGIN TRANSACTION;
-- или
START TRANSACTION;

-- Выполнение операций
INSERT INTO accounts (user_id, balance) VALUES (1, 1000);
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE user_id = 2;

-- Фиксация изменений
COMMIT;
-- или
END;

-- Отмена изменений
ROLLBACK;
```

### Пример: Перевод денег

```sql
BEGIN;

-- Проверяем баланс отправителя
SELECT balance FROM accounts WHERE user_id = 1 FOR UPDATE;

-- Списываем со счета отправителя
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;

-- Зачисляем на счет получателя
UPDATE accounts SET balance = balance + 100 WHERE user_id = 2;

-- Если все ок - фиксируем
COMMIT;

-- Если произошла ошибка - откатываем
-- ROLLBACK;
```

---

## SAVEPOINT — Точки сохранения

Позволяют откатить часть транзакции, не отменяя всю транзакцию:

```sql
BEGIN;

INSERT INTO orders (user_id, total) VALUES (1, 100);
SAVEPOINT order_created;

-- Пытаемся добавить товары
INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 1, 2);
-- Ошибка!
ROLLBACK TO SAVEPOINT order_created;

-- Пробуем другой вариант
INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 2, 1);

-- Удаляем savepoint (опционально)
RELEASE SAVEPOINT order_created;

COMMIT;
```

### Вложенные SAVEPOINT

```sql
BEGIN;
    INSERT INTO logs (message) VALUES ('Step 1');
    SAVEPOINT sp1;

        INSERT INTO logs (message) VALUES ('Step 2');
        SAVEPOINT sp2;

            INSERT INTO logs (message) VALUES ('Step 3');
            -- Откат к sp2 отменит Step 3
            ROLLBACK TO sp2;

        INSERT INTO logs (message) VALUES ('Step 3 retry');
        -- Откат к sp1 отменит Step 2 и Step 3 retry
        -- ROLLBACK TO sp1;

COMMIT;
```

---

## Уровни изоляции транзакций

PostgreSQL поддерживает 4 уровня изоляции:

| Уровень | Dirty Read | Non-Repeatable Read | Phantom Read |
|---------|------------|---------------------|--------------|
| Read Uncommitted* | Нет | Да | Да |
| Read Committed (default) | Нет | Да | Да |
| Repeatable Read | Нет | Нет | Нет** |
| Serializable | Нет | Нет | Нет |

*В PostgreSQL Read Uncommitted работает как Read Committed
**В PostgreSQL Repeatable Read также предотвращает phantom reads

### Установка уровня изоляции

```sql
-- Для текущей транзакции
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- или
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Для сессии
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Просмотр текущего уровня
SHOW transaction_isolation;
```

---

## Read Committed (по умолчанию)

Каждый запрос видит данные, зафиксированные до начала этого запроса.

```sql
-- Транзакция 1                    -- Транзакция 2
BEGIN;                              BEGIN;

SELECT balance FROM accounts
WHERE user_id = 1;
-- Результат: 1000
                                    UPDATE accounts
                                    SET balance = 500
                                    WHERE user_id = 1;
                                    COMMIT;

SELECT balance FROM accounts
WHERE user_id = 1;
-- Результат: 500 (видим изменения!)

COMMIT;
```

### Non-Repeatable Read

Один и тот же запрос может вернуть разные результаты в рамках одной транзакции.

---

## Repeatable Read

Транзакция видит данные на момент начала транзакции.

```sql
-- Транзакция 1                    -- Транзакция 2
BEGIN ISOLATION LEVEL               BEGIN;
REPEATABLE READ;

SELECT balance FROM accounts
WHERE user_id = 1;
-- Результат: 1000
                                    UPDATE accounts
                                    SET balance = 500
                                    WHERE user_id = 1;
                                    COMMIT;

SELECT balance FROM accounts
WHERE user_id = 1;
-- Результат: 1000 (старые данные!)

UPDATE accounts
SET balance = balance - 100
WHERE user_id = 1;
-- ERROR: could not serialize access
-- due to concurrent update

ROLLBACK;
```

### Обработка конфликтов

```sql
-- Необходимо ловить ошибки сериализации и повторять транзакцию
DO $$
DECLARE
    retry_count INTEGER := 0;
    max_retries INTEGER := 3;
BEGIN
    WHILE retry_count < max_retries LOOP
        BEGIN
            -- Начинаем транзакцию
            UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
            UPDATE accounts SET balance = balance + 100 WHERE user_id = 2;
            -- Успех - выходим
            EXIT;
        EXCEPTION
            WHEN serialization_failure THEN
                retry_count := retry_count + 1;
                IF retry_count >= max_retries THEN
                    RAISE;
                END IF;
                -- Ждем немного и повторяем
                PERFORM pg_sleep(0.1 * retry_count);
        END;
    END LOOP;
END $$;
```

---

## Serializable

Самый строгий уровень изоляции. Гарантирует, что результат параллельных транзакций эквивалентен их последовательному выполнению.

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;

-- Проверяем, есть ли уже запись
SELECT COUNT(*) FROM registrations
WHERE event_id = 1 AND user_id = 100;

-- Если нет - добавляем
INSERT INTO registrations (event_id, user_id)
VALUES (1, 100);

COMMIT;
```

### Особенности Serializable

- Более высокая вероятность ошибок сериализации
- Необходима логика повторных попыток
- Лучшая целостность данных

---

## Блокировки

### Явные блокировки строк

```sql
-- Блокировка для обновления
SELECT * FROM accounts WHERE user_id = 1 FOR UPDATE;

-- Блокировка без ожидания
SELECT * FROM accounts WHERE user_id = 1 FOR UPDATE NOWAIT;

-- Пропустить заблокированные строки
SELECT * FROM accounts WHERE user_id = 1 FOR UPDATE SKIP LOCKED;

-- Блокировка только для чтения (предотвращает изменения)
SELECT * FROM accounts WHERE user_id = 1 FOR SHARE;
```

### Типы блокировок FOR

| Режим | Описание |
|-------|----------|
| `FOR UPDATE` | Блокирует для UPDATE/DELETE |
| `FOR NO KEY UPDATE` | Блокирует для UPDATE (кроме ключей) |
| `FOR SHARE` | Разрешает другим читать, но не изменять |
| `FOR KEY SHARE` | Самая слабая, только защита ключей |

### Блокировки таблиц

```sql
-- Явная блокировка таблицы
LOCK TABLE accounts IN EXCLUSIVE MODE;

-- Режимы блокировки таблицы
-- ACCESS SHARE - SELECT
-- ROW SHARE - SELECT FOR UPDATE/SHARE
-- ROW EXCLUSIVE - UPDATE, DELETE, INSERT
-- SHARE UPDATE EXCLUSIVE - VACUUM, CREATE INDEX CONCURRENTLY
-- SHARE - CREATE INDEX
-- SHARE ROW EXCLUSIVE - редко используется
-- EXCLUSIVE - блокирует все кроме ACCESS SHARE
-- ACCESS EXCLUSIVE - полная блокировка (ALTER TABLE, DROP)
```

---

## Deadlock (Взаимная блокировка)

### Пример deadlock

```sql
-- Транзакция 1                    -- Транзакция 2
BEGIN;                              BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE user_id = 1;
                                    UPDATE accounts
                                    SET balance = balance - 100
                                    WHERE user_id = 2;

UPDATE accounts
SET balance = balance + 100
WHERE user_id = 2;
-- Ждет транзакцию 2...
                                    UPDATE accounts
                                    SET balance = balance + 100
                                    WHERE user_id = 1;
                                    -- Ждет транзакцию 1...
                                    -- DEADLOCK!
```

### Предотвращение deadlock

```sql
-- 1. Всегда блокировать в одном порядке
BEGIN;
-- Сначала блокируем обе записи в определенном порядке
SELECT * FROM accounts
WHERE user_id IN (1, 2)
ORDER BY user_id
FOR UPDATE;

-- Затем выполняем операции
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE user_id = 2;
COMMIT;

-- 2. Использовать таймауты
SET lock_timeout = '5s';

-- 3. Использовать NOWAIT
SELECT * FROM accounts WHERE user_id = 1 FOR UPDATE NOWAIT;
```

---

## Advisory Locks (Рекомендательные блокировки)

Блокировки на уровне приложения, не связанные с таблицами:

```sql
-- Захват блокировки (ждет освобождения)
SELECT pg_advisory_lock(12345);

-- Попытка захвата без ожидания
SELECT pg_try_advisory_lock(12345);  -- true/false

-- Освобождение блокировки
SELECT pg_advisory_unlock(12345);

-- Блокировка на уровне сессии
SELECT pg_advisory_lock(hashtext('my_unique_process'));

-- Блокировка на уровне транзакции
SELECT pg_advisory_xact_lock(12345);
-- Автоматически освобождается при COMMIT/ROLLBACK

-- Проверка текущих блокировок
SELECT * FROM pg_locks WHERE locktype = 'advisory';
```

### Практическое применение

```sql
-- Предотвращение параллельного запуска задачи
DO $$
DECLARE
    lock_acquired BOOLEAN;
BEGIN
    -- Пытаемся захватить блокировку
    SELECT pg_try_advisory_lock(hashtext('daily_report')) INTO lock_acquired;

    IF NOT lock_acquired THEN
        RAISE NOTICE 'Another instance is running';
        RETURN;
    END IF;

    -- Выполняем задачу
    PERFORM generate_daily_report();

    -- Освобождаем блокировку
    PERFORM pg_advisory_unlock(hashtext('daily_report'));
END $$;
```

---

## Автокоммит и транзакции

### Режим автокоммита

По умолчанию каждая команда выполняется в отдельной транзакции:

```sql
-- Каждая команда - отдельная транзакция
INSERT INTO logs (message) VALUES ('one');
INSERT INTO logs (message) VALUES ('two');
-- Если вторая упадет, первая уже зафиксирована

-- Явная транзакция
BEGIN;
INSERT INTO logs (message) VALUES ('one');
INSERT INTO logs (message) VALUES ('two');
COMMIT;
-- Обе вставки атомарны
```

### Отложенная фиксация

```sql
-- Синхронная фиксация (по умолчанию)
SET synchronous_commit = on;

-- Асинхронная фиксация (быстрее, менее надежно)
SET synchronous_commit = off;
BEGIN;
INSERT INTO logs (message) VALUES ('Fast log');
COMMIT;  -- Возвращается сразу, не ждет записи на диск
```

---

## Лучшие практики

### 1. Короткие транзакции

```sql
-- Плохо: долгая транзакция блокирует ресурсы
BEGIN;
SELECT * FROM large_table;  -- долгая операция
-- ... много времени проходит ...
UPDATE accounts SET balance = 0;
COMMIT;

-- Хорошо: разбить на части
-- Сначала читаем данные
SELECT * FROM large_table;

-- Потом отдельная короткая транзакция
BEGIN;
UPDATE accounts SET balance = 0;
COMMIT;
```

### 2. Обработка ошибок

```sql
-- Python пример
try:
    cursor.execute("BEGIN")
    cursor.execute("UPDATE accounts SET balance = balance - 100 WHERE user_id = 1")
    cursor.execute("UPDATE accounts SET balance = balance + 100 WHERE user_id = 2")
    cursor.execute("COMMIT")
except Exception as e:
    cursor.execute("ROLLBACK")
    raise
```

### 3. Использование правильного уровня изоляции

```sql
-- Read Committed: для большинства случаев
-- Repeatable Read: когда нужны согласованные отчеты
-- Serializable: для критических финансовых операций
```

### 4. Избегайте long-running транзакций

```sql
-- Проверка долгих транзакций
SELECT
    pid,
    now() - xact_start AS duration,
    query
FROM pg_stat_activity
WHERE state = 'active'
  AND xact_start < now() - interval '5 minutes';
```

---

## Типичные ошибки

### 1. Забыть COMMIT

```sql
BEGIN;
UPDATE accounts SET balance = 100;
-- Забыли COMMIT
-- Блокировка держится, изменения не видны другим
```

### 2. Транзакции в цикле

```sql
-- Плохо: много маленьких транзакций
FOR i IN 1..10000 LOOP
    INSERT INTO logs (value) VALUES (i);  -- отдельная транзакция
END LOOP;

-- Хорошо: одна транзакция
BEGIN;
FOR i IN 1..10000 LOOP
    INSERT INTO logs (value) VALUES (i);
END LOOP;
COMMIT;
```

### 3. Игнорирование ошибок сериализации

```sql
-- При использовании Repeatable Read / Serializable
-- необходимо обрабатывать serialization_failure
-- и повторять транзакцию!
```

---

## Мониторинг транзакций

```sql
-- Активные транзакции
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    xact_start,
    state,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL;

-- Блокировки
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query
FROM pg_stat_activity AS blocked
JOIN pg_locks AS blocked_locks ON blocked.pid = blocked_locks.pid
JOIN pg_locks AS blocking_locks ON blocked_locks.locktype = blocking_locks.locktype
    AND blocked_locks.relation = blocking_locks.relation
    AND blocked_locks.pid != blocking_locks.pid
JOIN pg_stat_activity AS blocking ON blocking_locks.pid = blocking.pid
WHERE NOT blocked_locks.granted;
```
