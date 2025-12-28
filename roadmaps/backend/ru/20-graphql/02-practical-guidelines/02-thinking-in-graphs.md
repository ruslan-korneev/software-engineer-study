# Мышление графами (Thinking in Graphs)

[prev: 01-introduction](./01-introduction.md) | [next: 03-serving-over-http](./03-serving-over-http.md)

---

## Что такое граф данных?

**Граф данных** — это способ моделирования информации, где данные представлены в виде узлов (nodes) и связей между ними (edges). В отличие от табличного представления в реляционных базах данных, граф естественно отражает связи между сущностями.

## От REST к графам

### REST-мышление

В REST API мы думаем ресурсами и эндпоинтами:

```
GET /users/1
GET /users/1/posts
GET /posts/5/comments
GET /users/1/followers
```

Каждый ресурс — отдельный эндпоинт. Для получения связанных данных нужно делать множество запросов.

### GraphQL-мышление

В GraphQL мы думаем графом связанных данных:

```graphql
query {
  user(id: "1") {
    name
    posts {
      title
      comments {
        text
        author {
          name
        }
      }
    }
    followers {
      name
    }
  }
}
```

Один запрос обходит граф и возвращает все нужные данные.

## Моделирование графа

### Узлы (Nodes)

Узлы — это сущности в вашей системе. В GraphQL они представлены типами:

```graphql
type User {
  id: ID!
  name: String!
  email: String!
}

type Post {
  id: ID!
  title: String!
  content: String!
}

type Comment {
  id: ID!
  text: String!
}
```

### Рёбра (Edges)

Рёбра — это связи между узлами. В GraphQL они представлены полями:

```graphql
type User {
  id: ID!
  name: String!
  posts: [Post!]!       # User -> Post (один ко многим)
  friends: [User!]!     # User -> User (многие ко многим)
}

type Post {
  id: ID!
  title: String!
  author: User!         # Post -> User (многие к одному)
  comments: [Comment!]! # Post -> Comment (один ко многим)
}

type Comment {
  id: ID!
  text: String!
  author: User!         # Comment -> User
  post: Post!           # Comment -> Post
}
```

## Типы связей

| Тип связи | Пример | GraphQL представление |
|-----------|--------|----------------------|
| Один к одному | User -> Profile | `profile: Profile!` |
| Один ко многим | User -> Posts | `posts: [Post!]!` |
| Многие к одному | Post -> Author | `author: User!` |
| Многие ко многим | Users -> Groups | `groups: [Group!]!` |

## Двунаправленные связи

В графе связи могут быть двунаправленными:

```graphql
type User {
  id: ID!
  name: String!
  posts: [Post!]!  # Получить посты пользователя
}

type Post {
  id: ID!
  title: String!
  author: User!    # Получить автора поста
}
```

Это позволяет навигировать по графу в любом направлении:

```graphql
# От пользователя к постам
query {
  user(id: "1") {
    posts {
      title
    }
  }
}

# От поста к пользователю
query {
  post(id: "5") {
    author {
      name
    }
  }
}
```

## Практические паттерны

### 1. Агрегированные корни

Определите точки входа в граф через Query:

```graphql
type Query {
  # Основные точки входа
  user(id: ID!): User
  post(id: ID!): Post

  # Коллекции
  users(first: Int, after: String): UserConnection!
  posts(first: Int, after: String): PostConnection!

  # Поиск
  search(query: String!): [SearchResult!]!

  # Текущий контекст
  me: User
}
```

### 2. Циклические связи

Графы могут содержать циклы. GraphQL поддерживает это:

```graphql
type User {
  id: ID!
  friends: [User!]!  # User ссылается на User
}

type Comment {
  id: ID!
  replies: [Comment!]!  # Comment ссылается на Comment
}
```

### 3. Полиморфные связи

Используйте интерфейсы и union для полиморфизма:

```graphql
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

union SearchResult = User | Post | Comment

type Query {
  node(id: ID!): Node
  search(query: String!): [SearchResult!]!
}
```

## Проектирование схемы как графа

### Шаг 1: Определите сущности

```
Пользователь, Пост, Комментарий, Категория, Тег
```

### Шаг 2: Определите связи

```
Пользователь --[пишет]--> Пост
Пост --[принадлежит]--> Категория
Пост --[имеет]--> Теги
Пользователь --[комментирует]--> Пост
Пользователь --[подписан на]--> Пользователя
```

### Шаг 3: Преобразуйте в схему

```graphql
type User {
  id: ID!
  name: String!
  posts: [Post!]!
  comments: [Comment!]!
  following: [User!]!
  followers: [User!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  category: Category!
  tags: [Tag!]!
  comments: [Comment!]!
}

type Category {
  id: ID!
  name: String!
  posts: [Post!]!
}

type Tag {
  id: ID!
  name: String!
  posts: [Post!]!
}

type Comment {
  id: ID!
  text: String!
  author: User!
  post: Post!
}
```

## Реализация резолверов для графа

```javascript
const resolvers = {
  Query: {
    user: (_, { id }) => db.users.findById(id),
    post: (_, { id }) => db.posts.findById(id),
  },

  User: {
    // Связь User -> Posts
    posts: (user) => db.posts.findByAuthorId(user.id),

    // Связь User -> Comments
    comments: (user) => db.comments.findByAuthorId(user.id),

    // Связь User -> Users (followers)
    followers: (user) => db.users.findFollowersOf(user.id),
  },

  Post: {
    // Связь Post -> User
    author: (post) => db.users.findById(post.authorId),

    // Связь Post -> Category
    category: (post) => db.categories.findById(post.categoryId),

    // Связь Post -> Tags
    tags: (post) => db.tags.findByPostId(post.id),

    // Связь Post -> Comments
    comments: (post) => db.comments.findByPostId(post.id),
  },

  Comment: {
    author: (comment) => db.users.findById(comment.authorId),
    post: (comment) => db.posts.findById(comment.postId),
  },
};
```

## Визуализация графа

Для понимания структуры данных полезно визуализировать граф:

```
        ┌─────────┐
        │  User   │
        └────┬────┘
             │
    ┌────────┼────────┐
    ▼        ▼        ▼
┌───────┐ ┌───────┐ ┌──────────┐
│ Post  │ │Comment│ │ Follower │
└───┬───┘ └───────┘ └──────────┘
    │
┌───┴───┐
│       │
▼       ▼
Category Tags
```

## Антипаттерны

### 1. Плоская структура

```graphql
# Плохо: нет связей между типами
type Query {
  user(id: ID!): User
  userPosts(userId: ID!): [Post]
  postComments(postId: ID!): [Comment]
}
```

### 2. Слишком глубокая вложенность

```graphql
# Потенциально опасно: неограниченная глубина
type Comment {
  replies: [Comment!]!  # Может уходить в бесконечность
}
```

Решение — лимитировать глубину:

```graphql
type Comment {
  replies(depth: Int = 1, first: Int = 10): [Comment!]!
}
```

## Практический совет

> Рисуйте граф данных перед написанием схемы. Это поможет увидеть все связи и избежать пропущенных отношений.

## Заключение

Мышление графами — фундаментальный навык для работы с GraphQL. Представляйте данные как связную сеть объектов, а не как набор изолированных ресурсов. Это позволит создавать более гибкие и удобные API.

---

[prev: 01-introduction](./01-introduction.md) | [next: 03-serving-over-http](./03-serving-over-http.md)
