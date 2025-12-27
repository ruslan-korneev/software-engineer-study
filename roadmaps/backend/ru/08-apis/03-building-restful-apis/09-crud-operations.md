# Handling CRUD Operations (Обработка CRUD операций)

## Введение

**CRUD** — это акроним для четырёх базовых операций с данными:
- **C**reate — создание
- **R**ead — чтение
- **U**pdate — обновление
- **D**elete — удаление

В REST API эти операции маппятся на HTTP-методы:

| CRUD | HTTP метод | Типичный URI | Описание |
|------|------------|--------------|----------|
| Create | POST | `/users` | Создать нового пользователя |
| Read | GET | `/users`, `/users/123` | Получить список или одного |
| Update | PUT/PATCH | `/users/123` | Обновить пользователя |
| Delete | DELETE | `/users/123` | Удалить пользователя |

## Create (POST)

### Базовая реализация

```http
POST /api/users HTTP/1.1
Content-Type: application/json

{
    "name": "John Doe",
    "email": "john@example.com",
    "role": "user"
}
```

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/users/123

{
    "id": 123,
    "name": "John Doe",
    "email": "john@example.com",
    "role": "user",
    "created_at": "2024-01-15T10:30:00Z"
}
```

### Python (FastAPI)

```python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel, EmailStr, Field
from typing import Optional
from datetime import datetime
import uuid

app = FastAPI()

# Модели
class UserCreate(BaseModel):
    name: str = Field(..., min_length=2, max_length=100)
    email: EmailStr
    role: str = Field(default="user", pattern="^(user|admin|moderator)$")

class User(BaseModel):
    id: str
    name: str
    email: str
    role: str
    created_at: datetime
    updated_at: Optional[datetime] = None

# Хранилище (в реальности — БД)
users_db: dict[str, dict] = {}

@app.post("/api/users", response_model=User, status_code=status.HTTP_201_CREATED)
async def create_user(user_data: UserCreate):
    # Проверка на дубликат email
    for user in users_db.values():
        if user["email"] == user_data.email:
            raise HTTPException(
                status_code=status.HTTP_409_CONFLICT,
                detail=f"User with email '{user_data.email}' already exists"
            )

    user_id = str(uuid.uuid4())
    now = datetime.utcnow()

    user = {
        "id": user_id,
        **user_data.dict(),
        "created_at": now,
        "updated_at": None
    }

    users_db[user_id] = user

    return user
```

### Node.js (Express)

```javascript
const express = require('express');
const { v4: uuidv4 } = require('uuid');

const app = express();
app.use(express.json());

const users = new Map();

// Валидация
const validateUserCreate = (req, res, next) => {
    const { name, email, role = 'user' } = req.body;
    const errors = [];

    if (!name || name.length < 2) {
        errors.push({ field: 'name', message: 'Name must be at least 2 characters' });
    }

    if (!email || !email.includes('@')) {
        errors.push({ field: 'email', message: 'Invalid email format' });
    }

    if (!['user', 'admin', 'moderator'].includes(role)) {
        errors.push({ field: 'role', message: 'Invalid role' });
    }

    if (errors.length > 0) {
        return res.status(422).json({ error: { code: 'VALIDATION_ERROR', details: errors } });
    }

    next();
};

app.post('/api/users', validateUserCreate, (req, res) => {
    const { name, email, role = 'user' } = req.body;

    // Проверка дубликата
    for (const user of users.values()) {
        if (user.email === email) {
            return res.status(409).json({
                error: { code: 'DUPLICATE', message: 'Email already exists' }
            });
        }
    }

    const id = uuidv4();
    const now = new Date().toISOString();

    const user = {
        id,
        name,
        email,
        role,
        created_at: now,
        updated_at: null
    };

    users.set(id, user);

    res
        .status(201)
        .location(`/api/users/${id}`)
        .json(user);
});

app.listen(3000);
```

## Read (GET)

### Получение списка с фильтрацией и пагинацией

```http
GET /api/users?role=admin&sort=-created_at&page=1&limit=20 HTTP/1.1
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
    "data": [
        { "id": "123", "name": "John", "email": "john@example.com", "role": "admin" },
        { "id": "456", "name": "Jane", "email": "jane@example.com", "role": "admin" }
    ],
    "pagination": {
        "page": 1,
        "limit": 20,
        "total": 50,
        "total_pages": 3
    }
}
```

### Получение одного ресурса

```http
GET /api/users/123 HTTP/1.1
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
    "id": "123",
    "name": "John Doe",
    "email": "john@example.com",
    "role": "admin",
    "created_at": "2024-01-15T10:30:00Z"
}
```

### Python (FastAPI)

```python
from fastapi import Query
from typing import List, Optional

