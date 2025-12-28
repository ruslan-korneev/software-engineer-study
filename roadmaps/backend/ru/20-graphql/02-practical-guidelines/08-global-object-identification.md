# Глобальная идентификация объектов

[prev: 07-schema-design](./07-schema-design.md) | [next: 09-caching](./09-caching.md)

---

## Введение

**Глобальная идентификация объектов** — это паттерн, позволяющий уникально идентифицировать любой объект в GraphQL схеме с помощью единого глобального ID. Это основа для эффективного кэширования на клиенте и универсального доступа к объектам.

## Relay Node Interface

Спецификация Relay определяет стандартный интерфейс для глобальной идентификации:

```graphql
interface Node {
  id: ID!
}

type Query {
  node(id: ID!): Node
  nodes(ids: [ID!]!): [Node]!
}
```

### Преимущества

| Преимущество | Описание |
|--------------|----------|
| Унифицированный доступ | Любой объект по одному эндпоинту |
| Кэширование | Клиент может кэшировать по глобальному ID |
| Рефетчинг | Обновление любого объекта без знания типа |
| Связи | Ссылки между объектами через ID |

## Формат глобального ID

### Структура

Глобальный ID обычно включает:
1. Тип объекта
2. Локальный идентификатор

```
base64("User:123") = "VXNlcjoxMjM="
```

### Реализация кодирования

```javascript
// Кодирование
function toGlobalId(type, localId) {
  return Buffer.from(`${type}:${localId}`).toString('base64');
}

// Декодирование
function fromGlobalId(globalId) {
  const decoded = Buffer.from(globalId, 'base64').toString('utf-8');
  const [type, localId] = decoded.split(':');
  return { type, localId };
}

// Примеры
toGlobalId('User', '123');      // "VXNlcjoxMjM="
toGlobalId('Post', 'abc-def');  // "UG9zdDphYmMtZGVm"

fromGlobalId('VXNlcjoxMjM=');   // { type: 'User', localId: '123' }
```

### Альтернативные форматы

```javascript
// URL-safe base64
function toGlobalId(type, localId) {
  return Buffer.from(`${type}:${localId}`)
    .toString('base64')
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=/g, '');
}

// Префикс + ID (без кодирования)
function toGlobalId(type, localId) {
  const prefixes = { User: 'usr', Post: 'pst', Comment: 'cmt' };
  return `${prefixes[type]}_${localId}`;
}
// Результат: "usr_123", "pst_abc-def"
```

## Реализация Node Interface

### Схема

```graphql
interface Node {
  id: ID!
}

type User implements Node {
  id: ID!
  name: String!
  email: String!
}

type Post implements Node {
  id: ID!
  title: String!
  content: String!
  author: User!
}

type Comment implements Node {
  id: ID!
  text: String!
  author: User!
}

type Query {
  node(id: ID!): Node
  nodes(ids: [ID!]!): [Node]!
}
```

### Резолверы

```javascript
const resolvers = {
  Query: {
    node: async (_, { id }, context) => {
      const { type, localId } = fromGlobalId(id);

      switch (type) {
        case 'User':
          return context.loaders.users.load(localId);
        case 'Post':
          return context.loaders.posts.load(localId);
        case 'Comment':
          return context.loaders.comments.load(localId);
        default:
          return null;
      }
    },

    nodes: async (_, { ids }, context) => {
      return Promise.all(
        ids.map(id => resolvers.Query.node(_, { id }, context))
      );
    },
  },

  Node: {
    __resolveType(obj) {
      // Определяем тип по наличию полей или __typename
      if (obj.__typename) return obj.__typename;
      if (obj.email) return 'User';
      if (obj.title) return 'Post';
      if (obj.text) return 'Comment';
      return null;
    },
  },

  User: {
    id: (user) => toGlobalId('User', user.id),
  },

  Post: {
    id: (post) => toGlobalId('Post', post.id),
  },

  Comment: {
    id: (comment) => toGlobalId('Comment', comment.id),
  },
};
```

### Оптимизация с DataLoader

```javascript
import DataLoader from 'dataloader';

function createNodeLoader(context) {
  return new DataLoader(async (ids) => {
    // Группируем ID по типам
    const grouped = {};
    ids.forEach((id, index) => {
      const { type, localId } = fromGlobalId(id);
      if (!grouped[type]) grouped[type] = [];
      grouped[type].push({ localId, index });
    });

    // Загружаем каждый тип пакетом
    const results = new Array(ids.length);

    await Promise.all(
      Object.entries(grouped).map(async ([type, items]) => {
        const localIds = items.map(i => i.localId);
        const entities = await loadByType(type, localIds);

        items.forEach((item, i) => {
          results[item.index] = entities[i];
        });
      })
    );

    return results;
  });
}
```

## Использование на клиенте

### Кэширование Apollo Client

```javascript
import { ApolloClient, InMemoryCache } from '@apollo/client';

const client = new ApolloClient({
  cache: new InMemoryCache({
    // Apollo использует id для нормализации
    typePolicies: {
      User: {
        keyFields: ['id'], // Глобальный ID как ключ кэша
      },
      Post: {
        keyFields: ['id'],
      },
    },
  }),
});
```

