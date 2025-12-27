# RFC 7807 - Problem Details for HTTP APIs

## Введение

**RFC 7807** (Problem Details for HTTP APIs) — это стандарт, определяющий структуру для представления ошибок в HTTP API. Он обеспечивает единообразный, машиночитаемый формат для описания проблем.

**Официальное название:** "Problem Details for HTTP APIs"
**Статус:** Proposed Standard (март 2016)
**Обновление:** RFC 9457 (июль 2023) — уточнения и расширения

## Зачем нужен стандарт?

### Проблема: разнобой в форматах ошибок

```json
// API 1
{"error": "User not found"}

// API 2
{"message": "User not found", "code": 404}

// API 3
{"errors": [{"msg": "User not found"}]}

// API 4
{"success": false, "data": null, "error_message": "User not found"}
```

Каждый API изобретает свой формат, что усложняет:
- Написание универсальных клиентов
- Обработку ошибок
- Документирование
- Интеграцию

### Решение: единый стандарт RFC 7807

```json
{
    "type": "https://api.example.com/errors/user-not-found",
    "title": "User Not Found",
    "status": 404,
    "detail": "User with ID 12345 was not found in the system",
    "instance": "/api/users/12345"
}
```

## Структура Problem Details

### Обязательные и опциональные поля

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| `type` | URI | Нет* | URI, идентифицирующий тип проблемы |
| `title` | string | Нет | Краткое человекочитаемое описание |
| `status` | integer | Нет | HTTP-код статуса |
| `detail` | string | Нет | Подробное описание конкретного случая |
| `instance` | URI | Нет | URI конкретного экземпляра проблемы |

*Если `type` не указан, используется `about:blank`

### Расширяемость

Можно добавлять собственные поля:

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
            "message": "Invalid email format",
            "value": "not-an-email"
        },
        {
            "field": "age",
            "message": "Must be at least 18",
            "value": 15
        }
    ],
    "trace_id": "abc123",
    "timestamp": "2024-01-15T10:30:00Z"
}
```

## Примеры Problem Details

### Ресурс не найден (404)

```json
{
    "type": "https://api.example.com/errors/resource-not-found",
    "title": "Resource Not Found",
    "status": 404,
    "detail": "User with ID '12345' does not exist",
    "instance": "/api/users/12345"
}
```

### Ошибка валидации (422)

```json
{
    "type": "https://api.example.com/errors/validation-error",
    "title": "Validation Error",
    "status": 422,
    "detail": "The submitted data failed validation",
    "instance": "/api/users",
    "errors": [
        {
            "pointer": "/data/attributes/email",
            "message": "Email is already taken"
        },
        {
            "pointer": "/data/attributes/password",
            "message": "Password must be at least 8 characters"
        }
    ]
}
```

### Ошибка авторизации (403)

```json
{
    "type": "https://api.example.com/errors/forbidden",
    "title": "Forbidden",
    "status": 403,
    "detail": "You do not have permission to delete this resource",
    "instance": "/api/posts/789",
    "required_permission": "posts:delete",
    "user_permissions": ["posts:read", "posts:write"]
}
```

### Rate Limit (429)

```json
{
    "type": "https://api.example.com/errors/rate-limit-exceeded",
    "title": "Rate Limit Exceeded",
    "status": 429,
    "detail": "You have exceeded the rate limit of 100 requests per minute",
    "instance": "/api/search",
    "limit": 100,
    "remaining": 0,
    "reset": "2024-01-15T10:31:00Z",
    "retry_after": 45
}
```

### Внутренняя ошибка (500)

```json
{
    "type": "https://api.example.com/errors/internal-error",
    "title": "Internal Server Error",
    "status": 500,
    "detail": "An unexpected error occurred while processing your request",
    "instance": "/api/orders/create",
    "trace_id": "abc-123-def-456",
    "support_url": "https://support.example.com/?trace=abc-123-def-456"
}
```

### Бизнес-ошибка

```json
{
    "type": "https://api.example.com/errors/insufficient-funds",
    "title": "Insufficient Funds",
    "status": 422,
    "detail": "Your account balance is insufficient for this transaction",
    "instance": "/api/transactions/transfer",
    "balance": {
        "current": 50.00,
        "required": 150.00,
        "currency": "USD"
    }
}
```

## Content-Type

RFC 7807 определяет специальные media types:

- `application/problem+json` — для JSON
- `application/problem+xml` — для XML

```http
HTTP/1.1 404 Not Found
Content-Type: application/problem+json

