# ACID — Фундаментальные свойства транзакций

## Введение

ACID — это акроним, описывающий четыре ключевых свойства, которые гарантируют надёжность транзакций в реляционных базах данных. Эти свойства обеспечивают целостность данных даже при сбоях системы, ошибках приложений или параллельном доступе.

```
┌─────────────────────────────────────────────────────────────┐
│                         ACID                                │
├───────────────┬───────────────┬───────────────┬─────────────┤
│   Atomicity   │  Consistency  │   Isolation   │ Durability  │
│  (Атомарность)│(Согласованность)│ (Изоляция)  │(Долговечность)│
└───────────────┴───────────────┴───────────────┴─────────────┘
```

## Atomicity (Атомарность)

**Атомарность** гарантирует, что транзакция выполняется как единое целое — либо все операции успешны, либо ни одна не применяется.

### Принцип "Всё или ничего"

```sql
-- Пример: перевод денег между счетами
BEGIN;
    UPDATE accounts SET balance = balance - 1000 WHERE id = 1;  -- Снятие
    UPDATE accounts SET balance = balance + 1000 WHERE id = 2;  -- Зачисление
COMMIT;

-- Если произойдёт сбой между операциями, обе будут отменены
-- Деньги не "потеряются" в системе
```

### Как PostgreSQL обеспечивает атомарность

1. **Write-Ahead Logging (WAL)** — все изменения сначала записываются в журнал
2. **Undo-механизм** — возможность откатить незавершённые операции
3. **Transaction ID (XID)** — каждая транзакция имеет уникальный идентификатор

```sql
-- Проверка текущего transaction ID
SELECT txid_current();

-- Принудительный откат транзакции
BEGIN;
    INSERT INTO orders (product_id, quantity) VALUES (100, 5);
    -- Что-то пошло не так...
ROLLBACK;  -- Все изменения отменены
```

## Consistency (Согласованность)

**Согласованность** гарантирует, что транзакция переводит базу данных из одного корректного состояния в другое, соблюдая все ограничения и правила.

### Виды ограничений целостности

```sql
-- Ограничения, обеспечивающие согласованность
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,                    -- Уникальность
    email VARCHAR(255) UNIQUE NOT NULL,       -- Уникальность + NOT NULL
    balance DECIMAL(10,2) CHECK (balance >= 0), -- Проверка условия
    user_id INTEGER REFERENCES users(id)      -- Внешний ключ
);

-- Пример нарушения согласованности
BEGIN;
    UPDATE accounts SET balance = balance - 5000
    WHERE id = 1 AND balance >= 5000;  -- Проверка перед снятием

    -- Если баланс < 5000, CHECK constraint предотвратит операцию
COMMIT;
```

### Триггеры для бизнес-правил

```sql
-- Триггер для проверки бизнес-логики
CREATE OR REPLACE FUNCTION check_order_limit()
RETURNS TRIGGER AS $$
BEGIN
    IF (SELECT SUM(amount) FROM orders WHERE user_id = NEW.user_id) > 100000 THEN
        RAISE EXCEPTION 'Превышен лимит заказов для пользователя';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER enforce_order_limit
    BEFORE INSERT ON orders
    FOR EACH ROW EXECUTE FUNCTION check_order_limit();
```

## Isolation (Изоляция)

**Изоляция** гарантирует, что параллельно выполняющиеся транзакции не влияют друг на друга неожиданным образом.

### Проблемы при отсутствии изоляции

```
┌────────────────────────────────────────────────────────────────┐
│                    Аномалии чтения                             │
├──────────────────┬─────────────────────────────────────────────┤
│ Dirty Read       │ Чтение незафиксированных данных             │
│ Non-Repeatable   │ Разные результаты одного запроса в          │
│ Read             │ рамках транзакции                           │
│ Phantom Read     │ Появление/исчезновение строк между          │
│                  │ запросами                                   │
└──────────────────┴─────────────────────────────────────────────┘
```

### Уровни изоляции в PostgreSQL

```sql
-- Установка уровня изоляции
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;      -- По умолчанию
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Пример использования
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    SELECT * FROM inventory WHERE product_id = 100;
    UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 100;
COMMIT;
```

| Уровень | Dirty Read | Non-Repeatable Read | Phantom Read |
|---------|------------|---------------------|--------------|
| Read Uncommitted* | Да | Да | Да |
| Read Committed | Нет | Да | Да |
| Repeatable Read | Нет | Нет | Возможно** |
| Serializable | Нет | Нет | Нет |

