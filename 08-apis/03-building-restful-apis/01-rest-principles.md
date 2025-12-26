# REST Principles (Принципы REST)

## Введение

**REST** (Representational State Transfer) — это архитектурный стиль для проектирования распределённых систем, предложенный Роем Филдингом в 2000 году в его докторской диссертации. REST не является протоколом или стандартом — это набор архитектурных ограничений, которые при соблюдении обеспечивают масштабируемость, производительность и простоту взаимодействия между клиентом и сервером.

## Шесть ключевых принципов REST

### 1. Client-Server (Клиент-Сервер)

Разделение ответственности между клиентом и сервером. Клиент отвечает за пользовательский интерфейс, сервер — за хранение и обработку данных.

**Преимущества:**
- Независимая эволюция клиента и сервера
- Упрощение серверной части (не нужно думать о UI)
- Возможность масштабирования компонентов по отдельности

```
┌──────────────┐         HTTP          ┌──────────────┐
│    Client    │ ◄──────────────────► │    Server    │
│  (Frontend)  │    Request/Response   │   (Backend)  │
└──────────────┘                       └──────────────┘
```

### 2. Stateless (Без сохранения состояния)

Каждый запрос от клиента к серверу должен содержать всю необходимую информацию для его обработки. Сервер не хранит контекст между запросами.

**Пример stateless запроса:**

```http
GET /api/users/123/orders HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Accept: application/json
```

**Пример stateful подхода (НЕ REST):**

```http
# Запрос 1: Авторизация, сервер запоминает сессию
POST /login
# Запрос 2: Сервер уже "знает" кто мы
GET /my-orders
```

**Преимущества stateless:**
- Легко масштабировать (любой сервер может обработать любой запрос)
- Повышенная надёжность
- Упрощённая отладка

### 3. Cacheable (Кэширование)

Ответы сервера должны явно или неявно указывать, можно ли их кэшировать.

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=3600, public
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"

{
    "id": 123,
    "name": "Product Name",
    "price": 99.99
}
```

**Заголовки кэширования:**

| Заголовок | Описание |
|-----------|----------|
| `Cache-Control` | Директивы кэширования |
| `ETag` | Идентификатор версии ресурса |
| `Last-Modified` | Время последнего изменения |
| `Expires` | Дата истечения кэша |

**Пример условного запроса:**

```http
GET /api/products/123 HTTP/1.1
If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"
```

```http
HTTP/1.1 304 Not Modified
```

### 4. Uniform Interface (Единообразный интерфейс)

Это центральный принцип REST, включающий четыре ограничения:

#### 4.1 Идентификация ресурсов (Resource Identification)

Каждый ресурс идентифицируется уникальным URI:

```
/users              # Коллекция пользователей
/users/123          # Конкретный пользователь
/users/123/orders   # Заказы пользователя
/orders/456         # Конкретный заказ
```

#### 4.2 Манипуляция ресурсами через представления

Клиент работает с представлениями ресурсов (JSON, XML), а не с самими ресурсами:

```http
PUT /api/users/123 HTTP/1.1
Content-Type: application/json

{
    "id": 123,
    "name": "Updated Name",
    "email": "new@example.com"
}
```

#### 4.3 Самоописывающие сообщения (Self-descriptive Messages)

Каждое сообщение содержит достаточно информации для его обработки:

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/users/124

{
    "id": 124,
    "name": "New User",
    "created_at": "2024-01-15T10:30:00Z"
}
```

#### 4.4 HATEOAS (Hypermedia as the Engine of Application State)

Ответы содержат ссылки для навигации:

```json
{
    "id": 123,
    "name": "John Doe",
    "_links": {
        "self": { "href": "/api/users/123" },
        "orders": { "href": "/api/users/123/orders" },
        "edit": { "href": "/api/users/123", "method": "PUT" }
    }
}
```

### 5. Layered System (Многоуровневая система)

Клиент не знает, взаимодействует ли он напрямую с сервером или через промежуточные слои (балансировщики, прокси, CDN).

```
┌────────┐    ┌─────────────┐    ┌───────┐    ┌────────┐
│ Client │───►│ Load Balancer │───►│ Cache │───►│ Server │
└────────┘    └─────────────┘    └───────┘    └────────┘
```

### 6. Code on Demand (опционально)

Сервер может передавать клиенту исполняемый код (JavaScript). Это единственный опциональный принцип.

```http
HTTP/1.1 200 OK
Content-Type: application/javascript

function validateForm(data) {
    // Код валидации
    return data.email.includes('@');
}
```

## HTTP-методы в REST

| Метод | CRUD | Идемпотентность | Безопасность | Описание |
|-------|------|-----------------|--------------|----------|
| GET | Read | Да | Да | Получение ресурса |
| POST | Create | Нет | Нет | Создание ресурса |
| PUT | Update/Replace | Да | Нет | Полное обновление |
| PATCH | Update/Modify | Нет* | Нет | Частичное обновление |
| DELETE | Delete | Да | Нет | Удаление ресурса |

*PATCH может быть идемпотентным при правильной реализации

## Примеры реализации

### Python (FastAPI)