{
    "type": "https://api.example.com/errors/not-found",
    "title": "Not Found",
    "status": 404,
    "detail": "The requested resource was not found"
}
```

## Реализация

### Python (FastAPI)

```python
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from pydantic import BaseModel
from typing import Optional, List, Any
from datetime import datetime
import uuid

app = FastAPI()

# Модели Problem Details
class ProblemDetail(BaseModel):
    type: str = "about:blank"
    title: str
    status: int
    detail: Optional[str] = None
    instance: Optional[str] = None

    class Config:
        extra = "allow"  # Разрешаем дополнительные поля

class ValidationErrorDetail(ProblemDetail):
    errors: List[dict]

# Кастомные исключения
class APIException(Exception):
    def __init__(
        self,
        type_uri: str,
        title: str,
        status: int,
        detail: str = None,
        instance: str = None,
        **kwargs
    ):
        self.type_uri = type_uri
        self.title = title
        self.status = status
        self.detail = detail
        self.instance = instance
        self.extra = kwargs
        super().__init__(detail or title)

class NotFoundException(APIException):
    def __init__(self, resource: str, resource_id: Any, instance: str = None):
        super().__init__(
            type_uri="https://api.example.com/errors/not-found",
            title="Resource Not Found",
            status=404,
            detail=f"{resource} with ID '{resource_id}' was not found",
            instance=instance,
            resource_type=resource,
            resource_id=str(resource_id)
        )

class ValidationException(APIException):
    def __init__(self, errors: List[dict], instance: str = None):
        super().__init__(
            type_uri="https://api.example.com/errors/validation-error",
            title="Validation Error",
            status=422,
            detail="One or more fields failed validation",
            instance=instance,
            errors=errors
        )

class ForbiddenException(APIException):
    def __init__(self, detail: str, required_permission: str = None, instance: str = None):
        super().__init__(
            type_uri="https://api.example.com/errors/forbidden",
            title="Forbidden",
            status=403,
            detail=detail,
            instance=instance,
            required_permission=required_permission
        )

class RateLimitException(APIException):
    def __init__(self, limit: int, reset_at: datetime, instance: str = None):
        super().__init__(
            type_uri="https://api.example.com/errors/rate-limit-exceeded",
            title="Rate Limit Exceeded",
            status=429,
            detail=f"You have exceeded the rate limit of {limit} requests",
            instance=instance,
            limit=limit,
            remaining=0,
            reset=reset_at.isoformat() + "Z"
        )

# Обработчики исключений
@app.exception_handler(APIException)
async def api_exception_handler(request: Request, exc: APIException):
    problem = {
        "type": exc.type_uri,
        "title": exc.title,
        "status": exc.status,
        "detail": exc.detail,
        "instance": exc.instance or str(request.url.path),
        **exc.extra
    }

    # Удаляем None значения
    problem = {k: v for k, v in problem.items() if v is not None}

    return JSONResponse(
        status_code=exc.status,
        content=problem,
        media_type="application/problem+json"
    )

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    errors = []
    for error in exc.errors():
        loc = "/".join(str(l) for l in error["loc"])
        errors.append({
            "pointer": f"/{loc}",
            "message": error["msg"],
            "value": error.get("input")
        })

    problem = {
        "type": "https://api.example.com/errors/validation-error",
        "title": "Validation Error",
        "status": 422,
        "detail": "The request body contains invalid data",
        "instance": str(request.url.path),
        "errors": errors
    }

    return JSONResponse(
        status_code=422,
        content=problem,
        media_type="application/problem+json"
    )

@app.exception_handler(Exception)
async def generic_exception_handler(request: Request, exc: Exception):
    trace_id = str(uuid.uuid4())

    # Логируем полную ошибку
    import logging
    logging.error(f"[{trace_id}] Unhandled exception: {exc}", exc_info=True)

    problem = {
        "type": "https://api.example.com/errors/internal-error",
        "title": "Internal Server Error",
        "status": 500,
        "detail": "An unexpected error occurred",
        "instance": str(request.url.path),
        "trace_id": trace_id
    }

    return JSONResponse(
        status_code=500,
        content=problem,
        media_type="application/problem+json"
    )

# Примеры использования
@app.get("/api/users/{user_id}")
async def get_user(user_id: str):
    users = {"1": {"id": "1", "name": "John"}}

    if user_id not in users:
        raise NotFoundException("User", user_id)

    return users[user_id]

@app.delete("/api/posts/{post_id}")
async def delete_post(post_id: str):
    # Проверка прав
    raise ForbiddenException(
        detail="You do not have permission to delete this post",
        required_permission="posts:delete"
    )