### Рефетчинг объекта

```javascript
// Обновление любого объекта по глобальному ID
const REFETCH_NODE = gql`
  query RefetchNode($id: ID!) {
    node(id: $id) {
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
`;

function refetchNode(id) {
  return client.query({
    query: REFETCH_NODE,
    variables: { id },
    fetchPolicy: 'network-only',
  });
}
```

### Fragment Matching

```javascript
// Фрагменты для повторного использования
const USER_FIELDS = gql`
  fragment UserFields on User {
    id
    name
    avatar {
      url
    }
  }
`;

const COMMENT_WITH_AUTHOR = gql`
  ${USER_FIELDS}
  fragment CommentWithAuthor on Comment {
    id
    text
    author {
      ...UserFields
    }
  }
`;
```

## Паттерны использования

### 1. Ссылки между объектами

```graphql
type Post {
  id: ID!
  title: String!
  # Вместо authorId храним глобальный ID
  author: User!
}

type Mutation {
  createComment(
    postId: ID!    # Глобальный ID поста
    text: String!
  ): Comment!
}
```

### 2. Оптимистичные обновления

```javascript
// Клиент может обновить кэш сразу
const [createComment] = useMutation(CREATE_COMMENT, {
  optimisticResponse: {
    createComment: {
      __typename: 'Comment',
      id: 'temp-id', // Временный ID
      text: input.text,
      author: currentUser,
    },
  },
  update(cache, { data }) {
    // Обновляем список комментариев
    cache.modify({
      id: cache.identify({ __typename: 'Post', id: postId }),
      fields: {
        comments(existing = []) {
          return [...existing, data.createComment];
        },
      },
    });
  },
});
```

### 3. Универсальные компоненты

```javascript
// Компонент, работающий с любым Node
function NodeCard({ nodeId }) {
  const { data } = useQuery(NODE_QUERY, {
    variables: { id: nodeId },
  });

  if (!data?.node) return null;

  switch (data.node.__typename) {
    case 'User':
      return <UserCard user={data.node} />;
    case 'Post':
      return <PostCard post={data.node} />;
    default:
      return <GenericCard node={data.node} />;
  }
}
```

## Связь с Relay Connections

```graphql
type Query {
  node(id: ID!): Node
}

type User implements Node {
  id: ID!
  posts(first: Int, after: String): PostConnection!
}

type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
}

type PostEdge {
  cursor: String!
  node: Post!  # Post также реализует Node
}
```

## Безопасность

### Валидация ID

```javascript
function fromGlobalId(globalId) {
  try {
    const decoded = Buffer.from(globalId, 'base64').toString('utf-8');
    const colonIndex = decoded.indexOf(':');

    if (colonIndex === -1) {
      throw new Error('Invalid global ID format');
    }

    const type = decoded.substring(0, colonIndex);
    const localId = decoded.substring(colonIndex + 1);

    // Проверяем, что тип существует
    const validTypes = ['User', 'Post', 'Comment'];
    if (!validTypes.includes(type)) {
      throw new Error('Unknown type in global ID');
    }

    return { type, localId };
  } catch (error) {
    throw new GraphQLError('Invalid ID', {
      extensions: { code: 'INVALID_ID' },
    });
  }
}
```

### Авторизация

```javascript
const resolvers = {
  Query: {
    node: async (_, { id }, context) => {
      const { type, localId } = fromGlobalId(id);
      const entity = await loadEntity(type, localId);

      if (!entity) return null;

      // Проверка прав доступа
      if (!canAccess(context.user, entity)) {
        return null; // или throw ForbiddenError
      }

      return entity;
    },
  },
};
```

## Миграция на глобальные ID

### Шаг 1: Добавьте поле nodeId

```graphql
type User {
  id: ID!       # Старый локальный ID
  nodeId: ID!   # Новый глобальный ID
}
```

### Шаг 2: Клиенты мигрируют

```javascript
// Старый код
const userId = user.id; // "123"

// Новый код
const nodeId = user.nodeId; // "VXNlcjoxMjM="
```

### Шаг 3: Переименование

```graphql
type User {
  id: ID!           # Теперь глобальный ID
  databaseId: Int!  # Локальный ID для совместимости
}
```

## Лучшие практики

1. **Используйте Node Interface** для всех сущностей с ID
2. **Делайте ID непрозрачными** — клиенты не должны их парсить
3. **Кодируйте тип в ID** — для маршрутизации запросов
4. **Используйте DataLoader** — для оптимизации запросов
5. **Валидируйте ID** — защита от инъекций

## Заключение

Глобальная идентификация объектов — мощный паттерн, обеспечивающий унифицированный доступ к данным, эффективное кэширование и простоту рефетчинга. Следуйте спецификации Relay Node Interface для совместимости с существующими инструментами.

---

[prev: 07-schema-design](./07-schema-design.md) | [next: 09-caching](./09-caching.md)
