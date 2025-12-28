# Разработка REST API с aiohttp

[prev: readme.md](./readme.md) | [next: 02-asgi.md](./02-asgi.md)

---

## Введение в aiohttp

**aiohttp** - это асинхронная HTTP-библиотека для Python, построенная на базе asyncio. Она предоставляет как клиентскую, так и серверную часть для работы с HTTP-протоколом. Это одна из первых зрелых асинхронных веб-библиотек в экосистеме Python.

### Установка

```bash
pip install aiohttp
# Для ускорения работы (опционально)
pip install aiodns cchardet
```

## Основы серверной части aiohttp

### Минимальное приложение

```python
from aiohttp import web

async def hello(request):
    """Простой обработчик запроса."""
    return web.Response(text="Привет, мир!")

async def hello_name(request):
    """Обработчик с параметром из URL."""
    name = request.match_info.get('name', 'Anonymous')
    return web.Response(text=f"Привет, {name}!")

# Создание приложения и маршрутов
app = web.Application()
app.add_routes([
    web.get('/', hello),
    web.get('/hello/{name}', hello_name),
])

if __name__ == '__main__':
    web.run_app(app, host='0.0.0.0', port=8080)
```

### Альтернативный синтаксис с декораторами

```python
from aiohttp import web

routes = web.RouteTableDef()

@routes.get('/')
async def index(request):
    return web.Response(text="Главная страница")

@routes.get('/users/{user_id}')
async def get_user(request):
    user_id = request.match_info['user_id']
    return web.json_response({'user_id': user_id})

@routes.post('/users')
async def create_user(request):
    data = await request.json()
    return web.json_response({'created': data}, status=201)

app = web.Application()
app.add_routes(routes)
```

## Создание полноценного REST API

### Структура проекта

```
myapi/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── routes.py
│   ├── handlers/
│   │   ├── __init__.py
│   │   └── users.py
│   ├── models/
│   │   └── user.py
│   └── middleware/
│       └── auth.py
├── config.py
└── requirements.txt
```

### Модель данных (models/user.py)

```python
from dataclasses import dataclass, asdict
from typing import Optional
import uuid

@dataclass
class User:
    id: str
    username: str
    email: str
    is_active: bool = True

    @classmethod
    def create(cls, username: str, email: str) -> 'User':
        return cls(
            id=str(uuid.uuid4()),
            username=username,
            email=email
        )

    def to_dict(self) -> dict:
        return asdict(self)
```

### Обработчики (handlers/users.py)

```python
from aiohttp import web
from typing import Dict

# Имитация базы данных
users_db: Dict[str, dict] = {}

class UserHandler:
    """Класс-обработчик для работы с пользователями."""

    async def list_users(self, request: web.Request) -> web.Response:
        """GET /api/users - список всех пользователей."""
        # Поддержка пагинации
        page = int(request.query.get('page', 1))
        per_page = int(request.query.get('per_page', 10))

        users = list(users_db.values())
        start = (page - 1) * per_page
        end = start + per_page

        return web.json_response({
            'users': users[start:end],
            'total': len(users),
            'page': page,
            'per_page': per_page
        })

    async def get_user(self, request: web.Request) -> web.Response:
        """GET /api/users/{id} - получение пользователя по ID."""
        user_id = request.match_info['id']

        if user_id not in users_db:
            raise web.HTTPNotFound(
                text='{"error": "User not found"}',
                content_type='application/json'
            )

        return web.json_response(users_db[user_id])

    async def create_user(self, request: web.Request) -> web.Response:
        """POST /api/users - создание нового пользователя."""
        try:
            data = await request.json()
        except Exception:
            raise web.HTTPBadRequest(
                text='{"error": "Invalid JSON"}',
                content_type='application/json'
            )

        # Валидация
        required_fields = ['username', 'email']
        for field in required_fields:
            if field not in data:
                raise web.HTTPBadRequest(
                    text=f'{{"error": "Missing field: {field}"}}',
                    content_type='application/json'
                )

        # Создание пользователя
        import uuid
        user_id = str(uuid.uuid4())
        user = {
            'id': user_id,
            'username': data['username'],
            'email': data['email'],
            'is_active': True
        }
        users_db[user_id] = user

        return web.json_response(user, status=201)

    async def update_user(self, request: web.Request) -> web.Response:
        """PUT /api/users/{id} - обновление пользователя."""
        user_id = request.match_info['id']

        if user_id not in users_db:
            raise web.HTTPNotFound(
                text='{"error": "User not found"}',
                content_type='application/json'
            )

        data = await request.json()
        users_db[user_id].update(data)

        return web.json_response(users_db[user_id])

    async def delete_user(self, request: web.Request) -> web.Response:
        """DELETE /api/users/{id} - удаление пользователя."""
        user_id = request.match_info['id']

        if user_id not in users_db:
            raise web.HTTPNotFound(
                text='{"error": "User not found"}',
                content_type='application/json'
            )

        del users_db[user_id]
        return web.Response(status=204)
```

