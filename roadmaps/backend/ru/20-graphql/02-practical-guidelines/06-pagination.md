# Пагинация в GraphQL

## Введение

**Пагинация** — это техника разбиения больших наборов данных на страницы. В GraphQL особенно важна правильная реализация пагинации, так как клиенты могут запрашивать связанные коллекции на любом уровне вложенности.

## Виды пагинации

| Тип | Описание | Плюсы | Минусы |
|-----|----------|-------|--------|
| Offset-based | Смещение + лимит | Простота, произвольный доступ | Проблемы при изменении данных |
| Cursor-based | Курсор + лимит | Стабильность, производительность | Нет произвольного доступа |
| Page-based | Номер страницы + размер | Понятно пользователю | Те же проблемы что и offset |

## Offset-based пагинация

### Схема

```graphql
type Query {
  posts(offset: Int = 0, limit: Int = 10): [Post!]!
}
```

### Реализация

```javascript
const resolvers = {
  Query: {
    posts: async (_, { offset = 0, limit = 10 }) => {
      return db.posts.findMany({
        skip: offset,
        take: Math.min(limit, 100), // Максимум 100
        orderBy: { createdAt: 'desc' },
      });
    },
  },
};
```

### Проблема сдвига данных

```
Страница 1: [A, B, C, D, E] (offset=0, limit=5)
-- Удаляем A --
Страница 2: [G, H, I, J, K] (offset=5, limit=5)
-- F пропущен! --
```

### Когда использовать

- Данные редко меняются
- Нужен произвольный доступ к страницам
- Небольшие наборы данных

## Cursor-based пагинация

### Relay Connection Specification

Стандарт де-факто для пагинации в GraphQL.