```

### Node.js (Express)

```javascript
const express = require('express');
const { v4: uuidv4 } = require('uuid');

const app = express();
app.use(express.json());

// Базовый класс для Problem Details ошибок
class ProblemDetail extends Error {
    constructor({
        type = 'about:blank',
        title,
        status,
        detail,
        instance,
        ...extra
    }) {
        super(detail || title);
        this.type = type;
        this.title = title;
        this.status = status;
        this.detail = detail;
        this.instance = instance;
        this.extra = extra;
    }

    toJSON() {
        const problem = {
            type: this.type,
            title: this.title,
            status: this.status,
            detail: this.detail,
            instance: this.instance,
            ...this.extra
        };

        // Удаляем undefined значения
        return Object.fromEntries(
            Object.entries(problem).filter(([_, v]) => v !== undefined)
        );
    }
}

// Специализированные ошибки
class NotFoundError extends ProblemDetail {
    constructor(resource, resourceId, instance) {
        super({
            type: 'https://api.example.com/errors/not-found',
            title: 'Resource Not Found',
            status: 404,
            detail: `${resource} with ID '${resourceId}' was not found`,
            instance,
            resource_type: resource,
            resource_id: String(resourceId)
        });
    }
}

class ValidationError extends ProblemDetail {
    constructor(errors, instance) {
        super({
            type: 'https://api.example.com/errors/validation-error',
            title: 'Validation Error',
            status: 422,
            detail: 'One or more fields failed validation',
            instance,
            errors
        });
    }
}

class ForbiddenError extends ProblemDetail {
    constructor(detail, requiredPermission, instance) {
        super({
            type: 'https://api.example.com/errors/forbidden',
            title: 'Forbidden',
            status: 403,
            detail,
            instance,
            required_permission: requiredPermission
        });
    }
}

class RateLimitError extends ProblemDetail {
    constructor(limit, resetAt, instance) {
        super({
            type: 'https://api.example.com/errors/rate-limit-exceeded',
            title: 'Rate Limit Exceeded',
            status: 429,
            detail: `You have exceeded the rate limit of ${limit} requests`,
            instance,
            limit,
            remaining: 0,
            reset: resetAt.toISOString()
        });
    }
}

// Middleware для добавления instance
app.use((req, res, next) => {
    req.problemInstance = req.originalUrl;
    next();
});

// Примеры routes
const users = new Map([
    ['1', { id: '1', name: 'John' }]
]);

app.get('/api/users/:id', (req, res, next) => {
    const user = users.get(req.params.id);

    if (!user) {
        return next(new NotFoundError('User', req.params.id, req.problemInstance));
    }

    res.json(user);
});

app.post('/api/users', (req, res, next) => {
    const { name, email, age } = req.body;
    const errors = [];

    if (!name || name.length < 2) {
        errors.push({
            pointer: '/name',
            message: 'Name must be at least 2 characters'
        });
    }

    if (!email || !email.includes('@')) {
        errors.push({
            pointer: '/email',
            message: 'Invalid email format'
        });
    }

    if (age !== undefined && age < 18) {
        errors.push({
            pointer: '/age',
            message: 'Must be at least 18 years old',
            value: age
        });
    }

    if (errors.length > 0) {
        return next(new ValidationError(errors, req.problemInstance));
    }

    // Создание пользователя...
    res.status(201).json({ id: '2', name, email, age });
});

app.delete('/api/posts/:id', (req, res, next) => {
    next(new ForbiddenError(
        'You do not have permission to delete this post',
        'posts:delete',
        req.problemInstance
    ));
});

// Error handler (должен быть последним)
app.use((err, req, res, next) => {
    // Если это ProblemDetail ошибка
    if (err instanceof ProblemDetail) {
        return res
            .status(err.status)
            .type('application/problem+json')
            .json(err.toJSON());
    }

    // Ошибка парсинга JSON
    if (err instanceof SyntaxError && err.status === 400) {
        return res
            .status(400)
            .type('application/problem+json')
            .json({
                type: 'https://api.example.com/errors/invalid-json',
                title: 'Invalid JSON',
                status: 400,
                detail: 'The request body contains invalid JSON',
                instance: req.originalUrl
            });
    }

    // Неизвестная ошибка
    const traceId = uuidv4();
    console.error(`[${traceId}] Unhandled error:`, err);

    res
        .status(500)
        .type('application/problem+json')
        .json({
            type: 'https://api.example.com/errors/internal-error',
            title: 'Internal Server Error',
            status: 500,
            detail: 'An unexpected error occurred',
            instance: req.originalUrl,
            trace_id: traceId
        });
});

