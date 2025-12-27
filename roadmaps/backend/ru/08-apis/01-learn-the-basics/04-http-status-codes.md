# HTTP Status Codes (Коды состояния HTTP)

## Обзор

HTTP-коды состояния — это трехзначные числа, которые сервер возвращает в ответе, чтобы сообщить клиенту о результате обработки запроса. Коды делятся на 5 классов:

| Диапазон | Класс | Описание |
|----------|-------|----------|
| 1xx | Informational | Информационные |
| 2xx | Success | Успешные |
| 3xx | Redirection | Перенаправление |
| 4xx | Client Error | Ошибки клиента |
| 5xx | Server Error | Ошибки сервера |

## 1xx — Информационные

Промежуточные ответы, указывающие, что запрос получен и обрабатывается.

### 100 Continue

Сервер получил заголовки запроса и клиент может продолжить отправку тела.

```http
# Запрос
POST /upload HTTP/1.1
Host: example.com
Content-Length: 10000000
Expect: 100-continue

# Ответ сервера
HTTP/1.1 100 Continue

# Клиент продолжает отправку тела
```

### 101 Switching Protocols

Сервер переключается на другой протокол (например, WebSocket).

```http
# Запрос WebSocket handshake
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade

# Ответ
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
```

## 2xx — Успешные

Запрос был успешно получен, понят и обработан.

### 200 OK

Стандартный успешный ответ.

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/users")
def get_users():
    return {"users": [...]}  # Автоматически вернет 200 OK
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"users": [...]}
```

### 201 Created

Ресурс успешно создан. Используется с POST.

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse

@app.post("/users", status_code=201)
def create_user(user: UserCreate):
    new_user = save_to_db(user)
    return new_user

# Или с заголовком Location
@app.post("/users")
def create_user(user: UserCreate):
    new_user = save_to_db(user)
    return JSONResponse(
        status_code=201,
        content=new_user.dict(),
        headers={"Location": f"/users/{new_user.id}"}
    )
```

```http
HTTP/1.1 201 Created
Location: /users/123
Content-Type: application/json

{"id": 123, "name": "John"}
```

### 202 Accepted

Запрос принят, но еще не обработан. Используется для асинхронных операций.

```python
@app.post("/reports", status_code=202)
def generate_report(params: ReportParams):
    task_id = background_tasks.add_task(generate_report_task, params)
    return {
        "task_id": task_id,
        "status_url": f"/reports/status/{task_id}"
    }
```

```http
HTTP/1.1 202 Accepted
Content-Type: application/json

{
    "task_id": "abc123",
    "status_url": "/reports/status/abc123"
}
```

### 204 No Content

Успешно, но нет содержимого для возврата. Типично для DELETE.

```python
@app.delete("/users/{user_id}", status_code=204)
def delete_user(user_id: int):
    delete_from_db(user_id)
    return None  # Тело ответа пустое
```

```http
HTTP/1.1 204 No Content
```

### 206 Partial Content

Частичное содержимое. Используется для загрузки файлов по частям.

```http
# Запрос части файла
GET /video.mp4 HTTP/1.1
Host: example.com
Range: bytes=0-999

# Ответ
HTTP/1.1 206 Partial Content
Content-Range: bytes 0-999/10000
Content-Length: 1000

[первые 1000 байт]
```

## 3xx — Перенаправление

Клиент должен предпринять дополнительные действия для завершения запроса.

### 301 Moved Permanently

Ресурс перемещен навсегда. Поисковики обновят ссылки.

```python
from fastapi.responses import RedirectResponse

@app.get("/old-path")
def old_endpoint():
    return RedirectResponse(
        url="/new-path",
        status_code=301
    )
```

```http
HTTP/1.1 301 Moved Permanently
Location: /new-path
```

### 302 Found (Temporary Redirect)

Временное перенаправление. Старый URL может снова работать.

```python
@app.get("/temp-redirect")
def temp_redirect():
    return RedirectResponse(url="/new-location", status_code=302)
```

### 303 See Other

Перенаправление на другой ресурс методом GET.

```python
@app.post("/orders")
def create_order(order: Order):
    new_order = save_order(order)
    return RedirectResponse(
        url=f"/orders/{new_order.id}",
        status_code=303
    )
```

### 304 Not Modified

Ресурс не изменился. Клиент может использовать кэшированную версию.

```http
# Запрос с условием
GET /resource HTTP/1.1
If-None-Match: "abc123"

# Ответ (ресурс не изменился)
HTTP/1.1 304 Not Modified
ETag: "abc123"
```

### 307 Temporary Redirect

Временное перенаправление с сохранением метода запроса.

```python
# Разница между 302 и 307:
# 302 - браузеры могут изменить POST на GET
# 307 - метод запроса сохраняется

@app.post("/submit")
def submit():
    return RedirectResponse(url="/process", status_code=307)
    # POST запрос будет перенаправлен как POST
```

