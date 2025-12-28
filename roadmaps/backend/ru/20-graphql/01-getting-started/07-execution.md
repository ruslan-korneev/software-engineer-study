# Выполнение запросов в GraphQL

[prev: 06-validation](./06-validation.md) | [next: 08-response](./08-response.md)

---

## Что такое Execution?

**Execution** (выполнение) — это процесс обработки GraphQL запроса сервером. После того как запрос прошёл парсинг и валидацию, GraphQL runtime выполняет его, вызывая резолверы для каждого поля и собирая результат в соответствии со структурой запроса.

## Этапы обработки запроса

```
┌─────────────────────────────────────────────────────────────────┐
│                        GraphQL Request                          │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. Parse                                                        │
│     Преобразование строки запроса в AST                         │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. Validate                                                     │
│     Проверка запроса на соответствие схеме                      │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. Execute                                                      │
│     Выполнение запроса и сбор результатов                       │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. Format Response                                              │
│     Формирование JSON ответа                                    │
└─────────────────────────────────────────────────────────────────┘
```

## Резолверы (Resolvers)

Резолверы — это функции, которые возвращают данные для полей схемы.

### Сигнатура резолвера

```javascript
function resolver(parent, args, context, info) {
  // parent — результат родительского резолвера
  // args — аргументы поля
  // context — общий контекст запроса
  // info — информация о запросе и схеме
}
```

| Параметр | Описание |
|----------|----------|
| `parent` (root) | Результат резолвера родительского поля |
| `args` | Аргументы, переданные в поле |
| `context` | Общий контекст для всех резолверов |
| `info` | Метаинформация о запросе |

### Базовые примеры резолверов

```javascript
const resolvers = {
  Query: {
    // Простой резолвер
    hello: () => 'Hello, World!',

    // Резолвер с аргументами
    user: (_, { id }) => {
      return users.find(user => user.id === id);
    },

    // Асинхронный резолвер
    users: async (_, args, context) => {
      return await context.db.users.findAll();
    }
  },

  User: {
    // Резолвер поля с использованием parent
    fullName: (parent) => {
      return `${parent.firstName} ${parent.lastName}`;
    },

    // Резолвер связанных данных
    posts: async (parent, _, context) => {
      return await context.db.posts.findByAuthorId(parent.id);
    }
  }
};
```

## Порядок выполнения

### Query — параллельное выполнение

Корневые поля Query выполняются параллельно.

```graphql
query {
  user(id: "1") { name }     # ─┐
  posts { title }             # ─┼─► Параллельно
  comments { text }           # ─┘
}
```

```javascript
// Внутренне это эквивалентно:
const result = await Promise.all([
  resolvers.Query.user(null, { id: "1" }),
  resolvers.Query.posts(null, {}),
  resolvers.Query.comments(null, {})
]);
```

### Mutation — последовательное выполнение

Корневые поля Mutation выполняются последовательно.

```graphql
mutation {
  createUser(name: "John") { id }  # 1. Первая
  createPost(title: "Hi") { id }   # 2. Вторая (после завершения первой)
  sendEmail(to: "john@test.com")   # 3. Третья (после завершения второй)
}
```

### Вложенные поля — параллельно на каждом уровне

```graphql
query {
  user(id: "1") {
    name
    posts {           # ─┐
      title           #  │
      comments {      #  ├─► Параллельно на уровне user
        text          #  │
      }               #  │
    }                 # ─┘
    followers {       # ─┐
      name            #  ├─► Параллельно на уровне user
    }                 # ─┘
  }
}
```

## Context (Контекст)

Context — объект, доступный всем резолверам, обычно содержит:
- Информацию о текущем пользователе
- Подключения к базе данных
- DataLoaders
- Сервисы

```javascript
// Apollo Server
const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: async ({ req }) => {
    // Получаем токен из заголовка
    const token = req.headers.authorization || '';

    // Проверяем токен и получаем пользователя
    const user = await getUserFromToken(token);

    // Создаём DataLoader для батчинга
    const userLoader = new DataLoader(async (ids) => {
      const users = await db.users.findByIds(ids);
      return ids.map(id => users.find(u => u.id === id));
    });

    return {
      user,
      db,
      loaders: {
        user: userLoader
      }
    };
  }
});
```

```javascript
// Использование контекста в резолверах
const resolvers = {
  Query: {
    me: (_, __, context) => {
      if (!context.user) {
        throw new AuthenticationError('Требуется авторизация');
      }
      return context.user;
    },

    users: async (_, __, context) => {
      return await context.db.users.findAll();
    }
  },

  Post: {
    author: async (parent, _, context) => {
      // Использование DataLoader для батчинга
      return await context.loaders.user.load(parent.authorId);
    }
  }
};
```

## Info объект

