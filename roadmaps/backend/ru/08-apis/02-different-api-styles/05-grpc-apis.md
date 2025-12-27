# gRPC APIs

## Что такое gRPC?

**gRPC (gRPC Remote Procedure Calls)** — это высокопроизводительный фреймворк для удалённого вызова процедур (RPC), разработанный Google. gRPC использует HTTP/2 для транспорта и Protocol Buffers (protobuf) для сериализации данных.

## Ключевые характеристики

- **HTTP/2** — мультиплексирование, сжатие заголовков, server push
- **Protocol Buffers** — бинарная сериализация, строгая типизация
- **Streaming** — unary, server streaming, client streaming, bidirectional
- **Кросс-платформенность** — поддержка 10+ языков программирования
- **Генерация кода** — автоматическое создание клиентов и серверов

## Архитектура gRPC

```
┌─────────────┐                           ┌─────────────┐
│   Client    │                           │   Server    │
│             │                           │             │
│  ┌───────┐  │       HTTP/2 Stream       │  ┌───────┐  │
│  │ Stub  │──┼───────────────────────────┼──│Service│  │
│  └───────┘  │                           │  └───────┘  │
│      ↑      │                           │      ↑      │
│  ┌───────┐  │                           │  ┌───────┐  │
│  │Proto- │  │                           │  │Proto- │  │
│  │ buf   │  │                           │  │ buf   │  │
│  └───────┘  │                           │  └───────┘  │
└─────────────┘                           └─────────────┘
       ↑                                         ↑
       └─────────────────────────────────────────┘
                        ↑
               ┌────────────────┐
               │  .proto file   │
               │  (контракт)    │
               └────────────────┘
```

## Protocol Buffers (Protobuf)

### Определение сообщений

```protobuf
// user.proto
syntax = "proto3";

package user;

option go_package = "github.com/example/user";

// Импорт стандартных типов
import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

// Enum
enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_INACTIVE = 2;
  USER_STATUS_BANNED = 3;
}

// Message (сообщение)
message User {
  string id = 1;
  string name = 2;
  string email = 3;
  UserStatus status = 4;
  google.protobuf.Timestamp created_at = 5;
  repeated string roles = 6;  // Список
  map<string, string> metadata = 7;  // Словарь
  optional string phone = 8;  // Опциональное поле
}

message GetUserRequest {
  string id = 1;
}

message GetUserResponse {
  User user = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
  UserStatus status_filter = 3;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total_count = 2;
  bool has_more = 3;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  string password = 3;
}

message UpdateUserRequest {
  string id = 1;
  optional string name = 2;
  optional string email = 3;
}

message DeleteUserRequest {
  string id = 1;
}

// Service definition
service UserService {
  // Unary RPC
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc UpdateUser(UpdateUserRequest) returns (User);
  rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);

  // Server streaming
  rpc ListUsers(ListUsersRequest) returns (stream User);

  // Client streaming
  rpc CreateUsers(stream CreateUserRequest) returns (ListUsersResponse);

  // Bidirectional streaming
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message ChatMessage {
  string user_id = 1;
  string content = 2;
  google.protobuf.Timestamp timestamp = 3;
}
```

### Типы данных в Protobuf

| Protobuf Type | Python Type | Описание |
|---------------|-------------|----------|
| `double` | `float` | 64-bit |
| `float` | `float` | 32-bit |
| `int32` | `int` | 32-bit signed |
| `int64` | `int` | 64-bit signed |
| `uint32` | `int` | 32-bit unsigned |
| `uint64` | `int` | 64-bit unsigned |
| `bool` | `bool` | Boolean |
| `string` | `str` | UTF-8 строка |
| `bytes` | `bytes` | Бинарные данные |
| `repeated T` | `list[T]` | Список элементов |
| `map<K, V>` | `dict[K, V]` | Словарь |

### Генерация кода

