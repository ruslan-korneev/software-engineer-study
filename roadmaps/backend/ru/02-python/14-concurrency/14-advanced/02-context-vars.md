# Контекстные переменные (Context Variables)

[prev: ./01-awaitable-api.md](./01-awaitable-api.md) | [next: ./03-force-iteration.md](./03-force-iteration.md)

---

## Введение

**Контекстные переменные** (Context Variables) — это механизм Python для хранения данных, специфичных для текущего контекста выполнения. В asyncio контекстные переменные особенно важны, так как они позволяют изолировать данные между разными асинхронными задачами, подобно тому, как `threading.local()` работает для потоков.

Модуль `contextvars` был добавлен в Python 3.7 специально для решения проблемы передачи контекста в асинхронном коде.

## Проблема: почему нужны контекстные переменные?

### Проблема с глобальными переменными

```python
import asyncio

# ПРОБЛЕМА: глобальная переменная общая для всех задач
current_user = None

async def process_request(user_id: int):
    global current_user
    current_user = user_id
    await asyncio.sleep(0.1)  # Имитация I/O операции
    # К этому моменту current_user может быть перезаписан другой задачей!
    print(f"Обрабатываем запрос для пользователя: {current_user}")

async def main():
    # Запускаем несколько задач параллельно
    await asyncio.gather(
        process_request(1),
        process_request(2),
        process_request(3),
    )
    # Вывод будет непредсказуемым!

asyncio.run(main())
```

### Решение с контекстными переменными

```python
import asyncio
from contextvars import ContextVar

# Контекстная переменная — изолирована для каждой задачи
current_user: ContextVar[int] = ContextVar('current_user', default=0)

async def process_request(user_id: int):
    current_user.set(user_id)
    await asyncio.sleep(0.1)
    # Значение остаётся правильным для этой задачи!
    print(f"Обрабатываем запрос для пользователя: {current_user.get()}")

async def main():
    await asyncio.gather(
        process_request(1),
        process_request(2),
        process_request(3),
    )
    # Каждая задача видит своё значение!

asyncio.run(main())
```

## Основы работы с ContextVar

### Создание и использование

```python
from contextvars import ContextVar

# Создание переменной с значением по умолчанию
request_id: ContextVar[str] = ContextVar('request_id', default='unknown')

# Создание переменной без значения по умолчанию
user_token: ContextVar[str] = ContextVar('user_token')

# Получение значения
print(request_id.get())  # 'unknown'

try:
    print(user_token.get())  # LookupError!
except LookupError:
    print("Значение не установлено")

# Получение с fallback
print(user_token.get("default_token"))  # 'default_token'

# Установка значения
request_id.set("req-123")
print(request_id.get())  # 'req-123'
```

### Токены для восстановления значения

Метод `set()` возвращает **токен**, который можно использовать для восстановления предыдущего значения:

```python
from contextvars import ContextVar

counter: ContextVar[int] = ContextVar('counter', default=0)

# Устанавливаем значение и сохраняем токен
token = counter.set(10)
print(counter.get())  # 10

counter.set(20)
print(counter.get())  # 20

# Восстанавливаем значение по токену
counter.reset(token)
print(counter.get())  # 0 (значение по умолчанию, т.к. до set(10) не было значения)
```

## Контекст и копирование

### Работа с объектом Context

```python
from contextvars import ContextVar, copy_context

user_id: ContextVar[int] = ContextVar('user_id')
request_id: ContextVar[str] = ContextVar('request_id')

# Устанавливаем значения
user_id.set(42)
request_id.set("req-abc")

# Копируем текущий контекст
ctx = copy_context()

# Контекст можно использовать для запуска функций
def print_context():
    print(f"User: {user_id.get()}, Request: {request_id.get()}")

# Запуск в скопированном контексте
ctx.run(print_context)  # User: 42, Request: req-abc

# Изменения в ctx.run не влияют на внешний контекст
def modify_context():
    user_id.set(100)
    print(f"Внутри: {user_id.get()}")

ctx.run(modify_context)  # Внутри: 100
print(f"Снаружи: {user_id.get()}")  # Снаружи: 42
```

## Контекстные переменные в asyncio