class UserList(BaseModel):
    data: List[User]
    pagination: dict

@app.get("/api/users", response_model=UserList)
async def get_users(
    role: Optional[str] = Query(None, description="Filter by role"),
    search: Optional[str] = Query(None, description="Search by name or email"),
    sort: str = Query("created_at", description="Sort field (prefix with - for desc)"),
    page: int = Query(1, ge=1),
    limit: int = Query(20, ge=1, le=100)
):
    # Фильтрация
    filtered = list(users_db.values())

    if role:
        filtered = [u for u in filtered if u["role"] == role]

    if search:
        search_lower = search.lower()
        filtered = [
            u for u in filtered
            if search_lower in u["name"].lower() or search_lower in u["email"].lower()
        ]

    # Сортировка
    reverse = sort.startswith("-")
    sort_field = sort.lstrip("-")

    if sort_field in ["created_at", "name", "email"]:
        filtered.sort(key=lambda x: x.get(sort_field, ""), reverse=reverse)

    # Пагинация
    total = len(filtered)
    total_pages = (total + limit - 1) // limit
    start = (page - 1) * limit
    end = start + limit

    return {
        "data": filtered[start:end],
        "pagination": {
            "page": page,
            "limit": limit,
            "total": total,
            "total_pages": total_pages
        }
    }

@app.get("/api/users/{user_id}", response_model=User)
async def get_user(user_id: str):
    if user_id not in users_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"User '{user_id}' not found"
        )

    return users_db[user_id]
```

### Node.js (Express)

```javascript
app.get('/api/users', (req, res) => {
    let { role, search, sort = 'created_at', page = 1, limit = 20 } = req.query;

    page = Math.max(1, parseInt(page));
    limit = Math.min(100, Math.max(1, parseInt(limit)));

    let filtered = Array.from(users.values());

    // Фильтрация
    if (role) {
        filtered = filtered.filter(u => u.role === role);
    }

    if (search) {
        const searchLower = search.toLowerCase();
        filtered = filtered.filter(u =>
            u.name.toLowerCase().includes(searchLower) ||
            u.email.toLowerCase().includes(searchLower)
        );
    }

    // Сортировка
    const reverse = sort.startsWith('-');
    const sortField = sort.replace('-', '');

    filtered.sort((a, b) => {
        const valA = a[sortField] || '';
        const valB = b[sortField] || '';
        const result = valA < valB ? -1 : valA > valB ? 1 : 0;
        return reverse ? -result : result;
    });

    // Пагинация
    const total = filtered.length;
    const totalPages = Math.ceil(total / limit);
    const start = (page - 1) * limit;
    const paged = filtered.slice(start, start + limit);

    res.json({
        data: paged,
        pagination: { page, limit, total, total_pages: totalPages }
    });
});

app.get('/api/users/:id', (req, res) => {
    const user = users.get(req.params.id);

    if (!user) {
        return res.status(404).json({
            error: { code: 'NOT_FOUND', message: 'User not found' }
        });
    }

    res.json(user);
});
```

## Update (PUT и PATCH)

### PUT — Полное обновление

Заменяет ресурс целиком. Все поля обязательны.

```http
PUT /api/users/123 HTTP/1.1
Content-Type: application/json

{
    "name": "John Smith",
    "email": "johnsmith@example.com",
    "role": "admin"
}
```

### PATCH — Частичное обновление

Обновляет только переданные поля.

```http
PATCH /api/users/123 HTTP/1.1
Content-Type: application/json

{
    "name": "John Smith"
}
```

### Python (FastAPI)

```python
class UserUpdate(BaseModel):
    """Модель для PUT — все поля обязательны."""
    name: str = Field(..., min_length=2, max_length=100)
    email: EmailStr
    role: str = Field(..., pattern="^(user|admin|moderator)$")

class UserPatch(BaseModel):
    """Модель для PATCH — все поля опциональны."""
    name: Optional[str] = Field(None, min_length=2, max_length=100)
    email: Optional[EmailStr] = None
    role: Optional[str] = Field(None, pattern="^(user|admin|moderator)$")

