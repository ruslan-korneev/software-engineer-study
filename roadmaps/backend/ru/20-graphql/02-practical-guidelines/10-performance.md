# Производительность GraphQL

[prev: 09-caching](./09-caching.md) | [next: 11-security](./11-security.md)

---

## Введение

Гибкость GraphQL может привести к проблемам производительности, если не принять соответствующие меры. Клиенты могут запрашивать слишком много данных, создавать глубоко вложенные запросы или вызывать проблему N+1. Понимание и решение этих проблем критически важно.

## Основные проблемы производительности

| Проблема | Описание | Решение |
|----------|----------|---------|
| N+1 запросов | Множество SQL запросов для связанных данных | DataLoader |
| Глубокие запросы | Неограниченная вложенность | Лимит глубины |
| Сложные запросы | Тяжёлые вычисления | Анализ стоимости |
| Over-fetching | Загрузка лишних данных | Ленивые резолверы |

## Проблема N+1

### Пример проблемы

```graphql
query {
  posts {          # 1 запрос
    title
    author {       # N запросов (по одному на пост)
      name
    }
  }
}
```

```sql
-- 1 запрос для постов
SELECT * FROM posts;

-- N запросов для авторов
SELECT * FROM users WHERE id = 1;
SELECT * FROM users WHERE id = 2;
SELECT * FROM users WHERE id = 3;
-- ... и так далее
```

### Решение: DataLoader

```javascript
import DataLoader from 'dataloader';

// Создание лоадера
const userLoader = new DataLoader(async (userIds) => {
  // Один запрос для всех пользователей
  const users = await db.query(`
    SELECT * FROM users WHERE id IN (${userIds.join(',')})
  `);

  // Возвращаем в том же порядке
  return userIds.map(id => users.find(u => u.id === id));
});

// Резолвер
const resolvers = {
  Post: {
    author: (post) => userLoader.load(post.authorId),
  },
};
```

### Контекст с DataLoaders

```javascript
// Создаём лоадеры для каждого запроса
function createLoaders() {
  return {
    users: new DataLoader((ids) => batchLoadUsers(ids)),
    posts: new DataLoader((ids) => batchLoadPosts(ids)),
    postsByAuthor: new DataLoader((authorIds) => batchLoadPostsByAuthors(authorIds)),
  };
}

const server = new ApolloServer({
  context: ({ req }) => ({
    loaders: createLoaders(), // Новые лоадеры для каждого запроса
    user: getUserFromToken(req),
  }),
});
```

## Ограничение глубины запросов

### Проблема

```graphql
# Потенциально бесконечная вложенность
query {
  user(id: "1") {
    friends {
      friends {
        friends {
          friends {
            # ... до бесконечности
          }
        }
      }
    }
  }
}
```

### Решение: depth-limit

```javascript
import depthLimit from 'graphql-depth-limit';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    depthLimit(5), // Максимальная глубина 5
  ],
});
```

### Кастомная реализация

```javascript
import { getDirectives } from '@graphql-tools/utils';

function createDepthLimitRule(maxDepth) {
  return function depthLimitRule(context) {
    return {
      Field(node, key, parent, path, ancestors) {
        const depth = ancestors.filter(
          (ancestor) => ancestor.kind === 'Field'
        ).length;

        if (depth > maxDepth) {
          context.reportError(
            new GraphQLError(
              `Query depth of ${depth} exceeds maximum allowed depth of ${maxDepth}`
            )
          );
        }
      },
    };
  };
}
```

## Анализ стоимости запросов

### Query Cost Analysis

```javascript
import { createComplexityRule } from 'graphql-query-complexity';

const complexityRule = createComplexityRule({
  maximumComplexity: 1000,
  estimators: [
    // Стоимость на основе множителей
    {
      type: 'list',
      multiplier: (args) => args.first || args.limit || 10,
    },
    // Базовая стоимость полей
    {
      type: 'field',
      cost: 1,
    },
  ],
  onComplete: (complexity) => {
    console.log('Query complexity:', complexity);
  },
});

const server = new ApolloServer({
  validationRules: [complexityRule],
});
```

### Директива @cost

```graphql
directive @cost(
  complexity: Int!
  multipliers: [String!]
) on FIELD_DEFINITION

type Query {
  users(first: Int = 10): [User!]! @cost(complexity: 10, multipliers: ["first"])
  expensiveQuery: Result! @cost(complexity: 100)
}

type User {
  id: ID!
  name: String! @cost(complexity: 1)
  posts(first: Int = 10): [Post!]! @cost(complexity: 5, multipliers: ["first"])
}
```

### Реализация подсчёта стоимости

```javascript
function calculateQueryCost(query, variables, schema) {
  let cost = 0;

  visit(query, {
    Field(node) {
      const fieldDef = getFieldDef(schema, node);
      const costDirective = fieldDef.astNode?.directives?.find(
        d => d.name.value === 'cost'
      );

      if (costDirective) {
        const complexityArg = costDirective.arguments.find(
          a => a.name.value === 'complexity'
        );
        let fieldCost = parseInt(complexityArg.value.value);

        // Применяем множители
        const multipliersArg = costDirective.arguments.find(
          a => a.name.value === 'multipliers'
        );
        if (multipliersArg) {
          multipliersArg.value.values.forEach(m => {
            const argValue = variables[m.value] || 10;
            fieldCost *= argValue;
          });
        }

        cost += fieldCost;
      } else {
        cost += 1; // Базовая стоимость
      }
    },
  });

  return cost;
}
```

