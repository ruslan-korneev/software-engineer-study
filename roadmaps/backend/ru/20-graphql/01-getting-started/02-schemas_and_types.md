# Схемы и типы данных в GraphQL

## Что такое схема?

**Схема (Schema)** — это контракт между клиентом и сервером, описывающий структуру данных API. Схема написана на SDL (Schema Definition Language) и определяет:
- Какие типы данных доступны
- Какие запросы можно выполнять
- Какие мутации можно делать
- Как связаны типы между собой

## Schema Definition Language (SDL)

SDL — декларативный язык для описания GraphQL схем.

```graphql
# Определение типа
type User {
  id: ID!
  name: String!
  email: String!
  age: Int
  isActive: Boolean!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String
  author: User!
  createdAt: String!
}
```

## Скалярные типы

GraphQL имеет 5 встроенных скалярных типов:

| Тип | Описание | Пример |
|-----|----------|--------|
| `Int` | 32-битное целое число | `42` |
| `Float` | Число с плавающей точкой | `3.14` |
| `String` | Строка UTF-8 | `"Hello"` |
| `Boolean` | Логическое значение | `true`, `false` |
| `ID` | Уникальный идентификатор | `"abc123"` |

### Пользовательские скалярные типы

```graphql
# Определение кастомного скаляра
scalar DateTime
scalar Email
scalar URL
scalar JSON

type Event {
  id: ID!
  name: String!
  startDate: DateTime!
  endDate: DateTime!
}
```

```javascript
// Реализация на сервере (Apollo Server)
const { GraphQLScalarType, Kind } = require('graphql');

const DateTimeScalar = new GraphQLScalarType({
  name: 'DateTime',
  description: 'Дата и время в формате ISO 8601',

  // Сериализация: сервер -> клиент
  serialize(value) {
    return value instanceof Date ? value.toISOString() : value;
  },

  // Парсинг значения из переменных
  parseValue(value) {
    return new Date(value);
  },

  // Парсинг литерала из запроса
  parseLiteral(ast) {
    if (ast.kind === Kind.STRING) {
      return new Date(ast.value);
    }
    return null;
  },
});
```

## Объектные типы (Object Types)

Объектные типы описывают сущности с полями.

```graphql
type User {
  # Поле с обязательным скалярным значением
  id: ID!

  # Поле с обязательной строкой
  name: String!

  # Необязательное поле (может быть null)
  bio: String

  # Список постов (обязательный, но может быть пустым)
  posts: [Post!]!

  # Поле с аргументами
  friends(first: Int = 10, after: String): [User!]!
}
```

### Модификаторы типов

| Синтаксис | Значение |
|-----------|----------|
| `String` | Nullable строка |
| `String!` | Non-null строка (обязательная) |
| `[String]` | Nullable список nullable строк |
| `[String!]` | Nullable список non-null строк |
| `[String]!` | Non-null список nullable строк |
| `[String!]!` | Non-null список non-null строк |

```graphql
# Примеры возможных значений
type Example {
  a: String          # null, "hello"
  b: String!         # "hello" (не может быть null)
  c: [String]        # null, [], ["a", null, "b"]
  d: [String!]       # null, [], ["a", "b"]
  e: [String]!       # [], ["a", null, "b"] (не может быть null)
  f: [String!]!      # [], ["a", "b"] (ничего не может быть null)
}
```

## Аргументы полей

Поля могут принимать аргументы для фильтрации, пагинации и т.д.

```graphql
type Query {
  # Простой аргумент
  user(id: ID!): User

  # Множество аргументов с значениями по умолчанию
  users(
    limit: Int = 10
    offset: Int = 0
    sortBy: String = "createdAt"
    order: SortOrder = DESC
  ): [User!]!

  # Поиск с фильтрами
  searchPosts(
    query: String!
    authorId: ID
    tags: [String!]
    publishedAfter: DateTime
  ): [Post!]!
}

enum SortOrder {
  ASC
  DESC
}
```

## Перечисления (Enums)