```bash
# Установка
pip install grpcio grpcio-tools

# Генерация Python кода
python -m grpc_tools.protoc \
  -I./protos \
  --python_out=./generated \
  --grpc_python_out=./generated \
  ./protos/user.proto

# Результат:
# generated/user_pb2.py - классы сообщений
# generated/user_pb2_grpc.py - stub и servicer
```

## Типы RPC

### 1. Unary RPC (один запрос — один ответ)

```protobuf
rpc GetUser(GetUserRequest) returns (GetUserResponse);
```

```python
# Сервер
class UserServicer(user_pb2_grpc.UserServiceServicer):
    def GetUser(self, request, context):
        user = get_user_from_db(request.id)
        if not user:
            context.abort(grpc.StatusCode.NOT_FOUND, "User not found")
        return user_pb2.GetUserResponse(user=user)

# Клиент
def get_user(stub, user_id):
    request = user_pb2.GetUserRequest(id=user_id)
    response = stub.GetUser(request)
    return response.user
```

### 2. Server Streaming (один запрос — поток ответов)

```protobuf
rpc ListUsers(ListUsersRequest) returns (stream User);
```

```python
# Сервер
class UserServicer(user_pb2_grpc.UserServiceServicer):
    def ListUsers(self, request, context):
        users = get_users_from_db(
            page=request.page,
            page_size=request.page_size
        )
        for user in users:
            yield user  # Отправляем по одному

# Клиент
def list_users(stub):
    request = user_pb2.ListUsersRequest(page=1, page_size=10)
    for user in stub.ListUsers(request):
        print(f"Received user: {user.name}")
```

### 3. Client Streaming (поток запросов — один ответ)

```protobuf
rpc CreateUsers(stream CreateUserRequest) returns (ListUsersResponse);
```

```python
# Сервер
class UserServicer(user_pb2_grpc.UserServiceServicer):
    def CreateUsers(self, request_iterator, context):
        created_users = []
        for request in request_iterator:
            user = create_user_in_db(request)
            created_users.append(user)

        return user_pb2.ListUsersResponse(
            users=created_users,
            total_count=len(created_users)
        )

# Клиент
def create_users_batch(stub, users_data):
    def generate_requests():
        for data in users_data:
            yield user_pb2.CreateUserRequest(
                name=data["name"],
                email=data["email"]
            )

    response = stub.CreateUsers(generate_requests())
    return response.users
```

### 4. Bidirectional Streaming (поток запросов — поток ответов)

```protobuf
rpc Chat(stream ChatMessage) returns (stream ChatMessage);
```

```python
# Сервер
class UserServicer(user_pb2_grpc.UserServiceServicer):
    def Chat(self, request_iterator, context):
        for message in request_iterator:
            # Обработка входящего сообщения
            response = process_chat_message(message)
            yield response

# Клиент (асинхронный)
async def chat_client(stub):
    async def send_messages():
        messages = ["Hello", "How are you?", "Bye"]
        for msg in messages:
            yield user_pb2.ChatMessage(
                user_id="user1",
                content=msg
            )
            await asyncio.sleep(1)

    responses = stub.Chat(send_messages())
    async for response in responses:
        print(f"Server: {response.content}")
```

## Полный пример: gRPC сервис на Python

### Структура проекта

```
grpc_example/
├── protos/
│   └── user.proto
├── generated/
│   ├── user_pb2.py
│   └── user_pb2_grpc.py
├── server.py
├── client.py
└── requirements.txt
```

### Сервер

