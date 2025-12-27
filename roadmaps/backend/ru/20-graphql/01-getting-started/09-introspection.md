# Интроспекция в GraphQL

## Что такое интроспекция?

**Интроспекция (Introspection)** — это возможность запрашивать информацию о самой схеме GraphQL. Клиенты могут узнать какие типы, поля, запросы и мутации доступны в API, не обращаясь к внешней документации. Это одна из ключевых особенностей GraphQL, делающая его самодокументируемым.

## Зачем нужна интроспекция?

| Применение | Описание |
|------------|----------|
| IDE и инструменты | GraphiQL, Apollo Studio автоматически строят интерфейс |
| Генерация кода | Создание типов TypeScript из схемы |
| Валидация запросов | Проверка запросов на клиенте до отправки |
| Документация | Автоматическая генерация документации |
| Тестирование | Проверка структуры схемы |

## Системные типы интроспекции

GraphQL определяет специальные типы с префиксом `__`:

```graphql
# Системные типы
__Schema
__Type
__Field
__InputValue
__EnumValue
__Directive
__DirectiveLocation
__TypeKind
```

## Основные запросы интроспекции

### Получение схемы

```graphql
query IntrospectionQuery {
  __schema {
    queryType { name }
    mutationType { name }
    subscriptionType { name }
    types {
      name
      kind
      description
    }
    directives {
      name
      description
      locations
      args {
        name
        type { name }
      }
    }
  }
}
```

### Информация о типе

```graphql
query TypeInfo {
  __type(name: "User") {
    name
    kind
    description
    fields {
      name
      description
      type {
        name
        kind
        ofType {
          name
          kind
        }
      }
      args {
        name
        type { name }
        defaultValue
      }
    }
  }
}
```

### Ответ для типа User

```json
{
  "data": {
    "__type": {
      "name": "User",
      "kind": "OBJECT",
      "description": "Пользователь системы",
      "fields": [
        {
          "name": "id",
          "description": "Уникальный идентификатор",
          "type": {
            "name": null,
            "kind": "NON_NULL",
            "ofType": {
              "name": "ID",
              "kind": "SCALAR"
            }
          },
          "args": []
        },
        {
          "name": "name",
          "description": "Имя пользователя",
          "type": {
            "name": null,
            "kind": "NON_NULL",
            "ofType": {
              "name": "String",
              "kind": "SCALAR"
            }
          },
          "args": []
        },
        {
          "name": "posts",
          "description": "Посты пользователя",
          "type": {
            "name": null,
            "kind": "NON_NULL",
            "ofType": {
              "name": null,
              "kind": "LIST"
            }
          },
          "args": [
            {
              "name": "first",
              "type": { "name": "Int" },
              "defaultValue": "10"
            }
          ]
        }
      ]
    }
  }
}
```

## TypeKind (Виды типов)

```graphql
enum __TypeKind {
  SCALAR
  OBJECT
  INTERFACE
  UNION
  ENUM
  INPUT_OBJECT
  LIST
  NON_NULL
}
```

| Kind | Описание | Пример |
|------|----------|--------|
| `SCALAR` | Скалярный тип | `String`, `Int`, `Boolean` |
| `OBJECT` | Объектный тип | `User`, `Post` |
| `INTERFACE` | Интерфейс | `Node`, `Timestamped` |
| `UNION` | Union тип | `SearchResult` |
| `ENUM` | Перечисление | `UserRole`, `Status` |
| `INPUT_OBJECT` | Input тип | `CreateUserInput` |
| `LIST` | Список | `[User]` |
| `NON_NULL` | Non-null обёртка | `User!` |

## Полный запрос интроспекции

```graphql
query IntrospectionQuery {
  __schema {
    queryType { name }
    mutationType { name }
    subscriptionType { name }
    types {
      ...FullType
    }
    directives {
      name
      description
      locations
      args {
        ...InputValue
      }
    }
  }
}

fragment FullType on __Type {
  kind
  name
  description
  fields(includeDeprecated: true) {
    name
    description
    args {
      ...InputValue
    }
    type {
      ...TypeRef
    }
    isDeprecated
    deprecationReason
  }
  inputFields {
    ...InputValue
  }
  interfaces {
    ...TypeRef
  }
  enumValues(includeDeprecated: true) {
    name
    description
    isDeprecated
    deprecationReason
  }
  possibleTypes {
    ...TypeRef
  }
}

fragment InputValue on __InputValue {
  name
  description
  type {
    ...TypeRef
  }
  defaultValue
}

fragment TypeRef on __Type {
  kind
  name
  ofType {
    kind
    name
    ofType {
      kind
      name
      ofType {
        kind
        name
        ofType {
          kind
          name
          ofType {
            kind
            name
            ofType {
              kind
              name
            }
          }
        }
      }
    }
  }
}
```