### 308 Permanent Redirect

Постоянное перенаправление с сохранением метода запроса.

## 4xx — Ошибки клиента

Запрос содержит ошибку со стороны клиента.

### 400 Bad Request

Некорректный запрос. Синтаксическая ошибка или невалидные данные.

```python
from fastapi import HTTPException

@app.post("/users")
def create_user(user: UserCreate):
    if not is_valid_email(user.email):
        raise HTTPException(
            status_code=400,
            detail={
                "error": "INVALID_EMAIL",
                "message": "Email has invalid format"
            }
        )
```

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
    "error": "INVALID_EMAIL",
    "message": "Email has invalid format"
}
```

### 401 Unauthorized

Требуется аутентификация. Клиент не идентифицирован.

```python
@app.get("/protected")
def protected_resource(authorization: str = Header(None)):
    if not authorization:
        raise HTTPException(
            status_code=401,
            detail="Authentication required",
            headers={"WWW-Authenticate": "Bearer"}
        )
```

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer

{"detail": "Authentication required"}
```

### 403 Forbidden

Доступ запрещен. Клиент идентифицирован, но не имеет прав.

```python
@app.delete("/users/{user_id}")
def delete_user(user_id: int, current_user: User = Depends(get_current_user)):
    if not current_user.is_admin:
        raise HTTPException(
            status_code=403,
            detail="You don't have permission to delete users"
        )
```

```http
HTTP/1.1 403 Forbidden

{"detail": "You don't have permission to delete users"}
```

### 404 Not Found

Ресурс не найден.

```python
@app.get("/users/{user_id}")
def get_user(user_id: int):
    user = db.get_user(user_id)
    if not user:
        raise HTTPException(
            status_code=404,
            detail=f"User with id {user_id} not found"
        )
    return user
```

```http
HTTP/1.1 404 Not Found

{"detail": "User with id 123 not found"}
```

### 405 Method Not Allowed

Метод не поддерживается для данного ресурса.

```http
# Запрос
DELETE /users HTTP/1.1

# Ответ
HTTP/1.1 405 Method Not Allowed
Allow: GET, POST
```

### 406 Not Acceptable

Сервер не может вернуть ответ в запрошенном формате.

```http
# Запрос
GET /users HTTP/1.1
Accept: application/xml

# Ответ (сервер поддерживает только JSON)
HTTP/1.1 406 Not Acceptable
```

### 408 Request Timeout

Истекло время ожидания запроса.

### 409 Conflict

Конфликт с текущим состоянием ресурса.

```python
@app.post("/users")
def create_user(user: UserCreate):
    if db.user_exists(user.email):
        raise HTTPException(
            status_code=409,
            detail="User with this email already exists"
        )
```

```http
HTTP/1.1 409 Conflict

{"detail": "User with this email already exists"}
```

### 410 Gone

Ресурс был удален навсегда.

```python
@app.get("/old-api/users")
def deprecated_endpoint():
    raise HTTPException(
        status_code=410,
        detail="This API version is no longer available. Use /v2/users"
    )
```

### 413 Payload Too Large

Тело запроса слишком большое.

```python
from fastapi import UploadFile

@app.post("/upload")
def upload_file(file: UploadFile):
    if file.size > 10 * 1024 * 1024:  # 10 MB
        raise HTTPException(
            status_code=413,
            detail="File too large. Maximum size is 10MB"
        )
```

### 415 Unsupported Media Type

Неподдерживаемый тип содержимого.

```http
# Запрос с неподдерживаемым Content-Type
POST /api/users HTTP/1.1
Content-Type: text/plain

# Ответ
HTTP/1.1 415 Unsupported Media Type
```

### 422 Unprocessable Entity

Синтаксически корректный запрос, но семантически неверный. Часто используется FastAPI для ошибок валидации.

```python
from pydantic import BaseModel, validator

class UserCreate(BaseModel):
    age: int

    @validator('age')
    def age_must_be_positive(cls, v):
        if v < 0:
            raise ValueError('Age must be positive')
        return v

# Запрос с age: -5 вернет 422
```

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json

{
    "detail": [
        {
            "loc": ["body", "age"],
            "msg": "Age must be positive",
            "type": "value_error"
        }
    ]
}
```

### 429 Too Many Requests

Слишком много запросов. Rate limiting.

```python
from fastapi import HTTPException
import time

request_counts = {}

def rate_limit(client_ip: str, limit: int = 100):
    current_minute = int(time.time() / 60)
    key = f"{client_ip}:{current_minute}"

    request_counts[key] = request_counts.get(key, 0) + 1

    if request_counts[key] > limit:
        raise HTTPException(
            status_code=429,
            detail="Too many requests",
            headers={"Retry-After": "60"}
        )
