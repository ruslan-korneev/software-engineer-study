# Проектирование схемы GraphQL

## Введение

**Схема GraphQL** — это контракт между клиентом и сервером. Правильное проектирование схемы определяет удобство использования API, возможности эволюции и производительность. Ошибки в дизайне схемы дорого исправлять после запуска.

## Основные принципы

### 1. Клиент-ориентированность

Схема должна отражать потребности клиентов, а не структуру базы данных:

```graphql
# Плохо: отражает структуру БД
type User {
  id: ID!
  first_name: String
  last_name: String
  avatar_url: String
}

# Хорошо: удобно для клиента
type User {
  id: ID!
  fullName: String!
  avatar: Image
}

type Image {
  url(size: ImageSize): String!
  alt: String
}
```

### 2. Именование

| Элемент | Конвенция | Пример |
|---------|-----------|--------|
| Типы | PascalCase | `User`, `BlogPost` |
| Поля | camelCase | `firstName`, `createdAt` |
| Аргументы | camelCase | `userId`, `first` |
| Enum значения | SCREAMING_SNAKE | `PUBLISHED`, `IN_PROGRESS` |
| Input типы | PascalCase + Input | `CreateUserInput` |

```graphql
type BlogPost {
  id: ID!
  title: String!
  status: PostStatus!
  author: User!
}

enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}

input CreatePostInput {
  title: String!
  content: String!
}
```

### 3. Nullability

Используйте non-null (`!`) осознанно:

```graphql
type User {
  id: ID!           # Всегда есть
  email: String!    # Обязательно при регистрации
  bio: String       # Может быть null
  avatar: Image     # Может быть null
  posts: [Post!]!   # Список всегда есть, элементы не null
}

# Для списков
posts: [Post!]!   # Список не null, элементы не null
posts: [Post!]    # Список может быть null, элементы не null
posts: [Post]!    # Список не null, элементы могут быть null
posts: [Post]     # Всё может быть null
```

## Проектирование типов

### Доменные типы

```graphql
# Богатые доменные типы вместо примитивов
type Money {
  amount: Float!
  currency: Currency!
  formatted: String!  # "$19.99"
}

type DateRange {
  start: DateTime!
  end: DateTime!
  durationDays: Int!
}

type Address {
  street: String!
  city: String!
  country: Country!
  postalCode: String!
  formatted: String!  # Полный адрес одной строкой
}
```

### Полиморфные типы

```graphql
# Интерфейсы для общего поведения
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
}

# Union для разнородных типов
union SearchResult = User | Post | Comment

type Query {
  search(query: String!): [SearchResult!]!
}
```

### Вложенные объекты vs плоская структура

```graphql
# Плохо: слишком плоско
type User {
  id: ID!
  name: String!
  addressStreet: String
  addressCity: String
  addressCountry: String
}

# Хорошо: логическая группировка
type User {
  id: ID!
  name: String!
  address: Address
  contactInfo: ContactInfo
}

# Плохо: слишком вложено
type User {
  profile: {
    personal: {
      name: {
        first: String
        last: String
      }
    }
  }
}

# Хорошо: баланс
type User {
  id: ID!
  fullName: String!
  firstName: String!
  lastName: String!
}
```

## Проектирование Query

### Точки входа

```graphql
type Query {
  # Получение по ID
  user(id: ID!): User
  post(id: ID!): Post

  # Глобальный поиск (Relay Node Interface)
  node(id: ID!): Node

  # Коллекции с пагинацией
  users(first: Int, after: String, filter: UserFilter): UserConnection!
  posts(first: Int, after: String, filter: PostFilter): PostConnection!

  # Текущий контекст
  me: User
  viewer: Viewer

  # Поиск
  search(query: String!, types: [SearchType!]): SearchConnection!
}
```

### Viewer паттерн

```graphql
# Все данные пользователя через Viewer
type Query {
  viewer: Viewer
}

type Viewer {
  user: User!
  notifications(first: Int): NotificationConnection!
  feed(first: Int): FeedConnection!
  settings: Settings!
}
```

## Проектирование Mutation

### Структура мутации

```graphql
# Input тип для аргументов
input CreatePostInput {
  title: String!
  content: String!
  categoryId: ID
  tags: [String!]
}

# Payload тип для ответа
type CreatePostPayload {
  post: Post
  errors: [UserError!]!
}

type UserError {
  field: String
  message: String!
  code: ErrorCode!
}

type Mutation {
  createPost(input: CreatePostInput!): CreatePostPayload!
}
```

### Именование мутаций

```graphql
type Mutation {
  # Глагол + существительное
  createUser(input: CreateUserInput!): CreateUserPayload!
  updateUser(input: UpdateUserInput!): UpdateUserPayload!
  deleteUser(id: ID!): DeleteUserPayload!

  # Для бизнес-операций
  publishPost(id: ID!): PublishPostPayload!
  archivePost(id: ID!): ArchivePostPayload!
  submitOrder(input: SubmitOrderInput!): SubmitOrderPayload!
}
```

### Возврат затронутых данных

```graphql
type CreatePostPayload {
  post: Post                    # Созданный пост
  author: User                  # Автор (мог измениться postCount)
  errors: [UserError!]!
}

type DeleteCommentPayload {
  deletedCommentId: ID         # ID удалённого комментария
  post: Post                   # Пост (изменился commentCount)
  errors: [UserError!]!
}
```

