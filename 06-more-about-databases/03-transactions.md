# Транзакции

## Что такое транзакция

**Транзакция** — это логическая единица работы с базой данных, которая состоит из одной или нескольких операций. Транзакция гарантирует, что все операции внутри неё либо выполняются полностью, либо не выполняются вообще.

Транзакции обеспечивают свойства **ACID**:

| Свойство | Описание |
|----------|----------|
| **Atomicity** (Атомарность) | Транзакция выполняется полностью или не выполняется вовсе. Если хотя бы одна операция не удалась — все изменения откатываются |
| **Consistency** (Согласованность) | Транзакция переводит БД из одного согласованного состояния в другое. Все ограничения (constraints) должны соблюдаться |
| **Isolation** (Изолированность) | Параллельные транзакции не влияют друг на друга. Каждая транзакция видит данные так, будто выполняется одна |
| **Durability** (Долговечность) | После успешного завершения транзакции изменения сохраняются даже при сбое системы |

### Пример необходимости транзакции

Классический пример — перевод денег между счетами:

```sql
-- Без транзакции: если произойдёт сбой между операциями,
-- деньги могут "исчезнуть" или "удвоиться"
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
-- Здесь мог произойти сбой!
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;
```

---

## Жизненный цикл транзакции

### Основные команды

```sql
-- Начало транзакции
BEGIN;
-- или
START TRANSACTION;

-- Подтверждение транзакции (сохранение изменений)
COMMIT;

-- Откат транзакции (отмена изменений)
ROLLBACK;
```

### Диаграмма жизненного цикла

```
    ┌─────────┐
    │  BEGIN  │
    └────┬────┘
         │
         ▼
    ┌─────────┐
    │ Активна │◄────────┐
    └────┬────┘         │
         │              │
    ┌────┴────┐         │
    │ Операции│─────────┘
    └────┬────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐  ┌──────────┐
│COMMIT │  │ ROLLBACK │
└───┬───┘  └────┬─────┘
    │           │
    ▼           ▼
┌───────┐  ┌──────────┐
│Успешно│  │ Откачено │
└───────┘  └──────────┘
```

### Пример с безопасным переводом

```sql
BEGIN;

-- Проверяем достаточность средств
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;

-- Если баланс достаточный, выполняем перевод
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;

-- Записываем в историю транзакций
INSERT INTO transaction_log (from_account, to_account, amount, created_at)
VALUES (1, 2, 1000, NOW());

COMMIT;
```

### SAVEPOINT — точки сохранения

Savepoint позволяет откатить часть транзакции, не отменяя её полностью:

```sql
BEGIN;

INSERT INTO orders (customer_id, total) VALUES (1, 5000);
SAVEPOINT order_created;

INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 100, 2);
-- Ошибка! Товара нет на складе
ROLLBACK TO SAVEPOINT order_created;

-- Пробуем другой товар
INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 101, 2);

COMMIT;
```

---

## Уровни изоляции транзакций

Уровень изоляции определяет, как транзакции "видят" изменения друг друга.

### Таблица уровней изоляции

| Уровень изоляции | Dirty Read | Non-repeatable Read | Phantom Read |
|-----------------|------------|---------------------|--------------|
| READ UNCOMMITTED | Возможно | Возможно | Возможно |
| READ COMMITTED | Невозможно | Возможно | Возможно |
| REPEATABLE READ | Невозможно | Невозможно | Возможно* |
| SERIALIZABLE | Невозможно | Невозможно | Невозможно |

*В PostgreSQL REPEATABLE READ также предотвращает phantom read благодаря MVCC.

### Установка уровня изоляции

```sql
-- Для текущей транзакции
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- операции...
COMMIT;

-- Для всей сессии
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Проверить текущий уровень
SHOW transaction_isolation;
```

### READ UNCOMMITTED

Самый низкий уровень. Транзакция может читать незакоммиченные изменения других транзакций.

```sql
-- Транзакция 1                    -- Транзакция 2
BEGIN;                              BEGIN;
UPDATE products
  SET price = 100
  WHERE id = 1;
                                    -- Читает незакоммиченное значение 100
                                    SELECT price FROM products WHERE id = 1;
ROLLBACK;
                                    -- Но оно было откачено!
                                    -- Транзакция 2 использовала "грязные" данные
```

