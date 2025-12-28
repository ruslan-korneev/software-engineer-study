# Выполнение запросов

[prev: ./03-schema.md](./03-schema.md) | [next: ./05-connection-pools.md](./05-connection-pools.md)

---

## Методы выполнения запросов в asyncpg

asyncpg предоставляет несколько методов для выполнения SQL-запросов. Каждый метод оптимизирован для определённого сценария использования.

## Основные методы запросов

### execute() - Выполнение без возврата данных

```python
import asyncio
import asyncpg


async def execute_examples():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Вставка одной записи
        await conn.execute('''
            INSERT INTO users (username, email)
            VALUES ($1, $2)
        ''', 'ivan', 'ivan@example.com')

        # Обновление записей
        result = await conn.execute('''
            UPDATE users SET is_active = $1
            WHERE created_at < $2
        ''', False, '2024-01-01')

        # execute() возвращает строку с информацией о выполнении
        print(f"Результат: {result}")  # "UPDATE 5" - обновлено 5 записей

        # Удаление записей
        result = await conn.execute('''
            DELETE FROM users WHERE is_active = FALSE
        ''')
        print(f"Удалено: {result}")  # "DELETE 5"

    finally:
        await conn.close()


asyncio.run(execute_examples())
```

### fetch() - Получение всех результатов

```python
import asyncio
import asyncpg


async def fetch_examples():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # fetch() возвращает список Record объектов
        rows = await conn.fetch('SELECT * FROM users')

        for row in rows:
            # Доступ по индексу
            print(f"ID: {row[0]}")

            # Доступ по имени колонки
            print(f"Username: {row['username']}")

            # Получение всех значений
            print(f"Values: {tuple(row)}")

            # Получение словаря
            print(f"Dict: {dict(row)}")

            # Получение списка ключей
            print(f"Keys: {row.keys()}")

    finally:
        await conn.close()


asyncio.run(fetch_examples())
```

### fetchrow() - Получение одной записи

```python
import asyncio
import asyncpg


async def fetchrow_examples():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Получение одной записи
        user = await conn.fetchrow('''
            SELECT * FROM users WHERE id = $1
        ''', 1)

        if user:
            print(f"Найден пользователь: {user['username']}")
        else:
            print("Пользователь не найден")

        # Получение первой записи из множества
        latest_user = await conn.fetchrow('''
            SELECT * FROM users
            ORDER BY created_at DESC
            LIMIT 1
        ''')

        if latest_user:
            print(f"Последний пользователь: {latest_user['username']}")

    finally:
        await conn.close()


asyncio.run(fetchrow_examples())
```

### fetchval() - Получение одного значения

```python
import asyncio
import asyncpg


async def fetchval_examples():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Получение скалярного значения
        count = await conn.fetchval('SELECT COUNT(*) FROM users')
        print(f"Количество пользователей: {count}")

        # Получение значения конкретной колонки (по умолчанию первая, индекс 0)
        username = await conn.fetchval(
            'SELECT username, email FROM users WHERE id = $1',
            1
        )
        print(f"Username: {username}")

        # Получение значения второй колонки (индекс 1)
        email = await conn.fetchval(
            'SELECT username, email FROM users WHERE id = $1',
            1,
            column=1  # индекс колонки
        )
        print(f"Email: {email}")

        # Проверка существования
        exists = await conn.fetchval('''
            SELECT EXISTS(SELECT 1 FROM users WHERE email = $1)
        ''', 'ivan@example.com')
        print(f"Email существует: {exists}")

    finally:
        await conn.close()


asyncio.run(fetchval_examples())
```

## Параметризованные запросы

### Позиционные параметры

```python
import asyncio
import asyncpg
from datetime import datetime, date
from decimal import Decimal
from uuid import UUID


async def parameterized_queries():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Простые типы
        await conn.execute('''
            INSERT INTO products (name, price, quantity)
            VALUES ($1, $2, $3)
        ''', 'Ноутбук', Decimal('999.99'), 10)

        # Множественные параметры
        await conn.execute('''
            UPDATE products
            SET price = $1, quantity = $2, updated_at = $3
            WHERE id = $4
        ''', Decimal('899.99'), 15, datetime.now(), 1)

        # Параметры с NULL
        await conn.execute('''
            INSERT INTO products (name, price, description)
            VALUES ($1, $2, $3)
        ''', 'Товар', Decimal('100.00'), None)

    finally:
        await conn.close()


asyncio.run(parameterized_queries())
```

### Работа с массивами

```python
import asyncio
import asyncpg


async def array_queries():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Передача массива в запрос
        user_ids = [1, 2, 3, 4, 5]
        users = await conn.fetch('''
            SELECT * FROM users WHERE id = ANY($1::int[])
        ''', user_ids)

        print(f"Найдено пользователей: {len(users)}")

        # Использование unnest для множественной вставки
        names = ['Товар 1', 'Товар 2', 'Товар 3']
        prices = [Decimal('100'), Decimal('200'), Decimal('300')]

        await conn.execute('''
            INSERT INTO products (name, price)
            SELECT * FROM unnest($1::text[], $2::numeric[])
        ''', names, prices)

        # Получение массивов из PostgreSQL
        row = await conn.fetchrow('''
            SELECT ARRAY_AGG(username) as usernames
            FROM users
            WHERE is_active = TRUE
        ''')
        print(f"Активные пользователи: {row['usernames']}")

    finally:
        await conn.close()


asyncio.run(array_queries())
```

