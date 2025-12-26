# HTTP Methods (HTTP методы)

## Обзор HTTP методов

HTTP методы (также называемые HTTP verbs) определяют тип операции, которую клиент хочет выполнить над ресурсом. Каждый метод имеет свою семантику и предназначение.

## Основные методы

### GET

**Назначение**: Получение данных с сервера.

**Характеристики**:
- Безопасный (safe) — не изменяет данные на сервере
- Идемпотентный — многократные запросы дают одинаковый результат
- Кэшируемый
- НЕ должен содержать тело запроса

```python
import requests

# Простой GET запрос
response = requests.get("https://api.example.com/users")
users = response.json()

# GET с query параметрами
response = requests.get(
    "https://api.example.com/users",
    params={"page": 1, "limit": 10, "status": "active"}
)
# URL: https://api.example.com/users?page=1&limit=10&status=active

# GET конкретного ресурса
response = requests.get("https://api.example.com/users/123")
user = response.json()
```

```http
GET /api/users?page=1&limit=10 HTTP/1.1
Host: api.example.com
Accept: application/json
```

### POST

**Назначение**: Создание нового ресурса или отправка данных для обработки.

**Характеристики**:
- НЕ безопасный — изменяет данные на сервере
- НЕ идемпотентный — повторный запрос создаст новый ресурс
- НЕ кэшируемый (по умолчанию)
- Содержит тело запроса

```python
import requests

# Создание нового ресурса
response = requests.post(
    "https://api.example.com/users",
    json={
        "name": "John Doe",
        "email": "john@example.com",
        "role": "user"
    }
)

if response.status_code == 201:
    new_user = response.json()
    print(f"Создан пользователь с ID: {new_user['id']}")

# POST с form data
response = requests.post(
    "https://api.example.com/login",
    data={
        "username": "john",
        "password": "secret"
    }
)

# POST с файлом
with open("avatar.png", "rb") as f:
    response = requests.post(
        "https://api.example.com/upload",
        files={"avatar": f}
    )
```

```http
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
    "name": "John Doe",
    "email": "john@example.com"
}
```

### PUT

**Назначение**: Полная замена существующего ресурса.

**Характеристики**:
- НЕ безопасный
- Идемпотентный — повторные запросы дают одинаковый результат
- Требует полное представление ресурса
- Создает ресурс, если он не существует (опционально)

```python
import requests

# Полное обновление пользователя
response = requests.put(
    "https://api.example.com/users/123",
    json={
        "name": "John Updated",
        "email": "john.updated@example.com",
        "role": "admin",
        "status": "active"
    }
)

if response.status_code == 200:
    updated_user = response.json()
```

```http
PUT /api/users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
    "name": "John Updated",
    "email": "john.updated@example.com",
    "role": "admin",
    "status": "active"
}
```

### PATCH

**Назначение**: Частичное обновление ресурса.

**Характеристики**:
- НЕ безопасный
- Может быть идемпотентным (зависит от реализации)
- Отправляет только изменяемые поля

```python
import requests

# Частичное обновление (только email)
response = requests.patch(
    "https://api.example.com/users/123",
    json={
        "email": "newemail@example.com"
    }
)

# JSON Patch формат (RFC 6902)
response = requests.patch(
    "https://api.example.com/users/123",
    json=[
        {"op": "replace", "path": "/email", "value": "new@example.com"},
        {"op": "add", "path": "/phone", "value": "+1234567890"}
    ],
    headers={"Content-Type": "application/json-patch+json"}
)
```

```http
PATCH /api/users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
    "email": "newemail@example.com"
}
```

### DELETE

**Назначение**: Удаление ресурса.

**Характеристики**:
- НЕ безопасный
- Идемпотентный — повторное удаление того же ресурса не должно вызывать ошибку
- Обычно не содержит тело запроса

```python
import requests

# Удаление ресурса
response = requests.delete("https://api.example.com/users/123")

if response.status_code == 204:
    print("Пользователь удален")
elif response.status_code == 404:
    print("Пользователь не найден")

# Удаление с условием
response = requests.delete(
    "https://api.example.com/users/123",
    headers={"If-Match": '"abc123"'}  # ETag
)
```