## Использование интроспекции

### GraphiQL и Apollo Studio

Эти инструменты автоматически выполняют интроспекцию для:
- Автодополнения при написании запросов
- Отображения документации в панели
- Валидации запросов в реальном времени

### Генерация TypeScript типов

```bash
# Apollo CLI
apollo codegen:generate --target=typescript

# GraphQL Code Generator
npx graphql-codegen
```

```yaml
# codegen.yml
schema: http://localhost:4000/graphql
documents: src/**/*.graphql
generates:
  src/generated/graphql.ts:
    plugins:
      - typescript
      - typescript-operations
      - typescript-react-apollo
```

Результат:

```typescript
// Сгенерированные типы
export type User = {
  __typename?: 'User';
  id: Scalars['ID'];
  name: Scalars['String'];
  email: Scalars['String'];
  posts: Array<Post>;
};

export type GetUserQuery = {
  __typename?: 'Query';
  user?: {
    __typename?: 'User';
    id: string;
    name: string;
  } | null;
};
```

### Программный доступ к схеме

```javascript
const { getIntrospectionQuery, buildClientSchema } = require('graphql');

async function getSchema(endpoint) {
  const response = await fetch(endpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query: getIntrospectionQuery() })
  });

  const { data } = await response.json();
  const schema = buildClientSchema(data);

  return schema;
}

// Использование
const schema = await getSchema('http://localhost:4000/graphql');

// Получение типа
const userType = schema.getType('User');
console.log(userType.getFields());

// Получение Query типа
const queryType = schema.getQueryType();
console.log(queryType.getFields());
```

### Валидация запросов на клиенте

```javascript
const { validate, parse } = require('graphql');

const query = `
  query GetUser($id: ID!) {
    user(id: $id) {
      name
      email
      unknownField  # Ошибка!
    }
  }
`;

const errors = validate(schema, parse(query));

if (errors.length > 0) {
  console.log('Validation errors:', errors);
  // [{ message: 'Cannot query field "unknownField" on type "User".' }]
}
```

## Отключение интроспекции

В production рекомендуется отключать интроспекцию для безопасности.

### Apollo Server

```javascript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  introspection: process.env.NODE_ENV !== 'production'
});
```

### Через middleware

```javascript
const { NoSchemaIntrospectionCustomRule } = require('graphql');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [NoSchemaIntrospectionCustomRule]
});
```

### Условное отключение

```javascript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    {
      async requestDidStart(requestContext) {
        return {
          async didResolveOperation(context) {
            // Разрешить интроспекцию только для админов
            const isAdmin = context.context.user?.role === 'ADMIN';
            const isIntrospection = context.operationName === 'IntrospectionQuery';

            if (isIntrospection && !isAdmin) {
              throw new Error('Introspection is disabled');
            }
          }
        };
      }
    }
  ]
});
```

## Безопасность и интроспекция

### Риски открытой интроспекции

| Риск | Описание |
|------|----------|
| Раскрытие структуры | Злоумышленник видит все endpoints |
| Поиск уязвимостей | Легче найти чувствительные данные |
| Автоматизированные атаки | Скрипты могут анализировать API |

### Рекомендации

1. **Production** — отключить интроспекцию
2. **Development** — включить для удобства разработки
3. **Staging** — защитить авторизацией
4. **Публичные API** — рассмотреть частичную интроспекцию

### Альтернативы в production

```javascript
// Предоставить статичную схему через отдельный endpoint
app.get('/schema.graphql', authMiddleware, (req, res) => {
  res.type('text/plain');
  res.send(printSchema(schema));
});

// Или через SDL файл
app.get('/schema', authMiddleware, (req, res) => {
  res.sendFile(path.join(__dirname, 'schema.graphql'));
});
```

## Документирование схемы

### Описания в SDL

