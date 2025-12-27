# HTTP Headers (HTTP заголовки)

## Что такое HTTP Headers

**HTTP заголовки (headers)** — это метаинформация, передаваемая между клиентом и сервером вместе с HTTP-запросами и ответами. Заголовки содержат данные о типе контента, авторизации, кэшировании, куках и многом другом.

Формат заголовка:
```
Header-Name: header-value
```

## Типы заголовков

### По направлению

1. **Request Headers** — отправляются клиентом
2. **Response Headers** — отправляются сервером
3. **General Headers** — применимы к обоим направлениям
4. **Entity Headers** — описывают тело сообщения

### По назначению

1. **Representation Headers** — Content-Type, Content-Length
2. **Authentication Headers** — Authorization, WWW-Authenticate
3. **Caching Headers** — Cache-Control, ETag
4. **CORS Headers** — Access-Control-*
5. **Security Headers** — X-Content-Type-Options, CSP

## Основные заголовки запросов

### Host (обязательный в HTTP/1.1)

Указывает доменное имя сервера.

```http
GET /api/users HTTP/1.1
Host: api.example.com
```

```python
import requests

# requests автоматически устанавливает Host
response = requests.get("https://api.example.com/users")
```

### User-Agent

Идентифицирует клиент (браузер, приложение).

```http
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/91.0
```

```python
import requests

# Кастомный User-Agent
response = requests.get(
    "https://api.example.com/users",
    headers={"User-Agent": "MyApp/1.0"}
)
```

### Accept

Указывает, какие типы контента клиент может принять.

```http
Accept: application/json
Accept: text/html, application/xhtml+xml, application/xml;q=0.9, */*;q=0.8
Accept: image/webp, image/png, image/*;q=0.8
```

```python
import requests

# Запрос JSON
response = requests.get(
    "https://api.example.com/users",
    headers={"Accept": "application/json"}
)

# Запрос XML (если сервер поддерживает)
response = requests.get(
    "https://api.example.com/users",
    headers={"Accept": "application/xml"}
)
```

### Accept-Language

Предпочтительные языки для ответа.

```http
Accept-Language: ru-RU, ru;q=0.9, en-US;q=0.8, en;q=0.7
```

### Accept-Encoding

Поддерживаемые алгоритмы сжатия.

```http
Accept-Encoding: gzip, deflate, br
```

### Content-Type

Тип содержимого в теле запроса.

```http
Content-Type: application/json
Content-Type: application/x-www-form-urlencoded
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary
```

```python
import requests

# JSON данные
response = requests.post(
    "https://api.example.com/users",
    json={"name": "John"},  # Автоматически установит Content-Type: application/json
)

# Form данные
response = requests.post(
    "https://api.example.com/login",
    data={"username": "john", "password": "secret"}
    # Content-Type: application/x-www-form-urlencoded
)

# Загрузка файла
with open("file.txt", "rb") as f:
    response = requests.post(
        "https://api.example.com/upload",
        files={"file": f}
        # Content-Type: multipart/form-data
    )
```

### Authorization

Учетные данные для аутентификации.

```http
# Basic Auth
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=

# Bearer Token (JWT)
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# API Key
Authorization: ApiKey your-api-key-here
```

```python
import requests
from requests.auth import HTTPBasicAuth

# Basic Auth
response = requests.get(
    "https://api.example.com/secure",
    auth=HTTPBasicAuth("username", "password")
)

# Bearer Token
response = requests.get(
    "https://api.example.com/secure",
    headers={"Authorization": "Bearer eyJhbGc..."}
)

# API Key в заголовке
response = requests.get(
    "https://api.example.com/data",
    headers={"X-API-Key": "your-api-key"}
)
```

### Cookie

Отправка куков на сервер.

```http
Cookie: session_id=abc123; user_pref=dark_mode
```

### If-None-Match / If-Modified-Since

Условные запросы для кэширования.

```http
If-None-Match: "abc123"
If-Modified-Since: Wed, 15 Jan 2024 10:00:00 GMT
```

## Основные заголовки ответов

### Content-Type

Тип содержимого в теле ответа.

```http
Content-Type: application/json; charset=utf-8
Content-Type: text/html; charset=UTF-8
Content-Type: image/png
```

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse, HTMLResponse

app = FastAPI()

