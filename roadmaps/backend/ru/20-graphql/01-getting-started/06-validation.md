# Валидация в GraphQL

## Что такое валидация в GraphQL?

**Валидация** в GraphQL — это процесс проверки корректности запросов до их выполнения. GraphQL имеет встроенную систему валидации, которая проверяет запросы на соответствие схеме, а также позволяет добавлять пользовательскую логику для проверки входных данных.

## Уровни валидации

| Уровень | Описание | Когда происходит |
|---------|----------|------------------|
| Синтаксический | Проверка синтаксиса GraphQL | При парсинге запроса |
| Схемный | Соответствие запроса схеме | До выполнения |
| Бизнес-логика | Пользовательские правила | В резолверах |
| Input валидация | Проверка входных данных | В резолверах/директивах |

## Встроенная валидация GraphQL

### Синтаксическая валидация

```graphql
# Ошибка: незакрытая скобка
query {
  user(id: "1" {
    name
  }
}

# Ошибка: неверный синтаксис
query GetUser($id ID!) {  # Пропущено двоеточие
  user(id: $id) {
    name
  }
}
```

### Валидация схемы

GraphQL автоматически проверяет:

```graphql
# Схема
type User {
  id: ID!
  name: String!
  email: String!
  role: UserRole!
}

enum UserRole {
  ADMIN
  USER
}

type Query {
  user(id: ID!): User
}
```

```graphql
# Ошибка: поле не существует
query {
  user(id: "1") {
    username  # Нет такого поля, есть 'name'
  }
}

# Ошибка: неверный тип аргумента
query {
  user(id: 123) {  # id должен быть String (ID)
    name
  }
}

# Ошибка: пропущен обязательный аргумент
query {
  user {  # id обязателен
    name
  }
}

# Ошибка: неверное значение enum
query GetUsers($role: UserRole!) {
  users(role: $role) {
    name
  }
}
# С переменной role: "MODERATOR" — нет такого значения
```

## Правила валидации GraphQL

### 1. Уникальность имён операций

```graphql
# Ошибка: дублирование имени
query GetUser {
  user(id: "1") { name }
}

query GetUser {  # Имя уже используется
  user(id: "2") { name }
}
```

### 2. Валидация фрагментов

```graphql
# Ошибка: фрагмент на несуществующий тип
fragment UserFields on Person {  # Тип Person не определён
  name
  email
}

# Ошибка: неиспользуемый фрагмент
fragment UserFields on User {
  name
}

query {
  user(id: "1") {
    email  # Фрагмент не использован
  }
}

# Ошибка: циклическая ссылка фрагментов
fragment A on User {
  ...B
}

fragment B on User {
  ...A  # Циклическая ссылка
}
```

### 3. Валидация переменных

```graphql
# Ошибка: неиспользуемая переменная
query GetUser($id: ID!, $unused: String) {
  user(id: $id) {
    name
  }
}

# Ошибка: неопределённая переменная
query {
  user(id: $userId) {  # $userId не объявлена
    name
  }
}

# Ошибка: несовместимый тип переменной
query GetUser($id: String!) {  # Ожидается ID!
  user(id: $id) {
    name
  }
}
```

### 4. Валидация директив

```graphql
# Ошибка: директива не в том месте
query @include(if: true) {  # @include нельзя на операцию
  user(id: "1") {
    name
  }
}

# Ошибка: пропущен обязательный аргумент директивы
query {
  user(id: "1") {
    name @include  # Нужен аргумент if
  }
}
```

## Валидация входных данных

### Кастомные скаляры

```graphql
# Схема
scalar Email
scalar URL
scalar PositiveInt
scalar DateTime
scalar UUID
```

```javascript
const { GraphQLScalarType, Kind, GraphQLError } = require('graphql');

const EmailScalar = new GraphQLScalarType({
  name: 'Email',
  description: 'Валидный email адрес',

  serialize(value) {
    return value;
  },

  parseValue(value) {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(value)) {
      throw new GraphQLError(`Невалидный email: ${value}`);
    }
    return value.toLowerCase();
  },

  parseLiteral(ast) {
    if (ast.kind !== Kind.STRING) {
      throw new GraphQLError('Email должен быть строкой');
    }
    return this.parseValue(ast.value);
  }
});

const PositiveIntScalar = new GraphQLScalarType({
  name: 'PositiveInt',
  description: 'Целое положительное число',

  serialize(value) {
    return value;
  },

  parseValue(value) {
    if (!Number.isInteger(value) || value <= 0) {
      throw new GraphQLError('Значение должно быть положительным целым числом');
    }
    return value;
  },

  parseLiteral(ast) {
    if (ast.kind !== Kind.INT) {
      throw new GraphQLError('Значение должно быть целым числом');
    }
    const value = parseInt(ast.value, 10);
    if (value <= 0) {
      throw new GraphQLError('Значение должно быть положительным');
    }
    return value;
  }
});
```