*В PostgreSQL Read Uncommitted работает как Read Committed
**PostgreSQL предотвращает phantom reads и на уровне Repeatable Read

## Durability (Долговечность)

**Долговечность** гарантирует, что после успешного завершения транзакции (COMMIT) данные сохранятся даже при сбое системы.

### Механизмы обеспечения долговечности

```
┌─────────────────────────────────────────────────────────────┐
│                     Запись данных                           │
│                                                             │
│   Приложение → WAL Buffer → WAL на диске → Data Files      │
│                     ↓                                       │
│              fsync при COMMIT                               │
└─────────────────────────────────────────────────────────────┘
```

### Настройки долговечности

```sql
-- Синхронный commit (максимальная надёжность, по умолчанию)
SET synchronous_commit = on;

-- Асинхронный commit (выше производительность, риск потери данных)
SET synchronous_commit = off;

-- Проверка настроек
SHOW wal_sync_method;
SHOW fsync;
```

### Параметры PostgreSQL для долговечности

```ini
# postgresql.conf

# Обязательная синхронизация с диском
fsync = on

# Метод синхронизации WAL
wal_sync_method = fdatasync

# Уровень WAL
wal_level = replica

# Количество WAL-сегментов
min_wal_size = 80MB
max_wal_size = 1GB
```

## Практический пример: банковская транзакция

```sql
-- Полный пример с соблюдением всех ACID свойств
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

    -- Проверяем баланс отправителя (Consistency)
    DO $$
    DECLARE
        sender_balance DECIMAL;
    BEGIN
        SELECT balance INTO sender_balance
        FROM accounts WHERE id = 1 FOR UPDATE;

        IF sender_balance < 1000 THEN
            RAISE EXCEPTION 'Недостаточно средств';
        END IF;
    END $$;

    -- Atomicity: обе операции или ни одной
    UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
    UPDATE accounts SET balance = balance + 1000 WHERE id = 2;

    -- Логирование транзакции
    INSERT INTO transaction_log (from_account, to_account, amount, timestamp)
    VALUES (1, 2, 1000, NOW());

COMMIT;  -- Durability: после этого данные гарантированно сохранены
```

## Best Practices

### 1. Минимизируйте время транзакций
```sql
-- Плохо: долгая транзакция
BEGIN;
    SELECT * FROM large_table;  -- Обработка данных
    -- ... много времени ...
    UPDATE accounts SET processed = true WHERE id = 1;
COMMIT;

-- Хорошо: подготовка вне транзакции
-- Подготовка данных
-- ...
BEGIN;
    UPDATE accounts SET processed = true WHERE id = 1;
COMMIT;
```

### 2. Используйте подходящий уровень изоляции
```sql
-- Для чтения отчётов достаточно Read Committed
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT * FROM sales_summary;

-- Для критичных операций — Serializable
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 100;
```

### 3. Обрабатывайте ошибки сериализации
```sql
-- Псевдокод для retry-логики
DO $$
DECLARE
    max_retries INTEGER := 3;
    retry_count INTEGER := 0;
BEGIN
    LOOP
        BEGIN
            -- Критичная операция
            PERFORM transfer_money(1, 2, 1000);
            EXIT;  -- Успех, выходим из цикла
        EXCEPTION
            WHEN serialization_failure OR deadlock_detected THEN
                retry_count := retry_count + 1;
                IF retry_count >= max_retries THEN
                    RAISE;
                END IF;
                PERFORM pg_sleep(0.1 * retry_count);  -- Exponential backoff
        END;
    END LOOP;
END $$;
```

## Компромиссы ACID

| Свойство | Усиление | Компромисс |
|----------|----------|------------|
| Atomicity | Использовать savepoints | Накладные расходы на журналирование |
| Consistency | Больше constraints | Снижение производительности INSERT/UPDATE |
| Isolation | Serializable level | Больше блокировок и retry |
| Durability | synchronous_commit = on | Задержка на fsync |

## Заключение

ACID-свойства являются фундаментом надёжности реляционных баз данных. PostgreSQL полностью реализует все четыре свойства, предоставляя гибкие настройки для баланса между надёжностью и производительностью. Понимание этих концепций критически важно для проектирования надёжных приложений.
