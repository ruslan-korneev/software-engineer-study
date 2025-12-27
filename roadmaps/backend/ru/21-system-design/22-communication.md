# Communication (Коммуникация между сервисами)

## Введение

**Межсервисная коммуникация** — это способ обмена данными между компонентами распределённой системы. Правильный выбор паттерна коммуникации критически важен для производительности, надёжности и масштабируемости системы.

### Основные вопросы при выборе

1. **Синхронная или асинхронная?** — Нужен ли немедленный ответ?
2. **Какой протокол?** — REST, gRPC, GraphQL, WebSocket?
3. **Как обрабатывать ошибки?** — Retry, circuit breaker, fallback?
4. **Как масштабировать?** — Load balancing, service discovery?

---

## Синхронная vs Асинхронная коммуникация

### Синхронная коммуникация

Клиент отправляет запрос и **ждёт** ответа. Поток заблокирован до получения результата.

```
┌────────┐         ┌─────────┐
│ Client │──req───▶│ Server  │
│(ждёт)  │◀──res───│         │
└────────┘         └─────────┘
```

**Характеристики:**
- Простая модель программирования
- Клиент получает результат сразу
- Сервисы связаны (coupled)
- Cascade failures — сбой одного сервиса влияет на другие

```python
# Синхронный вызов
import requests

def get_user_orders(user_id):
    # Ждём ответа от каждого сервиса последовательно
    user = requests.get(f"http://user-service/users/{user_id}").json()
    orders = requests.get(f"http://order-service/orders?user_id={user_id}").json()
    payments = requests.get(f"http://payment-service/payments?user_id={user_id}").json()

    return {
        "user": user,
        "orders": orders,
        "payments": payments
    }
    # Общее время = время user + время orders + время payments
```

### Асинхронная коммуникация

Клиент отправляет запрос и **продолжает работу**, не дожидаясь ответа.

```
┌────────┐         ┌───────────┐         ┌─────────┐
│ Client │──msg───▶│   Queue   │──msg───▶│ Server  │
│(работает)│        └───────────┘         └─────────┘
└────────┘
```

**Характеристики:**
- Развязка сервисов (decoupling)
- Устойчивость к сбоям
- Буферизация нагрузки
- Сложнее отслеживать и отлаживать

```python
# Асинхронный вызов через очередь
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode()
)

def process_order_async(order_data):
    # Отправляем событие и сразу возвращаем
    producer.send('order-events', {
        'type': 'ORDER_CREATED',
        'data': order_data
    })

    return {"status": "processing", "message": "Order is being processed"}
    # Клиент получает ответ мгновенно
```

### Сравнение

| Аспект | Синхронная | Асинхронная |
|--------|-----------|-------------|
| Latency | Зависит от всех сервисов | Низкая (для клиента) |
| Coupling | Высокое | Низкое |
| Reliability | Cascade failures | Изоляция сбоев |
| Complexity | Простая | Сложнее |
| Debugging | Проще | Сложнее |
| Use cases | CRUD, real-time | Events, tasks, workflows |

---

## REST API

**REST (Representational State Transfer)** — архитектурный стиль для построения веб-сервисов на основе HTTP.

### Принципы REST

1. **Stateless** — каждый запрос содержит всю информацию
2. **Client-Server** — разделение обязанностей
3. **Uniform Interface** — стандартные HTTP методы
4. **Cacheable** — ответы могут кэшироваться
5. **Layered System** — клиент не знает, с чем общается напрямую

### HTTP методы

| Метод | Действие | Идемпотентность | Безопасность |
|-------|----------|-----------------|--------------|
| GET | Чтение | Да | Да |
| POST | Создание | Нет | Нет |
| PUT | Полное обновление | Да | Нет |
| PATCH | Частичное обновление | Нет* | Нет |
| DELETE | Удаление | Да | Нет |

### Пример REST API

