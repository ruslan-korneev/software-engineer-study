# Пулы подключений

[prev: ./04-queries.md](./04-queries.md) | [next: ./06-transactions.md](./06-transactions.md)

---

## Что такое пул соединений?

**Пул соединений (Connection Pool)** — это кэш соединений с базой данных, который позволяет переиспользовать уже установленные соединения вместо создания новых для каждого запроса.

### Зачем нужен пул соединений?

```
Без пула соединений:
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Запрос 1   │ ──► │  Connect    │ ──► │  PostgreSQL │
└─────────────┘     │  Execute    │     └─────────────┘
                    │  Close      │
┌─────────────┐     ├─────────────┤     ┌─────────────┐
│  Запрос 2   │ ──► │  Connect    │ ──► │  PostgreSQL │
└─────────────┘     │  Execute    │     └─────────────┘
                    │  Close      │
                    └─────────────┘
Накладные расходы: ~50-100мс на каждый Connect

С пулом соединений:
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Запрос 1   │ ──► │             │     │             │
└─────────────┘     │    Pool     │ ══► │  PostgreSQL │
┌─────────────┐     │  (готовые   │     │  (5 conn)   │
│  Запрос 2   │ ──► │ соединения) │     │             │
└─────────────┘     │             │     │             │
┌─────────────┐     │             │     │             │
│  Запрос 3   │ ──► │             │     │             │
└─────────────┘     └─────────────┘     └─────────────┘
Накладные расходы: ~0.1мс на получение соединения из пула
```

## Создание пула соединений

### Базовое создание пула

```python
import asyncio
import asyncpg


async def basic_pool():
    # Создание пула соединений
    pool = await asyncpg.create_pool(
        'postgresql://postgres:password@localhost/mydb',
        min_size=5,      # Минимальное количество соединений
        max_size=20,     # Максимальное количество соединений
    )

    try:
        # Использование пула
        async with pool.acquire() as conn:
            result = await conn.fetch('SELECT * FROM users')
            print(f"Найдено пользователей: {len(result)}")

    finally:
        # Закрытие пула
        await pool.close()


asyncio.run(basic_pool())
```

### Полная конфигурация пула

```python
import asyncio
import asyncpg
import ssl


async def configured_pool():
    # SSL контекст для защищённого соединения
    ssl_context = ssl.create_default_context()
    ssl_context.check_hostname = False
    ssl_context.verify_mode = ssl.CERT_NONE

    pool = await asyncpg.create_pool(
        # Параметры подключения
        user='postgres',
        password='password',
        database='mydb',
        host='localhost',
        port=5432,

        # Параметры пула
        min_size=5,                    # Минимум соединений (всегда поддерживается)
        max_size=20,                   # Максимум соединений
        max_queries=50000,             # Макс. запросов на соединение до пересоздания
        max_inactive_connection_lifetime=300.0,  # Время жизни неактивного соединения (сек)

        # Таймауты
        timeout=30.0,                  # Таймаут получения соединения из пула
        command_timeout=60.0,          # Таймаут выполнения команд

        # SSL
        ssl=ssl_context,

        # Настройки соединений
        statement_cache_size=100,

        # Серверные настройки
        server_settings={
            'application_name': 'my_app',
            'timezone': 'Europe/Moscow',
        },

        # Callback при создании соединения
        init=init_connection,

        # Callback при возврате соединения в пул
        setup=setup_connection,
    )

    return pool


async def init_connection(conn):
    """Вызывается один раз при создании соединения."""
    # Установка типов данных
    await conn.set_type_codec(
        'json',
        encoder=lambda x: x,
        decoder=lambda x: x,
        schema='pg_catalog'
    )
    print(f"Инициализировано соединение: {conn.get_server_pid()}")


async def setup_connection(conn):
    """Вызывается каждый раз при получении соединения из пула."""
    # Сброс настроек сессии
    await conn.execute("SET statement_timeout = '30s'")
```

## Работа с пулом соединений

### Получение соединения через context manager

```python
import asyncio
import asyncpg


async def using_pool():
    pool = await asyncpg.create_pool(
        'postgresql://postgres:password@localhost/mydb'
    )

    try:
        # Рекомендуемый способ - context manager
        async with pool.acquire() as conn:
            # Соединение автоматически вернётся в пул после выхода
            users = await conn.fetch('SELECT * FROM users')

            # Можно использовать несколько запросов
            count = await conn.fetchval('SELECT COUNT(*) FROM users')

        # Здесь соединение уже вернулось в пул

    finally:
        await pool.close()


asyncio.run(using_pool())
```

### Прямые методы пула