**Примечание:** PostgreSQL не поддерживает READ UNCOMMITTED — он работает как READ COMMITTED.

### READ COMMITTED

Уровень по умолчанию в PostgreSQL. Читаются только закоммиченные данные.

```sql
-- Транзакция 1                    -- Транзакция 2
BEGIN;                              BEGIN;
                                    SELECT price FROM products WHERE id = 1;
                                    -- Результат: 50
UPDATE products
  SET price = 100
  WHERE id = 1;
COMMIT;
                                    SELECT price FROM products WHERE id = 1;
                                    -- Результат: 100 (изменился!)
                                    -- Non-repeatable read!
```

### REPEATABLE READ

Гарантирует, что повторное чтение вернёт те же данные.

```sql
-- Транзакция 1                    -- Транзакция 2
BEGIN;                              BEGIN;
SET TRANSACTION ISOLATION
  LEVEL REPEATABLE READ;
                                    SET TRANSACTION ISOLATION
                                      LEVEL REPEATABLE READ;
SELECT price FROM products
  WHERE id = 1;
-- Результат: 50
                                    UPDATE products SET price = 100 WHERE id = 1;
                                    COMMIT;

SELECT price FROM products
  WHERE id = 1;
-- Результат: 50 (не изменился!)
COMMIT;
```

### SERIALIZABLE

Самый строгий уровень. Транзакции выполняются так, будто они последовательны.

```sql
-- Транзакция 1                    -- Транзакция 2
BEGIN;                              BEGIN;
SET TRANSACTION ISOLATION
  LEVEL SERIALIZABLE;
                                    SET TRANSACTION ISOLATION
                                      LEVEL SERIALIZABLE;
SELECT SUM(balance)
  FROM accounts;
-- Результат: 10000
                                    SELECT SUM(balance) FROM accounts;
                                    -- Результат: 10000
UPDATE accounts
  SET balance = balance + 100
  WHERE id = 1;
                                    UPDATE accounts
                                      SET balance = balance + 200
                                      WHERE id = 2;
COMMIT;
                                    COMMIT;
                                    -- ERROR: could not serialize access
                                    -- due to read/write dependencies
```

---

## Проблемы параллельного доступа

### Dirty Read (Грязное чтение)

Чтение данных, которые ещё не были закоммичены другой транзакцией.

```sql
-- Транзакция A                    -- Транзакция B
BEGIN;
UPDATE products
  SET stock = 0
  WHERE id = 1;
                                    BEGIN;
                                    -- Dirty read: видит stock = 0
                                    SELECT stock FROM products WHERE id = 1;
                                    -- Решает не заказывать товар
                                    COMMIT;
ROLLBACK;
-- stock снова = 100, но B уже приняла решение
-- на основе "грязных" данных
```

### Non-repeatable Read (Неповторяющееся чтение)

Повторное чтение тех же данных возвращает разные значения.

```sql
-- Транзакция A                    -- Транзакция B
BEGIN;
SELECT balance FROM accounts
  WHERE id = 1;
-- Результат: 1000
                                    BEGIN;
                                    UPDATE accounts
                                      SET balance = 500
                                      WHERE id = 1;
                                    COMMIT;
SELECT balance FROM accounts
  WHERE id = 1;
-- Результат: 500 (изменилось!)
-- Это может нарушить логику транзакции A
```

### Phantom Read (Фантомное чтение)

Повторный запрос возвращает новые строки, добавленные другой транзакцией.

```sql
-- Транзакция A                    -- Транзакция B
BEGIN;
SELECT COUNT(*) FROM orders
  WHERE status = 'pending';
-- Результат: 5
                                    BEGIN;
                                    INSERT INTO orders (status)
                                      VALUES ('pending');
                                    COMMIT;
SELECT COUNT(*) FROM orders
  WHERE status = 'pending';
-- Результат: 6 (появилась "фантомная" строка)
```

### Lost Update (Потерянное обновление)

Два параллельных обновления, одно из которых теряется.

