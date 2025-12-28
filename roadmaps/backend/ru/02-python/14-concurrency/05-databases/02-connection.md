# Подключение к PostgreSQL

[prev: ./01-asyncpg-intro.md](./01-asyncpg-intro.md) | [next: ./03-schema.md](./03-schema.md)

---

## Способы подключения к базе данных

asyncpg предоставляет несколько способов подключения к PostgreSQL. Рассмотрим каждый из них подробно.

## Подключение через DSN (Data Source Name)

DSN — это строка подключения, содержащая все параметры соединения в одном формате.

```python
import asyncio
import asyncpg


async def connect_with_dsn():
    # Формат DSN: postgresql://user:password@host:port/database
    conn = await asyncpg.connect(
        'postgresql://postgres:mypassword@localhost:5432/mydb'
    )

    # Проверка соединения
    version = await conn.fetchval('SELECT version()')
    print(f"PostgreSQL version: {version}")

    await conn.close()


asyncio.run(connect_with_dsn())
```

### Расширенный формат DSN с параметрами

```python
async def connect_with_dsn_params():
    # DSN с дополнительными параметрами
    dsn = (
        'postgresql://postgres:password@localhost:5432/mydb'
        '?sslmode=require'
        '&application_name=my_app'
        '&connect_timeout=10'
    )

    conn = await asyncpg.connect(dsn)
    await conn.close()
```

## Подключение через именованные параметры

Этот способ более читаемый и удобен для конфигурирования.

```python
import asyncio
import asyncpg


async def connect_with_params():
    conn = await asyncpg.connect(
        user='postgres',
        password='mypassword',
        database='mydb',
        host='localhost',
        port=5432
    )

    # Работа с соединением
    result = await conn.fetchval('SELECT 1 + 1')
    print(f"Result: {result}")

    await conn.close()


asyncio.run(connect_with_params())
```

## Полный список параметров подключения

```python
import asyncio
import asyncpg
import ssl


async def connect_full_params():
    # Создание SSL контекста для защищённого соединения
    ssl_context = ssl.create_default_context(ssl.Purpose.SERVER_AUTH)
    ssl_context.check_hostname = True
    ssl_context.verify_mode = ssl.CERT_REQUIRED

    conn = await asyncpg.connect(
        # Основные параметры
        user='postgres',
        password='mypassword',
        database='mydb',
        host='localhost',
        port=5432,

        # Таймауты
        timeout=60,              # Общий таймаут операций (секунды)
        command_timeout=30,      # Таймаут выполнения команд

        # SSL
        ssl=ssl_context,         # или ssl='require', ssl='prefer', ssl=True

        # Дополнительные параметры
        statement_cache_size=100,  # Размер кэша prepared statements
        max_cached_statement_lifetime=300,  # Время жизни кэша (секунды)
        max_cacheable_statement_size=1024,  # Макс. размер кэшируемого запроса

        # Параметры сессии PostgreSQL
        server_settings={
            'application_name': 'my_application',
            'search_path': 'public, custom_schema',
            'timezone': 'Europe/Moscow',
            'statement_timeout': '30000',  # 30 секунд в миллисекундах
        }
    )

    await conn.close()
```

## Использование переменных окружения

asyncpg поддерживает стандартные переменные окружения PostgreSQL.

```python
import os
import asyncio
import asyncpg

# Установка переменных окружения
os.environ['PGHOST'] = 'localhost'
os.environ['PGPORT'] = '5432'
os.environ['PGDATABASE'] = 'mydb'
os.environ['PGUSER'] = 'postgres'
os.environ['PGPASSWORD'] = 'mypassword'


async def connect_with_env():
    # asyncpg автоматически использует переменные окружения
    conn = await asyncpg.connect()

    result = await conn.fetchval('SELECT current_database()')
    print(f"Connected to: {result}")

    await conn.close()


asyncio.run(connect_with_env())
```

## Context Manager для безопасного управления соединением

```python
import asyncio
import asyncpg
from contextlib import asynccontextmanager


@asynccontextmanager
async def get_connection(**kwargs):
    """Context manager для автоматического закрытия соединения."""
    conn = await asyncpg.connect(**kwargs)
    try:
        yield conn
    finally:
        await conn.close()


async def main():
    async with get_connection(
        user='postgres',
        password='password',
        database='mydb',
        host='localhost'
    ) as conn:
        # Соединение автоматически закроется после выхода из блока
        result = await conn.fetch('SELECT * FROM users LIMIT 5')
        for row in result:
            print(row)


asyncio.run(main())
```

## Обработка ошибок подключения

```python
import asyncio
import asyncpg


async def connect_with_error_handling():
    try:
        conn = await asyncpg.connect(
            user='postgres',
            password='wrong_password',  # Неверный пароль
            database='mydb',
            host='localhost',
            port=5432,
            timeout=10  # Таймаут подключения
        )
        return conn

    except asyncpg.InvalidPasswordError:
        print("Ошибка: неверный пароль")

    except asyncpg.InvalidCatalogNameError:
        print("Ошибка: база данных не существует")

    except asyncpg.ConnectionDoesNotExistError:
        print("Ошибка: соединение не существует")

    except asyncpg.CannotConnectNowError:
        print("Ошибка: сервер не принимает соединения")

    except asyncpg.PostgresConnectionError as e:
        print(f"Ошибка подключения к PostgreSQL: {e}")

    except asyncio.TimeoutError:
        print("Ошибка: превышено время ожидания подключения")

    except OSError as e:
        print(f"Сетевая ошибка: {e}")

    return None


asyncio.run(connect_with_error_handling())
```

## Подключение с повторными попытками

