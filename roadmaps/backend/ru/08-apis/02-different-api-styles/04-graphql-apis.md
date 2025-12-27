# GraphQL APIs

## Что такое GraphQL?

**GraphQL** — это язык запросов для API и среда выполнения для обработки этих запросов. Разработан Facebook в 2012 году и открыт в 2015. GraphQL предоставляет полное и понятное описание данных в API, даёт клиентам возможность запрашивать именно те данные, которые им нужны.

## Основные концепции

### Ключевые преимущества

1. **Запрашивай только нужное** — клиент определяет структуру ответа
2. **Один endpoint** — все запросы идут на `/graphql`
3. **Строгая типизация** — схема описывает все типы данных
4. **Интроспекция** — API самодокументируемо
5. **Эволюция без версий** — добавляй поля без breaking changes

### Архитектура

```
┌─────────────┐      GraphQL Query       ┌──────────────┐
│   Client    │ ───────────────────────▶ │   GraphQL    │
│  (Web/App)  │                          │   Server     │
│             │ ◀─────────────────────── │              │
└─────────────┘      JSON Response       └──────────────┘
                                                │
                    ┌───────────────────────────┼───────────────────────────┐
                    ▼                           ▼                           ▼
              ┌──────────┐               ┌──────────┐               ┌──────────┐
              │ Database │               │   REST   │               │  Other   │
              │          │               │   API    │               │ Services │
              └──────────┘               └──────────┘               └──────────┘
```

## Типы операций

### 1. Query (Запрос данных)

```graphql
# Запрос
query GetUser {
  user(id: "123") {
    id
    name
    email
    posts {
      title
      createdAt
    }
  }
}

# Ответ
{
  "data": {
    "user": {
      "id": "123",
      "name": "John Doe",
      "email": "john@example.com",
      "posts": [
        {"title": "First Post", "createdAt": "2024-01-01"},
        {"title": "Second Post", "createdAt": "2024-01-15"}
      ]
    }
  }
}
```

### 2. Mutation (Изменение данных)

```graphql
# Запрос
mutation CreateUser {
  createUser(input: {
    name: "Jane Doe"
    email: "jane@example.com"
  }) {
    id
    name
    email
  }
}

# Ответ
{
  "data": {
    "createUser": {
      "id": "456",
      "name": "Jane Doe",
      "email": "jane@example.com"
    }
  }
}
```

### 3. Subscription (Подписки на события)

```graphql
# Подписка
subscription OnNewMessage {
  messageCreated(channelId: "general") {
    id
    content
    author {
      name
    }
    createdAt
  }
}

# Сервер отправляет данные при каждом новом сообщении
{
  "data": {
    "messageCreated": {
      "id": "789",
      "content": "Hello!",
      "author": {"name": "John"},
      "createdAt": "2024-01-20T10:30:00Z"
    }
  }
}
```

## Схема GraphQL (SDL)

### Определение типов

```graphql
# Скалярные типы
scalar DateTime

# Enum
enum UserStatus {
  ACTIVE
  INACTIVE
  BANNED
}

# Object Type
type User {
  id: ID!
  name: String!
  email: String!
  status: UserStatus!
  createdAt: DateTime!
  posts: [Post!]!
  followers: [User!]!
  followersCount: Int!
}

type Post {
  id: ID!
  title: String!
  content: String
  author: User!
  comments: [Comment!]!
  tags: [String!]!
  published: Boolean!
  createdAt: DateTime!
}

type Comment {
  id: ID!
  text: String!
  author: User!
  post: Post!
}

# Input Types (для mutations)
input CreateUserInput {
  name: String!
  email: String!
  password: String!
}

input UpdateUserInput {
  name: String
  email: String
}

input PostFilterInput {
  authorId: ID
  published: Boolean
  tags: [String!]
}

# Query Type
type Query {
  user(id: ID!): User
  users(limit: Int = 10, offset: Int = 0): [User!]!
  post(id: ID!): Post
  posts(filter: PostFilterInput, limit: Int = 10): [Post!]!
  me: User
}

# Mutation Type
type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User
  deleteUser(id: ID!): Boolean!
  createPost(title: String!, content: String): Post!
  publishPost(id: ID!): Post
}

# Subscription Type
type Subscription {
  userCreated: User!
  postPublished: Post!
}
```

### Модификаторы типов

| Синтаксис | Описание |
|-----------|----------|
| `String` | Nullable строка |
| `String!` | Non-null строка (обязательное поле) |
| `[String]` | Nullable список nullable строк |
| `[String!]` | Nullable список non-null строк |
| `[String!]!` | Non-null список non-null строк |

## Практический пример: GraphQL на Python (Strawberry)

### Установка

```bash
pip install strawberry-graphql[fastapi] uvicorn
```

