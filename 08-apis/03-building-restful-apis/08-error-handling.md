# Error Handling (Обработка ошибок в REST API)

## Введение

Правильная обработка ошибок — критически важная часть дизайна API. Хорошо спроектированные ошибки помогают разработчикам быстро понять проблему и исправить её, а плохие ошибки приводят к часам отладки и фрустрации.

## HTTP-коды статуса

### Категории кодов:

| Диапазон | Категория | Описание |
|----------|-----------|----------|
| 1xx | Informational | Промежуточные ответы |
| 2xx | Success | Успешные запросы |
| 3xx | Redirection | Перенаправления |
| 4xx | Client Error | Ошибки клиента |
| 5xx | Server Error | Ошибки сервера |

### Успешные ответы (2xx):

| Код | Название | Когда использовать |
|-----|----------|-------------------|
| 200 | OK | GET, PUT, PATCH — успешное выполнение |
| 201 | Created | POST — ресурс создан |
| 202 | Accepted | Запрос принят, обработка в фоне |
| 204 | No Content | DELETE — успешное удаление без тела |

### Ошибки клиента (4xx):

| Код | Название | Когда использовать |
|-----|----------|-------------------|
| 400 | Bad Request | Неверный синтаксис запроса |
| 401 | Unauthorized | Требуется аутентификация |
| 403 | Forbidden | Доступ запрещён (даже с аутентификацией) |
| 404 | Not Found | Ресурс не найден |
| 405 | Method Not Allowed | HTTP-метод не поддерживается |
| 409 | Conflict | Конфликт (например, дублирование) |
| 410 | Gone | Ресурс был удалён навсегда |
| 415 | Unsupported Media Type | Неподдерживаемый Content-Type |
| 422 | Unprocessable Entity | Валидация не пройдена |
| 429 | Too Many Requests | Rate limit превышен |

### Ошибки сервера (5xx):

| Код | Название | Когда использовать |
|-----|----------|-------------------|
| 500 | Internal Server Error | Неожиданная ошибка сервера |
| 502 | Bad Gateway | Ошибка upstream сервера |
| 503 | Service Unavailable | Сервис временно недоступен |
| 504 | Gateway Timeout | Таймаут upstream сервера |

## Структура ошибки

### Минимальный формат:

```json
{
    "error": "validation_error",
    "message": "Email address is invalid"
}
```

### Расширенный формат:

```json
{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "The request contains invalid data",
        "details": [
            {
                "field": "email",
                "message": "Must be a valid email address",
                "value": "not-an-email"
            },
            {
                "field": "age",
                "message": "Must be at least 18",
                "value": 15
            }
        ],
        "request_id": "req_abc123",
        "timestamp": "2024-01-15T10:30:00Z",
        "documentation_url": "https://api.example.com/docs/errors#VALIDATION_ERROR"
    }
}
```

### RFC 7807 формат (Problem Details):

```json
{
    "type": "https://api.example.com/errors/validation-error",
    "title": "Validation Error",
    "status": 422,
    "detail": "One or more fields failed validation",
    "instance": "/api/users/registration",
    "errors": [
        {
            "field": "email",
            "message": "Invalid email format"
        }
    ]
}
```

## Реализация

### Python (FastAPI)

