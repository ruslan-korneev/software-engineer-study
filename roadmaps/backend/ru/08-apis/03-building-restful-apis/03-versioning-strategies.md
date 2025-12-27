# API Versioning Strategies (Стратегии версионирования API)

## Введение

**Версионирование API** — это практика управления изменениями в API таким образом, чтобы существующие клиенты продолжали работать, пока они не будут готовы к обновлению. Это критически важно для поддержания обратной совместимости и стабильности интеграций.

## Зачем нужно версионирование?

### Проблемы без версионирования:

```
# Исходный API
GET /users/123 → {"id": 123, "name": "John Doe"}

# После изменения (Breaking Change!)
GET /users/123 → {"id": 123, "firstName": "John", "lastName": "Doe"}
```

Клиенты, ожидающие поле `name`, ломаются.

### Типы изменений:

| Тип | Совместимость | Примеры |
|-----|---------------|---------|
| **Non-breaking** | Обратно совместимые | Добавление нового поля, нового endpoint |
| **Breaking** | Несовместимые | Удаление/переименование поля, изменение типа |

## Стратегии версионирования

### 1. URI Path Versioning (Версия в URL)

Самый популярный и наглядный подход.

```
https://api.example.com/v1/users
https://api.example.com/v2/users
```

**Примеры из реальных API:**
- Twitter: `https://api.twitter.com/2/tweets`
- Stripe: `https://api.stripe.com/v1/customers`
- GitHub: `https://api.github.com/` (v3 по умолчанию)

#### Реализация (Python/FastAPI):

```python
from fastapi import FastAPI, APIRouter

app = FastAPI()

# Роутеры для разных версий
v1_router = APIRouter(prefix="/api/v1")
v2_router = APIRouter(prefix="/api/v2")

# V1 API
@v1_router.get("/users/{user_id}")
async def get_user_v1(user_id: int):
    return {
        "id": user_id,
        "name": "John Doe",  # Полное имя в одном поле
        "email": "john@example.com"
    }

# V2 API — разделённое имя
@v2_router.get("/users/{user_id}")
async def get_user_v2(user_id: int):
    return {
        "id": user_id,
        "first_name": "John",
        "last_name": "Doe",
        "email": "john@example.com",
        "created_at": "2024-01-15T10:30:00Z"
    }

app.include_router(v1_router)
app.include_router(v2_router)
```

#### Реализация (Node.js/Express):

```javascript
const express = require('express');
const app = express();

// V1 routes
const v1Router = express.Router();
v1Router.get('/users/:id', (req, res) => {
    res.json({
        id: parseInt(req.params.id),
        name: 'John Doe',
        email: 'john@example.com'
    });
});

// V2 routes
const v2Router = express.Router();
v2Router.get('/users/:id', (req, res) => {
    res.json({
        id: parseInt(req.params.id),
        first_name: 'John',
        last_name: 'Doe',
        email: 'john@example.com',
        created_at: '2024-01-15T10:30:00Z'
    });
});

app.use('/api/v1', v1Router);
app.use('/api/v2', v2Router);

app.listen(3000);
```

**Преимущества:**
- Простота и наглядность
- Легко кэшировать (разные URL)
- Легко тестировать и документировать
- Просто переключаться между версиями

**Недостатки:**
- Нарушает принцип REST (URI должен идентифицировать ресурс, а не его версию)
- Дублирование кода при малых изменениях
- Миграция требует изменения URL

### 2. Query Parameter Versioning (Версия в параметре)

```
https://api.example.com/users?version=1
https://api.example.com/users?api-version=2024-01-15
```

**Примеры из реальных API:**
- Google APIs: `?v=3`
- Amazon AWS: `?Version=2012-11-05`

#### Реализация (Python/FastAPI):

```python
from fastapi import FastAPI, Query
from typing import Optional

app = FastAPI()

@app.get("/api/users/{user_id}")
async def get_user(
    user_id: int,
    version: Optional[int] = Query(1, alias="api-version", ge=1, le=2)
):
    if version == 1:
        return {
            "id": user_id,
            "name": "John Doe",
            "email": "john@example.com"
        }
    else:  # version == 2
        return {
            "id": user_id,
            "first_name": "John",
            "last_name": "Doe",
            "email": "john@example.com"
        }
```

**Преимущества:**
- URI остаётся неизменным
- Опциональный параметр с дефолтом

**Недостатки:**
- Менее очевидно, чем URI
- Сложнее кэшировать
- Query параметры обычно для фильтрации

### 3. Header Versioning (Версия в заголовке)

```http
GET /api/users/123 HTTP/1.1
Host: api.example.com
Accept: application/json
X-API-Version: 2
```

Или через Accept header:

