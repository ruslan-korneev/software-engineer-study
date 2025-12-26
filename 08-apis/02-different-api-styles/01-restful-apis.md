# RESTful APIs

## Что такое REST?

**REST (Representational State Transfer)** — это архитектурный стиль для разработки распределённых систем, предложенный Роем Филдингом в 2000 году в его докторской диссертации. REST не является протоколом или стандартом — это набор архитектурных принципов и ограничений для построения веб-сервисов.

RESTful API — это API, построенное в соответствии с принципами REST.

## Основные принципы REST

### 1. Client-Server (Клиент-сервер)

Разделение ответственности между клиентом и сервером. Клиент отвечает за пользовательский интерфейс, сервер — за хранение и обработку данных.

```
┌─────────┐     HTTP Request      ┌─────────┐
│ Client  │ ──────────────────▶  │ Server  │
│  (UI)   │ ◀──────────────────  │ (Data)  │
└─────────┘     HTTP Response     └─────────┘
```

### 2. Stateless (Отсутствие состояния)

Каждый запрос от клиента к серверу должен содержать всю необходимую информацию для его обработки. Сервер не хранит состояние клиента между запросами.

```python
# Плохо: сервер хранит состояние
@app.get("/next-page")
def get_next_page():
    # Сервер помнит текущую страницу — нарушение stateless
    session['page'] += 1
    return get_data(session['page'])

# Хорошо: клиент передаёт всю информацию
@app.get("/users")
def get_users(page: int = 1, limit: int = 10):
    # Все параметры приходят с запросом
    return get_data(page, limit)
```

### 3. Cacheable (Кэшируемость)

Ответы сервера должны явно или неявно указывать, могут ли они быть закэшированы.

```python
from fastapi import FastAPI, Response

@app.get("/static-data")
def get_static_data(response: Response):
    response.headers["Cache-Control"] = "max-age=3600, public"
    return {"data": "This can be cached for 1 hour"}

@app.get("/user-balance")
def get_user_balance(response: Response):
    response.headers["Cache-Control"] = "no-cache, no-store, must-revalidate"
    return {"balance": get_current_balance()}
```

### 4. Uniform Interface (Единообразный интерфейс)

Стандартизированный способ взаимодействия между клиентом и сервером:

- **Идентификация ресурсов через URI** — каждый ресурс имеет уникальный адрес
- **Манипуляция ресурсами через представления** — клиент получает представление ресурса (JSON, XML)
- **Самоописывающие сообщения** — сообщение содержит всю информацию для его понимания
- **HATEOAS** — гипермедиа как движок состояния приложения

### 5. Layered System (Слоистая система)

Клиент не знает, обращается ли он напрямую к серверу или к промежуточному узлу (балансировщик, кэш, прокси).

### 6. Code on Demand (Код по требованию) — опционально

Сервер может расширять функциональность клиента, передавая исполняемый код (JavaScript).

## HTTP методы в REST

| Метод | CRUD операция | Идемпотентность | Безопасность | Описание |
|-------|---------------|-----------------|--------------|----------|
| GET | Read | Да | Да | Получение ресурса |
| POST | Create | Нет | Нет | Создание ресурса |
| PUT | Update/Replace | Да | Нет | Полное обновление ресурса |
| PATCH | Update/Modify | Нет | Нет | Частичное обновление |
| DELETE | Delete | Да | Нет | Удаление ресурса |

### Идемпотентность

Операция идемпотентна, если многократное её выполнение даёт тот же результат, что и однократное.

```python
# PUT идемпотентен — повторный вызов даёт тот же результат
PUT /users/123
{"name": "John", "email": "john@example.com"}

# POST не идемпотентен — каждый вызов создаёт новый ресурс
POST /users
{"name": "John"}  # Создаст нового пользователя каждый раз
```

## Проектирование URL (endpoints)

### Правила именования ресурсов

```python
# Используй существительные во множественном числе
GET  /users          # Получить список пользователей
GET  /users/123      # Получить пользователя с id=123
POST /users          # Создать пользователя
PUT  /users/123      # Обновить пользователя
DELETE /users/123    # Удалить пользователя

# Вложенные ресурсы
GET /users/123/orders           # Заказы пользователя 123
GET /users/123/orders/456       # Конкретный заказ пользователя
POST /users/123/orders          # Создать заказ для пользователя

# Фильтрация, сортировка, пагинация через query parameters
GET /users?status=active&sort=created_at&order=desc&page=2&limit=20
```