### Автоматическое копирование контекста

В asyncio каждая задача (`Task`) автоматически получает **копию** текущего контекста при создании:

```python
import asyncio
from contextvars import ContextVar

current_task_name: ContextVar[str] = ContextVar('task_name')

async def child_task():
    # Дочерняя задача видит значение из родительского контекста
    print(f"Дочерняя задача видит: {current_task_name.get()}")

    # Изменения не влияют на родителя
    current_task_name.set("child")
    print(f"После изменения: {current_task_name.get()}")

async def parent_task():
    current_task_name.set("parent")

    # Создаём дочернюю задачу
    await asyncio.create_task(child_task())

    # Значение в родителе не изменилось
    print(f"Родитель после: {current_task_name.get()}")

asyncio.run(parent_task())
# Вывод:
# Дочерняя задача видит: parent
# После изменения: child
# Родитель после: parent
```

### Практический пример: Request Context

```python
import asyncio
from contextvars import ContextVar
from dataclasses import dataclass
from typing import Optional
import uuid

@dataclass
class RequestContext:
    request_id: str
    user_id: Optional[int]
    client_ip: str

# Контекстная переменная для хранения контекста запроса
_request_context: ContextVar[RequestContext] = ContextVar('request_context')

def get_request_context() -> RequestContext:
    """Получить текущий контекст запроса."""
    try:
        return _request_context.get()
    except LookupError:
        raise RuntimeError("Нет активного контекста запроса")

def set_request_context(ctx: RequestContext) -> None:
    """Установить контекст запроса."""
    _request_context.set(ctx)

async def log_message(message: str):
    """Логирование с автоматическим добавлением request_id."""
    ctx = get_request_context()
    print(f"[{ctx.request_id}] {message}")

async def get_user_data():
    """Получение данных пользователя."""
    ctx = get_request_context()
    await asyncio.sleep(0.1)  # Имитация запроса к БД
    await log_message(f"Получены данные для пользователя {ctx.user_id}")
    return {"user_id": ctx.user_id, "name": "Test User"}

async def handle_request(user_id: int, client_ip: str):
    """Обработка HTTP запроса."""
    # Устанавливаем контекст в начале обработки
    ctx = RequestContext(
        request_id=str(uuid.uuid4())[:8],
        user_id=user_id,
        client_ip=client_ip
    )
    set_request_context(ctx)

    await log_message(f"Начало обработки запроса от {client_ip}")

    # Все вложенные вызовы имеют доступ к контексту
    user_data = await get_user_data()

    await log_message(f"Запрос обработан успешно")
    return user_data

async def main():
    # Параллельная обработка нескольких запросов
    results = await asyncio.gather(
        handle_request(1, "192.168.1.1"),
        handle_request(2, "192.168.1.2"),
        handle_request(3, "192.168.1.3"),
    )
    print(f"\nРезультаты: {results}")

asyncio.run(main())
```

## Передача контекста в executor

При использовании `run_in_executor` контекст нужно передавать явно:

```python
import asyncio
from contextvars import ContextVar, copy_context
from concurrent.futures import ThreadPoolExecutor

request_id: ContextVar[str] = ContextVar('request_id')

def sync_operation():
    """Синхронная операция в потоке."""
    # Контекст автоматически передаётся в Python 3.11+
    print(f"Request ID в потоке: {request_id.get()}")

async def main():
    request_id.set("req-123")

    loop = asyncio.get_running_loop()

    # Способ 1: Использовать контекст напрямую (Python 3.11+)
    # Asyncio автоматически копирует контекст
    await loop.run_in_executor(None, sync_operation)

    # Способ 2: Явная передача через copy_context (совместимо со всеми версиями)
    ctx = copy_context()
    await loop.run_in_executor(None, lambda: ctx.run(sync_operation))

asyncio.run(main())
```

## Интеграция с веб-фреймворками

### Пример с aiohttp-подобной структурой

