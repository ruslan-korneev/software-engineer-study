# Управление транзакциями

[prev: ./05-connection-pools.md](./05-connection-pools.md) | [next: ./07-async-generators.md](./07-async-generators.md)

---

## Что такое транзакция?

**Транзакция** — это группа операций с базой данных, которые выполняются как единое целое. Транзакции обеспечивают свойства ACID:

- **Atomicity (Атомарность)** — все операции выполняются полностью или не выполняются вовсе
- **Consistency (Согласованность)** — база данных переходит из одного согласованного состояния в другое
- **Isolation (Изолированность)** — параллельные транзакции не влияют друг на друга
- **Durability (Долговечность)** — после commit изменения сохраняются даже при сбое

## Базовая работа с транзакциями

### Context Manager (рекомендуемый способ)

```python
import asyncio
import asyncpg


async def basic_transaction():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Транзакция через context manager
        async with conn.transaction():
            # Все операции внутри блока — часть одной транзакции
            await conn.execute('''
                INSERT INTO accounts (user_id, balance) VALUES ($1, $2)
            ''', 1, 1000.00)

            await conn.execute('''
                INSERT INTO transactions (account_id, amount, type) VALUES ($1, $2, $3)
            ''', 1, 1000.00, 'deposit')

            # Если всё успешно — автоматический COMMIT

        print("Транзакция успешно завершена")

    finally:
        await conn.close()


asyncio.run(basic_transaction())
```

### Автоматический откат при ошибке

```python
import asyncio
import asyncpg


async def transaction_with_error():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        try:
            async with conn.transaction():
                # Первая операция — успешно
                await conn.execute('''
                    UPDATE accounts SET balance = balance - 100 WHERE id = $1
                ''', 1)

                # Вторая операция — ошибка (деление на ноль)
                await conn.execute('SELECT 1 / 0')

                # Эта строка не выполнится
                await conn.execute('''
                    UPDATE accounts SET balance = balance + 100 WHERE id = $1
                ''', 2)

        except asyncpg.PostgresError as e:
            print(f"Ошибка: {e}")
            print("Транзакция автоматически откачена (ROLLBACK)")

        # Проверяем, что баланс не изменился
        balance = await conn.fetchval(
            'SELECT balance FROM accounts WHERE id = $1', 1
        )
        print(f"Баланс аккаунта 1: {balance}")  # Остался прежним

    finally:
        await conn.close()


asyncio.run(transaction_with_error())
```

### Ручное управление транзакцией

```python
import asyncio
import asyncpg


async def manual_transaction():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Начало транзакции
        tr = conn.transaction()
        await tr.start()

        try:
            await conn.execute('''
                UPDATE accounts SET balance = balance - 100 WHERE id = $1
            ''', 1)

            # Проверка бизнес-логики
            balance = await conn.fetchval(
                'SELECT balance FROM accounts WHERE id = $1', 1
            )

            if balance < 0:
                # Ручной откат
                await tr.rollback()
                print("Недостаточно средств, откат транзакции")
            else:
                await conn.execute('''
                    UPDATE accounts SET balance = balance + 100 WHERE id = $1
                ''', 2)
                # Ручной коммит
                await tr.commit()
                print("Транзакция успешно завершена")

        except Exception as e:
            await tr.rollback()
            raise

    finally:
        await conn.close()


asyncio.run(manual_transaction())
```

## Уровни изоляции транзакций

PostgreSQL поддерживает 4 уровня изоляции:

| Уровень | Dirty Read | Non-repeatable Read | Phantom Read |
|---------|------------|---------------------|--------------|
| READ UNCOMMITTED | Возможно* | Возможно | Возможно |
| READ COMMITTED | Нет | Возможно | Возможно |
| REPEATABLE READ | Нет | Нет | Возможно* |
| SERIALIZABLE | Нет | Нет | Нет |

*В PostgreSQL READ UNCOMMITTED работает как READ COMMITTED, а REPEATABLE READ предотвращает phantom reads.

```python
import asyncio
import asyncpg


async def isolation_levels():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # READ COMMITTED (по умолчанию)
        async with conn.transaction(isolation='read_committed'):
            result = await conn.fetch('SELECT * FROM accounts')
            # Другие транзакции могут изменить данные между запросами

        # REPEATABLE READ
        async with conn.transaction(isolation='repeatable_read'):
            result1 = await conn.fetch('SELECT * FROM accounts')
            await asyncio.sleep(1)  # Другие транзакции работают
            result2 = await conn.fetch('SELECT * FROM accounts')
            # result1 и result2 будут одинаковыми

        # SERIALIZABLE (самый строгий)
        async with conn.transaction(isolation='serializable'):
            # Транзакции выполняются как будто последовательно
            await conn.execute('UPDATE accounts SET balance = balance + 10 WHERE id = 1')

    finally:
        await conn.close()


asyncio.run(isolation_levels())
```

### Обработка конфликтов сериализации

