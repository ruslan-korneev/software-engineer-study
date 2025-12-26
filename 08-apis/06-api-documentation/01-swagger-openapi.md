# Swagger/OpenAPI

## Введение

**OpenAPI Specification (OAS)** — это стандарт описания RESTful API в формате JSON или YAML. **Swagger** — это набор инструментов для работы с OpenAPI спецификациями, включающий редактор, UI для визуализации и генераторы кода.

OpenAPI позволяет описать:
- Endpoints (пути и методы)
- Параметры запросов
- Тела запросов и ответов
- Схемы данных
- Аутентификацию и авторизацию
- Метаданные API

## История версий

| Версия | Год | Особенности |
|--------|-----|-------------|
| Swagger 1.x | 2011 | Начальная версия |
| Swagger 2.0 | 2014 | Стандартизация формата |
| OpenAPI 3.0 | 2017 | Переименование, улучшенная структура |
| OpenAPI 3.1 | 2021 | Полная совместимость с JSON Schema |

## Базовая структура OpenAPI 3.0

```yaml
openapi: 3.0.3
info:
  title: User Management API
  description: API для управления пользователями
  version: 1.0.0
  contact:
    name: API Support
    email: support@example.com
  license:
    name: MIT
    url: https://opensource.org/licenses/MIT

servers:
  - url: https://api.example.com/v1
    description: Production server
  - url: https://staging-api.example.com/v1
    description: Staging server

paths:
  /users:
    get:
      summary: Получить список пользователей
      description: Возвращает пагинированный список всех пользователей
      operationId: getUsers
      tags:
        - Users
      parameters:
        - name: page
          in: query
          description: Номер страницы
          required: false
          schema:
            type: integer
            default: 1
            minimum: 1
        - name: limit
          in: query
          description: Количество элементов на странице
          required: false
          schema:
            type: integer
            default: 20
            maximum: 100
      responses:
        '200':
          description: Успешный ответ
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserList'
        '401':
          $ref: '#/components/responses/Unauthorized'

    post:
      summary: Создать пользователя
      operationId: createUser
      tags:
        - Users
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: Пользователь создан
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          $ref: '#/components/responses/BadRequest'
        '422':
          $ref: '#/components/responses/ValidationError'

  /users/{userId}:
    get:
      summary: Получить пользователя по ID
      operationId: getUserById
      tags:
        - Users
      parameters:
        - $ref: '#/components/parameters/UserId'
      responses:
        '200':
          description: Успешный ответ
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          $ref: '#/components/responses/NotFound'

components:
  schemas:
    User:
      type: object
      required:
        - id
        - email
        - name
      properties:
        id:
          type: string
          format: uuid
          example: "550e8400-e29b-41d4-a716-446655440000"
        email:
          type: string
          format: email
          example: "user@example.com"
        name:
          type: string
          minLength: 2
          maxLength: 100
          example: "John Doe"
        role:
          type: string
          enum: [admin, user, moderator]
          default: user
        createdAt:
          type: string
          format: date-time
        updatedAt:
          type: string
          format: date-time

    CreateUserRequest:
      type: object
      required:
        - email
        - name
        - password
      properties:
        email:
          type: string
          format: email
        name:
          type: string
          minLength: 2
        password:
          type: string
          format: password
          minLength: 8

    UserList:
      type: object
      properties:
        data:
          type: array
          items:
            $ref: '#/components/schemas/User'
        pagination:
          $ref: '#/components/schemas/Pagination'

    Pagination:
      type: object
      properties:
        page:
          type: integer
        limit:
          type: integer
        total:
          type: integer
        totalPages:
          type: integer

    Error:
      type: object
      required:
        - code
        - message
      properties:
        code:
          type: string
        message:
          type: string
        details:
          type: array
          items:
            type: object
            properties:
              field:
                type: string
              message:
                type: string

  parameters:
    UserId:
      name: userId
      in: path
      required: true
      description: Уникальный идентификатор пользователя
      schema:
        type: string
        format: uuid

  responses:
    NotFound:
      description: Ресурс не найден
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: NOT_FOUND
            message: "Ресурс не найден"

    BadRequest:
      description: Неверный запрос
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'

    Unauthorized:
      description: Требуется аутентификация
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'

    ValidationError:
      description: Ошибка валидации
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'

  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

    apiKey:
      type: apiKey
      in: header
      name: X-API-Key

security:
  - bearerAuth: []
```

## Инструменты Swagger

### Swagger Editor

Онлайн-редактор для создания и редактирования OpenAPI спецификаций с real-time валидацией.

```bash
# Локальный запуск через Docker
docker run -d -p 8080:8080 swaggerapi/swagger-editor
```

### Swagger UI

Визуализация API документации с возможностью интерактивного тестирования.

