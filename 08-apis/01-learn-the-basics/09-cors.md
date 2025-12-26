# CORS (Cross-Origin Resource Sharing)

## Что такое CORS

**CORS (Cross-Origin Resource Sharing)** — это механизм безопасности браузера, который контролирует доступ к ресурсам с другого домена (origin). CORS позволяет серверам указывать, какие источники могут получать доступ к их ресурсам.

## Same-Origin Policy (SOP)

Браузеры по умолчанию применяют политику одного источника (Same-Origin Policy), которая запрещает JavaScript делать запросы к другому origin.

### Что такое Origin

Origin состоит из трех частей: **схема + хост + порт**.

```
https://example.com:443
|_____|  |________| |__|
схема      хост     порт
```

### Примеры same-origin и cross-origin

```
Базовый URL: https://example.com

https://example.com/page      - same-origin (только путь отличается)
https://example.com:443/page  - same-origin (443 - дефолтный для https)

http://example.com            - cross-origin (другая схема)
https://api.example.com       - cross-origin (другой хост)
https://example.com:8080      - cross-origin (другой порт)
https://example.org           - cross-origin (другой домен)
```

## Как работает CORS

### Простые запросы (Simple Requests)

Простые запросы отправляются сразу, без preflight. Условия:
- Метод: GET, HEAD, POST
- Заголовки: только "простые" (Accept, Accept-Language, Content-Language, Content-Type)
- Content-Type: text/plain, multipart/form-data, application/x-www-form-urlencoded

```
Браузер                           Сервер
   |                                 |
   |-- GET /api/users -------------->|
   |   Origin: https://frontend.com  |
   |                                 |
   |<--- 200 OK ---------------------|
   |   Access-Control-Allow-Origin:  |
   |   https://frontend.com          |
```

### Preflight запросы

Непростые запросы требуют предварительного OPTIONS-запроса.

```
Браузер                              Сервер
   |                                    |
   |-- OPTIONS /api/users ------------->|
   |   Origin: https://frontend.com     |
   |   Access-Control-Request-Method:   |
   |     DELETE                         |
   |   Access-Control-Request-Headers:  |
   |     Authorization                  |
   |                                    |
   |<--- 204 No Content ----------------|
   |   Access-Control-Allow-Origin:     |
   |     https://frontend.com           |
   |   Access-Control-Allow-Methods:    |
   |     GET, POST, PUT, DELETE         |
   |   Access-Control-Allow-Headers:    |
   |     Authorization                  |
   |   Access-Control-Max-Age: 86400    |
   |                                    |
   |-- DELETE /api/users/123 ---------->|
   |   Origin: https://frontend.com     |
   |   Authorization: Bearer token      |
   |                                    |
   |<--- 200 OK ------------------------|
```

## CORS заголовки

### Заголовки запроса

| Заголовок | Описание |
|-----------|----------|
| `Origin` | Источник запроса |
| `Access-Control-Request-Method` | Метод будущего запроса (preflight) |
| `Access-Control-Request-Headers` | Заголовки будущего запроса (preflight) |

### Заголовки ответа

| Заголовок | Описание |
|-----------|----------|
| `Access-Control-Allow-Origin` | Разрешенные источники |
| `Access-Control-Allow-Methods` | Разрешенные методы |
| `Access-Control-Allow-Headers` | Разрешенные заголовки |
| `Access-Control-Allow-Credentials` | Разрешить cookies/auth |
| `Access-Control-Expose-Headers` | Заголовки, доступные JS |
| `Access-Control-Max-Age` | Время кэширования preflight |

## Настройка CORS в FastAPI

### Базовая настройка

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# Разрешить все источники (для разработки)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### Настройка для production

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# Конкретные источники
origins = [
    "https://myapp.com",
    "https://www.myapp.com",
    "https://admin.myapp.com",
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE", "PATCH"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
    expose_headers=["X-Total-Count", "X-Request-ID"],
    max_age=600,  # Кэширование preflight на 10 минут
)
```

### Настройка с регулярными выражениями

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origin_regex=r"https://.*\.myapp\.com",  # Все поддомены
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### Динамическая настройка CORS

```python
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from starlette.middleware.base import BaseHTTPMiddleware