### Настройка маршрутов (routes.py)

```python
from aiohttp import web
from .handlers.users import UserHandler

def setup_routes(app: web.Application):
    """Настройка всех маршрутов приложения."""
    user_handler = UserHandler()

    # API маршруты для пользователей
    app.router.add_get('/api/users', user_handler.list_users)
    app.router.add_get('/api/users/{id}', user_handler.get_user)
    app.router.add_post('/api/users', user_handler.create_user)
    app.router.add_put('/api/users/{id}', user_handler.update_user)
    app.router.add_delete('/api/users/{id}', user_handler.delete_user)
```

## Middleware (Промежуточное ПО)

### Создание middleware

```python
from aiohttp import web
import time
import logging

logger = logging.getLogger(__name__)

@web.middleware
async def logging_middleware(request: web.Request, handler):
    """Middleware для логирования запросов."""
    start_time = time.time()

    try:
        response = await handler(request)
        elapsed = time.time() - start_time
        logger.info(
            f"{request.method} {request.path} - "
            f"{response.status} ({elapsed:.3f}s)"
        )
        return response
    except web.HTTPException as ex:
        elapsed = time.time() - start_time
        logger.warning(
            f"{request.method} {request.path} - "
            f"{ex.status} ({elapsed:.3f}s)"
        )
        raise

@web.middleware
async def error_middleware(request: web.Request, handler):
    """Middleware для обработки ошибок."""
    try:
        return await handler(request)
    except web.HTTPException:
        raise
    except Exception as ex:
        logger.exception("Unhandled exception")
        return web.json_response(
            {'error': 'Internal server error'},
            status=500
        )

@web.middleware
async def auth_middleware(request: web.Request, handler):
    """Middleware для аутентификации."""
    # Пропускаем публичные маршруты
    public_paths = ['/', '/api/auth/login', '/api/health']
    if request.path in public_paths:
        return await handler(request)

    # Проверяем токен
    auth_header = request.headers.get('Authorization', '')
    if not auth_header.startswith('Bearer '):
        raise web.HTTPUnauthorized(
            text='{"error": "Missing or invalid token"}',
            content_type='application/json'
        )

    token = auth_header[7:]  # Убираем "Bearer "
    # Здесь должна быть проверка токена
    request['user_id'] = 'decoded_user_id'

    return await handler(request)
```

### Применение middleware

```python
from aiohttp import web

app = web.Application(middlewares=[
    logging_middleware,
    error_middleware,
    auth_middleware,
])
```

## Работа с контекстом приложения

### Хранение общих ресурсов

```python
from aiohttp import web
import aiohttp
import asyncpg

async def init_db(app: web.Application):
    """Инициализация пула соединений с базой данных."""
    app['db_pool'] = await asyncpg.create_pool(
        'postgresql://user:password@localhost/mydb',
        min_size=5,
        max_size=20
    )
    yield  # Приложение работает
    # Очистка при завершении
    await app['db_pool'].close()

async def init_http_client(app: web.Application):
    """Инициализация HTTP клиента."""
    app['http_session'] = aiohttp.ClientSession()
    yield
    await app['http_session'].close()

# Создание приложения с cleanup контекстами
app = web.Application()
app.cleanup_ctx.append(init_db)
app.cleanup_ctx.append(init_http_client)
```

### Использование ресурсов в обработчиках

```python
async def get_user_from_db(request: web.Request) -> web.Response:
    """Получение пользователя из базы данных."""
    user_id = request.match_info['id']

    # Получаем пул из контекста приложения
    pool = request.app['db_pool']

    async with pool.acquire() as conn:
        row = await conn.fetchrow(
            'SELECT * FROM users WHERE id = $1',
            user_id
        )

    if not row:
        raise web.HTTPNotFound()

    return web.json_response(dict(row))

async def fetch_external_data(request: web.Request) -> web.Response:
    """Запрос к внешнему API."""
    session = request.app['http_session']

    async with session.get('https://api.example.com/data') as resp:
        data = await resp.json()

    return web.json_response(data)
```