### Работа с JSON/JSONB

```python
import asyncio
import asyncpg
import json


async def json_queries():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Создание таблицы с JSONB
        await conn.execute('''
            CREATE TABLE IF NOT EXISTS events (
                id SERIAL PRIMARY KEY,
                event_type VARCHAR(50),
                data JSONB NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')

        # Вставка JSON данных (Python dict автоматически преобразуется)
        event_data = {
            'user_id': 123,
            'action': 'login',
            'ip_address': '192.168.1.1',
            'user_agent': 'Mozilla/5.0',
            'metadata': {
                'country': 'Russia',
                'city': 'Moscow'
            }
        }

        await conn.execute('''
            INSERT INTO events (event_type, data) VALUES ($1, $2)
        ''', 'user_action', json.dumps(event_data))

        # Или напрямую (asyncpg умеет работать с dict)
        await conn.execute('''
            INSERT INTO events (event_type, data) VALUES ($1, $2::jsonb)
        ''', 'user_action', json.dumps(event_data))

        # Запросы к JSONB полям
        events = await conn.fetch('''
            SELECT * FROM events
            WHERE data->>'user_id' = $1
        ''', '123')

        # Фильтрация по вложенным полям
        events = await conn.fetch('''
            SELECT * FROM events
            WHERE data->'metadata'->>'country' = $1
        ''', 'Russia')

        # JSONB операторы
        events = await conn.fetch('''
            SELECT * FROM events
            WHERE data @> $1::jsonb
        ''', json.dumps({'action': 'login'}))

        for event in events:
            print(f"Event: {event['data']}")  # data возвращается как Python dict

    finally:
        await conn.close()


asyncio.run(json_queries())
```

## Подготовленные запросы (Prepared Statements)

```python
import asyncio
import asyncpg


async def prepared_statements():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Создание prepared statement
        stmt = await conn.prepare('''
            SELECT * FROM users WHERE is_active = $1 AND created_at > $2
        ''')

        # Информация о prepared statement
        print(f"Параметры: {stmt.get_parameters()}")
        print(f"Атрибуты: {stmt.get_attributes()}")

        # Многократное использование prepared statement
        from datetime import datetime, timedelta

        active_users = await stmt.fetch(True, datetime.now() - timedelta(days=30))
        print(f"Активных пользователей за 30 дней: {len(active_users)}")

        active_users = await stmt.fetch(True, datetime.now() - timedelta(days=7))
        print(f"Активных пользователей за 7 дней: {len(active_users)}")

        # Prepared statement для вставки
        insert_stmt = await conn.prepare('''
            INSERT INTO users (username, email) VALUES ($1, $2) RETURNING id
        ''')

        # Вставка нескольких записей с переиспользованием
        for i in range(5):
            user_id = await insert_stmt.fetchval(f'user_{i}', f'user_{i}@example.com')
            print(f"Создан пользователь с ID: {user_id}")

    finally:
        await conn.close()


asyncio.run(prepared_statements())
```

## Пакетная вставка (Batch Insert)

```python
import asyncio
import asyncpg
from typing import List, Tuple


async def batch_insert():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Подготовка данных
        users_data = [
            ('user1', 'user1@example.com', True),
            ('user2', 'user2@example.com', True),
            ('user3', 'user3@example.com', False),
            ('user4', 'user4@example.com', True),
            ('user5', 'user5@example.com', True),
        ]

        # Метод 1: executemany (не рекомендуется для больших объёмов)
        await conn.executemany('''
            INSERT INTO users (username, email, is_active)
            VALUES ($1, $2, $3)
        ''', users_data)

        # Метод 2: copy_records_to_table (очень быстрый)
        # Требует точного соответствия типов
        products_data = [
            ('Товар 1', 100.0, 10),
            ('Товар 2', 200.0, 20),
            ('Товар 3', 300.0, 30),
        ]

        await conn.copy_records_to_table(
            'products',
            records=products_data,
            columns=['name', 'price', 'quantity']
        )

        print("Пакетная вставка выполнена")

    finally:
        await conn.close()


async def efficient_batch_insert():
    """Эффективная пакетная вставка для больших объёмов данных."""
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Генерация большого объёма данных
        import random

        def generate_users(count: int):
            for i in range(count):
                yield (
                    f'user_{i}',
                    f'user_{i}@example.com',
                    random.choice([True, False])
                )

        # copy_records_to_table работает с итераторами
        # Это очень эффективно по памяти
        await conn.copy_records_to_table(
            'users',
            records=generate_users(10000),
            columns=['username', 'email', 'is_active']
        )

        count = await conn.fetchval('SELECT COUNT(*) FROM users')
        print(f"Вставлено записей: {count}")

    finally:
        await conn.close()


asyncio.run(batch_insert())
```

