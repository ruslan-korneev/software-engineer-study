# Авторизация в GraphQL

[prev: 04-file-uploads](./04-file-uploads.md) | [next: 06-pagination](./06-pagination.md)

---

## Введение

**Авторизация** — это процесс определения того, какие действия может выполнять пользователь. В GraphQL авторизация особенно важна, так как клиенты могут запрашивать любые поля схемы. Неправильная реализация может привести к утечке данных.

## Аутентификация vs Авторизация

| Аспект | Аутентификация | Авторизация |
|--------|----------------|-------------|
| Вопрос | Кто вы? | Что вам можно? |
| Когда | До авторизации | После аутентификации |
| Данные | Токены, сессии | Роли, разрешения |
| Пример | JWT валидация | Проверка доступа к ресурсу |

## Уровни авторизации

### 1. Уровень эндпоинта (HTTP)

```javascript
// Middleware для аутентификации
app.use('/graphql', async (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];

  if (token) {
    try {
      const user = await verifyToken(token);
      req.user = user;
    } catch (error) {
      // Токен невалиден, но запрос продолжаем
      // Некоторые операции могут быть публичными
    }
  }

  next();
});
```

### 2. Уровень операции

```javascript
const resolvers = {
  Mutation: {
    createPost: (_, args, context) => {
      if (!context.user) {
        throw new GraphQLError('Authentication required', {
          extensions: { code: 'UNAUTHENTICATED' },
        });
      }
      return db.posts.create({ ...args, authorId: context.user.id });
    },
  },
};
```

### 3. Уровень типа/поля

```javascript
const resolvers = {
  User: {
    email: (user, _, context) => {
      // Email видит только сам пользователь
      if (context.user?.id !== user.id) {
        return null;
      }
      return user.email;
    },

    privateData: (user, _, context) => {
      if (!context.user || context.user.id !== user.id) {
        throw new GraphQLError('Access denied', {
          extensions: { code: 'FORBIDDEN' },
        });
      }
      return user.privateData;
    },
  },
};
```

### 4. Уровень данных

```javascript
const resolvers = {
  Query: {
    posts: async (_, args, context) => {
      // Пользователь видит только опубликованные посты
      // или свои черновики
      const where = {
        OR: [
          { status: 'PUBLISHED' },
          { authorId: context.user?.id },
        ],
      };
      return db.posts.findMany({ where });
    },
  },
};
```

## Паттерны авторизации

### 1. Авторизация в резолверах (простой подход)

```javascript
const resolvers = {
  Mutation: {
    deletePost: async (_, { id }, context) => {
      const post = await db.posts.findById(id);

      if (!post) {
        throw new GraphQLError('Post not found');
      }

      // Проверка владельца
      if (post.authorId !== context.user?.id) {
        throw new GraphQLError('Not authorized');
      }

      // Или проверка роли
      if (!context.user?.roles.includes('ADMIN')) {
        throw new GraphQLError('Admin access required');
      }

      return db.posts.delete(id);
    },
  },
};
```

### 2. Авторизация через директивы

```graphql
directive @auth(
  requires: Role = USER
) on OBJECT | FIELD_DEFINITION

enum Role {
  ADMIN
  USER
  GUEST
}

type Query {
  publicPosts: [Post!]!
  myPosts: [Post!]! @auth(requires: USER)
  allUsers: [User!]! @auth(requires: ADMIN)
}

type User {
  id: ID!
  name: String!
  email: String! @auth(requires: USER)
  secretData: String @auth(requires: ADMIN)
}
```

### Реализация директивы

```javascript
import { mapSchema, getDirective, MapperKind } from '@graphql-tools/utils';
import { defaultFieldResolver } from 'graphql';

function authDirectiveTransformer(schema) {
  return mapSchema(schema, {
    [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
      const authDirective = getDirective(schema, fieldConfig, 'auth')?.[0];

      if (authDirective) {
        const { requires } = authDirective;
        const originalResolve = fieldConfig.resolve || defaultFieldResolver;

        fieldConfig.resolve = async (source, args, context, info) => {
          const user = context.user;

          if (!user) {
            throw new GraphQLError('Authentication required', {
              extensions: { code: 'UNAUTHENTICATED' },
            });
          }

          const userRole = user.role || 'GUEST';
          const roles = ['GUEST', 'USER', 'ADMIN'];
          const requiredIndex = roles.indexOf(requires);
          const userIndex = roles.indexOf(userRole);

          if (userIndex < requiredIndex) {
            throw new GraphQLError('Insufficient permissions', {
              extensions: { code: 'FORBIDDEN' },
            });
          }

          return originalResolve(source, args, context, info);
        };
      }

      return fieldConfig;
    },
  });
}
```

### 3. Авторизация через слой бизнес-логики

```javascript
// services/PostService.js
class PostService {
  constructor(context) {
    this.user = context.user;
    this.db = context.db;
  }

  async getPost(id) {
    const post = await this.db.posts.findById(id);

    if (!post) return null;

    // Авторизация в сервисе
    if (post.status === 'DRAFT' && post.authorId !== this.user?.id) {
      return null;
    }

    return post;
  }

  async deletePost(id) {
    const post = await this.db.posts.findById(id);

    if (!post) {
      throw new Error('Post not found');
    }

    if (post.authorId !== this.user?.id && !this.user?.isAdmin) {
      throw new Error('Not authorized to delete this post');
    }

    return this.db.posts.delete(id);
  }
}

// Резолверы используют сервис
const resolvers = {
  Query: {
    post: (_, { id }, context) => {
      return new PostService(context).getPost(id);
    },
  },
  Mutation: {
    deletePost: (_, { id }, context) => {
      return new PostService(context).deletePost(id);
    },
  },
};
```