Enums определяют набор допустимых значений.

```graphql
enum UserRole {
  ADMIN
  MODERATOR
  USER
  GUEST
}

enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}

enum SortDirection {
  ASC
  DESC
}

type User {
  id: ID!
  name: String!
  role: UserRole!
}

type Post {
  id: ID!
  title: String!
  status: PostStatus!
}

type Query {
  users(role: UserRole): [User!]!
  posts(status: PostStatus, sortDir: SortDirection): [Post!]!
}
```

## Интерфейсы (Interfaces)

Интерфейсы определяют набор полей, которые должны реализовать типы.

```graphql
interface Node {
  id: ID!
}

interface Timestamped {
  createdAt: DateTime!
  updatedAt: DateTime!
}

type User implements Node & Timestamped {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  name: String!
  email: String!
}

type Post implements Node & Timestamped {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  title: String!
  content: String!
}

type Query {
  node(id: ID!): Node
}
```

```graphql
# Запрос с использованием интерфейса
query {
  node(id: "user:1") {
    id
    ... on User {
      name
      email
    }
    ... on Post {
      title
      content
    }
  }
}
```

## Union Types

Union типы позволяют полю возвращать один из нескольких типов.

```graphql
union SearchResult = User | Post | Comment

type Query {
  search(query: String!): [SearchResult!]!
}
```

```graphql
# Запрос с union типом
query {
  search(query: "GraphQL") {
    ... on User {
      name
      email
    }
    ... on Post {
      title
      content
    }
    ... on Comment {
      text
      author {
        name
      }
    }
  }
}
```

### Различие между Interface и Union

| Характеристика | Interface | Union |
|----------------|-----------|-------|
| Общие поля | Обязательны | Нет |
| Использование | Типы с общей структурой | Разнородные типы |
| Inline fragments | Опционально для общих полей | Обязательно |

## Input Types

Input типы используются для передачи сложных аргументов в запросы и мутации.

```graphql
input CreateUserInput {
  name: String!
  email: String!
  password: String!
  role: UserRole = USER
}

input UpdateUserInput {
  name: String
  email: String
  bio: String
}

input UserFilterInput {
  roles: [UserRole!]
  isActive: Boolean
  createdAfter: DateTime
  createdBefore: DateTime
}

input PaginationInput {
  limit: Int = 20
  offset: Int = 0
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
}

type Query {
  users(filter: UserFilterInput, pagination: PaginationInput): [User!]!
}
```

```graphql
# Использование input типов
mutation {
  createUser(input: {
    name: "John Doe"
    email: "john@example.com"
    password: "secret123"
    role: ADMIN
  }) {
    id
    name
    role
  }
}
```

## Директивы

Директивы модифицируют выполнение запросов или схемы.

### Встроенные директивы

```graphql
query GetUser($includeEmail: Boolean!, $skipPosts: Boolean!) {
  user(id: "1") {
    name
    # Включить поле, если условие истинно
    email @include(if: $includeEmail)
    # Пропустить поле, если условие истинно
    posts @skip(if: $skipPosts) {
      title
    }
  }
}

# Директива deprecated в схеме
type User {
  id: ID!
  name: String!
  username: String @deprecated(reason: "Используйте поле 'name'")
}
```

### Пользовательские директивы

```graphql
# Определение директив в схеме
directive @auth(requires: Role!) on FIELD_DEFINITION
directive @length(min: Int, max: Int) on INPUT_FIELD_DEFINITION
directive @uppercase on FIELD_DEFINITION

type Query {
  # Требует авторизации
  me: User @auth(requires: USER)

  # Только для админов
  adminDashboard: AdminData @auth(requires: ADMIN)
}

type User {
  id: ID!
  # Преобразует значение в верхний регистр
  name: String! @uppercase
}

input CreatePostInput {
  # Валидация длины
  title: String! @length(min: 5, max: 100)
  content: String! @length(min: 10)
}
```

## Корневые типы

GraphQL схема имеет три корневых типа:

```graphql
schema {
  query: Query        # Обязательный
  mutation: Mutation  # Опциональный
  subscription: Subscription  # Опциональный
}

type Query {
  users: [User!]!
  user(id: ID!): User
  posts: [Post!]!
  post(id: ID!): Post
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
}

type Subscription {
  userCreated: User!
  postPublished: Post!
}
```

## Расширение типов (Type Extensions)

Позволяют разбивать схему на модули.

```graphql
# base-schema.graphql
type Query {
  health: String!
}

type User {
  id: ID!
  name: String!
}

# users-extension.graphql
extend type Query {
  users: [User!]!
  user(id: ID!): User
}

extend type User {
  email: String!
  posts: [Post!]!
}

# posts-extension.graphql
extend type Query {
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  author: User!
}
```

## Практический пример полной схемы

```graphql
# Скаляры
scalar DateTime
scalar Email

# Перечисления
enum UserRole {
  ADMIN
  USER
  GUEST
}

enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}

# Интерфейсы
interface Node {
  id: ID!
}

interface Timestamped {
  createdAt: DateTime!
  updatedAt: DateTime!
}

# Объектные типы
type User implements Node & Timestamped {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  name: String!
  email: Email!
  role: UserRole!
  bio: String
  posts(status: PostStatus): [Post!]!
  followers: [User!]!
  following: [User!]!
}

type Post implements Node & Timestamped {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  title: String!
  content: String!
  status: PostStatus!
  author: User!
  tags: [String!]!
  comments: [Comment!]!
}

type Comment implements Node & Timestamped {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  text: String!
  author: User!
  post: Post!
}

# Input типы
input CreateUserInput {
  name: String!
  email: Email!
  password: String!
  role: UserRole = USER
}

input CreatePostInput {
  title: String!
  content: String!
  tags: [String!] = []
  status: PostStatus = DRAFT
}

input PostFilterInput {
  authorId: ID
  status: PostStatus
  tags: [String!]
  search: String
}

# Корневые типы
type Query {
  # Пользователи
  me: User
  user(id: ID!): User
  users(role: UserRole, limit: Int = 10): [User!]!

  # Посты
  post(id: ID!): Post
  posts(filter: PostFilterInput, limit: Int = 10, offset: Int = 0): [Post!]!

  # Общий поиск
  node(id: ID!): Node
}

type Mutation {
  # Авторизация
  signUp(input: CreateUserInput!): AuthPayload!
  signIn(email: Email!, password: String!): AuthPayload!

  # Посты
  createPost(input: CreatePostInput!): Post!
  updatePost(id: ID!, input: CreatePostInput!): Post!
  deletePost(id: ID!): Boolean!
  publishPost(id: ID!): Post!
}

type Subscription {
  postPublished: Post!
  commentAdded(postId: ID!): Comment!
}

type AuthPayload {
  token: String!
  user: User!
}
```

## Практические советы

1. **Используйте Non-null (`!`) осознанно** — это контракт, который сложно изменить
2. **Предпочитайте Input типы** — вместо множества аргументов
3. **Следуйте конвенциям именования**:
   - Типы: PascalCase (`User`, `Post`)
   - Поля: camelCase (`firstName`, `createdAt`)
   - Enums: SCREAMING_SNAKE_CASE (`ADMIN`, `PUBLISHED`)
4. **Проектируйте для клиентов** — думайте о том, какие данные нужны UI
5. **Документируйте схему** — используйте описания для полей и типов

```graphql
"""
Пользователь системы.
Может иметь разные роли и публиковать посты.
"""
type User {
  "Уникальный идентификатор пользователя"
  id: ID!

  "Полное имя пользователя"
  name: String!

  "Email адрес (уникальный)"
  email: Email!
}
```

## Заключение

Схема — это сердце GraphQL API. Правильно спроектированная схема делает API интуитивно понятным, типобезопасным и легко расширяемым. В следующих разделах мы рассмотрим, как выполнять запросы к этой схеме.
