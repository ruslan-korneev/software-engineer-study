# Типичные ошибки GraphQL over HTTP

[prev: 12-federation](./12-federation.md) | [next: 01-introduction](../../21-system-design/01-introduction.md)

---

## Введение

При работе с GraphQL через HTTP возникают специфические ошибки, связанные как с транспортным уровнем (HTTP), так и с выполнением GraphQL операций. Понимание этих ошибок и способов их обработки важно для создания надёжного API.

## Классификация ошибок

| Уровень | Тип ошибки | HTTP статус | Пример |
|---------|------------|-------------|--------|
| Транспорт | Сетевые | 5xx | Timeout, connection refused |
| HTTP | Клиентские | 4xx | 400, 401, 403, 405 |
| GraphQL | Синтаксис | 400 | Невалидный запрос |
| GraphQL | Валидация | 200 | Неизвестное поле |
| GraphQL | Выполнение | 200 | Ошибка в резолвере |

## Ошибки транспортного уровня

### 1. Connection Refused / Timeout

```javascript
// Клиент
try {
  const response = await fetch('/graphql', {
    method: 'POST',
    body: JSON.stringify({ query }),
  });
} catch (error) {
  if (error.name === 'TypeError') {
    // Network error
    console.error('Network error:', error.message);
  }
}

// Обработка таймаута
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 30000);

try {
  const response = await fetch('/graphql', {
    signal: controller.signal,
    // ...
  });
} catch (error) {
  if (error.name === 'AbortError') {
    console.error('Request timeout');
  }
}
```

### 2. SSL/TLS ошибки

```
Error: unable to verify the first certificate
Error: self signed certificate
```

Решение:
- Убедитесь в правильности сертификатов
- Не используйте `rejectUnauthorized: false` в production

## HTTP ошибки (4xx)

### 400 Bad Request

Причины:
- Невалидный JSON в теле запроса
- Отсутствует поле `query`
- Синтаксическая ошибка GraphQL

```http
POST /graphql HTTP/1.1
Content-Type: application/json

{ invalid json }
```

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "errors": [{
    "message": "Syntax Error: Expected Name, found <EOF>",
    "locations": [{ "line": 1, "column": 1 }]
  }]
}
```

### 401 Unauthorized

```http
POST /graphql HTTP/1.1
Content-Type: application/json
Authorization: Bearer invalid_token

{ "query": "{ me { name } }" }
```

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="api"

{
  "errors": [{
    "message": "Invalid or expired token",
    "extensions": {
      "code": "UNAUTHENTICATED"
    }
  }]
}
```

Обработка на сервере:

```javascript
const server = new ApolloServer({
  context: async ({ req }) => {
    const token = req.headers.authorization?.split(' ')[1];

    if (!token) {
      throw new GraphQLError('Authentication required', {
        extensions: {
          code: 'UNAUTHENTICATED',
          http: { status: 401 },
        },
      });
    }

    try {
      const user = await verifyToken(token);
      return { user };
    } catch (error) {
      throw new GraphQLError('Invalid token', {
        extensions: {
          code: 'UNAUTHENTICATED',
          http: { status: 401 },
        },
      });
    }
  },
});
```

### 403 Forbidden

```javascript
// Резолвер
const resolvers = {
  Mutation: {
    deleteUser: (_, { id }, context) => {
      if (!context.user.isAdmin) {
        throw new GraphQLError('Admin access required', {
          extensions: {
            code: 'FORBIDDEN',
            http: { status: 403 },
          },
        });
      }
      return db.users.delete(id);
    },
  },
};
```

### 405 Method Not Allowed

```http
DELETE /graphql HTTP/1.1

HTTP/1.1 405 Method Not Allowed
Allow: GET, POST
```

Сервер должен принимать только GET и POST:

```javascript
app.use('/graphql', (req, res, next) => {
  if (!['GET', 'POST'].includes(req.method)) {
    res.status(405).set('Allow', 'GET, POST').send('Method Not Allowed');
    return;
  }
  next();
});
```

