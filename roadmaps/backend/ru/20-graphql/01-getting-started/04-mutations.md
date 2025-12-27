# Мутации в GraphQL (Mutations)

## Что такое Mutation?

**Mutation** — это операция в GraphQL для изменения данных на сервере. Если Query используется для чтения, то Mutation — для создания, обновления и удаления данных. Мутации выполняются последовательно (в отличие от параллельного выполнения Query), что гарантирует предсказуемый порядок изменений.

## Сравнение с REST

| Операция | REST | GraphQL |
|----------|------|---------|
| Создание | `POST /users` | `mutation { createUser(...) }` |
| Обновление | `PUT /users/1` или `PATCH /users/1` | `mutation { updateUser(...) }` |
| Удаление | `DELETE /users/1` | `mutation { deleteUser(...) }` |

## Базовая структура мутации

```graphql
# Схема
type Mutation {
  createUser(name: String!, email: String!): User!
  updateUser(id: ID!, name: String, email: String): User!
  deleteUser(id: ID!): Boolean!
}

# Запрос мутации
mutation CreateUser {
  createUser(name: "John Doe", email: "john@example.com") {
    id
    name
    email
    createdAt
  }
}
```

### Ответ сервера

```json
{
  "data": {
    "createUser": {
      "id": "123",
      "name": "John Doe",
      "email": "john@example.com",
      "createdAt": "2024-01-15T10:30:00Z"
    }
  }
}
```

## Input типы в мутациях

Для сложных данных используйте Input типы вместо множества аргументов.

```graphql
# Схема
input CreateUserInput {
  name: String!
  email: String!
  password: String!
  role: UserRole = USER
  profile: CreateProfileInput
}

input CreateProfileInput {
  bio: String
  avatar: String
  website: String
}

type Mutation {
  createUser(input: CreateUserInput!): User!
}
```

```graphql
# Использование
mutation CreateUser($input: CreateUserInput!) {
  createUser(input: $input) {
    id
    name
    email
    profile {
      bio
      avatar
    }
  }
}
```

```json
// Переменные
{
  "input": {
    "name": "John Doe",
    "email": "john@example.com",
    "password": "secret123",
    "role": "ADMIN",
    "profile": {
      "bio": "Software Developer",
      "website": "https://johndoe.dev"
    }
  }
}
```

## Паттерны ответов мутаций

### 1. Возврат изменённого объекта

```graphql
type Mutation {
  updatePost(id: ID!, input: UpdatePostInput!): Post!
}

mutation {
  updatePost(id: "1", input: { title: "New Title" }) {
    id
    title
    updatedAt
  }
}
```

### 2. Payload-паттерн (рекомендуется)

```graphql
type CreateUserPayload {
  user: User!
  success: Boolean!
  message: String
}

type UpdateUserPayload {
  user: User
  success: Boolean!
  errors: [UserError!]
}

type UserError {
  field: String!
  message: String!
  code: ErrorCode!
}

type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload!
  updateUser(id: ID!, input: UpdateUserInput!): UpdateUserPayload!
}
```

```graphql
mutation CreateUser($input: CreateUserInput!) {
  createUser(input: $input) {
    success
    message
    user {
      id
      name
      email
    }
  }
}
```

### 3. Union для успеха/ошибки

```graphql
type CreateUserSuccess {
  user: User!
}

type ValidationError {
  field: String!
  message: String!
}

type CreateUserValidationError {
  errors: [ValidationError!]!
}

union CreateUserResult = CreateUserSuccess | CreateUserValidationError

type Mutation {
  createUser(input: CreateUserInput!): CreateUserResult!
}
```

```graphql
mutation CreateUser($input: CreateUserInput!) {
  createUser(input: $input) {
    ... on CreateUserSuccess {
      user {
        id
        name
      }
    }
    ... on CreateUserValidationError {
      errors {
        field
        message
      }
    }
  }
}
```

## CRUD операции

### Create (Создание)

```graphql
# Схема
input CreatePostInput {
  title: String!
  content: String!
  tags: [String!]
  categoryId: ID!
  status: PostStatus = DRAFT
}

type Mutation {
  createPost(input: CreatePostInput!): Post!
}
```

```graphql
# Мутация
mutation CreatePost($input: CreatePostInput!) {
  createPost(input: $input) {
    id
    title
    content
    status
    author {
      name
    }
    createdAt
  }
}
```

### Update (Обновление)

```graphql
# Схема - все поля опциональные для partial update
input UpdatePostInput {
  title: String
  content: String
  tags: [String!]
  categoryId: ID
  status: PostStatus
}

type Mutation {
  updatePost(id: ID!, input: UpdatePostInput!): Post!
}
```