Info содержит информацию о текущем запросе и схеме.

```javascript
const resolvers = {
  Query: {
    users: (_, args, context, info) => {
      // info.fieldName — имя поля ('users')
      // info.fieldNodes — AST узлы полей
      // info.returnType — тип возврата
      // info.parentType — родительский тип
      // info.path — путь до текущего поля
      // info.schema — схема GraphQL
      // info.fragments — фрагменты из запроса
      // info.operation — операция (query/mutation/subscription)
      // info.variableValues — значения переменных

      console.log('Field:', info.fieldName);
      console.log('Path:', info.path);

      return users;
    }
  }
};
```

### Анализ запрошенных полей

```javascript
const { parseResolveInfo } = require('graphql-parse-resolve-info');

const resolvers = {
  Query: {
    users: (_, args, context, info) => {
      const parsedInfo = parseResolveInfo(info);

      // Узнаём какие поля запрошены
      const requestedFields = Object.keys(parsedInfo.fieldsByTypeName.User || {});

      // Оптимизируем запрос к БД
      if (requestedFields.includes('posts')) {
        return db.users.findAll({ include: ['posts'] });
      }

      return db.users.findAll();
    }
  }
};
```

## Default Resolvers

GraphQL имеет встроенные резолверы по умолчанию.

```javascript
// Если резолвер не определён, GraphQL использует:
const defaultResolver = (parent, args, context, info) => {
  return parent[info.fieldName];
};
```

```javascript
// Это работает без явного резолвера:
const typeDefs = `
  type User {
    id: ID!
    name: String!
    email: String!
  }
`;

const resolvers = {
  Query: {
    user: () => ({
      id: '1',
      name: 'John',
      email: 'john@example.com'
    })
    // Резолверы для id, name, email не нужны!
  }
};
```

## Обработка ошибок

### Выброс ошибок в резолверах

```javascript
const { ApolloError, AuthenticationError, ForbiddenError } = require('apollo-server');

const resolvers = {
  Query: {
    user: async (_, { id }, context) => {
      // Ошибка аутентификации
      if (!context.user) {
        throw new AuthenticationError('Требуется авторизация');
      }

      const user = await context.db.users.findById(id);

      // Ошибка не найдено
      if (!user) {
        throw new ApolloError('Пользователь не найден', 'NOT_FOUND', {
          resourceId: id
        });
      }

      // Ошибка авторизации
      if (user.id !== context.user.id && context.user.role !== 'ADMIN') {
        throw new ForbiddenError('Нет доступа к этому пользователю');
      }

      return user;
    }
  },

  Mutation: {
    createPost: async (_, { input }, context) => {
      try {
        return await context.db.posts.create(input);
      } catch (error) {
        if (error.code === 'DUPLICATE_KEY') {
          throw new ApolloError(
            'Пост с таким заголовком уже существует',
            'DUPLICATE',
            { field: 'title' }
          );
        }
        throw error;
      }
    }
  }
};
```

### Partial errors

GraphQL возвращает данные даже при наличии ошибок.

```graphql
query {
  user(id: "1") {
    name
    secretField  # Ошибка авторизации
    posts {
      title
    }
  }
}
```

```json
{
  "data": {
    "user": {
      "name": "John",
      "secretField": null,
      "posts": [
        { "title": "Post 1" }
      ]
    }
  },
  "errors": [
    {
      "message": "Нет доступа к полю secretField",
      "path": ["user", "secretField"],
      "extensions": {
        "code": "FORBIDDEN"
      }
    }
  ]
}
```

## DataLoader и N+1 проблема

### Проблема N+1

```graphql
query {
  posts {          # 1 запрос к БД
    title
    author {       # N запросов к БД (по одному на каждый пост)
      name
    }
  }
}
```

```javascript
// Без DataLoader — N+1 запросов
const resolvers = {
  Post: {
    author: async (post, _, context) => {
      // Этот запрос выполнится для КАЖДОГО поста
      return await context.db.users.findById(post.authorId);
    }
  }
};
```

### Решение с DataLoader

```javascript
const DataLoader = require('dataloader');

// Создание DataLoader
const createUserLoader = (db) => new DataLoader(async (userIds) => {
  // Один запрос для всех пользователей
  const users = await db.users.findByIds(userIds);

  // Возвращаем в том же порядке, что и входные ID
  const userMap = new Map(users.map(u => [u.id, u]));
  return userIds.map(id => userMap.get(id) || null);
});

// Контекст с DataLoader
const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: () => ({
    loaders: {
      user: createUserLoader(db)
    }
  })
});

// Использование
const resolvers = {
  Post: {
    author: (post, _, context) => {
      // DataLoader автоматически батчит запросы
      return context.loaders.user.load(post.authorId);
    }
  }
};
```