### Определение схемы

```python
# schema.py
import strawberry
from strawberry.types import Info
from typing import Optional
from datetime import datetime
from enum import Enum

# Enums
@strawberry.enum
class UserStatus(Enum):
    ACTIVE = "active"
    INACTIVE = "inactive"
    BANNED = "banned"

# Types
@strawberry.type
class User:
    id: strawberry.ID
    name: str
    email: str
    status: UserStatus
    created_at: datetime

    @strawberry.field
    def posts(self, info: Info) -> list["Post"]:
        # Resolve posts for this user
        return get_posts_by_user(self.id)

    @strawberry.field
    def followers_count(self) -> int:
        return get_followers_count(self.id)

@strawberry.type
class Post:
    id: strawberry.ID
    title: str
    content: Optional[str]
    published: bool
    created_at: datetime
    author_id: strawberry.Private[str]  # Hidden from schema

    @strawberry.field
    def author(self, info: Info) -> User:
        return get_user_by_id(self.author_id)

    @strawberry.field
    def comments(self, limit: int = 10) -> list["Comment"]:
        return get_comments_for_post(self.id, limit)

@strawberry.type
class Comment:
    id: strawberry.ID
    text: str
    author: User
    created_at: datetime

# Input Types
@strawberry.input
class CreateUserInput:
    name: str
    email: str
    password: str

@strawberry.input
class UpdateUserInput:
    name: Optional[str] = None
    email: Optional[str] = None

@strawberry.input
class PostFilterInput:
    author_id: Optional[strawberry.ID] = None
    published: Optional[bool] = None
    tags: Optional[list[str]] = None

# Имитация базы данных
users_db: dict[str, dict] = {
    "1": {"id": "1", "name": "John Doe", "email": "john@example.com",
          "status": UserStatus.ACTIVE, "created_at": datetime.now()},
    "2": {"id": "2", "name": "Jane Smith", "email": "jane@example.com",
          "status": UserStatus.ACTIVE, "created_at": datetime.now()},
}

posts_db: dict[str, dict] = {
    "1": {"id": "1", "title": "First Post", "content": "Hello World",
          "author_id": "1", "published": True, "created_at": datetime.now()},
    "2": {"id": "2", "title": "Draft Post", "content": "Work in progress",
          "author_id": "1", "published": False, "created_at": datetime.now()},
}

# Helper functions
def get_user_by_id(user_id: str) -> Optional[User]:
    if user_id in users_db:
        return User(**users_db[user_id])
    return None

def get_posts_by_user(user_id: str) -> list[Post]:
    return [
        Post(**post)
        for post in posts_db.values()
        if post["author_id"] == user_id
    ]

def get_followers_count(user_id: str) -> int:
    return 42  # Заглушка

def get_comments_for_post(post_id: str, limit: int) -> list[Comment]:
    return []  # Заглушка

# Query
@strawberry.type
class Query:
    @strawberry.field
    def user(self, id: strawberry.ID) -> Optional[User]:
        return get_user_by_id(id)

    @strawberry.field
    def users(self, limit: int = 10, offset: int = 0) -> list[User]:
        all_users = [User(**u) for u in users_db.values()]
        return all_users[offset:offset + limit]

    @strawberry.field
    def post(self, id: strawberry.ID) -> Optional[Post]:
        if id in posts_db:
            return Post(**posts_db[id])
        return None

    @strawberry.field
    def posts(
        self,
        filter: Optional[PostFilterInput] = None,
        limit: int = 10
    ) -> list[Post]:
        result = list(posts_db.values())

        if filter:
            if filter.author_id:
                result = [p for p in result if p["author_id"] == filter.author_id]
            if filter.published is not None:
                result = [p for p in result if p["published"] == filter.published]

        return [Post(**p) for p in result[:limit]]

# Mutation
@strawberry.type
class Mutation:
    @strawberry.mutation
    def create_user(self, input: CreateUserInput) -> User:
        new_id = str(len(users_db) + 1)
        new_user = {
            "id": new_id,
            "name": input.name,
            "email": input.email,
            "status": UserStatus.ACTIVE,
            "created_at": datetime.now()
        }
        users_db[new_id] = new_user
        return User(**new_user)

    @strawberry.mutation
    def update_user(
        self,
        id: strawberry.ID,
        input: UpdateUserInput
    ) -> Optional[User]:
        if id not in users_db:
            return None

        user = users_db[id]
        if input.name:
            user["name"] = input.name
        if input.email:
            user["email"] = input.email

        return User(**user)

    @strawberry.mutation
    def delete_user(self, id: strawberry.ID) -> bool:
        if id in users_db:
            del users_db[id]
            return True
        return False

    @strawberry.mutation
    def create_post(
        self,
        title: str,
        content: Optional[str] = None
    ) -> Post:
        new_id = str(len(posts_db) + 1)
        new_post = {
            "id": new_id,
            "title": title,
            "content": content,
            "author_id": "1",  # TODO: get from context
            "published": False,
            "created_at": datetime.now()
        }
        posts_db[new_id] = new_post
        return Post(**new_post)

# Schema
schema = strawberry.Schema(query=Query, mutation=Mutation)
```