## COPY операции

```python
import asyncio
import asyncpg
import io


async def copy_operations():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # COPY FROM - импорт из CSV-подобного формата
        csv_data = b'''username,email,is_active
john,john@example.com,true
jane,jane@example.com,true
bob,bob@example.com,false
'''

        await conn.copy_to_table(
            'users',
            source=io.BytesIO(csv_data),
            format='csv',
            header=True,
            columns=['username', 'email', 'is_active']
        )

        # COPY TO - экспорт в файл или поток
        output = io.BytesIO()
        await conn.copy_from_query(
            'SELECT username, email FROM users WHERE is_active = TRUE',
            output=output,
            format='csv',
            header=True
        )

        exported_data = output.getvalue().decode('utf-8')
        print(f"Экспортированные данные:\n{exported_data}")

    finally:
        await conn.close()


asyncio.run(copy_operations())
```

## Курсоры

```python
import asyncio
import asyncpg


async def cursor_usage():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Использование курсора для больших результатов
        async with conn.transaction():
            # Создание курсора
            async for record in conn.cursor(
                'SELECT * FROM users ORDER BY id',
                prefetch=100  # Количество записей для предварительной загрузки
            ):
                print(f"User: {record['username']}")
                # Обработка записи

        # Курсор с параметрами
        async with conn.transaction():
            async for record in conn.cursor(
                'SELECT * FROM users WHERE is_active = $1',
                True
            ):
                process_user(record)

    finally:
        await conn.close()


def process_user(record):
    print(f"Processing: {record['username']}")


asyncio.run(cursor_usage())
```

## Обработка ошибок запросов

```python
import asyncio
import asyncpg


async def error_handling():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Ошибка синтаксиса SQL
        try:
            await conn.execute('SELEC * FROM users')  # Опечатка
        except asyncpg.PostgresSyntaxError as e:
            print(f"Синтаксическая ошибка: {e}")

        # Нарушение уникальности
        try:
            await conn.execute('''
                INSERT INTO users (username, email) VALUES ($1, $2)
            ''', 'existing_user', 'existing@example.com')
        except asyncpg.UniqueViolationError as e:
            print(f"Нарушение уникальности: {e}")
            print(f"Имя ограничения: {e.constraint_name}")

        # Нарушение внешнего ключа
        try:
            await conn.execute('''
                INSERT INTO orders (user_id, total_amount) VALUES ($1, $2)
            ''', 99999, 100.0)  # Несуществующий user_id
        except asyncpg.ForeignKeyViolationError as e:
            print(f"Нарушение внешнего ключа: {e}")

        # Нарушение CHECK constraint
        try:
            await conn.execute('''
                INSERT INTO products (name, price) VALUES ($1, $2)
            ''', 'Товар', -100)  # Отрицательная цена
        except asyncpg.CheckViolationError as e:
            print(f"Нарушение CHECK: {e}")

        # Таймаут запроса
        try:
            await asyncio.wait_for(
                conn.execute('SELECT pg_sleep(60)'),
                timeout=5.0
            )
        except asyncio.TimeoutError:
            print("Превышено время выполнения запроса")

    finally:
        await conn.close()


asyncio.run(error_handling())
```

## Лучшие практики

1. **Всегда используйте параметризованные запросы** для защиты от SQL-инъекций
2. **Используйте правильный метод** для каждого сценария (fetch/fetchrow/fetchval)
3. **Применяйте prepared statements** для часто выполняемых запросов
4. **Используйте copy_records_to_table** для пакетной вставки больших объёмов
5. **Обрабатывайте специфичные исключения** для понятных сообщений об ошибках
6. **Ограничивайте размер результатов** с помощью LIMIT для больших таблиц

```python
# Пример хорошей практики - репозиторий с типизацией
from dataclasses import dataclass
from typing import Optional, List
import asyncpg


@dataclass
class User:
    id: int
    username: str
    email: str
    is_active: bool


class UserRepository:
    def __init__(self, conn: asyncpg.Connection):
        self.conn = conn

    async def get_by_id(self, user_id: int) -> Optional[User]:
        row = await self.conn.fetchrow(
            'SELECT id, username, email, is_active FROM users WHERE id = $1',
            user_id
        )
        return User(**dict(row)) if row else None

    async def get_active(self) -> List[User]:
        rows = await self.conn.fetch(
            'SELECT id, username, email, is_active FROM users WHERE is_active = TRUE'
        )
        return [User(**dict(row)) for row in rows]

    async def create(self, username: str, email: str) -> User:
        row = await self.conn.fetchrow('''
            INSERT INTO users (username, email)
            VALUES ($1, $2)
            RETURNING id, username, email, is_active
        ''', username, email)
        return User(**dict(row))
```

---

[prev: ./03-schema.md](./03-schema.md) | [next: ./05-connection-pools.md](./05-connection-pools.md)