@app.put("/api/users/{user_id}", response_model=User)
async def update_user(user_id: str, user_data: UserUpdate):
    """Полное обновление (замена)."""
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail="User not found")

    # Проверка уникальности email (исключая текущего пользователя)
    for uid, user in users_db.items():
        if uid != user_id and user["email"] == user_data.email:
            raise HTTPException(status_code=409, detail="Email already exists")

    # Сохраняем created_at, обновляем остальное
    existing = users_db[user_id]

    users_db[user_id] = {
        "id": user_id,
        **user_data.dict(),
        "created_at": existing["created_at"],
        "updated_at": datetime.utcnow()
    }

    return users_db[user_id]

@app.patch("/api/users/{user_id}", response_model=User)
async def patch_user(user_id: str, user_data: UserPatch):
    """Частичное обновление."""
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail="User not found")

    existing = users_db[user_id]

    # Обновляем только переданные поля
    update_data = user_data.dict(exclude_unset=True)

    if "email" in update_data:
        for uid, user in users_db.items():
            if uid != user_id and user["email"] == update_data["email"]:
                raise HTTPException(status_code=409, detail="Email already exists")

    for field, value in update_data.items():
        existing[field] = value

    existing["updated_at"] = datetime.utcnow()

    return existing
```

### Node.js (Express)

```javascript
// PUT — полное обновление
app.put('/api/users/:id', validateUserCreate, (req, res) => {
    const { id } = req.params;
    const existing = users.get(id);

    if (!existing) {
        return res.status(404).json({
            error: { code: 'NOT_FOUND', message: 'User not found' }
        });
    }

    const { name, email, role } = req.body;

    // Проверка уникальности email
    for (const [userId, user] of users) {
        if (userId !== id && user.email === email) {
            return res.status(409).json({
                error: { code: 'DUPLICATE', message: 'Email already exists' }
            });
        }
    }

    const updated = {
        id,
        name,
        email,
        role,
        created_at: existing.created_at,
        updated_at: new Date().toISOString()
    };

    users.set(id, updated);
    res.json(updated);
});

// PATCH — частичное обновление
app.patch('/api/users/:id', (req, res) => {
    const { id } = req.params;
    const existing = users.get(id);

    if (!existing) {
        return res.status(404).json({
            error: { code: 'NOT_FOUND', message: 'User not found' }
        });
    }

    const updates = req.body;

    // Валидация переданных полей
    if (updates.name !== undefined && updates.name.length < 2) {
        return res.status(422).json({
            error: { code: 'VALIDATION_ERROR', message: 'Name too short' }
        });
    }

    if (updates.email !== undefined) {
        for (const [userId, user] of users) {
            if (userId !== id && user.email === updates.email) {
                return res.status(409).json({
                    error: { code: 'DUPLICATE', message: 'Email already exists' }
                });
            }
        }
    }

    const updated = {
        ...existing,
        ...updates,
        id, // ID нельзя менять
        created_at: existing.created_at, // created_at нельзя менять
        updated_at: new Date().toISOString()
    };

    users.set(id, updated);
    res.json(updated);
});
```

## Delete (DELETE)

### Простое удаление

```http
DELETE /api/users/123 HTTP/1.1
```

```http
HTTP/1.1 204 No Content
```

### Мягкое удаление (Soft Delete)

Вместо удаления помечаем запись как удалённую:

```http
DELETE /api/users/123 HTTP/1.1
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
    "id": "123",
    "name": "John Doe",
    "deleted_at": "2024-01-15T10:30:00Z"
}
```

### Python (FastAPI)

```python
@app.delete("/api/users/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(user_id: str):
    """Жёсткое удаление."""
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail="User not found")

    del users_db[user_id]
    return None  # 204 No Content

# Альтернатива: Soft Delete
class UserWithDeleted(User):
    deleted_at: Optional[datetime] = None

@app.delete("/api/users/{user_id}", response_model=UserWithDeleted)
async def soft_delete_user(user_id: str):
    """Мягкое удаление."""
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail="User not found")

    user = users_db[user_id]

    if user.get("deleted_at"):
        raise HTTPException(status_code=410, detail="User already deleted")

    user["deleted_at"] = datetime.utcnow()

    return user

# При чтении исключаем удалённых
@app.get("/api/users", response_model=UserList)
async def get_users(include_deleted: bool = False):
    filtered = list(users_db.values())

    if not include_deleted:
        filtered = [u for u in filtered if not u.get("deleted_at")]

    # ... остальная логика
```

### Node.js (Express)

```javascript
// Жёсткое удаление
app.delete('/api/users/:id', (req, res) => {
    const { id } = req.params;

    if (!users.has(id)) {
        return res.status(404).json({
            error: { code: 'NOT_FOUND', message: 'User not found' }
        });
    }

    users.delete(id);
    res.status(204).send();
});