```graphql
# Мутация - обновляем только нужные поля
mutation UpdatePost($id: ID!, $input: UpdatePostInput!) {
  updatePost(id: $id, input: $input) {
    id
    title
    content
    updatedAt
  }
}
```

### Delete (Удаление)

```graphql
# Схема
type DeletePostPayload {
  success: Boolean!
  deletedId: ID
  message: String
}

type Mutation {
  deletePost(id: ID!): DeletePostPayload!
}
```

```graphql
# Мутация
mutation DeletePost($id: ID!) {
  deletePost(id: $id) {
    success
    deletedId
    message
  }
}
```

## Множественные мутации

В одном запросе можно выполнить несколько мутаций. Они выполняются последовательно.

```graphql
mutation BatchOperations {
  # Выполнится первой
  user1: createUser(input: { name: "User 1", email: "u1@test.com" }) {
    id
    name
  }

  # Выполнится второй
  user2: createUser(input: { name: "User 2", email: "u2@test.com" }) {
    id
    name
  }

  # Выполнится третьей
  updateSettings: updateAppSettings(theme: DARK) {
    theme
  }
}
```

### Порядок выполнения

```
┌─────────────────┐
│  createUser 1   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  createUser 2   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ updateSettings  │
└─────────────────┘
```

## Аутентификация и авторизация

### Мутации с аутентификацией

```graphql
# Схема
type AuthPayload {
  token: String!
  user: User!
  expiresAt: DateTime!
}

type Mutation {
  # Публичные мутации
  signUp(input: SignUpInput!): AuthPayload!
  signIn(email: String!, password: String!): AuthPayload!

  # Требуют аутентификации
  updateProfile(input: UpdateProfileInput!): User! @auth
  deleteAccount: Boolean! @auth

  # Требуют роли админа
  deleteUser(id: ID!): Boolean! @auth(requires: ADMIN)
}
```

```graphql
# Вход в систему
mutation SignIn($email: String!, $password: String!) {
  signIn(email: $email, password: $password) {
    token
    user {
      id
      name
      email
    }
    expiresAt
  }
}
```

### Передача токена

```javascript
const mutation = `
  mutation UpdateProfile($input: UpdateProfileInput!) {
    updateProfile(input: $input) {
      id
      name
      bio
    }
  }
`;

fetch('/graphql', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}` // Токен авторизации
  },
  body: JSON.stringify({
    query: mutation,
    variables: { input: { bio: 'New bio' } }
  })
});
```

## Валидация данных

### На уровне схемы

```graphql
# Кастомные скаляры с валидацией
scalar Email
scalar URL
scalar PositiveInt

input CreateUserInput {
  name: String!        # Обязательное поле
  email: Email!        # Валидный email
  age: PositiveInt     # Положительное число
  website: URL         # Валидный URL
}
```

### На уровне резолвера

```javascript
const resolvers = {
  Mutation: {
    createUser: async (_, { input }, context) => {
      const errors = [];

      // Валидация имени
      if (input.name.length < 2) {
        errors.push({
          field: 'name',
          message: 'Имя должно содержать минимум 2 символа'
        });
      }

      // Проверка уникальности email
      const existingUser = await context.db.findUserByEmail(input.email);
      if (existingUser) {
        errors.push({
          field: 'email',
          message: 'Пользователь с таким email уже существует'
        });
      }

      if (errors.length > 0) {
        return {
          success: false,
          errors,
          user: null
        };
      }

      const user = await context.db.createUser(input);
      return {
        success: true,
        errors: [],
        user
      };
    }
  }
};
```

## Обработка ошибок

### Типы ошибок

| Тип | Описание | Пример |
|-----|----------|--------|
| Validation Error | Ошибка валидации входных данных | Неверный формат email |
| Authentication Error | Требуется аутентификация | Невалидный токен |
| Authorization Error | Нет прав на действие | Удаление чужого поста |
| Not Found Error | Ресурс не найден | Несуществующий ID |
| Business Logic Error | Ошибка бизнес-логики | Недостаточно средств |

### Паттерн с typed errors

```graphql
interface Error {
  message: String!
}

type ValidationError implements Error {
  message: String!
  field: String!
}

type NotFoundError implements Error {
  message: String!
  resourceType: String!
  resourceId: ID!
}

type AuthorizationError implements Error {
  message: String!
  requiredRole: UserRole
}

union CreatePostResult =
  | Post
  | ValidationError
  | AuthorizationError