## Валидация данных

### Использование pydantic для валидации

```python
from aiohttp import web
from pydantic import BaseModel, EmailStr, ValidationError
from typing import Optional

class UserCreate(BaseModel):
    username: str
    email: EmailStr
    password: str
    age: Optional[int] = None

class UserUpdate(BaseModel):
    username: Optional[str] = None
    email: Optional[EmailStr] = None
    age: Optional[int] = None

async def create_user_validated(request: web.Request) -> web.Response:
    """Создание пользователя с валидацией через pydantic."""
    try:
        data = await request.json()
        user_data = UserCreate(**data)
    except ValidationError as e:
        return web.json_response(
            {'errors': e.errors()},
            status=400
        )
    except Exception:
        return web.json_response(
            {'error': 'Invalid JSON'},
            status=400
        )

    # Используем провалидированные данные
    # user_data.username, user_data.email, etc.
    return web.json_response(
        {'message': 'User created', 'data': user_data.model_dump()},
        status=201
    )
```

## Обработка ошибок

### Централизованная обработка ошибок

```python
from aiohttp import web
from typing import Type

class APIError(Exception):
    """Базовый класс для API ошибок."""
    status_code: int = 500
    message: str = "Internal server error"

    def __init__(self, message: str = None, details: dict = None):
        self.message = message or self.message
        self.details = details or {}

class NotFoundError(APIError):
    status_code = 404
    message = "Resource not found"

class ValidationError(APIError):
    status_code = 400
    message = "Validation error"

class UnauthorizedError(APIError):
    status_code = 401
    message = "Unauthorized"

@web.middleware
async def api_error_middleware(request: web.Request, handler):
    """Middleware для обработки API ошибок."""
    try:
        return await handler(request)
    except APIError as e:
        return web.json_response(
            {
                'error': e.message,
                'details': e.details
            },
            status=e.status_code
        )
    except web.HTTPException:
        raise
    except Exception as e:
        return web.json_response(
            {'error': 'Internal server error'},
            status=500
        )

# Использование
async def get_item(request: web.Request) -> web.Response:
    item_id = request.match_info['id']
    item = await find_item(item_id)

    if not item:
        raise NotFoundError(f"Item {item_id} not found")

    return web.json_response(item)
```

## Best Practices (Лучшие практики)

### 1. Структурирование кода

```python
# Используйте классы для группировки связанных обработчиков
class ArticleResource:
    def __init__(self, db_pool):
        self.db_pool = db_pool

    async def list(self, request):
        pass

    async def get(self, request):
        pass

    async def create(self, request):
        pass
```

### 2. Использование типов

```python
from aiohttp import web
from typing import TypedDict

class UserDict(TypedDict):
    id: str
    username: str
    email: str

async def get_user(request: web.Request) -> web.Response:
    user: UserDict = await fetch_user(request.match_info['id'])
    return web.json_response(user)
```

### 3. Graceful shutdown

```python
import signal
from aiohttp import web

async def on_shutdown(app: web.Application):
    """Действия при завершении приложения."""
    # Закрытие соединений, сохранение состояния и т.д.
    print("Shutting down...")

app = web.Application()
app.on_shutdown.append(on_shutdown)
```

## Частые ошибки

1. **Блокирующие операции в async-коде**: Не используйте синхронные библиотеки без обертки в `run_in_executor`.

2. **Утечки соединений**: Всегда закрывайте HTTP-сессии и пулы соединений при завершении.

3. **Отсутствие обработки ошибок**: Всегда обрабатывайте исключения в middleware.

4. **Хранение состояния в глобальных переменных**: Используйте `app['key']` для хранения общих ресурсов.

## Тестирование

```python
import pytest
from aiohttp import web
from aiohttp.test_utils import AioHTTPTestCase, unittest_run_loop

class TestUserAPI(AioHTTPTestCase):
    async def get_application(self):
        app = web.Application()
        # Настройка маршрутов
        return app

    @unittest_run_loop
    async def test_create_user(self):
        resp = await self.client.post(
            '/api/users',
            json={'username': 'test', 'email': 'test@example.com'}
        )
        assert resp.status == 201
        data = await resp.json()
        assert 'id' in data
```

---

[prev: readme.md](./readme.md) | [next: 02-asgi.md](./02-asgi.md)
