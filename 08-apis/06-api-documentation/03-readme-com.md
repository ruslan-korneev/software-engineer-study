# Readme.com

## Введение

**Readme.com** (ReadMe) — это платформа для создания интерактивной документации API с встроенным API Explorer, аналитикой и персонализацией. ReadMe превращает OpenAPI спецификации в красивую, интерактивную документацию с возможностью тестирования прямо в браузере.

## Ключевые возможности

| Функция | Описание |
|---------|----------|
| API Reference | Автогенерация из OpenAPI |
| API Explorer | Интерактивное тестирование |
| Guides | Руководства и туториалы |
| Changelog | История изменений |
| Recipes | Пошаговые инструкции |
| Metrics | Аналитика использования |
| Personalization | Персонализация по пользователю |

## Структура документации

### Основные разделы

```
Documentation Hub
├── 🏠 Home (Landing Page)
├── 📚 Guides
│   ├── Getting Started
│   ├── Authentication
│   ├── Rate Limits
│   └── Webhooks
├── 📖 API Reference
│   ├── Users
│   ├── Orders
│   └── Products
├── 📝 Changelog
│   ├── v2.1.0
│   ├── v2.0.0
│   └── v1.0.0
├── 🔧 Recipes
│   ├── Create First User
│   └── Process Payment
└── 📊 Error Codes
```

### Конфигурация проекта

```yaml
# readme.yaml
name: "My API"
version: "2.0"
logo:
  url: "https://example.com/logo.png"
  altText: "My API Logo"
colors:
  main: "#2563EB"
  secondary: "#1E40AF"
navigation:
  - label: "Guides"
    path: "/docs"
  - label: "API Reference"
    path: "/reference"
  - label: "Changelog"
    path: "/changelog"
```

## Создание документации

### Импорт OpenAPI

ReadMe поддерживает автоматический импорт из:
- OpenAPI 3.0/3.1
- Swagger 2.0
- Postman Collections
- GraphQL схемы

```bash
# Синхронизация через CLI
npm install -g rdme

# Аутентификация
rdme login

# Загрузка OpenAPI спецификации
rdme openapi ./openapi.yaml --key=YOUR_API_KEY

# Синхронизация документов
rdme docs ./docs --key=YOUR_API_KEY --version=1.0
```

### Формат Markdown-документов

```markdown
---
title: "Getting Started"
slug: "getting-started"
excerpt: "Начало работы с API"
category: "guides"
order: 1
hidden: false
---

# Начало работы

Добро пожаловать в документацию нашего API!

## Аутентификация

Для работы с API вам понадобится API ключ.

[block:callout]
{
  "type": "info",
  "title": "Получение API ключа",
  "body": "Зарегистрируйтесь в [личном кабинете](https://app.example.com) для получения ключа."
}
[/block]

## Первый запрос

[block:code]
{
  "codes": [
    {
      "code": "curl -X GET 'https://api.example.com/users' \\\n  -H 'Authorization: Bearer YOUR_API_KEY'",
      "language": "bash",
      "name": "cURL"
    },
    {
      "code": "import requests\n\nresponse = requests.get(\n    'https://api.example.com/users',\n    headers={'Authorization': 'Bearer YOUR_API_KEY'}\n)\nprint(response.json())",
      "language": "python",
      "name": "Python"
    },
    {
      "code": "const response = await fetch('https://api.example.com/users', {\n  headers: {\n    'Authorization': 'Bearer YOUR_API_KEY'\n  }\n});\nconst data = await response.json();",
      "language": "javascript",
      "name": "JavaScript"
    }
  ]
}
[/block]
```

### Блоки контента

#### Callouts (Предупреждения)

```markdown
[block:callout]
{
  "type": "info",
  "title": "Информация",
  "body": "Это информационное сообщение."
}
[/block]

[block:callout]
{
  "type": "warning",
  "title": "Внимание",
  "body": "Это важное предупреждение."
}
[/block]

[block:callout]
{
  "type": "danger",
  "title": "Опасность",
  "body": "Это критически важное сообщение."
}
[/block]

[block:callout]
{
  "type": "success",
  "title": "Успех",
  "body": "Операция выполнена успешно."
}
[/block]
```

#### Таблицы

```markdown
[block:parameters]
{
  "data": {
    "h-0": "Параметр",
    "h-1": "Тип",
    "h-2": "Описание",
    "0-0": "email",
    "0-1": "string",
    "0-2": "Email пользователя (обязательный)",
    "1-0": "name",
    "1-1": "string",
    "1-2": "Имя пользователя"
  },
  "cols": 3,
  "rows": 2
}
[/block]
```

#### API Endpoint

```markdown
[block:api-header]
{
  "title": "Создание пользователя",
  "slug": "create-user"
}
[/block]

[block:endpoint]
{
  "method": "POST",
  "path": "/users",
  "auth": "required"
}
[/block]
```