## Input типы

### Создание vs Обновление

```graphql
# Все поля обязательны при создании
input CreateUserInput {
  email: String!
  name: String!
  password: String!
}

# Все поля опциональны при обновлении
input UpdateUserInput {
  name: String
  bio: String
  avatar: Upload
}

# Partial updates
type Mutation {
  updateUser(id: ID!, input: UpdateUserInput!): UpdateUserPayload!
}
```

### Вложенные Input типы

```graphql
input CreateOrderInput {
  items: [OrderItemInput!]!
  shippingAddress: AddressInput!
  billingAddress: AddressInput
  paymentMethod: PaymentMethodInput!
}

input OrderItemInput {
  productId: ID!
  quantity: Int!
}

input AddressInput {
  street: String!
  city: String!
  postalCode: String!
  country: String!
}
```

## Обработка ошибок

### User Errors vs System Errors

```graphql
type CreateUserPayload {
  user: User
  errors: [UserError!]!  # Ошибки валидации, бизнес-логики
}

type UserError {
  field: String          # Поле с ошибкой
  message: String!       # Человекочитаемое сообщение
  code: ErrorCode!       # Машиночитаемый код
}

enum ErrorCode {
  INVALID_INPUT
  NOT_FOUND
  ALREADY_EXISTS
  PERMISSION_DENIED
  RATE_LIMITED
}
```

### Реализация

```javascript
const resolvers = {
  Mutation: {
    createUser: async (_, { input }) => {
      const errors = [];

      // Валидация
      if (!isValidEmail(input.email)) {
        errors.push({
          field: 'email',
          message: 'Invalid email format',
          code: 'INVALID_INPUT',
        });
      }

      // Проверка уникальности
      const existing = await db.users.findByEmail(input.email);
      if (existing) {
        errors.push({
          field: 'email',
          message: 'Email already registered',
          code: 'ALREADY_EXISTS',
        });
      }

      if (errors.length > 0) {
        return { user: null, errors };
      }

      const user = await db.users.create(input);
      return { user, errors: [] };
    },
  },
};
```

## Эволюция схемы

### Добавление полей (безопасно)

```graphql
type User {
  id: ID!
  name: String!
  # Новое поле - не ломает клиентов
  avatar: Image
}
```

### Deprecation

```graphql
type User {
  id: ID!
  name: String!

  # Помечаем как устаревшее
  fullName: String @deprecated(reason: "Use `name` instead")

  # Устаревшие аргументы
  posts(
    first: Int
    limit: Int @deprecated(reason: "Use `first` instead")
  ): [Post!]!
}
```

### Breaking Changes (избегать!)

```graphql
# Нельзя: удаление полей
# Нельзя: изменение типа поля
# Нельзя: добавление обязательного аргумента
# Нельзя: удаление значения enum
```

## Документирование схемы

```graphql
"""
Пользователь системы.
Содержит персональные данные и связи с другими сущностями.
"""
type User {
  "Уникальный идентификатор пользователя"
  id: ID!

  "Полное имя пользователя для отображения"
  name: String!

  """
  Email пользователя.
  Используется для входа и уведомлений.
  Виден только самому пользователю и администраторам.
  """
  email: String!

  "Публикации пользователя с пагинацией"
  posts(
    "Количество записей"
    first: Int = 10
    "Курсор для пагинации"
    after: String
  ): PostConnection!
}
```

## Антипаттерны

### 1. CRUD-схема

```graphql
# Плохо: примитивный CRUD
type Mutation {
  createPost(title: String!, content: String!): Post
  updatePost(id: ID!, title: String, content: String): Post
  deletePost(id: ID!): Boolean
}

# Хорошо: бизнес-операции
type Mutation {
  draftPost(input: DraftPostInput!): DraftPostPayload!
  publishPost(id: ID!): PublishPostPayload!
  archivePost(id: ID!): ArchivePostPayload!
}
```

### 2. Anemic Types

```graphql
# Плохо: просто контейнер данных
type Order {
  id: ID!
  status: String!
  total: Float!
}

# Хорошо: богатый тип с поведением
type Order {
  id: ID!
  status: OrderStatus!
  total: Money!
  canBeCancelled: Boolean!
  estimatedDelivery: DateRange
  trackingUrl: String
}
```

### 3. God Object

```graphql
# Плохо: слишком много в одном типе
type User {
  id: ID!
  # ... 50 полей ...
  allOrders: [Order!]!
  allPosts: [Post!]!
  allComments: [Comment!]!
  allNotifications: [Notification!]!
}

# Хорошо: разделение на подтипы
type User {
  id: ID!
  profile: UserProfile!
  activity: UserActivity!
}
```

## Чек-лист проектирования

- [ ] Именование следует конвенциям
- [ ] Nullability определена осознанно
- [ ] Input и Payload типы для мутаций
- [ ] Пагинация для коллекций
- [ ] Документация для всех типов и полей
- [ ] Нет breaking changes для существующих клиентов
- [ ] Схема прошла ревью

## Заключение

Хорошо спроектированная схема — основа успешного GraphQL API. Думайте о потребностях клиентов, используйте богатые типы, правильно обрабатывайте ошибки и планируйте эволюцию схемы заранее.
