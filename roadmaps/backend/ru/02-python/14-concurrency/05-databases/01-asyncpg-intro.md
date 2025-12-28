# Введение в asyncpg

[prev: ./readme.md](./readme.md) | [next: ./02-connection.md](./02-connection.md)

---

## Что такое asyncpg?

**asyncpg** — это высокопроизводительная асинхронная библиотека для работы с PostgreSQL в Python. Она была создана специально для использования с `asyncio` и предоставляет полностью неблокирующий интерфейс для взаимодействия с базой данных.

### Ключевые особенности asyncpg

1. **Высокая производительность** — asyncpg написана на Cython и использует бинарный протокол PostgreSQL, что делает её одной из самых быстрых библиотек для работы с PostgreSQL
2. **Полная асинхронность** — все операции с базой данных выполняются асинхронно, не блокируя event loop
3. **Поддержка пулов соединений** — встроенная поддержка connection pooling для эффективного управления соединениями
4. **Поддержка prepared statements** — автоматическое кэширование подготовленных запросов
5. **Нативная поддержка типов PostgreSQL** — автоматическое преобразование типов данных между Python и PostgreSQL

## Установка asyncpg

```bash
# Установка через pip
pip install asyncpg

# Или с использованием poetry
poetry add asyncpg

# Или с использованием uv
uv add asyncpg
```

## Сравнение с синхронными драйверами

### Синхронный подход (psycopg2)

```python
import psycopg2
import time

def sync_query():
    conn = psycopg2.connect("postgresql://user:pass@localhost/db")
    cursor = conn.cursor()

    # Этот запрос блокирует весь поток выполнения
    cursor.execute("SELECT pg_sleep(1)")  # Симуляция долгого запроса
    cursor.execute("SELECT * FROM users")

    results = cursor.fetchall()
    conn.close()
    return results

# При выполнении 10 запросов последовательно:
# Время выполнения ~ 10 секунд
```

### Асинхронный подход (asyncpg)

```python
import asyncpg
import asyncio

async def async_query():
    conn = await asyncpg.connect("postgresql://user:pass@localhost/db")

    # Этот запрос НЕ блокирует event loop
    await conn.execute("SELECT pg_sleep(1)")
    results = await conn.fetch("SELECT * FROM users")

    await conn.close()
    return results

async def main():
    # При параллельном выполнении 10 запросов:
    # Время выполнения ~ 1 секунда
    tasks = [async_query() for _ in range(10)]
    results = await asyncio.gather(*tasks)
    return results

asyncio.run(main())
```

## Когда использовать asyncpg?

### Идеальные сценарии использования

1. **Web-приложения на асинхронных фреймворках**
   - FastAPI
   - Starlette
   - aiohttp
   - Quart

2. **Микросервисы с высокой нагрузкой**
   - Большое количество одновременных запросов к БД
   - Необходимость обработки множества соединений

3. **Real-time приложения**
   - WebSocket серверы
   - Чат-приложения
   - Системы уведомлений

### Когда asyncpg может быть избыточен

- Простые скрипты и CLI-утилиты
- Приложения с минимальной нагрузкой на БД
- Синхронные фреймворки (Django без async, Flask)

## Базовый пример использования

```python
import asyncio
import asyncpg


async def main():
    # Подключение к базе данных
    conn = await asyncpg.connect(
        user='postgres',
        password='password',
        database='mydb',
        host='localhost',
        port=5432
    )

    try:
        # Создание таблицы
        await conn.execute('''
            CREATE TABLE IF NOT EXISTS users (
                id SERIAL PRIMARY KEY,
                name VARCHAR(100) NOT NULL,
                email VARCHAR(100) UNIQUE NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')

        # Вставка данных
        await conn.execute('''
            INSERT INTO users (name, email) VALUES ($1, $2)
        ''', 'Иван Петров', 'ivan@example.com')

        # Получение данных
        rows = await conn.fetch('SELECT * FROM users')

        for row in rows:
            print(f"ID: {row['id']}, Name: {row['name']}, Email: {row['email']}")

    finally:
        # Закрытие соединения
        await conn.close()


if __name__ == '__main__':
    asyncio.run(main())
```

## Архитектура asyncpg

```
┌─────────────────────────────────────────────────────┐
│                   Python Application                │
├─────────────────────────────────────────────────────┤
│                     asyncpg API                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │ Connection  │  │    Pool     │  │ Transaction │  │
│  └─────────────┘  └─────────────┘  └─────────────┘  │
├─────────────────────────────────────────────────────┤
│              Cython Protocol Layer                  │
│         (Бинарный протокол PostgreSQL)              │
├─────────────────────────────────────────────────────┤
│                   asyncio Event Loop                │
├─────────────────────────────────────────────────────┤
│                    PostgreSQL Server                │
└─────────────────────────────────────────────────────┘
```

## Преимущества использования бинарного протокола

asyncpg использует бинарный протокол PostgreSQL вместо текстового, что даёт несколько преимуществ:

1. **Меньший размер передаваемых данных** — бинарный формат компактнее текстового
2. **Отсутствие накладных расходов на парсинг** — данные уже в нужном формате
3. **Более точное преобразование типов** — нет потери точности при конвертации

```python
import asyncpg
import asyncio
from decimal import Decimal
from datetime import datetime, date
from uuid import UUID


async def demonstrate_types():
    conn = await asyncpg.connect("postgresql://user:pass@localhost/db")

    # asyncpg автоматически преобразует типы PostgreSQL в Python типы
    row = await conn.fetchrow('''
        SELECT
            123::integer as int_val,
            123.456::numeric as decimal_val,
            NOW() as timestamp_val,
            CURRENT_DATE as date_val,
            '550e8400-e29b-41d4-a716-446655440000'::uuid as uuid_val,
            ARRAY[1, 2, 3] as array_val,
            '{"key": "value"}'::jsonb as json_val
    ''')

    print(f"Integer: {row['int_val']} (type: {type(row['int_val'])})")
    print(f"Decimal: {row['decimal_val']} (type: {type(row['decimal_val'])})")
    print(f"Timestamp: {row['timestamp_val']} (type: {type(row['timestamp_val'])})")
    print(f"Date: {row['date_val']} (type: {type(row['date_val'])})")
    print(f"UUID: {row['uuid_val']} (type: {type(row['uuid_val'])})")
    print(f"Array: {row['array_val']} (type: {type(row['array_val'])})")
    print(f"JSON: {row['json_val']} (type: {type(row['json_val'])})")

    await conn.close()


asyncio.run(demonstrate_types())
```

## Лучшие практики

1. **Всегда закрывайте соединения** — используйте `try/finally` или context managers
2. **Используйте пулы соединений** — для production-приложений
3. **Параметризуйте запросы** — используйте `$1, $2, ...` для защиты от SQL-инъекций
4. **Обрабатывайте исключения** — asyncpg выбрасывает специфичные исключения

```python
import asyncpg

async def safe_query():
    try:
        conn = await asyncpg.connect("postgresql://user:pass@localhost/db")
        try:
            result = await conn.fetch("SELECT * FROM users WHERE id = $1", 1)
            return result
        finally:
            await conn.close()
    except asyncpg.PostgresConnectionError as e:
        print(f"Ошибка подключения: {e}")
    except asyncpg.PostgresError as e:
        print(f"Ошибка PostgreSQL: {e}")
```

---

[prev: ./readme.md](./readme.md) | [next: ./02-connection.md](./02-connection.md)