```python
import asyncio
import asyncpg


async def serializable_with_retry(max_retries: int = 3):
    """Транзакция SERIALIZABLE с повторными попытками."""
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        for attempt in range(max_retries):
            try:
                async with conn.transaction(isolation='serializable'):
                    # Операции, которые могут конфликтовать
                    balance = await conn.fetchval(
                        'SELECT balance FROM accounts WHERE id = $1', 1
                    )

                    if balance >= 100:
                        await conn.execute('''
                            UPDATE accounts SET balance = balance - 100 WHERE id = $1
                        ''', 1)
                        await conn.execute('''
                            UPDATE accounts SET balance = balance + 100 WHERE id = $1
                        ''', 2)

                print(f"Успешно с попытки {attempt + 1}")
                return True

            except asyncpg.SerializationError:
                print(f"Конфликт сериализации, попытка {attempt + 1}/{max_retries}")
                if attempt < max_retries - 1:
                    await asyncio.sleep(0.1 * (attempt + 1))  # Экспоненциальная задержка
                continue

        print("Превышено количество попыток")
        return False

    finally:
        await conn.close()


asyncio.run(serializable_with_retry())
```

## Режимы доступа транзакций

```python
import asyncio
import asyncpg


async def access_modes():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # READ ONLY транзакция (только чтение)
        async with conn.transaction(readonly=True):
            users = await conn.fetch('SELECT * FROM users')

            # Это вызовет ошибку:
            # await conn.execute('INSERT INTO users (name) VALUES ($1)', 'test')

        # DEFERRABLE транзакция (для SERIALIZABLE READ ONLY)
        async with conn.transaction(
            isolation='serializable',
            readonly=True,
            deferrable=True
        ):
            # Отложенная транзакция может ждать момента, когда не будет конфликтов
            # Полезно для длительных отчётов
            report_data = await conn.fetch('''
                SELECT date_trunc('day', created_at) as day, COUNT(*)
                FROM orders
                GROUP BY day
                ORDER BY day
            ''')

    finally:
        await conn.close()


asyncio.run(access_modes())
```

## Вложенные транзакции (Savepoints)

```python
import asyncio
import asyncpg


async def nested_transactions():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        async with conn.transaction():  # Основная транзакция
            await conn.execute('''
                INSERT INTO orders (user_id, status) VALUES ($1, $2)
            ''', 1, 'pending')

            order_id = await conn.fetchval('SELECT lastval()')

            try:
                # Вложенная транзакция (savepoint)
                async with conn.transaction():
                    await conn.execute('''
                        INSERT INTO order_items (order_id, product_id, quantity)
                        VALUES ($1, $2, $3)
                    ''', order_id, 1, 5)

                    # Проверка наличия товара
                    available = await conn.fetchval(
                        'SELECT quantity FROM products WHERE id = $1', 1
                    )

                    if available < 5:
                        raise ValueError("Недостаточно товара на складе")

            except ValueError as e:
                print(f"Ошибка: {e}")
                # Вложенная транзакция откатывается
                # Основная транзакция продолжается

            # Можно продолжить работу в основной транзакции
            await conn.execute('''
                UPDATE orders SET status = $1 WHERE id = $2
            ''', 'partial', order_id)

        # Здесь основная транзакция коммитится

    finally:
        await conn.close()


asyncio.run(nested_transactions())
```

## Транзакции с пулом соединений

```python
import asyncio
import asyncpg


async def pool_transactions():
    pool = await asyncpg.create_pool(
        'postgresql://postgres:password@localhost/mydb',
        min_size=5,
        max_size=20
    )

    try:
        # Способ 1: через acquire
        async with pool.acquire() as conn:
            async with conn.transaction():
                await conn.execute('UPDATE users SET is_active = $1 WHERE id = $2', True, 1)

        # Способ 2: через pool.transaction() напрямую (НЕ РАБОТАЕТ!)
        # У пула нет метода transaction(), нужно сначала получить соединение

        # Правильный способ для множественных операций
        async def transfer_money(from_id: int, to_id: int, amount: float):
            async with pool.acquire() as conn:
                async with conn.transaction():
                    # Списание
                    await conn.execute('''
                        UPDATE accounts SET balance = balance - $1 WHERE id = $2
                    ''', amount, from_id)

                    # Проверка
                    balance = await conn.fetchval(
                        'SELECT balance FROM accounts WHERE id = $1', from_id
                    )
                    if balance < 0:
                        raise ValueError("Недостаточно средств")

                    # Зачисление
                    await conn.execute('''
                        UPDATE accounts SET balance = balance + $1 WHERE id = $2
                    ''', amount, to_id)

        await transfer_money(1, 2, 100.0)
        print("Перевод выполнен успешно")

    finally:
        await pool.close()


asyncio.run(pool_transactions())
```

## Блокировки в транзакциях