### Директивы валидации

```graphql
# Определение директив
directive @length(min: Int, max: Int) on INPUT_FIELD_DEFINITION | ARGUMENT_DEFINITION
directive @range(min: Float, max: Float) on INPUT_FIELD_DEFINITION | ARGUMENT_DEFINITION
directive @pattern(regex: String!) on INPUT_FIELD_DEFINITION | ARGUMENT_DEFINITION

# Использование в схеме
input CreateUserInput {
  name: String! @length(min: 2, max: 50)
  email: String! @pattern(regex: "^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$")
  password: String! @length(min: 8, max: 100)
  age: Int @range(min: 0, max: 150)
}
```

```javascript
// Реализация директивы @length
const { mapSchema, getDirective, MapperKind } = require('@graphql-tools/utils');

function lengthDirective(directiveName) {
  return {
    lengthDirectiveTransformer: (schema) => mapSchema(schema, {
      [MapperKind.INPUT_OBJECT_FIELD]: (fieldConfig) => {
        const directive = getDirective(schema, fieldConfig, directiveName)?.[0];

        if (directive) {
          const { min, max } = directive;
          const { resolve = (v) => v } = fieldConfig;

          fieldConfig.resolve = async (source, args, context, info) => {
            const value = await resolve(source, args, context, info);

            if (typeof value === 'string') {
              if (min !== undefined && value.length < min) {
                throw new Error(`Минимальная длина: ${min} символов`);
              }
              if (max !== undefined && value.length > max) {
                throw new Error(`Максимальная длина: ${max} символов`);
              }
            }

            return value;
          };

          return fieldConfig;
        }
      }
    })
  };
}
```

## Валидация в резолверах

### Ручная валидация

```javascript
const resolvers = {
  Mutation: {
    createUser: async (_, { input }, context) => {
      const errors = [];

      // Валидация имени
      if (!input.name || input.name.trim().length < 2) {
        errors.push({
          field: 'name',
          message: 'Имя должно содержать минимум 2 символа',
          code: 'VALIDATION_ERROR'
        });
      }

      // Валидация email
      const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
      if (!emailRegex.test(input.email)) {
        errors.push({
          field: 'email',
          message: 'Невалидный формат email',
          code: 'VALIDATION_ERROR'
        });
      }

      // Проверка уникальности email
      const existingUser = await context.db.user.findByEmail(input.email);
      if (existingUser) {
        errors.push({
          field: 'email',
          message: 'Пользователь с таким email уже существует',
          code: 'DUPLICATE_ERROR'
        });
      }

      // Валидация пароля
      if (input.password.length < 8) {
        errors.push({
          field: 'password',
          message: 'Пароль должен содержать минимум 8 символов',
          code: 'VALIDATION_ERROR'
        });
      }

      // Возврат ошибок или создание пользователя
      if (errors.length > 0) {
        return {
          success: false,
          errors,
          user: null
        };
      }

      const user = await context.db.user.create(input);
      return {
        success: true,
        errors: [],
        user
      };
    }
  }
};
```

### Использование библиотек валидации

#### Joi

```javascript
const Joi = require('joi');

const createUserSchema = Joi.object({
  name: Joi.string().min(2).max(50).required()
    .messages({
      'string.min': 'Имя должно содержать минимум {#limit} символа',
      'string.max': 'Имя не должно превышать {#limit} символов',
      'any.required': 'Имя обязательно'
    }),
  email: Joi.string().email().required()
    .messages({
      'string.email': 'Невалидный формат email',
      'any.required': 'Email обязателен'
    }),
  password: Joi.string().min(8).max(100).required()
    .pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
    .messages({
      'string.min': 'Пароль должен содержать минимум {#limit} символов',
      'string.pattern.base': 'Пароль должен содержать буквы в разном регистре и цифры'
    }),
  age: Joi.number().integer().min(0).max(150).optional()
});

const resolvers = {
  Mutation: {
    createUser: async (_, { input }) => {
      const { error, value } = createUserSchema.validate(input, {
        abortEarly: false // Собрать все ошибки
      });

      if (error) {
        const errors = error.details.map(detail => ({
          field: detail.path.join('.'),
          message: detail.message,
          code: 'VALIDATION_ERROR'
        }));

        return { success: false, errors, user: null };
      }

      // Использовать value — валидированные и преобразованные данные
      const user = await createUser(value);
      return { success: true, errors: [], user };
    }
  }
};
```

