# ACID

## Введение

**ACID** — это акроним, описывающий четыре ключевых свойства транзакций в реляционных базах данных, которые гарантируют надёжность и предсказуемость операций с данными:

- **A**tomicity (Атомарность)
- **C**onsistency (Согласованность)
- **I**solation (Изоляция)
- **D**urability (Долговечность)

Эти свойства были сформулированы Джимом Греем в 1970-х годах и стали фундаментом для проектирования надёжных систем управления базами данных.

### Почему ACID важен?

В реальных приложениях данные постоянно изменяются: пользователи совершают покупки, переводят деньги, обновляют профили. Без ACID-гарантий:
- Деньги могут "исчезнуть" при переводе между счетами
- Два пользователя могут купить последний товар на складе
- Данные могут быть потеряны при сбое сервера
- Параллельные операции могут привести к некорректному состоянию

---

## Atomicity (Атомарность)

### Определение

**Атомарность** гарантирует, что транзакция выполняется как единая неделимая единица работы: либо все операции внутри транзакции применяются успешно, либо ни одна из них не применяется.

### Принцип "Всё или ничего"

Представьте банковский перевод денег с одного счёта на другой:

```sql
BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 1000 WHERE id = 1;  -- Списание
    UPDATE accounts SET balance = balance + 1000 WHERE id = 2;  -- Зачисление
COMMIT;
```

Если между этими двумя операциями произойдёт сбой (отключение питания, ошибка сети, краш сервера), атомарность гарантирует:
- Либо обе операции выполнены
- Либо ни одна не выполнена (откат к исходному состоянию)

### Механизм реализации

БД использует **журнал транзакций (Transaction Log / WAL)** для обеспечения атомарности:

1. Перед изменением данных, БД записывает намерение в журнал
2. Выполняются изменения в памяти
3. При COMMIT — изменения фиксируются
4. При сбое — БД откатывает незавершённые транзакции по журналу

### Пример нарушения атомарности

```sql
-- БЕЗ транзакции (опасно!)
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
-- Здесь происходит сбой сервера...
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;
-- Эта строка не выполнится, деньги "потеряны"
```

### Откат транзакции (ROLLBACK)

```sql
BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 1000 WHERE id = 1;

    -- Проверяем, достаточно ли средств
    IF (SELECT balance FROM accounts WHERE id = 1) < 0 THEN
        ROLLBACK;  -- Отменяем всю транзакцию
        RETURN;
    END IF;

    UPDATE accounts SET balance = balance + 1000 WHERE id = 2;
COMMIT;
```

---

## Consistency (Согласованность)

### Определение

**Согласованность** гарантирует, что транзакция переводит базу данных из одного корректного (согласованного) состояния в другое корректное состояние. Все правила целостности данных должны соблюдаться.

### Типы ограничений целостности

```sql
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    user_id INT NOT NULL REFERENCES users(id),      -- Ссылочная целостность
    balance DECIMAL(15, 2) CHECK (balance >= 0),    -- Проверка условия
    account_type VARCHAR(20) NOT NULL,              -- NOT NULL ограничение
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (user_id, account_type)                  -- Уникальность
);
```

### Примеры согласованности

#### 1. Ссылочная целостность (Foreign Key)

```sql
-- Таблица заказов ссылается на пользователей
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id) ON DELETE RESTRICT,
    total DECIMAL(10, 2)
);

-- Попытка создать заказ для несуществующего пользователя — ошибка
INSERT INTO orders (user_id, total) VALUES (999999, 100.00);
-- ERROR: insert or update on table "orders" violates foreign key constraint
```

#### 2. Проверка условий (CHECK)

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10, 2) CHECK (price > 0),
    quantity INT CHECK (quantity >= 0)
);

-- Попытка установить отрицательную цену — ошибка
UPDATE products SET price = -10 WHERE id = 1;
-- ERROR: new row for relation "products" violates check constraint
```

#### 3. Бизнес-правила через триггеры

```sql
-- Триггер проверяет, что сумма всех балансов не изменяется при переводе
CREATE OR REPLACE FUNCTION check_transfer_balance()
RETURNS TRIGGER AS $$
BEGIN
    IF (SELECT SUM(balance) FROM accounts) !=
       (SELECT SUM(balance) FROM accounts_snapshot) THEN
        RAISE EXCEPTION 'Нарушение баланса системы!';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### Согласованность vs Атомарность

- **Атомарность** — техническое свойство (всё или ничего)
- **Согласованность** — логическое свойство (данные соответствуют бизнес-правилам)

---

## Isolation (Изоляция)