```

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60

{"detail": "Too many requests"}
```

## 5xx — Ошибки сервера

Сервер не смог выполнить корректный запрос.

### 500 Internal Server Error

Общая ошибка сервера.

```python
@app.get("/users")
def get_users():
    try:
        return db.get_all_users()
    except DatabaseError as e:
        # Логируем ошибку, но не показываем детали клиенту
        logger.error(f"Database error: {e}")
        raise HTTPException(
            status_code=500,
            detail="Internal server error"
        )
```

```http
HTTP/1.1 500 Internal Server Error

{"detail": "Internal server error"}
```

### 501 Not Implemented

Функциональность не реализована.

```python
@app.patch("/users/{user_id}")
def update_user(user_id: int):
    raise HTTPException(
        status_code=501,
        detail="PATCH method is not implemented yet"
    )
```

### 502 Bad Gateway

Прокси/шлюз получил некорректный ответ от upstream-сервера.

### 503 Service Unavailable

Сервис временно недоступен (перегрузка, обслуживание).

```python
maintenance_mode = True

@app.get("/")
def root():
    if maintenance_mode:
        raise HTTPException(
            status_code=503,
            detail="Service is under maintenance",
            headers={"Retry-After": "3600"}
        )
```

```http
HTTP/1.1 503 Service Unavailable
Retry-After: 3600

{"detail": "Service is under maintenance"}
```

### 504 Gateway Timeout

Upstream-сервер не ответил вовремя.

## Практическое использование

### Обработка ошибок в API

```python
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

class ErrorResponse(BaseModel):
    error_code: str
    message: str
    details: Optional[dict] = None

class AppException(Exception):
    def __init__(self, status_code: int, error_code: str, message: str, details: dict = None):
        self.status_code = status_code
        self.error_code = error_code
        self.message = message
        self.details = details

@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    return JSONResponse(
        status_code=exc.status_code,
        content=ErrorResponse(
            error_code=exc.error_code,
            message=exc.message,
            details=exc.details
        ).dict()
    )

# Использование
@app.get("/users/{user_id}")
def get_user(user_id: int):
    user = db.get_user(user_id)
    if not user:
        raise AppException(
            status_code=404,
            error_code="USER_NOT_FOUND",
            message=f"User with id {user_id} not found",
            details={"user_id": user_id}
        )
    return user
```

### Проверка кодов ответа в клиенте

```python
import requests

response = requests.get("https://api.example.com/users/123")

if response.status_code == 200:
    user = response.json()
elif response.status_code == 404:
    print("Пользователь не найден")
elif response.status_code == 401:
    print("Требуется аутентификация")
elif response.status_code >= 500:
    print("Ошибка сервера, попробуйте позже")

# Или использовать raise_for_status()
try:
    response.raise_for_status()
except requests.HTTPError as e:
    if e.response.status_code == 404:
        print("Не найдено")
    else:
        raise
```

## Best Practices

### 1. Используйте правильные коды для операций

```python
# POST создание -> 201 Created
@app.post("/users", status_code=201)

# DELETE удаление -> 204 No Content
@app.delete("/users/{id}", status_code=204)

# PUT/PATCH обновление -> 200 OK
@app.put("/users/{id}", status_code=200)
```

### 2. Различайте 401 и 403

```python
# 401 — не аутентифицирован (кто ты?)
# 403 — не авторизован (тебе нельзя)

if not token:
    raise HTTPException(status_code=401)  # Нет токена

if not user.can_delete_posts:
    raise HTTPException(status_code=403)  # Нет прав
```

### 3. Информативные ответы об ошибках

```python
# Плохо
raise HTTPException(status_code=400)

# Хорошо
raise HTTPException(
    status_code=400,
    detail={
        "error_code": "VALIDATION_ERROR",
        "message": "Invalid request data",
        "errors": [
            {"field": "email", "message": "Invalid email format"},
            {"field": "age", "message": "Must be positive"}
        ]
    }
)
```

### 4. Не раскрывайте внутренние ошибки

```python
# Плохо (раскрывает структуру БД)
raise HTTPException(
    status_code=500,
    detail="psycopg2.errors.UniqueViolation: duplicate key value violates unique constraint"
)

# Хорошо
raise HTTPException(
    status_code=409,
    detail="User with this email already exists"
)
```

## Типичные ошибки

1. **Возврат 200 при ошибках** — всегда возвращайте соответствующий код ошибки
2. **404 вместо 403** — не скрывайте существование ресурса через 404
3. **500 для ошибок валидации** — используйте 400 или 422
4. **Игнорирование кодов ответа в клиенте** — всегда проверяйте status_code
5. **Отсутствие Retry-After для 429 и 503** — помогает клиенту понять когда повторить