```bash
# Запуск Swagger UI
docker run -d -p 8081:8080 \
  -e SWAGGER_JSON=/api/openapi.yaml \
  -v $(pwd)/openapi.yaml:/api/openapi.yaml \
  swaggerapi/swagger-ui
```

### Swagger Codegen / OpenAPI Generator

Генерация клиентских SDK и серверных stub'ов на разных языках.

```bash
# OpenAPI Generator (рекомендуется)
npm install @openapitools/openapi-generator-cli -g

# Генерация Python клиента
openapi-generator-cli generate \
  -i openapi.yaml \
  -g python \
  -o ./generated/python-client

# Генерация TypeScript клиента
openapi-generator-cli generate \
  -i openapi.yaml \
  -g typescript-axios \
  -o ./generated/ts-client

# Генерация серверного кода FastAPI
openapi-generator-cli generate \
  -i openapi.yaml \
  -g python-fastapi \
  -o ./generated/fastapi-server
```

## Интеграция с фреймворками

### FastAPI (Python)

FastAPI автоматически генерирует OpenAPI спецификацию:

```python
from fastapi import FastAPI, HTTPException, Query
from pydantic import BaseModel, EmailStr, Field
from typing import Optional, List
from uuid import UUID
from datetime import datetime

app = FastAPI(
    title="User Management API",
    description="API для управления пользователями",
    version="1.0.0",
    openapi_tags=[
        {"name": "users", "description": "Операции с пользователями"},
        {"name": "auth", "description": "Аутентификация"}
    ]
)

class UserBase(BaseModel):
    email: EmailStr = Field(..., description="Email пользователя")
    name: str = Field(..., min_length=2, max_length=100, description="Имя пользователя")

class UserCreate(UserBase):
    password: str = Field(..., min_length=8, description="Пароль")

class User(UserBase):
    id: UUID
    role: str = "user"
    created_at: datetime

    class Config:
        json_schema_extra = {
            "example": {
                "id": "550e8400-e29b-41d4-a716-446655440000",
                "email": "user@example.com",
                "name": "John Doe",
                "role": "user",
                "created_at": "2024-01-15T10:30:00Z"
            }
        }

class PaginatedUsers(BaseModel):
    data: List[User]
    total: int
    page: int
    limit: int

@app.get(
    "/users",
    response_model=PaginatedUsers,
    tags=["users"],
    summary="Получить список пользователей",
    description="Возвращает пагинированный список всех пользователей"
)
async def get_users(
    page: int = Query(1, ge=1, description="Номер страницы"),
    limit: int = Query(20, ge=1, le=100, description="Количество на странице")
):
    # Логика получения пользователей
    return {"data": [], "total": 0, "page": page, "limit": limit}

@app.post(
    "/users",
    response_model=User,
    status_code=201,
    tags=["users"],
    summary="Создать пользователя",
    responses={
        201: {"description": "Пользователь успешно создан"},
        422: {"description": "Ошибка валидации"}
    }
)
async def create_user(user: UserCreate):
    # Логика создания пользователя
    pass

@app.get(
    "/users/{user_id}",
    response_model=User,
    tags=["users"],
    summary="Получить пользователя по ID",
    responses={
        404: {"description": "Пользователь не найден"}
    }
)
async def get_user(user_id: UUID):
    raise HTTPException(status_code=404, detail="User not found")
```

Документация доступна по адресам:
- Swagger UI: `http://localhost:8000/docs`
- ReDoc: `http://localhost:8000/redoc`
- OpenAPI JSON: `http://localhost:8000/openapi.json`

### Flask (Python)

```python
from flask import Flask
from flask_restx import Api, Resource, fields

app = Flask(__name__)
api = Api(
    app,
    version='1.0',
    title='User API',
    description='API для управления пользователями'
)

ns = api.namespace('users', description='Операции с пользователями')

user_model = api.model('User', {
    'id': fields.String(required=True, description='ID пользователя'),
    'email': fields.String(required=True, description='Email'),
    'name': fields.String(required=True, description='Имя')
})

@ns.route('/')
class UserList(Resource):
    @ns.doc('list_users')
    @ns.marshal_list_with(user_model)
    def get(self):
        """Получить список всех пользователей"""
        return []

    @ns.doc('create_user')
    @ns.expect(user_model)
    @ns.marshal_with(user_model, code=201)
    def post(self):
        """Создать нового пользователя"""
        pass
```

### Express.js (Node.js)