## API Reference из OpenAPI

### Расширения ReadMe для OpenAPI

```yaml
openapi: 3.0.3
info:
  title: User API
  version: 1.0.0
  x-readme:
    explorer-enabled: true
    proxy-enabled: true
    samples-enabled: true
    samples-languages:
      - curl
      - python
      - javascript
      - ruby
      - php

paths:
  /users:
    get:
      summary: Получить пользователей
      x-readme:
        code-samples:
          - language: python
            name: Python SDK
            code: |
              from myapi import Client

              client = Client(api_key="YOUR_KEY")
              users = client.users.list()
        samples-languages:
          - python
          - javascript
      responses:
        '200':
          description: Success
          x-readme:
            headers:
              - name: X-RateLimit-Remaining
                description: Оставшееся количество запросов
```

### Группировка эндпоинтов

```yaml
paths:
  /users:
    get:
      tags:
        - Users
      x-readme:
        explorer-enabled: true
    post:
      tags:
        - Users
      x-readme:
        explorer-enabled: true

tags:
  - name: Users
    description: Операции с пользователями
    x-readme:
      order: 1
  - name: Orders
    description: Операции с заказами
    x-readme:
      order: 2
```

## Персонализация

### Переменные пользователя

ReadMe позволяет персонализировать документацию для авторизованных пользователей:

```javascript
// Конфигурация в ReadMe Dashboard
{
  "apiKey": "{{user.apiKey}}",
  "userId": "{{user.id}}",
  "email": "{{user.email}}",
  "plan": "{{user.plan}}"
}
```

### JWT-интеграция

```javascript
// Генерация JWT для персонализации
const jwt = require('jsonwebtoken');

const user = {
  name: 'John Doe',
  email: 'john@example.com',
  apiKey: 'user-api-key-xxx',
  keys: [
    { name: 'Production', apiKey: 'prod-key' },
    { name: 'Development', apiKey: 'dev-key' }
  ]
};

const token = jwt.sign(user, README_JWT_SECRET, {
  expiresIn: '1h'
});

// Redirect URL
const readmeUrl = `https://your-docs.readme.io?auth_token=${token}`;
```

### Webhooks для синхронизации

```javascript
// Webhook handler для синхронизации пользователей
app.post('/readme-webhook', (req, res) => {
  const { email } = req.body;

  // Найти пользователя и вернуть его данные
  const user = findUserByEmail(email);

  res.json({
    name: user.name,
    email: user.email,
    apiKey: user.apiKey,
    keys: user.apiKeys.map(key => ({
      name: key.name,
      apiKey: key.value
    }))
  });
});
```

## Recipes (Рецепты)

Рецепты — это пошаговые интерактивные туториалы:

```markdown
---
title: "Создание первого пользователя"
slug: "create-first-user"
type: "recipe"
---

# Создание первого пользователя

Этот рецепт покажет, как создать пользователя через API.

## Шаг 1: Получение API ключа

Перейдите в [личный кабинет](https://app.example.com/settings) и скопируйте ваш API ключ.

[block:tutorial-step]
{
  "type": "api",
  "title": "Проверка аутентификации",
  "endpoint": "get-current-user"
}
[/block]

## Шаг 2: Создание пользователя

Теперь создадим нового пользователя.

[block:tutorial-step]
{
  "type": "api",
  "title": "Создать пользователя",
  "endpoint": "create-user",
  "params": {
    "body": {
      "email": "newuser@example.com",
      "name": "New User"
    }
  }
}
[/block]

## Шаг 3: Проверка

Проверим, что пользователь создан.

[block:tutorial-step]
{
  "type": "api",
  "title": "Получить пользователя",
  "endpoint": "get-user-by-id",
  "params": {
    "path": {
      "userId": "{{step2.response.id}}"
    }
  }
}
[/block]
```

## Changelog (История изменений)

```markdown
---
title: "v2.1.0"
slug: "v2-1-0"
type: "changelog"
createdAt: "2024-01-15"
---

# v2.1.0 - Улучшения производительности

[block:callout]
{
  "type": "info",
  "title": "Дата выхода: 15 января 2024"
}
[/block]

## Новые функции

- **Bulk операции**: Добавлена возможность массового создания пользователей
- **Webhooks**: Поддержка webhook-уведомлений о событиях

## Улучшения

- Увеличена скорость ответа API на 40%
- Улучшена обработка ошибок валидации

## Исправления

- Исправлена ошибка при фильтрации по дате
- Исправлена проблема с пагинацией

## Breaking Changes

[block:callout]
{
  "type": "warning",
  "title": "Важно",
  "body": "Поле `user_id` переименовано в `userId` во всех ответах."
}
[/block]

## Migration Guide

```python
# Старый код
user_id = response['user_id']

# Новый код
user_id = response['userId']
```
```

## Аналитика и Metrics

### Доступная аналитика

- **Page Views**: Просмотры страниц
- **API Calls**: Количество вызовов из Try It
- **Search Queries**: Поисковые запросы
- **Popular Endpoints**: Популярные эндпоинты
- **Error Rates**: Частота ошибок

### API для метрик

```bash
# Получение метрик через API
curl -X GET 'https://dash.readme.com/api/v1/projects/{project}/metrics' \
  -H 'Authorization: Basic YOUR_API_KEY'
```

### Кастомные события

```javascript
// Отправка кастомных событий
fetch('https://dash.readme.com/api/v1/events', {
  method: 'POST',
  headers: {
    'Authorization': 'Basic ' + btoa(README_API_KEY + ':'),
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    event: 'api_key_created',
    properties: {
      userId: 'user-123',
      plan: 'pro'
    }
  })
});
```

## Интеграция с CI/CD

### GitHub Action

```yaml
name: Sync API Docs