```python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel
from typing import Optional, List
import uuid

app = FastAPI()

# Модели
class UserCreate(BaseModel):
    name: str
    email: str

class User(UserCreate):
    id: str
    created_at: str

class UserUpdate(BaseModel):
    name: Optional[str] = None
    email: Optional[str] = None

# In-memory storage (для примера)
users_db = {}

# CRUD операции
@app.post("/users", response_model=User, status_code=status.HTTP_201_CREATED)
def create_user(user: UserCreate):
    user_id = str(uuid.uuid4())
    user_data = User(
        id=user_id,
        name=user.name,
        email=user.email,
        created_at=datetime.utcnow().isoformat()
    )
    users_db[user_id] = user_data
    return user_data

@app.get("/users", response_model=List[User])
def list_users(skip: int = 0, limit: int = 10):
    return list(users_db.values())[skip:skip + limit]

@app.get("/users/{user_id}", response_model=User)
def get_user(user_id: str):
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail="User not found")
    return users_db[user_id]

@app.put("/users/{user_id}", response_model=User)
def update_user(user_id: str, user: UserCreate):
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail="User not found")

    user_data = User(
        id=user_id,
        name=user.name,
        email=user.email,
        created_at=users_db[user_id].created_at
    )
    users_db[user_id] = user_data
    return user_data

@app.patch("/users/{user_id}", response_model=User)
def partial_update_user(user_id: str, user: UserUpdate):
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail="User not found")

    stored_user = users_db[user_id]
    update_data = user.dict(exclude_unset=True)
    updated_user = stored_user.copy(update=update_data)
    users_db[user_id] = updated_user
    return updated_user

@app.delete("/users/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_user(user_id: str):
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail="User not found")
    del users_db[user_id]
```

### HTTP Status Codes

```python
# 2xx — Success
200 OK              # Успешный GET/PUT/PATCH
201 Created         # Успешный POST
204 No Content      # Успешный DELETE

# 3xx — Redirection
301 Moved Permanently
304 Not Modified    # Кэшированный ответ актуален

# 4xx — Client Error
400 Bad Request     # Невалидные данные
401 Unauthorized    # Требуется аутентификация
403 Forbidden       # Нет прав доступа
404 Not Found       # Ресурс не найден
409 Conflict        # Конфликт (дубликат)
422 Unprocessable   # Валидация не пройдена
429 Too Many Requests # Rate limit

# 5xx — Server Error
500 Internal Server Error
502 Bad Gateway     # Ошибка upstream сервиса
503 Service Unavailable
504 Gateway Timeout
```

### Версионирование API

```python
# 1. URL versioning (наиболее распространён)
@app.get("/v1/users")
@app.get("/v2/users")  # Новая версия

# 2. Header versioning
@app.get("/users")
def get_users(request: Request):
    version = request.headers.get("API-Version", "1")
    if version == "2":
        return get_users_v2()
    return get_users_v1()

# 3. Query parameter
@app.get("/users")
def get_users(version: str = "1"):
    pass
```

### Pagination

```python
from fastapi import Query

# Offset-based pagination
@app.get("/users")
def list_users(
    offset: int = Query(0, ge=0),
    limit: int = Query(20, ge=1, le=100)
):
    users = db.query(User).offset(offset).limit(limit).all()
    total = db.query(User).count()

    return {
        "data": users,
        "pagination": {
            "offset": offset,
            "limit": limit,
            "total": total,
            "has_more": offset + limit < total
        }
    }

# Cursor-based pagination (лучше для больших данных)
@app.get("/users")
def list_users(
    cursor: Optional[str] = None,
    limit: int = Query(20, ge=1, le=100)
):
    query = db.query(User).order_by(User.id)

    if cursor:
        query = query.filter(User.id > cursor)

    users = query.limit(limit + 1).all()
    has_more = len(users) > limit
    users = users[:limit]

    next_cursor = users[-1].id if has_more and users else None

    return {
        "data": users,
        "next_cursor": next_cursor
    }
```

---

## gRPC

**gRPC** — высокопроизводительный RPC фреймворк от Google, использующий HTTP/2 и Protocol Buffers.

### Преимущества gRPC

1. **Производительность** — бинарный протокол, HTTP/2
2. **Типизация** — строгие контракты через .proto
3. **Streaming** — клиентский, серверный, двунаправленный
4. **Генерация кода** — автоматическая для многих языков
5. **Встроенные фичи** — deadlines, cancellation, interceptors

### Protocol Buffers

```protobuf
// user.proto
syntax = "proto3";

package user;

service UserService {
    // Unary RPC
    rpc GetUser(GetUserRequest) returns (User);
    rpc CreateUser(CreateUserRequest) returns (User);

    // Server streaming
    rpc ListUsers(ListUsersRequest) returns (stream User);

    // Client streaming
    rpc CreateUsers(stream CreateUserRequest) returns (CreateUsersResponse);

    // Bidirectional streaming
    rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message User {
    string id = 1;
    string name = 2;
    string email = 3;
    int64 created_at = 4;
}

message GetUserRequest {
    string id = 1;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
}

message ListUsersRequest {
    int32 page_size = 1;
    string page_token = 2;
}

message CreateUsersResponse {
    int32 created_count = 1;
}

message ChatMessage {
    string user_id = 1;
    string content = 2;
    int64 timestamp = 3;
}
```

### gRPC Server (Python)