#### Yup

```javascript
const yup = require('yup');

const createUserSchema = yup.object({
  name: yup.string()
    .min(2, 'Имя должно содержать минимум 2 символа')
    .max(50, 'Имя не должно превышать 50 символов')
    .required('Имя обязательно'),
  email: yup.string()
    .email('Невалидный формат email')
    .required('Email обязателен'),
  password: yup.string()
    .min(8, 'Пароль должен содержать минимум 8 символов')
    .matches(
      /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
      'Пароль должен содержать буквы в разном регистре и цифры'
    )
    .required('Пароль обязателен'),
  confirmPassword: yup.string()
    .oneOf([yup.ref('password')], 'Пароли должны совпадать')
    .required('Подтверждение пароля обязательно')
});

const resolvers = {
  Mutation: {
    createUser: async (_, { input }) => {
      try {
        const validatedInput = await createUserSchema.validate(input, {
          abortEarly: false
        });

        const user = await createUser(validatedInput);
        return { success: true, errors: [], user };

      } catch (err) {
        if (err instanceof yup.ValidationError) {
          const errors = err.inner.map(e => ({
            field: e.path,
            message: e.message,
            code: 'VALIDATION_ERROR'
          }));

          return { success: false, errors, user: null };
        }
        throw err;
      }
    }
  }
};
```

#### Zod (TypeScript)

```typescript
import { z } from 'zod';

const createUserSchema = z.object({
  name: z.string()
    .min(2, 'Имя должно содержать минимум 2 символа')
    .max(50, 'Имя не должно превышать 50 символов'),
  email: z.string()
    .email('Невалидный формат email'),
  password: z.string()
    .min(8, 'Пароль должен содержать минимум 8 символов')
    .regex(
      /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
      'Пароль должен содержать буквы в разном регистре и цифры'
    ),
  role: z.enum(['USER', 'ADMIN']).default('USER'),
  profile: z.object({
    bio: z.string().max(500).optional(),
    website: z.string().url().optional()
  }).optional()
});

type CreateUserInput = z.infer<typeof createUserSchema>;

const resolvers = {
  Mutation: {
    createUser: async (_: any, { input }: { input: unknown }) => {
      const result = createUserSchema.safeParse(input);

      if (!result.success) {
        const errors = result.error.errors.map(e => ({
          field: e.path.join('.'),
          message: e.message,
          code: 'VALIDATION_ERROR'
        }));

        return { success: false, errors, user: null };
      }

      const user = await createUser(result.data);
      return { success: true, errors: [], user };
    }
  }
};
```

## Централизованная валидация

### Middleware подход

```javascript
const validationSchemas = {
  createUser: createUserSchema,
  updateUser: updateUserSchema,
  createPost: createPostSchema
};

function validationMiddleware(resolve, root, args, context, info) {
  const operationName = info.fieldName;
  const schema = validationSchemas[operationName];

  if (schema && args.input) {
    const { error, value } = schema.validate(args.input, {
      abortEarly: false
    });

    if (error) {
      const errors = error.details.map(d => ({
        field: d.path.join('.'),
        message: d.message
      }));

      // Возвращаем payload с ошибками
      return { success: false, errors, [operationName.replace('create', '').toLowerCase()]: null };
    }

    // Заменяем input валидированными данными
    args.input = value;
  }

  return resolve(root, args, context, info);
}

// Применение middleware
const schemaWithMiddleware = applyMiddleware(schema, validationMiddleware);
```

## Типизированные ошибки валидации

### Схема с типами ошибок

```graphql
interface Error {
  message: String!
  code: ErrorCode!
}

type ValidationError implements Error {
  message: String!
  code: ErrorCode!
  field: String!
  constraints: [String!]!
}

type AuthenticationError implements Error {
  message: String!
  code: ErrorCode!
}

type NotFoundError implements Error {
  message: String!
  code: ErrorCode!
  resourceType: String!
  resourceId: ID!
}

enum ErrorCode {
  VALIDATION_ERROR
  AUTHENTICATION_ERROR
  AUTHORIZATION_ERROR
  NOT_FOUND
  DUPLICATE
  INTERNAL_ERROR
}

type CreateUserPayload {
  user: User
  errors: [Error!]!
}

union CreateUserResult = User | ValidationError | AuthenticationError
```