### Как работает DataLoader

```
Запрос:
  posts[0].author → loader.load("user1")
  posts[1].author → loader.load("user2")
  posts[2].author → loader.load("user1")  # Дубликат
  posts[3].author → loader.load("user3")

DataLoader собирает уникальные ID:
  ["user1", "user2", "user3"]

Один запрос к БД:
  SELECT * FROM users WHERE id IN ('user1', 'user2', 'user3')

DataLoader распределяет результаты по запросам
```

## Middleware и Plugins

### Field Middleware

```javascript
const { makeExecutableSchema } = require('@graphql-tools/schema');
const { applyMiddleware } = require('graphql-middleware');

// Middleware для логирования
const loggerMiddleware = async (resolve, parent, args, context, info) => {
  const start = Date.now();

  const result = await resolve(parent, args, context, info);

  const duration = Date.now() - start;
  console.log(`${info.parentType.name}.${info.fieldName}: ${duration}ms`);

  return result;
};

// Middleware для кеширования
const cacheMiddleware = async (resolve, parent, args, context, info) => {
  const cacheKey = `${info.fieldName}:${JSON.stringify(args)}`;

  const cached = await context.cache.get(cacheKey);
  if (cached) {
    return cached;
  }

  const result = await resolve(parent, args, context, info);

  await context.cache.set(cacheKey, result, { ttl: 60 });

  return result;
};

// Применение middleware
const schemaWithMiddleware = applyMiddleware(
  schema,
  loggerMiddleware,
  cacheMiddleware
);
```

### Apollo Plugins

```javascript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    {
      // Перед выполнением запроса
      async requestDidStart(requestContext) {
        console.log('Request started:', requestContext.request.query);

        return {
          // После парсинга
          async parsingDidStart(context) {
            console.log('Parsing started');
          },

          // После валидации
          async validationDidStart(context) {
            console.log('Validation started');
          },

          // Перед выполнением
          async executionDidStart(context) {
            console.log('Execution started');

            return {
              // После выполнения каждого поля
              willResolveField({ info }) {
                const start = Date.now();
                return () => {
                  const duration = Date.now() - start;
                  if (duration > 100) {
                    console.log(`Slow field: ${info.fieldName} (${duration}ms)`);
                  }
                };
              }
            };
          },

          // При ошибках
          async didEncounterErrors(context) {
            console.log('Errors:', context.errors);
          },

          // После отправки ответа
          async willSendResponse(context) {
            console.log('Response sent');
          }
        };
      }
    }
  ]
});
```

## Оптимизация выполнения

### Ограничение глубины запроса

```javascript
const depthLimit = require('graphql-depth-limit');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [depthLimit(5)]
});
```

### Query Complexity

```javascript
const { createComplexityLimitRule } = require('graphql-validation-complexity');

const complexityRule = createComplexityLimitRule(1000, {
  scalarCost: 1,
  objectCost: 10,
  listFactor: 10
});

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [complexityRule]
});
```

### Таймауты

```javascript
const resolvers = {
  Query: {
    slowQuery: async () => {
      const timeout = new Promise((_, reject) =>
        setTimeout(() => reject(new Error('Timeout')), 5000)
      );

      const data = fetchSlowData();

      return Promise.race([data, timeout]);
    }
  }
};
```

## Отложенное выполнение (@defer)

```graphql
query {
  user(id: "1") {
    name
    email
    # Это поле будет отправлено позже
    ... @defer {
      posts {
        title
        content
      }
    }
  }
}
```

Ответ приходит частями:

```json
// Первая часть
{
  "data": {
    "user": {
      "name": "John",
      "email": "john@example.com"
    }
  },
  "hasNext": true
}

// Вторая часть
{
  "data": {
    "user": {
      "posts": [
        { "title": "Post 1", "content": "..." }
      ]
    }
  },
  "path": ["user"],
  "hasNext": false
}
```

## Практические советы

1. **Используйте DataLoader** — решает N+1 проблему
2. **Оптимизируйте запросы к БД** — анализируйте запрошенные поля
3. **Кешируйте результаты** — особенно для тяжёлых вычислений
4. **Ограничивайте сложность** — защита от злоупотреблений
5. **Логируйте медленные резолверы** — для оптимизации
6. **Используйте контекст правильно** — создавайте новый для каждого запроса
7. **Обрабатывайте ошибки gracefully** — возвращайте partial data

## Заключение

Понимание процесса выполнения запросов в GraphQL критически важно для построения производительных API. Резолверы, контекст, DataLoader и middleware — ключевые инструменты для эффективной обработки запросов. Правильная организация кода и оптимизация помогают избежать типичных проблем производительности.

---

[prev: 06-validation](./06-validation.md) | [next: 08-response](./08-response.md)