```python
# server.py
import grpc
from concurrent import futures
import user_pb2
import user_pb2_grpc
from datetime import datetime
import uuid

class UserServicer(user_pb2_grpc.UserServiceServicer):
    def __init__(self):
        self.users = {}

    def GetUser(self, request, context):
        user_id = request.id

        if user_id not in self.users:
            context.set_code(grpc.StatusCode.NOT_FOUND)
            context.set_details(f"User {user_id} not found")
            return user_pb2.User()

        return self.users[user_id]

    def CreateUser(self, request, context):
        user_id = str(uuid.uuid4())
        user = user_pb2.User(
            id=user_id,
            name=request.name,
            email=request.email,
            created_at=int(datetime.utcnow().timestamp())
        )
        self.users[user_id] = user
        return user

    def ListUsers(self, request, context):
        # Server streaming — возвращаем пользователей по одному
        for user in self.users.values():
            yield user

    def CreateUsers(self, request_iterator, context):
        # Client streaming — получаем пользователей по одному
        count = 0
        for request in request_iterator:
            user_id = str(uuid.uuid4())
            user = user_pb2.User(
                id=user_id,
                name=request.name,
                email=request.email,
                created_at=int(datetime.utcnow().timestamp())
            )
            self.users[user_id] = user
            count += 1

        return user_pb2.CreateUsersResponse(created_count=count)

    def Chat(self, request_iterator, context):
        # Bidirectional streaming
        for message in request_iterator:
            # Echo back with modification
            response = user_pb2.ChatMessage(
                user_id="server",
                content=f"Echo: {message.content}",
                timestamp=int(datetime.utcnow().timestamp())
            )
            yield response

def serve():
    server = grpc.server(
        futures.ThreadPoolExecutor(max_workers=10),
        options=[
            ('grpc.max_receive_message_length', 50 * 1024 * 1024),  # 50MB
        ]
    )
    user_pb2_grpc.add_UserServiceServicer_to_server(UserServicer(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    print("Server started on port 50051")
    server.wait_for_termination()

if __name__ == '__main__':
    serve()
```

### gRPC Client

```python
# client.py
import grpc
import user_pb2
import user_pb2_grpc

def run():
    # Создание канала
    channel = grpc.insecure_channel('localhost:50051')
    stub = user_pb2_grpc.UserServiceStub(channel)

    # Unary call
    user = stub.CreateUser(user_pb2.CreateUserRequest(
        name="John Doe",
        email="john@example.com"
    ))
    print(f"Created user: {user.id}")

    # Get user
    fetched_user = stub.GetUser(user_pb2.GetUserRequest(id=user.id))
    print(f"Fetched user: {fetched_user.name}")

    # Server streaming
    print("All users:")
    for user in stub.ListUsers(user_pb2.ListUsersRequest()):
        print(f"  - {user.name}")

    # Client streaming
    def generate_users():
        for i in range(5):
            yield user_pb2.CreateUserRequest(
                name=f"User {i}",
                email=f"user{i}@example.com"
            )

    response = stub.CreateUsers(generate_users())
    print(f"Created {response.created_count} users")

# С обработкой ошибок и таймаутами
def run_with_error_handling():
    channel = grpc.insecure_channel('localhost:50051')
    stub = user_pb2_grpc.UserServiceStub(channel)

    try:
        user = stub.GetUser(
            user_pb2.GetUserRequest(id="non-existent"),
            timeout=5.0  # 5 seconds timeout
        )
    except grpc.RpcError as e:
        if e.code() == grpc.StatusCode.NOT_FOUND:
            print(f"User not found: {e.details()}")
        elif e.code() == grpc.StatusCode.DEADLINE_EXCEEDED:
            print("Request timed out")
        else:
            print(f"RPC failed: {e.code()} - {e.details()}")
```

### gRPC Interceptors

```python
# Логирование всех вызовов
class LoggingInterceptor(grpc.UnaryUnaryClientInterceptor):
    def intercept_unary_unary(self, continuation, client_call_details, request):
        method = client_call_details.method
        print(f"Calling {method}")

        start_time = time.time()
        response = continuation(client_call_details, request)
        duration = time.time() - start_time

        print(f"{method} completed in {duration:.3f}s")
        return response

# Применение interceptor
channel = grpc.insecure_channel('localhost:50051')
channel = grpc.intercept_channel(channel, LoggingInterceptor())
stub = user_pb2_grpc.UserServiceStub(channel)
```

---

## GraphQL

**GraphQL** — язык запросов для API, позволяющий клиенту запрашивать именно те данные, которые нужны.

### Преимущества GraphQL

