# Stoplight

## Введение

**Stoplight** — это платформа для проектирования API по принципу Design-First. Stoplight предоставляет визуальные инструменты для создания OpenAPI спецификаций, генерации документации, мокирования API и совместной работы команды.

## Ключевые продукты

| Продукт | Описание |
|---------|----------|
| Stoplight Studio | Визуальный редактор OpenAPI |
| Stoplight Platform | Облачная платформа для команд |
| Stoplight Elements | React-компоненты для документации |
| Prism | Mock-сервер и прокси для валидации |
| Spectral | Линтер для OpenAPI/AsyncAPI |

## Stoplight Studio

### Установка

```bash
# Desktop приложение
# Скачайте с https://stoplight.io/studio

# VS Code расширение
code --install-extension stoplight.spectral

# Веб-версия
# https://stoplight.io/studio-web
```

### Структура проекта

```
api-project/
├── reference/
│   └── openapi.yaml      # OpenAPI спецификация
├── models/
│   ├── User.yaml         # Схема User
│   ├── Order.yaml        # Схема Order
│   └── Error.yaml        # Схема Error
├── docs/
│   ├── getting-started.md
│   └── authentication.md
└── .spectral.yaml        # Правила линтинга
```

### Визуальное редактирование

Studio позволяет создавать API без написания YAML:

1. **Form View** — редактирование через формы
2. **Code View** — редактирование YAML напрямую
3. **Preview** — предпросмотр документации

### Создание модели данных

```yaml
# models/User.yaml
title: User
type: object
description: Представление пользователя в системе
required:
  - id
  - email
  - name
properties:
  id:
    type: string
    format: uuid
    description: Уникальный идентификатор
    example: "550e8400-e29b-41d4-a716-446655440000"
  email:
    type: string
    format: email
    description: Email пользователя
    example: "user@example.com"
  name:
    type: string
    minLength: 2
    maxLength: 100
    description: Полное имя пользователя
    example: "John Doe"
  role:
    type: string
    enum:
      - admin
      - user
      - moderator
    default: user
    description: Роль пользователя
  avatar:
    type: string
    format: uri
    nullable: true
    description: URL аватара
  createdAt:
    type: string
    format: date-time
    readOnly: true
    description: Дата создания
  updatedAt:
    type: string
    format: date-time
    readOnly: true
    description: Дата последнего обновления
```

### Ссылки на модели

```yaml
# reference/openapi.yaml
openapi: 3.1.0
info:
  title: User API
  version: 1.0.0

paths:
  /users:
    get:
      summary: Получить список пользователей
      responses:
        '200':
          description: Успешный ответ
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '../models/User.yaml'
                  pagination:
                    $ref: '../models/Pagination.yaml'

    post:
      summary: Создать пользователя
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '../models/CreateUserRequest.yaml'
      responses:
        '201':
          description: Пользователь создан
          content:
            application/json:
              schema:
                $ref: '../models/User.yaml'
        '400':
          $ref: '#/components/responses/BadRequest'
        '422':
          $ref: '#/components/responses/ValidationError'

components:
  responses:
    BadRequest:
      description: Неверный запрос
      content:
        application/json:
          schema:
            $ref: '../models/Error.yaml'
    ValidationError:
      description: Ошибка валидации
      content:
        application/json:
          schema:
            $ref: '../models/ValidationError.yaml'
```

## Spectral — Линтер для API

### Установка

```bash
npm install -g @stoplight/spectral-cli

# Или локально в проект
npm install --save-dev @stoplight/spectral-cli
```

### Базовое использование

```bash
# Линтинг файла
spectral lint openapi.yaml

# С выводом в JSON
spectral lint openapi.yaml -f json

# Для CI/CD
spectral lint openapi.yaml --fail-severity=warn
```

### Конфигурация (.spectral.yaml)

```yaml
extends:
  - spectral:oas
  - spectral:asyncapi

# Кастомные правила
rules:
  # Отключение правила
  info-contact: off

  # Изменение severity
  operation-description: warn

  # Кастомное правило
  operation-id-kebab-case:
    description: operationId должен быть в kebab-case
    severity: error
    given: "$.paths.*[get,post,put,patch,delete].operationId"
    then:
      function: pattern
      functionOptions:
        match: "^[a-z][a-z0-9-]*$"

  # Требование примеров
  response-examples:
    description: Все ответы должны иметь примеры
    severity: warn
    given: "$.paths.*.*.responses.*.content.*.examples"
    then:
      function: truthy

  # Проверка описаний
  description-min-length:
    description: Описание должно быть информативным
    severity: warn
    given: "$.paths.*.*[description,summary]"
    then:
      function: length
      functionOptions:
        min: 10

  # Проверка тегов
  operation-tags:
    description: Все операции должны иметь теги
    severity: error
    given: "$.paths.*[get,post,put,patch,delete]"
    then:
      field: tags
      function: truthy

# Кастомные функции
functions:
  - custom-functions

# Алиасы для путей
aliases:
  PathItem: "$.paths[*]"
  OperationObject: "#PathItem[get,put,post,delete,patch]"
```

