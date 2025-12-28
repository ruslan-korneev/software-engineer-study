# GraphQL через HTTP

[prev: 02-thinking-in-graphs](./02-thinking-in-graphs.md) | [next: 04-file-uploads](./04-file-uploads.md)

---

## Введение

GraphQL — это спецификация языка запросов, а не транспортный протокол. Однако на практике GraphQL чаще всего работает поверх HTTP. Понимание того, как правильно интегрировать GraphQL с HTTP, критически важно для построения надёжного API.

## Базовые принципы

### Единый эндпоинт

В отличие от REST, GraphQL использует один эндпоинт для всех операций:

```
POST /graphql
```

Все запросы (query, mutation, subscription) отправляются на этот адрес.

### HTTP методы

| Операция | Рекомендуемый метод | Альтернатива |
|----------|---------------------|--------------|
| Query | POST | GET (для простых запросов) |
| Mutation | POST | - |
| Subscription | WebSocket | SSE |

## Формат запроса

### POST запрос

```http
POST /graphql HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "query": "query GetUser($id: ID!) { user(id: $id) { name email } }",
  "variables": { "id": "123" },
  "operationName": "GetUser"
}
```

### Структура тела запроса

```typescript
interface GraphQLRequest {
  query: string;              // Обязательно: GraphQL документ
  variables?: Record<string, any>;  // Опционально: переменные
  operationName?: string;     // Опционально: имя операции
  extensions?: Record<string, any>; // Опционально: расширения
}
```

### GET запрос

Для простых запросов можно использовать GET:

```http
GET /graphql?query={user(id:"123"){name}}&variables={"id":"123"}
```

Параметры должны быть URL-encoded:

```javascript
const query = encodeURIComponent(`{ user(id: "123") { name } }`);
const url = `/graphql?query=${query}`;
```

## Формат ответа

### Успешный ответ

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "data": {
    "user": {
      "name": "John Doe",
      "email": "john@example.com"
    }
  }
}
```

### Ответ с ошибками

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "data": {
    "user": null
  },
  "errors": [
    {
      "message": "User not found",
      "locations": [{ "line": 1, "column": 3 }],
      "path": ["user"],
      "extensions": {
        "code": "NOT_FOUND",
        "timestamp": "2024-01-15T10:30:00Z"
      }
    }
  ]
}
```

### Структура ответа

```typescript
interface GraphQLResponse {
  data?: Record<string, any> | null;  // Результат выполнения
  errors?: GraphQLError[];             // Массив ошибок
  extensions?: Record<string, any>;    // Метаданные
}

interface GraphQLError {
  message: string;                    // Описание ошибки
  locations?: { line: number; column: number }[];  // Позиция в запросе
  path?: (string | number)[];         // Путь к полю с ошибкой
  extensions?: Record<string, any>;   // Дополнительная информация
}
```

## HTTP статус-коды

### Рекомендации по статус-кодам

| Ситуация | Статус-код | Описание |
|----------|------------|----------|
| Успех | 200 OK | Запрос выполнен (даже с частичными ошибками) |
| Синтаксическая ошибка | 400 Bad Request | Невалидный GraphQL документ |
| Неавторизован | 401 Unauthorized | Требуется аутентификация |
| Forbidden | 403 Forbidden | Нет прав доступа |
| Method Not Allowed | 405 | Неподдерживаемый HTTP метод |
| Server Error | 500 | Ошибка сервера |

### Важное замечание

GraphQL часто возвращает 200 OK даже при ошибках в данных:

```javascript
// Это вернёт 200 OK, не 404
{
  "data": { "user": null },
  "errors": [{ "message": "User not found" }]
}
```

## Заголовки HTTP

### Обязательные заголовки запроса

```http
Content-Type: application/json
Accept: application/json
```

### Рекомендуемые заголовки

```http
# Аутентификация
Authorization: Bearer <token>

# Кэширование
Cache-Control: no-cache

# Отслеживание запросов
X-Request-ID: uuid-here

# Версионирование (если нужно)
X-API-Version: 2024-01
```