```python
from fastapi import FastAPI, HTTPException, Request, status
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from pydantic import BaseModel, EmailStr, Field
from typing import List, Optional, Any
from datetime import datetime
import uuid
import traceback
import logging

app = FastAPI()
logger = logging.getLogger(__name__)

# Модели ошибок
class ErrorDetail(BaseModel):
    field: Optional[str] = None
    message: str
    value: Optional[Any] = None

class ErrorResponse(BaseModel):
    code: str
    message: str
    details: Optional[List[ErrorDetail]] = None
    request_id: str
    timestamp: str
    documentation_url: Optional[str] = None

# Кастомные исключения
class APIError(Exception):
    def __init__(
        self,
        code: str,
        message: str,
        status_code: int = 400,
        details: List[ErrorDetail] = None
    ):
        self.code = code
        self.message = message
        self.status_code = status_code
        self.details = details or []
        super().__init__(message)

class NotFoundError(APIError):
    def __init__(self, resource: str, resource_id: Any):
        super().__init__(
            code="RESOURCE_NOT_FOUND",
            message=f"{resource} with id '{resource_id}' not found",
            status_code=404
        )

class ValidationError(APIError):
    def __init__(self, details: List[ErrorDetail]):
        super().__init__(
            code="VALIDATION_ERROR",
            message="The request contains invalid data",
            status_code=422,
            details=details
        )

class ConflictError(APIError):
    def __init__(self, message: str):
        super().__init__(
            code="CONFLICT",
            message=message,
            status_code=409
        )

# Middleware для добавления request_id
@app.middleware("http")
async def add_request_id(request: Request, call_next):
    request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
    request.state.request_id = request_id

    response = await call_next(request)
    response.headers["X-Request-ID"] = request_id
    return response

# Обработчики ошибок
@app.exception_handler(APIError)
async def api_error_handler(request: Request, exc: APIError):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": {
                "code": exc.code,
                "message": exc.message,
                "details": [d.dict() for d in exc.details] if exc.details else None,
                "request_id": getattr(request.state, 'request_id', 'unknown'),
                "timestamp": datetime.utcnow().isoformat() + "Z"
            }
        }
    )

@app.exception_handler(RequestValidationError)
async def validation_error_handler(request: Request, exc: RequestValidationError):
    """Преобразуем ошибки валидации Pydantic в наш формат."""
    details = []
    for error in exc.errors():
        field = ".".join(str(loc) for loc in error["loc"][1:])  # Убираем 'body'
        details.append(ErrorDetail(
            field=field,
            message=error["msg"],
            value=error.get("input")
        ))

    return JSONResponse(
        status_code=422,
        content={
            "error": {
                "code": "VALIDATION_ERROR",
                "message": "Request validation failed",
                "details": [d.dict() for d in details],
                "request_id": getattr(request.state, 'request_id', 'unknown'),
                "timestamp": datetime.utcnow().isoformat() + "Z"
            }
        }
    )

@app.exception_handler(Exception)
async def generic_error_handler(request: Request, exc: Exception):
    """Ловим все необработанные исключения."""
    request_id = getattr(request.state, 'request_id', 'unknown')

    # Логируем полную ошибку
    logger.error(
        f"Unhandled exception [request_id={request_id}]: {exc}",
        exc_info=True
    )

    # Возвращаем безопасный ответ (без деталей)
    return JSONResponse(
        status_code=500,
        content={
            "error": {
                "code": "INTERNAL_ERROR",
                "message": "An unexpected error occurred",
                "request_id": request_id,
                "timestamp": datetime.utcnow().isoformat() + "Z"
            }
        }
    )

# Модели данных
class UserCreate(BaseModel):
    name: str = Field(..., min_length=2, max_length=100)
    email: EmailStr
    age: int = Field(..., ge=18, le=120)

# Примеры endpoints с ошибками
users_db = {"1": {"id": "1", "name": "John", "email": "john@example.com"}}

@app.get("/api/users/{user_id}")
async def get_user(user_id: str):
    if user_id not in users_db:
        raise NotFoundError("User", user_id)
    return users_db[user_id]

@app.post("/api/users", status_code=201)
async def create_user(user: UserCreate):
    # Проверка на дубликат email
    for existing in users_db.values():
        if existing["email"] == user.email:
            raise ConflictError(f"User with email '{user.email}' already exists")

    user_id = str(len(users_db) + 1)
    users_db[user_id] = {"id": user_id, **user.dict()}
    return users_db[user_id]

@app.post("/api/users/{user_id}/verify")
async def verify_user(user_id: str):
    if user_id not in users_db:
        raise NotFoundError("User", user_id)

    user = users_db[user_id]
    if user.get("verified"):
        raise APIError(
            code="ALREADY_VERIFIED",
            message="User is already verified",
            status_code=422
        )

    user["verified"] = True
    return user
```

### Node.js (Express)