```javascript
const express = require('express');
const swaggerUi = require('swagger-ui-express');
const swaggerJsdoc = require('swagger-jsdoc');

const app = express();

const options = {
  definition: {
    openapi: '3.0.0',
    info: {
      title: 'User API',
      version: '1.0.0',
      description: 'API для управления пользователями'
    },
    servers: [
      { url: 'http://localhost:3000' }
    ]
  },
  apis: ['./routes/*.js']
};

const specs = swaggerJsdoc(options);
app.use('/docs', swaggerUi.serve, swaggerUi.setup(specs));

/**
 * @openapi
 * /users:
 *   get:
 *     summary: Получить список пользователей
 *     tags: [Users]
 *     parameters:
 *       - in: query
 *         name: page
 *         schema:
 *           type: integer
 *         description: Номер страницы
 *     responses:
 *       200:
 *         description: Список пользователей
 *         content:
 *           application/json:
 *             schema:
 *               type: array
 *               items:
 *                 $ref: '#/components/schemas/User'
 */
app.get('/users', (req, res) => {
  res.json([]);
});

/**
 * @openapi
 * components:
 *   schemas:
 *     User:
 *       type: object
 *       properties:
 *         id:
 *           type: string
 *         email:
 *           type: string
 *         name:
 *           type: string
 */
```

## Продвинутые возможности

### Наследование схем (allOf, oneOf, anyOf)

```yaml
components:
  schemas:
    # Базовая схема
    Pet:
      type: object
      required:
        - name
      properties:
        name:
          type: string
        tag:
          type: string

    # Наследование с allOf
    Dog:
      allOf:
        - $ref: '#/components/schemas/Pet'
        - type: object
          properties:
            breed:
              type: string

    # Полиморфизм с oneOf
    Animal:
      oneOf:
        - $ref: '#/components/schemas/Dog'
        - $ref: '#/components/schemas/Cat'
      discriminator:
        propertyName: petType
        mapping:
          dog: '#/components/schemas/Dog'
          cat: '#/components/schemas/Cat'
```

### Callbacks (Webhooks)

```yaml
paths:
  /webhooks:
    post:
      summary: Подписаться на webhook
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                callbackUrl:
                  type: string
                  format: uri
      callbacks:
        onEvent:
          '{$request.body#/callbackUrl}':
            post:
              summary: Callback при событии
              requestBody:
                content:
                  application/json:
                    schema:
                      $ref: '#/components/schemas/Event'
              responses:
                '200':
                  description: OK
```

### Links (HATEOAS)

```yaml
paths:
  /users/{userId}:
    get:
      responses:
        '200':
          description: User
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
          links:
            GetUserOrders:
              operationId: getUserOrders
              parameters:
                userId: '$response.body#/id'
              description: Получить заказы пользователя
```

## Валидация спецификации

```bash
# Spectral - линтер для OpenAPI
npm install -g @stoplight/spectral-cli

# Валидация
spectral lint openapi.yaml

# Кастомные правила в .spectral.yaml
extends: spectral:oas
rules:
  operation-operationId:
    severity: error
  operation-description:
    severity: warn
  info-contact:
    severity: warn
```

## Best Practices

### 1. Структура проекта

```
api/
├── openapi.yaml           # Главный файл
├── paths/                  # Endpoints
│   ├── users.yaml
│   └── orders.yaml
├── components/
│   ├── schemas/           # Схемы данных
│   │   ├── User.yaml
│   │   └── Order.yaml
│   ├── parameters/        # Переиспользуемые параметры
│   ├── responses/         # Стандартные ответы
│   └── securitySchemes/   # Схемы авторизации
└── examples/              # Примеры запросов/ответов
```

### 2. Объединение файлов

```yaml
# openapi.yaml
openapi: 3.0.3
info:
  title: API
  version: 1.0.0
paths:
  $ref: './paths/_index.yaml'
components:
  schemas:
    $ref: './components/schemas/_index.yaml'
```

```bash
# Объединение в один файл
npm install -g @redocly/cli
redocly bundle openapi.yaml -o dist/openapi.yaml
```

### 3. Версионирование API

```yaml
servers:
  - url: https://api.example.com/v1
    description: Version 1
  - url: https://api.example.com/v2
    description: Version 2 (current)
```

### 4. Документирование ошибок

```yaml
components:
  responses:
    BadRequest:
      description: Неверный запрос
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          examples:
            validation_error:
              summary: Ошибка валидации
              value:
                code: VALIDATION_ERROR
                message: "Поле email обязательно"
                details:
                  - field: email
                    message: "Required field"
            invalid_format:
              summary: Неверный формат
              value:
                code: INVALID_FORMAT
                message: "Неверный формат даты"
```

## Инструменты визуализации

| Инструмент | Описание | Ссылка |
|------------|----------|--------|
| Swagger UI | Интерактивная документация | swagger.io/tools/swagger-ui |
| ReDoc | Красивая статическая документация | redocly.github.io/redoc |
| Stoplight Elements | Встраиваемые компоненты | stoplight.io/open-source/elements |
| RapiDoc | Кастомизируемая документация | rapidocweb.com |

## Ресурсы

- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html) - официальная спецификация
- [Swagger Tools](https://swagger.io/tools/) - инструменты Swagger
- [OpenAPI Generator](https://openapi-generator.tech/) - генерация кода
- [Redocly CLI](https://redocly.com/redocly-cli/) - линтинг и сборка