```sql
-- Транзакция A                    -- Транзакция B
BEGIN;                              BEGIN;
SELECT stock FROM products
  WHERE id = 1;
-- stock = 10
                                    SELECT stock FROM products WHERE id = 1;
                                    -- stock = 10
UPDATE products
  SET stock = 10 - 3
  WHERE id = 1;
-- stock = 7
COMMIT;
                                    UPDATE products
                                      SET stock = 10 - 2
                                      WHERE id = 1;
                                    -- stock = 8 (потеряно обновление A!)
                                    COMMIT;
```

**Решение — использовать FOR UPDATE:**

```sql
BEGIN;
SELECT stock FROM products WHERE id = 1 FOR UPDATE;
-- Блокирует строку для других транзакций
UPDATE products SET stock = stock - 3 WHERE id = 1;
COMMIT;
```

---

## Блокировки и Deadlocks

### Типы блокировок в PostgreSQL

#### Блокировки строк (Row-level locks)

```sql
-- FOR UPDATE — эксклюзивная блокировка для обновления
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;

-- FOR NO KEY UPDATE — как FOR UPDATE, но разрешает изменение не-ключевых полей
SELECT * FROM accounts WHERE id = 1 FOR NO KEY UPDATE;

-- FOR SHARE — разделяемая блокировка (другие могут читать, но не менять)
SELECT * FROM accounts WHERE id = 1 FOR SHARE;

-- FOR KEY SHARE — самая слабая, блокирует только удаление и изменение ключа
SELECT * FROM accounts WHERE id = 1 FOR KEY SHARE;

-- NOWAIT — не ждать освобождения блокировки
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;
-- Если заблокировано, сразу возвращает ошибку

-- SKIP LOCKED — пропустить заблокированные строки
SELECT * FROM tasks WHERE status = 'pending'
  FOR UPDATE SKIP LOCKED LIMIT 1;
-- Полезно для очередей задач
```

#### Блокировки таблиц (Table-level locks)

```sql
-- Эксклюзивная блокировка всей таблицы
LOCK TABLE accounts IN ACCESS EXCLUSIVE MODE;

-- Другие режимы:
-- ACCESS SHARE — для SELECT
-- ROW SHARE — для SELECT FOR UPDATE/SHARE
-- ROW EXCLUSIVE — для UPDATE, DELETE, INSERT
-- SHARE UPDATE EXCLUSIVE — для VACUUM, CREATE INDEX CONCURRENTLY
-- SHARE — для CREATE INDEX
-- SHARE ROW EXCLUSIVE — редко используется
-- EXCLUSIVE — блокирует всё кроме ACCESS SHARE
-- ACCESS EXCLUSIVE — полная блокировка
```

### Deadlock (Взаимная блокировка)

Deadlock возникает, когда две транзакции ждут друг друга.

```sql
-- Транзакция A                    -- Транзакция B
BEGIN;                              BEGIN;
UPDATE accounts
  SET balance = balance - 100
  WHERE id = 1;
-- Блокирует строку id=1
                                    UPDATE accounts
                                      SET balance = balance - 100
                                      WHERE id = 2;
                                    -- Блокирует строку id=2

UPDATE accounts
  SET balance = balance + 100
  WHERE id = 2;
-- Ждёт освобождения id=2
                                    UPDATE accounts
                                      SET balance = balance + 100
                                      WHERE id = 1;
                                    -- Ждёт освобождения id=1

-- DEADLOCK! PostgreSQL убьёт одну из транзакций
-- ERROR: deadlock detected
```

### Предотвращение Deadlock

```sql
-- 1. Упорядочивание доступа к ресурсам
-- Всегда обращаемся к строкам в одном порядке (по id)
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- 2. Блокировка всех нужных строк заранее
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
-- Теперь обе строки заблокированы атомарно
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- 3. Использование таймаутов
SET lock_timeout = '5s';
-- Если не удалось получить блокировку за 5 секунд — ошибка
```

### Мониторинг блокировок