```http
DELETE /api/users/123 HTTP/1.1
Host: api.example.com
Authorization: Bearer token123
```

## Дополнительные методы

### HEAD

**Назначение**: Получить только заголовки ответа (без тела).

```python
import requests

# Проверка существования ресурса без загрузки данных
response = requests.head("https://api.example.com/files/large-file.zip")

print(response.headers.get("Content-Length"))  # Размер файла
print(response.headers.get("Last-Modified"))   # Дата изменения

# Проверка доступности
if response.status_code == 200:
    print("Файл существует")
```

### OPTIONS

**Назначение**: Узнать поддерживаемые методы для ресурса (часто используется в CORS).

```python
import requests

response = requests.options("https://api.example.com/users")
allowed_methods = response.headers.get("Allow")
print(f"Разрешенные методы: {allowed_methods}")
# Вывод: GET, POST, OPTIONS
```

```http
OPTIONS /api/users HTTP/1.1
Host: api.example.com

# Ответ
HTTP/1.1 204 No Content
Allow: GET, POST, OPTIONS
Access-Control-Allow-Methods: GET, POST, OPTIONS
```

### CONNECT

**Назначение**: Установка туннеля к серверу (для HTTPS через прокси).

```http
CONNECT example.com:443 HTTP/1.1
Host: example.com
```

### TRACE

**Назначение**: Диагностика — возвращает полученный запрос обратно клиенту.

```http
TRACE /api/test HTTP/1.1
Host: api.example.com
```

> **Внимание**: TRACE часто отключен из соображений безопасности (XST атаки).

## Сравнение методов

| Метод   | Безопасный | Идемпотентный | Тело запроса | Тело ответа | Кэшируемый |
|---------|------------|---------------|--------------|-------------|------------|
| GET     | Да         | Да            | Нет          | Да          | Да         |
| POST    | Нет        | Нет           | Да           | Да          | Нет*       |
| PUT     | Нет        | Да            | Да           | Да          | Нет        |
| PATCH   | Нет        | Нет*          | Да           | Да          | Нет        |
| DELETE  | Нет        | Да            | Нет*         | Да*         | Нет        |
| HEAD    | Да         | Да            | Нет          | Нет         | Да         |
| OPTIONS | Да         | Да            | Нет          | Да          | Нет        |

*С оговорками

## Идемпотентность подробнее

**Идемпотентный метод** — это метод, многократный вызов которого с теми же параметрами даёт тот же результат, что и один вызов.

```python
# GET — идемпотентный
# Каждый вызов возвращает те же данные
requests.get("/users/123")  # {"id": 123, "name": "John"}
requests.get("/users/123")  # {"id": 123, "name": "John"}
requests.get("/users/123")  # {"id": 123, "name": "John"}

# POST — НЕ идемпотентный
# Каждый вызов создает новый ресурс
requests.post("/users", json={"name": "John"})  # Создан user 1
requests.post("/users", json={"name": "John"})  # Создан user 2
requests.post("/users", json={"name": "John"})  # Создан user 3

# PUT — идемпотентный
# Результат всегда одинаковый
requests.put("/users/123", json={"name": "John"})  # User обновлен
requests.put("/users/123", json={"name": "John"})  # User остался тем же
requests.put("/users/123", json={"name": "John"})  # User остался тем же

# DELETE — идемпотентный
requests.delete("/users/123")  # User удален (204)
requests.delete("/users/123")  # User не найден (404 или 204)
requests.delete("/users/123")  # User не найден (404 или 204)
```

## Безопасные методы

**Безопасный метод** — это метод, который не изменяет состояние сервера.

```python
# Безопасные методы (только чтение)
requests.get("/users")      # Чтение данных
requests.head("/users")     # Проверка заголовков
requests.options("/users")  # Получение метаданных

# НЕ безопасные методы (изменение данных)
requests.post("/users", json={...})    # Создание
requests.put("/users/1", json={...})   # Замена
requests.patch("/users/1", json={...}) # Обновление
requests.delete("/users/1")            # Удаление
```

## RESTful соглашения

