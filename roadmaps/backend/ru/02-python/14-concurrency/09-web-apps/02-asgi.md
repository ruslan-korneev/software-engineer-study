# ASGI vs WSGI

[prev: 01-rest-aiohttp.md](./01-rest-aiohttp.md) | [next: 03-starlette.md](./03-starlette.md)

---

## Введение

**WSGI** (Web Server Gateway Interface) и **ASGI** (Asynchronous Server Gateway Interface) - это спецификации интерфейса между веб-серверами и Python веб-приложениями. ASGI является преемником WSGI, добавляющим поддержку асинхронного программирования.

## WSGI - Web Server Gateway Interface

### История и предназначение

WSGI был описан в PEP 333 (2003) и обновлен в PEP 3333 (для Python 3). Это стандартный интерфейс для синхронных Python веб-приложений.

### Как работает WSGI

```python
def simple_wsgi_app(environ, start_response):
    """
    Простейшее WSGI-приложение.

    Args:
        environ: словарь с переменными окружения и данными запроса
        start_response: callable для установки статуса и заголовков

    Returns:
        Итерируемый объект с телом ответа
    """
    status = '200 OK'
    headers = [('Content-Type', 'text/plain')]
    start_response(status, headers)
    return [b'Hello, World!']
```

### Структура environ

```python
def wsgi_app(environ, start_response):
    # Данные запроса из environ
    method = environ['REQUEST_METHOD']      # GET, POST, etc.
    path = environ['PATH_INFO']             # /users/123
    query = environ['QUERY_STRING']         # page=1&limit=10
    content_type = environ.get('CONTENT_TYPE', '')
    content_length = environ.get('CONTENT_LENGTH', 0)

    # Тело запроса (для POST, PUT)
    body = environ['wsgi.input'].read()

    # Заголовки (с префиксом HTTP_)
    host = environ.get('HTTP_HOST', '')
    user_agent = environ.get('HTTP_USER_AGENT', '')

    start_response('200 OK', [('Content-Type', 'text/plain')])
    return [f'Method: {method}, Path: {path}'.encode()]
```

### Практический пример WSGI

```python
class WSGIRouter:
    """Простой WSGI роутер."""

    def __init__(self):
        self.routes = {}

    def route(self, path, methods=['GET']):
        def decorator(func):
            for method in methods:
                self.routes[(method, path)] = func
            return func
        return decorator

    def __call__(self, environ, start_response):
        method = environ['REQUEST_METHOD']
        path = environ['PATH_INFO']

        handler = self.routes.get((method, path))

        if handler:
            return handler(environ, start_response)

        start_response('404 Not Found', [('Content-Type', 'text/plain')])
        return [b'Not Found']

# Использование
app = WSGIRouter()

@app.route('/', methods=['GET'])
def index(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/html')])
    return [b'<h1>Welcome!</h1>']

@app.route('/api/users', methods=['GET'])
def list_users(environ, start_response):
    import json
    users = [{'id': 1, 'name': 'Alice'}, {'id': 2, 'name': 'Bob'}]
    start_response('200 OK', [('Content-Type', 'application/json')])
    return [json.dumps(users).encode()]
```

### Ограничения WSGI

1. **Синхронность**: Один поток обрабатывает один запрос
2. **Нет поддержки WebSocket**: Протокол не предусматривает длительные соединения
3. **Блокирующий I/O**: Ожидание базы данных блокирует весь поток
4. **Масштабирование**: Требуется много потоков/процессов для параллелизма

## ASGI - Asynchronous Server Gateway Interface

### Введение в ASGI

ASGI расширяет WSGI для поддержки асинхронного программирования, WebSocket, HTTP/2 и других протоколов.

### Структура ASGI приложения

```python
async def simple_asgi_app(scope, receive, send):
    """
    Простейшее ASGI-приложение.

    Args:
        scope: словарь с информацией о соединении
        receive: async callable для получения сообщений
        send: async callable для отправки сообщений
    """
    assert scope['type'] == 'http'

    # Отправляем начало ответа
    await send({
        'type': 'http.response.start',
        'status': 200,
        'headers': [
            [b'content-type', b'text/plain'],
        ],
    })

    # Отправляем тело ответа
    await send({
        'type': 'http.response.body',
        'body': b'Hello, World!',
    })
```

### Типы сообщений в ASGI

