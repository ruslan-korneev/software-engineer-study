# Лучшие практики безопасности API

[prev: 06-server-security](./06-server-security.md) | [next: 08-hashing-algorithms](./08-hashing-algorithms.md)

---

## Введение

API (Application Programming Interface) — основа современных веб-приложений. Безопасность API критически важна, так как они часто обрабатывают конфиденциальные данные и предоставляют доступ к бизнес-логике. В этом разделе рассмотрим лучшие практики защиты REST и GraphQL API.

## Аутентификация

### JWT (JSON Web Tokens)

```javascript
const jwt = require('jsonwebtoken');

// Конфигурация
const JWT_SECRET = process.env.JWT_SECRET;  // Храните в переменных окружения!
const JWT_EXPIRES_IN = '15m';  // Короткое время жизни access token
const REFRESH_TOKEN_EXPIRES_IN = '7d';

// Генерация токенов
function generateTokens(user) {
    const accessToken = jwt.sign(
        {
            userId: user.id,
            role: user.role
        },
        JWT_SECRET,
        {
            expiresIn: JWT_EXPIRES_IN,
            issuer: 'api.example.com',
            audience: 'example.com'
        }
    );

    const refreshToken = jwt.sign(
        { userId: user.id, tokenVersion: user.tokenVersion },
        JWT_SECRET,
        { expiresIn: REFRESH_TOKEN_EXPIRES_IN }
    );

    return { accessToken, refreshToken };
}

// Middleware для проверки токена
function authenticateToken(req, res, next) {
    const authHeader = req.headers['authorization'];
    const token = authHeader && authHeader.split(' ')[1];  // Bearer TOKEN

    if (!token) {
        return res.status(401).json({ error: 'Authentication required' });
    }

    try {
        const decoded = jwt.verify(token, JWT_SECRET, {
            issuer: 'api.example.com',
            audience: 'example.com'
        });
        req.user = decoded;
        next();
    } catch (err) {
        if (err.name === 'TokenExpiredError') {
            return res.status(401).json({ error: 'Token expired' });
        }
        return res.status(403).json({ error: 'Invalid token' });
    }
}
```

### OAuth 2.0 Flow

```javascript
const passport = require('passport');
const OAuth2Strategy = require('passport-oauth2');

passport.use('oauth2', new OAuth2Strategy({
    authorizationURL: 'https://auth.example.com/authorize',
    tokenURL: 'https://auth.example.com/token',
    clientID: process.env.CLIENT_ID,
    clientSecret: process.env.CLIENT_SECRET,
    callbackURL: 'https://api.example.com/callback',
    scope: ['read', 'write'],
    state: true  // CSRF protection
}, async (accessToken, refreshToken, profile, done) => {
    try {
        const user = await User.findOrCreate({ oauthId: profile.id });
        return done(null, user);
    } catch (err) {
        return done(err);
    }
}));
```

### API Keys

```javascript
const crypto = require('crypto');

// Генерация API ключа
function generateApiKey() {
    const prefix = 'sk_live_';  // Префикс для идентификации типа ключа
    const key = crypto.randomBytes(32).toString('hex');
    return prefix + key;
}

// Хранение API ключа (хэшируем, как пароль)
async function storeApiKey(userId, apiKey) {
    const hash = crypto.createHash('sha256').update(apiKey).digest('hex');

    await db.apiKeys.create({
        userId,
        keyHash: hash,
        prefix: apiKey.substring(0, 12),  // Для отображения пользователю
        createdAt: new Date()
    });
}

// Middleware для проверки API ключа
async function authenticateApiKey(req, res, next) {
    const apiKey = req.headers['x-api-key'];

    if (!apiKey) {
        return res.status(401).json({ error: 'API key required' });
    }

    const hash = crypto.createHash('sha256').update(apiKey).digest('hex');
    const keyRecord = await db.apiKeys.findOne({ keyHash: hash });

    if (!keyRecord) {
        return res.status(403).json({ error: 'Invalid API key' });
    }

    // Обновляем время последнего использования
    await db.apiKeys.updateOne(
        { _id: keyRecord._id },
        { lastUsedAt: new Date() }
    );

    req.apiKeyOwner = keyRecord.userId;
    next();
}
```

