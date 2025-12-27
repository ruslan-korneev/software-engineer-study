# Безопасность GraphQL

## Введение

GraphQL открывает новые векторы атак по сравнению с REST API. Гибкость запросов, интроспекция схемы и возможность сложных вложенных запросов требуют особого внимания к безопасности.

## Основные угрозы

| Угроза | Описание | Последствия |
|--------|----------|-------------|
| DoS через сложные запросы | Глубокие вложенные запросы | Перегрузка сервера |
| Интроспекция | Раскрытие структуры API | Утечка информации |
| Batching атаки | Множество операций в одном запросе | Обход rate limiting |
| Инъекции | Вредоносные данные в переменных | SQL/NoSQL инъекции |
| IDOR | Прямой доступ к объектам | Несанкционированный доступ |

## Защита от DoS-атак

### Ограничение глубины запросов

```javascript
import depthLimit from 'graphql-depth-limit';

const server = new ApolloServer({
  validationRules: [
    depthLimit(7), // Максимум 7 уровней вложенности
  ],
});
```

### Ограничение сложности

```javascript
import {
  createComplexityLimitRule,
  fieldExtensionsEstimator,
  simpleEstimator,
} from 'graphql-query-complexity';

const complexityLimitRule = createComplexityLimitRule(1000, {
  estimators: [
    fieldExtensionsEstimator(),
    simpleEstimator({ defaultComplexity: 1 }),
  ],
  onComplete: (complexity) => {
    console.log('Query Complexity:', complexity);
  },
});
```

### Таймауты выполнения

```javascript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    {
      async requestDidStart() {
        const timeout = setTimeout(() => {
          throw new Error('Query timeout exceeded');
        }, 30000); // 30 секунд

        return {
          async willSendResponse() {
            clearTimeout(timeout);
          },
        };
      },
    },
  ],
});
```

### Rate Limiting

```javascript
import rateLimit from 'express-rate-limit';
import { RedisStore } from 'rate-limit-redis';

// Общий rate limit
app.use('/graphql', rateLimit({
  windowMs: 15 * 60 * 1000, // 15 минут
  max: 100, // 100 запросов на IP
  store: new RedisStore({ client: redisClient }),
}));

// Rate limit по операциям
const operationLimiter = {
  requestDidStart({ request, context }) {
    const key = `ratelimit:${context.user?.id || context.ip}`;
    const limit = 10; // 10 запросов в минуту

    const count = await redis.incr(key);
    if (count === 1) {
      await redis.expire(key, 60);
    }

    if (count > limit) {
      throw new GraphQLError('Rate limit exceeded', {
        extensions: { code: 'RATE_LIMITED' },
      });
    }
  },
};
```

### Ограничение batching

```javascript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  allowBatchedHttpRequests: false, // Отключаем batching

  // Или ограничиваем количество операций
  plugins: [{
    requestDidStart({ request }) {
      if (Array.isArray(request.query) && request.query.length > 5) {
        throw new GraphQLError('Too many batched operations');
      }
    },
  }],
});
```

## Отключение интроспекции

```javascript
import { NoSchemaIntrospectionCustomRule } from 'graphql';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: process.env.NODE_ENV === 'production'
    ? [NoSchemaIntrospectionCustomRule]
    : [],
});
```

### Частичная интроспекция

```javascript
// Скрываем определённые типы
const schema = wrapSchema({
  schema: originalSchema,
  transforms: [
    new FilterTypes((type) => {
      // Скрываем внутренние типы
      return !type.name.startsWith('Internal');
    }),
  ],
});
```

## Валидация входных данных

### Валидация в резолверах

```javascript
import { UserInputError } from '@apollo/server';
import { z } from 'zod';

const CreateUserSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).max(100),
  name: z.string().min(1).max(100),
});

const resolvers = {
  Mutation: {
    createUser: async (_, { input }) => {
      // Валидация
      const result = CreateUserSchema.safeParse(input);
      if (!result.success) {
        throw new GraphQLError('Validation failed', {
          extensions: {
            code: 'VALIDATION_ERROR',
            errors: result.error.errors,
          },
        });
      }

      return db.users.create(result.data);
    },
  },
};
```

### Директива @constraint

```graphql
directive @constraint(
  minLength: Int
  maxLength: Int
  pattern: String
  format: String
  min: Float
  max: Float
) on INPUT_FIELD_DEFINITION | ARGUMENT_DEFINITION

input CreateUserInput {
  email: String! @constraint(format: "email", maxLength: 255)
  password: String! @constraint(minLength: 8, maxLength: 100)
  age: Int @constraint(min: 0, max: 150)
}
```

### Санитизация данных

```javascript
import xss from 'xss';

const resolvers = {
  Mutation: {
    createPost: async (_, { input }) => {
      // Санитизация HTML
      const sanitizedContent = xss(input.content, {
        whiteList: {
          p: [],
          br: [],
          strong: [],
          em: [],
          a: ['href', 'title'],
        },
      });

      return db.posts.create({
        ...input,
        content: sanitizedContent,
      });
    },
  },
};
```

## Защита от SQL/NoSQL инъекций