```python
# server.py
import grpc
from concurrent import futures
from datetime import datetime
from google.protobuf.timestamp_pb2 import Timestamp
from google.protobuf import empty_pb2

import user_pb2
import user_pb2_grpc

# Имитация базы данных
users_db = {}
next_id = 1


def datetime_to_timestamp(dt: datetime) -> Timestamp:
    ts = Timestamp()
    ts.FromDatetime(dt)
    return ts


class UserServicer(user_pb2_grpc.UserServiceServicer):

    def GetUser(self, request, context):
        """Unary RPC: получить пользователя по ID"""
        user_id = request.id

        if user_id not in users_db:
            context.abort(
                grpc.StatusCode.NOT_FOUND,
                f"User with id {user_id} not found"
            )

        user = users_db[user_id]
        return user_pb2.GetUserResponse(user=user)

    def CreateUser(self, request, context):
        """Unary RPC: создать пользователя"""
        global next_id

        # Валидация
        if not request.name:
            context.abort(
                grpc.StatusCode.INVALID_ARGUMENT,
                "Name is required"
            )

        if not request.email:
            context.abort(
                grpc.StatusCode.INVALID_ARGUMENT,
                "Email is required"
            )

        # Проверка уникальности email
        for user in users_db.values():
            if user.email == request.email:
                context.abort(
                    grpc.StatusCode.ALREADY_EXISTS,
                    "Email already registered"
                )

        # Создание пользователя
        user_id = str(next_id)
        next_id += 1

        user = user_pb2.User(
            id=user_id,
            name=request.name,
            email=request.email,
            status=user_pb2.USER_STATUS_ACTIVE,
            created_at=datetime_to_timestamp(datetime.now())
        )

        users_db[user_id] = user
        return user

    def UpdateUser(self, request, context):
        """Unary RPC: обновить пользователя"""
        if request.id not in users_db:
            context.abort(
                grpc.StatusCode.NOT_FOUND,
                f"User with id {request.id} not found"
            )

        user = users_db[request.id]

        # Обновляем только переданные поля
        if request.HasField("name"):
            user.name = request.name
        if request.HasField("email"):
            user.email = request.email

        return user

    def DeleteUser(self, request, context):
        """Unary RPC: удалить пользователя"""
        if request.id not in users_db:
            context.abort(
                grpc.StatusCode.NOT_FOUND,
                f"User with id {request.id} not found"
            )

        del users_db[request.id]
        return empty_pb2.Empty()

    def ListUsers(self, request, context):
        """Server Streaming RPC: список пользователей"""
        users = list(users_db.values())

        # Фильтрация
        if request.status_filter != user_pb2.USER_STATUS_UNSPECIFIED:
            users = [u for u in users if u.status == request.status_filter]

        # Пагинация
        start = (request.page - 1) * request.page_size
        end = start + request.page_size
        paginated = users[start:end]

        # Streaming response
        for user in paginated:
            yield user

    def CreateUsers(self, request_iterator, context):
        """Client Streaming RPC: создать несколько пользователей"""
        global next_id
        created_users = []

        for request in request_iterator:
            user_id = str(next_id)
            next_id += 1

            user = user_pb2.User(
                id=user_id,
                name=request.name,
                email=request.email,
                status=user_pb2.USER_STATUS_ACTIVE,
                created_at=datetime_to_timestamp(datetime.now())
            )

            users_db[user_id] = user
            created_users.append(user)

        return user_pb2.ListUsersResponse(
            users=created_users,
            total_count=len(created_users),
            has_more=False
        )


def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    user_pb2_grpc.add_UserServiceServicer_to_server(UserServicer(), server)

    # Незащищённый порт
    server.add_insecure_port('[::]:50051')

    # С TLS
    # with open('server.key', 'rb') as f:
    #     private_key = f.read()
    # with open('server.crt', 'rb') as f:
    #     certificate_chain = f.read()
    # credentials = grpc.ssl_server_credentials([(private_key, certificate_chain)])
    # server.add_secure_port('[::]:50051', credentials)

    server.start()
    print("gRPC server running on port 50051")
    server.wait_for_termination()


if __name__ == '__main__':
    serve()
```

### Клиент