## Оптимизация резолверов

### Ленивые резолверы

```javascript
const resolvers = {
  User: {
    // Вычисляем только если поле запрошено
    posts: async (user, args, context) => {
      return context.loaders.postsByAuthor.load(user.id);
    },

    // Дорогое вычисление только по запросу
    statistics: async (user, _, context) => {
      return computeUserStatistics(user.id);
    },
  },
};
```

### Параллельное выполнение

```javascript
const resolvers = {
  Query: {
    dashboard: async (_, __, context) => {
      // Параллельная загрузка независимых данных
      const [stats, recentPosts, notifications] = await Promise.all([
        getStats(),
        getRecentPosts(),
        getNotifications(context.user.id),
      ]);

      return { stats, recentPosts, notifications };
    },
  },
};
```

### Defer и Stream (новые возможности)

```graphql
# @defer - отложенная загрузка части ответа
query {
  user(id: "1") {
    name
    ... @defer {
      statistics {
        postsCount
        followersCount
      }
    }
  }
}

# @stream - потоковая загрузка списков
query {
  posts @stream(initialCount: 10) {
    id
    title
  }
}
```

## Оптимизация базы данных

### Проекция полей

```javascript
const resolvers = {
  Query: {
    users: async (_, args, context, info) => {
      // Получаем запрошенные поля
      const requestedFields = getFieldsFromInfo(info);

      // Загружаем только нужные колонки
      return db.users.findMany({
        select: requestedFields.reduce((acc, field) => {
          acc[field] = true;
          return acc;
        }, {}),
      });
    },
  },
};

// Извлечение полей из info
function getFieldsFromInfo(info) {
  return info.fieldNodes[0].selectionSet.selections
    .filter(s => s.kind === 'Field')
    .map(s => s.name.value);
}
```

### Join Monster

```javascript
import joinMonster from 'join-monster';

const resolvers = {
  Query: {
    users: (_, args, context, info) => {
      return joinMonster(info, context, (sql) => {
        return db.raw(sql);
      });
    },
  },
};

// Схема с аннотациями для JOIN Monster
const User = new GraphQLObjectType({
  name: 'User',
  sqlTable: 'users',
  uniqueKey: 'id',
  fields: () => ({
    id: { type: GraphQLInt },
    name: { type: GraphQLString },
    posts: {
      type: new GraphQLList(Post),
      sqlJoin: (userTable, postTable) =>
        `${userTable}.id = ${postTable}.author_id`,
    },
  }),
});
```

## Мониторинг производительности

### Apollo Studio Tracing

```javascript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    ApolloServerPluginUsageReporting({
      sendVariableValues: { all: true },
      sendHeaders: { all: true },
    }),
  ],
});
```

### Кастомный трейсинг

```javascript
const tracingPlugin = {
  requestDidStart() {
    const start = Date.now();

    return {
      didResolveOperation({ operationName }) {
        console.log(`Operation: ${operationName}`);
      },

      willSendResponse({ response }) {
        const duration = Date.now() - start;
        console.log(`Request completed in ${duration}ms`);

        // Добавляем время в extensions
        response.extensions = {
          ...response.extensions,
          timing: { duration },
        };
      },

      executionDidStart() {
        return {
          willResolveField({ info }) {
            const start = Date.now();
            return () => {
              const duration = Date.now() - start;
              if (duration > 100) {
                console.log(
                  `Slow resolver: ${info.parentType.name}.${info.fieldName} (${duration}ms)`
                );
              }
            };
          },
        };
      },
    };
  },
};
```

### Метрики Prometheus

```javascript
import { Counter, Histogram } from 'prom-client';

const queryDuration = new Histogram({
  name: 'graphql_query_duration_seconds',
  help: 'Duration of GraphQL queries',
  labelNames: ['operationName', 'operationType'],
});

const queryErrors = new Counter({
  name: 'graphql_query_errors_total',
  help: 'Total GraphQL query errors',
  labelNames: ['operationName'],
});

const metricsPlugin = {
  requestDidStart({ request }) {
    const end = queryDuration.startTimer();

    return {
      didResolveOperation({ operationName, operation }) {
        return () => {
          end({
            operationName: operationName || 'anonymous',
            operationType: operation.operation,
          });
        };
      },

      didEncounterErrors({ operationName }) {
        queryErrors.inc({ operationName: operationName || 'anonymous' });
      },
    };
  },
};
```

## Лучшие практики

1. **Всегда используйте DataLoader** — избегайте N+1 проблемы
2. **Ограничивайте сложность** — защита от тяжёлых запросов
3. **Мониторьте резолверы** — находите узкие места
4. **Используйте проекцию** — загружайте только нужные поля
5. **Кэшируйте** — на всех уровнях
6. **Устанавливайте таймауты** — для резолверов и запросов

```javascript
// Таймаут для резолвера
const resolvers = {
  Query: {
    heavyQuery: async () => {
      return Promise.race([
        computeHeavyData(),
        new Promise((_, reject) =>
          setTimeout(() => reject(new Error('Timeout')), 5000)
        ),
      ]);
    },
  },
};
```

## Заключение

Производительность GraphQL требует внимания на нескольких уровнях: от батчинга запросов с DataLoader до ограничения сложности и мониторинга. Правильная оптимизация позволяет сохранить гибкость GraphQL без ущерба для скорости.

---

[prev: 09-caching](./09-caching.md) | [next: 11-security](./11-security.md)
