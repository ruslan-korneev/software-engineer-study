# Реализация ASGI в Starlette

[prev: 02-asgi.md](./02-asgi.md) | [next: 04-django-async.md](./04-django-async.md)

---

## Введение в Starlette

**Starlette** - это легковесный ASGI фреймворк для создания высокопроизводительных асинхронных веб-приложений. Он является основой для FastAPI и предоставляет все необходимые компоненты для разработки REST API, WebSocket серверов и веб-приложений.

### Установка

```bash
pip install starlette
# С дополнительными зависимостями
pip install starlette[full]
# ASGI сервер
pip install uvicorn
```

## Основы Starlette

### Минимальное приложение

```python
from starlette.applications import Starlette
from starlette.responses import JSONResponse, PlainTextResponse
from starlette.routing import Route

async def homepage(request):
    return PlainTextResponse('Привет, мир!')

async def user_info(request):
    user_id = request.path_params['user_id']
    return JSONResponse({'user_id': user_id})

# Определение маршрутов
routes = [
    Route('/', homepage),
    Route('/users/{user_id:int}', user_info),
]

app = Starlette(routes=routes)
```

### Запуск приложения

```bash
uvicorn myapp:app --reload
```

```python
# Или программно
import uvicorn

if __name__ == "__main__":
    uvicorn.run("myapp:app", host="0.0.0.0", port=8000, reload=True)
```

## Маршрутизация

### Типы маршрутов

```python
from starlette.routing import Route, Mount, WebSocketRoute
from starlette.staticfiles import StaticFiles

# Обычные HTTP маршруты
routes = [
    Route('/', homepage, methods=['GET']),
    Route('/items', list_items, methods=['GET']),
    Route('/items', create_item, methods=['POST']),
    Route('/items/{item_id:int}', get_item, methods=['GET']),
    Route('/items/{item_id:int}', update_item, methods=['PUT']),
    Route('/items/{item_id:int}', delete_item, methods=['DELETE']),
]
```

### Группировка маршрутов с Mount

```python
from starlette.routing import Route, Mount

# Обработчики для API v1
async def api_v1_users(request):
    return JSONResponse({'version': 'v1', 'endpoint': 'users'})

async def api_v1_items(request):
    return JSONResponse({'version': 'v1', 'endpoint': 'items'})

# Обработчики для API v2
async def api_v2_users(request):
    return JSONResponse({'version': 'v2', 'endpoint': 'users'})

routes = [
    Route('/', homepage),
    Mount('/api/v1', routes=[
        Route('/users', api_v1_users),
        Route('/items', api_v1_items),
    ]),
    Mount('/api/v2', routes=[
        Route('/users', api_v2_users),
    ]),
    # Статические файлы
    Mount('/static', app=StaticFiles(directory='static'), name='static'),
]

app = Starlette(routes=routes)
```

### Конвертеры параметров URL

```python
from starlette.routing import Route

# Доступные конвертеры:
# str - любая строка (по умолчанию)
# int - целое число
# float - число с плавающей точкой
# path - строка включая слеши
# uuid - UUID формат

routes = [
    Route('/users/{user_id:int}', get_user),           # /users/123
    Route('/files/{path:path}', get_file),             # /files/docs/readme.txt
    Route('/items/{item_uuid:uuid}', get_item_by_uuid), # /items/550e8400-...
    Route('/prices/{price:float}', get_by_price),      # /prices/19.99
]
```

## Объект Request

### Получение данных запроса

```python
from starlette.requests import Request
from starlette.responses import JSONResponse

async def handle_request(request: Request):
    # Метод и путь
    method = request.method  # GET, POST, etc.
    path = request.url.path  # /users/123

    # Параметры URL
    user_id = request.path_params.get('user_id')

    # Query параметры
    page = request.query_params.get('page', '1')
    limit = request.query_params.get('limit', '10')
    # Для множественных значений
    tags = request.query_params.getlist('tag')  # ?tag=a&tag=b -> ['a', 'b']

    # Заголовки
    content_type = request.headers.get('content-type')
    auth = request.headers.get('authorization')

    # Тело запроса
    body_bytes = await request.body()
    body_json = await request.json()

    # Форма и файлы
    form_data = await request.form()
    uploaded_file = form_data.get('file')

    # Клиент
    client_host = request.client.host
    client_port = request.client.port

    # Состояние (для передачи данных между middleware)
    user = request.state.user  # если установлено в middleware

    return JSONResponse({
        'method': method,
        'path': path,
        'user_id': user_id,
    })
```