```python
import asyncio
import asyncpg


async def direct_pool_methods():
    pool = await asyncpg.create_pool(
        'postgresql://postgres:password@localhost/mydb'
    )

    try:
        # Пул предоставляет те же методы, что и Connection
        # Соединение автоматически берётся из пула и возвращается

        # fetch
        users = await pool.fetch('SELECT * FROM users LIMIT 10')

        # fetchrow
        user = await pool.fetchrow('SELECT * FROM users WHERE id = $1', 1)

        # fetchval
        count = await pool.fetchval('SELECT COUNT(*) FROM users')

        # execute
        await pool.execute(
            'UPDATE users SET is_active = $1 WHERE id = $2',
            True, 1
        )

        print(f"Пользователей: {count}")

    finally:
        await pool.close()


asyncio.run(direct_pool_methods())
```

### Параллельные запросы через пул

```python
import asyncio
import asyncpg


async def parallel_queries():
    pool = await asyncpg.create_pool(
        'postgresql://postgres:password@localhost/mydb',
        min_size=5,
        max_size=10
    )

    try:
        # Параллельное выполнение запросов
        # Каждый запрос получит своё соединение из пула
        results = await asyncio.gather(
            pool.fetchval('SELECT COUNT(*) FROM users'),
            pool.fetchval('SELECT COUNT(*) FROM products'),
            pool.fetchval('SELECT COUNT(*) FROM orders'),
            pool.fetch('SELECT * FROM categories'),
        )

        users_count, products_count, orders_count, categories = results

        print(f"Пользователей: {users_count}")
        print(f"Товаров: {products_count}")
        print(f"Заказов: {orders_count}")
        print(f"Категорий: {len(categories)}")

    finally:
        await pool.close()


asyncio.run(parallel_queries())
```

## Мониторинг пула

```python
import asyncio
import asyncpg


async def monitor_pool():
    pool = await asyncpg.create_pool(
        'postgresql://postgres:password@localhost/mydb',
        min_size=2,
        max_size=10
    )

    try:
        # Информация о пуле
        print(f"Размер пула: {pool.get_size()}")
        print(f"Свободных соединений: {pool.get_idle_size()}")
        print(f"Минимальный размер: {pool.get_min_size()}")
        print(f"Максимальный размер: {pool.get_max_size()}")

        # Симуляция нагрузки
        async def worker(worker_id: int):
            async with pool.acquire() as conn:
                print(f"Worker {worker_id}: получил соединение, "
                      f"свободно: {pool.get_idle_size()}")
                await asyncio.sleep(1)  # Симуляция работы
            print(f"Worker {worker_id}: вернул соединение")

        # Запуск нескольких воркеров
        await asyncio.gather(*[worker(i) for i in range(5)])

        print(f"\nПосле завершения:")
        print(f"Размер пула: {pool.get_size()}")
        print(f"Свободных соединений: {pool.get_idle_size()}")

    finally:
        await pool.close()


asyncio.run(monitor_pool())
```

## Обработка исчерпания пула

```python
import asyncio
import asyncpg


async def pool_exhaustion():
    # Маленький пул для демонстрации
    pool = await asyncpg.create_pool(
        'postgresql://postgres:password@localhost/mydb',
        min_size=1,
        max_size=2,
        timeout=5.0  # Таймаут ожидания соединения
    )

    try:
        async def long_running_task(task_id: int):
            try:
                async with pool.acquire() as conn:
                    print(f"Task {task_id}: начало работы")
                    await conn.execute('SELECT pg_sleep(3)')
                    print(f"Task {task_id}: завершено")
            except asyncio.TimeoutError:
                print(f"Task {task_id}: не удалось получить соединение (таймаут)")

        # Запуск 5 задач при пуле размером 2
        await asyncio.gather(*[long_running_task(i) for i in range(5)])

    finally:
        await pool.close()


asyncio.run(pool_exhaustion())
```

## Глобальный пул в приложении