```javascript
const express = require('express');
const { v4: uuidv4 } = require('uuid');

const app = express();
app.use(express.json());

// Кастомные классы ошибок
class APIError extends Error {
    constructor(code, message, statusCode = 400, details = null) {
        super(message);
        this.code = code;
        this.statusCode = statusCode;
        this.details = details;
        this.name = 'APIError';
    }
}

class NotFoundError extends APIError {
    constructor(resource, resourceId) {
        super(
            'RESOURCE_NOT_FOUND',
            `${resource} with id '${resourceId}' not found`,
            404
        );
    }
}

class ValidationError extends APIError {
    constructor(details) {
        super(
            'VALIDATION_ERROR',
            'The request contains invalid data',
            422,
            details
        );
    }
}

class ConflictError extends APIError {
    constructor(message) {
        super('CONFLICT', message, 409);
    }
}

// Middleware для request ID
app.use((req, res, next) => {
    req.requestId = req.headers['x-request-id'] || uuidv4();
    res.setHeader('X-Request-ID', req.requestId);
    next();
});

// Валидация
const validateUser = (req, res, next) => {
    const errors = [];
    const { name, email, age } = req.body;

    if (!name || name.length < 2) {
        errors.push({
            field: 'name',
            message: 'Name must be at least 2 characters',
            value: name
        });
    }

    if (!email || !email.includes('@')) {
        errors.push({
            field: 'email',
            message: 'Must be a valid email address',
            value: email
        });
    }

    if (age === undefined || age < 18) {
        errors.push({
            field: 'age',
            message: 'Age must be at least 18',
            value: age
        });
    }

    if (errors.length > 0) {
        return next(new ValidationError(errors));
    }

    next();
};

// Данные
const users = new Map([
    ['1', { id: '1', name: 'John', email: 'john@example.com' }]
]);

// Routes
app.get('/api/users/:id', (req, res, next) => {
    const user = users.get(req.params.id);

    if (!user) {
        return next(new NotFoundError('User', req.params.id));
    }

    res.json(user);
});

app.post('/api/users', validateUser, (req, res, next) => {
    const { name, email, age } = req.body;

    // Проверка дубликата
    for (const user of users.values()) {
        if (user.email === email) {
            return next(new ConflictError(`User with email '${email}' already exists`));
        }
    }

    const id = String(users.size + 1);
    const newUser = { id, name, email, age };
    users.set(id, newUser);

    res.status(201).json(newUser);
});

// Error handler (должен быть последним)
app.use((err, req, res, next) => {
    // Логирование
    console.error(`[${req.requestId}] Error:`, err);

    // Если это наша кастомная ошибка
    if (err instanceof APIError) {
        return res.status(err.statusCode).json({
            error: {
                code: err.code,
                message: err.message,
                details: err.details,
                request_id: req.requestId,
                timestamp: new Date().toISOString()
            }
        });
    }

    // Ошибка парсинга JSON
    if (err instanceof SyntaxError && err.status === 400) {
        return res.status(400).json({
            error: {
                code: 'INVALID_JSON',
                message: 'Request body contains invalid JSON',
                request_id: req.requestId,
                timestamp: new Date().toISOString()
            }
        });
    }

    // Неизвестная ошибка
    res.status(500).json({
        error: {
            code: 'INTERNAL_ERROR',
            message: 'An unexpected error occurred',
            request_id: req.requestId,
            timestamp: new Date().toISOString()
        }
    });
});

// 404 handler
app.use((req, res) => {
    res.status(404).json({
        error: {
            code: 'ENDPOINT_NOT_FOUND',
            message: `Cannot ${req.method} ${req.path}`,
            request_id: req.requestId,
            timestamp: new Date().toISOString()
        }
    });
});

app.listen(3000);
```

## Типичные сценарии ошибок

### 1. Ошибки валидации (422)

```json
{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Validation failed",
        "details": [
            {
                "field": "email",
                "message": "Invalid email format",
                "value": "not-an-email"
            },
            {
                "field": "password",
                "message": "Must be at least 8 characters",
                "value": "***"
            }
        ]
    }
}
```

### 2. Ресурс не найден (404)

```json
{
    "error": {
        "code": "USER_NOT_FOUND",
        "message": "User with id '12345' not found",
        "request_id": "req_abc123"
    }
}
```

### 3. Конфликт (409)

```json
{
    "error": {
        "code": "DUPLICATE_RESOURCE",
        "message": "A user with this email already exists",
        "details": {
            "conflicting_field": "email",
            "existing_resource_id": "user_789"
        }
    }
}
```

