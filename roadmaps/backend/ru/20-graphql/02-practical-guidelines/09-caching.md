# Кэширование в GraphQL

[prev: 08-global-object-identification](./08-global-object-identification.md) | [next: 10-performance](./10-performance.md)

---

## Введение

Кэширование в GraphQL сложнее, чем в REST, из-за динамической природы запросов. Каждый запрос может быть уникальным, что делает традиционное HTTP-кэширование менее эффективным. Однако существуют стратегии для эффективного кэширования на разных уровнях.

## Уровни кэширования

| Уровень | Описание | Инструменты |
|---------|----------|-------------|
| CDN/HTTP | Кэширование ответов | Cloudflare, Fastly |
| Серверный | Кэширование в памяти/Redis | Redis, Memcached |
| Резолверы | Кэширование результатов | DataLoader |
| Клиентский | Нормализованный кэш | Apollo Client |

## HTTP кэширование

### Проблема с POST

GraphQL обычно использует POST-запросы, которые не кэшируются по умолчанию:

```http
POST /graphql
Cache-Control: no-store
```

### Решение: GET для запросов

```javascript
// Apollo Client с GET для queries
const link = createHttpLink({
  uri: '/graphql',
  useGETForQueries: true, // Queries через GET
});

// Запрос превращается в
// GET /graphql?query={...}&variables={...}
```

### Automatic Persisted Queries (APQ)

```javascript
// Apollo Server
const server = new ApolloServer({
  typeDefs,
  resolvers,
  persistedQueries: {
    cache: new Redis({ host: 'localhost' }),
  },
});

// Запрос с хешем
// GET /graphql?extensions={"persistedQuery":{"sha256Hash":"abc..."}}
```

### Cache-Control заголовки

```javascript
// Установка Cache-Control для GraphQL ответов
const resolvers = {
  Query: {
    posts: (_, __, context) => {
      // Публичные данные можно кэшировать
      context.cacheControl.setCacheHint({ maxAge: 60, scope: 'PUBLIC' });
      return db.posts.findMany();
    },

    me: (_, __, context) => {
      // Приватные данные не кэшируются
      context.cacheControl.setCacheHint({ maxAge: 0, scope: 'PRIVATE' });
      return context.user;
    },
  },
};
```

### Директива @cacheControl

```graphql
type Query {
  posts: [Post!]! @cacheControl(maxAge: 60)
  me: User @cacheControl(maxAge: 0, scope: PRIVATE)
}

type Post @cacheControl(maxAge: 120) {
  id: ID!
  title: String!
  viewCount: Int! @cacheControl(maxAge: 10)  # Часто меняется
}
```

## Серверное кэширование

### Redis кэш

```javascript
import Redis from 'ioredis';

const redis = new Redis();

// Кэширование результатов резолвера
const resolvers = {
  Query: {
    post: async (_, { id }) => {
      const cacheKey = `post:${id}`;

      // Проверяем кэш
      const cached = await redis.get(cacheKey);
      if (cached) {
        return JSON.parse(cached);
      }

      // Загружаем из БД
      const post = await db.posts.findById(id);

      // Сохраняем в кэш
      if (post) {
        await redis.setex(cacheKey, 3600, JSON.stringify(post));
      }

      return post;
    },
  },
};
```

### Инвалидация кэша

```javascript
const resolvers = {
  Mutation: {
    updatePost: async (_, { id, input }) => {
      const post = await db.posts.update(id, input);

      // Инвалидируем кэш
      await redis.del(`post:${id}`);

      // Инвалидируем связанные кэши
      await redis.del(`user:${post.authorId}:posts`);

      return post;
    },
  },
};
```

### Паттерн Cache-Aside

```javascript
class CacheService {
  constructor(redis, ttl = 3600) {
    this.redis = redis;
    this.ttl = ttl;
  }

  async get(key, loader) {
    // Пробуем получить из кэша
    const cached = await this.redis.get(key);
    if (cached) {
      return JSON.parse(cached);
    }

    // Загружаем данные
    const data = await loader();

    // Сохраняем в кэш
    if (data) {
      await this.redis.setex(key, this.ttl, JSON.stringify(data));
    }

    return data;
  }

  async invalidate(pattern) {
    const keys = await this.redis.keys(pattern);
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
  }
}

// Использование
const resolvers = {
  Query: {
    post: (_, { id }, { cache }) => {
      return cache.get(`post:${id}`, () => db.posts.findById(id));
    },
  },
};
```

## DataLoader

### Решение N+1 проблемы

```javascript
import DataLoader from 'dataloader';

// Создание лоадеров
function createLoaders() {
  return {
    users: new DataLoader(async (ids) => {
      const users = await db.users.findMany({
        where: { id: { in: ids } },
      });

      // Возвращаем в том же порядке
      return ids.map((id) => users.find((u) => u.id === id));
    }),

    postsByAuthor: new DataLoader(async (authorIds) => {
      const posts = await db.posts.findMany({
        where: { authorId: { in: authorIds } },
      });

      return authorIds.map((id) =>
        posts.filter((p) => p.authorId === id)
      );
    }),
  };
}

// Контекст
const server = new ApolloServer({
  context: () => ({
    loaders: createLoaders(),
  }),
});

// Резолверы
const resolvers = {
  Post: {
    author: (post, _, { loaders }) => {
      return loaders.users.load(post.authorId);
    },
  },

  User: {
    posts: (user, _, { loaders }) => {
      return loaders.postsByAuthor.load(user.id);
    },
  },
};
```

