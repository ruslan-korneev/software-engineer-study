# Федерация GraphQL

## Введение

**Apollo Federation** — это архитектура для построения распределённого GraphQL API из нескольких независимых сервисов (subgraphs). Каждый сервис отвечает за свою часть схемы, а Gateway объединяет их в единый граф.

## Зачем нужна федерация?

| Проблема монолита | Решение с федерацией |
|-------------------|----------------------|
| Большая кодовая база | Независимые микросервисы |
| Сложность деплоя | Независимый деплой каждого сервиса |
| Bottleneck команд | Разные команды владеют разными частями |
| Масштабирование | Масштабирование отдельных сервисов |

## Архитектура федерации

```
┌──────────────────────────────────────────────────┐
│                    Клиенты                        │
└──────────────────────────────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────┐
│              Apollo Gateway/Router               │
│         (Объединение схем, маршрутизация)        │
└──────────────────────────────────────────────────┘
          │                  │                  │
          ▼                  ▼                  ▼
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│   Users        │  │   Products     │  │   Reviews      │
│   Subgraph     │  │   Subgraph     │  │   Subgraph     │
└────────────────┘  └────────────────┘  └────────────────┘
```

## Основные директивы Federation 2

```graphql
# Помечает тип как сущность (entity)
extend schema @link(url: "https://specs.apollo.dev/federation/v2.0",
  import: ["@key", "@shareable", "@external", "@requires", "@provides"])

# @key - определяет первичный ключ сущности
type User @key(fields: "id") {
  id: ID!
  name: String!
}

# @shareable - поле может быть определено в нескольких subgraphs
type Product @key(fields: "id") {
  id: ID!
  name: String! @shareable
  price: Float! @shareable
}

# @external - поле определено в другом subgraph
type Product @key(fields: "id") {
  id: ID! @external
  reviews: [Review!]!
}

# @requires - поле требует данных из другого subgraph
type Product @key(fields: "id") {
  id: ID! @external
  weight: Float! @external
  shippingCost: Float! @requires(fields: "weight")
}

# @provides - поле предоставляет данные для других subgraphs
type Review @key(fields: "id") {
  id: ID!
  author: User! @provides(fields: "name")
}
```

## Создание Subgraph

### Users Subgraph

```javascript
// users/schema.graphql
import { gql } from '@apollo/server';

const typeDefs = gql`
  extend schema @link(url: "https://specs.apollo.dev/federation/v2.0",
    import: ["@key", "@shareable"])

  type Query {
    me: User
    user(id: ID!): User
  }

  type Mutation {
    createUser(input: CreateUserInput!): User!
  }

  type User @key(fields: "id") {
    id: ID!
    name: String!
    email: String!
    createdAt: DateTime!
  }

  input CreateUserInput {
    name: String!
    email: String!
  }
`;

const resolvers = {
  Query: {
    me: (_, __, { user }) => user,
    user: (_, { id }) => db.users.findById(id),
  },

  User: {
    // Reference resolver для федерации
    __resolveReference(reference) {
      return db.users.findById(reference.id);
    },
  },
};

// Настройка сервера
import { ApolloServer } from '@apollo/server';
import { buildSubgraphSchema } from '@apollo/subgraph';

const server = new ApolloServer({
  schema: buildSubgraphSchema({ typeDefs, resolvers }),
});
```

### Products Subgraph

```javascript
// products/schema.graphql
const typeDefs = gql`
  extend schema @link(url: "https://specs.apollo.dev/federation/v2.0",
    import: ["@key", "@external", "@requires"])

  type Query {
    products(first: Int, after: String): ProductConnection!
    product(id: ID!): Product
  }

  type Product @key(fields: "id") {
    id: ID!
    name: String!
    price: Float!
    weight: Float!
    inStock: Boolean!
  }

  # Расширяем User из другого subgraph
  type User @key(fields: "id") {
    id: ID!
    purchasedProducts: [Product!]!
  }
`;

const resolvers = {
  Query: {
    products: (_, args) => getPaginatedProducts(args),
    product: (_, { id }) => db.products.findById(id),
  },

  Product: {
    __resolveReference(reference) {
      return db.products.findById(reference.id);
    },
  },

  User: {
    // Добавляем поле к User
    purchasedProducts: (user) => {
      return db.orders
        .findByUserId(user.id)
        .flatMap(order => order.products);
    },
  },
};
```

### Reviews Subgraph

```javascript
// reviews/schema.graphql
const typeDefs = gql`
  extend schema @link(url: "https://specs.apollo.dev/federation/v2.0",
    import: ["@key", "@external", "@provides"])

  type Query {
    reviews(productId: ID!): [Review!]!
  }

  type Review @key(fields: "id") {
    id: ID!
    rating: Int!
    text: String!
    author: User! @provides(fields: "name")
    product: Product!
  }

  # Расширяем типы из других subgraphs
  type User @key(fields: "id") {
    id: ID!
    name: String! @external
    reviews: [Review!]!
  }

  type Product @key(fields: "id") {
    id: ID!
    reviews: [Review!]!
    averageRating: Float
  }