### 415 Unsupported Media Type

```http
POST /graphql HTTP/1.1
Content-Type: text/plain

query { users { name } }
```

```http
HTTP/1.1 415 Unsupported Media Type

{
  "errors": [{
    "message": "Content-Type must be application/json"
  }]
}
```

### 429 Too Many Requests

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60

{
  "errors": [{
    "message": "Rate limit exceeded",
    "extensions": {
      "code": "RATE_LIMITED",
      "retryAfter": 60
    }
  }]
}
```

## GraphQL ошибки (возвращаются с 200 OK)

### Ошибки синтаксиса

```graphql
query {
  user(id: "1" {  # Забыли закрыть скобку
    name
  }
}
```

```json
{
  "errors": [{
    "message": "Syntax Error: Expected Name, found \"{\"",
    "locations": [{ "line": 2, "column": 17 }]
  }]
}
```

### Ошибки валидации

```graphql
query {
  user(id: "1") {
    unknownField    # Поле не существует
  }
}
```

```json
{
  "errors": [{
    "message": "Cannot query field \"unknownField\" on type \"User\"",
    "locations": [{ "line": 3, "column": 5 }]
  }]
}
```

### Ошибки выполнения

```json
{
  "data": {
    "user": null
  },
  "errors": [{
    "message": "User not found",
    "locations": [{ "line": 2, "column": 3 }],
    "path": ["user"],
    "extensions": {
      "code": "NOT_FOUND"
    }
  }]
}
```

### Частичные данные с ошибками

```json
{
  "data": {
    "users": [
      { "id": "1", "name": "John", "email": "john@example.com" },
      { "id": "2", "name": "Jane", "email": null }
    ]
  },
  "errors": [{
    "message": "User 2 email is private",
    "path": ["users", 1, "email"],
    "extensions": { "code": "FORBIDDEN" }
  }]
}
```

## Форматирование ошибок

### Структура ошибки по спецификации

```typescript
interface GraphQLError {
  message: string;                    // Обязательно
  locations?: { line: number; column: number }[];  // Позиция в запросе
  path?: (string | number)[];         // Путь к полю
  extensions?: {                      // Дополнительные данные
    code?: string;
    timestamp?: string;
    [key: string]: any;
  };
}
```

### Кастомное форматирование

```javascript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  formatError: (formattedError, error) => {
    // Логируем полную ошибку
    console.error(error);

    // В production скрываем детали
    if (process.env.NODE_ENV === 'production') {
      // Скрываем stack trace
      delete formattedError.extensions?.exception;

      // Скрываем внутренние ошибки
      if (formattedError.extensions?.code === 'INTERNAL_SERVER_ERROR') {
        return {
          message: 'Internal server error',
          extensions: {
            code: 'INTERNAL_SERVER_ERROR',
            timestamp: new Date().toISOString(),
          },
        };
      }
    }

    return formattedError;
  },
});
```

## Коды ошибок

### Стандартные коды Apollo

| Код | Описание | HTTP аналог |
|-----|----------|-------------|
| `GRAPHQL_PARSE_FAILED` | Синтаксическая ошибка | 400 |
| `GRAPHQL_VALIDATION_FAILED` | Ошибка валидации | 400 |
| `BAD_USER_INPUT` | Неверный ввод | 400 |
| `UNAUTHENTICATED` | Не аутентифицирован | 401 |
| `FORBIDDEN` | Нет доступа | 403 |
| `PERSISTED_QUERY_NOT_FOUND` | Запрос не найден | 400 |
| `INTERNAL_SERVER_ERROR` | Ошибка сервера | 500 |

### Кастомные коды

```javascript
// Определение кодов ошибок
const ErrorCodes = {
  USER_NOT_FOUND: 'USER_NOT_FOUND',
  EMAIL_ALREADY_EXISTS: 'EMAIL_ALREADY_EXISTS',
  INVALID_CREDENTIALS: 'INVALID_CREDENTIALS',
  RESOURCE_LOCKED: 'RESOURCE_LOCKED',
};

