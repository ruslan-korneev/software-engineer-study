# Pagination (Пагинация)

## Введение

**Пагинация** — это техника разбиения больших наборов данных на более мелкие части (страницы) для передачи клиенту. Без пагинации запрос миллионов записей может:

- Перегрузить сервер и базу данных
- Исчерпать память клиента
- Создать огромный объём сетевого трафика
- Привести к таймаутам

## Основные стратегии пагинации

### 1. Offset-based Pagination (Смещение)

Самый простой и популярный подход.

```http
GET /api/users?offset=20&limit=10
GET /api/users?page=3&per_page=10
```

**Как работает:**
- `offset` (или `skip`) — количество записей для пропуска
- `limit` (или `per_page`) — количество записей на странице

```
Страница 1: offset=0,  limit=10 → записи 1-10
Страница 2: offset=10, limit=10 → записи 11-20
Страница 3: offset=20, limit=10 → записи 21-30
```

#### SQL запрос:

```sql
SELECT * FROM users
ORDER BY created_at DESC
LIMIT 10 OFFSET 20;
```

#### Реализация (Python/FastAPI):

```python
from fastapi import FastAPI, Query
from typing import List, Optional
from pydantic import BaseModel

app = FastAPI()

class User(BaseModel):
    id: int
    name: str
    email: str

class PaginatedResponse(BaseModel):
    data: List[User]
    pagination: dict

@app.get("/api/users", response_model=PaginatedResponse)
async def get_users(
    page: int = Query(1, ge=1, description="Page number"),
    per_page: int = Query(20, ge=1, le=100, description="Items per page")
):
    # Вычисляем offset
    offset = (page - 1) * per_page

    # Запрос к БД (псевдокод)
    # users = db.query(User).offset(offset).limit(per_page).all()
    # total = db.query(User).count()

    users = []  # Заглушка
    total = 150

    total_pages = (total + per_page - 1) // per_page

    return {
        "data": users,
        "pagination": {
            "page": page,
            "per_page": per_page,
            "total": total,
            "total_pages": total_pages,
            "has_next": page < total_pages,
            "has_prev": page > 1
        }
    }
```

#### Реализация (Node.js/Express):

```javascript
const express = require('express');
const app = express();

app.get('/api/users', async (req, res) => {
    const page = Math.max(1, parseInt(req.query.page) || 1);
    const perPage = Math.min(100, Math.max(1, parseInt(req.query.per_page) || 20));
    const offset = (page - 1) * perPage;

    // Запрос к БД (псевдокод)
    // const users = await User.find().skip(offset).limit(perPage);
    // const total = await User.countDocuments();

    const users = [];
    const total = 150;
    const totalPages = Math.ceil(total / perPage);

    res.json({
        data: users,
        pagination: {
            page,
            per_page: perPage,
            total,
            total_pages: totalPages,
            has_next: page < totalPages,
            has_prev: page > 1
        }
    });
});

app.listen(3000);
```

**Преимущества:**
- Простота реализации
- Возможность перехода к любой странице
- Понятный для пользователей интерфейс

**Недостатки:**
- Производительность падает на больших offset'ах
- Проблемы при добавлении/удалении данных (пропуск или дублирование)
- Неэффективно для миллионов записей

### 2. Cursor-based Pagination (Курсорная пагинация)

Использует уникальный идентификатор (курсор) для определения позиции.

```http
GET /api/users?cursor=eyJpZCI6MTAwfQ&limit=10
```

**Как работает:**
- Курсор кодирует позицию последнего элемента
- Следующая страница начинается после этого элемента

```
Запрос 1: GET /users?limit=10
Ответ:    [...10 пользователей...], next_cursor: "abc123"

Запрос 2: GET /users?cursor=abc123&limit=10
Ответ:    [...следующие 10 пользователей...], next_cursor: "def456"
```

#### SQL запрос:

```sql
-- Вместо OFFSET используем WHERE
SELECT * FROM users
WHERE id > 100  -- курсор указывает на ID последнего элемента
ORDER BY id ASC
LIMIT 10;
```

#### Реализация (Python/FastAPI):

```python
from fastapi import FastAPI, Query
from typing import Optional
import base64
import json

app = FastAPI()

def encode_cursor(data: dict) -> str:
    """Кодирует данные курсора в Base64."""
    return base64.urlsafe_b64encode(
        json.dumps(data).encode()
    ).decode()

def decode_cursor(cursor: str) -> dict:
    """Декодирует курсор из Base64."""
    try:
        return json.loads(
            base64.urlsafe_b64decode(cursor.encode()).decode()
        )
    except Exception:
        return {}

@app.get("/api/users")
async def get_users(
    cursor: Optional[str] = Query(None, description="Pagination cursor"),
    limit: int = Query(20, ge=1, le=100)
):
    # Декодируем курсор
    cursor_data = decode_cursor(cursor) if cursor else {}
    last_id = cursor_data.get("id", 0)

    # Запрос к БД (псевдокод)
    # users = db.query(User)\
    #     .filter(User.id > last_id)\
    #     .order_by(User.id)\
    #     .limit(limit + 1)\
    #     .all()

    # Заглушка
    users = [{"id": i, "name": f"User {i}"} for i in range(last_id + 1, last_id + limit + 2)]

    # Проверяем, есть ли следующая страница
    has_next = len(users) > limit
    if has_next:
        users = users[:limit]  # Убираем лишний элемент

    # Создаём курсор для следующей страницы
    next_cursor = None
    if has_next and users:
        next_cursor = encode_cursor({"id": users[-1]["id"]})

    return {
        "data": users,
        "pagination": {
            "next_cursor": next_cursor,
            "has_next": has_next,
            "limit": limit
        }
    }
```

#### Реализация (Node.js/Express):

```javascript
const express = require('express');
const app = express();

// Кодирование/декодирование курсора
const encodeCursor = (data) => {
    return Buffer.from(JSON.stringify(data)).toString('base64url');
};

const decodeCursor = (cursor) => {
    try {
        return JSON.parse(Buffer.from(cursor, 'base64url').toString());
    } catch {
        return {};
    }
};

app.get('/api/users', async (req, res) => {
    const limit = Math.min(100, Math.max(1, parseInt(req.query.limit) || 20));
    const cursorData = req.query.cursor ? decodeCursor(req.query.cursor) : {};
    const lastId = cursorData.id || 0;

    // Запрос к БД (псевдокод)
    // const users = await User.find({ _id: { $gt: lastId } })
    //     .sort({ _id: 1 })
    //     .limit(limit + 1);

    // Заглушка
    const users = Array.from({ length: limit + 1 }, (_, i) => ({
        id: lastId + i + 1,
        name: `User ${lastId + i + 1}`
    }));

    const hasNext = users.length > limit;
    if (hasNext) users.pop();

    const nextCursor = hasNext && users.length > 0
        ? encodeCursor({ id: users[users.length - 1].id })
        : null;

    res.json({
        data: users,
        pagination: {
            next_cursor: nextCursor,
            has_next: hasNext,
            limit
        }
    });
});

app.listen(3000);
```

**Преимущества:**
- Высокая производительность (использует индексы)
- Стабильность при изменении данных
- Идеально для бесконечной прокрутки

**Недостатки:**
- Нельзя перейти к произвольной странице
- Сложнее реализовать
- Курсор может стать невалидным

### 3. Keyset Pagination (Pagination по ключу)

Вариация курсорной пагинации с явным указанием ключа.

```http
GET /api/users?created_after=2024-01-15T10:30:00Z&limit=10
```

```sql
SELECT * FROM users
WHERE created_at > '2024-01-15T10:30:00Z'
ORDER BY created_at ASC
LIMIT 10;
```

**Для сортировки по нескольким полям:**