ALLOWED_ORIGINS = {
    "https://app1.com",
    "https://app2.com",
}

class DynamicCORSMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        origin = request.headers.get("origin")

        response = await call_next(request)

        if origin in ALLOWED_ORIGINS:
            response.headers["Access-Control-Allow-Origin"] = origin
            response.headers["Access-Control-Allow-Credentials"] = "true"
            response.headers["Access-Control-Allow-Methods"] = "GET, POST, PUT, DELETE"
            response.headers["Access-Control-Allow-Headers"] = "Authorization, Content-Type"

        return response

app = FastAPI()
app.add_middleware(DynamicCORSMiddleware)
```

## CORS с credentials

При отправке cookies или Authorization заголовков:

### Сервер

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com"],  # НЕ "*" при credentials!
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### Клиент (JavaScript)

```javascript
// fetch
fetch("https://api.example.com/data", {
    credentials: "include",  // Отправлять cookies
    headers: {
        "Authorization": "Bearer token"
    }
});

// axios
axios.get("https://api.example.com/data", {
    withCredentials: true
});
```

> **Важно**: При `allow_credentials=True` нельзя использовать `allow_origins=["*"]`. Нужно указать конкретные домены.

## CORS и Preflight

### Когда отправляется preflight

Preflight (OPTIONS запрос) отправляется, если:
1. Метод не GET, HEAD или POST
2. POST с Content-Type не из списка простых
3. Есть кастомные заголовки

```python
# Этот запрос вызовет preflight
fetch("https://api.example.com/users", {
    method: "PUT",
    headers: {
        "Content-Type": "application/json",
        "Authorization": "Bearer token"
    },
    body: JSON.stringify({ name: "John" })
});
```

### Обработка preflight в FastAPI

FastAPI middleware автоматически обрабатывает OPTIONS:

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com"],
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
    max_age=600,  # Кэширование preflight
)
```

### Ручная обработка OPTIONS

```python
from fastapi import FastAPI, Request
from fastapi.responses import Response

app = FastAPI()

@app.options("/{path:path}")
async def options_handler(request: Request):
    origin = request.headers.get("origin", "")

    return Response(
        status_code=204,
        headers={
            "Access-Control-Allow-Origin": origin,
            "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE, OPTIONS",
            "Access-Control-Allow-Headers": "Authorization, Content-Type",
            "Access-Control-Max-Age": "86400",
        }
    )
```

## Expose Headers

По умолчанию JavaScript имеет доступ только к "простым" заголовкам ответа. Для доступа к кастомным заголовкам нужен `Access-Control-Expose-Headers`.

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com"],
    expose_headers=["X-Total-Count", "X-Page", "X-RateLimit-Remaining"],
)
```

```javascript
// Теперь JS может читать эти заголовки
const response = await fetch("https://api.example.com/users");
console.log(response.headers.get("X-Total-Count"));  // Работает
```

## Отладка CORS

### Проверка в браузере

```javascript
// Ошибка CORS в консоли:
// Access to fetch at 'https://api.example.com' from origin 'https://myapp.com'
// has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header
// is present on the requested resource.
```

### Проверка с curl

```bash
# Проверка простого запроса
curl -I -X GET https://api.example.com/users \
  -H "Origin: https://myapp.com"

# Проверка preflight
curl -I -X OPTIONS https://api.example.com/users \
  -H "Origin: https://myapp.com" \
  -H "Access-Control-Request-Method: DELETE" \
  -H "Access-Control-Request-Headers: Authorization"
```

### Логирование CORS

```python
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware
import logging

logger = logging.getLogger(__name__)

class CORSLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        origin = request.headers.get("origin")

        if request.method == "OPTIONS":
            logger.info(f"Preflight request from: {origin}")
            logger.info(f"Requested method: {request.headers.get('access-control-request-method')}")
            logger.info(f"Requested headers: {request.headers.get('access-control-request-headers')}")

        response = await call_next(request)

        logger.info(f"CORS headers in response: {dict(response.headers)}")

        return response

app = FastAPI()
app.add_middleware(CORSLoggingMiddleware)
app.add_middleware(CORSMiddleware, ...)  # CORSMiddleware после логирования
```

## Типичные сценарии

### SPA + API на разных доменах

```python
# Backend: api.myapp.com
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com", "https://www.myapp.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### Разработка (localhost)

```python
# Для локальной разработки
if settings.DEBUG:
    app.add_middleware(
        CORSMiddleware,
        allow_origins=["http://localhost:3000", "http://127.0.0.1:3000"],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
```

### Публичный API

```python
# API доступен всем
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["GET"],  # Только чтение
    allow_headers=["X-API-Key"],  # Для API ключа
)
```

### Микросервисы

```python
# Внутренние сервисы
INTERNAL_ORIGINS = [
    "https://service1.internal.com",
    "https://service2.internal.com",
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=INTERNAL_ORIGINS,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## Best Practices

### 1. Никогда не используйте `*` с credentials

```python
# Плохо (не работает)
allow_origins=["*"],
allow_credentials=True

# Хорошо
allow_origins=["https://myapp.com"],
allow_credentials=True
```

### 2. Минимизируйте разрешенные origins

```python
# Плохо (слишком открыто)
allow_origins=["*"]

# Хорошо (только нужные)
allow_origins=["https://myapp.com", "https://admin.myapp.com"]
```

### 3. Используйте max_age для preflight

```python
# Кэширование preflight уменьшает количество OPTIONS запросов
app.add_middleware(
    CORSMiddleware,
    max_age=600,  # 10 минут
)
```

### 4. Ограничивайте методы и заголовки

```python
# Плохо
allow_methods=["*"],
allow_headers=["*"]

# Хорошо
allow_methods=["GET", "POST", "PUT", "DELETE"],
allow_headers=["Authorization", "Content-Type"]
```

### 5. Не забывайте про expose_headers

```python
# Если клиент должен читать кастомные заголовки
app.add_middleware(
    CORSMiddleware,
    expose_headers=["X-Total-Count", "X-Request-ID"]
)
```

## Типичные ошибки

### 1. CORS на неправильном уровне

```python
# Плохо - CORS только на одном endpoint
@app.get("/users")
async def get_users(response: Response):
    response.headers["Access-Control-Allow-Origin"] = "*"
    return users

# Хорошо - middleware для всего приложения
app.add_middleware(CORSMiddleware, ...)
```

### 2. Забытый preflight

```python
# Ошибка: обрабатывается только GET, OPTIONS вернет 405
@app.get("/data")
async def get_data():
    return data

# Правильно: middleware автоматически обрабатывает OPTIONS
app.add_middleware(CORSMiddleware, ...)
```

### 3. Неправильный порядок middleware

```python
# Важен порядок! CORS middleware должен быть последним добавленным
# (первым в цепочке обработки)
app.add_middleware(SomeMiddleware)
app.add_middleware(CORSMiddleware, ...)  # Последний добавленный = первый обработчик
```

### 4. CORS на прокси/балансировщике

```nginx
# nginx может переопределять CORS заголовки
# Убедитесь, что заголовки проходят через прокси

location /api {
    proxy_pass http://backend;
    # Передаем заголовки от backend
    proxy_pass_header Access-Control-Allow-Origin;
}
```

### 5. Путаница с HTTP и HTTPS

```python
# Плохо - смешивание протоколов
allow_origins=["http://myapp.com"]  # А сайт на https

# Хорошо
allow_origins=["https://myapp.com"]
```