1. **Точные запросы** — клиент получает только нужные поля
2. **Один endpoint** — вместо множества REST endpoints
3. **Сильная типизация** — схема как контракт
4. **Интроспекция** — API самодокументируется
5. **Вложенные запросы** — получение связанных данных за один запрос

### Схема GraphQL

```graphql
# schema.graphql
type Query {
    user(id: ID!): User
    users(first: Int, after: String): UserConnection!
    searchUsers(query: String!): [User!]!
}

type Mutation {
    createUser(input: CreateUserInput!): User!
    updateUser(id: ID!, input: UpdateUserInput!): User!
    deleteUser(id: ID!): Boolean!
}

type Subscription {
    userCreated: User!
    userUpdated(id: ID!): User!
}

type User {
    id: ID!
    name: String!
    email: String!
    posts: [Post!]!
    followers: [User!]!
    createdAt: DateTime!
}

type Post {
    id: ID!
    title: String!
    content: String!
    author: User!
    comments: [Comment!]!
}

type Comment {
    id: ID!
    text: String!
    author: User!
}

type UserConnection {
    edges: [UserEdge!]!
    pageInfo: PageInfo!
}

type UserEdge {
    node: User!
    cursor: String!
}

type PageInfo {
    hasNextPage: Boolean!
    endCursor: String
}

input CreateUserInput {
    name: String!
    email: String!
}

input UpdateUserInput {
    name: String
    email: String
}

scalar DateTime
```

### GraphQL Server (Python с Strawberry)

```python
# server.py
import strawberry
from strawberry.fastapi import GraphQLRouter
from fastapi import FastAPI
from typing import List, Optional
from datetime import datetime
import uuid

# Типы
@strawberry.type
class User:
    id: strawberry.ID
    name: str
    email: str
    created_at: datetime

    @strawberry.field
    def posts(self, info) -> List["Post"]:
        # Resolver для постов пользователя
        return [p for p in posts_db.values() if p.author_id == self.id]

    @strawberry.field
    def followers(self, info) -> List["User"]:
        # Resolver для подписчиков
        follower_ids = followers_db.get(self.id, [])
        return [users_db[fid] for fid in follower_ids if fid in users_db]

@strawberry.type
class Post:
    id: strawberry.ID
    title: str
    content: str
    author_id: strawberry.Private[str]  # Не экспонируем напрямую

    @strawberry.field
    def author(self, info) -> User:
        return users_db[self.author_id]

# Input типы
@strawberry.input
class CreateUserInput:
    name: str
    email: str

@strawberry.input
class UpdateUserInput:
    name: Optional[str] = None
    email: Optional[str] = None

# In-memory storage
users_db = {}
posts_db = {}
followers_db = {}

# Query
@strawberry.type
class Query:
    @strawberry.field
    def user(self, id: strawberry.ID) -> Optional[User]:
        return users_db.get(id)

    @strawberry.field
    def users(self, first: int = 10, after: Optional[str] = None) -> List[User]:
        all_users = list(users_db.values())

        if after:
            # Находим позицию курсора
            start_idx = next(
                (i for i, u in enumerate(all_users) if u.id == after),
                0
            ) + 1
            all_users = all_users[start_idx:]

        return all_users[:first]

    @strawberry.field
    def search_users(self, query: str) -> List[User]:
        query_lower = query.lower()
        return [
            u for u in users_db.values()
            if query_lower in u.name.lower() or query_lower in u.email.lower()
        ]

# Mutation
@strawberry.type
class Mutation:
    @strawberry.mutation
    def create_user(self, input: CreateUserInput) -> User:
        user_id = str(uuid.uuid4())
        user = User(
            id=strawberry.ID(user_id),
            name=input.name,
            email=input.email,
            created_at=datetime.utcnow()
        )
        users_db[user_id] = user
        return user

    @strawberry.mutation
    def update_user(self, id: strawberry.ID, input: UpdateUserInput) -> Optional[User]:
        if id not in users_db:
            return None

        user = users_db[id]
        if input.name is not None:
            user.name = input.name
        if input.email is not None:
            user.email = input.email

        return user

    @strawberry.mutation
    def delete_user(self, id: strawberry.ID) -> bool:
        if id in users_db:
            del users_db[id]
            return True
        return False

# Schema и приложение
schema = strawberry.Schema(query=Query, mutation=Mutation)
graphql_app = GraphQLRouter(schema)

app = FastAPI()
app.include_router(graphql_app, prefix="/graphql")
```

### GraphQL Queries