### DataLoader с кэшированием

```javascript
const usersLoader = new DataLoader(
  async (ids) => {
    // Проверяем Redis
    const cached = await redis.mget(ids.map(id => `user:${id}`));
    const result = [];
    const toLoad = [];

    ids.forEach((id, index) => {
      if (cached[index]) {
        result[index] = JSON.parse(cached[index]);
      } else {
        toLoad.push({ id, index });
      }
    });

    // Загружаем отсутствующие
    if (toLoad.length > 0) {
      const users = await db.users.findMany({
        where: { id: { in: toLoad.map(t => t.id) } },
      });

      // Сохраняем в Redis и результат
      const pipeline = redis.pipeline();
      toLoad.forEach(({ id, index }) => {
        const user = users.find(u => u.id === id);
        result[index] = user;
        if (user) {
          pipeline.setex(`user:${id}`, 3600, JSON.stringify(user));
        }
      });
      await pipeline.exec();
    }

    return result;
  },
  {
    cache: true,      // In-memory кэш на время запроса
    maxBatchSize: 100, // Максимум в одном батче
  }
);
```

## Клиентское кэширование

### Apollo Client Normalized Cache

```javascript
import { ApolloClient, InMemoryCache } from '@apollo/client';

const client = new ApolloClient({
  cache: new InMemoryCache({
    typePolicies: {
      Post: {
        // Уникальный ключ для кэширования
        keyFields: ['id'],

        fields: {
          // Кастомное слияние для пагинированных данных
          comments: {
            keyArgs: false,
            merge(existing = [], incoming) {
              return [...existing, ...incoming];
            },
          },
        },
      },
    },
  }),
});
```

### Политики кэширования

```javascript
const cache = new InMemoryCache({
  typePolicies: {
    Query: {
      fields: {
        posts: {
          // Аргументы, влияющие на кэш
          keyArgs: ['filter', 'orderBy'],

          // Объединение страниц
          merge(existing, incoming, { args }) {
            if (!args?.after) {
              return incoming;
            }
            return {
              ...incoming,
              edges: [...(existing?.edges || []), ...incoming.edges],
            };
          },

          // Чтение из кэша
          read(existing) {
            return existing;
          },
        },
      },
    },
  },
});
```

### Fetch Policies

```javascript
// network-only: всегда с сервера
// cache-first: сначала кэш (по умолчанию)
// cache-only: только кэш
// cache-and-network: кэш + обновление с сервера

const { data } = useQuery(GET_POSTS, {
  fetchPolicy: 'cache-and-network',
});
```

## Стратегии инвалидации

### Time-based (TTL)

```javascript
// Redis с TTL
await redis.setex('post:123', 3600, JSON.stringify(post)); // 1 час

// Apollo @cacheControl
type Post @cacheControl(maxAge: 3600) {
  id: ID!
}
```

### Event-based

```javascript
// При обновлении данных
const resolvers = {
  Mutation: {
    updatePost: async (_, { id, input }, { pubsub }) => {
      const post = await db.posts.update(id, input);

      // Публикуем событие для инвалидации
      await pubsub.publish('POST_UPDATED', { postId: id });

      return post;
    },
  },
};

// Слушатель инвалидации
pubsub.subscribe('POST_UPDATED', async ({ postId }) => {
  await redis.del(`post:${postId}`);
});
```

### Tag-based

```javascript
class TaggedCache {
  async set(key, value, tags) {
    await redis.set(key, JSON.stringify(value));

    // Сохраняем связи тегов
    for (const tag of tags) {
      await redis.sadd(`tag:${tag}`, key);
    }
  }

  async invalidateByTag(tag) {
    const keys = await redis.smembers(`tag:${tag}`);
    if (keys.length > 0) {
      await redis.del(...keys);
    }
    await redis.del(`tag:${tag}`);
  }
}

// Использование
await cache.set('post:123', post, ['posts', `author:${post.authorId}`]);
await cache.invalidateByTag('posts'); // Инвалидирует все посты
```

## Response Caching

### Полное кэширование ответов

```javascript
import responseCachePlugin from '@apollo/server-plugin-response-cache';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    responseCachePlugin({
      // Кастомный ключ кэша
      sessionId: (context) => context.user?.id || null,

      // Определение кэшируемости
      shouldReadFromCache: (context) => {
        return context.request.http?.headers.get('x-bypass-cache') !== 'true';
      },
    }),
  ],
});
```

## Лучшие практики

1. **Используйте DataLoader** — решает N+1 и кэширует в рамках запроса
2. **Кэшируйте на разных уровнях** — CDN, Redis, клиент
3. **Инвалидируйте умно** — события лучше TTL для согласованности
4. **Учитывайте авторизацию** — разные пользователи = разные кэши
5. **Мониторьте hit rate** — отслеживайте эффективность кэша

```javascript
// Метрики кэша
const cacheStats = {
  hits: 0,
  misses: 0,

  recordHit() { this.hits++; },
  recordMiss() { this.misses++; },

  getHitRate() {
    const total = this.hits + this.misses;
    return total > 0 ? this.hits / total : 0;
  },
};
```

## Заключение

Кэширование в GraphQL требует многоуровневого подхода. Используйте DataLoader для батчинга, Redis для серверного кэша, и нормализованный кэш на клиенте. Правильная стратегия инвалидации — ключ к согласованности данных.

---

[prev: 08-global-object-identification](./08-global-object-identification.md) | [next: 10-performance](./10-performance.md)