```python
import asyncio
from contextvars import ContextVar
from typing import Callable, Any

# Контекстные переменные для веб-приложения
current_request: ContextVar[dict] = ContextVar('current_request')
current_user: ContextVar[dict] = ContextVar('current_user')

class Middleware:
    """Базовый класс middleware."""

    async def __call__(self, request: dict, handler: Callable) -> Any:
        return await handler(request)

class RequestContextMiddleware(Middleware):
    """Middleware для установки контекста запроса."""

    async def __call__(self, request: dict, handler: Callable) -> Any:
        current_request.set(request)
        return await handler(request)

class AuthMiddleware(Middleware):
    """Middleware для аутентификации."""

    async def __call__(self, request: dict, handler: Callable) -> Any:
        # Имитация проверки токена
        token = request.get('headers', {}).get('Authorization')
        if token:
            user = {"id": 1, "name": "John"}  # Имитация
            current_user.set(user)
        return await handler(request)

async def my_handler(request: dict) -> dict:
    """Обработчик запроса."""
    req = current_request.get()
    try:
        user = current_user.get()
        return {"status": "ok", "user": user['name'], "path": req['path']}
    except LookupError:
        return {"status": "error", "message": "Unauthorized"}

async def process_with_middleware(request: dict, handler: Callable):
    """Обработка запроса через цепочку middleware."""
    middlewares = [RequestContextMiddleware(), AuthMiddleware()]

    # Строим цепочку middleware
    async def final_handler(req):
        return await handler(req)

    chain = final_handler
    for middleware in reversed(middlewares):
        prev_chain = chain
        chain = lambda req, m=middleware, h=prev_chain: m(req, h)

    return await chain(request)

async def main():
    # Имитация запроса
    request = {
        'path': '/api/user',
        'headers': {'Authorization': 'Bearer token123'}
    }

    result = await process_with_middleware(request, my_handler)
    print(f"Результат: {result}")

asyncio.run(main())
```

## Best Practices

### 1. Используйте типизацию

```python
from contextvars import ContextVar
from typing import Optional

# Всегда указывайте тип
user_id: ContextVar[Optional[int]] = ContextVar('user_id', default=None)
```

### 2. Создавайте обёртки для доступа

```python
from contextvars import ContextVar

_db_session: ContextVar['Session'] = ContextVar('db_session')

def get_db() -> 'Session':
    """Безопасное получение сессии БД."""
    try:
        return _db_session.get()
    except LookupError:
        raise RuntimeError("Сессия БД не инициализирована")

def set_db(session: 'Session') -> None:
    """Установка сессии БД."""
    _db_session.set(session)
```

### 3. Используйте контекстные менеджеры

```python
from contextvars import ContextVar, Token
from contextlib import contextmanager
from typing import Generator

_trace_id: ContextVar[str] = ContextVar('trace_id')

@contextmanager
def trace_context(trace_id: str) -> Generator[None, None, None]:
    """Контекстный менеджер для трейсинга."""
    token: Token[str] = _trace_id.set(trace_id)
    try:
        yield
    finally:
        _trace_id.reset(token)

# Использование
with trace_context("trace-123"):
    print(f"Trace ID: {_trace_id.get()}")
```

## Распространённые ошибки

### 1. Забыли установить значение

```python
from contextvars import ContextVar

value: ContextVar[int] = ContextVar('value')

# ОШИБКА: LookupError
try:
    print(value.get())
except LookupError:
    print("Значение не установлено!")

# ПРАВИЛЬНО: используйте default
print(value.get(0))
```

### 2. Модификация mutable объектов

```python
from contextvars import ContextVar

# ОСТОРОЖНО с mutable объектами
data: ContextVar[list] = ContextVar('data', default=[])

# ПРОБЛЕМА: все задачи будут использовать один и тот же список!
data.get().append(1)

# ПРАВИЛЬНО: создавайте новый объект
data.set([1])
```

## Резюме

- Контекстные переменные решают проблему передачи данных в асинхронном коде
- Каждая задача asyncio получает копию контекста при создании
- Используйте `ContextVar` вместо глобальных переменных в async-коде
- Токены позволяют восстанавливать предыдущие значения
- `copy_context()` создаёт снимок текущего контекста
- Контекстные переменные идеальны для request_id, user context, database sessions

---

[prev: ./01-awaitable-api.md](./01-awaitable-api.md) | [next: ./03-force-iteration.md](./03-force-iteration.md)