### 4. Авторизация (401/403)

```json
{
    "error": {
        "code": "INVALID_TOKEN",
        "message": "The access token is invalid or has expired",
        "documentation_url": "https://api.example.com/docs/authentication"
    }
}
```

```json
{
    "error": {
        "code": "INSUFFICIENT_PERMISSIONS",
        "message": "You don't have permission to delete this resource",
        "required_permission": "users:delete"
    }
}
```

### 5. Rate Limit (429)

```json
{
    "error": {
        "code": "RATE_LIMIT_EXCEEDED",
        "message": "Too many requests",
        "retry_after": 60,
        "limit": 100,
        "remaining": 0,
        "reset_at": "2024-01-15T10:31:00Z"
    }
}
```

### 6. Бизнес-логика (422)

```json
{
    "error": {
        "code": "INSUFFICIENT_BALANCE",
        "message": "Cannot complete transaction: insufficient funds",
        "details": {
            "required": 150.00,
            "available": 50.00,
            "currency": "USD"
        }
    }
}
```

## Логирование ошибок

```python
import structlog

logger = structlog.get_logger()

@app.exception_handler(Exception)
async def generic_error_handler(request: Request, exc: Exception):
    request_id = getattr(request.state, 'request_id', 'unknown')

    # Структурированное логирование
    logger.error(
        "unhandled_exception",
        request_id=request_id,
        path=str(request.url.path),
        method=request.method,
        error_type=type(exc).__name__,
        error_message=str(exc),
        traceback=traceback.format_exc()
    )

    return JSONResponse(
        status_code=500,
        content={
            "error": {
                "code": "INTERNAL_ERROR",
                "message": "An unexpected error occurred",
                "request_id": request_id
            }
        }
    )
```

## Примеры из реальных API

### Stripe

```json
{
    "error": {
        "type": "card_error",
        "code": "card_declined",
        "message": "Your card was declined.",
        "param": "source",
        "doc_url": "https://stripe.com/docs/error-codes/card-declined"
    }
}
```

### GitHub

```json
{
    "message": "Validation Failed",
    "errors": [
        {
            "resource": "Issue",
            "field": "title",
            "code": "missing_field"
        }
    ],
    "documentation_url": "https://docs.github.com/rest/reference/issues"
}
```

### Twilio

```json
{
    "code": 21211,
    "message": "The 'To' number is not a valid phone number.",
    "more_info": "https://www.twilio.com/docs/errors/21211",
    "status": 400
}
```

## Best Practices

### Что делать:

1. **Используйте правильные HTTP-коды:**
   - 400 — плохой запрос (синтаксис)
   - 422 — валидация не пройдена
   - 404 — ресурс не найден

2. **Структурированные ошибки:**
   ```json
   { "error": { "code": "...", "message": "...", "details": [...] } }
   ```

3. **Включайте request_id:**
   - Для отладки и поддержки

4. **Не раскрывайте внутренности:**
   ```python
   # Плохо
   {"error": "SQLException: ORA-00942: table or view does not exist"}

   # Хорошо
   {"error": {"code": "INTERNAL_ERROR", "message": "An unexpected error occurred"}}
   ```

5. **Документируйте коды ошибок:**
   - `documentation_url` в ответе

6. **Консистентный формат:**
   - Одинаковая структура для всех ошибок

### Типичные ошибки:

1. **Всегда 200 OK:**
   ```json
   HTTP 200 OK
   {"success": false, "error": "User not found"}  // Неправильно!
   ```

2. **Stack trace в production:**
   - Утечка информации о системе

3. **Непонятные сообщения:**
   ```json
   {"error": "Error occurred"}  // Бесполезно
   ```

4. **Несогласованные форматы:**
   - Разные структуры для разных endpoints

## Заключение

Хорошая обработка ошибок включает:

- **Правильные HTTP-коды** — 4xx для клиента, 5xx для сервера
- **Структурированный формат** — code, message, details
- **Request ID** — для отладки
- **Безопасность** — не раскрывать внутренности
- **Документация** — ссылки на описание ошибок

Инвестиции в качественные ошибки окупаются снижением нагрузки на поддержку и ускорением интеграции.