## Авторизация

### RBAC (Role-Based Access Control)

```javascript
// Определение ролей и разрешений
const permissions = {
    admin: ['read', 'write', 'delete', 'manage_users'],
    editor: ['read', 'write'],
    viewer: ['read']
};

// Middleware для проверки разрешений
function authorize(...requiredPermissions) {
    return (req, res, next) => {
        const userRole = req.user.role;
        const userPermissions = permissions[userRole] || [];

        const hasPermission = requiredPermissions.every(
            perm => userPermissions.includes(perm)
        );

        if (!hasPermission) {
            return res.status(403).json({
                error: 'Insufficient permissions'
            });
        }

        next();
    };
}

// Использование
app.delete('/api/users/:id',
    authenticateToken,
    authorize('manage_users'),
    deleteUser
);
```

### ABAC (Attribute-Based Access Control)

```javascript
// Более гибкий контроль на основе атрибутов
function authorizeResource(action, resourceType) {
    return async (req, res, next) => {
        const user = req.user;
        const resourceId = req.params.id;

        const resource = await getResource(resourceType, resourceId);

        if (!resource) {
            return res.status(404).json({ error: 'Resource not found' });
        }

        // Проверяем политики доступа
        const allowed = await checkPolicy({
            subject: {
                id: user.id,
                role: user.role,
                department: user.department
            },
            action: action,
            resource: {
                type: resourceType,
                owner: resource.ownerId,
                status: resource.status
            },
            context: {
                time: new Date(),
                ip: req.ip
            }
        });

        if (!allowed) {
            return res.status(403).json({ error: 'Access denied' });
        }

        req.resource = resource;
        next();
    };
}

// Пример политики
const policies = [
    {
        // Владелец может делать что угодно
        match: (s, a, r) => r.owner === s.id,
        allow: true
    },
    {
        // Админ может всё
        match: (s) => s.role === 'admin',
        allow: true
    },
    {
        // Менеджер отдела может редактировать ресурсы своего отдела
        match: (s, a, r) => s.role === 'manager' && r.department === s.department,
        allow: ['read', 'write'].includes(a)
    }
];
```

## Валидация входных данных

### JSON Schema валидация

```javascript
const Ajv = require('ajv');
const addFormats = require('ajv-formats');

const ajv = new Ajv({ allErrors: true });
addFormats(ajv);

// Схема пользователя
const userSchema = {
    type: 'object',
    required: ['email', 'password', 'name'],
    additionalProperties: false,  // Запрещаем лишние поля
    properties: {
        email: {
            type: 'string',
            format: 'email',
            maxLength: 255
        },
        password: {
            type: 'string',
            minLength: 12,
            maxLength: 128,
            pattern: '^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]+$'
        },
        name: {
            type: 'string',
            minLength: 1,
            maxLength: 100,
            pattern: '^[\\p{L}\\s-]+$'  // Unicode буквы, пробелы, дефисы
        },
        age: {
            type: 'integer',
            minimum: 13,
            maximum: 150
        }
    }
};

// Middleware для валидации
function validateBody(schema) {
    const validate = ajv.compile(schema);

    return (req, res, next) => {
        const valid = validate(req.body);

        if (!valid) {
            return res.status(400).json({
                error: 'Validation failed',
                details: validate.errors.map(e => ({
                    field: e.instancePath,
                    message: e.message
                }))
            });
        }

        next();
    };
}

app.post('/api/users', validateBody(userSchema), createUser);
```

### Sanitization

```javascript
const sanitizeHtml = require('sanitize-html');
const validator = require('validator');

// Санитизация HTML
function sanitize(dirty) {
    return sanitizeHtml(dirty, {
        allowedTags: [],  // Убираем все теги
        allowedAttributes: {}
    });
}

// Middleware для санитизации
function sanitizeInput(req, res, next) {
    if (req.body) {
        for (const key in req.body) {
            if (typeof req.body[key] === 'string') {
                req.body[key] = sanitize(req.body[key]);
            }
        }
    }
    next();
}

// Валидация специфичных типов
function validateInput(data) {
    const errors = [];

    if (!validator.isEmail(data.email)) {
        errors.push('Invalid email format');
    }

    if (data.url && !validator.isURL(data.url, { protocols: ['https'] })) {
        errors.push('URL must use HTTPS');
    }

    if (data.phone && !validator.isMobilePhone(data.phone, 'ru-RU')) {
        errors.push('Invalid phone number');
    }

    return errors;
}
```

