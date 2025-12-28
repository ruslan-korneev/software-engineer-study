# CORS (Cross-Origin Resource Sharing)

[prev: 02-owasp-risks](./02-owasp-risks.md) | [next: 04-ssl-tls](./04-ssl-tls.md)

---

## Что такое CORS?

**CORS** (Cross-Origin Resource Sharing) — это механизм безопасности браузера, который контролирует, какие веб-страницы могут делать запросы к ресурсам на другом домене. CORS позволяет серверу указать, каким источникам разрешено получать доступ к его ресурсам.

## Same-Origin Policy (SOP)

Прежде чем понять CORS, нужно понять **Same-Origin Policy** — политику одного источника.

**Origin (источник)** состоит из трёх компонентов:
- Протокол (http/https)
- Домен (example.com)
- Порт (80, 443, 3000)

```
https://example.com:443/page
└─┬──┘ └────┬─────┘└┬─┘
протокол   домен  порт
```

**Примеры сравнения источников:**

| URL | Сравнение с `https://example.com` | Результат |
|-----|-----------------------------------|-----------|
| `https://example.com/page` | Тот же источник | Разрешено |
| `http://example.com` | Разный протокол | Заблокировано |
| `https://api.example.com` | Разный домен (поддомен) | Заблокировано |
| `https://example.com:8080` | Разный порт | Заблокировано |

## Как работает CORS?

### Простые запросы (Simple Requests)

Запросы считаются "простыми", если:
- Метод: GET, HEAD или POST
- Заголовки: только стандартные (Accept, Content-Type с базовыми типами)
- Content-Type: `text/plain`, `multipart/form-data`, `application/x-www-form-urlencoded`

```
Браузер                                Сервер (api.example.com)
   |                                          |
   |--- GET /data --------------------------->|
   |    Origin: https://frontend.com          |
   |                                          |
   |<-- 200 OK -------------------------------|
   |    Access-Control-Allow-Origin: *        |
   |    (или конкретный домен)                |
```

### Предварительные запросы (Preflight)

Для "сложных" запросов браузер сначала отправляет OPTIONS-запрос:

```
Браузер                                Сервер
   |                                      |
   |--- OPTIONS /api/data --------------->|  Preflight запрос
   |    Origin: https://frontend.com      |
   |    Access-Control-Request-Method: PUT|
   |    Access-Control-Request-Headers:   |
   |      Content-Type                    |
   |                                      |
   |<-- 200 OK ---------------------------|  Preflight ответ
   |    Access-Control-Allow-Origin: *    |
   |    Access-Control-Allow-Methods: PUT |
   |    Access-Control-Allow-Headers:     |
   |      Content-Type                    |
   |    Access-Control-Max-Age: 86400     |
   |                                      |
   |--- PUT /api/data ------------------->|  Основной запрос
   |    Origin: https://frontend.com      |
   |    Content-Type: application/json    |
   |                                      |
   |<-- 200 OK ---------------------------|  Основной ответ
```

## CORS-заголовки

### Заголовки ответа сервера

| Заголовок | Описание | Пример |
|-----------|----------|--------|
| `Access-Control-Allow-Origin` | Разрешённые источники | `*` или `https://example.com` |
| `Access-Control-Allow-Methods` | Разрешённые HTTP-методы | `GET, POST, PUT, DELETE` |
| `Access-Control-Allow-Headers` | Разрешённые заголовки | `Content-Type, Authorization` |
| `Access-Control-Allow-Credentials` | Разрешить cookies | `true` |
| `Access-Control-Expose-Headers` | Заголовки, доступные JS | `X-Custom-Header` |
| `Access-Control-Max-Age` | Время кэширования preflight | `86400` (секунды) |

### Заголовки запроса браузера

| Заголовок | Описание |
|-----------|----------|
| `Origin` | Источник запроса |
| `Access-Control-Request-Method` | Метод основного запроса (preflight) |
| `Access-Control-Request-Headers` | Заголовки основного запроса (preflight) |