```graphql
# Простой запрос
query {
    user(id: "123") {
        name
        email
    }
}

# Вложенный запрос — получаем пользователя с постами и комментариями
query GetUserWithPosts {
    user(id: "123") {
        name
        email
        posts {
            title
            content
            comments {
                text
                author {
                    name
                }
            }
        }
    }
}

# Запрос с переменными
query GetUser($userId: ID!) {
    user(id: $userId) {
        name
        email
        followers {
            name
        }
    }
}

# Mutation
mutation CreateNewUser {
    createUser(input: { name: "John", email: "john@example.com" }) {
        id
        name
        createdAt
    }
}

# Несколько операций в одном запросе
query Dashboard {
    currentUser: user(id: "123") {
        name
        email
    }
    recentUsers: users(first: 5) {
        name
    }
}
```

### N+1 Problem и DataLoader

```python
# Проблема N+1: для каждого пользователя отдельный запрос к БД
# Решение: DataLoader для батчинга

from strawberry.dataloader import DataLoader

async def load_users(user_ids: List[str]) -> List[User]:
    # Один запрос для всех пользователей
    users = await db.fetch_users_by_ids(user_ids)
    # Важно: порядок должен соответствовать порядку user_ids
    user_map = {u.id: u for u in users}
    return [user_map.get(uid) for uid in user_ids]

# В контексте запроса
@strawberry.type
class Query:
    @strawberry.field
    async def user(self, id: strawberry.ID, info) -> Optional[User]:
        loader = info.context["user_loader"]
        return await loader.load(id)

# Создание лоадера для каждого запроса
async def get_context():
    return {
        "user_loader": DataLoader(load_fn=load_users)
    }

schema = strawberry.Schema(query=Query)
graphql_app = GraphQLRouter(schema, context_getter=get_context)
```

---

## WebSockets

**WebSocket** — протокол для двунаправленной связи в реальном времени поверх TCP.

### Характеристики

- **Full-duplex** — одновременная отправка и получение
- **Persistent connection** — соединение остаётся открытым
- **Low latency** — нет overhead на установку соединения
- **Real-time** — идеально для чатов, игр, уведомлений

### WebSocket Server

```python
# FastAPI WebSocket
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import List, Dict
import json

app = FastAPI()

class ConnectionManager:
    def __init__(self):
        # Активные соединения по комнатам
        self.active_connections: Dict[str, List[WebSocket]] = {}

    async def connect(self, websocket: WebSocket, room: str):
        await websocket.accept()
        if room not in self.active_connections:
            self.active_connections[room] = []
        self.active_connections[room].append(websocket)

    def disconnect(self, websocket: WebSocket, room: str):
        if room in self.active_connections:
            self.active_connections[room].remove(websocket)

    async def send_personal_message(self, message: str, websocket: WebSocket):
        await websocket.send_text(message)

    async def broadcast(self, message: str, room: str, exclude: WebSocket = None):
        if room in self.active_connections:
            for connection in self.active_connections[room]:
                if connection != exclude:
                    await connection.send_text(message)

manager = ConnectionManager()

@app.websocket("/ws/{room}/{user_id}")
async def websocket_endpoint(websocket: WebSocket, room: str, user_id: str):
    await manager.connect(websocket, room)

    # Уведомляем других о подключении
    await manager.broadcast(
        json.dumps({"type": "user_joined", "user_id": user_id}),
        room,
        exclude=websocket
    )

    try:
        while True:
            data = await websocket.receive_text()
            message = json.loads(data)

            # Обработка разных типов сообщений
            if message["type"] == "chat":
                await manager.broadcast(
                    json.dumps({
                        "type": "chat",
                        "user_id": user_id,
                        "content": message["content"]
                    }),
                    room
                )
            elif message["type"] == "typing":
                await manager.broadcast(
                    json.dumps({"type": "typing", "user_id": user_id}),
                    room,
                    exclude=websocket
                )

    except WebSocketDisconnect:
        manager.disconnect(websocket, room)
        await manager.broadcast(
            json.dumps({"type": "user_left", "user_id": user_id}),
            room
        )
```

### WebSocket Client

```python
import asyncio
import websockets
import json

async def chat_client(room: str, user_id: str):
    uri = f"ws://localhost:8000/ws/{room}/{user_id}"

    async with websockets.connect(uri) as websocket:
        # Задача для получения сообщений
        async def receive_messages():
            async for message in websocket:
                data = json.loads(message)
                if data["type"] == "chat":
                    print(f"{data['user_id']}: {data['content']}")
                elif data["type"] == "user_joined":
                    print(f">>> {data['user_id']} joined")
                elif data["type"] == "user_left":
                    print(f">>> {data['user_id']} left")

        # Задача для отправки сообщений
        async def send_messages():
            while True:
                content = await asyncio.get_event_loop().run_in_executor(
                    None, input
                )
                await websocket.send(json.dumps({
                    "type": "chat",
                    "content": content
                }))

        # Запускаем обе задачи параллельно
        await asyncio.gather(
            receive_messages(),
            send_messages()
        )

# Запуск
asyncio.run(chat_client("general", "user123"))
```