`;

const resolvers = {
  Query: {
    reviews: (_, { productId }) => db.reviews.findByProductId(productId),
  },

  Review: {
    __resolveReference(ref) {
      return db.reviews.findById(ref.id);
    },
    author: (review) => ({ id: review.authorId }),
    product: (review) => ({ id: review.productId }),
  },

  User: {
    reviews: (user) => db.reviews.findByAuthorId(user.id),
  },

  Product: {
    reviews: (product) => db.reviews.findByProductId(product.id),
    averageRating: async (product) => {
      const reviews = await db.reviews.findByProductId(product.id);
      if (reviews.length === 0) return null;
      return reviews.reduce((sum, r) => sum + r.rating, 0) / reviews.length;
    },
  },
};
```

## Apollo Gateway / Router

### Gateway (Node.js)

```javascript
import { ApolloServer } from '@apollo/server';
import { ApolloGateway, IntrospectAndCompose } from '@apollo/gateway';

const gateway = new ApolloGateway({
  supergraphSdl: new IntrospectAndCompose({
    subgraphs: [
      { name: 'users', url: 'http://users:4001/graphql' },
      { name: 'products', url: 'http://products:4002/graphql' },
      { name: 'reviews', url: 'http://reviews:4003/graphql' },
    ],
  }),
});

const server = new ApolloServer({
  gateway,
});
```

### Apollo Router (Rust, рекомендуется для production)

```yaml
# router.yaml
supergraph:
  introspection: true
  listen: 0.0.0.0:4000

subgraphs:
  users:
    routing_url: http://users:4001/graphql
  products:
    routing_url: http://products:4002/graphql
  reviews:
    routing_url: http://reviews:4003/graphql

cors:
  origins:
    - https://app.example.com
```

```bash
# Запуск Router
./router --config router.yaml --supergraph supergraph.graphql
```

## Supergraph Schema

### Композиция схемы

```bash
# rover CLI для композиции
rover supergraph compose --config ./supergraph.yaml > supergraph.graphql
```

```yaml
# supergraph.yaml
federation_version: =2.0.0
subgraphs:
  users:
    routing_url: http://users:4001/graphql
    schema:
      file: ./users/schema.graphql
  products:
    routing_url: http://products:4002/graphql
    schema:
      file: ./products/schema.graphql
  reviews:
    routing_url: http://reviews:4003/graphql
    schema:
      file: ./reviews/schema.graphql
```

## Паттерны федерации

### Entity References

```javascript
// Передача ссылки на сущность
const resolvers = {
  Review: {
    // Возвращаем только id, Gateway загрузит остальное
    author: (review) => ({ __typename: 'User', id: review.authorId }),
  },
};
```

### Computed Fields

```graphql
# reviews subgraph добавляет поле к Product
type Product @key(fields: "id") {
  id: ID!
  reviews: [Review!]!
  averageRating: Float  # Вычисляется из reviews
}
```

### Value Types

```graphql
# Типы без @key (не сущности)
type Address {
  street: String!
  city: String!
  country: String!
}

# Можно определить в нескольких subgraphs
type Money @shareable {
  amount: Float!
  currency: String!
}
```

## Обработка ошибок

```javascript
// В subgraph
const resolvers = {
  User: {
    __resolveReference: async (reference, context) => {
      try {
        const user = await db.users.findById(reference.id);
        if (!user) {
          return null; // Gateway обработает null
        }
        return user;
      } catch (error) {
        // Логируем ошибку
        console.error('Failed to resolve user:', error);
        throw error; // Gateway пометит поле как errored
      }
    },
  },
};
```

## Мониторинг и отладка

### Apollo Studio

```javascript
import { ApolloServerPluginUsageReporting } from '@apollo/server/plugin/usageReporting';

const server = new ApolloServer({
  schema: buildSubgraphSchema({ typeDefs, resolvers }),
  plugins: [
    ApolloServerPluginUsageReporting({
      sendVariableValues: { all: true },
    }),
  ],
});
```

### Query Plan

```graphql
# Gateway показывает план выполнения
query {
  user(id: "1") {
    name           # -> Users subgraph
    reviews {      # -> Reviews subgraph
      rating
      product {    # -> Products subgraph
        name
      }
    }
  }
}
```

## Лучшие практики

1. **Чёткие границы сервисов** — один домен = один subgraph
2. **Минимум внешних зависимостей** — @external только когда необходимо
3. **Reference resolvers должны быть быстрыми** — используйте DataLoader
4. **Версионирование схемы** — rover для проверки совместимости
5. **Мониторинг** — Apollo Studio для отслеживания производительности

```bash
# Проверка изменений схемы
rover subgraph check my-graph@production \
  --schema ./schema.graphql \
  --name users
```

## Миграция на федерацию

### Шаг 1: Выделите первый subgraph

```graphql
# Начните с одного домена (например, Users)
type Query {
  me: User
  user(id: ID!): User
}

type User @key(fields: "id") {
  id: ID!
  name: String!
  email: String!
}
```

### Шаг 2: Настройте Gateway

```javascript
const gateway = new ApolloGateway({
  subgraphs: [
    { name: 'users', url: 'http://users:4001/graphql' },
    // Остальной монолит пока здесь
    { name: 'monolith', url: 'http://monolith:4000/graphql' },
  ],
});
```

### Шаг 3: Постепенно выделяйте сервисы

Выделяйте домен за доменом, обновляя Gateway.

## Заключение

Apollo Federation позволяет строить масштабируемые GraphQL API из независимых микросервисов. Каждая команда может владеть своим subgraph, деплоить независимо, при этом клиенты видят единый граф данных.