```python
async def asgi_app(scope, receive, send):
    """ASGI приложение с обработкой разных типов соединений."""

    if scope['type'] == 'http':
        # HTTP запрос
        await handle_http(scope, receive, send)

    elif scope['type'] == 'websocket':
        # WebSocket соединение
        await handle_websocket(scope, receive, send)

    elif scope['type'] == 'lifespan':
        # События жизненного цикла (startup/shutdown)
        await handle_lifespan(scope, receive, send)

async def handle_http(scope, receive, send):
    """Обработка HTTP запроса."""
    # Получение тела запроса
    body = b''
    more_body = True

    while more_body:
        message = await receive()
        body += message.get('body', b'')
        more_body = message.get('more_body', False)

    # Данные из scope
    method = scope['method']  # GET, POST, etc.
    path = scope['path']      # /users/123
    query_string = scope['query_string']  # b'page=1'
    headers = dict(scope['headers'])  # [(b'host', b'example.com')]

    # Отправка ответа
    response_body = f'Method: {method}, Path: {path}'.encode()

    await send({
        'type': 'http.response.start',
        'status': 200,
        'headers': [[b'content-type', b'text/plain']],
    })
    await send({
        'type': 'http.response.body',
        'body': response_body,
    })

async def handle_websocket(scope, receive, send):
    """Обработка WebSocket соединения."""
    # Принимаем соединение
    await send({'type': 'websocket.accept'})

    while True:
        message = await receive()

        if message['type'] == 'websocket.receive':
            text = message.get('text', '')
            # Эхо-ответ
            await send({
                'type': 'websocket.send',
                'text': f'Echo: {text}'
            })

        elif message['type'] == 'websocket.disconnect':
            break

async def handle_lifespan(scope, receive, send):
    """Обработка событий жизненного цикла."""
    while True:
        message = await receive()

        if message['type'] == 'lifespan.startup':
            # Инициализация ресурсов
            print("Application starting...")
            await send({'type': 'lifespan.startup.complete'})

        elif message['type'] == 'lifespan.shutdown':
            # Очистка ресурсов
            print("Application shutting down...")
            await send({'type': 'lifespan.shutdown.complete'})
            return
```

### Практический пример: ASGI Router

```python
import json
import re
from typing import Callable, Dict, List, Tuple
from urllib.parse import parse_qs

class ASGIRouter:
    """Простой ASGI роутер с поддержкой параметров URL."""

    def __init__(self):
        self.routes: List[Tuple[str, str, Callable, re.Pattern]] = []
        self.startup_handlers: List[Callable] = []
        self.shutdown_handlers: List[Callable] = []

    def route(self, path: str, methods: List[str] = ['GET']):
        """Декоратор для регистрации маршрута."""
        # Преобразуем {param} в именованные группы регулярного выражения
        pattern = re.sub(r'\{(\w+)\}', r'(?P<\1>[^/]+)', path)
        pattern = re.compile(f'^{pattern}$')

        def decorator(func: Callable):
            for method in methods:
                self.routes.append((method, path, func, pattern))
            return func
        return decorator

    def on_startup(self, func: Callable):
        """Регистрация обработчика startup."""
        self.startup_handlers.append(func)
        return func

    def on_shutdown(self, func: Callable):
        """Регистрация обработчика shutdown."""
        self.shutdown_handlers.append(func)
        return func

    async def __call__(self, scope, receive, send):
        if scope['type'] == 'lifespan':
            await self._handle_lifespan(scope, receive, send)
        elif scope['type'] == 'http':
            await self._handle_http(scope, receive, send)
        elif scope['type'] == 'websocket':
            await self._handle_websocket(scope, receive, send)

    async def _handle_lifespan(self, scope, receive, send):
        while True:
            message = await receive()
            if message['type'] == 'lifespan.startup':
                for handler in self.startup_handlers:
                    await handler()
                await send({'type': 'lifespan.startup.complete'})
            elif message['type'] == 'lifespan.shutdown':
                for handler in self.shutdown_handlers:
                    await handler()
                await send({'type': 'lifespan.shutdown.complete'})
                return

    async def _handle_http(self, scope, receive, send):
        method = scope['method']
        path = scope['path']

        # Поиск подходящего маршрута
        for route_method, _, handler, pattern in self.routes:
            if route_method != method:
                continue

            match = pattern.match(path)
            if match:
                # Создаем объект запроса
                request = await self._build_request(scope, receive, match)
                response = await handler(request)
                await self._send_response(send, response)
                return

        # 404 Not Found
        await self._send_response(send, {
            'status': 404,
            'body': {'error': 'Not Found'}
        })

    async def _build_request(self, scope, receive, match) -> dict:
        """Создание объекта запроса."""
        body = b''
        more_body = True
        while more_body:
            message = await receive()
            body += message.get('body', b'')
            more_body = message.get('more_body', False)

        return {
            'method': scope['method'],
            'path': scope['path'],
            'query_params': parse_qs(scope['query_string'].decode()),
            'path_params': match.groupdict(),
            'headers': dict(scope['headers']),
            'body': body,
            'json': json.loads(body) if body else None,
        }

    async def _send_response(self, send, response: dict):
        """Отправка HTTP ответа."""
        status = response.get('status', 200)
        body = response.get('body', '')

        if isinstance(body, dict):
            body = json.dumps(body).encode()
            content_type = b'application/json'
        elif isinstance(body, str):
            body = body.encode()
            content_type = b'text/plain'
        else:
            content_type = b'application/octet-stream'

        await send({
            'type': 'http.response.start',
            'status': status,
            'headers': [[b'content-type', content_type]],
        })
        await send({
            'type': 'http.response.body',
            'body': body,
        })

# Использование
app = ASGIRouter()

@app.on_startup
async def startup():
    print("Application started!")

@app.on_shutdown
async def shutdown():
    print("Application stopped!")

@app.route('/')
async def index(request):
    return {'body': {'message': 'Welcome to ASGI!'}}

@app.route('/users/{user_id}', methods=['GET'])
async def get_user(request):
    user_id = request['path_params']['user_id']
    return {'body': {'user_id': user_id}}

@app.route('/users', methods=['POST'])
async def create_user(request):
    data = request['json']
    return {'status': 201, 'body': {'created': data}}
```