@app.get("/json")
def get_json():
    return {"message": "Hello"}  # Content-Type: application/json

@app.get("/html", response_class=HTMLResponse)
def get_html():
    return "<h1>Hello</h1>"  # Content-Type: text/html
```

### Content-Length

Размер тела ответа в байтах.

```http
Content-Length: 1234
```

### Content-Encoding

Алгоритм сжатия тела ответа.

```http
Content-Encoding: gzip
```

### Set-Cookie

Установка куков на клиенте.

```http
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Strict; Max-Age=3600
Set-Cookie: user_pref=dark_mode; Path=/; Expires=Wed, 15 Jan 2025 10:00:00 GMT
```

```python
from fastapi import FastAPI
from fastapi.responses import Response

app = FastAPI()

@app.post("/login")
def login(response: Response):
    response.set_cookie(
        key="session_id",
        value="abc123",
        httponly=True,
        secure=True,
        samesite="strict",
        max_age=3600
    )
    return {"message": "Logged in"}
```

### Location

URL для перенаправления (с кодами 3xx) или созданного ресурса (с 201).

```http
HTTP/1.1 201 Created
Location: /api/users/123

HTTP/1.1 302 Found
Location: https://example.com/new-page
```

### Cache-Control

Директивы кэширования.

```http
Cache-Control: no-cache
Cache-Control: no-store
Cache-Control: max-age=3600
Cache-Control: public, max-age=86400
Cache-Control: private, max-age=3600
```

```python
from fastapi import FastAPI
from fastapi.responses import Response

app = FastAPI()

@app.get("/static-data")
def get_static_data():
    response = Response(content='{"data": "static"}')
    response.headers["Cache-Control"] = "public, max-age=86400"
    return response

@app.get("/user-data")
def get_user_data():
    response = Response(content='{"data": "private"}')
    response.headers["Cache-Control"] = "private, no-cache"
    return response
```

### ETag

Идентификатор версии ресурса для кэширования.

```http
ETag: "abc123"
ETag: W/"weak-etag"
```

```python
import hashlib
from fastapi import FastAPI, Request
from fastapi.responses import Response

app = FastAPI()

@app.get("/resource")
def get_resource(request: Request):
    data = {"id": 1, "name": "Resource"}
    content = json.dumps(data)

    # Генерируем ETag
    etag = hashlib.md5(content.encode()).hexdigest()

    # Проверяем If-None-Match
    if request.headers.get("If-None-Match") == f'"{etag}"':
        return Response(status_code=304)

    response = Response(content=content, media_type="application/json")
    response.headers["ETag"] = f'"{etag}"'
    return response
```

### WWW-Authenticate

Указывает метод аутентификации при 401 ошибке.

```http
WWW-Authenticate: Bearer realm="api"
WWW-Authenticate: Basic realm="Protected Area"
```

## Заголовки безопасности

### Strict-Transport-Security (HSTS)

Принудительное использование HTTPS.

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

### X-Content-Type-Options

Запрет MIME-sniffing.

```http
X-Content-Type-Options: nosniff
```

### X-Frame-Options

Защита от clickjacking.

```http
X-Frame-Options: DENY
X-Frame-Options: SAMEORIGIN
```

### Content-Security-Policy (CSP)

Политика безопасности контента.

```http
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'
```

### X-XSS-Protection

Защита от XSS (устаревший, но все еще используется).

```http
X-XSS-Protection: 1; mode=block
```

```python
from fastapi import FastAPI
from fastapi.middleware.trustedhost import TrustedHostMiddleware

app = FastAPI()

@app.middleware("http")
async def add_security_headers(request, call_next):
    response = await call_next(request)
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["X-XSS-Protection"] = "1; mode=block"
    response.headers["Strict-Transport-Security"] = "max-age=31536000"
    return response
```

## CORS заголовки

```http
# Ответ сервера
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 86400
Access-Control-Expose-Headers: X-Custom-Header
```

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://example.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## Кастомные заголовки

Кастомные заголовки обычно начинаются с `X-` (хотя это уже не обязательно).

```http
X-Request-ID: uuid-123-456
X-Rate-Limit-Remaining: 99
X-Api-Version: 2.0
```

```python
from fastapi import FastAPI, Request
from fastapi.responses import Response
import uuid