### 4. Авторизация с graphql-shield

```javascript
import { shield, rule, and, or, not } from 'graphql-shield';

// Правила
const isAuthenticated = rule()((parent, args, context) => {
  return context.user !== null;
});

const isAdmin = rule()((parent, args, context) => {
  return context.user?.role === 'ADMIN';
});

const isOwner = rule()((parent, args, context) => {
  return parent.authorId === context.user?.id;
});

// Разрешения
const permissions = shield({
  Query: {
    '*': isAuthenticated,
    publicPosts: true, // Публичный доступ
  },
  Mutation: {
    createPost: isAuthenticated,
    deletePost: or(isAdmin, isOwner),
    updateUser: and(isAuthenticated, isOwner),
  },
  User: {
    email: or(isOwner, isAdmin),
    password: not(true), // Никогда не показывать
  },
});

// Применение
const server = new ApolloServer({
  schema: applyMiddleware(schema, permissions),
});
```

## Передача контекста

### Настройка контекста

```javascript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: async ({ req }) => {
    // Получаем токен
    const token = req.headers.authorization?.replace('Bearer ', '');

    // Аутентификация
    let user = null;
    if (token) {
      try {
        const payload = jwt.verify(token, process.env.JWT_SECRET);
        user = await db.users.findById(payload.userId);
      } catch (error) {
        // Токен невалиден
      }
    }

    return {
      user,
      db,
      loaders: createLoaders(user), // DataLoaders с учётом пользователя
    };
  },
});
```

### Типизированный контекст (TypeScript)

```typescript
interface Context {
  user: User | null;
  db: Database;
  loaders: DataLoaders;
}

interface User {
  id: string;
  email: string;
  role: 'ADMIN' | 'USER' | 'GUEST';
  permissions: string[];
}

// Резолвер с типизацией
const resolvers: Resolvers<Context> = {
  Query: {
    me: (_, __, context) => {
      return context.user; // TypeScript знает тип
    },
  },
};
```

## RBAC (Role-Based Access Control)

### Схема с ролями

```graphql
enum Role {
  ADMIN
  MODERATOR
  USER
  GUEST
}

type User {
  id: ID!
  role: Role!
}

type Query {
  # Доступно всем аутентифицированным
  me: User @auth

  # Только для модераторов и выше
  reportedPosts: [Post!]! @auth(roles: [ADMIN, MODERATOR])

  # Только для админов
  users: [User!]! @auth(roles: [ADMIN])
}
```

### Реализация проверки ролей

```javascript
const roleHierarchy = {
  ADMIN: 4,
  MODERATOR: 3,
  USER: 2,
  GUEST: 1,
};

function hasRole(user, requiredRoles) {
  if (!user) return false;

  return requiredRoles.some(
    (role) => roleHierarchy[user.role] >= roleHierarchy[role]
  );
}

function hasPermission(user, permission) {
  if (!user) return false;
  return user.permissions.includes(permission);
}
```

## Авторизация на уровне полей

### Фильтрация данных

```javascript
const resolvers = {
  User: {
    email: (user, _, context) => {
      // Показываем email только:
      // 1. Самому пользователю
      // 2. Админу
      // 3. Если пользователь разрешил
      if (
        context.user?.id === user.id ||
        context.user?.role === 'ADMIN' ||
        user.emailPublic
      ) {
        return user.email;
      }
      return null;
    },

    // Полностью скрываем поле
    secretNotes: (user, _, context) => {
      if (context.user?.id !== user.id) {
        throw new GraphQLError('Access denied');
      }
      return user.secretNotes;
    },
  },
};
```

### Условные поля в схеме

```graphql
type User {
  id: ID!
  name: String!
  publicEmail: String    # Всегда доступно (если установлено)
  privateEmail: String   # Только владельцу
  adminNotes: String     # Только админам
}
```

## Обработка ошибок авторизации

```javascript
import { GraphQLError } from 'graphql';

class AuthenticationError extends GraphQLError {
  constructor(message = 'Authentication required') {
    super(message, {
      extensions: {
        code: 'UNAUTHENTICATED',
        http: { status: 401 },
      },
    });
  }
}

class ForbiddenError extends GraphQLError {
  constructor(message = 'Access denied') {
    super(message, {
      extensions: {
        code: 'FORBIDDEN',
        http: { status: 403 },
      },
    });
  }
}

// Использование
if (!context.user) {
  throw new AuthenticationError();
}

if (!hasPermission(context.user, 'DELETE_POST')) {
  throw new ForbiddenError('Cannot delete posts');
}
```

## Лучшие практики

1. **Авторизация в бизнес-логике** — не только в резолверах
2. **Принцип минимальных привилегий** — давайте только необходимые права
3. **Централизованная логика** — используйте сервисы или middleware
4. **Логирование** — записывайте попытки несанкционированного доступа
5. **Тестирование** — пишите тесты для правил авторизации
6. **Не доверяйте клиенту** — проверяйте права на каждый запрос

## Заключение

Авторизация в GraphQL требует внимательного подхода из-за гибкости запросов. Используйте многоуровневую защиту: на уровне HTTP, операций, типов и данных. Выберите подходящий паттерн (директивы, shield, сервисы) в зависимости от сложности вашего приложения.

---

[prev: 04-file-uploads](./04-file-uploads.md) | [next: 06-pagination](./06-pagination.md)