### Работа с формами и файлами

```python
from starlette.requests import Request
from starlette.responses import JSONResponse
import aiofiles

async def upload_file(request: Request):
    form = await request.form()

    # Получение файла
    uploaded_file = form['document']
    filename = uploaded_file.filename
    content_type = uploaded_file.content_type

    # Сохранение файла
    async with aiofiles.open(f'uploads/{filename}', 'wb') as f:
        content = await uploaded_file.read()
        await f.write(content)

    return JSONResponse({
        'filename': filename,
        'content_type': content_type,
        'size': len(content)
    })
```

## Типы Response

### Основные типы ответов

```python
from starlette.responses import (
    Response,
    PlainTextResponse,
    HTMLResponse,
    JSONResponse,
    RedirectResponse,
    StreamingResponse,
    FileResponse,
)

# Простой текст
async def plain_text(request):
    return PlainTextResponse('Hello, World!')

# HTML
async def html_page(request):
    html_content = """
    <!DOCTYPE html>
    <html>
        <head><title>My Page</title></head>
        <body><h1>Welcome!</h1></body>
    </html>
    """
    return HTMLResponse(html_content)

# JSON
async def json_data(request):
    return JSONResponse(
        {'message': 'success', 'data': [1, 2, 3]},
        status_code=200,
        headers={'X-Custom-Header': 'value'}
    )

# Редирект
async def redirect(request):
    return RedirectResponse(url='/new-location', status_code=302)

# Файл
async def download_file(request):
    return FileResponse(
        'files/document.pdf',
        filename='download.pdf',
        media_type='application/pdf'
    )
```

### Streaming Response

```python
from starlette.responses import StreamingResponse
import asyncio

async def slow_numbers():
    """Генератор для streaming."""
    for i in range(10):
        yield f"data: {i}\n\n"
        await asyncio.sleep(1)

async def stream_numbers(request):
    return StreamingResponse(
        slow_numbers(),
        media_type='text/event-stream'
    )

# Streaming большого файла
async def stream_large_file(request):
    async def file_iterator():
        async with aiofiles.open('large_file.bin', 'rb') as f:
            while chunk := await f.read(8192):
                yield chunk

    return StreamingResponse(
        file_iterator(),
        media_type='application/octet-stream'
    )
```

## Middleware

### Создание middleware

```python
from starlette.middleware import Middleware
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
import time

class TimingMiddleware(BaseHTTPMiddleware):
    """Middleware для измерения времени запроса."""

    async def dispatch(self, request: Request, call_next):
        start_time = time.time()

        response = await call_next(request)

        process_time = time.time() - start_time
        response.headers['X-Process-Time'] = str(process_time)

        return response

class AuthMiddleware(BaseHTTPMiddleware):
    """Middleware для аутентификации."""

    async def dispatch(self, request: Request, call_next):
        # Пропускаем публичные маршруты
        if request.url.path in ['/', '/health', '/login']:
            return await call_next(request)

        # Проверяем токен
        auth_header = request.headers.get('Authorization', '')
        if not auth_header.startswith('Bearer '):
            return JSONResponse(
                {'error': 'Unauthorized'},
                status_code=401
            )

        token = auth_header[7:]
        # Валидация токена...
        request.state.user_id = 'decoded_user_id'

        return await call_next(request)
```

### Использование встроенных middleware

```python
from starlette.applications import Starlette
from starlette.middleware import Middleware
from starlette.middleware.cors import CORSMiddleware
from starlette.middleware.gzip import GZipMiddleware
from starlette.middleware.trustedhost import TrustedHostMiddleware
from starlette.middleware.httpsredirect import HTTPSRedirectMiddleware

middleware = [
    # CORS
    Middleware(
        CORSMiddleware,
        allow_origins=['https://example.com', 'https://www.example.com'],
        allow_methods=['GET', 'POST', 'PUT', 'DELETE'],
        allow_headers=['*'],
        allow_credentials=True,
    ),
    # Сжатие
    Middleware(GZipMiddleware, minimum_size=1000),
    # Проверка хоста
    Middleware(TrustedHostMiddleware, allowed_hosts=['example.com', '*.example.com']),
    # Редирект на HTTPS
    Middleware(HTTPSRedirectMiddleware),
    # Кастомный middleware
    Middleware(TimingMiddleware),
    Middleware(AuthMiddleware),
]

app = Starlette(routes=routes, middleware=middleware)
```

### Чистый ASGI middleware