```graphql
type Query {
  posts(
    first: Int
    after: String
    last: Int
    before: String
  ): PostConnection!
}

type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  totalCount: Int
}

type PostEdge {
  node: Post!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

### Пример запроса

```graphql
query {
  posts(first: 10, after: "cursor_abc") {
    edges {
      cursor
      node {
        id
        title
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
    totalCount
  }
}
```

### Реализация курсоров

```javascript
// Простой курсор на основе ID
function encodeCursor(id) {
  return Buffer.from(`cursor:${id}`).toString('base64');
}

function decodeCursor(cursor) {
  const decoded = Buffer.from(cursor, 'base64').toString('utf-8');
  return decoded.replace('cursor:', '');
}

// Курсор с сортировкой
function encodeCursor(item, sortField) {
  const data = { id: item.id, [sortField]: item[sortField] };
  return Buffer.from(JSON.stringify(data)).toString('base64');
}
```

### Полная реализация

```javascript
const resolvers = {
  Query: {
    posts: async (_, { first, after, last, before }) => {
      // Определяем направление
      const isForward = first != null;
      const limit = first || last || 10;
      const cursor = after || before;

      // Строим запрос
      let where = {};
      let orderBy = { id: isForward ? 'asc' : 'desc' };

      if (cursor) {
        const cursorId = decodeCursor(cursor);
        where.id = isForward ? { gt: cursorId } : { lt: cursorId };
      }

      // Запрашиваем на 1 больше для определения hasNextPage
      const items = await db.posts.findMany({
        where,
        orderBy,
        take: limit + 1,
      });

      // Определяем, есть ли ещё страницы
      const hasMore = items.length > limit;
      if (hasMore) items.pop();

      // Для обратной навигации разворачиваем
      if (!isForward) items.reverse();

      // Формируем edges
      const edges = items.map((item) => ({
        cursor: encodeCursor(item.id),
        node: item,
      }));

      return {
        edges,
        pageInfo: {
          hasNextPage: isForward ? hasMore : cursor != null,
          hasPreviousPage: isForward ? cursor != null : hasMore,
          startCursor: edges[0]?.cursor,
          endCursor: edges[edges.length - 1]?.cursor,
        },
        totalCount: () => db.posts.count({ where: {} }),
      };
    },
  },
};
```

## Пагинация вложенных коллекций

### Схема

```graphql
type User {
  id: ID!
  name: String!
  posts(first: Int, after: String): PostConnection!
  followers(first: Int, after: String): UserConnection!
}
```

### Реализация с DataLoader

```javascript
const resolvers = {
  User: {
    posts: async (user, { first = 10, after }, context) => {
      // Используем DataLoader для батчинга
      return context.loaders.userPosts.load({
        userId: user.id,
        first,
        after,
      });
    },
  },
};

// DataLoader
const userPostsLoader = new DataLoader(async (keys) => {
  // Группируем запросы
  const results = await Promise.all(
    keys.map(({ userId, first, after }) =>
      getPaginatedPosts({ userId, first, after })
    )
  );
  return results;
});
```

## Оптимизация производительности

### Индексы базы данных

```sql
-- Для cursor-based пагинации по дате
CREATE INDEX idx_posts_created_at ON posts(created_at, id);

-- Для фильтрации с пагинацией
CREATE INDEX idx_posts_author_created ON posts(author_id, created_at, id);
```

### Кэширование totalCount

```javascript
const resolvers = {
  PostConnection: {
    // Ленивое вычисление totalCount
    totalCount: async (connection, _, context) => {
      // Кэшируем результат
      const cacheKey = `posts:count:${connection.filter}`;
      let count = await cache.get(cacheKey);

      if (count === null) {
        count = await db.posts.count({ where: connection.where });
        await cache.set(cacheKey, count, 60); // TTL 60 секунд
      }

      return count;
    },
  },
};
```

### Оценочный подсчёт

```javascript
// Для очень больших таблиц
async function estimateCount(tableName) {
  const result = await db.$queryRaw`
    SELECT reltuples::bigint AS estimate
    FROM pg_class
    WHERE relname = ${tableName}
  `;
  return result[0].estimate;
}
```

## Пагинация с фильтрацией и сортировкой

### Схема

```graphql
input PostFilter {
  authorId: ID
  status: PostStatus
  createdAfter: DateTime
}

enum PostOrderBy {
  CREATED_AT_ASC
  CREATED_AT_DESC
  TITLE_ASC
  VIEWS_DESC
}

type Query {
  posts(
    first: Int
    after: String
    filter: PostFilter
    orderBy: PostOrderBy = CREATED_AT_DESC
  ): PostConnection!
}
```

### Комбинированные курсоры

```javascript
// Курсор с учётом сортировки
function encodeCursor(item, orderBy) {
  const data = {
    id: item.id,
    sortValue: getSortValue(item, orderBy),
  };
  return Buffer.from(JSON.stringify(data)).toString('base64');
}

function getSortValue(item, orderBy) {
  switch (orderBy) {
    case 'CREATED_AT_DESC':
      return item.createdAt.toISOString();
    case 'VIEWS_DESC':
      return item.views;
    default:
      return item.id;
  }
}

// Построение where с учётом курсора и сортировки
function buildCursorWhere(cursor, orderBy) {
  if (!cursor) return {};

  const { id, sortValue } = JSON.parse(
    Buffer.from(cursor, 'base64').toString()
  );

  // Составной курсор для стабильной сортировки
  switch (orderBy) {
    case 'CREATED_AT_DESC':
      return {
        OR: [
          { createdAt: { lt: new Date(sortValue) } },
          {
            createdAt: { equals: new Date(sortValue) },
            id: { lt: id },
          },
        ],
      };
    default:
      return { id: { gt: id } };
  }
}
```

## Бесконечная прокрутка

### Клиентская реализация

```javascript
const POSTS_QUERY = gql`
  query Posts($first: Int!, $after: String) {
    posts(first: $first, after: $after) {
      edges {
        node {
          id
          title
        }
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
`;

function PostList() {
  const { data, fetchMore, loading } = useQuery(POSTS_QUERY, {
    variables: { first: 10 },
  });

  const loadMore = () => {
    if (!data.posts.pageInfo.hasNextPage) return;

    fetchMore({
      variables: {
        after: data.posts.pageInfo.endCursor,
      },
      updateQuery: (prev, { fetchMoreResult }) => ({
        posts: {
          ...fetchMoreResult.posts,
          edges: [...prev.posts.edges, ...fetchMoreResult.posts.edges],
        },
      }),
    });
  };

  return (
    <>
      {data.posts.edges.map(({ node }) => (
        <PostCard key={node.id} post={node} />
      ))}
      {data.posts.pageInfo.hasNextPage && (
        <button onClick={loadMore} disabled={loading}>
          Load More
        </button>
      )}
    </>
  );
}
```

## Лучшие практики

1. **Используйте cursor-based для production** — стабильнее и быстрее
2. **Ограничивайте максимум элементов** — защита от слишком больших запросов
3. **Делайте totalCount опциональным** — дорогая операция
4. **Индексируйте поля сортировки** — производительность БД
5. **Используйте составные курсоры** — для стабильной сортировки
6. **Кэшируйте подсчёты** — для часто запрашиваемых фильтров

```javascript
// Ограничение максимума
const limit = Math.min(first || 10, 100);
```

## Заключение

Cursor-based пагинация по Relay спецификации — рекомендуемый подход для GraphQL API. Она обеспечивает стабильность при изменении данных и хорошую производительность. Offset-based пагинация подходит для простых случаев с редко меняющимися данными.