```sql
-- Текущие блокировки
SELECT
    pid,
    locktype,
    relation::regclass,
    mode,
    granted
FROM pg_locks
WHERE relation IS NOT NULL;

-- Ожидающие транзакции
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks blocked_locks ON blocked.pid = blocked_locks.pid
JOIN pg_locks blocking_locks ON blocked_locks.locktype = blocking_locks.locktype
    AND blocked_locks.relation = blocking_locks.relation
    AND blocked_locks.pid != blocking_locks.pid
JOIN pg_stat_activity blocking ON blocking_locks.pid = blocking.pid
WHERE NOT blocked_locks.granted;
```

---

## Примеры кода SQL

### Перевод денег с проверками

```sql
CREATE OR REPLACE FUNCTION transfer_money(
    from_account_id INT,
    to_account_id INT,
    amount DECIMAL
) RETURNS BOOLEAN AS $$
DECLARE
    from_balance DECIMAL;
BEGIN
    -- Блокируем обе строки в определённом порядке
    IF from_account_id < to_account_id THEN
        PERFORM * FROM accounts WHERE id = from_account_id FOR UPDATE;
        PERFORM * FROM accounts WHERE id = to_account_id FOR UPDATE;
    ELSE
        PERFORM * FROM accounts WHERE id = to_account_id FOR UPDATE;
        PERFORM * FROM accounts WHERE id = from_account_id FOR UPDATE;
    END IF;

    -- Проверяем баланс
    SELECT balance INTO from_balance
    FROM accounts
    WHERE id = from_account_id;

    IF from_balance < amount THEN
        RAISE EXCEPTION 'Недостаточно средств';
    END IF;

    -- Выполняем перевод
    UPDATE accounts SET balance = balance - amount WHERE id = from_account_id;
    UPDATE accounts SET balance = balance + amount WHERE id = to_account_id;

    -- Логируем
    INSERT INTO transactions (from_id, to_id, amount, created_at)
    VALUES (from_account_id, to_account_id, amount, NOW());

    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Использование
SELECT transfer_money(1, 2, 500.00);
```

### Оптимистичная блокировка

```sql
-- Таблица с версией
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL,
    version INT DEFAULT 1
);

-- Обновление с проверкой версии
UPDATE products
SET
    price = 150.00,
    version = version + 1
WHERE id = 1 AND version = 5;

-- Если version изменилась, rows affected = 0
-- Нужно перечитать данные и попробовать снова
```

### Очередь задач с SKIP LOCKED

```sql
-- Таблица задач
CREATE TABLE tasks (
    id SERIAL PRIMARY KEY,
    payload JSONB,
    status VARCHAR(20) DEFAULT 'pending',
    locked_at TIMESTAMP,
    processed_at TIMESTAMP
);

-- Воркер забирает задачу
WITH task AS (
    SELECT id
    FROM tasks
    WHERE status = 'pending'
    ORDER BY id
    FOR UPDATE SKIP LOCKED
    LIMIT 1
)
UPDATE tasks
SET
    status = 'processing',
    locked_at = NOW()
FROM task
WHERE tasks.id = task.id
RETURNING tasks.*;
```

---

## Примеры на Python

### psycopg2 — низкоуровневая работа

```python
import psycopg2
from psycopg2 import sql, errors
from contextlib import contextmanager

# Контекстный менеджер для транзакций
@contextmanager
def transaction(connection):
    """Контекстный менеджер для транзакций с автоматическим коммитом/роллбеком."""
    cursor = connection.cursor()
    try:
        yield cursor
        connection.commit()
    except Exception as e:
        connection.rollback()
        raise e
    finally:
        cursor.close()

# Подключение
conn = psycopg2.connect(
    host="localhost",
    database="mydb",
    user="user",
    password="password"
)

# Отключаем autocommit для ручного управления транзакциями
conn.autocommit = False

# Использование
try:
    with transaction(conn) as cur:
        # Блокируем строки
        cur.execute("""
            SELECT balance FROM accounts
            WHERE id IN (%s, %s)
            ORDER BY id
            FOR UPDATE
        """, (1, 2))

        balances = cur.fetchall()

        # Выполняем перевод
        cur.execute("""
            UPDATE accounts SET balance = balance - %s WHERE id = %s
        """, (100, 1))

        cur.execute("""
            UPDATE accounts SET balance = balance + %s WHERE id = %s
        """, (100, 2))

        print("Перевод выполнен успешно")

except errors.SerializationFailure as e:
    print(f"Ошибка сериализации, нужно повторить: {e}")
except errors.DeadlockDetected as e:
    print(f"Обнаружен deadlock: {e}")
except Exception as e:
    print(f"Ошибка: {e}")
finally:
    conn.close()
```