### Параметризованные запросы

```javascript
// Плохо: конкатенация строк
const user = await db.query(`
  SELECT * FROM users WHERE id = '${id}'
`);

// Хорошо: параметризованные запросы
const user = await db.query(`
  SELECT * FROM users WHERE id = $1
`, [id]);

// С ORM (Prisma)
const user = await prisma.user.findUnique({
  where: { id }, // Автоматическая защита
});
```

### Валидация ID

```javascript
import { GraphQLError } from 'graphql';

function validateId(id) {
  // UUID формат
  if (!/^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i.test(id)) {
    throw new GraphQLError('Invalid ID format');
  }
  return id;
}

const resolvers = {
  Query: {
    user: (_, { id }) => {
      validateId(id);
      return db.users.findById(id);
    },
  },
};
```

## IDOR (Insecure Direct Object References)

### Проверка прав доступа

```javascript
const resolvers = {
  Query: {
    order: async (_, { id }, context) => {
      const order = await db.orders.findById(id);

      if (!order) {
        return null;
      }

      // Проверка владельца
      if (order.userId !== context.user?.id && !context.user?.isAdmin) {
        throw new GraphQLError('Access denied', {
          extensions: { code: 'FORBIDDEN' },
        });
      }

      return order;
    },
  },

  Mutation: {
    updateOrder: async (_, { id, input }, context) => {
      const order = await db.orders.findById(id);

      if (order.userId !== context.user?.id) {
        throw new GraphQLError('Access denied');
      }

      return db.orders.update(id, input);
    },
  },
};
```

### Фильтрация на уровне данных

```javascript
const resolvers = {
  Query: {
    myOrders: async (_, args, context) => {
      // Пользователь видит только свои заказы
      return db.orders.findMany({
        where: { userId: context.user.id },
      });
    },
  },

  User: {
    orders: async (user, _, context) => {
      // Можно видеть заказы только если это свой профиль
      if (user.id !== context.user?.id && !context.user?.isAdmin) {
        return [];
      }

      return db.orders.findMany({
        where: { userId: user.id },
      });
    },
  },
};
```

## Защита чувствительных данных

### Маскирование данных

```javascript
const resolvers = {
  User: {
    email: (user, _, context) => {
      // Полный email только владельцу
      if (context.user?.id === user.id) {
        return user.email;
      }

      // Маскируем для других
      const [name, domain] = user.email.split('@');
      return `${name[0]}***@${domain}`;
    },

    phone: (user, _, context) => {
      if (context.user?.id !== user.id) {
        return null; // Не показываем телефон другим
      }
      return user.phone;
    },
  },
};
```

### Логирование без чувствительных данных

```javascript
const loggingPlugin = {
  requestDidStart({ request }) {
    // Не логируем пароли
    const sanitizedVariables = { ...request.variables };
    if (sanitizedVariables.password) {
      sanitizedVariables.password = '[REDACTED]';
    }

    console.log('GraphQL Request:', {
      operationName: request.operationName,
      variables: sanitizedVariables,
    });
  },
};
```

## CORS и заголовки безопасности

```javascript
import cors from 'cors';
import helmet from 'helmet';

// CORS
app.use('/graphql', cors({
  origin: ['https://app.example.com'],
  credentials: true,
  methods: ['POST', 'GET'],
}));

// Security headers
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
    },
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
  },
}));
```

## Аудит и мониторинг

### Логирование операций

```javascript
const auditPlugin = {
  requestDidStart({ request, context }) {
    return {
      didResolveOperation({ operationName, operation }) {
        // Логируем все мутации
        if (operation.operation === 'mutation') {
          auditLog.log({
            userId: context.user?.id,
            operation: operationName,
            timestamp: new Date(),
            ip: context.ip,
          });
        }
      },

      didEncounterErrors({ errors }) {
        // Логируем ошибки авторизации
        errors
          .filter(e => e.extensions?.code === 'FORBIDDEN')
          .forEach(e => {
            securityLog.warn({
              message: 'Unauthorized access attempt',
              userId: context.user?.id,
              error: e.message,
            });
          });
      },
    };
  },
};
```

### Детекция аномалий

```javascript
const anomalyDetector = {
  async requestDidStart({ context }) {
    const recentRequests = await redis.get(`requests:${context.user?.id}`);

    if (recentRequests > 1000) {
      securityLog.alert({
        message: 'Unusual request volume',
        userId: context.user?.id,
      });
    }
  },
};
```

## Чек-лист безопасности

- [ ] Ограничена глубина запросов
- [ ] Настроен rate limiting
- [ ] Отключена интроспекция в production
- [ ] Валидация всех входных данных
- [ ] Параметризованные запросы к БД
- [ ] Проверка прав доступа к объектам
- [ ] Маскирование чувствительных данных
- [ ] Настроены CORS и security headers
- [ ] Логирование и мониторинг
- [ ] Батчинг ограничен или отключён

## Заключение

Безопасность GraphQL требует комплексного подхода. Защита от DoS, правильная авторизация, валидация данных и мониторинг — все эти аспекты должны быть реализованы для создания безопасного API.