## Сравнение WSGI и ASGI

### Таблица сравнения

| Характеристика | WSGI | ASGI |
|---------------|------|------|
| Модель выполнения | Синхронная | Асинхронная |
| HTTP/1.1 | Да | Да |
| HTTP/2 | Нет | Да |
| WebSocket | Нет | Да |
| Server-Sent Events | Сложно | Да |
| Long Polling | Неэффективно | Да |
| Concurrency модель | Потоки/процессы | Event loop |
| Масштабируемость | Линейная | Высокая |
| Память на соединение | Высокая (поток) | Низкая (корутина) |

### Производительность

```python
# WSGI - каждый запрос блокирует поток
import time

def wsgi_slow_endpoint(environ, start_response):
    # Симуляция медленной операции I/O
    time.sleep(1)  # Блокирует весь поток!

    start_response('200 OK', [('Content-Type', 'text/plain')])
    return [b'Done']

# ASGI - корутина освобождает event loop
import asyncio

async def asgi_slow_endpoint(scope, receive, send):
    # Асинхронное ожидание
    await asyncio.sleep(1)  # Не блокирует!

    await send({
        'type': 'http.response.start',
        'status': 200,
        'headers': [[b'content-type', b'text/plain']],
    })
    await send({
        'type': 'http.response.body',
        'body': b'Done',
    })
```

## ASGI серверы

### Uvicorn

```bash
# Установка
pip install uvicorn

# Запуск
uvicorn myapp:app --host 0.0.0.0 --port 8000 --workers 4
```

```python
# Программный запуск
import uvicorn

if __name__ == "__main__":
    uvicorn.run(
        "myapp:app",
        host="0.0.0.0",
        port=8000,
        reload=True,  # Автоперезагрузка при изменениях
        workers=4,
        log_level="info"
    )
```

### Hypercorn

```bash
pip install hypercorn
hypercorn myapp:app --bind 0.0.0.0:8000 --workers 4
```

### Daphne (для Django Channels)

```bash
pip install daphne
daphne myapp.asgi:application -b 0.0.0.0 -p 8000
```

## Миграция с WSGI на ASGI

### Обертка WSGI в ASGI

```python
from asgiref.wsgi import WsgiToAsgi

# Ваше старое WSGI приложение
def wsgi_app(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/plain')])
    return [b'Hello from WSGI!']

# Оборачиваем в ASGI
asgi_app = WsgiToAsgi(wsgi_app)
```

### Постепенная миграция

```python
async def hybrid_app(scope, receive, send):
    """Гибридное приложение с ASGI и WSGI маршрутами."""
    from asgiref.wsgi import WsgiToAsgi

    path = scope['path']

    # Новые асинхронные эндпоинты
    if path.startswith('/api/v2/'):
        await new_asgi_app(scope, receive, send)

    # Старые синхронные эндпоинты (через обертку)
    else:
        wrapped = WsgiToAsgi(old_wsgi_app)
        await wrapped(scope, receive, send)
```

## Best Practices

### 1. Используйте правильный сервер

```python
# Для разработки
uvicorn myapp:app --reload

# Для продакшена
uvicorn myapp:app --workers 4 --loop uvloop --http httptools
```

### 2. Обрабатывайте все типы scope

```python
async def app(scope, receive, send):
    if scope['type'] not in ('http', 'websocket', 'lifespan'):
        raise ValueError(f"Unknown scope type: {scope['type']}")

    # ...
```

### 3. Правильно обрабатывайте lifespan

```python
async def app(scope, receive, send):
    if scope['type'] == 'lifespan':
        while True:
            message = await receive()
            if message['type'] == 'lifespan.startup':
                try:
                    await startup()
                    await send({'type': 'lifespan.startup.complete'})
                except Exception:
                    await send({'type': 'lifespan.startup.failed'})
                    return
            elif message['type'] == 'lifespan.shutdown':
                await shutdown()
                await send({'type': 'lifespan.shutdown.complete'})
                return
```

## Частые ошибки

1. **Смешивание sync и async кода без обертки**
2. **Игнорирование lifespan событий** - утечки ресурсов
3. **Неправильная обработка тела запроса** - чтение должно быть полным
4. **Блокирующие вызовы в async обработчиках**

---

[prev: 01-rest-aiohttp.md](./01-rest-aiohttp.md) | [next: 03-starlette.md](./03-starlette.md)