### Формат ответа с ошибками

```json
{
  "data": {
    "createUser": {
      "user": null,
      "errors": [
        {
          "message": "Email уже используется",
          "code": "DUPLICATE",
          "field": "email",
          "constraints": ["unique"]
        },
        {
          "message": "Пароль слишком короткий",
          "code": "VALIDATION_ERROR",
          "field": "password",
          "constraints": ["min:8"]
        }
      ]
    }
  }
}
```

## Валидация сложных структур

### Вложенные объекты

```javascript
const createOrderSchema = Joi.object({
  customerId: Joi.string().uuid().required(),
  items: Joi.array().items(
    Joi.object({
      productId: Joi.string().uuid().required(),
      quantity: Joi.number().integer().min(1).required(),
      options: Joi.object({
        size: Joi.string().valid('S', 'M', 'L', 'XL'),
        color: Joi.string()
      }).optional()
    })
  ).min(1).required(),
  shippingAddress: Joi.object({
    street: Joi.string().required(),
    city: Joi.string().required(),
    postalCode: Joi.string().pattern(/^\d{6}$/).required(),
    country: Joi.string().length(2).required()
  }).required(),
  promoCode: Joi.string().optional()
});
```

### Условная валидация

```javascript
const paymentSchema = Joi.object({
  method: Joi.string().valid('card', 'bank_transfer', 'crypto').required(),

  // Поля для карты
  cardNumber: Joi.string().when('method', {
    is: 'card',
    then: Joi.string().creditCard().required(),
    otherwise: Joi.forbidden()
  }),
  cardExpiry: Joi.string().when('method', {
    is: 'card',
    then: Joi.string().pattern(/^\d{2}\/\d{2}$/).required(),
    otherwise: Joi.forbidden()
  }),

  // Поля для банковского перевода
  bankAccount: Joi.string().when('method', {
    is: 'bank_transfer',
    then: Joi.string().required(),
    otherwise: Joi.forbidden()
  }),

  // Поля для криптовалюты
  walletAddress: Joi.string().when('method', {
    is: 'crypto',
    then: Joi.string().required(),
    otherwise: Joi.forbidden()
  })
});
```

## Query Complexity Validation

Защита от слишком сложных запросов.

```javascript
const { createComplexityLimitRule } = require('graphql-validation-complexity');

const complexityLimitRule = createComplexityLimitRule(1000, {
  onCost: (cost) => {
    console.log('Query complexity:', cost);
  },
  formatErrorMessage: (cost) =>
    `Запрос слишком сложный: ${cost}. Максимум: 1000`
});

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [complexityLimitRule]
});
```

```graphql
# Схема с указанием стоимости
type Query {
  users(first: Int = 10): [User!]! @complexity(value: 10, multipliers: ["first"])
  posts(first: Int = 10): [Post!]! @complexity(value: 5, multipliers: ["first"])
}

type User {
  id: ID!
  name: String!
  posts: [Post!]! @complexity(value: 5)
  followers: [User!]! @complexity(value: 10)
}
```

## Практические советы

### Рекомендации по валидации

| Что валидировать | Где валидировать |
|------------------|------------------|
| Формат данных | Кастомные скаляры |
| Длина строк | Директивы или резолверы |
| Диапазоны чисел | Директивы или резолверы |
| Бизнес-правила | Резолверы |
| Уникальность | Резолверы (с БД) |
| Сложность запроса | Validation rules |

### Чек-лист валидации

1. **Используйте Non-null типы** — первая линия защиты
2. **Создавайте кастомные скаляры** — для часто используемых форматов
3. **Применяйте директивы** — для декларативной валидации
4. **Используйте библиотеки** — Joi, Yup, Zod проверены временем
5. **Возвращайте typed errors** — клиенты смогут обработать ошибки
6. **Валидируйте глубину запросов** — защита от DoS
7. **Логируйте ошибки валидации** — для мониторинга

## Заключение

Валидация в GraphQL — многоуровневый процесс, начинающийся со встроенной проверки синтаксиса и схемы и заканчивающийся пользовательской бизнес-логикой. Правильная организация валидации обеспечивает безопасность API, улучшает пользовательский опыт и упрощает отладку. Комбинация кастомных скаляров, директив и библиотек валидации позволяет построить надёжную систему проверки данных.