### FastAPI интеграция

```python
# main.py
from fastapi import FastAPI
from strawberry.fastapi import GraphQLRouter
from schema import schema

app = FastAPI()

graphql_app = GraphQLRouter(schema)
app.include_router(graphql_app, prefix="/graphql")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Продвинутые возможности

### Fragments (Фрагменты)

Переиспользуемые части запроса:

```graphql
# Определение фрагмента
fragment UserBasicInfo on User {
  id
  name
  email
}

# Использование
query GetUsers {
  user1: user(id: "1") {
    ...UserBasicInfo
    posts {
      title
    }
  }
  user2: user(id: "2") {
    ...UserBasicInfo
  }
}
```

### Variables (Переменные)

```graphql
# Запрос с переменными
query GetUser($userId: ID!) {
  user(id: $userId) {
    id
    name
    email
  }
}

# Переменные (отправляются отдельно)
{
  "userId": "123"
}
```

### Directives (Директивы)

```graphql
query GetUser($userId: ID!, $includeEmail: Boolean!, $skipPosts: Boolean!) {
  user(id: $userId) {
    id
    name
    email @include(if: $includeEmail)
    posts @skip(if: $skipPosts) {
      title
    }
  }
}
```

### Aliases (Псевдонимы)

```graphql
query GetTwoUsers {
  john: user(id: "1") {
    name
    email
  }
  jane: user(id: "2") {
    name
    email
  }
}
```

### Interfaces и Unions

```graphql
# Interface
interface Node {
  id: ID!
}

type User implements Node {
  id: ID!
  name: String!
}

type Post implements Node {
  id: ID!
  title: String!
}

# Union
union SearchResult = User | Post | Comment

type Query {
  search(query: String!): [SearchResult!]!
}

# Запрос с Union
query Search {
  search(query: "hello") {
    ... on User {
      name
      email
    }
    ... on Post {
      title
      author {
        name
      }
    }
    ... on Comment {
      text
    }
  }
}
```

## DataLoader — решение N+1 проблемы

```python
from strawberry.dataloader import DataLoader
from typing import List

# DataLoader для пакетной загрузки пользователей
async def load_users(user_ids: List[str]) -> List[User]:
    # Один запрос к БД вместо N
    users = await db.fetch_users_by_ids(user_ids)
    # Сохраняем порядок
    user_map = {u.id: u for u in users}
    return [user_map.get(uid) for uid in user_ids]

user_loader = DataLoader(load_fn=load_users)

@strawberry.type
class Post:
    id: strawberry.ID
    author_id: strawberry.Private[str]

    @strawberry.field
    async def author(self, info: Info) -> User:
        # DataLoader автоматически батчит запросы
        return await info.context["user_loader"].load(self.author_id)

# Context с DataLoader
async def get_context():
    return {
        "user_loader": DataLoader(load_fn=load_users)
    }
```

## Subscriptions (WebSocket)

```python
import asyncio
from typing import AsyncGenerator
import strawberry

@strawberry.type
class Subscription:
    @strawberry.subscription
    async def count(self, target: int = 10) -> AsyncGenerator[int, None]:
        for i in range(target):
            yield i
            await asyncio.sleep(1)

    @strawberry.subscription
    async def messages(
        self,
        channel_id: str
    ) -> AsyncGenerator[Message, None]:
        async for message in message_broker.subscribe(channel_id):
            yield message