on:
  push:
    branches: [main]
    paths:
      - 'openapi.yaml'
      - 'docs/**'

jobs:
  sync-docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install rdme CLI
        run: npm install -g rdme

      - name: Sync OpenAPI
        run: |
          rdme openapi ./openapi.yaml \
            --key=${{ secrets.README_API_KEY }} \
            --id=${{ secrets.README_API_DEFINITION_ID }}

      - name: Sync Docs
        run: |
          rdme docs ./docs \
            --key=${{ secrets.README_API_KEY }} \
            --version=1.0
```

### Автоматическое версионирование

```bash
# Создание новой версии
rdme versions:create 2.0 \
  --fork=1.0 \
  --key=YOUR_API_KEY \
  --main=false \
  --beta=true

# Обновление версии
rdme openapi ./openapi-v2.yaml \
  --key=YOUR_API_KEY \
  --version=2.0
```

## Кастомизация темы

### CSS-кастомизация

```css
/* Custom CSS в настройках ReadMe */
:root {
  --color-primary: #2563EB;
  --color-primary-dark: #1E40AF;
  --font-family: 'Inter', sans-serif;
}

/* Кастомный header */
.rm-Header {
  background-color: var(--color-primary);
}

/* Стили для code blocks */
.rm-CodeBlock {
  border-radius: 8px;
  font-size: 14px;
}

/* Кастомные callouts */
.rm-Callout--info {
  border-left-color: var(--color-primary);
}
```

### JavaScript-кастомизация

```javascript
// Custom JavaScript
window.addEventListener('load', () => {
  // Добавление кастомной аналитики
  document.querySelectorAll('.rm-TryIt button').forEach(button => {
    button.addEventListener('click', () => {
      analytics.track('api_try_it_clicked', {
        endpoint: window.location.pathname
      });
    });
  });
});
```

## Best Practices

### 1. Структура документации

```
✅ Хорошая структура:
├── Guides (концептуальные гайды)
│   ├── Getting Started (быстрый старт)
│   ├── Authentication (подробно об авторизации)
│   └── Rate Limits (лимиты)
├── API Reference (автогенерация из OpenAPI)
├── Recipes (пошаговые сценарии)
└── Changelog (история изменений)
```

### 2. Каждый эндпоинт должен иметь

- Понятное описание
- Примеры запросов на разных языках
- Примеры успешных ответов
- Примеры ошибок
- Связанные эндпоинты

### 3. Персонализация

```javascript
// Всегда показывайте актуальные ключи пользователя
Authorization: Bearer {{user.apiKey}}

// Вместо
Authorization: Bearer YOUR_API_KEY_HERE
```

### 4. Синхронизация с кодом

```yaml
# Автоматическая синхронизация при изменениях
on:
  push:
    paths:
      - 'openapi.yaml'
```

### 5. Changelog

- Публикуйте changelog при каждом релизе
- Указывайте breaking changes явно
- Предоставляйте migration guides

## Сравнение с альтернативами

| Функция | ReadMe | Swagger UI | Redoc |
|---------|--------|------------|-------|
| Интерактивность | Высокая | Средняя | Низкая |
| Персонализация | Да | Нет | Нет |
| Аналитика | Да | Нет | Нет |
| Guides/Tutorials | Да | Нет | Нет |
| Хостинг | SaaS | Self-host | Оба |
| Цена | Платный | Бесплатный | Бесплатный |

## Ресурсы

- [ReadMe Documentation](https://docs.readme.com/) - официальная документация
- [rdme CLI](https://github.com/readmeio/rdme) - CLI инструмент
- [ReadMe API](https://docs.readme.com/reference) - API для управления
- [OpenAPI Extensions](https://docs.readme.com/docs/openapi-extensions) - расширения OpenAPI