### Чего следует избегать

```python
# Плохо: глаголы в URL
GET /getUsers
POST /createUser
DELETE /deleteUser/123

# Плохо: действия вместо ресурсов
POST /users/123/activate

# Лучше: использовать вложенный ресурс или PATCH
PATCH /users/123
{"status": "active"}

# Или отдельный ресурс для активации
POST /users/123/activations
```

## Практические примеры кода

### Пример RESTful API на FastAPI

```python
from fastapi import FastAPI, HTTPException, status, Response
from pydantic import BaseModel
from typing import Optional
from datetime import datetime

app = FastAPI()

# Модели данных
class UserCreate(BaseModel):
    name: str
    email: str

class UserUpdate(BaseModel):
    name: Optional[str] = None
    email: Optional[str] = None

class User(BaseModel):
    id: int
    name: str
    email: str
    created_at: datetime

# Имитация базы данных
users_db: dict[int, User] = {}
next_id = 1

# GET /users — получить список пользователей
@app.get("/users", response_model=list[User])
def get_users(
    skip: int = 0,
    limit: int = 10,
    status: Optional[str] = None
):
    users = list(users_db.values())
    if status:
        users = [u for u in users if u.status == status]
    return users[skip:skip + limit]

# GET /users/{id} — получить пользователя по ID
@app.get("/users/{user_id}", response_model=User)
def get_user(user_id: int):
    if user_id not in users_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"User with id {user_id} not found"
        )
    return users_db[user_id]

# POST /users — создать пользователя
@app.post("/users", response_model=User, status_code=status.HTTP_201_CREATED)
def create_user(user: UserCreate, response: Response):
    global next_id
    new_user = User(
        id=next_id,
        name=user.name,
        email=user.email,
        created_at=datetime.now()
    )
    users_db[next_id] = new_user
    response.headers["Location"] = f"/users/{next_id}"
    next_id += 1
    return new_user

# PUT /users/{id} — полное обновление пользователя
@app.put("/users/{user_id}", response_model=User)
def update_user(user_id: int, user: UserCreate):
    if user_id not in users_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"User with id {user_id} not found"
        )
    existing = users_db[user_id]
    updated_user = User(
        id=user_id,
        name=user.name,
        email=user.email,
        created_at=existing.created_at
    )
    users_db[user_id] = updated_user
    return updated_user

# PATCH /users/{id} — частичное обновление
@app.patch("/users/{user_id}", response_model=User)
def partial_update_user(user_id: int, user: UserUpdate):
    if user_id not in users_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"User with id {user_id} not found"
        )
    existing = users_db[user_id]
    update_data = user.model_dump(exclude_unset=True)
    updated_user = existing.model_copy(update=update_data)
    users_db[user_id] = updated_user
    return updated_user

# DELETE /users/{id} — удалить пользователя
@app.delete("/users/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_user(user_id: int):
    if user_id not in users_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"User with id {user_id} not found"
        )
    del users_db[user_id]
    return None
```

## HTTP статус-коды

### Успешные ответы (2xx)

| Код | Название | Использование |
|-----|----------|---------------|
| 200 | OK | GET, PUT, PATCH — успешный запрос |
| 201 | Created | POST — ресурс создан |
| 204 | No Content | DELETE — успешное удаление |

### Ошибки клиента (4xx)

| Код | Название | Использование |
|-----|----------|---------------|
| 400 | Bad Request | Невалидные данные в запросе |
| 401 | Unauthorized | Требуется аутентификация |
| 403 | Forbidden | Доступ запрещён |
| 404 | Not Found | Ресурс не найден |
| 405 | Method Not Allowed | Метод не поддерживается |
| 409 | Conflict | Конфликт (например, дубликат) |
| 422 | Unprocessable Entity | Ошибка валидации |

### Ошибки сервера (5xx)

| Код | Название | Использование |
|-----|----------|---------------|
| 500 | Internal Server Error | Внутренняя ошибка сервера |
| 502 | Bad Gateway | Ошибка прокси/балансировщика |
| 503 | Service Unavailable | Сервис временно недоступен |

## HATEOAS (Hypermedia as the Engine of Application State)

HATEOAS — продвинутый принцип REST, при котором сервер возвращает ссылки на связанные ресурсы и доступные действия.

