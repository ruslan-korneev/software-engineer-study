# JSON APIs

## Что такое JSON API?

**JSON API** — это спецификация для построения API, использующих JSON в качестве формата обмена данными. Существует два понятия:

1. **JSON API Specification** — конкретная спецификация (jsonapi.org), определяющая строгий формат запросов и ответов
2. **JSON-based APIs** — общий термин для любых API, использующих JSON

В этом документе мы рассмотрим оба аспекта.

## JSON как формат данных

### Почему JSON?

**JSON (JavaScript Object Notation)** стал стандартом де-факто для веб-API благодаря:

- **Человекочитаемости** — легко понять структуру данных
- **Простоте парсинга** — нативная поддержка в JavaScript, библиотеки для всех языков
- **Компактности** — меньше накладных расходов, чем XML
- **Широкой поддержке** — все современные языки и фреймворки

```json
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "roles": ["user", "admin"],
  "profile": {
    "age": 30,
    "city": "Moscow"
  },
  "active": true,
  "balance": 1500.50
}
```

### Типы данных в JSON

| Тип | Пример | Python эквивалент |
|-----|--------|-------------------|
| String | `"hello"` | `str` |
| Number | `42`, `3.14` | `int`, `float` |
| Boolean | `true`, `false` | `bool` |
| Null | `null` | `None` |
| Array | `[1, 2, 3]` | `list` |
| Object | `{"key": "value"}` | `dict` |

## JSON API Specification (jsonapi.org)

### Основные принципы

JSON:API — это спецификация, которая определяет:
- Структуру запросов и ответов
- Способы работы со связями между ресурсами
- Форматы ошибок
- Механизмы пагинации, сортировки, фильтрации

### Структура документа JSON:API

```json
{
  "data": {
    "type": "users",
    "id": "123",
    "attributes": {
      "name": "John Doe",
      "email": "john@example.com"
    },
    "relationships": {
      "posts": {
        "data": [
          {"type": "posts", "id": "1"},
          {"type": "posts", "id": "2"}
        ],
        "links": {
          "related": "/users/123/posts"
        }
      }
    },
    "links": {
      "self": "/users/123"
    }
  },
  "included": [
    {
      "type": "posts",
      "id": "1",
      "attributes": {
        "title": "First Post"
      }
    }
  ],
  "meta": {
    "total": 100
  },
  "links": {
    "self": "/users/123",
    "next": "/users?page[number]=2"
  }
}
```

### Ключевые элементы

#### data
Основные данные ответа — один ресурс или массив ресурсов.

```json
// Один ресурс
{
  "data": {
    "type": "users",
    "id": "1",
    "attributes": {"name": "John"}
  }
}

// Коллекция ресурсов
{
  "data": [
    {"type": "users", "id": "1", "attributes": {"name": "John"}},
    {"type": "users", "id": "2", "attributes": {"name": "Jane"}}
  ]
}
```

#### type и id
Каждый ресурс должен иметь `type` (тип ресурса) и `id` (уникальный идентификатор).

#### attributes
Атрибуты ресурса (данные, не являющиеся связями).

#### relationships
Связи с другими ресурсами.

```json
{
  "data": {
    "type": "posts",
    "id": "1",
    "attributes": {
      "title": "My Post"
    },
    "relationships": {
      "author": {
        "data": {"type": "users", "id": "5"}
      },
      "comments": {
        "data": [
          {"type": "comments", "id": "10"},
          {"type": "comments", "id": "11"}
        ]
      }
    }
  }
}
```

#### included
Связанные ресурсы, включённые в ответ (compound documents).

#### meta
Метаинформация (счётчики, версия API и т.д.).

#### links
Навигационные ссылки.

## Практический пример: JSON:API на FastAPI

