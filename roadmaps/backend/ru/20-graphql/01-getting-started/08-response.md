# Формат ответа в GraphQL

## Структура ответа

Каждый ответ GraphQL — это JSON объект с определённой структурой. Спецификация GraphQL строго определяет формат ответа.

### Основные поля ответа

```json
{
  "data": { ... },      // Результат выполнения запроса
  "errors": [ ... ],    // Массив ошибок (если есть)
  "extensions": { ... } // Дополнительная информация (опционально)
}
```

| Поле | Тип | Обязательность | Описание |
|------|-----|----------------|----------|
| `data` | Object/null | Условно | Результат выполнения |
| `errors` | Array | Условно | Ошибки выполнения |
| `extensions` | Object | Нет | Метаданные |

## Поле data

### Успешный ответ

Структура `data` точно соответствует структуре запроса.

```graphql
query {
  user(id: "1") {
    id
    name
    email
    posts {
      title
    }
  }
}
```

```json
{
  "data": {
    "user": {
      "id": "1",
      "name": "John Doe",
      "email": "john@example.com",
      "posts": [
        { "title": "First Post" },
        { "title": "Second Post" }
      ]
    }
  }
}
```

### Null значения

```graphql
query {
  user(id: "999") {  # Несуществующий пользователь
    name
  }
  post(id: "1") {
    title
    subtitle  # Nullable поле без значения
  }
}
```

```json
{
  "data": {
    "user": null,
    "post": {
      "title": "My Post",
      "subtitle": null
    }
  }
}
```

### Пустые списки

```graphql
query {
  user(id: "1") {
    name
    posts {  # Нет постов
      title
    }
  }
}
```

```json
{
  "data": {
    "user": {
      "name": "John",
      "posts": []
    }
  }
}
```

## Поле errors

### Структура ошибки

```json
{
  "errors": [
    {
      "message": "Описание ошибки",
      "locations": [
        {
          "line": 3,
          "column": 5
        }
      ],
      "path": ["user", "posts", 0, "title"],
      "extensions": {
        "code": "INTERNAL_ERROR",
        "timestamp": "2024-01-15T10:30:00Z"
      }
    }
  ]
}
```

| Поле | Описание |
|------|----------|
| `message` | Человекочитаемое описание ошибки |
| `locations` | Позиция в запросе (строка, колонка) |
| `path` | Путь до поля, вызвавшего ошибку |
| `extensions` | Дополнительная информация |

### Типы ошибок

#### Синтаксические ошибки

```json
{
  "errors": [
    {
      "message": "Syntax Error: Expected Name, found \"}\"",
      "locations": [
        { "line": 4, "column": 1 }
      ]
    }
  ]
}
```

#### Ошибки валидации

```json
{
  "errors": [
    {
      "message": "Cannot query field \"username\" on type \"User\". Did you mean \"name\"?",
      "locations": [
        { "line": 3, "column": 5 }
      ]
    }
  ]
}
```

#### Ошибки выполнения

```json
{
  "errors": [
    {
      "message": "User not found",
      "locations": [
        { "line": 2, "column": 3 }
      ],
      "path": ["user"],
      "extensions": {
        "code": "NOT_FOUND",
        "resourceType": "User",
        "resourceId": "999"
      }
    }
  ]
}
```

## Partial Response (Частичный ответ)

GraphQL возвращает данные даже при наличии ошибок.

```graphql
query {
  user(id: "1") {
    name
    email
    secretField  # Требует авторизации
  }
  posts {
    title
  }
}
```

```json
{
  "data": {
    "user": {
      "name": "John",
      "email": "john@example.com",
      "secretField": null
    },
    "posts": [
      { "title": "Post 1" },
      { "title": "Post 2" }
    ]
  },
  "errors": [
    {
      "message": "Not authorized to access secretField",
      "path": ["user", "secretField"],
      "extensions": {
        "code": "FORBIDDEN"
      }
    }
  ]
}
```

### Null Propagation

Если non-null поле возвращает ошибку, null "всплывает" к ближайшему nullable родителю.

```graphql
# Схема
type User {
  id: ID!
  name: String!      # Non-null
  email: String!     # Non-null
}

type Query {
  user(id: ID!): User  # Nullable
}
```

```graphql
query {
  user(id: "1") {
    name
    email  # Ошибка при получении
  }
}
```

