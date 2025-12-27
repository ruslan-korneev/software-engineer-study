# Запросы GraphQL (Queries)

## Что такое Query?

**Query** — это операция чтения данных в GraphQL. Запросы позволяют клиенту точно указать, какие данные ему нужны, и получить их в предсказуемой структуре. В отличие от REST, где структура ответа определяется сервером, в GraphQL клиент полностью контролирует форму ответа.

## Базовая структура запроса

```graphql
# Анонимный запрос (сокращённая форма)
{
  users {
    id
    name
  }
}

# Именованный запрос (рекомендуется)
query GetUsers {
  users {
    id
    name
  }
}

# Запрос с переменными
query GetUser($id: ID!) {
  user(id: $id) {
    id
    name
    email
  }
}
```

## Поля (Fields)

Поля — основной строительный блок запросов. Каждое поле соответствует полю в схеме.

```graphql
query {
  # Скалярные поля
  user(id: "1") {
    id        # ID!
    name      # String!
    email     # String!
    age       # Int
    isActive  # Boolean!
  }
}
```

### Вложенные поля

```graphql
query {
  user(id: "1") {
    name
    # Вложенный объект
    profile {
      avatar
      bio
      website
    }
    # Связанные сущности
    posts {
      title
      comments {
        text
        author {
          name
        }
      }
    }
  }
}
```

### Ответ сервера

Структура ответа точно соответствует структуре запроса:

```json
{
  "data": {
    "user": {
      "name": "John Doe",
      "profile": {
        "avatar": "https://example.com/avatar.jpg",
        "bio": "Software Developer",
        "website": "https://johndoe.dev"
      },
      "posts": [
        {
          "title": "Introduction to GraphQL",
          "comments": [
            {
              "text": "Great article!",
              "author": {
                "name": "Jane Smith"
              }
            }
          ]
        }
      ]
    }
  }
}
```

## Аргументы (Arguments)

Аргументы позволяют фильтровать, сортировать и настраивать запросы.

```graphql
query {
  # Получение одной записи по ID
  user(id: "123") {
    name
  }

  # Фильтрация списка
  users(role: ADMIN, isActive: true) {
    name
    email
  }

  # Пагинация
  posts(limit: 10, offset: 20) {
    title
  }

  # Сортировка
  articles(sortBy: "createdAt", order: DESC) {
    title
    createdAt
  }

  # Поиск
  search(query: "GraphQL", category: TECH) {
    title
    url
  }
}
```

### Аргументы на вложенных полях

```graphql
query {
  user(id: "1") {
    name
    # Аргументы на связанных полях
    posts(status: PUBLISHED, first: 5) {
      title
    }
    # Форматирование на уровне поля
    createdAt(format: "DD.MM.YYYY")
    # Локализация
    biography(locale: "ru")
  }
}
```

## Переменные (Variables)

Переменные позволяют динамически передавать значения в запрос.

### Объявление и использование

```graphql
# Определение переменных в сигнатуре запроса
query GetUser($userId: ID!, $includeEmail: Boolean = false) {
  user(id: $userId) {
    id
    name
    email @include(if: $includeEmail)
  }
}
```

### Передача переменных

```json
{
  "userId": "123",
  "includeEmail": true
}
```

### Типы переменных

```graphql
query Example(
  $id: ID!                    # Обязательный ID
  $name: String               # Необязательная строка
  $limit: Int = 10            # Число со значением по умолчанию
  $tags: [String!]            # Список строк
  $filter: UserFilterInput!   # Input тип
  $status: PostStatus         # Enum
) {
  # ...
}
```

### Пример с JavaScript

```javascript
const GET_USER = `
  query GetUser($id: ID!) {
    user(id: $id) {
      id
      name
      email
    }
  }
`;

async function fetchUser(id) {
  const response = await fetch('/graphql', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      query: GET_USER,
      variables: { id }
    })
  });

  const { data, errors } = await response.json();

  if (errors) {
    throw new Error(errors[0].message);
  }

  return data.user;
}
```

## Псевдонимы (Aliases)

Псевдонимы позволяют переименовывать поля в ответе и запрашивать одно поле несколько раз с разными аргументами.

```graphql
query {
  # Переименование поля
  currentUser: user(id: "1") {
    fullName: name
    emailAddress: email
  }

  # Множественные запросы одного поля
  adminUsers: users(role: ADMIN) {
    name
  }
  regularUsers: users(role: USER) {
    name
  }

  # Сравнение данных
  firstPost: post(id: "1") {
    title
    viewCount
  }
  latestPost: post(id: "999") {
    title
    viewCount
  }
}
```

### Ответ с псевдонимами

```json
{
  "data": {
    "currentUser": {
      "fullName": "John Doe",
      "emailAddress": "john@example.com"
    },
    "adminUsers": [
      { "name": "Admin One" }
    ],
    "regularUsers": [
      { "name": "User One" },
      { "name": "User Two" }
    ],
    "firstPost": {
      "title": "First Post",
      "viewCount": 1500
    },
    "latestPost": {
      "title": "Latest Post",
      "viewCount": 42
    }
  }
}
```

## Фрагменты (Fragments)

Фрагменты — переиспользуемые наборы полей.

### Именованные фрагменты

```graphql
# Определение фрагмента
fragment UserBasicInfo on User {
  id
  name
  email
  avatar
}

fragment PostDetails on Post {
  id
  title
  content
  createdAt
  author {
    ...UserBasicInfo
  }
}

# Использование фрагментов
query {
  me {
    ...UserBasicInfo
    posts {
      ...PostDetails
    }
  }

  user(id: "2") {
    ...UserBasicInfo
  }
}
```

### Inline фрагменты