### Встроенные правила

| Правило | Описание |
|---------|----------|
| `info-contact` | Info должен содержать contact |
| `info-description` | Info должен содержать description |
| `operation-operationId` | Операции должны иметь operationId |
| `operation-description` | Операции должны иметь описание |
| `operation-tags` | Операции должны иметь теги |
| `path-params` | Path параметры должны быть определены |
| `no-$ref-siblings` | $ref не должен иметь siblings |
| `oas3-valid-schema-example` | Примеры должны соответствовать схеме |

## Prism — Mock Server

### Установка

```bash
npm install -g @stoplight/prism-cli

# Или через Docker
docker pull stoplight/prism
```

### Запуск mock-сервера

```bash
# Локальный запуск
prism mock openapi.yaml

# С указанием порта
prism mock openapi.yaml -p 4010

# Динамические ответы
prism mock openapi.yaml --dynamic

# Docker
docker run --rm -it \
  -v $(pwd):/api \
  -p 4010:4010 \
  stoplight/prism mock /api/openapi.yaml
```

### Режимы работы Prism

| Режим | Описание |
|-------|----------|
| Static | Возвращает примеры из спецификации |
| Dynamic | Генерирует случайные данные по схеме |

### Примеры запросов

```bash
# Базовый запрос
curl http://localhost:4010/users

# С header для выбора примера
curl http://localhost:4010/users \
  -H "Prefer: example=success"

# С header для кода ответа
curl http://localhost:4010/users \
  -H "Prefer: code=404"

# Динамические данные
curl http://localhost:4010/users \
  -H "Prefer: dynamic=true"
```

### Prism как Proxy

```bash
# Валидация реальных запросов
prism proxy openapi.yaml https://api.example.com -p 4010

# Теперь запросы к localhost:4010 проксируются
# и валидируются против спецификации
curl http://localhost:4010/users
```

## Stoplight Elements

### Установка

```bash
npm install @stoplight/elements
```

### React-интеграция

```jsx
import React from 'react';
import { API } from '@stoplight/elements';
import '@stoplight/elements/styles.min.css';

function ApiDocumentation() {
  return (
    <API
      apiDescriptionUrl="/openapi.yaml"
      router="hash"
      layout="sidebar"
      hideSchemas={false}
      hideTryIt={false}
    />
  );
}

export default ApiDocumentation;
```

### HTML-интеграция

```html
<!DOCTYPE html>
<html>
<head>
  <title>API Documentation</title>
  <link rel="stylesheet" href="https://unpkg.com/@stoplight/elements/styles.min.css">
  <script src="https://unpkg.com/@stoplight/elements/web-components.min.js"></script>
</head>
<body>
  <elements-api
    apiDescriptionUrl="/openapi.yaml"
    router="hash"
    layout="sidebar"
  />
</body>
</html>
```

### Настройки Elements

```jsx
<API
  apiDescriptionUrl="/openapi.yaml"

  // Роутинг
  router="hash"  // или "memory", "history"
  basePath="/docs"

  // Layout
  layout="sidebar"  // или "stacked"

  // Скрытие секций
  hideSchemas={false}
  hideTryIt={false}
  hideExport={false}

  // Try It настройки
  tryItCredentialsPolicy="same-origin"
  tryItCorsProxy="https://cors-proxy.example.com"

  // Логотип
  logo="/logo.png"
/>
```

## Stoplight Platform

### Возможности платформы

- **Design** — визуальное проектирование API
- **Documentation** — автоматическая публикация
- **Mocking** — hosted mock-серверы
- **Governance** — централизованные правила стиля
- **Collaboration** — совместная работа команды

### Структура workspace

```
Organization
├── Workspace: Production APIs
│   ├── Project: User Service
│   │   ├── openapi.yaml
│   │   ├── models/
│   │   └── docs/
│   └── Project: Order Service
│       ├── openapi.yaml
│       └── models/
└── Workspace: Internal APIs
    └── Project: Admin Service
```

### Git-интеграция

```yaml
# .stoplight.json
{
  "formats": {
    "openapi": {
      "rootDir": "reference",
      "include": ["**/*.yaml", "**/*.json"]
    },
    "json_schema": {
      "rootDir": "models",
      "include": ["**/*.yaml"]
    },
    "markdown": {
      "rootDir": "docs"
    }
  },
  "spectralRuleset": ".spectral.yaml"
}
```

## Интеграция с CI/CD

### GitHub Actions

