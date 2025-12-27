# Введение в GraphQL

## Что такое GraphQL?

**GraphQL** — это язык запросов для API и среда выполнения для этих запросов, разработанная Facebook в 2012 году и открытая для публичного использования в 2015 году. GraphQL предоставляет полное и понятное описание данных в API, даёт клиентам возможность запрашивать именно те данные, которые им нужны, и ничего лишнего.

## История создания

| Год | Событие |
|-----|---------|
| 2012 | Facebook начинает разработку GraphQL для внутренних нужд |
| 2015 | GraphQL становится open-source проектом |
| 2018 | Создание GraphQL Foundation под управлением Linux Foundation |
| 2021+ | Широкое распространение в индустрии |

## GraphQL vs REST

### Сравнительная таблица

| Характеристика | REST | GraphQL |
|----------------|------|---------|
| Количество endpoint-ов | Множество | Один |
| Формат запроса | HTTP методы (GET, POST, PUT, DELETE) | POST с телом запроса |
| Over-fetching | Частая проблема | Отсутствует |
| Under-fetching | Частая проблема | Отсутствует |
| Версионирование | Требуется (v1, v2) | Не требуется |
| Типизация | Опциональная | Строгая |
| Документация | Внешняя (Swagger, OpenAPI) | Встроенная (интроспекция) |

### Проблемы REST, которые решает GraphQL

#### Over-fetching (избыточная загрузка)
REST API часто возвращает больше данных, чем необходимо клиенту.

```
# REST: получаем все поля пользователя, хотя нужно только имя
GET /users/1
{
  "id": 1,
  "name": "John",
  "email": "john@example.com",
  "address": { ... },
  "orders": [ ... ],
  "preferences": { ... }
}
```

```graphql
# GraphQL: запрашиваем только нужные поля
query {
  user(id: 1) {
    name
  }
}
# Ответ: { "data": { "user": { "name": "John" } } }
```

#### Under-fetching (недостаточная загрузка)
В REST для получения связанных данных часто требуется несколько запросов.

```
# REST: нужно 3 запроса
GET /users/1
GET /users/1/posts
GET /users/1/followers
```

```graphql
# GraphQL: один запрос
query {
  user(id: 1) {
    name
    posts {
      title
    }
    followers {
      name
    }
  }
}
```

## Основные концепции GraphQL

### 1. Схема (Schema)
Схема определяет структуру данных и доступные операции.

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
}

type Query {
  user(id: ID!): User
  users: [User!]!
}
```

### 2. Запросы (Queries)
Запросы используются для чтения данных.

```graphql
query GetUser {
  user(id: "1") {
    name
    email
  }
}
```

### 3. Мутации (Mutations)
Мутации используются для изменения данных.

```graphql
mutation CreateUser {
  createUser(input: { name: "John", email: "john@example.com" }) {
    id
    name
  }
}
```

### 4. Подписки (Subscriptions)
Подписки позволяют получать данные в реальном времени.

```graphql
subscription OnNewPost {
  postCreated {
    id
    title
    author {
      name
    }
  }
}
```

## Преимущества GraphQL

### Для клиентов
- **Гибкость запросов** — клиент определяет структуру ответа
- **Один endpoint** — упрощает работу с API
- **Строгая типизация** — ошибки обнаруживаются на этапе разработки
- **Самодокументирование** — схема служит документацией

### Для серверов
- **Эволюция без версий** — добавление новых полей без breaking changes
- **Агрегация данных** — объединение нескольких источников данных
- **Мониторинг использования** — понимание какие поля используются клиентами

## Недостатки GraphQL

| Недостаток | Описание |
|------------|----------|
| Сложность кеширования | HTTP кеширование не работает "из коробки" |
| Overhead на сервере | Сложные запросы могут создавать нагрузку |
| Кривая обучения | Требует времени на освоение |
| File upload | Нет нативной поддержки загрузки файлов |
| Rate limiting | Сложнее контролировать нагрузку |

## Когда использовать GraphQL

### GraphQL подходит, когда:
- Приложение имеет сложную структуру данных
- Множество клиентов с разными требованиями к данным
- Необходима гибкость в запросах
- Важна производительность на мобильных устройствах
- Команда готова инвестировать в изучение

### REST предпочтительнее, когда:
- Простое CRUD приложение
- Необходимо HTTP кеширование
- Команда имеет опыт только с REST
- Публичное API с простой структурой

## Экосистема GraphQL

### Серверные библиотеки

```javascript
// Apollo Server (Node.js)
const { ApolloServer, gql } = require('apollo-server');

const typeDefs = gql`
  type Query {
    hello: String
  }
`;

const resolvers = {
  Query: {
    hello: () => 'Hello, World!'
  }
};

const server = new ApolloServer({ typeDefs, resolvers });
server.listen().then(({ url }) => {
  console.log(`Server ready at ${url}`);
});
```

```python
# Strawberry (Python)
import strawberry

@strawberry.type
class Query:
    @strawberry.field
    def hello(self) -> str:
        return "Hello, World!"

schema = strawberry.Schema(query=Query)
```

### Клиентские библиотеки
- **Apollo Client** — полнофункциональный клиент для JavaScript/TypeScript
- **Relay** — клиент от Facebook для React
- **urql** — легковесная альтернатива Apollo
- **graphql-request** — минимальный клиент для простых запросов

### Инструменты разработки
- **GraphQL Playground** — интерактивная среда для запросов
- **GraphiQL** — официальный IDE для GraphQL
- **Apollo Studio** — платформа для управления GraphQL API

## Первый запрос GraphQL

### Пример с использованием fetch

```javascript
const query = `
  query GetUser($id: ID!) {
    user(id: $id) {
      name
      email
    }
  }
`;

const variables = { id: "1" };

fetch('https://api.example.com/graphql', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ query, variables }),
})
  .then(res => res.json())
  .then(data => console.log(data));
```

### Ответ сервера

```json
{
  "data": {
    "user": {
      "name": "John Doe",
      "email": "john@example.com"
    }
  }
}
```

## Практические советы

1. **Начните с малого** — не пытайтесь перевести весь API сразу
2. **Используйте инструменты** — GraphQL Playground упрощает отладку
3. **Думайте о графах** — моделируйте данные как связанные сущности
4. **Изучите паттерны** — Relay Cursor Connections для пагинации
5. **Мониторьте запросы** — отслеживайте сложные и медленные запросы

## Заключение

GraphQL — мощный инструмент для построения API, который решает многие проблемы REST. Однако он не является универсальным решением и требует осознанного выбора. В следующих разделах мы подробно рассмотрим схемы, типы, запросы и другие аспекты GraphQL.