### Определение

**Изоляция** определяет, насколько транзакции "видят" изменения друг друга во время параллельного выполнения. Идеальная изоляция означает, что каждая транзакция выполняется так, будто она единственная в системе.

### Проблемы параллельного доступа

#### 1. Dirty Read (Грязное чтение)

Транзакция читает данные, изменённые другой незавершённой транзакцией.

```
Транзакция A                    Транзакция B
─────────────                   ─────────────
BEGIN;
UPDATE accounts
SET balance = 500
WHERE id = 1;
                                BEGIN;
                                SELECT balance FROM accounts
                                WHERE id = 1;
                                -- Читает 500 (грязные данные!)
ROLLBACK;
                                -- Транзакция B использует
                                -- данные, которых нет в БД
```

#### 2. Non-Repeatable Read (Неповторяющееся чтение)

Повторное чтение тех же данных в транзакции даёт разные результаты.

```
Транзакция A                    Транзакция B
─────────────                   ─────────────
BEGIN;
SELECT balance FROM accounts
WHERE id = 1;
-- Читает 1000
                                BEGIN;
                                UPDATE accounts
                                SET balance = 500
                                WHERE id = 1;
                                COMMIT;
SELECT balance FROM accounts
WHERE id = 1;
-- Читает 500 (другое значение!)
```

#### 3. Phantom Read (Фантомное чтение)

Повторный запрос возвращает новые строки, добавленные другой транзакцией.

```
Транзакция A                    Транзакция B
─────────────                   ─────────────
BEGIN;
SELECT COUNT(*) FROM orders
WHERE status = 'pending';
-- Возвращает 10
                                BEGIN;
                                INSERT INTO orders
                                (status) VALUES ('pending');
                                COMMIT;
SELECT COUNT(*) FROM orders
WHERE status = 'pending';
-- Возвращает 11 (фантомная строка!)
```

#### 4. Lost Update (Потерянное обновление)

Два параллельных обновления, где одно перезаписывает другое.

```
Транзакция A                    Транзакция B
─────────────                   ─────────────
BEGIN;
SELECT balance FROM accounts
WHERE id = 1;
-- Читает 1000
                                BEGIN;
                                SELECT balance FROM accounts
                                WHERE id = 1;
                                -- Читает 1000
UPDATE accounts
SET balance = 1100
WHERE id = 1;  -- +100
                                UPDATE accounts
                                SET balance = 1200
                                WHERE id = 1;  -- +200
COMMIT;
                                COMMIT;
-- Итог: balance = 1200
-- Потеряно обновление +100!
```

### Уровни изоляции SQL

| Уровень | Dirty Read | Non-Repeatable Read | Phantom Read |
|---------|------------|---------------------|--------------|
| READ UNCOMMITTED | Возможно | Возможно | Возможно |
| READ COMMITTED | Невозможно | Возможно | Возможно |
| REPEATABLE READ | Невозможно | Невозможно | Возможно* |
| SERIALIZABLE | Невозможно | Невозможно | Невозможно |

*В PostgreSQL REPEATABLE READ также защищает от фантомного чтения

### Установка уровня изоляции

```sql
-- Для текущей транзакции
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- ... операции ...
COMMIT;

-- Для сессии
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Проверка текущего уровня
SHOW transaction_isolation;
```

### Примеры использования уровней

#### READ COMMITTED (по умолчанию в PostgreSQL)

```sql
-- Подходит для большинства OLTP операций
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
    SELECT * FROM products WHERE id = 1;
    UPDATE products SET quantity = quantity - 1 WHERE id = 1;
COMMIT;
```

#### REPEATABLE READ

```sql
-- Подходит для отчётов, где важна консистентность данных
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    SELECT SUM(balance) FROM accounts;  -- Первый подсчёт
    -- Даже если другие транзакции меняют данные...
    SELECT SUM(balance) FROM accounts;  -- Результат тот же
COMMIT;
```

#### SERIALIZABLE

```sql
-- Для критичных финансовых операций
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    -- Проверяем баланс и списываем
    SELECT balance FROM accounts WHERE id = 1;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    -- Гарантированно атомарно и изолированно
COMMIT;
```

### Механизмы реализации изоляции

1. **Блокировки (Locks)** — пессимистичный подход
2. **MVCC (Multi-Version Concurrency Control)** — оптимистичный подход (PostgreSQL)

---

## Durability (Долговечность)

### Определение

**Долговечность** гарантирует, что после успешного завершения транзакции (COMMIT), все её изменения сохранены в энергонезависимой памяти и переживут любые сбои: отключение питания, краш системы, аппаратные ошибки.

