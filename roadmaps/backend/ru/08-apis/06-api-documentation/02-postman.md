# Postman

## Введение

**Postman** — это комплексная платформа для разработки, тестирования и документирования API. Postman позволяет:

- Создавать и отправлять HTTP-запросы
- Организовывать запросы в коллекции
- Писать автоматизированные тесты
- Генерировать документацию
- Создавать mock-серверы
- Мониторить API
- Работать в команде с синхронизацией

## Основные концепции

### Рабочие пространства (Workspaces)

Workspaces позволяют организовать работу:

| Тип | Описание | Использование |
|-----|----------|---------------|
| Personal | Личное пространство | Индивидуальная работа |
| Team | Командное пространство | Совместная работа |
| Private | Приватное командное | Конфиденциальные API |
| Public | Публичное | Open source проекты |

### Коллекции (Collections)

Коллекция — это группа запросов с общей конфигурацией:

```json
{
  "info": {
    "name": "User API",
    "description": "API для управления пользователями",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "variable": [
    {
      "key": "baseUrl",
      "value": "https://api.example.com/v1"
    }
  ],
  "item": [
    {
      "name": "Users",
      "item": [
        {
          "name": "Get All Users",
          "request": {
            "method": "GET",
            "url": "{{baseUrl}}/users"
          }
        }
      ]
    }
  ]
}
```

### Окружения (Environments)

Окружения хранят переменные для разных сред:

```json
{
  "name": "Production",
  "values": [
    {
      "key": "baseUrl",
      "value": "https://api.example.com",
      "enabled": true
    },
    {
      "key": "apiKey",
      "value": "prod-api-key-xxx",
      "enabled": true,
      "type": "secret"
    }
  ]
}
```

```json
{
  "name": "Development",
  "values": [
    {
      "key": "baseUrl",
      "value": "http://localhost:3000",
      "enabled": true
    },
    {
      "key": "apiKey",
      "value": "dev-api-key-xxx",
      "enabled": true
    }
  ]
}
```

## Работа с запросами

### Базовый запрос

```
GET {{baseUrl}}/users?page=1&limit=20
Authorization: Bearer {{accessToken}}
Content-Type: application/json
```

### POST-запрос с телом

```
POST {{baseUrl}}/users
Content-Type: application/json

{
  "email": "user@example.com",
  "name": "John Doe",
  "password": "securePassword123"
}
```

### Загрузка файлов

```
POST {{baseUrl}}/upload
Content-Type: multipart/form-data

- file: [выберите файл]
- description: "Аватар пользователя"
```

## Переменные

### Типы переменных

| Область | Приоритет | Описание |
|---------|-----------|----------|
| Global | Низший | Доступны везде |
| Collection | Средний | В рамках коллекции |
| Environment | Высший | Зависят от окружения |
| Data | Высший | Из файла данных |
| Local | Высший | В рамках запроса |

### Использование переменных

```javascript
// В URL
{{baseUrl}}/users/{{userId}}

// В заголовках
Authorization: Bearer {{accessToken}}

// В теле запроса
{
  "email": "{{userEmail}}",
  "name": "{{userName}}"
}
```

### Динамические переменные

```javascript
// Встроенные динамические переменные
{{$guid}}           // UUID
{{$timestamp}}      // Unix timestamp
{{$isoTimestamp}}   // ISO timestamp
{{$randomInt}}      // Случайное число
{{$randomEmail}}    // Случайный email
{{$randomFirstName}} // Случайное имя
{{$randomUUID}}     // UUID v4
```

## Скрипты

### Pre-request Scripts

Выполняются перед отправкой запроса:

```javascript
// Генерация timestamp
pm.variables.set("timestamp", Date.now());

// Генерация случайного email
const randomEmail = `user_${Date.now()}@example.com`;
pm.variables.set("testEmail", randomEmail);

// Вычисление подписи
const crypto = require('crypto-js');
const secretKey = pm.environment.get("secretKey");
const timestamp = Date.now().toString();
const signature = crypto.HmacSHA256(timestamp, secretKey).toString();

pm.variables.set("signature", signature);
pm.variables.set("requestTimestamp", timestamp);

// Получение токена если истёк
const tokenExpiry = pm.environment.get("tokenExpiry");
if (!tokenExpiry || Date.now() > parseInt(tokenExpiry)) {
    pm.sendRequest({
        url: pm.environment.get("baseUrl") + "/auth/refresh",
        method: "POST",
        header: {
            "Content-Type": "application/json"
        },
        body: {
            mode: "raw",
            raw: JSON.stringify({
                refreshToken: pm.environment.get("refreshToken")
            })
        }
    }, (err, response) => {
        if (!err) {
            const data = response.json();
            pm.environment.set("accessToken", data.accessToken);
            pm.environment.set("tokenExpiry", Date.now() + 3600000);
        }
    });
}
```

### Tests (Post-response Scripts)

Выполняются после получения ответа:

```javascript
// Проверка статуса
pm.test("Status code is 200", () => {
    pm.response.to.have.status(200);
});

// Проверка времени ответа
pm.test("Response time is less than 500ms", () => {
    pm.expect(pm.response.responseTime).to.be.below(500);
});

// Проверка структуры ответа
pm.test("Response has correct structure", () => {
    const jsonData = pm.response.json();

    pm.expect(jsonData).to.have.property("data");
    pm.expect(jsonData).to.have.property("pagination");
    pm.expect(jsonData.data).to.be.an("array");
});

// Проверка типов данных
pm.test("User has correct properties", () => {
    const jsonData = pm.response.json();
    const user = jsonData.data[0];

    pm.expect(user.id).to.be.a("string");
    pm.expect(user.email).to.match(/^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$/);
    pm.expect(user.createdAt).to.match(/^\d{4}-\d{2}-\d{2}/);
});

// Сохранение данных для следующих запросов
pm.test("Save user ID for next request", () => {
    const jsonData = pm.response.json();
    pm.environment.set("userId", jsonData.id);
    pm.environment.set("userEmail", jsonData.email);
});

// Проверка заголовков
pm.test("Content-Type is application/json", () => {
    pm.expect(pm.response.headers.get("Content-Type")).to.include("application/json");
});

// Проверка с использованием JSON Schema
const schema = {
    type: "object",
    required: ["id", "email", "name"],
    properties: {
        id: { type: "string", format: "uuid" },
        email: { type: "string", format: "email" },
        name: { type: "string", minLength: 2 }
    }
};

pm.test("Response matches schema", () => {
    pm.response.to.have.jsonSchema(schema);
});
```

### Тестирование различных сценариев

```javascript
// Тестирование ошибок
pm.test("Error response for invalid request", () => {
    pm.response.to.have.status(400);

    const error = pm.response.json();
    pm.expect(error).to.have.property("code");
    pm.expect(error).to.have.property("message");
});

// Тестирование аутентификации
pm.test("Unauthorized without token", () => {
    pm.response.to.have.status(401);
});

// Тестирование пагинации
pm.test("Pagination works correctly", () => {
    const jsonData = pm.response.json();
    const pagination = jsonData.pagination;

    pm.expect(pagination.page).to.equal(1);
    pm.expect(pagination.limit).to.equal(20);
    pm.expect(pagination.total).to.be.a("number");
    pm.expect(jsonData.data.length).to.be.at.most(pagination.limit);
});
```

## Коллекция для полного CRUD

### Структура коллекции

```
User API Collection
├── Auth
│   ├── Login
│   ├── Refresh Token
│   └── Logout
├── Users
│   ├── Get All Users
│   ├── Get User by ID
│   ├── Create User
│   ├── Update User
│   └── Delete User
└── Admin
    └── Get Statistics
```

### Workflow тестирования

```javascript
// 1. Login - сохраняем токен
pm.test("Login successful", () => {
    pm.response.to.have.status(200);
    const data = pm.response.json();
    pm.environment.set("accessToken", data.accessToken);
    pm.environment.set("refreshToken", data.refreshToken);
});

// 2. Create User - сохраняем ID
pm.test("User created", () => {
    pm.response.to.have.status(201);
    const user = pm.response.json();
    pm.collectionVariables.set("createdUserId", user.id);
});

// 3. Get User - проверяем созданного
pm.test("User retrieved correctly", () => {
    const user = pm.response.json();
    pm.expect(user.id).to.equal(pm.collectionVariables.get("createdUserId"));
});

// 4. Update User
pm.test("User updated", () => {
    pm.response.to.have.status(200);
    const user = pm.response.json();
    pm.expect(user.name).to.equal("Updated Name");
});

// 5. Delete User
pm.test("User deleted", () => {
    pm.response.to.have.status(204);
});

// 6. Verify deletion
pm.test("User not found after deletion", () => {
    pm.response.to.have.status(404);
});
```

## Newman - CLI для Postman

Newman позволяет запускать коллекции из командной строки:

```bash
# Установка
npm install -g newman

# Запуск коллекции
newman run collection.json

# С окружением
newman run collection.json -e environment.json

# С отчётом
newman run collection.json \
  -e environment.json \
  -r htmlextra \
  --reporter-htmlextra-export report.html

# С данными для итераций
newman run collection.json \
  -e environment.json \
  -d testdata.json \
  -n 10

# Сохранение переменных
newman run collection.json \
  -e environment.json \
  --export-environment updated-env.json
```

### Файл данных для итераций

```json
[
  {
    "email": "user1@example.com",
    "name": "User One",
    "role": "admin"
  },
  {
    "email": "user2@example.com",
    "name": "User Two",
    "role": "user"
  },
  {
    "email": "user3@example.com",
    "name": "User Three",
    "role": "moderator"
  }
]
```

### Интеграция с CI/CD

```yaml
# GitHub Actions
name: API Tests

on: [push, pull_request]

jobs:
  api-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install Newman
        run: |
          npm install -g newman
          npm install -g newman-reporter-htmlextra

      - name: Run API Tests
        run: |
          newman run ./postman/collection.json \
            -e ./postman/environment.json \
            -r htmlextra,cli \
            --reporter-htmlextra-export ./reports/api-test-report.html

      - name: Upload Report
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: api-test-report
          path: ./reports/
```