```python
# client.py
import grpc
import user_pb2
import user_pb2_grpc


def run():
    # Создание канала
    channel = grpc.insecure_channel('localhost:50051')

    # С TLS
    # with open('ca.crt', 'rb') as f:
    #     trusted_certs = f.read()
    # credentials = grpc.ssl_channel_credentials(root_certificates=trusted_certs)
    # channel = grpc.secure_channel('localhost:50051', credentials)

    stub = user_pb2_grpc.UserServiceStub(channel)

    # 1. Создание пользователя
    print("Creating user...")
    try:
        user = stub.CreateUser(user_pb2.CreateUserRequest(
            name="John Doe",
            email="john@example.com",
            password="secret123"
        ))
        print(f"Created: {user.id} - {user.name}")
    except grpc.RpcError as e:
        print(f"Error: {e.code()} - {e.details()}")

    # 2. Получение пользователя
    print("\nGetting user...")
    try:
        response = stub.GetUser(user_pb2.GetUserRequest(id=user.id))
        print(f"Got: {response.user.name} ({response.user.email})")
    except grpc.RpcError as e:
        print(f"Error: {e.code()} - {e.details()}")

    # 3. Обновление пользователя
    print("\nUpdating user...")
    try:
        updated = stub.UpdateUser(user_pb2.UpdateUserRequest(
            id=user.id,
            name="John Updated"
        ))
        print(f"Updated: {updated.name}")
    except grpc.RpcError as e:
        print(f"Error: {e.code()} - {e.details()}")

    # 4. Streaming: получение списка
    print("\nListing users (streaming)...")
    try:
        for user in stub.ListUsers(user_pb2.ListUsersRequest(
            page=1,
            page_size=10
        )):
            print(f"  - {user.id}: {user.name}")
    except grpc.RpcError as e:
        print(f"Error: {e.code()} - {e.details()}")

    # 5. Удаление
    print("\nDeleting user...")
    try:
        stub.DeleteUser(user_pb2.DeleteUserRequest(id=user.id))
        print("Deleted successfully")
    except grpc.RpcError as e:
        print(f"Error: {e.code()} - {e.details()}")


if __name__ == '__main__':
    run()
```

## Interceptors (Middleware)

### Server Interceptor

```python
import grpc
import logging

class LoggingInterceptor(grpc.ServerInterceptor):
    def intercept_service(self, continuation, handler_call_details):
        method = handler_call_details.method
        logging.info(f"Received request: {method}")

        # Продолжаем обработку
        return continuation(handler_call_details)


class AuthInterceptor(grpc.ServerInterceptor):
    def intercept_service(self, continuation, handler_call_details):
        # Получаем metadata
        metadata = dict(handler_call_details.invocation_metadata)
        token = metadata.get("authorization")

        if not token or not validate_token(token):
            # Возвращаем ошибку аутентификации
            return grpc.unary_unary_rpc_method_handler(
                lambda req, ctx: ctx.abort(
                    grpc.StatusCode.UNAUTHENTICATED,
                    "Invalid or missing token"
                )
            )

        return continuation(handler_call_details)


# Использование
server = grpc.server(
    futures.ThreadPoolExecutor(max_workers=10),
    interceptors=[LoggingInterceptor(), AuthInterceptor()]
)
```

### Client Interceptor

```python
class AuthClientInterceptor(grpc.UnaryUnaryClientInterceptor):
    def __init__(self, token):
        self.token = token

    def intercept_unary_unary(self, continuation, client_call_details, request):
        # Добавляем токен в metadata
        metadata = list(client_call_details.metadata or [])
        metadata.append(("authorization", f"Bearer {self.token}"))

        new_details = client_call_details._replace(metadata=metadata)
        return continuation(new_details, request)


# Использование
channel = grpc.insecure_channel('localhost:50051')
intercept_channel = grpc.intercept_channel(
    channel,
    AuthClientInterceptor("my-jwt-token")
)
stub = user_pb2_grpc.UserServiceStub(intercept_channel)
```

## Error Handling

### Status Codes