### Механизмы обеспечения долговечности

#### 1. Write-Ahead Logging (WAL)

```
┌─────────────────────────────────────────────────────┐
│                    Процесс записи                    │
├─────────────────────────────────────────────────────┤
│ 1. Транзакция изменяет данные в памяти              │
│ 2. Изменения записываются в WAL (на диск)           │
│ 3. COMMIT подтверждается клиенту                    │
│ 4. Позже данные переносятся в основные файлы        │
└─────────────────────────────────────────────────────┘
```

#### 2. Checkpoint

Периодически БД создаёт контрольные точки:
- Все изменения из памяти записываются на диск
- WAL до этой точки можно удалить
- Ускоряет восстановление после сбоя

```sql
-- Принудительное создание checkpoint (PostgreSQL)
CHECKPOINT;

-- Настройки checkpoint
SHOW checkpoint_timeout;      -- Интервал между checkpoint
SHOW checkpoint_completion_target;  -- Размер WAL
```

#### 3. Синхронная запись (fsync)

```sql
-- Проверка настройки синхронной записи
SHOW fsync;  -- on = данные гарантированно на диске

-- Настройка уровня синхронизации
SHOW synchronous_commit;
-- on     = ждать записи на диск (максимальная надёжность)
-- off    = не ждать (быстрее, но риск потери данных)
-- local  = синхронно локально
```

### Восстановление после сбоя

```
┌─────────────────────────────────────────────────────┐
│                 Процесс восстановления               │
├─────────────────────────────────────────────────────┤
│ 1. БД запускается после сбоя                        │
│ 2. Читается последний checkpoint                     │
│ 3. Проигрываются записи WAL после checkpoint        │
│ 4. Откатываются незавершённые транзакции            │
│ 5. БД готова к работе                               │
└─────────────────────────────────────────────────────┘
```

### Репликация для дополнительной защиты

```sql
-- Настройка синхронной репликации (PostgreSQL)
synchronous_standby_names = 'replica1'
synchronous_commit = remote_write

-- COMMIT подтверждается только когда данные
-- записаны на реплику
```

---

## Примеры кода SQL

### Полный пример банковского перевода

```sql
-- Создание таблицы счетов
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    owner_name VARCHAR(100) NOT NULL,
    balance DECIMAL(15, 2) NOT NULL DEFAULT 0,
    CHECK (balance >= 0)
);

-- Создание таблицы истории операций
CREATE TABLE transactions_log (
    id SERIAL PRIMARY KEY,
    from_account INT REFERENCES accounts(id),
    to_account INT REFERENCES accounts(id),
    amount DECIMAL(15, 2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(20) NOT NULL
);

-- Функция перевода денег с ACID-гарантиями
CREATE OR REPLACE FUNCTION transfer_money(
    p_from_account INT,
    p_to_account INT,
    p_amount DECIMAL(15, 2)
) RETURNS BOOLEAN AS $$
DECLARE
    v_from_balance DECIMAL(15, 2);
BEGIN
    -- Начало транзакции уже неявно начато в функции

    -- Проверяем существование счетов и блокируем строки
    SELECT balance INTO v_from_balance
    FROM accounts
    WHERE id = p_from_account
    FOR UPDATE;  -- Блокировка строки

    IF NOT FOUND THEN
        RAISE EXCEPTION 'Счёт отправителя не найден';
    END IF;

    IF v_from_balance < p_amount THEN
        RAISE EXCEPTION 'Недостаточно средств на счёте';
    END IF;

    -- Проверяем существование счёта получателя
    PERFORM 1 FROM accounts WHERE id = p_to_account FOR UPDATE;
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Счёт получателя не найден';
    END IF;

    -- Выполняем перевод
    UPDATE accounts SET balance = balance - p_amount
    WHERE id = p_from_account;

    UPDATE accounts SET balance = balance + p_amount
    WHERE id = p_to_account;

    -- Логируем транзакцию
    INSERT INTO transactions_log (from_account, to_account, amount, status)
    VALUES (p_from_account, p_to_account, p_amount, 'completed');

    RETURN TRUE;

EXCEPTION
    WHEN OTHERS THEN
        -- Логируем неудачную попытку
        INSERT INTO transactions_log (from_account, to_account, amount, status)
        VALUES (p_from_account, p_to_account, p_amount, 'failed');
        RAISE;
END;
$$ LANGUAGE plpgsql;

-- Использование
SELECT transfer_money(1, 2, 500.00);
```

### Пример с уровнями изоляции