### WebSocket с аутентификацией

```python
from fastapi import WebSocket, Query, HTTPException
import jwt

async def get_current_user(websocket: WebSocket):
    token = websocket.query_params.get("token")
    if not token:
        await websocket.close(code=4001)
        raise HTTPException(status_code=401, detail="Missing token")

    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        return payload["user_id"]
    except jwt.InvalidTokenError:
        await websocket.close(code=4001)
        raise HTTPException(status_code=401, detail="Invalid token")

@app.websocket("/ws/secure")
async def secure_websocket(websocket: WebSocket):
    user_id = await get_current_user(websocket)
    await websocket.accept()

    # Теперь работаем с аутентифицированным пользователем
    await websocket.send_text(f"Welcome, {user_id}!")
```

---

## Паттерны коммуникации

### 1. Request-Response

Классический синхронный паттерн.

```python
# Клиент ждёт ответа
response = requests.post("/api/orders", json=order_data)
order = response.json()
```

### 2. Fire and Forget

Отправка без ожидания ответа.

```python
# Отправляем событие и не ждём
producer.send('notifications', {
    'type': 'email',
    'to': 'user@example.com',
    'template': 'welcome'
})
# Продолжаем работу сразу
```

### 3. Publish-Subscribe

Один publisher, много subscribers.

```python
# Publisher
async def publish_order_created(order):
    await redis.publish('orders', json.dumps({
        'event': 'order.created',
        'data': order
    }))

# Subscribers
async def inventory_subscriber():
    pubsub = redis.pubsub()
    await pubsub.subscribe('orders')

    async for message in pubsub.listen():
        if message['type'] == 'message':
            event = json.loads(message['data'])
            if event['event'] == 'order.created':
                await reserve_inventory(event['data'])

async def notification_subscriber():
    pubsub = redis.pubsub()
    await pubsub.subscribe('orders')

    async for message in pubsub.listen():
        if message['type'] == 'message':
            event = json.loads(message['data'])
            if event['event'] == 'order.created':
                await send_order_confirmation(event['data'])
```

### 4. Request-Reply (Async)

Асинхронный запрос-ответ через очереди.

```python
import uuid

class AsyncRPC:
    def __init__(self):
        self.pending_requests = {}

    async def call(self, method, params, timeout=30):
        correlation_id = str(uuid.uuid4())

        # Создаём future для ожидания ответа
        future = asyncio.Future()
        self.pending_requests[correlation_id] = future

        # Отправляем запрос
        await self.producer.send('rpc_requests', {
            'correlation_id': correlation_id,
            'reply_to': 'rpc_responses',
            'method': method,
            'params': params
        })

        try:
            # Ожидаем ответ
            result = await asyncio.wait_for(future, timeout=timeout)
            return result
        finally:
            del self.pending_requests[correlation_id]

    async def handle_response(self, message):
        correlation_id = message['correlation_id']
        if correlation_id in self.pending_requests:
            future = self.pending_requests[correlation_id]
            future.set_result(message['result'])
```

### 5. Saga Pattern (Distributed Transactions)

Координация транзакций через последовательность событий.

```python
# Choreography-based Saga
class OrderSaga:
    async def handle_order_created(self, event):
        order_id = event['order_id']

        # Шаг 1: Резервирование инвентаря
        await self.publish('inventory.reserve', {
            'order_id': order_id,
            'items': event['items']
        })

    async def handle_inventory_reserved(self, event):
        order_id = event['order_id']

        # Шаг 2: Списание средств
        await self.publish('payment.charge', {
            'order_id': order_id,
            'amount': event['total']
        })

    async def handle_payment_charged(self, event):
        order_id = event['order_id']

        # Шаг 3: Подтверждение заказа
        await self.publish('order.confirm', {'order_id': order_id})

    # Компенсирующие действия при ошибках
    async def handle_payment_failed(self, event):
        order_id = event['order_id']

        # Откат: освобождаем инвентарь
        await self.publish('inventory.release', {'order_id': order_id})

        # Откат: отменяем заказ
        await self.publish('order.cancel', {
            'order_id': order_id,
            'reason': 'Payment failed'
        })
```

---

## Service Discovery и Load Balancing

### Service Discovery