### psycopg2 — уровни изоляции

```python
import psycopg2
from psycopg2.extensions import (
    ISOLATION_LEVEL_AUTOCOMMIT,
    ISOLATION_LEVEL_READ_COMMITTED,
    ISOLATION_LEVEL_REPEATABLE_READ,
    ISOLATION_LEVEL_SERIALIZABLE
)

conn = psycopg2.connect(...)

# Установка уровня изоляции
conn.set_isolation_level(ISOLATION_LEVEL_SERIALIZABLE)

# Или для конкретной транзакции
with conn.cursor() as cur:
    cur.execute("BEGIN")
    cur.execute("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE")
    # ... операции
    cur.execute("COMMIT")
```

### SQLAlchemy — ORM подход

```python
from sqlalchemy import create_engine, Column, Integer, Numeric, String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from sqlalchemy.exc import IntegrityError
from sqlalchemy import event

Base = declarative_base()

class Account(Base):
    __tablename__ = 'accounts'

    id = Column(Integer, primary_key=True)
    owner = Column(String(100))
    balance = Column(Numeric(10, 2))
    version = Column(Integer, default=1)  # Для оптимистичной блокировки

engine = create_engine('postgresql://user:pass@localhost/mydb')
Session = sessionmaker(bind=engine)

def transfer_money(from_id: int, to_id: int, amount: float):
    """Перевод денег с пессимистичной блокировкой."""
    session = Session()

    try:
        # Получаем аккаунты с блокировкой
        # ORDER BY для предотвращения deadlock
        accounts = session.query(Account)\
            .filter(Account.id.in_([from_id, to_id]))\
            .order_by(Account.id)\
            .with_for_update()\
            .all()

        if len(accounts) != 2:
            raise ValueError("Один или оба аккаунта не найдены")

        from_acc = next(a for a in accounts if a.id == from_id)
        to_acc = next(a for a in accounts if a.id == to_id)

        if from_acc.balance < amount:
            raise ValueError("Недостаточно средств")

        from_acc.balance -= amount
        to_acc.balance += amount

        session.commit()
        return True

    except Exception as e:
        session.rollback()
        raise
    finally:
        session.close()
```

### SQLAlchemy — оптимистичная блокировка

```python
from sqlalchemy.orm import Session
from sqlalchemy import event

class Product(Base):
    __tablename__ = 'products'

    id = Column(Integer, primary_key=True)
    name = Column(String(100))
    price = Column(Numeric(10, 2))
    version = Column(Integer, default=1)

    __mapper_args__ = {
        "version_id_col": version  # Включаем версионирование
    }

def update_price(product_id: int, new_price: float, max_retries: int = 3):
    """Обновление с оптимистичной блокировкой и повторами."""
    from sqlalchemy.orm.exc import StaleDataError

    for attempt in range(max_retries):
        session = Session()
        try:
            product = session.query(Product).get(product_id)
            if not product:
                raise ValueError("Продукт не найден")

            product.price = new_price
            session.commit()
            return True

        except StaleDataError:
            session.rollback()
            if attempt == max_retries - 1:
                raise
            print(f"Конфликт версий, попытка {attempt + 2}...")
            continue
        finally:
            session.close()
```

### SQLAlchemy — контекстные менеджеры

```python
from contextlib import contextmanager
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, scoped_session

engine = create_engine('postgresql://user:pass@localhost/mydb')
session_factory = sessionmaker(bind=engine)
ScopedSession = scoped_session(session_factory)

@contextmanager
def transaction_scope(isolation_level=None):
    """Контекстный менеджер для транзакций."""
    session = ScopedSession()

    if isolation_level:
        session.connection(execution_options={
            "isolation_level": isolation_level
        })

    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        ScopedSession.remove()

# Использование
with transaction_scope(isolation_level="SERIALIZABLE") as session:
    account = session.query(Account).filter_by(id=1).with_for_update().first()
    account.balance += 100
```