```python
from starlette.types import ASGIApp, Receive, Scope, Send

class PureASGIMiddleware:
    """Middleware в чистом ASGI стиле."""

    def __init__(self, app: ASGIApp):
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send):
        if scope['type'] != 'http':
            await self.app(scope, receive, send)
            return

        # Модификация scope
        scope['state']['custom_data'] = 'value'

        # Перехват ответа
        async def send_wrapper(message):
            if message['type'] == 'http.response.start':
                # Добавляем заголовок
                headers = list(message.get('headers', []))
                headers.append((b'x-custom', b'value'))
                message['headers'] = headers
            await send(message)

        await self.app(scope, receive, send_wrapper)
```

## WebSocket

### Обработка WebSocket соединений

```python
from starlette.websockets import WebSocket
from starlette.routing import WebSocketRoute
import json

async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()

    try:
        while True:
            # Получение данных
            data = await websocket.receive_text()
            # Или: data = await websocket.receive_json()
            # Или: data = await websocket.receive_bytes()

            # Обработка
            message = json.loads(data)
            response = {'echo': message, 'status': 'received'}

            # Отправка ответа
            await websocket.send_json(response)
            # Или: await websocket.send_text(text)
            # Или: await websocket.send_bytes(bytes)

    except Exception as e:
        print(f"WebSocket error: {e}")
    finally:
        await websocket.close()

routes = [
    WebSocketRoute('/ws', websocket_endpoint),
]
```

### WebSocket с аутентификацией

```python
from starlette.websockets import WebSocket

async def authenticated_websocket(websocket: WebSocket):
    # Получаем токен из query параметров
    token = websocket.query_params.get('token')

    if not token or not validate_token(token):
        await websocket.close(code=4001, reason="Unauthorized")
        return

    await websocket.accept()
    # ... обработка сообщений
```

### Broadcast паттерн

```python
from starlette.websockets import WebSocket
from typing import List
import asyncio

class ConnectionManager:
    """Менеджер WebSocket соединений."""

    def __init__(self):
        self.active_connections: List[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)

    async def broadcast(self, message: dict):
        """Отправка сообщения всем подключенным клиентам."""
        for connection in self.active_connections:
            try:
                await connection.send_json(message)
            except Exception:
                self.disconnect(connection)

manager = ConnectionManager()

async def chat_websocket(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_json()
            await manager.broadcast({
                'user': data.get('user', 'anonymous'),
                'message': data.get('message', '')
            })
    except Exception:
        manager.disconnect(websocket)
```

## События жизненного цикла

### Startup и Shutdown

```python
from starlette.applications import Starlette
import asyncpg
import aiohttp

# Хранилище ресурсов
resources = {}

async def startup():
    """Инициализация при запуске."""
    print("Starting up...")

    # Создание пула соединений к БД
    resources['db_pool'] = await asyncpg.create_pool(
        'postgresql://user:pass@localhost/db',
        min_size=5,
        max_size=20
    )

    # HTTP клиент
    resources['http_client'] = aiohttp.ClientSession()

    print("Startup complete!")

async def shutdown():
    """Очистка при завершении."""
    print("Shutting down...")

    # Закрытие пула БД
    if 'db_pool' in resources:
        await resources['db_pool'].close()

    # Закрытие HTTP клиента
    if 'http_client' in resources:
        await resources['http_client'].close()

    print("Shutdown complete!")

app = Starlette(
    routes=routes,
    on_startup=[startup],
    on_shutdown=[shutdown],
)

# Использование ресурсов в обработчиках
async def get_users(request):
    pool = resources['db_pool']
    async with pool.acquire() as conn:
        rows = await conn.fetch('SELECT * FROM users')
    return JSONResponse([dict(row) for row in rows])
```

## Обработка исключений

### Глобальная обработка ошибок

```python
from starlette.applications import Starlette
from starlette.exceptions import HTTPException
from starlette.requests import Request
from starlette.responses import JSONResponse

async def http_exception_handler(request: Request, exc: HTTPException):
    """Обработчик HTTP исключений."""
    return JSONResponse(
        {'error': exc.detail},
        status_code=exc.status_code
    )

async def general_exception_handler(request: Request, exc: Exception):
    """Обработчик всех остальных исключений."""
    return JSONResponse(
        {'error': 'Internal server error'},
        status_code=500
    )

# Кастомные исключения
class ValidationError(Exception):
    def __init__(self, detail: str):
        self.detail = detail

async def validation_exception_handler(request: Request, exc: ValidationError):
    return JSONResponse(
        {'error': 'Validation failed', 'detail': exc.detail},
        status_code=400
    )

exception_handlers = {
    HTTPException: http_exception_handler,
    ValidationError: validation_exception_handler,
    Exception: general_exception_handler,
}

app = Starlette(
    routes=routes,
    exception_handlers=exception_handlers,
)
```