```python
from fastapi import FastAPI, HTTPException, Query
from pydantic import BaseModel
from typing import Optional, Any

app = FastAPI()

# Вспомогательные функции для формирования JSON:API ответов
def make_resource(type_: str, id_: str, attributes: dict, relationships: dict = None):
    resource = {
        "type": type_,
        "id": str(id_),
        "attributes": attributes
    }
    if relationships:
        resource["relationships"] = relationships
    return resource

def make_response(data, included=None, meta=None, links=None):
    response = {"data": data}
    if included:
        response["included"] = included
    if meta:
        response["meta"] = meta
    if links:
        response["links"] = links
    return response

def make_error(status: str, title: str, detail: str, source: dict = None):
    error = {
        "status": status,
        "title": title,
        "detail": detail
    }
    if source:
        error["source"] = source
    return error

# Имитация данных
users_db = {
    "1": {"name": "John Doe", "email": "john@example.com"},
    "2": {"name": "Jane Smith", "email": "jane@example.com"}
}

posts_db = {
    "1": {"title": "First Post", "content": "Hello World", "author_id": "1"},
    "2": {"title": "Second Post", "content": "Another post", "author_id": "1"},
    "3": {"title": "Jane's Post", "content": "Hi there", "author_id": "2"}
}

# GET /users — список пользователей
@app.get("/users")
def get_users(
    page_number: int = Query(1, alias="page[number]"),
    page_size: int = Query(10, alias="page[size]"),
    include: Optional[str] = None
):
    users_list = list(users_db.items())
    total = len(users_list)

    # Пагинация
    start = (page_number - 1) * page_size
    end = start + page_size
    paginated = users_list[start:end]

    data = []
    included = []

    for user_id, user_attrs in paginated:
        # Найти посты пользователя
        user_posts = [
            {"type": "posts", "id": pid}
            for pid, post in posts_db.items()
            if post["author_id"] == user_id
        ]

        relationships = {
            "posts": {
                "data": user_posts,
                "links": {"related": f"/users/{user_id}/posts"}
            }
        }

        data.append(make_resource("users", user_id, user_attrs, relationships))

        # Включить связанные ресурсы
        if include == "posts":
            for pid, post in posts_db.items():
                if post["author_id"] == user_id:
                    post_attrs = {"title": post["title"], "content": post["content"]}
                    included.append(make_resource("posts", pid, post_attrs))

    links = {
        "self": f"/users?page[number]={page_number}&page[size]={page_size}",
        "first": f"/users?page[number]=1&page[size]={page_size}",
        "last": f"/users?page[number]={(total + page_size - 1) // page_size}&page[size]={page_size}"
    }

    if page_number > 1:
        links["prev"] = f"/users?page[number]={page_number - 1}&page[size]={page_size}"
    if end < total:
        links["next"] = f"/users?page[number]={page_number + 1}&page[size]={page_size}"

    return make_response(
        data,
        included=included if included else None,
        meta={"total": total},
        links=links
    )

# GET /users/{id} — один пользователь
@app.get("/users/{user_id}")
def get_user(user_id: str, include: Optional[str] = None):
    if user_id not in users_db:
        return {
            "errors": [
                make_error("404", "Not Found", f"User with id {user_id} not found")
            ]
        }

    user = users_db[user_id]
    user_posts = [
        {"type": "posts", "id": pid}
        for pid, post in posts_db.items()
        if post["author_id"] == user_id
    ]

    relationships = {
        "posts": {
            "data": user_posts,
            "links": {"related": f"/users/{user_id}/posts"}
        }
    }

    data = make_resource("users", user_id, user, relationships)
    included = []

    if include == "posts":
        for pid, post in posts_db.items():
            if post["author_id"] == user_id:
                post_attrs = {"title": post["title"], "content": post["content"]}
                included.append(make_resource("posts", pid, post_attrs))

    return make_response(
        data,
        included=included if included else None,
        links={"self": f"/users/{user_id}"}
    )

# POST /users — создание пользователя
@app.post("/users", status_code=201)
def create_user(body: dict):
    # JSON:API требует специфичный формат запроса
    if "data" not in body or body["data"].get("type") != "users":
        return {
            "errors": [
                make_error(
                    "400",
                    "Bad Request",
                    "Request must include data with type 'users'",
                    {"pointer": "/data/type"}
                )
            ]
        }

    attrs = body["data"].get("attributes", {})
    new_id = str(max(int(k) for k in users_db.keys()) + 1)
    users_db[new_id] = attrs

    return make_response(
        make_resource("users", new_id, attrs),
        links={"self": f"/users/{new_id}"}
    )
```

## Пример запроса на создание ресурса

```json
POST /users
Content-Type: application/vnd.api+json

{
  "data": {
    "type": "users",
    "attributes": {
      "name": "New User",
      "email": "new@example.com"
    }
  }
}
```

## Sparse Fieldsets (выборочные поля)

JSON:API позволяет клиенту запрашивать только нужные поля:

```
GET /users?fields[users]=name,email&fields[posts]=title
```

```python
@app.get("/users")
def get_users(
    fields_users: Optional[str] = Query(None, alias="fields[users]"),
    fields_posts: Optional[str] = Query(None, alias="fields[posts]")
):
    requested_user_fields = fields_users.split(",") if fields_users else None
    requested_post_fields = fields_posts.split(",") if fields_posts else None

    # Фильтруем атрибуты
    for user_id, user in users_db.items():
        if requested_user_fields:
            filtered = {k: v for k, v in user.items() if k in requested_user_fields}
        else:
            filtered = user
        # ...
```

## Сортировка и фильтрация

### Сортировка