```python
# Коллекция ресурсов
GET    /users          # Получить список пользователей
POST   /users          # Создать нового пользователя

# Конкретный ресурс
GET    /users/123      # Получить пользователя 123
PUT    /users/123      # Полностью заменить пользователя 123
PATCH  /users/123      # Частично обновить пользователя 123
DELETE /users/123      # Удалить пользователя 123

# Вложенные ресурсы
GET    /users/123/orders       # Заказы пользователя 123
POST   /users/123/orders       # Создать заказ для пользователя 123
GET    /users/123/orders/456   # Заказ 456 пользователя 123
```

## Практический пример: CRUD API

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

class UserCreate(BaseModel):
    name: str
    email: str

class UserUpdate(BaseModel):
    name: Optional[str] = None
    email: Optional[str] = None

# Хранилище
users = {}
counter = 0

# CREATE
@app.post("/users", status_code=201)
def create_user(user: UserCreate):
    global counter
    counter += 1
    users[counter] = {"id": counter, **user.dict()}
    return users[counter]

# READ (список)
@app.get("/users")
def list_users(skip: int = 0, limit: int = 10):
    return list(users.values())[skip:skip + limit]

# READ (один)
@app.get("/users/{user_id}")
def get_user(user_id: int):
    if user_id not in users:
        raise HTTPException(status_code=404, detail="User not found")
    return users[user_id]

# UPDATE (полный)
@app.put("/users/{user_id}")
def replace_user(user_id: int, user: UserCreate):
    if user_id not in users:
        raise HTTPException(status_code=404, detail="User not found")
    users[user_id] = {"id": user_id, **user.dict()}
    return users[user_id]

# UPDATE (частичный)
@app.patch("/users/{user_id}")
def update_user(user_id: int, user: UserUpdate):
    if user_id not in users:
        raise HTTPException(status_code=404, detail="User not found")
    stored = users[user_id]
    update_data = user.dict(exclude_unset=True)
    users[user_id] = {**stored, **update_data}
    return users[user_id]

# DELETE
@app.delete("/users/{user_id}", status_code=204)
def delete_user(user_id: int):
    if user_id not in users:
        raise HTTPException(status_code=404, detail="User not found")
    del users[user_id]
```

## Best Practices

### 1. Используйте правильный метод для операции

```python
# Плохо: использование POST для получения данных
@app.post("/users/search")
def search_users(query: str):
    pass

# Хорошо: использование GET с параметрами
@app.get("/users")
def search_users(q: str = None, status: str = None):
    pass
```

### 2. Не используйте глаголы в URL

```python
# Плохо
POST /createUser
GET /getUser/123
POST /deleteUser/123

# Хорошо
POST /users
GET /users/123
DELETE /users/123
```

### 3. Возвращайте правильные коды ответа

```python
@app.post("/users", status_code=201)  # Created
def create_user(user: UserCreate):
    return new_user

@app.delete("/users/{id}", status_code=204)  # No Content
def delete_user(id: int):
    return None

@app.get("/users/{id}")  # 200 OK (по умолчанию)
def get_user(id: int):
    return user
```

### 4. PUT vs PATCH

```python
# PUT — полная замена (все поля обязательны)
PUT /users/123
{
    "name": "John",
    "email": "john@example.com",
    "role": "admin",
    "status": "active"
}

# PATCH — частичное обновление (только изменяемые поля)
PATCH /users/123
{
    "status": "inactive"
}
```

## Типичные ошибки

1. **Использование GET для изменения данных**
   ```python
   # Плохо (GET изменяет данные!)
   GET /users/123/delete

   # Хорошо
   DELETE /users/123
   ```

2. **Игнорирование идемпотентности**
   ```python
   # Плохо: PUT создает новый ресурс при каждом вызове
   # Хорошо: PUT всегда дает одинаковый результат
   ```

3. **Неправильные коды ответа**
   ```python
   # Плохо: POST возвращает 200
   # Хорошо: POST возвращает 201 (Created)

   # Плохо: DELETE возвращает 200 с телом
   # Хорошо: DELETE возвращает 204 (No Content)
   ```

4. **Тело в GET запросе**
   ```python
   # Плохо (технически возможно, но не рекомендуется)
   GET /search
   {"query": "test"}

   # Хорошо
   GET /search?query=test
   ```