```python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel
from typing import List, Optional

app = FastAPI()

# Модель данных
class User(BaseModel):
    id: Optional[int] = None
    name: str
    email: str

class UserUpdate(BaseModel):
    name: Optional[str] = None
    email: Optional[str] = None

# Хранилище (в реальности — база данных)
users_db: dict[int, User] = {}

# GET — получить всех пользователей
@app.get("/api/users", response_model=List[User])
async def get_users():
    return list(users_db.values())

# GET — получить конкретного пользователя
@app.get("/api/users/{user_id}", response_model=User)
async def get_user(user_id: int):
    if user_id not in users_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"User {user_id} not found"
        )
    return users_db[user_id]

# POST — создать пользователя
@app.post("/api/users", response_model=User, status_code=status.HTTP_201_CREATED)
async def create_user(user: User):
    user_id = max(users_db.keys(), default=0) + 1
    user.id = user_id
    users_db[user_id] = user
    return user

# PUT — полное обновление
@app.put("/api/users/{user_id}", response_model=User)
async def update_user(user_id: int, user: User):
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail="User not found")
    user.id = user_id
    users_db[user_id] = user
    return user

# PATCH — частичное обновление
@app.patch("/api/users/{user_id}", response_model=User)
async def patch_user(user_id: int, user_update: UserUpdate):
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail="User not found")

    stored_user = users_db[user_id]
    update_data = user_update.model_dump(exclude_unset=True)

    for field, value in update_data.items():
        setattr(stored_user, field, value)

    return stored_user

# DELETE — удаление
@app.delete("/api/users/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(user_id: int):
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail="User not found")
    del users_db[user_id]
```

### Node.js (Express)

```javascript
const express = require('express');
const app = express();

app.use(express.json());

// Хранилище
let users = new Map();
let nextId = 1;

// GET — список пользователей
app.get('/api/users', (req, res) => {
    res.json(Array.from(users.values()));
});

// GET — один пользователь
app.get('/api/users/:id', (req, res) => {
    const user = users.get(parseInt(req.params.id));
    if (!user) {
        return res.status(404).json({ error: 'User not found' });
    }
    res.json(user);
});

// POST — создание
app.post('/api/users', (req, res) => {
    const user = {
        id: nextId++,
        name: req.body.name,
        email: req.body.email
    };
    users.set(user.id, user);
    res.status(201)
       .location(`/api/users/${user.id}`)
       .json(user);
});

// PUT — полное обновление
app.put('/api/users/:id', (req, res) => {
    const id = parseInt(req.params.id);
    if (!users.has(id)) {
        return res.status(404).json({ error: 'User not found' });
    }
    const user = { id, name: req.body.name, email: req.body.email };
    users.set(id, user);
    res.json(user);
});

// PATCH — частичное обновление
app.patch('/api/users/:id', (req, res) => {
    const id = parseInt(req.params.id);
    const existing = users.get(id);
    if (!existing) {
        return res.status(404).json({ error: 'User not found' });
    }
    const updated = { ...existing, ...req.body, id };
    users.set(id, updated);
    res.json(updated);
});

// DELETE — удаление
app.delete('/api/users/:id', (req, res) => {
    const id = parseInt(req.params.id);
    if (!users.has(id)) {
        return res.status(404).json({ error: 'User not found' });
    }
    users.delete(id);
    res.status(204).send();
});

app.listen(3000);
```

## Best Practices

### Что делать:

1. **Используйте существительные для ресурсов:**
   - `/users`, `/orders`, `/products`

2. **Используйте HTTP-методы правильно:**
   - GET для чтения, POST для создания и т.д.

3. **Возвращайте правильные коды статуса:**
   - 200 OK, 201 Created, 204 No Content, 404 Not Found

4. **Делайте API предсказуемым:**
   - Консистентные имена, форматы ответов

5. **Используйте версионирование:**
   - `/api/v1/users`

### Типичные ошибки:

1. **Глаголы в URL:**
   - `/getUsers` — неправильно
   - `/users` с GET — правильно

2. **Хранение состояния на сервере:**
   - Сессии вместо токенов

3. **Игнорирование кодов статуса:**
   - Всегда возвращать 200, даже при ошибках

4. **Отсутствие пагинации:**
   - Возвращение миллионов записей одним запросом

## Уровни зрелости REST API (Richardson Maturity Model)

```
Уровень 3: Hypermedia Controls (HATEOAS)
    │
Уровень 2: HTTP Verbs (правильное использование методов)
    │
Уровень 1: Resources (использование URI для ресурсов)
    │
Уровень 0: The Swamp of POX (один endpoint, всё через POST)
```

| Уровень | Описание | Пример |
|---------|----------|--------|
| 0 | Один endpoint | `POST /api` с телом `{action: "getUser", id: 1}` |
| 1 | Отдельные URI для ресурсов | `POST /api/users/1` |
| 2 | Правильные HTTP-методы | `GET /api/users/1` |
| 3 | HATEOAS | Ответ содержит ссылки на связанные ресурсы |

## Примеры из реальных API

### GitHub API

```http
GET https://api.github.com/users/octocat/repos
```

### Stripe API

```http
POST https://api.stripe.com/v1/customers
Authorization: Bearer sk_test_...
Content-Type: application/x-www-form-urlencoded

email=customer@example.com&name=John%20Doe
```

### Twitter API v2

```http
GET https://api.twitter.com/2/users/12345/tweets
Authorization: Bearer YOUR_BEARER_TOKEN
```

## Заключение

REST — это не строгий стандарт, а архитектурный стиль. Полное соблюдение всех принципов (особенно HATEOAS) встречается редко, но понимание этих принципов помогает создавать масштабируемые, понятные и удобные API.

**Ключевые моменты:**
- Разделение клиента и сервера
- Stateless взаимодействие
- Единообразный интерфейс
- Правильное использование HTTP
- Кэширование для производительности