```
GET /users?sort=name        # По возрастанию
GET /users?sort=-created_at # По убыванию (минус перед полем)
GET /users?sort=name,-age   # Несколько полей
```

### Фильтрация

```
GET /users?filter[status]=active
GET /users?filter[age][gte]=18
GET /posts?filter[author]=1
```

## Сравнение с другими стилями API

### JSON API vs Plain REST

| Критерий | JSON:API | Plain REST |
|----------|----------|------------|
| Структура ответа | Строго определена | Произвольная |
| Связанные ресурсы | `included` + `relationships` | Вложенные объекты или N+1 |
| Пагинация | Стандартизирована | Разные подходы |
| Ошибки | Единый формат | Произвольный |
| Размер ответа | Больше (много метаданных) | Компактнее |
| Кривая обучения | Выше | Ниже |

### JSON API vs GraphQL

| Критерий | JSON:API | GraphQL |
|----------|----------|---------|
| Выбор полей | `fields[type]=field1,field2` | Встроен в запрос |
| Связанные данные | `include=relation` | Вложенные запросы |
| Один endpoint | Нет (много endpoints) | Да |
| Introspection | Нет | Да |
| Мутации | HTTP методы | Mutations |

## Когда использовать JSON API

**Подходит для:**
- Проектов, где нужна стандартизация ответов
- Команд с несколькими клиентами (web, mobile, desktop)
- API с множеством связанных ресурсов
- Когда важна возможность включения связанных данных

**Не подходит для:**
- Простых API с несколькими endpoints
- Высоконагруженных систем (overhead от метаданных)
- Внутренних микросервисов
- Когда клиенты не готовы работать с спецификацией

## Best Practices

### 1. Используйте правильный Content-Type

```http
Content-Type: application/vnd.api+json
Accept: application/vnd.api+json
```

### 2. Консистентная структура ошибок

```json
{
  "errors": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "status": "422",
      "code": "validation_error",
      "title": "Validation Error",
      "detail": "Email is not valid",
      "source": {
        "pointer": "/data/attributes/email"
      },
      "meta": {
        "field": "email",
        "value": "invalid-email"
      }
    }
  ]
}
```

### 3. Эффективное использование include

```python
# Вместо N+1 запросов
GET /posts         # 1 запрос
GET /users/1       # N запросов для авторов
GET /users/2
...

# Один запрос с include
GET /posts?include=author
```

### 4. Кэширование с ETag

```python
from hashlib import md5

@app.get("/users/{user_id}")
def get_user(user_id: str, response: Response):
    user = users_db[user_id]
    etag = md5(json.dumps(user).encode()).hexdigest()
    response.headers["ETag"] = f'"{etag}"'
    return make_response(make_resource("users", user_id, user))
```

## Типичные ошибки

### 1. Неправильная структура запроса на создание

```json
// Плохо: атрибуты на верхнем уровне
{
  "name": "John",
  "email": "john@example.com"
}

// Хорошо: JSON:API структура
{
  "data": {
    "type": "users",
    "attributes": {
      "name": "John",
      "email": "john@example.com"
    }
  }
}
```

### 2. Забытый type в связях

```json
// Плохо: только id
{
  "relationships": {
    "author": {
      "data": {"id": "1"}
    }
  }
}

// Хорошо: type + id
{
  "relationships": {
    "author": {
      "data": {"type": "users", "id": "1"}
    }
  }
}
```

### 3. Смешивание атрибутов и связей

```json
// Плохо: author как атрибут
{
  "data": {
    "type": "posts",
    "attributes": {
      "title": "Post",
      "author": {"id": 1, "name": "John"}
    }
  }
}

// Хорошо: author как relationship + included
{
  "data": {
    "type": "posts",
    "attributes": {"title": "Post"},
    "relationships": {
      "author": {"data": {"type": "users", "id": "1"}}
    }
  },
  "included": [
    {"type": "users", "id": "1", "attributes": {"name": "John"}}
  ]
}
```

## Библиотеки для работы с JSON:API

### Python
- **marshmallow-jsonapi** — сериализация в JSON:API формат
- **Flask-JSONAPI** — интеграция с Flask
- **safrs** — SQLAlchemy + JSON:API

### JavaScript
- **jsonapi-serializer** — сериализация/десериализация
- **devour-client** — клиент для JSON:API

### Инструменты
- **JSON:API Playground** — тестирование запросов
- **OpenAPI JSON:API** — генерация OpenAPI спецификации

## Дополнительные ресурсы

- [JSON:API Specification](https://jsonapi.org/)
- [JSON:API Examples](https://jsonapi.org/examples/)
- [JSON Schema](https://json-schema.org/)
- [RFC 8259 - The JSON Data Interchange Format](https://datatracker.ietf.org/doc/html/rfc8259)