```json
{
  "data": {
    "user": null  // Весь user = null из-за ошибки в non-null email
  },
  "errors": [
    {
      "message": "Failed to fetch email",
      "path": ["user", "email"]
    }
  ]
}
```

### Null Propagation в списках

```graphql
# Схема
type Post {
  id: ID!
  title: String!
}

type Query {
  posts: [Post!]!  # Non-null список non-null элементов
}
```

```graphql
query {
  posts {
    id
    title  # Ошибка в одном из постов
  }
}
```

```json
{
  "data": null,  // Весь ответ null, т.к. [Post!]! не допускает null элементы
  "errors": [
    {
      "message": "Failed to fetch title for post 2",
      "path": ["posts", 1, "title"]
    }
  ]
}
```

## Extensions

Поле `extensions` используется для передачи метаданных.

### Стандартные расширения

```json
{
  "data": { ... },
  "extensions": {
    "tracing": {
      "version": 1,
      "startTime": "2024-01-15T10:30:00.000Z",
      "endTime": "2024-01-15T10:30:00.150Z",
      "duration": 150000000,
      "execution": {
        "resolvers": [
          {
            "path": ["user"],
            "parentType": "Query",
            "fieldName": "user",
            "returnType": "User",
            "startOffset": 10000,
            "duration": 50000000
          }
        ]
      }
    }
  }
}
```

### Пользовательские расширения

```json
{
  "data": { ... },
  "extensions": {
    "requestId": "abc-123-def",
    "complexity": 42,
    "rateLimit": {
      "remaining": 98,
      "resetAt": "2024-01-15T11:00:00Z"
    },
    "cacheControl": {
      "maxAge": 300,
      "scope": "PRIVATE"
    },
    "deprecationWarnings": [
      {
        "field": "user.username",
        "reason": "Use 'name' instead"
      }
    ]
  }
}
```

### Добавление extensions на сервере

```javascript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    {
      async requestDidStart() {
        return {
          async willSendResponse(requestContext) {
            requestContext.response.extensions = {
              ...requestContext.response.extensions,
              requestId: requestContext.context.requestId,
              serverTime: new Date().toISOString()
            };
          }
        };
      }
    }
  ]
});
```

## HTTP статусы

GraphQL всегда возвращает HTTP 200 OK, даже при ошибках (за редкими исключениями).

| Ситуация | HTTP статус | Ответ |
|----------|-------------|-------|
| Успешный запрос | 200 | `{ "data": {...} }` |
| Частичный успех | 200 | `{ "data": {...}, "errors": [...] }` |
| Ошибка выполнения | 200 | `{ "data": null, "errors": [...] }` |
| Синтаксическая ошибка | 400 | `{ "errors": [...] }` |
| Невалидный JSON | 400 | Ошибка парсинга |
| Internal server error | 500 | Зависит от реализации |

## Форматирование ответов

### Кастомные форматтеры ошибок

```javascript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  formatError: (formattedError, error) => {
    // Не показывать внутренние ошибки клиенту
    if (error.originalError instanceof DatabaseError) {
      return {
        message: 'Internal server error',
        extensions: {
          code: 'INTERNAL_ERROR'
        }
      };
    }

    // Добавить timestamp
    return {
      ...formattedError,
      extensions: {
        ...formattedError.extensions,
        timestamp: new Date().toISOString()
      }
    };
  }
});
```

### Форматирование ответа

```javascript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  formatResponse: (response, requestContext) => {
    // Добавить метаданные
    return {
      ...response,
      extensions: {
        ...response.extensions,
        requestId: requestContext.context.requestId,
        executionTime: Date.now() - requestContext.context.startTime
      }
    };
  }
});
```

## Обработка на клиенте

### JavaScript/TypeScript

```typescript
interface GraphQLResponse<T> {
  data?: T;
  errors?: GraphQLError[];
  extensions?: Record<string, any>;
}

interface GraphQLError {
  message: string;
  locations?: { line: number; column: number }[];
  path?: (string | number)[];
  extensions?: {
    code?: string;
    [key: string]: any;
  };
}

async function executeQuery<T>(query: string, variables?: object): Promise<T> {
  const response = await fetch('/graphql', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query, variables })
  });

  const result: GraphQLResponse<T> = await response.json();

  // Обработка ошибок
  if (result.errors?.length) {
    const authError = result.errors.find(
      e => e.extensions?.code === 'UNAUTHENTICATED'
    );
    if (authError) {
      redirectToLogin();
      throw new AuthenticationError(authError.message);
    }

    // Partial data — можно продолжить
    if (result.data) {
      console.warn('Partial response:', result.errors);
    } else {
      throw new GraphQLExecutionError(result.errors);
    }
  }

  return result.data!;
}
```