// 404 handler
app.use((req, res) => {
    res
        .status(404)
        .type('application/problem+json')
        .json({
            type: 'https://api.example.com/errors/endpoint-not-found',
            title: 'Endpoint Not Found',
            status: 404,
            detail: `Cannot ${req.method} ${req.path}`,
            instance: req.originalUrl
        });
});

app.listen(3000);
```

## Type URI как документация

URI в поле `type` должен вести к документации:

```
https://api.example.com/errors/validation-error
```

Страница по этому адресу должна объяснять:
- Что означает эта ошибка
- Возможные причины
- Как исправить
- Примеры

### Пример страницы документации:

```markdown
# Validation Error

## Description
This error occurs when the submitted data fails validation rules.

## HTTP Status
422 Unprocessable Entity

## Common Causes
- Missing required fields
- Invalid email format
- Value out of allowed range
- Duplicate unique field

## Response Structure
```json
{
    "type": "https://api.example.com/errors/validation-error",
    "title": "Validation Error",
    "status": 422,
    "detail": "One or more fields failed validation",
    "errors": [
        {
            "pointer": "/email",
            "message": "Invalid email format",
            "value": "not-an-email"
        }
    ]
}
```

## How to Fix
1. Check the `errors` array for specific field errors
2. Correct the invalid values
3. Retry the request
```

## Примеры из реальных API

### Zalando (Problem Details)

```json
{
    "type": "https://api.zalando.com/errors/out-of-stock",
    "title": "Out of Stock",
    "status": 409,
    "detail": "Item 'SKU-123' is no longer available",
    "instance": "/orders/123"
}
```

### Microsoft Azure

```json
{
    "error": {
        "code": "ResourceNotFound",
        "message": "The Resource 'xyz' under resource group 'rg1' was not found.",
        "details": []
    }
}
```
(Не полностью RFC 7807, но похожий подход)

### Atlassian

```json
{
    "type": "https://developer.atlassian.com/cloud/jira/platform/rest/v3/intro/#about",
    "title": "Forbidden",
    "status": 403,
    "detail": "You do not have permission to view this issue"
}
```

## RFC 9457 — обновление RFC 7807

В июле 2023 вышел RFC 9457, который уточняет и расширяет RFC 7807:

### Изменения:
1. Уточнение работы с `about:blank`
2. Рекомендации по безопасности
3. Уточнения для JSON-сериализации
4. Регистрация типов проблем

```json
{
    "type": "about:blank",
    "title": "Not Found",
    "status": 404
}
```

Когда `type` = `about:blank`, `title` должен соответствовать HTTP reason phrase.

## Best Practices

### Что делать:

1. **Используйте полные URI для type:**
   ```json
   "type": "https://api.example.com/errors/validation-error"
   ```

2. **Документируйте типы ошибок:**
   - Создайте страницы для каждого URI

3. **Включайте полезные расширения:**
   ```json
   {
       "trace_id": "abc123",
       "timestamp": "2024-01-15T10:30:00Z",
       "errors": [...]
   }
   ```

4. **Используйте правильный Content-Type:**
   ```
   Content-Type: application/problem+json
   ```

5. **Локализуйте title и detail:**
   - Используйте Accept-Language header

6. **Не раскрывайте внутренности:**
   - Убирайте stack traces
   - Скрывайте детали реализации

### Типичные ошибки:

1. **Относительные URI:**
   ```json
   "type": "/errors/not-found"  // Плохо
   "type": "https://api.example.com/errors/not-found"  // Хорошо
   ```

2. **Неинформативные сообщения:**
   ```json
   "detail": "An error occurred"  // Бесполезно
   ```

3. **Несоответствие status и HTTP-кода:**
   ```
   HTTP/1.1 200 OK
   {"status": 400, ...}  // Плохо!
   ```

4. **Отсутствие instance:**
   - Сложнее отлаживать

## Заключение

RFC 7807 предоставляет стандартный способ представления ошибок:

- **type** — URI типа проблемы (со ссылкой на документацию)
- **title** — краткое описание
- **status** — HTTP-код
- **detail** — подробности конкретного случая
- **instance** — URI конкретного запроса

Использование стандарта упрощает:
- Разработку клиентов
- Обработку ошибок
- Документирование API
- Интеграцию с мониторингом

**Рекомендация:** Внедряйте RFC 7807 с самого начала разработки API — это инвестиция в качество и удобство использования.