```sql
-- Сессия 1: Создание отчёта с консистентными данными
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Получаем снимок данных на момент начала транзакции
SELECT
    SUM(balance) as total_balance,
    COUNT(*) as total_accounts
FROM accounts;

-- Даже если другие сессии меняют данные,
-- мы видим консистентный снимок
SELECT
    owner_name,
    balance,
    balance * 100.0 / (SELECT SUM(balance) FROM accounts) as percentage
FROM accounts
ORDER BY balance DESC;

COMMIT;
```

### Пример обработки конфликтов

```sql
-- Сессия 1
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT quantity FROM products WHERE id = 1;  -- quantity = 10
-- Планируем уменьшить на 3

-- Сессия 2 (параллельно)
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT quantity FROM products WHERE id = 1;  -- quantity = 10
UPDATE products SET quantity = quantity - 5 WHERE id = 1;
COMMIT;  -- Успешно

-- Сессия 1 (продолжение)
UPDATE products SET quantity = quantity - 3 WHERE id = 1;
COMMIT;
-- ERROR: could not serialize access due to concurrent update
-- Нужно повторить транзакцию!
```

### Обработка ошибок сериализации

```python
import psycopg2
from psycopg2 import extensions
import time

def execute_with_retry(conn, operation, max_retries=3):
    """Выполнение операции с повторами при ошибках сериализации"""
    for attempt in range(max_retries):
        try:
            conn.set_isolation_level(
                extensions.ISOLATION_LEVEL_SERIALIZABLE
            )
            with conn.cursor() as cur:
                operation(cur)
            conn.commit()
            return True
        except psycopg2.errors.SerializationFailure:
            conn.rollback()
            if attempt < max_retries - 1:
                time.sleep(0.1 * (2 ** attempt))  # Exponential backoff
                continue
            raise
    return False

# Использование
def transfer(cur):
    cur.execute(
        "UPDATE accounts SET balance = balance - 100 WHERE id = 1"
    )
    cur.execute(
        "UPDATE accounts SET balance = balance + 100 WHERE id = 2"
    )

execute_with_retry(connection, transfer)
```

---

## ACID vs BASE

### BASE — альтернативный подход

**BASE** — акроним, описывающий свойства распределённых NoSQL систем:

- **B**asically **A**vailable (Базовая доступность) — система всегда отвечает
- **S**oft state (Мягкое состояние) — состояние может меняться со временем
- **E**ventual consistency (Согласованность в конечном счёте) — данные станут консистентными через некоторое время

### Сравнение

| Аспект | ACID | BASE |
|--------|------|------|
| Консистентность | Строгая, немедленная | Eventual, отложенная |
| Доступность | Может блокироваться | Всегда доступен |
| Масштабирование | Сложнее горизонтально | Легко горизонтально |
| Производительность | Ниже из-за блокировок | Выше |
| Сложность логики | Проще для разработчика | Сложнее, нужна компенсация |
| Применение | Финансы, критичные данные | Соцсети, аналитика, логи |

### CAP-теорема

```
       Consistency (Согласованность)
              /\
             /  \
            /    \
           /  CA  \
          /________\
         /\        /\
        /  \      /  \
       / CP \    / AP \
      /______\  /______\
Partition      Availability
Tolerance      (Доступность)
(Устойчивость к разделению)
```

- **CA** — ACID базы данных (PostgreSQL, MySQL)
- **CP** — MongoDB, HBase (consistency + partition tolerance)
- **AP** — Cassandra, DynamoDB (availability + partition tolerance)

### Eventual Consistency пример

```python
# В системе с eventual consistency
# Запись на мастер-ноду
write_to_master("user:123", {"name": "John", "email": "john@example.com"})

# Чтение с реплики (может вернуть старые данные!)
user = read_from_replica("user:123")
# user может ещё не содержать новый email

# Через некоторое время данные синхронизируются
time.sleep(1)  # ждём репликации
user = read_from_replica("user:123")
# Теперь данные актуальны
```

### Когда что использовать

**ACID подходит для:**
- Банковских и финансовых систем
- Систем бронирования
- Инвентаризации и учёта
- Медицинских записей
- Любых данных, где ошибка недопустима

**BASE подходит для:**
- Социальных сетей (лайки, комментарии)
- Аналитики и логирования
- Кэширования
- Систем рекомендаций
- Данных, где временная несогласованность допустима

---

## Типичные проблемы нарушения ACID

### 1. Отсутствие транзакций в коде