| Code | Описание |
|------|----------|
| `OK` | Успех |
| `CANCELLED` | Операция отменена |
| `UNKNOWN` | Неизвестная ошибка |
| `INVALID_ARGUMENT` | Невалидные аргументы |
| `DEADLINE_EXCEEDED` | Превышен таймаут |
| `NOT_FOUND` | Ресурс не найден |
| `ALREADY_EXISTS` | Ресурс уже существует |
| `PERMISSION_DENIED` | Доступ запрещён |
| `RESOURCE_EXHAUSTED` | Ресурсы исчерпаны |
| `UNAUTHENTICATED` | Не аутентифицирован |
| `UNAVAILABLE` | Сервис недоступен |
| `INTERNAL` | Внутренняя ошибка |

### Обработка ошибок

```python
# Сервер
def GetUser(self, request, context):
    try:
        user = get_user_from_db(request.id)
        if not user:
            context.abort(
                grpc.StatusCode.NOT_FOUND,
                f"User {request.id} not found"
            )
        return user
    except DatabaseError as e:
        context.abort(
            grpc.StatusCode.INTERNAL,
            f"Database error: {str(e)}"
        )

# Клиент
try:
    response = stub.GetUser(request)
except grpc.RpcError as e:
    if e.code() == grpc.StatusCode.NOT_FOUND:
        print("User not found")
    elif e.code() == grpc.StatusCode.DEADLINE_EXCEEDED:
        print("Request timed out")
    elif e.code() == grpc.StatusCode.UNAVAILABLE:
        print("Service unavailable, retrying...")
    else:
        print(f"Error: {e.code()} - {e.details()}")
```

## Сравнение с другими стилями API

### gRPC vs REST

| Критерий | gRPC | REST |
|----------|------|------|
| Протокол | HTTP/2 | HTTP/1.1 |
| Формат данных | Protocol Buffers | JSON |
| Производительность | Высокая | Ниже |
| Размер сообщений | Компактный | Больше |
| Browser support | Ограниченная | Полная |
| Streaming | Полный | Ограниченный |
| Типизация | Строгая | Опциональная |
| Tooling | Генерация кода | Ручная разработка |
| Debugging | Сложнее (бинарный) | Проще (читаемый JSON) |

### gRPC vs GraphQL

| Критерий | gRPC | GraphQL |
|----------|------|---------|
| Формат запроса | Фиксированные методы | Гибкие запросы |
| Over-fetching | Возможен | Нет |
| Streaming | Bidirectional | Subscriptions only |
| Производительность | Выше | Ниже |
| Схема | .proto файлы | GraphQL Schema |
| Клиенты | Генерируются | Ручные / Apollo |

## Когда использовать gRPC

**Подходит для:**
- **Микросервисы** — высокая производительность, строгие контракты
- **Polyglot-окружения** — код генерируется для всех языков
- **Real-time приложения** — bidirectional streaming
- **IoT и мобильные** — компактные сообщения, экономия трафика
- **Внутренние API** — когда не нужна browser-совместимость

**Не подходит для:**
- Публичных веб-API (браузеры не поддерживают напрямую)
- Простых CRUD-приложений
- Когда важна читаемость трафика для debugging
- Legacy-систем без HTTP/2 поддержки

## Best Practices

### 1. Версионирование proto-файлов

```protobuf
// Используй package с версией
package user.v1;

// Или service с версией
service UserServiceV1 { ... }
service UserServiceV2 { ... }
```

### 2. Deadline/Timeout

```python
# Всегда устанавливай timeout
try:
    response = stub.GetUser(
        request,
        timeout=5.0  # секунды
    )
except grpc.RpcError as e:
    if e.code() == grpc.StatusCode.DEADLINE_EXCEEDED:
        print("Request timed out")
```

### 3. Retry Logic

```python
from tenacity import retry, stop_after_attempt, retry_if_exception

def is_retryable(e):
    if isinstance(e, grpc.RpcError):
        return e.code() in [
            grpc.StatusCode.UNAVAILABLE,
            grpc.StatusCode.DEADLINE_EXCEEDED,
            grpc.StatusCode.RESOURCE_EXHAUSTED
        ]
    return False

@retry(
    stop=stop_after_attempt(3),
    retry=retry_if_exception(is_retryable)
)
def get_user_with_retry(stub, user_id):
    return stub.GetUser(user_pb2.GetUserRequest(id=user_id))
```