## Практические примеры

### Express.js (Node.js)

```javascript
const express = require('express');
const cors = require('cors');
const app = express();

// Вариант 1: Разрешить все источники (НЕ для production!)
app.use(cors());

// Вариант 2: Конкретный источник
app.use(cors({
    origin: 'https://frontend.example.com'
}));

// Вариант 3: Несколько источников
const allowedOrigins = [
    'https://frontend.example.com',
    'https://admin.example.com'
];

app.use(cors({
    origin: function(origin, callback) {
        // Разрешаем запросы без origin (например, curl)
        if (!origin) return callback(null, true);

        if (allowedOrigins.includes(origin)) {
            callback(null, true);
        } else {
            callback(new Error('Not allowed by CORS'));
        }
    },
    methods: ['GET', 'POST', 'PUT', 'DELETE'],
    allowedHeaders: ['Content-Type', 'Authorization'],
    credentials: true,  // Разрешить cookies
    maxAge: 86400       // Кэшировать preflight на 24 часа
}));

// Вариант 4: Ручная настройка
app.use((req, res, next) => {
    const origin = req.headers.origin;

    if (allowedOrigins.includes(origin)) {
        res.setHeader('Access-Control-Allow-Origin', origin);
    }

    res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
    res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
    res.setHeader('Access-Control-Allow-Credentials', 'true');

    // Обработка preflight
    if (req.method === 'OPTIONS') {
        res.setHeader('Access-Control-Max-Age', '86400');
        return res.status(204).end();
    }

    next();
});
```

### Python Flask

```python
from flask import Flask, request
from flask_cors import CORS

app = Flask(__name__)

# Вариант 1: Базовая настройка
CORS(app)

# Вариант 2: Детальная настройка
CORS(app, resources={
    r"/api/*": {
        "origins": ["https://frontend.example.com"],
        "methods": ["GET", "POST", "PUT", "DELETE"],
        "allow_headers": ["Content-Type", "Authorization"],
        "supports_credentials": True,
        "max_age": 86400
    }
})

# Вариант 3: Ручная настройка
@app.after_request
def add_cors_headers(response):
    origin = request.headers.get('Origin')
    allowed_origins = ['https://frontend.example.com', 'https://admin.example.com']

    if origin in allowed_origins:
        response.headers['Access-Control-Allow-Origin'] = origin
        response.headers['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE'
        response.headers['Access-Control-Allow-Headers'] = 'Content-Type, Authorization'
        response.headers['Access-Control-Allow-Credentials'] = 'true'

    return response

@app.route('/api/data', methods=['GET', 'OPTIONS'])
def get_data():
    if request.method == 'OPTIONS':
        return '', 204
    return {'data': 'value'}
```

### FastAPI (Python)

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# Настройка CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://frontend.example.com",
        "https://admin.example.com"
    ],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Content-Type", "Authorization"],
    max_age=86400
)

@app.get("/api/data")
async def get_data():
    return {"data": "value"}
```

### Nginx (прокси)

```nginx
server {
    listen 443 ssl;
    server_name api.example.com;

    location /api/ {
        # Динамическая проверка Origin
        set $cors_origin "";
        if ($http_origin ~* "^https://(frontend|admin)\.example\.com$") {
            set $cors_origin $http_origin;
        }

        # CORS заголовки
        add_header 'Access-Control-Allow-Origin' $cors_origin always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Content-Type, Authorization' always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;
        add_header 'Access-Control-Max-Age' 86400 always;

        # Обработка preflight
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' $cors_origin;
            add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'Content-Type, Authorization';
            add_header 'Access-Control-Max-Age' 86400;
            add_header 'Content-Length' 0;
            return 204;
        }

        proxy_pass http://backend:3000;
    }
}
```

## Работа с Credentials (cookies)

Для отправки cookies между доменами нужна специальная настройка:

### На сервере:
```javascript
// Express
app.use(cors({
    origin: 'https://frontend.example.com',  // НЕ используйте *
    credentials: true
}));
```

### На клиенте:
```javascript
// Fetch API
fetch('https://api.example.com/data', {
    credentials: 'include'  // Важно!
});