```python
# ПЛОХО: Операции без транзакции
def process_order(order_id):
    db.execute("UPDATE inventory SET quantity = quantity - 1")
    # Сбой здесь оставит данные в некорректном состоянии
    db.execute("INSERT INTO orders ...")
    db.execute("UPDATE user_balance ...")

# ХОРОШО: Всё в транзакции
def process_order(order_id):
    with db.transaction():
        db.execute("UPDATE inventory SET quantity = quantity - 1")
        db.execute("INSERT INTO orders ...")
        db.execute("UPDATE user_balance ...")
```

### 2. Слишком долгие транзакции

```python
# ПЛОХО: Транзакция держит блокировки слишком долго
def process_with_api():
    with db.transaction():
        user = db.query("SELECT * FROM users WHERE id = 1 FOR UPDATE")
        # Внешний API вызов внутри транзакции!
        result = external_api.validate(user)  # Может занять секунды
        db.execute("UPDATE users SET validated = true")

# ХОРОШО: Минимизируем время транзакции
def process_with_api():
    user = db.query("SELECT * FROM users WHERE id = 1")
    result = external_api.validate(user)  # Вне транзакции

    with db.transaction():
        # Только критичные операции в транзакции
        db.execute("UPDATE users SET validated = true WHERE id = 1")
```

### 3. Игнорирование уровней изоляции

```python
# ПЛОХО: Чтение-проверка-запись без блокировки
def book_seat(seat_id, user_id):
    seat = db.query("SELECT * FROM seats WHERE id = ?", seat_id)
    if seat.is_available:  # Проверка
        # Другой поток может забронировать между проверкой и записью!
        db.execute("UPDATE seats SET user_id = ? WHERE id = ?", user_id, seat_id)

# ХОРОШО: Атомарная проверка и обновление
def book_seat(seat_id, user_id):
    result = db.execute("""
        UPDATE seats
        SET user_id = %s
        WHERE id = %s AND is_available = true
        RETURNING id
    """, user_id, seat_id)

    if result.rowcount == 0:
        raise Exception("Место уже занято")
```

### 4. Распределённые транзакции без координации

```python
# ПЛОХО: Изменения в разных системах без координации
def transfer_cross_system(amount):
    system_a.debit(amount)   # Успешно
    # Сбой сети...
    system_b.credit(amount)  # Не выполнено!
    # Деньги "потеряны"

# ХОРОШО: Saga pattern с компенсацией
def transfer_cross_system(amount):
    try:
        saga_id = create_saga()
        system_a.debit(amount, saga_id)
        system_b.credit(amount, saga_id)
        complete_saga(saga_id)
    except Exception:
        # Компенсирующая транзакция
        system_a.compensate(saga_id)
        raise
```

### 5. Некорректная обработка ошибок

```python
# ПЛОХО: Глотание ошибок
def update_data():
    try:
        db.execute("BEGIN")
        db.execute("UPDATE ...")
        db.execute("COMMIT")
    except:
        pass  # Транзакция осталась открытой!

# ХОРОШО: Правильная обработка
def update_data():
    try:
        db.execute("BEGIN")
        db.execute("UPDATE ...")
        db.execute("COMMIT")
    except Exception as e:
        db.execute("ROLLBACK")
        raise
```

---

## Краткое резюме

### Свойства ACID

| Свойство | Гарантия | Механизм |
|----------|----------|----------|
| **Atomicity** | Всё или ничего | WAL, Rollback |
| **Consistency** | Данные корректны | Constraints, Triggers |
| **Isolation** | Изоляция транзакций | Locks, MVCC |
| **Durability** | Данные не потеряются | WAL, fsync, репликация |

### Уровни изоляции (от слабого к сильному)

1. **READ UNCOMMITTED** — минимальная изоляция, возможно грязное чтение
2. **READ COMMITTED** — защита от грязного чтения (по умолчанию в PostgreSQL)
3. **REPEATABLE READ** — защита от неповторяющегося чтения
4. **SERIALIZABLE** — полная изоляция, как последовательное выполнение

### Ключевые правила

1. **Всегда используйте транзакции** для связанных операций
2. **Держите транзакции короткими** — не делайте внешних вызовов
3. **Выбирайте правильный уровень изоляции** — баланс между надёжностью и производительностью
4. **Обрабатывайте ошибки сериализации** — будьте готовы к повторным попыткам
5. **Используйте FOR UPDATE** при read-modify-write операциях
6. **Тестируйте конкурентный доступ** — проблемы проявляются под нагрузкой

### Когда ACID, когда BASE?

- **ACID**: Критичные данные, финансы, где ошибка недопустима
- **BASE**: Высокая нагрузка, распределённые системы, допустима временная несогласованность