### Асинхронная работа с asyncpg

```python
import asyncio
import asyncpg

async def transfer_money_async(pool, from_id: int, to_id: int, amount: float):
    """Асинхронный перевод денег."""
    async with pool.acquire() as conn:
        async with conn.transaction(isolation='serializable'):
            # Блокируем строки
            rows = await conn.fetch("""
                SELECT id, balance FROM accounts
                WHERE id = ANY($1::int[])
                ORDER BY id
                FOR UPDATE
            """, [from_id, to_id])

            balances = {row['id']: row['balance'] for row in rows}

            if balances.get(from_id, 0) < amount:
                raise ValueError("Недостаточно средств")

            await conn.execute("""
                UPDATE accounts SET balance = balance - $1 WHERE id = $2
            """, amount, from_id)

            await conn.execute("""
                UPDATE accounts SET balance = balance + $1 WHERE id = $2
            """, amount, to_id)

async def main():
    pool = await asyncpg.create_pool(
        'postgresql://user:pass@localhost/mydb',
        min_size=5,
        max_size=20
    )

    try:
        await transfer_money_async(pool, 1, 2, 100.0)
        print("Перевод выполнен")
    except asyncpg.SerializationError:
        print("Конфликт сериализации, нужно повторить")
    finally:
        await pool.close()

asyncio.run(main())
```

---

## Best Practices

### 1. Минимизируйте время транзакции

```python
# Плохо: долгая транзакция
with transaction_scope() as session:
    data = session.query(HugeTable).all()
    processed = external_api_call(data)  # Долгий вызов!
    session.add(Result(data=processed))

# Хорошо: разделяем чтение, обработку и запись
with transaction_scope() as session:
    data = session.query(HugeTable).all()

processed = external_api_call(data)  # Вне транзакции

with transaction_scope() as session:
    session.add(Result(data=processed))
```

### 2. Выбирайте правильный уровень изоляции

```python
# READ COMMITTED — для большинства случаев (по умолчанию)
# Быстро, достаточно для простых операций

# REPEATABLE READ — когда нужно несколько чтений одних данных
with transaction_scope(isolation_level="REPEATABLE_READ") as session:
    total = session.query(func.sum(Order.amount)).scalar()
    orders = session.query(Order).all()
    # total соответствует orders

# SERIALIZABLE — для критичных финансовых операций
with transaction_scope(isolation_level="SERIALIZABLE") as session:
    # Гарантированная согласованность
    pass
```

### 3. Обрабатывайте ошибки сериализации

```python
from sqlalchemy.exc import OperationalError
import time
import random

def with_retry(func, max_retries=3, base_delay=0.1):
    """Декоратор для повторов при ошибках сериализации."""
    def wrapper(*args, **kwargs):
        for attempt in range(max_retries):
            try:
                return func(*args, **kwargs)
            except OperationalError as e:
                if 'serialization' in str(e).lower() or 'deadlock' in str(e).lower():
                    if attempt == max_retries - 1:
                        raise
                    # Exponential backoff с jitter
                    delay = base_delay * (2 ** attempt) + random.uniform(0, 0.1)
                    time.sleep(delay)
                else:
                    raise
    return wrapper

@with_retry
def critical_operation():
    with transaction_scope(isolation_level="SERIALIZABLE") as session:
        # Критичная операция
        pass
```

### 4. Упорядочивайте доступ к ресурсам

```python
def update_multiple_accounts(account_ids: list, amounts: dict):
    """Обновление нескольких аккаунтов без deadlock."""
    # Всегда сортируем ID для предсказуемого порядка блокировок
    sorted_ids = sorted(account_ids)

    with transaction_scope() as session:
        accounts = session.query(Account)\
            .filter(Account.id.in_(sorted_ids))\
            .order_by(Account.id)\
            .with_for_update()\
            .all()

        for account in accounts:
            if account.id in amounts:
                account.balance += amounts[account.id]
```

### 5. Используйте SKIP LOCKED для очередей