```python
@app.get("/users/{user_id}")
def get_user_with_links(user_id: int):
    user = users_db[user_id]
    return {
        "data": user,
        "_links": {
            "self": {"href": f"/users/{user_id}"},
            "orders": {"href": f"/users/{user_id}/orders"},
            "update": {"href": f"/users/{user_id}", "method": "PUT"},
            "delete": {"href": f"/users/{user_id}", "method": "DELETE"},
            "collection": {"href": "/users"}
        }
    }
```

## Сравнение с другими стилями API

### REST vs GraphQL

| Критерий | REST | GraphQL |
|----------|------|---------|
| Формат запроса | HTTP методы + URL | Единый POST endpoint |
| Over-fetching | Часто | Нет — клиент выбирает поля |
| Under-fetching | Часто (N+1) | Нет — один запрос |
| Кэширование | Простое (HTTP cache) | Сложное |
| Версионирование | URL или заголовки | Схема расширяема |
| Кривая обучения | Низкая | Выше |

### REST vs gRPC

| Критерий | REST | gRPC |
|----------|------|------|
| Протокол | HTTP/1.1, HTTP/2 | HTTP/2 |
| Формат данных | JSON/XML | Protocol Buffers |
| Производительность | Ниже | Выше |
| Browser support | Полная | Ограниченная |
| Streaming | Ограниченный | Полный bidirectional |
| Типизация | Опциональная | Строгая |

## Когда использовать REST

**Подходит для:**
- Публичных API (веб, мобильные приложения)
- CRUD-операций над ресурсами
- Когда важна простота и широкая совместимость
- Когда нужно хорошее кэширование
- Микросервисов с разными технологиями

**Не подходит для:**
- Real-time приложений (используй WebSocket)
- Высоконагруженных внутренних сервисов (используй gRPC)
- Сложных вложенных запросов (используй GraphQL)

## Best Practices

### 1. Версионирование API

```python
# В URL (рекомендуется для публичных API)
GET /api/v1/users
GET /api/v2/users

# В заголовке
GET /users
Accept: application/vnd.myapi.v1+json
```

### 2. Стандартизированные ответы об ошибках

```python
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": {
                "code": exc.status_code,
                "message": exc.detail,
                "path": str(request.url),
                "timestamp": datetime.now().isoformat()
            }
        }
    )
```

### 3. Пагинация

```python
@app.get("/users")
def get_users(page: int = 1, per_page: int = 20):
    total = len(users_db)
    users = list(users_db.values())[(page-1)*per_page:page*per_page]

    return {
        "data": users,
        "pagination": {
            "page": page,
            "per_page": per_page,
            "total": total,
            "total_pages": (total + per_page - 1) // per_page
        },
        "_links": {
            "self": f"/users?page={page}&per_page={per_page}",
            "next": f"/users?page={page+1}&per_page={per_page}" if page * per_page < total else None,
            "prev": f"/users?page={page-1}&per_page={per_page}" if page > 1 else None
        }
    }
```

### 4. Используй правильные Content-Type

```python
# Запрос
POST /users HTTP/1.1
Content-Type: application/json

{"name": "John"}

# Ответ
HTTP/1.1 201 Created
Content-Type: application/json
Location: /users/123

{"id": 123, "name": "John"}
```

## Типичные ошибки

### 1. Глаголы в URL
```python
# Плохо
POST /createUser
GET /getUserById/123

# Хорошо
POST /users
GET /users/123
```

### 2. Неправильные статус-коды
```python
# Плохо: всегда 200, даже при ошибке
return JSONResponse(status_code=200, content={"error": "Not found"})

# Хорошо
raise HTTPException(status_code=404, detail="User not found")
```

### 3. Игнорирование идемпотентности
```python
# PUT должен быть идемпотентным — это полная замена
PUT /users/123
{"name": "John"}  # Все остальные поля сбросятся!

# Для частичного обновления используй PATCH
PATCH /users/123
{"name": "John"}  # Остальные поля не изменятся
```

### 4. Хранение состояния на сервере
```python
# Плохо: сервер помнит "текущий" ресурс
session['current_user_id'] = 123

# Хорошо: клиент всегда указывает ресурс явно
GET /users/123
Authorization: Bearer <token>
```

## Дополнительные ресурсы

- [REST API Tutorial](https://restfulapi.net/)
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)
- [JSON:API Specification](https://jsonapi.org/)
- [OpenAPI Specification](https://swagger.io/specification/)