```sql
-- Сортировка по created_at DESC, id DESC
SELECT * FROM users
WHERE (created_at, id) < ('2024-01-15T10:30:00Z', 100)
ORDER BY created_at DESC, id DESC
LIMIT 10;
```

### 4. Seek Pagination (Time-based)

Пагинация на основе временных меток.

```http
GET /api/messages?before=2024-01-15T10:30:00Z&limit=50
GET /api/messages?after=2024-01-15T10:30:00Z&limit=50
```

Используется в:
- Чатах и мессенджерах
- Лентах новостей
- Логах событий

```python
@app.get("/api/messages")
async def get_messages(
    before: Optional[datetime] = None,
    after: Optional[datetime] = None,
    limit: int = Query(50, le=100)
):
    query = db.query(Message)

    if before:
        query = query.filter(Message.created_at < before)
    if after:
        query = query.filter(Message.created_at > after)

    messages = query.order_by(Message.created_at.desc()).limit(limit).all()

    return {
        "data": messages,
        "pagination": {
            "oldest": messages[-1].created_at if messages else None,
            "newest": messages[0].created_at if messages else None
        }
    }
```

## Формат ответа с пагинацией

### Простой формат

```json
{
    "data": [
        {"id": 1, "name": "User 1"},
        {"id": 2, "name": "User 2"}
    ],
    "page": 1,
    "per_page": 20,
    "total": 150
}
```

### Расширенный формат

```json
{
    "data": [
        {"id": 1, "name": "User 1"},
        {"id": 2, "name": "User 2"}
    ],
    "pagination": {
        "page": 2,
        "per_page": 20,
        "total": 150,
        "total_pages": 8,
        "has_next": true,
        "has_prev": true
    },
    "links": {
        "self": "/api/users?page=2&per_page=20",
        "first": "/api/users?page=1&per_page=20",
        "prev": "/api/users?page=1&per_page=20",
        "next": "/api/users?page=3&per_page=20",
        "last": "/api/users?page=8&per_page=20"
    }
}
```

### Формат с курсором

```json
{
    "data": [
        {"id": 101, "name": "User 101"},
        {"id": 102, "name": "User 102"}
    ],
    "pagination": {
        "next_cursor": "eyJpZCI6MTAyfQ",
        "prev_cursor": "eyJpZCI6MTAxLCJkaXIiOiJiYWNrIn0",
        "has_next": true,
        "has_prev": true
    }
}
```

## HTTP Headers для пагинации

Некоторые API используют заголовки вместо тела ответа:

```http
HTTP/1.1 200 OK
X-Total-Count: 150
X-Page: 2
X-Per-Page: 20
X-Total-Pages: 8
Link: <https://api.example.com/users?page=1>; rel="first",
      <https://api.example.com/users?page=1>; rel="prev",
      <https://api.example.com/users?page=3>; rel="next",
      <https://api.example.com/users?page=8>; rel="last"
```

### Реализация Link header:

```python
from fastapi import FastAPI, Response

@app.get("/api/users")
async def get_users(
    response: Response,
    page: int = 1,
    per_page: int = 20
):
    total = 150
    total_pages = (total + per_page - 1) // per_page

    # Формируем Link header
    base_url = "/api/users"
    links = []
    links.append(f'<{base_url}?page=1&per_page={per_page}>; rel="first"')
    links.append(f'<{base_url}?page={total_pages}&per_page={per_page}>; rel="last"')

    if page > 1:
        links.append(f'<{base_url}?page={page-1}&per_page={per_page}>; rel="prev"')
    if page < total_pages:
        links.append(f'<{base_url}?page={page+1}&per_page={per_page}>; rel="next"')

    response.headers["Link"] = ", ".join(links)
    response.headers["X-Total-Count"] = str(total)
    response.headers["X-Total-Pages"] = str(total_pages)

    return {"data": [], "page": page, "per_page": per_page}
```

## Сравнение стратегий