## Rate Limiting и Throttling

### Разноуровневое ограничение

```javascript
const rateLimit = require('express-rate-limit');
const slowDown = require('express-slow-down');
const RedisStore = require('rate-limit-redis');

// Базовый лимит для всех запросов
const generalLimiter = rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 1000,
    message: { error: 'Too many requests' },
    standardHeaders: true
});

// Строгий лимит для аутентификации
const authLimiter = rateLimit({
    windowMs: 60 * 60 * 1000,
    max: 5,
    skipSuccessfulRequests: true,
    message: { error: 'Too many login attempts' }
});

// Постепенное замедление (не блокировка)
const speedLimiter = slowDown({
    windowMs: 15 * 60 * 1000,
    delayAfter: 100,
    delayMs: (hits) => hits * 100  // Увеличивающаяся задержка
});

// Лимит по API ключу (не по IP)
const apiKeyLimiter = rateLimit({
    windowMs: 60 * 1000,
    max: 100,
    keyGenerator: (req) => req.headers['x-api-key'],
    message: { error: 'API rate limit exceeded' }
});

app.use('/api/', generalLimiter, speedLimiter);
app.use('/api/auth/', authLimiter);
app.use('/api/v1/', apiKeyLimiter);
```

## Защита от распространённых атак

### SQL Injection (Параметризованные запросы)

```javascript
// ПЛОХО - SQL Injection возможен
const userId = req.params.id;
const query = `SELECT * FROM users WHERE id = ${userId}`;

// ХОРОШО - Параметризованный запрос
const query = 'SELECT * FROM users WHERE id = $1';
const result = await db.query(query, [userId]);

// ORM (безопасно по умолчанию)
const user = await User.findByPk(userId);
```

### NoSQL Injection

```javascript
// ПЛОХО - MongoDB injection
const user = await User.findOne({
    username: req.body.username,
    password: req.body.password  // { "$ne": "" } обойдёт проверку
});

// ХОРОШО - Принудительная типизация
const user = await User.findOne({
    username: String(req.body.username),
    password: String(req.body.password)
});

// Или валидация с mongoose
const userSchema = new mongoose.Schema({
    username: { type: String, required: true },
    password: { type: String, required: true }
});
```

### Mass Assignment

```javascript
// ПЛОХО - Все поля из запроса попадают в модель
const user = await User.create(req.body);
// Атака: { "username": "hacker", "role": "admin" }

// ХОРОШО - Явный whitelist
const allowedFields = ['username', 'email', 'name'];
const userData = {};
for (const field of allowedFields) {
    if (req.body[field] !== undefined) {
        userData[field] = req.body[field];
    }
}
const user = await User.create(userData);
```

### IDOR (Insecure Direct Object Reference)

```javascript
// ПЛОХО - Нет проверки владельца
app.get('/api/documents/:id', async (req, res) => {
    const doc = await Document.findById(req.params.id);
    res.json(doc);  // Любой может получить любой документ
});

// ХОРОШО - Проверка прав доступа
app.get('/api/documents/:id', authenticateToken, async (req, res) => {
    const doc = await Document.findOne({
        _id: req.params.id,
        $or: [
            { ownerId: req.user.id },
            { sharedWith: req.user.id },
            { isPublic: true }
        ]
    });

    if (!doc) {
        return res.status(404).json({ error: 'Document not found' });
    }

    res.json(doc);
});
```

## Логирование и мониторинг

### Логирование безопасности