```

## Сравнение с другими стилями API

### GraphQL vs REST

| Критерий | GraphQL | REST |
|----------|---------|------|
| Endpoints | Один `/graphql` | Множество |
| Over-fetching | Нет | Часто |
| Under-fetching | Нет | Часто (N+1) |
| Кэширование | Сложное | Простое (HTTP cache) |
| Версионирование | Эволюция схемы | URL версии |
| Типизация | Обязательная | Опциональная |
| Introspection | Встроена | OpenAPI отдельно |
| Learning curve | Выше | Ниже |
| Tooling | GraphiQL, Apollo | Swagger, Postman |

### GraphQL vs gRPC

| Критерий | GraphQL | gRPC |
|----------|---------|------|
| Формат | JSON | Protocol Buffers |
| Протокол | HTTP/1.1, WebSocket | HTTP/2 |
| Производительность | Средняя | Высокая |
| Гибкость запросов | Высокая | Фиксированные методы |
| Browser support | Полная | Ограниченная |
| Streaming | Subscriptions | Bidirectional |
| Use case | Клиентские API | Межсервисное общение |

## Когда использовать GraphQL

**Подходит для:**
- **Мобильные приложения** — минимизация трафика
- **Сложные связанные данные** — один запрос для вложенных структур
- **Быстрая итерация** — не ломать клиентов при изменениях
- **BFF (Backend for Frontend)** — разные клиенты, разные потребности
- **Real-time приложения** — subscriptions

**Не подходит для:**
- Простых CRUD API
- Файловых операций (upload/download)
- Межсервисного общения (используй gRPC)
- Когда важно HTTP-кэширование

## Best Practices

### 1. Пагинация (Cursor-based)

```graphql
type Query {
  users(first: Int, after: String): UserConnection!
}

type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type UserEdge {
  node: User!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

### 2. Обработка ошибок

```graphql
type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload!
}

type CreateUserPayload {
  user: User
  errors: [CreateUserError!]!
}

type CreateUserError {
  field: String!
  message: String!
}
```

```python
@strawberry.type
class CreateUserPayload:
    user: Optional[User]
    errors: list["CreateUserError"]

@strawberry.mutation
def create_user(self, input: CreateUserInput) -> CreateUserPayload:
    errors = []

    if not is_valid_email(input.email):
        errors.append(CreateUserError(
            field="email",
            message="Invalid email format"
        ))

    if email_exists(input.email):
        errors.append(CreateUserError(
            field="email",
            message="Email already registered"
        ))

    if errors:
        return CreateUserPayload(user=None, errors=errors)

    user = create_user_in_db(input)
    return CreateUserPayload(user=user, errors=[])
```

### 3. Ограничение глубины запросов

```python
from strawberry.extensions import QueryDepthLimiter

schema = strawberry.Schema(
    query=Query,
    extensions=[
        QueryDepthLimiter(max_depth=10)
    ]
)
```

### 4. Rate Limiting и Complexity Analysis

```python
from strawberry.extensions import MaxAliasesLimiter

schema = strawberry.Schema(
    query=Query,
    extensions=[
        QueryDepthLimiter(max_depth=10),
        MaxAliasesLimiter(max_alias_count=15)
    ]
)
```

## Типичные ошибки

### 1. Игнорирование N+1 проблемы

```python
# Плохо: N+1 запросов
@strawberry.field
def author(self) -> User:
    return db.get_user(self.author_id)  # Запрос для каждого поста!

# Хорошо: DataLoader
@strawberry.field
async def author(self, info: Info) -> User:
    return await info.context["user_loader"].load(self.author_id)
```

### 2. Слишком большие запросы

```graphql
# Опасно: рекурсивный запрос
query DangerousQuery {
  users {
    followers {
      followers {
        followers {
          # Бесконечная вложенность...
        }
      }
    }
  }
}
```

Решение: ограничение глубины и сложности запросов.

### 3. Мутации без валидации

```python
# Плохо: доверяем input
@strawberry.mutation
def update_user(self, id: str, email: str) -> User:
    return db.update_user(id, email=email)

# Хорошо: валидация
@strawberry.mutation
def update_user(self, id: str, email: str) -> UpdateUserPayload:
    if not is_valid_email(email):
        return UpdateUserPayload(
            user=None,
            errors=[Error(field="email", message="Invalid email")]
        )
    user = db.update_user(id, email=email)
    return UpdateUserPayload(user=user, errors=[])
```

### 4. Отсутствие авторизации на уровне полей

```python
@strawberry.type
class User:
    id: strawberry.ID
    name: str

    @strawberry.field
    def email(self, info: Info) -> Optional[str]:
        # Проверка прав доступа
        current_user = info.context.get("user")
        if not current_user or current_user.id != self.id:
            return None  # Скрываем email от других пользователей
        return self._email
```

## Инструменты

### IDE и Playground
- **GraphiQL** — встроенный playground
- **Apollo Studio** — продвинутая IDE
- **Altair GraphQL Client** — десктоп клиент

### Тестирование
- **pytest-strawberry** — тестирование Strawberry схем
- **graphql-test** — генерация тестов

### Мониторинг
- **Apollo Tracing** — трейсинг запросов
- **GraphQL Voyager** — визуализация схемы

## Дополнительные ресурсы

- [GraphQL Specification](https://spec.graphql.org/)
- [Strawberry Documentation](https://strawberry.rocks/)
- [Apollo GraphQL](https://www.apollographql.com/docs/)
- [How to GraphQL](https://www.howtographql.com/)
- [GraphQL Best Practices](https://graphql.org/learn/best-practices/)