// Мягкое удаление
app.delete('/api/users/:id/soft', (req, res) => {
    const { id } = req.params;
    const user = users.get(id);

    if (!user) {
        return res.status(404).json({
            error: { code: 'NOT_FOUND', message: 'User not found' }
        });
    }

    if (user.deleted_at) {
        return res.status(410).json({
            error: { code: 'ALREADY_DELETED', message: 'User already deleted' }
        });
    }

    user.deleted_at = new Date().toISOString();
    res.json(user);
});
```

## Массовые операции (Batch)

### Batch Create

```http
POST /api/users/batch HTTP/1.1
Content-Type: application/json

[
    {"name": "User 1", "email": "user1@example.com"},
    {"name": "User 2", "email": "user2@example.com"}
]
```

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
    "created": [
        {"id": "1", "name": "User 1", "email": "user1@example.com"},
        {"id": "2", "name": "User 2", "email": "user2@example.com"}
    ],
    "errors": []
}
```

### Batch Delete

```http
DELETE /api/users?ids=1,2,3 HTTP/1.1
```

или

```http
POST /api/users/batch-delete HTTP/1.1
Content-Type: application/json

{"ids": ["1", "2", "3"]}
```

```python
class BatchDeleteRequest(BaseModel):
    ids: List[str]

class BatchDeleteResponse(BaseModel):
    deleted: List[str]
    not_found: List[str]

@app.post("/api/users/batch-delete", response_model=BatchDeleteResponse)
async def batch_delete_users(request: BatchDeleteRequest):
    deleted = []
    not_found = []

    for user_id in request.ids:
        if user_id in users_db:
            del users_db[user_id]
            deleted.append(user_id)
        else:
            not_found.append(user_id)

    return {"deleted": deleted, "not_found": not_found}
```

## Транзакционность и консистентность

### Оптимистичная блокировка (ETag/Version)

```http
GET /api/users/123 HTTP/1.1
```

```http
HTTP/1.1 200 OK
ETag: "abc123"

{"id": "123", "name": "John", "version": 1}
```

```http
PUT /api/users/123 HTTP/1.1
If-Match: "abc123"

{"name": "John Updated"}
```

```http
HTTP/1.1 200 OK (если версия совпала)
HTTP/1.1 412 Precondition Failed (если данные изменились)
```

```python
@app.put("/api/users/{user_id}")
async def update_user(
    user_id: str,
    user_data: UserUpdate,
    if_match: Optional[str] = Header(None, alias="If-Match")
):
    if user_id not in users_db:
        raise HTTPException(status_code=404)

    user = users_db[user_id]
    current_etag = f'"{user["version"]}"'

    if if_match and if_match != current_etag:
        raise HTTPException(
            status_code=412,
            detail="Resource has been modified"
        )

    user["version"] += 1
    # ... обновление

    response = JSONResponse(content=user)
    response.headers["ETag"] = f'"{user["version"]}"'
    return response
```

## Best Practices

### Что делать:

1. **Возвращайте созданный ресурс:**
   ```python
   # POST возвращает полный объект с ID и timestamps
   return {"id": "123", "name": "John", "created_at": "..."}
   ```

2. **Используйте правильные коды:**
   - 201 Created для POST
   - 200 OK для GET, PUT, PATCH
   - 204 No Content для DELETE

3. **Добавляйте Location header:**
   ```python
   response.headers["Location"] = f"/api/users/{user_id}"
   ```

4. **Валидируйте на входе:**
   - Используйте Pydantic/Joi/Zod

5. **Обрабатывайте конфликты:**
   - 409 для дубликатов
   - 412 для optimistic locking

### Типичные ошибки:

1. **POST возвращает только ID:**
   ```json
   {"id": "123"}  // Плохо — клиенту нужен полный объект
   ```

2. **PUT создаёт ресурс:**
   ```
   PUT /users/new-id → создаёт пользователя
   // Это допустимо, но нужно документировать
   ```

3. **DELETE возвращает 200 с телом:**
   ```
   // Лучше 204 No Content
   ```

4. **Нет валидации уникальности:**
   - Дубликаты в базе данных

## Заключение

CRUD операции — основа любого REST API:

- **POST** — создание с 201 Created
- **GET** — чтение с фильтрацией и пагинацией
- **PUT/PATCH** — обновление (полное/частичное)
- **DELETE** — удаление (жёсткое/мягкое)

Следуйте HTTP-семантике, валидируйте входные данные и обрабатывайте краевые случаи (конфликты, not found, concurrency).