app = FastAPI()

@app.middleware("http")
async def add_request_id(request: Request, call_next):
    request_id = str(uuid.uuid4())
    response = await call_next(request)
    response.headers["X-Request-ID"] = request_id
    return response

@app.get("/users")
def get_users(request: Request, response: Response):
    # Получение кастомного заголовка
    api_version = request.headers.get("X-Api-Version", "1.0")

    # Установка кастомного заголовка
    response.headers["X-Total-Count"] = "100"

    return {"users": [...]}
```

## Работа с заголовками в Python

### Чтение заголовков (клиент)

```python
import requests

response = requests.get("https://api.example.com/users")

# Все заголовки
print(response.headers)

# Конкретный заголовок (регистронезависимый)
content_type = response.headers.get("Content-Type")
content_length = response.headers.get("content-length")

# Заголовки запроса
print(response.request.headers)
```

### Чтение заголовков (сервер)

```python
from fastapi import FastAPI, Header, Request
from typing import Optional

app = FastAPI()

# Через параметр Header
@app.get("/secure")
def secure_endpoint(
    authorization: Optional[str] = Header(None),
    user_agent: Optional[str] = Header(None, alias="User-Agent"),
    x_request_id: Optional[str] = Header(None, alias="X-Request-ID")
):
    return {
        "auth": authorization,
        "user_agent": user_agent,
        "request_id": x_request_id
    }

# Через Request объект
@app.get("/all-headers")
def all_headers(request: Request):
    return dict(request.headers)
```

### Установка заголовков (сервер)

```python
from fastapi import FastAPI
from fastapi.responses import Response, JSONResponse

app = FastAPI()

# Через Response
@app.get("/custom-headers")
def custom_headers(response: Response):
    response.headers["X-Custom"] = "value"
    response.headers["X-Another"] = "another-value"
    return {"message": "Hello"}

# Через JSONResponse
@app.get("/json-headers")
def json_headers():
    return JSONResponse(
        content={"message": "Hello"},
        headers={
            "X-Custom": "value",
            "Cache-Control": "no-cache"
        }
    )
```

## Best Practices

### 1. Всегда устанавливайте Content-Type

```python
# requests делает это автоматически для json=
response = requests.post(url, json=data)

# Явная установка при необходимости
response = requests.post(
    url,
    data=xml_data,
    headers={"Content-Type": "application/xml"}
)
```

### 2. Используйте заголовки безопасности

```python
security_headers = {
    "X-Content-Type-Options": "nosniff",
    "X-Frame-Options": "DENY",
    "X-XSS-Protection": "1; mode=block",
    "Strict-Transport-Security": "max-age=31536000",
    "Content-Security-Policy": "default-src 'self'"
}

@app.middleware("http")
async def add_security_headers(request, call_next):
    response = await call_next(request)
    for header, value in security_headers.items():
        response.headers[header] = value
    return response
```

### 3. Не передавайте секреты в URL

```python
# Плохо
requests.get("https://api.example.com/data?api_key=secret")

# Хорошо
requests.get(
    "https://api.example.com/data",
    headers={"Authorization": "Bearer secret"}
)
```

### 4. Добавляйте Request-ID для трассировки

```python
import uuid

request_id = str(uuid.uuid4())
response = requests.get(
    url,
    headers={"X-Request-ID": request_id}
)

# Для отладки
print(f"Request ID: {request_id}")
```

## Типичные ошибки

1. **Неправильный Content-Type**
   ```python
   # Ошибка: отправка JSON с неправильным Content-Type
   requests.post(url, data='{"key": "value"}')  # Без Content-Type: application/json

   # Правильно
   requests.post(url, json={"key": "value"})
   ```

2. **Забытый charset**
   ```python
   # Может вызвать проблемы с кодировкой
   Content-Type: application/json

   # Лучше
   Content-Type: application/json; charset=utf-8
   ```

3. **Отсутствие заголовков кэширования**
   ```python
   # Всегда указывайте политику кэширования
   Cache-Control: no-store  # для приватных данных
   Cache-Control: public, max-age=3600  # для статических ресурсов
   ```

4. **Передача токенов в URL**
   ```python
   # Небезопасно (попадает в логи, историю)
   /api/data?token=secret

   # Безопасно
   Authorization: Bearer secret
   ```