```http
GET /api/users/123 HTTP/1.1
Host: api.example.com
Accept: application/vnd.myapi.v2+json
```

**Примеры из реальных API:**
- GitHub: `Accept: application/vnd.github.v3+json`
- Microsoft Azure: `api-version` header

#### Реализация (Python/FastAPI):

```python
from fastapi import FastAPI, Header, HTTPException
from typing import Optional

app = FastAPI()

@app.get("/api/users/{user_id}")
async def get_user(
    user_id: int,
    x_api_version: Optional[int] = Header(1, alias="X-API-Version")
):
    if x_api_version == 1:
        return {"id": user_id, "name": "John Doe"}
    elif x_api_version == 2:
        return {"id": user_id, "first_name": "John", "last_name": "Doe"}
    else:
        raise HTTPException(
            status_code=400,
            detail=f"Unsupported API version: {x_api_version}"
        )
```

#### Реализация (Node.js/Express):

```javascript
const express = require('express');
const app = express();

// Middleware для определения версии
const versionMiddleware = (req, res, next) => {
    const version = parseInt(req.headers['x-api-version']) || 1;
    req.apiVersion = version;
    next();
};

app.use(versionMiddleware);

app.get('/api/users/:id', (req, res) => {
    const userId = parseInt(req.params.id);

    if (req.apiVersion === 1) {
        res.json({ id: userId, name: 'John Doe' });
    } else if (req.apiVersion === 2) {
        res.json({ id: userId, first_name: 'John', last_name: 'Doe' });
    } else {
        res.status(400).json({
            error: `Unsupported API version: ${req.apiVersion}`
        });
    }
});

app.listen(3000);
```

**Преимущества:**
- Чистый URI
- Соответствует принципам REST
- Гибкость в выборе заголовка

**Недостатки:**
- Менее очевидно для разработчиков
- Сложнее тестировать в браузере
- Сложнее документировать

### 4. Media Type Versioning (Content Negotiation)

Версия указывается в Media Type:

```http
GET /api/users/123 HTTP/1.1
Accept: application/vnd.company.user.v2+json
```

```http
HTTP/1.1 200 OK
Content-Type: application/vnd.company.user.v2+json

{
    "first_name": "John",
    "last_name": "Doe"
}
```

**Примеры из реальных API:**
- GitHub: `application/vnd.github+json`
- JIRA: `application/vnd.jira.v3+json`

#### Реализация:

```python
from fastapi import FastAPI, Header, HTTPException
import re

app = FastAPI()

def parse_accept_version(accept: str) -> int:
    """Извлекает версию из Accept header."""
    # application/vnd.myapi.v2+json -> 2
    match = re.search(r'application/vnd\.myapi\.v(\d+)\+json', accept)
    if match:
        return int(match.group(1))
    return 1  # default version

@app.get("/api/users/{user_id}")
async def get_user(
    user_id: int,
    accept: str = Header("application/json")
):
    version = parse_accept_version(accept)

    response_data = {}
    if version == 1:
        response_data = {"id": user_id, "name": "John Doe"}
    elif version == 2:
        response_data = {"id": user_id, "first_name": "John", "last_name": "Doe"}

    return response_data
```

**Преимущества:**
- Полностью RESTful
- Чистый URI
- Можно версионировать отдельные ресурсы

**Недостатки:**
- Самый сложный в реализации
- Менее интуитивен для разработчиков
- Сложнее тестировать

### 5. Date-based Versioning (Версионирование по дате)

Используется дата вместо номера версии:

```
https://api.example.com/2024-01-15/users
```

Или в заголовке:
```http
X-API-Version: 2024-01-15
```

**Примеры из реальных API:**
- Stripe: `Stripe-Version: 2024-01-15`
- Twilio: `/2010-04-01/Accounts`

```python
from fastapi import FastAPI, Header
from datetime import date
from typing import Optional

app = FastAPI()

# Версии API по датам
API_VERSIONS = {
    "2023-01-01": "v1",
    "2023-06-15": "v1.1",
    "2024-01-15": "v2",
}

def get_effective_version(requested_date: str) -> str:
    """Находит подходящую версию для запрошенной даты."""
    sorted_dates = sorted(API_VERSIONS.keys(), reverse=True)
    for version_date in sorted_dates:
        if requested_date >= version_date:
            return API_VERSIONS[version_date]
    return "v1"  # fallback

@app.get("/api/users/{user_id}")
async def get_user(
    user_id: int,
    stripe_version: Optional[str] = Header(None, alias="Stripe-Version")
):
    version_date = stripe_version or "2024-01-15"
    effective_version = get_effective_version(version_date)

    if effective_version == "v1":
        return {"id": user_id, "name": "John Doe"}
    else:
        return {"id": user_id, "first_name": "John", "last_name": "Doe"}
```