### 4. Health Checking

```protobuf
// health.proto (стандартный)
service Health {
  rpc Check(HealthCheckRequest) returns (HealthCheckResponse);
  rpc Watch(HealthCheckRequest) returns (stream HealthCheckResponse);
}
```

```python
from grpc_health.v1 import health_pb2, health_pb2_grpc
from grpc_health.v1.health import HealthServicer

# Добавляем health check к серверу
health_servicer = HealthServicer()
health_pb2_grpc.add_HealthServicer_to_server(health_servicer, server)
```

### 5. Graceful Shutdown

```python
import signal

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    # ... добавляем сервисы ...
    server.start()

    def handle_sigterm(*_):
        print("Shutting down gracefully...")
        all_rpcs_done = server.stop(30)  # 30 секунд grace period
        all_rpcs_done.wait()

    signal.signal(signal.SIGTERM, handle_sigterm)
    server.wait_for_termination()
```

## Типичные ошибки

### 1. Игнорирование backward compatibility

```protobuf
// Плохо: удаление поля
message User {
  string id = 1;
  // string name = 2;  -- УДАЛЕНО
  string email = 3;
}

// Хорошо: помечаем как reserved
message User {
  string id = 1;
  reserved 2;
  reserved "name";
  string email = 3;
  string display_name = 4;  // Новое поле
}
```

### 2. Отсутствие timeout

```python
# Плохо: без timeout — может висеть вечно
response = stub.GetUser(request)

# Хорошо: с timeout
response = stub.GetUser(request, timeout=5.0)
```

### 3. Блокирующие операции в async коде

```python
# Плохо: блокирует event loop
async def get_user():
    stub = user_pb2_grpc.UserServiceStub(channel)
    return stub.GetUser(request)  # Синхронный вызов!

# Хорошо: используем async stub
async def get_user():
    async with grpc.aio.insecure_channel('localhost:50051') as channel:
        stub = user_pb2_grpc.UserServiceStub(channel)
        return await stub.GetUser(request)
```

### 4. Огромные сообщения

```protobuf
// Плохо: всё в одном сообщении
message GetFileResponse {
  bytes file_content = 1;  // Может быть гигабайты!
}

// Хорошо: streaming
rpc DownloadFile(DownloadRequest) returns (stream FileChunk);

message FileChunk {
  bytes data = 1;  // Маленький чанк
  int64 offset = 2;
}
```

## gRPC-Web для браузеров

```javascript
// Браузеры не поддерживают gRPC напрямую
// Используй grpc-web proxy (Envoy)

import { UserServiceClient } from './generated/user_grpc_web_pb';
import { GetUserRequest } from './generated/user_pb';

const client = new UserServiceClient('http://localhost:8080');

const request = new GetUserRequest();
request.setId('123');

client.getUser(request, {}, (err, response) => {
  if (err) {
    console.error(err);
    return;
  }
  console.log(response.getUser().getName());
});
```

## Инструменты

### Разработка
- **grpcurl** — curl для gRPC
- **BloomRPC** — GUI клиент
- **Postman** — поддержка gRPC

### Тестирование
- **pytest-grpc** — тестирование gRPC-сервисов
- **grpc-testing** — официальная библиотека для тестов

### Мониторинг
- **grpc-prometheus** — метрики для Prometheus
- **OpenTelemetry** — distributed tracing

## Дополнительные ресурсы

- [gRPC Official Documentation](https://grpc.io/docs/)
- [Protocol Buffers Language Guide](https://developers.google.com/protocol-buffers/docs/proto3)
- [gRPC Python Documentation](https://grpc.github.io/grpc/python/)
- [Awesome gRPC](https://github.com/grpc-ecosystem/awesome-grpc)
- [gRPC Best Practices](https://grpc.io/docs/guides/performance/)