```javascript
const winston = require('winston');

const securityLogger = winston.createLogger({
    level: 'info',
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
    ),
    transports: [
        new winston.transports.File({
            filename: '/var/log/app/security.log'
        })
    ]
});

// Middleware для логирования
function securityLogging(req, res, next) {
    const startTime = Date.now();

    res.on('finish', () => {
        const duration = Date.now() - startTime;

        const logData = {
            method: req.method,
            path: req.path,
            status: res.statusCode,
            duration,
            ip: req.ip,
            userAgent: req.headers['user-agent'],
            userId: req.user?.id
        };

        // Логируем подозрительную активность
        if (res.statusCode === 401 || res.statusCode === 403) {
            securityLogger.warn('Access denied', logData);
        } else if (res.statusCode >= 500) {
            securityLogger.error('Server error', logData);
        } else if (duration > 5000) {
            securityLogger.warn('Slow request', logData);
        }
    });

    next();
}
```

### Мониторинг аномалий

```javascript
const Redis = require('ioredis');
const redis = new Redis();

async function detectAnomalies(userId, action) {
    const key = `anomaly:${userId}:${action}`;
    const count = await redis.incr(key);

    if (count === 1) {
        await redis.expire(key, 3600);  // 1 час
    }

    // Подозрительная активность
    if (count > 100) {
        await securityLogger.warn('Anomaly detected', {
            userId,
            action,
            count
        });

        // Уведомляем команду безопасности
        await notifySecurityTeam({
            type: 'anomaly',
            userId,
            action,
            count
        });
    }
}
```

## Безопасность GraphQL

```javascript
const { ApolloServer } = require('apollo-server-express');
const depthLimit = require('graphql-depth-limit');
const { createComplexityLimitRule } = require('graphql-validation-complexity');

const server = new ApolloServer({
    typeDefs,
    resolvers,

    // Ограничение глубины запросов (защита от DoS)
    validationRules: [
        depthLimit(5),
        createComplexityLimitRule(1000, {
            // Стоимость различных операций
            scalarCost: 1,
            objectCost: 10,
            listFactor: 10
        })
    ],

    // Отключаем introspection в production
    introspection: process.env.NODE_ENV !== 'production',

    // Маскируем ошибки
    formatError: (error) => {
        if (error.extensions?.code === 'INTERNAL_SERVER_ERROR') {
            return { message: 'Internal server error' };
        }
        return error;
    },

    // Контекст с аутентификацией
    context: async ({ req }) => {
        const token = req.headers.authorization?.split(' ')[1];
        const user = await verifyToken(token);
        return { user };
    }
});
```

## Версионирование и документация

### Версионирование API

```javascript
// URL versioning
app.use('/api/v1/', v1Router);
app.use('/api/v2/', v2Router);

// Header versioning
app.use('/api/', (req, res, next) => {
    const version = req.headers['api-version'] || '1';
    req.apiVersion = parseInt(version);
    next();
});

// Deprecation headers
app.use('/api/v1/', (req, res, next) => {
    res.set('Deprecation', 'true');
    res.set('Sunset', 'Sat, 01 Jan 2025 00:00:00 GMT');
    res.set('Link', '</api/v2/>; rel="successor-version"');
    next();
});
```

## Чек-лист безопасности API

### Аутентификация
- [ ] Используются надёжные токены (JWT с подписью)
- [ ] Короткое время жизни access token
- [ ] Refresh token ротация
- [ ] Безопасное хранение секретов

### Авторизация
- [ ] Проверка прав на каждый запрос
- [ ] RBAC или ABAC реализован
- [ ] Защита от IDOR

### Валидация
- [ ] Все входные данные валидируются
- [ ] Санитизация применяется
- [ ] Параметризованные запросы

### Защита
- [ ] Rate limiting настроен
- [ ] HTTPS обязателен
- [ ] CORS правильно настроен
- [ ] Заголовки безопасности установлены

### Мониторинг
- [ ] Логирование безопасности
- [ ] Алерты на подозрительную активность
- [ ] Отслеживание ошибок

## Заключение

Безопасность API — непрерывный процесс. Основные принципы:

1. **Аутентификация и авторизация** — проверяйте каждый запрос
2. **Валидация** — никогда не доверяйте входным данным
3. **Минимальные привилегии** — давайте только необходимые права
4. **Мониторинг** — отслеживайте и реагируйте на инциденты
5. **Обновления** — следите за уязвимостями в зависимостях

---

[prev: 06-server-security](./06-server-security.md) | [next: 08-hashing-algorithms](./08-hashing-algorithms.md)