```python
import asyncio
import asyncpg
from contextlib import asynccontextmanager
from typing import Optional


class Database:
    """Класс для управления пулом соединений."""

    _pool: Optional[asyncpg.Pool] = None

    @classmethod
    async def connect(cls, dsn: str, **kwargs):
        """Создание пула соединений."""
        if cls._pool is not None:
            raise RuntimeError("Пул уже инициализирован")

        cls._pool = await asyncpg.create_pool(dsn, **kwargs)
        print("Пул соединений создан")

    @classmethod
    async def disconnect(cls):
        """Закрытие пула соединений."""
        if cls._pool is not None:
            await cls._pool.close()
            cls._pool = None
            print("Пул соединений закрыт")

    @classmethod
    def get_pool(cls) -> asyncpg.Pool:
        """Получение пула."""
        if cls._pool is None:
            raise RuntimeError("Пул не инициализирован")
        return cls._pool

    @classmethod
    @asynccontextmanager
    async def connection(cls):
        """Context manager для получения соединения."""
        async with cls.get_pool().acquire() as conn:
            yield conn

    @classmethod
    async def execute(cls, query: str, *args):
        """Выполнение запроса через пул."""
        return await cls.get_pool().execute(query, *args)

    @classmethod
    async def fetch(cls, query: str, *args):
        """Получение данных через пул."""
        return await cls.get_pool().fetch(query, *args)

    @classmethod
    async def fetchrow(cls, query: str, *args):
        """Получение одной записи через пул."""
        return await cls.get_pool().fetchrow(query, *args)

    @classmethod
    async def fetchval(cls, query: str, *args, column: int = 0):
        """Получение одного значения через пул."""
        return await cls.get_pool().fetchval(query, *args, column=column)


# Использование
async def main():
    # Инициализация при старте приложения
    await Database.connect(
        'postgresql://postgres:password@localhost/mydb',
        min_size=5,
        max_size=20
    )

    try:
        # Использование в любом месте приложения
        users = await Database.fetch('SELECT * FROM users')
        print(f"Пользователей: {len(users)}")

        # Или через context manager
        async with Database.connection() as conn:
            await conn.execute('UPDATE users SET is_active = TRUE WHERE id = $1', 1)

    finally:
        # Закрытие при завершении приложения
        await Database.disconnect()


asyncio.run(main())
```

## Интеграция с FastAPI

```python
import asyncpg
from fastapi import FastAPI, Depends
from contextlib import asynccontextmanager
from typing import AsyncGenerator


# Глобальная переменная для пула
pool: asyncpg.Pool = None


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Lifecycle manager для FastAPI."""
    global pool

    # Startup
    pool = await asyncpg.create_pool(
        'postgresql://postgres:password@localhost/mydb',
        min_size=5,
        max_size=20,
        command_timeout=60
    )
    print("Database pool created")

    yield

    # Shutdown
    await pool.close()
    print("Database pool closed")


app = FastAPI(lifespan=lifespan)


async def get_db() -> AsyncGenerator[asyncpg.Connection, None]:
    """Dependency для получения соединения."""
    async with pool.acquire() as connection:
        yield connection


@app.get("/users")
async def get_users(db: asyncpg.Connection = Depends(get_db)):
    """Эндпоинт для получения пользователей."""
    users = await db.fetch('SELECT id, username, email FROM users')
    return [dict(user) for user in users]


@app.get("/users/{user_id}")
async def get_user(user_id: int, db: asyncpg.Connection = Depends(get_db)):
    """Эндпоинт для получения пользователя по ID."""
    user = await db.fetchrow(
        'SELECT id, username, email FROM users WHERE id = $1',
        user_id
    )
    if not user:
        from fastapi import HTTPException
        raise HTTPException(status_code=404, detail="User not found")
    return dict(user)


@app.post("/users")
async def create_user(
    username: str,
    email: str,
    db: asyncpg.Connection = Depends(get_db)
):
    """Эндпоинт для создания пользователя."""
    user = await db.fetchrow('''
        INSERT INTO users (username, email)
        VALUES ($1, $2)
        RETURNING id, username, email
    ''', username, email)
    return dict(user)
```

## Настройка пула для разных сценариев

```python
import asyncpg

# Для веб-приложения с высокой нагрузкой
async def create_web_pool():
    return await asyncpg.create_pool(
        'postgresql://postgres:password@localhost/mydb',
        min_size=10,
        max_size=50,
        max_queries=10000,
        max_inactive_connection_lifetime=60.0,
        command_timeout=30.0,
    )


# Для фоновых задач с редкими запросами
async def create_worker_pool():
    return await asyncpg.create_pool(
        'postgresql://postgres:password@localhost/mydb',
        min_size=1,
        max_size=5,
        max_queries=1000,
        max_inactive_connection_lifetime=300.0,
        command_timeout=300.0,  # Долгие запросы допустимы
    )


# Для тестирования
async def create_test_pool():
    return await asyncpg.create_pool(
        'postgresql://postgres:password@localhost/test_db',
        min_size=1,
        max_size=2,
        command_timeout=10.0,
    )
```

## Лучшие практики

1. **Правильно подбирайте размер пула**
   - `min_size` = количество постоянных соединений
   - `max_size` = максимум одновременных запросов
   - Рекомендация: `max_size` = `max_connections` PostgreSQL / кол-во инстансов приложения

2. **Используйте context manager** для гарантированного возврата соединения

3. **Не храните соединения** — получайте из пула, используйте, возвращайте

4. **Настраивайте таймауты** для предотвращения зависания

5. **Мониторьте метрики пула** в production

```python
# Хороший паттерн
async with pool.acquire() as conn:
    await conn.fetch(...)  # Соединение вернётся в пул

# Плохой паттерн
conn = await pool.acquire()
await conn.fetch(...)
# Забыли вернуть в пул!
```

---

[prev: ./04-queries.md](./04-queries.md) | [next: ./06-transactions.md](./06-transactions.md)