```yaml
name: API Linting and Docs

on:
  push:
    branches: [main]
    paths:
      - 'reference/**'
      - 'models/**'
  pull_request:
    paths:
      - 'reference/**'
      - 'models/**'

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install Spectral
        run: npm install -g @stoplight/spectral-cli

      - name: Lint OpenAPI
        run: spectral lint reference/openapi.yaml --fail-severity=warn

  mock-test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install Prism
        run: npm install -g @stoplight/prism-cli

      - name: Start Mock Server
        run: prism mock reference/openapi.yaml &

      - name: Wait for server
        run: sleep 5

      - name: Test Mock Endpoints
        run: |
          curl -f http://localhost:4010/users
          curl -f http://localhost:4010/users/123

  deploy-docs:
    runs-on: ubuntu-latest
    needs: [lint, mock-test]
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3

      - name: Build Documentation
        run: |
          npm install @stoplight/elements
          npm run build-docs

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./docs-dist
```

### Pre-commit hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: spectral-lint
        name: Spectral Lint
        entry: spectral lint
        language: node
        files: \.(yaml|yml|json)$
        args: ['--fail-severity=warn']
```

## Design-First Workflow

### 1. Проектирование

```yaml
# Начинаем с определения основных сущностей
# models/User.yaml
title: User
type: object
properties:
  id:
    type: string
    format: uuid
  email:
    type: string
    format: email
  # ... остальные поля
```

### 2. Определение endpoints

```yaml
# reference/openapi.yaml
paths:
  /users:
    get:
      summary: List users
      # ...
    post:
      summary: Create user
      # ...
```

### 3. Добавление примеров

```yaml
responses:
  '200':
    content:
      application/json:
        schema:
          $ref: '../models/User.yaml'
        examples:
          success:
            value:
              id: "550e8400-e29b-41d4-a716-446655440000"
              email: "user@example.com"
              name: "John Doe"
```

### 4. Запуск mock-сервера

```bash
prism mock reference/openapi.yaml
```

### 5. Разработка frontend

```javascript
// Frontend может разрабатываться параллельно
const response = await fetch('http://localhost:4010/users');
const users = await response.json();
```

### 6. Реализация backend

```python
# Backend реализует контракт из спецификации
from fastapi import FastAPI
from pydantic import BaseModel, EmailStr

class User(BaseModel):
    id: str
    email: EmailStr
    name: str

@app.get("/users", response_model=List[User])
async def get_users():
    # Реализация согласно спецификации
    pass
```

### 7. Валидация

```bash
# Проверка соответствия реализации спецификации
prism proxy reference/openapi.yaml http://localhost:8000
```

## Best Practices

### 1. Модульная структура

```
api/
├── reference/
│   ├── openapi.yaml        # Главный файл
│   └── paths/
│       ├── users.yaml      # /users endpoints
│       └── orders.yaml     # /orders endpoints
├── models/
│   ├── User.yaml
│   ├── CreateUser.yaml
│   └── UpdateUser.yaml
├── examples/
│   ├── user.json
│   └── users-list.json
└── docs/
    ├── getting-started.md
    └── authentication.md
```

### 2. Переиспользование компонентов

```yaml
# Общая схема ошибки
# models/Error.yaml
title: Error
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
```

### 3. Строгий линтинг

```yaml
# .spectral.yaml
extends: spectral:oas
rules:
  # Обязательные описания
  operation-description: error
  operation-operationId: error

  # Консистентный нейминг
  operation-id-kebab-case: error

  # Документирование
  info-contact: warn
  info-description: error
```

### 4. Автоматизация

```bash
# Объединение файлов
npx @redocly/cli bundle reference/openapi.yaml -o dist/openapi.yaml

# Валидация
spectral lint dist/openapi.yaml

# Генерация кода
openapi-generator-cli generate -i dist/openapi.yaml -g python-fastapi -o server/
```

### 5. Версионирование

```yaml
# Семантическое версионирование
info:
  version: 2.1.0

# Версия в URL
servers:
  - url: https://api.example.com/v2
```

## Сравнение с альтернативами

| Функция | Stoplight | Swagger | Postman |
|---------|-----------|---------|---------|
| Visual Editor | Отличный | Хороший | Средний |
| Linting | Spectral | Базовый | Нет |
| Mocking | Prism | Swagger Mock | Postman Mock |
| Collaboration | Да | Да | Да |
| Git Integration | Да | Нет | Нет |
| Self-hosted | Да | Да | Нет |
| Цена | Freemium | Freemium | Freemium |

## Ресурсы

- [Stoplight Documentation](https://stoplight.io/docs) - официальная документация
- [Spectral](https://github.com/stoplightio/spectral) - линтер
- [Prism](https://github.com/stoplightio/prism) - mock server
- [Elements](https://github.com/stoplightio/elements) - React компоненты
- [OpenAPI.tools](https://openapi.tools/) - экосистема инструментов OpenAPI