```python
# Consul для service discovery
import consul

class ServiceRegistry:
    def __init__(self, consul_host='localhost'):
        self.consul = consul.Consul(host=consul_host)

    def register(self, service_name, service_id, port, health_check_url):
        self.consul.agent.service.register(
            name=service_name,
            service_id=service_id,
            port=port,
            check=consul.Check.http(
                health_check_url,
                interval='10s',
                timeout='5s'
            )
        )

    def deregister(self, service_id):
        self.consul.agent.service.deregister(service_id)

    def get_service(self, service_name):
        _, services = self.consul.health.service(service_name, passing=True)
        if not services:
            raise ServiceNotAvailable(f"No healthy instances of {service_name}")

        # Round-robin или random selection
        service = random.choice(services)
        address = service['Service']['Address']
        port = service['Service']['Port']
        return f"http://{address}:{port}"
```

### Client-side Load Balancing

```python
import random
from collections import defaultdict

class LoadBalancer:
    def __init__(self, strategy='round_robin'):
        self.strategy = strategy
        self.instances = {}
        self.counters = defaultdict(int)

    def update_instances(self, service_name, instances):
        self.instances[service_name] = instances

    def get_instance(self, service_name):
        instances = self.instances.get(service_name, [])
        if not instances:
            raise ServiceNotAvailable(service_name)

        if self.strategy == 'round_robin':
            idx = self.counters[service_name] % len(instances)
            self.counters[service_name] += 1
            return instances[idx]

        elif self.strategy == 'random':
            return random.choice(instances)

        elif self.strategy == 'least_connections':
            # Выбираем инстанс с наименьшим количеством соединений
            return min(instances, key=lambda i: i.active_connections)

# Использование с retry
class ResilientClient:
    def __init__(self, load_balancer):
        self.lb = load_balancer

    async def call(self, service_name, path, max_retries=3):
        for attempt in range(max_retries):
            instance = self.lb.get_instance(service_name)
            try:
                response = await httpx.get(f"{instance}{path}")
                return response.json()
            except Exception as e:
                if attempt == max_retries - 1:
                    raise
                # Попробуем другой инстанс
                continue
```

---

## Resilience Patterns

### Circuit Breaker

```python
from enum import Enum
import time

class CircuitState(Enum):
    CLOSED = "closed"      # Нормальная работа
    OPEN = "open"          # Блокируем вызовы
    HALF_OPEN = "half_open"  # Пробуем восстановиться

class CircuitBreaker:
    def __init__(
        self,
        failure_threshold=5,
        recovery_timeout=30,
        success_threshold=3
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.success_threshold = success_threshold

        self.state = CircuitState.CLOSED
        self.failures = 0
        self.successes = 0
        self.last_failure_time = None

    def can_execute(self):
        if self.state == CircuitState.CLOSED:
            return True

        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                return True
            return False

        if self.state == CircuitState.HALF_OPEN:
            return True

        return False

    def record_success(self):
        if self.state == CircuitState.HALF_OPEN:
            self.successes += 1
            if self.successes >= self.success_threshold:
                self.state = CircuitState.CLOSED
                self.failures = 0
                self.successes = 0

    def record_failure(self):
        self.failures += 1
        self.last_failure_time = time.time()

        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.OPEN
            self.successes = 0

        elif self.failures >= self.failure_threshold:
            self.state = CircuitState.OPEN

# Использование
circuit_breaker = CircuitBreaker()

async def call_service(url):
    if not circuit_breaker.can_execute():
        raise CircuitOpenError("Circuit breaker is open")

    try:
        response = await httpx.get(url, timeout=5.0)
        circuit_breaker.record_success()
        return response
    except Exception as e:
        circuit_breaker.record_failure()
        raise
```

### Retry с Exponential Backoff

```python
import asyncio
import random

async def retry_with_backoff(
    func,
    max_retries=3,
    base_delay=1,
    max_delay=60,
    exponential_base=2,
    jitter=True
):
    for attempt in range(max_retries):
        try:
            return await func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise

            delay = min(base_delay * (exponential_base ** attempt), max_delay)

            if jitter:
                delay = delay * (0.5 + random.random())

            print(f"Attempt {attempt + 1} failed, retrying in {delay:.2f}s")
            await asyncio.sleep(delay)

# Использование
result = await retry_with_backoff(
    lambda: fetch_data_from_service(),
    max_retries=5,
    base_delay=1
)
```

### Bulkhead Pattern