```python
import asyncio
import asyncpg
from typing import Optional


async def connect_with_retry(
    dsn: str,
    max_retries: int = 5,
    retry_delay: float = 1.0,
    backoff_factor: float = 2.0
) -> Optional[asyncpg.Connection]:
    """
    Подключение к базе данных с повторными попытками.

    Args:
        dsn: Строка подключения
        max_retries: Максимальное количество попыток
        retry_delay: Начальная задержка между попытками (секунды)
        backoff_factor: Множитель увеличения задержки
    """
    last_error = None
    current_delay = retry_delay

    for attempt in range(1, max_retries + 1):
        try:
            print(f"Попытка подключения {attempt}/{max_retries}...")
            conn = await asyncpg.connect(dsn, timeout=10)
            print("Успешное подключение!")
            return conn

        except (asyncpg.PostgresConnectionError, OSError) as e:
            last_error = e
            print(f"Ошибка: {e}")

            if attempt < max_retries:
                print(f"Повторная попытка через {current_delay:.1f} секунд...")
                await asyncio.sleep(current_delay)
                current_delay *= backoff_factor

    print(f"Не удалось подключиться после {max_retries} попыток")
    raise last_error


async def main():
    try:
        conn = await connect_with_retry(
            'postgresql://postgres:password@localhost/mydb'
        )
        if conn:
            await conn.close()
    except Exception as e:
        print(f"Финальная ошибка: {e}")


asyncio.run(main())
```

## Проверка состояния соединения

```python
import asyncio
import asyncpg


async def check_connection_status():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    # Проверка активности соединения
    print(f"Соединение закрыто: {conn.is_closed()}")

    # Получение информации о соединении
    print(f"ID процесса PostgreSQL: {conn.get_server_pid()}")

    # Получение параметров сервера
    server_version = conn.get_server_version()
    print(f"Версия сервера: {server_version}")

    # Получение настроек сессии
    settings = conn.get_settings()
    print(f"Кодировка: {settings.client_encoding}")
    print(f"Часовой пояс: {settings.timezone}")

    await conn.close()
    print(f"Соединение закрыто: {conn.is_closed()}")


asyncio.run(check_connection_status())
```

## Настройка кодировки и локали

```python
import asyncio
import asyncpg


async def configure_encoding():
    conn = await asyncpg.connect(
        user='postgres',
        password='password',
        database='mydb',
        host='localhost',
        server_settings={
            'client_encoding': 'UTF8',
            'lc_messages': 'ru_RU.UTF-8',
            'datestyle': 'ISO, DMY',
        }
    )

    # Проверка настроек
    encoding = await conn.fetchval("SHOW client_encoding")
    datestyle = await conn.fetchval("SHOW datestyle")

    print(f"Кодировка клиента: {encoding}")
    print(f"Формат даты: {datestyle}")

    await conn.close()


asyncio.run(configure_encoding())
```

## Работа с несколькими базами данных

```python
import asyncio
import asyncpg
from dataclasses import dataclass
from typing import Dict


@dataclass
class DatabaseConfig:
    host: str
    port: int
    database: str
    user: str
    password: str


class MultiDatabaseManager:
    def __init__(self):
        self.connections: Dict[str, asyncpg.Connection] = {}

    async def add_connection(self, name: str, config: DatabaseConfig):
        """Добавить соединение с базой данных."""
        conn = await asyncpg.connect(
            host=config.host,
            port=config.port,
            database=config.database,
            user=config.user,
            password=config.password
        )
        self.connections[name] = conn
        print(f"Добавлено соединение: {name}")

    def get_connection(self, name: str) -> asyncpg.Connection:
        """Получить соединение по имени."""
        if name not in self.connections:
            raise ValueError(f"Соединение '{name}' не найдено")
        return self.connections[name]

    async def close_all(self):
        """Закрыть все соединения."""
        for name, conn in self.connections.items():
            await conn.close()
            print(f"Закрыто соединение: {name}")
        self.connections.clear()


async def main():
    manager = MultiDatabaseManager()

    # Конфигурации разных баз данных
    configs = {
        'main': DatabaseConfig('localhost', 5432, 'main_db', 'user', 'pass'),
        'analytics': DatabaseConfig('localhost', 5432, 'analytics_db', 'user', 'pass'),
        'logs': DatabaseConfig('localhost', 5432, 'logs_db', 'user', 'pass'),
    }

    try:
        # Подключение ко всем базам данных
        for name, config in configs.items():
            await manager.add_connection(name, config)

        # Использование разных соединений
        main_conn = manager.get_connection('main')
        analytics_conn = manager.get_connection('analytics')

        # Параллельные запросы к разным базам
        main_result, analytics_result = await asyncio.gather(
            main_conn.fetchval('SELECT count(*) FROM users'),
            analytics_conn.fetchval('SELECT count(*) FROM events')
        )

        print(f"Пользователей в main: {main_result}")
        print(f"Событий в analytics: {analytics_result}")

    finally:
        await manager.close_all()


asyncio.run(main())
```

## Лучшие практики

1. **Используйте пулы соединений** для production-приложений вместо одиночных соединений
2. **Всегда устанавливайте таймауты** для предотвращения зависания приложения
3. **Обрабатывайте ошибки подключения** с возможностью повторных попыток
4. **Закрывайте соединения** в блоке `finally` или используйте context managers
5. **Не храните пароли в коде** — используйте переменные окружения или секреты
6. **Настраивайте SSL** для production-соединений

---

[prev: ./01-asyncpg-intro.md](./01-asyncpg-intro.md) | [next: ./03-schema.md](./03-schema.md)