### Заголовки ответа

```http
Content-Type: application/json; charset=utf-8
X-Request-ID: uuid-here
X-Response-Time: 45ms
```

## Реализация сервера

### Node.js с Express

```javascript
import express from 'express';
import { graphqlHTTP } from 'express-graphql';
import { schema } from './schema';

const app = express();

app.use('/graphql', graphqlHTTP({
  schema,
  graphiql: process.env.NODE_ENV === 'development',
  customFormatErrorFn: (error) => ({
    message: error.message,
    locations: error.locations,
    path: error.path,
    extensions: {
      code: error.extensions?.code || 'INTERNAL_ERROR',
    },
  }),
}));

app.listen(4000);
```

### Apollo Server

```javascript
import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';
import express from 'express';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  formatError: (error) => {
    // Логирование ошибок
    console.error(error);

    // Скрываем детали в продакшене
    if (process.env.NODE_ENV === 'production') {
      return { message: 'Internal server error' };
    }
    return error;
  },
});

await server.start();

const app = express();
app.use('/graphql', express.json(), expressMiddleware(server));
```

### Python с Strawberry

```python
from strawberry.flask.views import GraphQLView
from flask import Flask
import strawberry

@strawberry.type
class Query:
    @strawberry.field
    def hello(self) -> str:
        return "Hello World"

schema = strawberry.Schema(query=Query)

app = Flask(__name__)
app.add_url_rule(
    "/graphql",
    view_func=GraphQLView.as_view("graphql_view", schema=schema),
)
```

## Batching (Пакетные запросы)

### Отправка нескольких запросов

```http
POST /graphql HTTP/1.1
Content-Type: application/json

[
  { "query": "{ user(id: \"1\") { name } }" },
  { "query": "{ post(id: \"5\") { title } }" }
]
```

### Ответ на пакетный запрос

```json
[
  { "data": { "user": { "name": "John" } } },
  { "data": { "post": { "title": "Hello World" } } }
]
```

### Настройка batching в Apollo

```javascript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  allowBatchedHttpRequests: true,
});
```

## Persisted Queries

### Зачем нужны

- Уменьшение размера запросов
- Улучшение безопасности
- Возможность кэширования

### Автоматические persisted queries (APQ)

```http
# Первый запрос - отправляем хеш
POST /graphql
{
  "extensions": {
    "persistedQuery": {
      "version": 1,
      "sha256Hash": "abc123..."
    }
  }
}

# Если сервер не знает хеш, он вернёт ошибку
{
  "errors": [{
    "message": "PersistedQueryNotFound"
  }]
}

# Клиент отправляет полный запрос с хешем
POST /graphql
{
  "query": "{ user(id: \"1\") { name } }",
  "extensions": {
    "persistedQuery": {
      "version": 1,
      "sha256Hash": "abc123..."
    }
  }
}
```

## CORS (Cross-Origin Resource Sharing)

```javascript
import cors from 'cors';

app.use('/graphql', cors({
  origin: ['https://app.example.com'],
  methods: ['POST', 'GET', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
}));
```

## Сжатие

### Gzip/Brotli

```javascript
import compression from 'compression';

app.use(compression());
app.use('/graphql', expressMiddleware(server));
```

### Клиентская сторона

```http
GET /graphql?query=...
Accept-Encoding: gzip, br
```

## Практические советы

1. **Всегда используйте POST для мутаций** — это предотвращает случайное кэширование
2. **Логируйте все запросы** — используйте X-Request-ID для трассировки
3. **Устанавливайте таймауты** — защита от медленных запросов
4. **Ограничивайте размер тела запроса** — защита от DoS

```javascript
app.use('/graphql', express.json({ limit: '1mb' }));
```

## Заключение

Правильная настройка HTTP-транспорта для GraphQL обеспечивает надёжную работу API. Следуйте стандартам, используйте правильные статус-коды и заголовки, и ваш API будет удобен для интеграции.

---

[prev: 02-thinking-in-graphs](./02-thinking-in-graphs.md) | [next: 04-file-uploads](./04-file-uploads.md)