```python
import asyncio

class Bulkhead:
    """Изолирует ресурсы для разных типов операций"""

    def __init__(self, max_concurrent: int):
        self.semaphore = asyncio.Semaphore(max_concurrent)

    async def execute(self, func):
        async with self.semaphore:
            return await func()

# Разные bulkheads для разных сервисов
user_service_bulkhead = Bulkhead(max_concurrent=10)
payment_service_bulkhead = Bulkhead(max_concurrent=5)
notification_bulkhead = Bulkhead(max_concurrent=20)

async def get_user(user_id):
    return await user_service_bulkhead.execute(
        lambda: fetch_user(user_id)
    )

async def process_payment(payment_data):
    return await payment_service_bulkhead.execute(
        lambda: call_payment_service(payment_data)
    )
```

---

## Best Practices

### 1. Таймауты везде

```python
# ❌ Неправильно
response = requests.get(url)

# ✅ Правильно
response = requests.get(url, timeout=(3.05, 27))  # (connect, read)
```

### 2. Идемпотентность

```python
# ❌ Неправильно: POST без идемпотентности
@app.post("/orders")
def create_order(order: OrderCreate):
    return db.create_order(order)  # Повторный запрос создаст дубликат

# ✅ Правильно: идемпотентный ключ
@app.post("/orders")
def create_order(
    order: OrderCreate,
    idempotency_key: str = Header(...)
):
    existing = db.get_order_by_idempotency_key(idempotency_key)
    if existing:
        return existing

    return db.create_order(order, idempotency_key=idempotency_key)
```

### 3. Graceful Degradation

```python
async def get_product_with_recommendations(product_id):
    product = await get_product(product_id)  # Critical

    try:
        # Non-critical: если не работает — показываем без рекомендаций
        recommendations = await asyncio.wait_for(
            get_recommendations(product_id),
            timeout=2.0
        )
    except (asyncio.TimeoutError, ServiceError):
        recommendations = []  # Fallback

    return {
        "product": product,
        "recommendations": recommendations
    }
```

### 4. Correlation ID для трассировки

```python
import uuid
from contextvars import ContextVar

correlation_id: ContextVar[str] = ContextVar('correlation_id')

@app.middleware("http")
async def add_correlation_id(request: Request, call_next):
    # Получаем или генерируем correlation ID
    corr_id = request.headers.get('X-Correlation-ID', str(uuid.uuid4()))
    correlation_id.set(corr_id)

    response = await call_next(request)
    response.headers['X-Correlation-ID'] = corr_id

    return response

# Передача в исходящие запросы
async def call_external_service(url):
    headers = {'X-Correlation-ID': correlation_id.get()}
    return await httpx.get(url, headers=headers)
```

---

## Типичные ошибки

### 1. Синхронные вызовы в цепочке

```python
# ❌ Cascade latency
def get_order_details(order_id):
    order = requests.get(f"{ORDER_SERVICE}/orders/{order_id}").json()
    user = requests.get(f"{USER_SERVICE}/users/{order['user_id']}").json()
    products = requests.get(f"{PRODUCT_SERVICE}/products", params={...}).json()
    # Total time = order_time + user_time + products_time

# ✅ Параллельные запросы
async def get_order_details(order_id):
    order = await fetch_order(order_id)

    user_task = asyncio.create_task(fetch_user(order['user_id']))
    products_task = asyncio.create_task(fetch_products(order['product_ids']))

    user, products = await asyncio.gather(user_task, products_task)
    # Total time = order_time + max(user_time, products_time)
```

### 2. Отсутствие обработки ошибок

```python
# ❌ Crash при любой ошибке
response = requests.get(url)
data = response.json()

# ✅ Правильная обработка
try:
    response = requests.get(url, timeout=5)
    response.raise_for_status()
    data = response.json()
except requests.Timeout:
    logger.error("Service timeout")
    raise ServiceUnavailable()
except requests.HTTPError as e:
    logger.error(f"HTTP error: {e}")
    raise
except requests.RequestException as e:
    logger.error(f"Request failed: {e}")
    raise ServiceUnavailable()
```

### 3. Chatty API

```python
# ❌ Много мелких запросов
for product_id in product_ids:
    product = await fetch_product(product_id)  # N запросов

# ✅ Batch запрос
products = await fetch_products(product_ids)  # 1 запрос
```

---

## Заключение

Выбор паттерна коммуникации зависит от требований системы:

| Требование | Рекомендация |
|-----------|--------------|
| Real-time updates | WebSocket, Server-Sent Events |
| High throughput | gRPC, Kafka |
| Flexible queries | GraphQL |
| Simple CRUD | REST |
| Fire and forget | Message queues |
| Distributed transactions | Saga pattern |

Ключевые принципы:
1. **Всегда используйте таймауты**
2. **Делайте операции идемпотентными**
3. **Планируйте обработку ошибок**
4. **Используйте circuit breaker**
5. **Трассируйте запросы через correlation ID**
