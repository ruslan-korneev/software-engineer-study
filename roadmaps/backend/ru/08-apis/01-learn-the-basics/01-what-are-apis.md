# What are APIs (Что такое API)

## Определение

**API (Application Programming Interface)** — это интерфейс программирования приложений, набор правил и протоколов, который позволяет различным программным компонентам взаимодействовать друг с другом. API определяет, какие запросы можно отправлять, как их формировать и какие ответы ожидать.

Простая аналогия: API — это как меню в ресторане. Вы (клиент) не заходите на кухню готовить еду сами. Вместо этого вы смотрите меню (API документация), делаете заказ (запрос), и официант приносит вам блюдо (ответ).

## Типы API

### 1. Web API (HTTP API)

Наиболее распространенный тип API, работающий через протокол HTTP/HTTPS.

```python
import requests

# Пример запроса к Web API
response = requests.get("https://api.github.com/users/octocat")
user_data = response.json()
print(user_data["name"])  # The Octocat
```

### 2. REST API

Архитектурный стиль для создания Web API, основанный на принципах REST (Representational State Transfer).

```python
# REST API следует стандартным HTTP методам
# GET    - получение данных
# POST   - создание данных
# PUT    - полное обновление
# PATCH  - частичное обновление
# DELETE - удаление

import requests

# Получение списка пользователей
users = requests.get("https://api.example.com/users")

# Создание нового пользователя
new_user = requests.post(
    "https://api.example.com/users",
    json={"name": "John", "email": "john@example.com"}
)

# Обновление пользователя
updated = requests.put(
    "https://api.example.com/users/123",
    json={"name": "John Doe", "email": "johndoe@example.com"}
)

# Удаление пользователя
requests.delete("https://api.example.com/users/123")
```

### 3. GraphQL API

Язык запросов для API, позволяющий клиенту запрашивать только нужные данные.

```graphql
# GraphQL запрос
query {
  user(id: "123") {
    name
    email
    posts {
      title
    }
  }
}
```

```python
import requests

query = """
query {
  user(id: "123") {
    name
    email
  }
}
"""

response = requests.post(
    "https://api.example.com/graphql",
    json={"query": query}
)
```

### 4. SOAP API

Протокол обмена структурированными сообщениями на основе XML. Часто используется в корпоративных системах.

```xml
<?xml version="1.0"?>
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
  <soap:Body>
    <GetUser xmlns="http://example.com/users">
      <UserId>123</UserId>
    </GetUser>
  </soap:Body>
</soap:Envelope>
```

### 5. gRPC API

Высокопроизводительный RPC фреймворк от Google, использующий Protocol Buffers.

```protobuf
// user.proto
service UserService {
  rpc GetUser (GetUserRequest) returns (User);
}

message GetUserRequest {
  string id = 1;
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
}
```

### 6. WebSocket API

Протокол для двусторонней связи в реальном времени.

```python
import websockets
import asyncio

async def connect():
    async with websockets.connect("wss://api.example.com/ws") as ws:
        await ws.send("Hello!")
        response = await ws.recv()
        print(response)

asyncio.run(connect())
```

## Ключевые компоненты API

### 1. Endpoint (Конечная точка)

URL-адрес, по которому доступен определенный ресурс.

```
GET  /api/v1/users          # Список пользователей
GET  /api/v1/users/123      # Конкретный пользователь
POST /api/v1/users          # Создание пользователя
```

### 2. Request (Запрос)

Сообщение от клиента к серверу.

```http
POST /api/v1/users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer token123

{
    "name": "John",
    "email": "john@example.com"
}
```

### 3. Response (Ответ)

Сообщение от сервера к клиенту.

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
    "id": 123,
    "name": "John",
    "email": "john@example.com",
    "created_at": "2024-01-15T10:30:00Z"
}
```

### 4. Authentication (Аутентификация)

Проверка личности клиента.

```python
# API Key
headers = {"X-API-Key": "your-api-key"}

# Bearer Token (JWT)
headers = {"Authorization": "Bearer eyJhbGciOiJIUzI1NiIs..."}

# Basic Auth
import base64
credentials = base64.b64encode(b"username:password").decode()
headers = {"Authorization": f"Basic {credentials}"}
```

## Пример создания простого API на Python (FastAPI)

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional

app = FastAPI()

# Модель данных
class User(BaseModel):
    id: Optional[int] = None
    name: str
    email: str

# Хранилище (в реальности — база данных)
users_db: List[User] = []
counter = 0

# GET - получить всех пользователей
@app.get("/users", response_model=List[User])
def get_users():
    return users_db

# GET - получить пользователя по ID
@app.get("/users/{user_id}", response_model=User)
def get_user(user_id: int):
    for user in users_db:
        if user.id == user_id:
            return user
    raise HTTPException(status_code=404, detail="User not found")

# POST - создать пользователя
@app.post("/users", response_model=User, status_code=201)
def create_user(user: User):
    global counter
    counter += 1
    user.id = counter
    users_db.append(user)
    return user

# DELETE - удалить пользователя
@app.delete("/users/{user_id}", status_code=204)
def delete_user(user_id: int):
    for i, user in enumerate(users_db):
        if user.id == user_id:
            users_db.pop(i)
            return
    raise HTTPException(status_code=404, detail="User not found")
```

## Best Practices

### 1. Версионирование API

```
/api/v1/users
/api/v2/users
```

### 2. Понятное именование

```
# Хорошо
GET /users
GET /users/123/orders

# Плохо
GET /getUsers
GET /user123orders
```

### 3. Правильные HTTP-коды ответов

```python
# 200 OK - успешный GET
# 201 Created - успешный POST
# 204 No Content - успешный DELETE
# 400 Bad Request - ошибка клиента
# 401 Unauthorized - не авторизован
# 404 Not Found - ресурс не найден
# 500 Internal Server Error - ошибка сервера
```

### 4. Документация API

Используйте OpenAPI (Swagger) для документирования:

```python
from fastapi import FastAPI

app = FastAPI(
    title="User API",
    description="API для управления пользователями",
    version="1.0.0"
)

@app.get("/users", summary="Получить всех пользователей")
def get_users():
    """
    Возвращает список всех пользователей в системе.

    - **page**: номер страницы (опционально)
    - **limit**: количество записей на странице (опционально)
    """
    pass
```

## Типичные ошибки

1. **Отсутствие версионирования** — невозможно обновить API без поломки клиентов
2. **Непоследовательный дизайн** — разные стили именования в разных endpoints
3. **Отсутствие пагинации** — возврат всех данных сразу при большом объеме
4. **Игнорирование безопасности** — отсутствие аутентификации и rate limiting
5. **Плохая обработка ошибок** — непонятные сообщения об ошибках

```python
# Плохо
{"error": "Error"}

# Хорошо
{
    "error": {
        "code": "USER_NOT_FOUND",
        "message": "Пользователь с ID 123 не найден",
        "details": {"user_id": 123}
    }
}
```

## Заключение

API — это фундаментальная концепция современной разработки. Понимание принципов работы API необходимо для:
- Интеграции сторонних сервисов
- Создания микросервисной архитектуры
- Разделения frontend и backend
- Построения масштабируемых систем