## Шаблоны с Jinja2

### Настройка шаблонов

```python
from starlette.applications import Starlette
from starlette.templating import Jinja2Templates
from starlette.staticfiles import StaticFiles

templates = Jinja2Templates(directory='templates')

async def homepage(request):
    return templates.TemplateResponse(
        'index.html',
        {
            'request': request,  # Обязательно!
            'title': 'Главная страница',
            'items': ['Item 1', 'Item 2', 'Item 3'],
        }
    )

routes = [
    Route('/', homepage),
    Mount('/static', StaticFiles(directory='static'), name='static'),
]

app = Starlette(routes=routes)
```

```html
<!-- templates/index.html -->
<!DOCTYPE html>
<html>
<head>
    <title>{{ title }}</title>
    <link rel="stylesheet" href="{{ url_for('static', path='style.css') }}">
</head>
<body>
    <h1>{{ title }}</h1>
    <ul>
        {% for item in items %}
            <li>{{ item }}</li>
        {% endfor %}
    </ul>
</body>
</html>
```

## Тестирование

### Использование TestClient

```python
from starlette.testclient import TestClient
from starlette.applications import Starlette
from starlette.responses import JSONResponse
from starlette.routing import Route
import pytest

# Приложение для тестирования
async def get_users(request):
    return JSONResponse([{'id': 1, 'name': 'Alice'}])

async def create_user(request):
    data = await request.json()
    return JSONResponse({'id': 2, **data}, status_code=201)

app = Starlette(routes=[
    Route('/users', get_users, methods=['GET']),
    Route('/users', create_user, methods=['POST']),
])

# Тесты
def test_get_users():
    client = TestClient(app)
    response = client.get('/users')
    assert response.status_code == 200
    assert response.json() == [{'id': 1, 'name': 'Alice'}]

def test_create_user():
    client = TestClient(app)
    response = client.post('/users', json={'name': 'Bob'})
    assert response.status_code == 201
    assert response.json()['name'] == 'Bob'

def test_websocket():
    client = TestClient(app)
    with client.websocket_connect('/ws') as websocket:
        websocket.send_json({'message': 'hello'})
        data = websocket.receive_json()
        assert data['echo']['message'] == 'hello'
```

### Асинхронные тесты с pytest-asyncio

```python
import pytest
from httpx import AsyncClient, ASGITransport

@pytest.mark.asyncio
async def test_async_get_users():
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test"
    ) as client:
        response = await client.get('/users')
        assert response.status_code == 200
```

## Best Practices

### 1. Структура проекта

```
myproject/
├── app/
│   ├── __init__.py
│   ├── main.py          # Точка входа
│   ├── routes/
│   │   ├── __init__.py
│   │   ├── users.py
│   │   └── items.py
│   ├── middleware/
│   │   └── auth.py
│   ├── services/
│   │   └── user_service.py
│   └── models/
│       └── user.py
├── templates/
├── static/
├── tests/
└── config.py
```

### 2. Dependency Injection

```python
from starlette.requests import Request

class Database:
    async def get_user(self, user_id: int):
        # ...
        pass

def get_db(request: Request) -> Database:
    return request.app.state.db

async def get_user(request: Request):
    db = get_db(request)
    user = await db.get_user(request.path_params['user_id'])
    return JSONResponse(user)
```

### 3. Конфигурация

```python
from starlette.config import Config
from starlette.datastructures import Secret

config = Config('.env')

DEBUG = config('DEBUG', cast=bool, default=False)
DATABASE_URL = config('DATABASE_URL')
SECRET_KEY = config('SECRET_KEY', cast=Secret)
ALLOWED_HOSTS = config('ALLOWED_HOSTS', cast=lambda v: v.split(','))
```

## Частые ошибки

1. **Забыть передать `request` в шаблон** - Jinja2Templates требует его для url_for
2. **Блокирующие операции** - используйте `run_in_executor` для синхронного кода
3. **Утечки WebSocket соединений** - всегда обрабатывайте отключение
4. **Отсутствие cleanup** - регистрируйте on_shutdown для освобождения ресурсов

---

[prev: 02-asgi.md](./02-asgi.md) | [next: 04-django-async.md](./04-django-async.md)