```python
import asyncio
import asyncpg


async def locking_examples():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        async with conn.transaction():
            # FOR UPDATE — блокировка строк для обновления
            user = await conn.fetchrow('''
                SELECT * FROM users WHERE id = $1 FOR UPDATE
            ''', 1)
            # Другие транзакции не смогут изменить эту строку

            await conn.execute('''
                UPDATE users SET login_count = login_count + 1 WHERE id = $1
            ''', 1)

        async with conn.transaction():
            # FOR UPDATE NOWAIT — без ожидания, ошибка если заблокировано
            try:
                user = await conn.fetchrow('''
                    SELECT * FROM users WHERE id = $1 FOR UPDATE NOWAIT
                ''', 1)
            except asyncpg.LockNotAvailableError:
                print("Строка заблокирована другой транзакцией")

        async with conn.transaction():
            # FOR UPDATE SKIP LOCKED — пропуск заблокированных строк
            # Полезно для обработки очередей
            tasks = await conn.fetch('''
                SELECT * FROM task_queue
                WHERE status = 'pending'
                FOR UPDATE SKIP LOCKED
                LIMIT 10
            ''')

            for task in tasks:
                await process_task(task)
                await conn.execute('''
                    UPDATE task_queue SET status = 'completed' WHERE id = $1
                ''', task['id'])

        async with conn.transaction():
            # FOR SHARE — разделяемая блокировка (для чтения)
            products = await conn.fetch('''
                SELECT * FROM products WHERE category_id = $1 FOR SHARE
            ''', 1)
            # Другие транзакции могут читать, но не изменять

    finally:
        await conn.close()


async def process_task(task):
    print(f"Processing task: {task['id']}")


asyncio.run(locking_examples())
```

## Практический пример: банковский перевод

```python
import asyncio
import asyncpg
from decimal import Decimal
from dataclasses import dataclass
from typing import Optional


@dataclass
class TransferResult:
    success: bool
    message: str
    from_balance: Optional[Decimal] = None
    to_balance: Optional[Decimal] = None


class BankService:
    def __init__(self, pool: asyncpg.Pool):
        self.pool = pool

    async def transfer(
        self,
        from_account: int,
        to_account: int,
        amount: Decimal
    ) -> TransferResult:
        """Перевод денег между счетами."""

        if amount <= 0:
            return TransferResult(False, "Сумма должна быть положительной")

        async with self.pool.acquire() as conn:
            try:
                async with conn.transaction(isolation='serializable'):
                    # Блокировка счетов в определённом порядке для избежания deadlock
                    accounts = sorted([from_account, to_account])

                    rows = await conn.fetch('''
                        SELECT id, balance FROM accounts
                        WHERE id = ANY($1::int[])
                        FOR UPDATE
                        ORDER BY id
                    ''', accounts)

                    if len(rows) != 2:
                        return TransferResult(False, "Один или оба счёта не найдены")

                    balances = {row['id']: row['balance'] for row in rows}

                    # Проверка достаточности средств
                    if balances[from_account] < amount:
                        return TransferResult(
                            False,
                            f"Недостаточно средств. Баланс: {balances[from_account]}"
                        )

                    # Выполнение перевода
                    await conn.execute('''
                        UPDATE accounts SET balance = balance - $1 WHERE id = $2
                    ''', amount, from_account)

                    await conn.execute('''
                        UPDATE accounts SET balance = balance + $1 WHERE id = $2
                    ''', amount, to_account)

                    # Запись в историю
                    await conn.execute('''
                        INSERT INTO transfers (from_account, to_account, amount)
                        VALUES ($1, $2, $3)
                    ''', from_account, to_account, amount)

                    # Получение новых балансов
                    new_balances = await conn.fetch('''
                        SELECT id, balance FROM accounts WHERE id = ANY($1::int[])
                    ''', accounts)

                    balances = {row['id']: row['balance'] for row in new_balances}

                    return TransferResult(
                        True,
                        "Перевод выполнен успешно",
                        balances[from_account],
                        balances[to_account]
                    )

            except asyncpg.SerializationError:
                return TransferResult(False, "Конфликт транзакций, попробуйте снова")

            except asyncpg.PostgresError as e:
                return TransferResult(False, f"Ошибка базы данных: {e}")


async def main():
    pool = await asyncpg.create_pool(
        'postgresql://postgres:password@localhost/mydb'
    )

    try:
        bank = BankService(pool)

        result = await bank.transfer(1, 2, Decimal('100.00'))

        if result.success:
            print(f"Успешно! Баланс отправителя: {result.from_balance}, "
                  f"получателя: {result.to_balance}")
        else:
            print(f"Ошибка: {result.message}")

    finally:
        await pool.close()


asyncio.run(main())
```

## Лучшие практики

1. **Используйте context manager** для автоматического commit/rollback
2. **Выбирайте правильный уровень изоляции** — не используйте SERIALIZABLE без необходимости
3. **Держите транзакции короткими** — долгие транзакции блокируют ресурсы
4. **Обрабатывайте ошибки сериализации** с повторными попытками
5. **Сортируйте блокировки** для предотвращения deadlock
6. **Используйте savepoints** для частичного отката

---

[prev: ./05-connection-pools.md](./05-connection-pools.md) | [next: ./07-async-generators.md](./07-async-generators.md)