| Критерий | Offset | Cursor | Keyset |
|----------|--------|--------|--------|
| Простота | ★★★★★ | ★★★☆☆ | ★★★☆☆ |
| Производительность | ★★☆☆☆ | ★★★★★ | ★★★★★ |
| Произвольный доступ | ★★★★★ | ☆☆☆☆☆ | ☆☆☆☆☆ |
| Стабильность данных | ★★☆☆☆ | ★★★★★ | ★★★★★ |
| Real-time данные | ★★☆☆☆ | ★★★★★ | ★★★★★ |

## Примеры из реальных API

### GitHub API (Link header + per_page)

```http
GET https://api.github.com/users?per_page=30&since=135

HTTP/1.1 200 OK
Link: <https://api.github.com/users?since=135>; rel="next",
      <https://api.github.com/users{?since}>; rel="first"
```

### Stripe API (Cursor-based)

```http
GET https://api.stripe.com/v1/customers?limit=3&starting_after=cus_ABC123

{
    "data": [...],
    "has_more": true,
    "url": "/v1/customers"
}
```

### Twitter API v2 (Pagination token)

```http
GET https://api.twitter.com/2/users/12345/tweets?pagination_token=xyz789

{
    "data": [...],
    "meta": {
        "next_token": "abc123",
        "result_count": 10
    }
}
```

### Slack API (Cursor-based)

```http
GET https://slack.com/api/conversations.list?cursor=dXNlcjpVMDYxTkZUVDI

{
    "channels": [...],
    "response_metadata": {
        "next_cursor": "dGVhbTpDMDYxRkE1UEI"
    }
}
```

## Best Practices

### Что делать:

1. **Устанавливайте разумные лимиты:**
   ```python
   limit: int = Query(20, ge=1, le=100)  # Не более 100
   ```

2. **Всегда сортируйте данные:**
   - Без сортировки пагинация непредсказуема

3. **Возвращайте метаданные:**
   - Общее количество, текущая страница, наличие следующей

4. **Используйте курсоры для больших данных:**
   - Offset неэффективен при offset > 10000

5. **Предоставляйте навигационные ссылки:**
   - HATEOAS links или Link header

6. **Документируйте ограничения:**
   - Максимальный limit, формат курсора

### Типичные ошибки:

1. **Отсутствие пагинации:**
   - Возвращение всех данных одним запросом

2. **Огромные лимиты:**
   - `?limit=1000000` — нужна валидация

3. **Offset без сортировки:**
   - Результаты могут дублироваться или пропускаться

4. **Неэффективные запросы:**
   ```sql
   -- Плохо: сканирует все записи
   SELECT * FROM logs ORDER BY id OFFSET 1000000 LIMIT 10;

   -- Хорошо: использует индекс
   SELECT * FROM logs WHERE id > 1000000 ORDER BY id LIMIT 10;
   ```

5. **Отсутствие информации о has_next:**
   - Клиент не знает, когда остановиться

## Оптимизация производительности

### Проблема больших offset'ов

```sql
-- На миллионной записи это очень медленно
SELECT * FROM users ORDER BY id OFFSET 1000000 LIMIT 10;
```

### Решение: Deferred Join

```sql
-- Сначала получаем только ID с использованием индекса
SELECT u.*
FROM users u
INNER JOIN (
    SELECT id FROM users ORDER BY id LIMIT 10 OFFSET 1000000
) AS tmp ON u.id = tmp.id;
```

### Решение: Cursor pagination

```sql
-- Быстро благодаря индексу по id
SELECT * FROM users WHERE id > 1000000 ORDER BY id LIMIT 10;
```

## Заключение

Выбор стратегии пагинации зависит от use case:

- **Offset** — для небольших наборов данных с UI-навигацией
- **Cursor** — для больших данных, бесконечной прокрутки, API
- **Keyset** — когда нужна сортировка по нескольким полям

**Рекомендация:** Начните с offset-пагинации для простоты, но будьте готовы переходить на cursor-based при росте данных.