Используются для запроса полей конкретных типов в union или interface.

```graphql
query SearchQuery($query: String!) {
  search(query: $query) {
    # Общие поля интерфейса
    ... on Node {
      id
    }

    # Поля конкретных типов
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

### Фрагменты с переменными (условные поля)

```graphql
fragment UserFields on User {
  id
  name
  email @include(if: $showEmail)
  phone @skip(if: $hidePhone)
}

query GetUsers($showEmail: Boolean!, $hidePhone: Boolean!) {
  users {
    ...UserFields
  }
}
```

## Директивы

Директивы модифицируют выполнение запроса.

### @include и @skip

```graphql
query GetUser($id: ID!, $withPosts: Boolean!, $withoutComments: Boolean!) {
  user(id: $id) {
    name
    email

    # Включить поле, если withPosts = true
    posts @include(if: $withPosts) {
      title

      # Пропустить поле, если withoutComments = true
      comments @skip(if: $withoutComments) {
        text
      }
    }
  }
}
```

### Таблица директив

| Директива | Действие | Когда применяется |
|-----------|----------|-------------------|
| `@include(if: Boolean!)` | Включает поле | Если условие `true` |
| `@skip(if: Boolean!)` | Пропускает поле | Если условие `true` |
| `@deprecated(reason: String)` | Помечает устаревшим | В схеме |

## Операция по умолчанию

Если в документе только один запрос, ключевое слово `query` можно опустить:

```graphql
# Полная форма
query {
  users {
    name
  }
}

# Сокращённая форма
{
  users {
    name
  }
}
```

## Множественные операции

В одном документе может быть несколько операций, но выполнить можно только одну:

```graphql
query GetUsers {
  users {
    name
  }
}

query GetPosts {
  posts {
    title
  }
}

mutation CreateUser {
  createUser(name: "John") {
    id
  }
}
```

При выполнении нужно указать имя операции:

```json
{
  "query": "query GetUsers { ... } query GetPosts { ... }",
  "operationName": "GetUsers"
}
```

## Паттерны построения запросов

### Relay-style Pagination (Cursor-based)

```graphql
query GetPosts($first: Int!, $after: String) {
  posts(first: $first, after: $after) {
    edges {
      cursor
      node {
        id
        title
        content
      }
    }
    pageInfo {
      hasNextPage
      hasPreviousPage
      startCursor
      endCursor
    }
    totalCount
  }
}
```

### Offset-based Pagination

```graphql
query GetPosts($limit: Int!, $offset: Int!) {
  posts(limit: $limit, offset: $offset) {
    items {
      id
      title
    }
    total
    hasMore
  }
}
```

### Поиск с фильтрами

```graphql
query SearchPosts($input: SearchInput!) {
  searchPosts(input: $input) {
    results {
      id
      title
      highlights
      score
    }
    facets {
      category {
        value
        count
      }
      author {
        value
        count
      }
    }
    totalResults
  }
}
```

## Практические примеры

### Загрузка профиля пользователя

```graphql
query UserProfile($userId: ID!) {
  user(id: $userId) {
    id
    name
    email
    avatar
    bio
    createdAt

    stats {
      postsCount
      followersCount
      followingCount
    }

    recentPosts: posts(first: 5, orderBy: CREATED_AT_DESC) {
      id
      title
      excerpt
      createdAt
    }
  }
}
```

### Лента новостей

```graphql
query Feed($cursor: String) {
  feed(first: 20, after: $cursor) {
    edges {
      node {
        id
        ... on Post {
          title
          content
          author {
            name
            avatar
          }
        }
        ... on Share {
          comment
          originalPost {
            title
          }
        }
      }
    }
    pageInfo {
      endCursor
      hasNextPage
    }
  }
}
```

### Дашборд с агрегированными данными

```graphql
query Dashboard($period: Period!) {
  analytics(period: $period) {
    visitors {
      total
      unique
      trend
    }
    revenue {
      total
      average
      byCategory {
        category
        amount
      }
    }
    topPosts {
      title
      views
      shares
    }
  }

  recentActivities(limit: 10) {
    type
    description
    timestamp
    user {
      name
      avatar
    }
  }
}
```

## Оптимизация запросов

### Запрашивайте только нужные поля

```graphql
# Плохо: запрашиваем всё
query {
  users {
    id
    name
    email
    avatar
    bio
    createdAt
    updatedAt
    settings { ... }
    posts { ... }
    followers { ... }
  }
}

# Хорошо: только нужное
query {
  users {
    id
    name
    avatar
  }
}
```

### Используйте фрагменты для переиспользования

```graphql
# Определите фрагменты для частых паттернов
fragment UserCard on User {
  id
  name
  avatar
}

fragment PostPreview on Post {
  id
  title
  excerpt
  author {
    ...UserCard
  }
}
```

### Избегайте глубокой вложенности

```graphql
# Избегайте слишком глубоких запросов
query DeepQuery {
  user(id: "1") {
    posts {
      comments {
        author {
          posts {
            comments {
              # Слишком глубоко!
            }
          }
        }
      }
    }
  }
}
```

## Практические советы

1. **Именуйте запросы** — это помогает при отладке и логировании
2. **Используйте переменные** — не встраивайте значения в запрос
3. **Применяйте фрагменты** — для переиспользования и читаемости
4. **Минимизируйте вложенность** — глубокие запросы медленнее
5. **Используйте псевдонимы** — для множественных запросов одного поля

## Заключение

Запросы — это основа работы с GraphQL. Правильное построение запросов позволяет эффективно получать данные, минимизируя нагрузку на сеть и сервер. Освоив базовые концепции (поля, аргументы, переменные, фрагменты), вы сможете строить сложные и эффективные запросы для любых задач.