// Axios
axios.get('https://api.example.com/data', {
    withCredentials: true
});
```

**Важно:** При `credentials: true` нельзя использовать `Access-Control-Allow-Origin: *`!

## Типичные ошибки и их решение

### Ошибка 1: "No 'Access-Control-Allow-Origin' header"

```
Access to fetch at 'https://api.example.com' from origin 'https://frontend.example.com'
has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present.
```

**Решение:** Добавьте заголовок `Access-Control-Allow-Origin` на сервере.

### Ошибка 2: "Wildcard with credentials"

```
The value of the 'Access-Control-Allow-Origin' header must not be the wildcard '*'
when the request's credentials mode is 'include'.
```

**Решение:** Укажите конкретный origin вместо `*`.

### Ошибка 3: "Method not allowed"

```
Method PUT is not allowed by Access-Control-Allow-Methods in preflight response.
```

**Решение:** Добавьте метод в `Access-Control-Allow-Methods`.

### Ошибка 4: "Request header not allowed"

```
Request header field Authorization is not allowed by Access-Control-Allow-Headers.
```

**Решение:** Добавьте заголовок в `Access-Control-Allow-Headers`.

## Лучшие практики

### 1. Никогда не используйте `*` в production с credentials

```javascript
// ПЛОХО
app.use(cors({ origin: '*', credentials: true }));

// ХОРОШО
app.use(cors({
    origin: ['https://frontend.example.com'],
    credentials: true
}));
```

### 2. Валидируйте Origin на сервере

```javascript
const allowedOrigins = new Set([
    'https://frontend.example.com',
    'https://admin.example.com'
]);

app.use((req, res, next) => {
    const origin = req.headers.origin;

    // Проверяем, что origin в белом списке
    if (origin && !allowedOrigins.has(origin)) {
        return res.status(403).json({ error: 'Origin not allowed' });
    }

    next();
});
```

### 3. Ограничивайте методы и заголовки

```javascript
// Разрешайте только необходимые методы
app.use(cors({
    methods: ['GET', 'POST'],  // Не разрешайте DELETE если не нужен
    allowedHeaders: ['Content-Type']  // Минимум необходимых заголовков
}));
```

### 4. Используйте Max-Age для кэширования preflight

```javascript
app.use(cors({
    maxAge: 86400  // Кэшировать preflight на 24 часа
}));
```

## Отладка CORS

### В браузере (DevTools)

1. Откройте Network tab
2. Найдите заблокированный запрос
3. Проверьте Response Headers на наличие CORS-заголовков
4. Посмотрите Console на сообщение об ошибке

### Через curl

```bash
# Простой запрос
curl -i -H "Origin: https://frontend.example.com" https://api.example.com/data

# Preflight запрос
curl -i -X OPTIONS \
  -H "Origin: https://frontend.example.com" \
  -H "Access-Control-Request-Method: PUT" \
  -H "Access-Control-Request-Headers: Content-Type" \
  https://api.example.com/data
```

## Заключение

CORS — важный механизм безопасности браузера. Ключевые моменты:

1. **Понимайте Same-Origin Policy** — это основа
2. **Настраивайте явно** — не используйте `*` в production
3. **Проверяйте Origin** — белый список надёжнее wildcard
4. **Тестируйте preflight** — сложные запросы требуют OPTIONS
5. **Credentials требуют конкретный origin** — нельзя совмещать с `*`

---

[prev: 02-owasp-risks](./02-owasp-risks.md) | [next: 04-ssl-tls](./04-ssl-tls.md)