### Apollo Client

```javascript
import { ApolloClient, InMemoryCache, ApolloLink } from '@apollo/client';
import { onError } from '@apollo/client/link/error';

const errorLink = onError(({ graphQLErrors, networkError, operation }) => {
  if (graphQLErrors) {
    graphQLErrors.forEach(({ message, locations, path, extensions }) => {
      console.log(
        `[GraphQL error]: Message: ${message}, Path: ${path}`
      );

      // Обработка по коду ошибки
      switch (extensions?.code) {
        case 'UNAUTHENTICATED':
          // Обновить токен или перенаправить на логин
          break;
        case 'FORBIDDEN':
          // Показать сообщение о недостаточных правах
          break;
        case 'NOT_FOUND':
          // Перенаправить на 404
          break;
      }
    });
  }

  if (networkError) {
    console.log(`[Network error]: ${networkError}`);
  }
});

const client = new ApolloClient({
  link: ApolloLink.from([errorLink, httpLink]),
  cache: new InMemoryCache()
});
```

## Паттерны ответов

### Result Type Pattern

```graphql
type CreateUserSuccess {
  user: User!
}

type CreateUserError {
  message: String!
  code: String!
  field: String
}

union CreateUserResult = CreateUserSuccess | CreateUserError

type Mutation {
  createUser(input: CreateUserInput!): CreateUserResult!
}
```

```graphql
mutation {
  createUser(input: { name: "John", email: "invalid" }) {
    ... on CreateUserSuccess {
      user {
        id
        name
      }
    }
    ... on CreateUserError {
      message
      code
      field
    }
  }
}
```

### Payload Pattern

```graphql
type CreateUserPayload {
  success: Boolean!
  user: User
  errors: [FieldError!]!
}

type FieldError {
  field: String!
  message: String!
}
```

```json
{
  "data": {
    "createUser": {
      "success": false,
      "user": null,
      "errors": [
        { "field": "email", "message": "Invalid email format" },
        { "field": "password", "message": "Too short" }
      ]
    }
  }
}
```

### Connection Pattern (Пагинация)

```graphql
type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type UserEdge {
  cursor: String!
  node: User!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

```json
{
  "data": {
    "users": {
      "edges": [
        {
          "cursor": "YXJyYXljb25uZWN0aW9uOjA=",
          "node": { "id": "1", "name": "Alice" }
        },
        {
          "cursor": "YXJyYXljb25uZWN0aW9uOjE=",
          "node": { "id": "2", "name": "Bob" }
        }
      ],
      "pageInfo": {
        "hasNextPage": true,
        "hasPreviousPage": false,
        "startCursor": "YXJyYXljb25uZWN0aW9uOjA=",
        "endCursor": "YXJyYXljb25uZWN0aW9uOjE="
      },
      "totalCount": 100
    }
  }
}
```

## Практические советы

### Что делать

1. **Проверяйте наличие errors** — даже при наличии data
2. **Используйте extensions** — для метаданных, не смешивайте с data
3. **Типизируйте ответы** — для надёжности на клиенте
4. **Обрабатывайте partial data** — не отбрасывайте успешные части
5. **Логируйте path ошибок** — для отладки

### Чего избегать

1. **Не полагайтесь на HTTP статусы** — всегда проверяйте тело ответа
2. **Не игнорируйте extensions** — могут содержать важную информацию
3. **Не раскрывайте внутренние ошибки** — форматируйте для безопасности
4. **Не забывайте про null propagation** — учитывайте при проектировании схемы

## Заключение

Понимание формата ответа GraphQL критически важно для построения надёжных клиентов и серверов. Правильная обработка ошибок, partial responses и extensions позволяет создавать устойчивые приложения, которые корректно работают даже при частичных сбоях. Всегда проектируйте схему с учётом null propagation и используйте паттерны Result Type или Payload для явной обработки ошибок бизнес-логики.