## Документирование API

### Настройка документации в коллекции

Postman автоматически генерирует документацию на основе:
- Описаний коллекций и запросов
- Примеров запросов и ответов
- Markdown-форматирования

### Markdown в описаниях

```markdown
# Создание пользователя

Создаёт нового пользователя в системе.

## Параметры запроса

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| email | string | Да | Email пользователя |
| name | string | Да | Полное имя |
| password | string | Да | Пароль (мин. 8 символов) |

## Пример ответа

```json
{
  "id": "uuid",
  "email": "user@example.com",
  "name": "John Doe",
  "createdAt": "2024-01-15T10:00:00Z"
}
```

## Возможные ошибки

- `400` - Неверные данные
- `409` - Email уже используется
- `422` - Ошибка валидации
```

### Публикация документации

1. Откройте коллекцию
2. Нажмите "View Documentation"
3. Нажмите "Publish"
4. Настройте URL и стилизацию
5. Получите публичную ссылку

## Mock-серверы

### Создание mock-сервера

```javascript
// Postman автоматически создаёт mock на основе примеров

// Пример ответа в коллекции
{
  "name": "Success Response",
  "originalRequest": {
    "method": "GET",
    "url": "{{baseUrl}}/users"
  },
  "status": "OK",
  "code": 200,
  "body": {
    "data": [
      {
        "id": "1",
        "email": "user@example.com",
        "name": "John Doe"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 1
    }
  }
}
```

### Использование mock-сервера

```bash
# Mock URL
https://mock-api-id.mock.pstmn.io/users

# С header для выбора примера
x-mock-response-name: Success Response

# С header для матчинга по коду
x-mock-response-code: 200
```

## Мониторинг API

### Настройка мониторов

Мониторы позволяют запускать коллекции по расписанию:

- Интервал: от 5 минут до 1 недели
- Регионы: разные географические локации
- Уведомления: email, Slack, webhook

### Пример конфигурации

```json
{
  "name": "Production API Health Check",
  "collection": "User API Collection",
  "environment": "Production",
  "schedule": {
    "cron": "0 */5 * * *",
    "timezone": "UTC"
  },
  "options": {
    "followRedirects": true,
    "requestTimeout": 30000
  },
  "notifications": {
    "onError": ["email", "slack"],
    "onFailure": ["email", "slack", "pagerduty"]
  }
}
```

## Flows (Визуальное тестирование)

Postman Flows позволяет создавать визуальные рабочие процессы:

```
[Start]
    ↓
[Login Request]
    ↓
[Save Token] → [Variable]
    ↓
[Get Users Request]
    ↓
[For Each User]
    ↓
[Process User] → [Output]
    ↓
[End]
```

## Best Practices

### 1. Организация коллекций

```
Project API
├── 📁 Auth
│   ├── Login
│   ├── Register
│   └── Refresh Token
├── 📁 Users (CRUD)
│   ├── Create User
│   ├── Get Users
│   ├── Get User by ID
│   ├── Update User
│   └── Delete User
├── 📁 Error Cases
│   ├── Invalid Token
│   ├── Not Found
│   └── Validation Error
└── 📁 Integration Tests
    ├── Full User Flow
    └── Admin Workflow
```

### 2. Именование

```
✅ Хорошо:
- GET Users - List all users
- POST Users - Create user
- GET Users/:id - Get user by ID

❌ Плохо:
- Request 1
- New Request
- test
```

### 3. Использование переменных

```javascript
// ✅ Хорошо - используем переменные
{{baseUrl}}/users/{{userId}}

// ❌ Плохо - хардкод
https://api.example.com/users/12345
```

### 4. Документирование запросов

Каждый запрос должен иметь:
- Понятное имя
- Описание назначения
- Примеры успешных ответов
- Примеры ошибок

### 5. Тесты для каждого запроса

```javascript
// Минимальный набор тестов
pm.test("Status code is correct", () => {
    pm.response.to.have.status(expectedStatus);
});

pm.test("Response time is acceptable", () => {
    pm.expect(pm.response.responseTime).to.be.below(1000);
});

pm.test("Response structure is valid", () => {
    pm.response.to.have.jsonBody();
});
```

## Интеграция с OpenAPI

### Импорт OpenAPI спецификации

1. File → Import
2. Выберите OpenAPI/Swagger файл
3. Postman создаст коллекцию с:
   - Всеми endpoints
   - Параметрами и схемами
   - Примерами запросов

### Экспорт в OpenAPI

```bash
# Postman CLI
postman collection export \
  --collection-id=xxx \
  --format=openapi3 \
  --output=openapi.yaml
```

## Ресурсы

- [Postman Learning Center](https://learning.postman.com/) - официальная документация
- [Postman API Network](https://www.postman.com/explore) - публичные API
- [Newman](https://github.com/postmanlabs/newman) - CLI runner
- [Postman Integrations](https://www.postman.com/integrations/) - интеграции с другими инструментами