**Преимущества:**
- Чёткая хронология изменений
- Легко понять, когда появились изменения
- Хорошо для финансовых/регулируемых API

**Недостатки:**
- Сложнее запомнить даты
- Нужна документация по датам

## Сравнение стратегий

| Стратегия | Простота | RESTful | Кэширование | Тестирование |
|-----------|----------|---------|-------------|--------------|
| URI Path | ★★★★★ | ★★☆☆☆ | ★★★★★ | ★★★★★ |
| Query Param | ★★★★☆ | ★★★☆☆ | ★★★☆☆ | ★★★★☆ |
| Header | ★★★☆☆ | ★★★★☆ | ★★★★☆ | ★★★☆☆ |
| Media Type | ★★☆☆☆ | ★★★★★ | ★★★★☆ | ★★☆☆☆ |
| Date-based | ★★★☆☆ | ★★★☆☆ | ★★★★☆ | ★★★☆☆ |

## Управление версиями на практике

### Deprecation Policy (Политика устаревания)

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 01 Jan 2025 00:00:00 GMT
Link: <https://api.example.com/v2/users>; rel="successor-version"
```

#### Пример реализации:

```python
from fastapi import FastAPI, Response
from datetime import datetime

app = FastAPI()

@app.get("/api/v1/users/{user_id}")
async def get_user_v1(user_id: int, response: Response):
    # Добавляем заголовки deprecation
    response.headers["Deprecation"] = "true"
    response.headers["Sunset"] = "Sat, 01 Jan 2025 00:00:00 GMT"
    response.headers["Link"] = '</api/v2/users>; rel="successor-version"'

    return {"id": user_id, "name": "John Doe"}
```

### Changelog и миграция

Документируйте изменения между версиями:

```markdown
## API Changelog

### v2.0.0 (2024-01-15)
#### Breaking Changes
- `name` поле разделено на `first_name` и `last_name`
- Удалено поле `legacy_id`

#### New Features
- Добавлено поле `created_at`
- Добавлен endpoint `/users/{id}/preferences`

### Migration Guide
Для миграции с v1 на v2:
1. Замените `user.name` на `user.first_name + ' ' + user.last_name`
2. Удалите использование `user.legacy_id`
```

### Поддержка нескольких версий

```python
from fastapi import FastAPI, Depends
from typing import Callable

app = FastAPI()

# Фабрика для версионированных обработчиков
def versioned_endpoint(handlers: dict) -> Callable:
    async def endpoint(version: int, **kwargs):
        handler = handlers.get(version, handlers.get(1))
        return await handler(**kwargs)
    return endpoint

# Обработчики для разных версий
async def get_user_v1(user_id: int):
    return {"id": user_id, "name": "John Doe"}

async def get_user_v2(user_id: int):
    return {"id": user_id, "first_name": "John", "last_name": "Doe"}

# Регистрация
user_handlers = {
    1: get_user_v1,
    2: get_user_v2
}
```

## Best Practices

### Что делать:

1. **Версионируйте с самого начала:**
   - Даже если пока одна версия, начните с `/v1/`

2. **Документируйте изменения:**
   - Changelog для каждой версии
   - Гайды по миграции

3. **Предупреждайте о deprecation:**
   - Minimum 6-12 месяцев до удаления версии
   - Заголовки `Deprecation` и `Sunset`

4. **Поддерживайте N-1 версию:**
   - Как минимум предыдущую версию

5. **Избегайте breaking changes:**
   - Добавляйте новые поля вместо переименования
   - Делайте новые поля опциональными

### Типичные ошибки:

1. **Отсутствие версионирования:**
   - Breaking changes ломают клиентов

2. **Слишком много версий:**
   - Сложно поддерживать 5+ активных версий

3. **Breaking changes без новой версии:**
   - Переименование полей в той же версии

4. **Внезапное удаление версий:**
   - Без предупреждения и периода deprecation

## Примеры из реальных API

### Stripe (Date-based + Header)

```http
POST /v1/customers HTTP/1.1
Stripe-Version: 2024-01-15
Authorization: Bearer sk_test_...
```

### GitHub (URI + Media Type)

```http
GET /repos/owner/repo HTTP/1.1
Accept: application/vnd.github.v3+json
```

### Google Cloud (URI)

```
https://compute.googleapis.com/compute/v1/projects/{project}/zones
```

## Заключение

Выбор стратегии версионирования зависит от:

- **URI Path** — лучший выбор для большинства случаев (простота, наглядность)
- **Header** — когда важна RESTful чистота
- **Date-based** — для API с частыми инкрементальными изменениями
- **Media Type** — для строгого следования REST

**Рекомендация:** Начните с URI Path versioning (`/v1/`) — это проверенный подход, который используют большинство популярных API.