// Использование
throw new GraphQLError('User not found', {
  extensions: {
    code: ErrorCodes.USER_NOT_FOUND,
    userId: id,
  },
});
```

## Обработка ошибок на клиенте

### Apollo Client

```javascript
import { ApolloClient, from, HttpLink } from '@apollo/client';
import { onError } from '@apollo/client/link/error';

const errorLink = onError(({ graphQLErrors, networkError, operation }) => {
  if (graphQLErrors) {
    graphQLErrors.forEach(({ message, locations, path, extensions }) => {
      console.error(`GraphQL Error: ${message}`);

      switch (extensions?.code) {
        case 'UNAUTHENTICATED':
          // Редирект на логин
          window.location.href = '/login';
          break;
        case 'FORBIDDEN':
          // Показать уведомление
          showToast('Access denied');
          break;
      }
    });
  }

  if (networkError) {
    console.error(`Network Error: ${networkError.message}`);

    if (networkError.statusCode === 429) {
      showToast('Too many requests. Please wait.');
    }
  }
});

const client = new ApolloClient({
  link: from([errorLink, new HttpLink({ uri: '/graphql' })]),
  cache: new InMemoryCache(),
});
```

### React компонент

```javascript
function UserProfile({ userId }) {
  const { data, loading, error } = useQuery(GET_USER, {
    variables: { id: userId },
  });

  if (loading) return <Spinner />;

  if (error) {
    const code = error.graphQLErrors?.[0]?.extensions?.code;

    switch (code) {
      case 'NOT_FOUND':
        return <NotFound message="User not found" />;
      case 'FORBIDDEN':
        return <AccessDenied />;
      default:
        return <ErrorMessage error={error} />;
    }
  }

  return <UserCard user={data.user} />;
}
```

## Логирование ошибок

```javascript
const loggingPlugin = {
  requestDidStart({ request }) {
    return {
      didEncounterErrors({ errors, operation }) {
        errors.forEach((error) => {
          const errorLog = {
            message: error.message,
            code: error.extensions?.code,
            path: error.path,
            operation: operation.operationName,
            timestamp: new Date().toISOString(),
          };

          if (error.extensions?.code === 'INTERNAL_SERVER_ERROR') {
            // Критическая ошибка
            logger.error('GraphQL Internal Error', errorLog);
            alerting.notify(errorLog);
          } else {
            // Обычная ошибка
            logger.warn('GraphQL Error', errorLog);
          }
        });
      },
    };
  },
};
```

## Retry стратегии

```javascript
import { RetryLink } from '@apollo/client/link/retry';

const retryLink = new RetryLink({
  delay: {
    initial: 300,
    max: 5000,
    jitter: true,
  },
  attempts: {
    max: 3,
    retryIf: (error, operation) => {
      // Retry только для сетевых ошибок
      return !!error && !error.graphQLErrors;
    },
  },
});
```

## Лучшие практики

1. **Используйте коды ошибок** — машиночитаемые, не только сообщения
2. **Не раскрывайте детали** — скрывайте stack trace в production
3. **Логируйте всё** — для отладки и мониторинга
4. **Различайте типы ошибок** — клиентские, серверные, бизнес-логика
5. **Возвращайте частичные данные** — когда возможно
6. **Документируйте ошибки** — какие коды возможны для каждой операции

## Заключение

Правильная обработка ошибок GraphQL over HTTP требует понимания различных уровней, где могут возникнуть проблемы. Используйте стандартные коды ошибок, форматируйте ответы для удобства клиентов и не забывайте о логировании и мониторинге.

---

[prev: 12-federation](./12-federation.md) | [next: 01-introduction](../../21-system-design/01-introduction.md)