```python
def get_next_task():
    """Получение задачи из очереди без блокировки других воркеров."""
    with transaction_scope() as session:
        task = session.query(Task)\
            .filter(Task.status == 'pending')\
            .order_by(Task.created_at)\
            .with_for_update(skip_locked=True)\
            .first()

        if task:
            task.status = 'processing'
            task.started_at = datetime.utcnow()
            return task.id
        return None
```

### 6. Логируйте транзакции

```python
import logging
import uuid
from functools import wraps

logger = logging.getLogger(__name__)

def logged_transaction(func):
    """Декоратор для логирования транзакций."""
    @wraps(func)
    def wrapper(*args, **kwargs):
        tx_id = str(uuid.uuid4())[:8]
        logger.info(f"[TX-{tx_id}] Starting: {func.__name__}")

        try:
            result = func(*args, **kwargs)
            logger.info(f"[TX-{tx_id}] Committed: {func.__name__}")
            return result
        except Exception as e:
            logger.error(f"[TX-{tx_id}] Rolled back: {func.__name__} - {e}")
            raise

    return wrapper

@logged_transaction
def transfer_money(from_id, to_id, amount):
    # ...
    pass
```

### 7. Избегайте транзакций в циклах

```python
# Плохо: много маленьких транзакций
for item in items:
    with transaction_scope() as session:
        session.add(Item(**item))

# Хорошо: одна транзакция для пакета
with transaction_scope() as session:
    for item in items:
        session.add(Item(**item))

# Ещё лучше: bulk insert
with transaction_scope() as session:
    session.bulk_insert_mappings(Item, items)
```

---

## Распределённые транзакции (2PC)

### Проблема распределённых транзакций

Когда данные хранятся в нескольких базах данных или сервисах, нужно гарантировать атомарность операций между ними.

### Two-Phase Commit (2PC)

**Фаза 1: Prepare (Подготовка)**
- Координатор отправляет запрос PREPARE всем участникам
- Каждый участник готовит транзакцию и отвечает READY или ABORT

**Фаза 2: Commit/Rollback**
- Если все ответили READY — координатор отправляет COMMIT
- Если хоть один ABORT — координатор отправляет ROLLBACK

```
Координатор                Участник A         Участник B
     │                          │                  │
     │──── PREPARE ────────────>│                  │
     │──── PREPARE ─────────────┼─────────────────>│
     │                          │                  │
     │<─── READY ───────────────│                  │
     │<─── READY ───────────────┼──────────────────│
     │                          │                  │
     │──── COMMIT ─────────────>│                  │
     │──── COMMIT ──────────────┼─────────────────>│
     │                          │                  │
     │<─── ACK ─────────────────│                  │
     │<─── ACK ─────────────────┼──────────────────│
```

### PostgreSQL Prepared Transactions

```sql
-- Участник A
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
PREPARE TRANSACTION 'tx_transfer_a';

-- Участник B
BEGIN;
UPDATE accounts SET balance = balance + 100 WHERE id = 1;
PREPARE TRANSACTION 'tx_transfer_b';

-- Координатор проверяет, что оба PREPARED успешно
-- Если да:
COMMIT PREPARED 'tx_transfer_a';
COMMIT PREPARED 'tx_transfer_b';

-- Если нет:
ROLLBACK PREPARED 'tx_transfer_a';
ROLLBACK PREPARED 'tx_transfer_b';
```

### Проблемы 2PC

1. **Блокировка при сбое координатора** — участники ждут решения
2. **Производительность** — много сетевых вызовов
3. **Не масштабируется** — чем больше участников, тем хуже

### Альтернативы 2PC

#### Saga Pattern

Saga разбивает распределённую транзакцию на локальные транзакции с компенсирующими действиями.