```graphql
"""
Пользователь системы.
Содержит информацию о зарегистрированном пользователе.
"""
type User {
  "Уникальный идентификатор пользователя"
  id: ID!

  "Полное имя пользователя"
  name: String!

  """
  Email адрес пользователя.
  Используется для авторизации и уведомлений.
  """
  email: String!

  "Публикации пользователя"
  posts(
    "Количество записей для возврата"
    first: Int = 10
    "Курсор для пагинации"
    after: String
  ): PostConnection!
}
```

### Устаревшие поля

```graphql
type User {
  id: ID!
  name: String!

  "Используйте поле 'name'"
  username: String @deprecated(reason: "Use 'name' field instead")

  "Старый формат имени"
  fullName: String @deprecated
}
```

В интроспекции:

```json
{
  "__type": {
    "fields": [
      {
        "name": "username",
        "isDeprecated": true,
        "deprecationReason": "Use 'name' field instead"
      }
    ]
  }
}
```

## Инструменты для работы со схемой

### graphql-inspector

```bash
# Сравнение схем
graphql-inspector diff old-schema.graphql new-schema.graphql

# Валидация схемы
graphql-inspector validate schema.graphql

# Проверка покрытия
graphql-inspector coverage schema.graphql documents/*.graphql
```

### GraphQL Voyager

Визуализация схемы как интерактивного графа.

```javascript
import { Voyager } from 'graphql-voyager';

function App() {
  return (
    <Voyager
      introspection={introspectionProvider}
      displayOptions={{ skipRelay: true }}
    />
  );
}

function introspectionProvider(query) {
  return fetch('/graphql', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query })
  }).then(res => res.json());
}
```

### SDL-first vs Code-first

```javascript
// SDL-first (схема в .graphql файлах)
const typeDefs = fs.readFileSync('schema.graphql', 'utf-8');

// Code-first (схема генерируется из кода)
import { objectType, queryType, makeSchema } from 'nexus';

const User = objectType({
  name: 'User',
  description: 'Пользователь системы',
  definition(t) {
    t.nonNull.id('id', { description: 'Уникальный идентификатор' });
    t.nonNull.string('name', { description: 'Имя пользователя' });
  }
});

const schema = makeSchema({
  types: [User],
  outputs: {
    schema: path.join(__dirname, 'schema.graphql'),
    typegen: path.join(__dirname, 'types.ts')
  }
});
```

## Практические примеры

### Проверка наличия поля

```javascript
async function hasField(endpoint, typeName, fieldName) {
  const query = `
    query {
      __type(name: "${typeName}") {
        fields {
          name
        }
      }
    }
  `;

  const response = await fetch(endpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query })
  });

  const { data } = await response.json();
  const fields = data.__type?.fields || [];

  return fields.some(f => f.name === fieldName);
}

// Использование
const hasEmail = await hasField('/graphql', 'User', 'email');
```

### Получение списка всех запросов

```javascript
async function getAvailableQueries(endpoint) {
  const query = `
    query {
      __schema {
        queryType {
          fields {
            name
            description
            args {
              name
              type { name }
            }
          }
        }
      }
    }
  `;

  const response = await fetch(endpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query })
  });

  const { data } = await response.json();
  return data.__schema.queryType.fields;
}
```

### Мониторинг изменений схемы

```javascript
const crypto = require('crypto');

async function getSchemaHash(endpoint) {
  const { data } = await fetchIntrospection(endpoint);
  const hash = crypto
    .createHash('md5')
    .update(JSON.stringify(data))
    .digest('hex');
  return hash;
}

// Проверка изменений
const previousHash = await redis.get('schema:hash');
const currentHash = await getSchemaHash('/graphql');

if (previousHash !== currentHash) {
  console.log('Schema has changed!');
  await notifyTeam('GraphQL schema was updated');
  await redis.set('schema:hash', currentHash);
}
```

## Практические советы

1. **Документируйте схему** — описания видны через интроспекцию
2. **Отключайте в production** — если нет веских причин
3. **Используйте @deprecated** — вместо удаления полей
4. **Автоматизируйте** — генерацию типов и валидацию
5. **Мониторьте изменения** — для контроля breaking changes

## Заключение

Интроспекция — мощная возможность GraphQL, делающая API самодокументируемым и удобным для разработки. Инструменты вроде GraphiQL, генераторы кода и валидаторы запросов основаны на интроспекции. Однако в production-окружении рекомендуется ограничивать доступ к интроспекции для обеспечения безопасности.