type Mutation {
  createPost(input: CreatePostInput!): CreatePostResult!
}
```

```graphql
mutation CreatePost($input: CreatePostInput!) {
  createPost(input: $input) {
    ... on Post {
      id
      title
    }
    ... on ValidationError {
      message
      field
    }
    ... on AuthorizationError {
      message
      requiredRole
    }
  }
}
```

## Оптимистичные обновления

Клиент может обновить UI до получения ответа сервера.

```javascript
// Apollo Client пример
const [createPost] = useMutation(CREATE_POST, {
  // Оптимистичный ответ
  optimisticResponse: {
    createPost: {
      __typename: 'Post',
      id: 'temp-id-' + Date.now(),
      title: input.title,
      content: input.content,
      createdAt: new Date().toISOString()
    }
  },

  // Обновление кеша
  update(cache, { data: { createPost } }) {
    cache.modify({
      fields: {
        posts(existingPosts = []) {
          const newPostRef = cache.writeFragment({
            data: createPost,
            fragment: gql`
              fragment NewPost on Post {
                id
                title
                content
                createdAt
              }
            `
          });
          return [newPostRef, ...existingPosts];
        }
      }
    });
  }
});
```

## Транзакции и атомарность

### Batch операции

```graphql
input BulkCreateUserInput {
  users: [CreateUserInput!]!
}

type BulkCreateUserPayload {
  users: [User!]!
  failedCount: Int!
  errors: [BulkError!]!
}

type BulkError {
  index: Int!
  errors: [UserError!]!
}

type Mutation {
  bulkCreateUsers(input: BulkCreateUserInput!): BulkCreateUserPayload!
}
```

### Транзакционная мутация

```graphql
type TransferMoneyPayload {
  success: Boolean!
  fromAccount: Account!
  toAccount: Account!
  transaction: Transaction!
}

type Mutation {
  # Атомарная операция - либо всё успешно, либо откат
  transferMoney(
    fromAccountId: ID!
    toAccountId: ID!
    amount: Float!
  ): TransferMoneyPayload!
}
```

## Практические примеры

### Регистрация пользователя

```graphql
input SignUpInput {
  email: String!
  password: String!
  confirmPassword: String!
  name: String!
  acceptTerms: Boolean!
}

type SignUpPayload {
  user: User
  token: String
  errors: [FormError!]!
}

type FormError {
  field: String!
  message: String!
}

mutation SignUp($input: SignUpInput!) {
  signUp(input: $input) {
    user {
      id
      name
      email
    }
    token
    errors {
      field
      message
    }
  }
}
```

### Обновление настроек

```graphql
input UpdateSettingsInput {
  notifications: NotificationSettingsInput
  privacy: PrivacySettingsInput
  theme: Theme
  language: String
}

input NotificationSettingsInput {
  email: Boolean
  push: Boolean
  sms: Boolean
}

mutation UpdateSettings($input: UpdateSettingsInput!) {
  updateSettings(input: $input) {
    notifications {
      email
      push
      sms
    }
    privacy {
      profileVisibility
    }
    theme
    language
  }
}
```

### Публикация поста с медиа

```graphql
input CreatePostInput {
  title: String!
  content: String!
  mediaIds: [ID!]
  tags: [String!]
  scheduledAt: DateTime
}

mutation CreatePost($input: CreatePostInput!) {
  createPost(input: $input) {
    id
    title
    content
    media {
      id
      url
      type
    }
    tags
    status
    scheduledAt
    author {
      id
      name
    }
  }
}
```

## Практические советы

### Именование мутаций

```graphql
type Mutation {
  # Используйте глаголы действия
  createUser(...)    # ✓ Хорошо
  addUser(...)       # ✓ Допустимо
  user(...)          # ✗ Плохо - не понятно действие

  # Будьте конкретны
  publishPost(...)   # ✓ Лучше
  updatePost(status: PUBLISHED)  # ✗ Менее понятно

  # Группируйте по сущностям
  createUser(...)
  updateUser(...)
  deleteUser(...)

  createPost(...)
  updatePost(...)
  publishPost(...)
  archivePost(...)
}
```

### Рекомендации

1. **Используйте Input типы** — упрощают поддержку и расширение
2. **Возвращайте Payload** — включайте информацию об успехе/ошибках
3. **Мутации должны быть идемпотентными** — повторный вызов безопасен
4. **Валидируйте на сервере** — не доверяйте клиенту
5. **Логируйте мутации** — для аудита и отладки
6. **Используйте транзакции** — для связанных изменений

## Заключение

Мутации — ключевой механизм изменения данных в GraphQL. Правильное проектирование мутаций с использованием Input типов, Payload паттернов и обработки ошибок делает API надёжным и удобным для клиентов. Следующий раздел посвящён подпискам для работы с данными в реальном времени.