```python
class TransferSaga:
    """Saga для перевода между сервисами."""

    def __init__(self, from_service, to_service):
        self.from_service = from_service
        self.to_service = to_service
        self.steps_completed = []

    def execute(self, from_id: int, to_id: int, amount: float):
        try:
            # Шаг 1: Резервируем средства
            reservation_id = self.from_service.reserve(from_id, amount)
            self.steps_completed.append(('reserve', reservation_id))

            # Шаг 2: Пополняем счёт получателя
            credit_id = self.to_service.credit(to_id, amount)
            self.steps_completed.append(('credit', credit_id))

            # Шаг 3: Подтверждаем списание
            self.from_service.confirm_debit(reservation_id)
            self.steps_completed.append(('confirm', reservation_id))

            return True

        except Exception as e:
            self.compensate()
            raise

    def compensate(self):
        """Откат выполненных шагов в обратном порядке."""
        for step, step_id in reversed(self.steps_completed):
            try:
                if step == 'confirm':
                    pass  # Уже подтверждено, ничего не делаем
                elif step == 'credit':
                    self.to_service.reverse_credit(step_id)
                elif step == 'reserve':
                    self.from_service.cancel_reservation(step_id)
            except Exception as e:
                # Логируем и продолжаем откат
                logger.error(f"Compensation failed for {step}: {e}")
```

#### Outbox Pattern

Гарантирует атомарность между изменением данных и публикацией события.

```python
class OutboxService:
    """Паттерн Outbox для надёжной публикации событий."""

    def transfer_with_event(self, session, from_id: int, to_id: int, amount: float):
        # Всё в одной транзакции!
        account = session.query(Account).filter_by(id=from_id).with_for_update().first()
        account.balance -= amount

        # Событие сохраняется в той же БД
        event = OutboxEvent(
            aggregate_type='Account',
            aggregate_id=from_id,
            event_type='MoneyTransferred',
            payload={
                'from_id': from_id,
                'to_id': to_id,
                'amount': str(amount)
            }
        )
        session.add(event)

        # Коммит гарантирует атомарность изменения и события

# Отдельный процесс публикует события
class OutboxPublisher:
    def publish_pending_events(self):
        with transaction_scope() as session:
            events = session.query(OutboxEvent)\
                .filter_by(published=False)\
                .with_for_update(skip_locked=True)\
                .limit(100)\
                .all()

            for event in events:
                try:
                    self.message_broker.publish(event.event_type, event.payload)
                    event.published = True
                    event.published_at = datetime.utcnow()
                except Exception as e:
                    logger.error(f"Failed to publish event {event.id}: {e}")
```

---

## Краткое резюме

### Основные концепции

| Концепция | Описание |
|-----------|----------|
| **Транзакция** | Атомарная единица работы с БД (всё или ничего) |
| **ACID** | Atomicity, Consistency, Isolation, Durability |
| **BEGIN/COMMIT/ROLLBACK** | Управление жизненным циклом транзакции |
| **SAVEPOINT** | Точка сохранения для частичного отката |

### Уровни изоляции

| Уровень | Когда использовать |
|---------|-------------------|
| **READ COMMITTED** | По умолчанию, для большинства случаев |
| **REPEATABLE READ** | Когда нужны согласованные повторные чтения |
| **SERIALIZABLE** | Для критичных финансовых операций |

### Типы блокировок

| Тип | Использование |
|-----|--------------|
| **FOR UPDATE** | Эксклюзивная блокировка для изменения |
| **FOR SHARE** | Разделяемая блокировка для чтения |
| **SKIP LOCKED** | Для очередей задач |
| **NOWAIT** | Немедленная ошибка при невозможности блокировки |

### Best Practices

1. **Минимизируйте время транзакции** — выносите тяжёлые операции за пределы транзакции
2. **Упорядочивайте блокировки** — всегда в одном порядке для предотвращения deadlock
3. **Обрабатывайте ошибки сериализации** — retry с exponential backoff
4. **Используйте правильный уровень изоляции** — не переусложняйте
5. **Логируйте транзакции** — для отладки и мониторинга

### Распределённые транзакции

| Паттерн | Когда использовать |
|---------|-------------------|
| **2PC** | Когда нужна строгая согласованность (редко) |
| **Saga** | Для длительных бизнес-процессов |
| **Outbox** | Для надёжной публикации событий |

### Полезные команды PostgreSQL

```sql
-- Проверить уровень изоляции
SHOW transaction_isolation;

-- Посмотреть активные блокировки
SELECT * FROM pg_locks WHERE granted = false;

-- Найти блокирующие запросы
SELECT * FROM pg_stat_activity WHERE state = 'active';

-- Убить зависшую транзакцию
SELECT pg_terminate_backend(pid);
```
