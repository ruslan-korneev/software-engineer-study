# Token-Based Authentication (Аутентификация на основе токенов)

## Что такое Token-Based Authentication?

**Token-Based Authentication** — это метод аутентификации, при котором после успешной проверки учётных данных сервер выдаёт клиенту специальный токен. Этот токен затем используется для подтверждения личности пользователя в последующих запросах вместо постоянной передачи логина и пароля.

## Как это работает

### Основной механизм

1. Пользователь отправляет учётные данные (логин/пароль) на сервер
2. Сервер проверяет данные и генерирует уникальный токен
3. Токен отправляется клиенту и сохраняется на стороне клиента
4. Клиент включает токен в каждый последующий запрос
5. Сервер проверяет токен и обрабатывает запрос

### Диаграмма взаимодействия

```
Клиент                                         Сервер
   |                                              |
   |  POST /auth/login                            |
   |  {username, password}                        |
   |--------------------------------------------->|
   |                                              |  Проверка учётных данных
   |                                              |  Генерация токена
   |  200 OK                                      |  Сохранение токена в БД
   |  {token: "abc123...", expires_in: 3600}      |
   |<---------------------------------------------|
   |                                              |
   |  GET /api/protected                          |
   |  Authorization: Bearer abc123...             |
   |--------------------------------------------->|
   |                                              |  Проверка токена в БД
   |  200 OK                                      |
   |  {data: ...}                                 |
   |<---------------------------------------------|
   |                                              |
   |  POST /auth/logout                           |
   |  Authorization: Bearer abc123...             |
   |--------------------------------------------->|
   |                                              |  Удаление токена из БД
   |  200 OK                                      |
   |<---------------------------------------------|
```

## Типы токенов

### 1. Opaque Tokens (Непрозрачные токены)

Случайная строка, не содержащая данных. Информация о пользователе хранится на сервере.

```python
import secrets

def generate_opaque_token():
    """Генерирует случайный 32-байтовый токен"""
    return secrets.token_urlsafe(32)

# Пример: "dGhpcyBpcyBhIHNhbXBsZSB0b2tlbg..."
```

### 2. Self-Contained Tokens (Самодостаточные токены)

Содержат данные о пользователе внутри себя (JWT — яркий пример).

```python
# JWT токен содержит payload с данными
# eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
# eyJ1c2VyX2lkIjoxMjMsInJvbGUiOiJhZG1pbiJ9.
# SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### Сравнение типов токенов

| Характеристика | Opaque Token | Self-Contained (JWT) |
|----------------|--------------|----------------------|
| Размер | Малый (~44 байта) | Большой (сотни байт) |
| Валидация | Требует БД | Локальная (подпись) |
| Отзыв токена | Простой | Сложный |
| Stateless | Нет | Да |
| Информация | Только на сервере | Внутри токена |

## Практические примеры кода

### Python (FastAPI) — полная реализация

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from pydantic import BaseModel
from datetime import datetime, timedelta
import secrets
import hashlib
from typing import Optional

app = FastAPI()
security = HTTPBearer()

# Модели данных
class UserLogin(BaseModel):
    username: str
    password: str

class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"
    expires_in: int

class User(BaseModel):
    id: int
    username: str
    role: str

# Имитация базы данных
USERS_DB = {
    "admin": {
        "id": 1,
        "password_hash": hashlib.sha256("supersecret".encode()).hexdigest(),
        "role": "admin"
    },
    "user": {
        "id": 2,
        "password_hash": hashlib.sha256("userpass".encode()).hexdigest(),
        "role": "user"
    }
}

# Хранилище токенов (в production — Redis или БД)
TOKENS_DB = {}

def hash_password(password: str) -> str:
    """Хеширует пароль (упрощённо, в production используй bcrypt)"""
    return hashlib.sha256(password.encode()).hexdigest()

def generate_token() -> str:
    """Генерирует безопасный случайный токен"""
    return secrets.token_urlsafe(32)

def create_token(user_id: int, username: str, role: str, ttl_seconds: int = 3600) -> str:
    """Создаёт токен и сохраняет его в хранилище"""
    token = generate_token()
    expires_at = datetime.utcnow() + timedelta(seconds=ttl_seconds)

    TOKENS_DB[token] = {
        "user_id": user_id,
        "username": username,
        "role": role,
        "expires_at": expires_at,
        "created_at": datetime.utcnow()
    }

    return token

def validate_token(token: str) -> Optional[dict]:
    """Проверяет токен и возвращает данные пользователя"""
    token_data = TOKENS_DB.get(token)

    if not token_data:
        return None

    if datetime.utcnow() > token_data["expires_at"]:
        # Токен истёк — удаляем его
        del TOKENS_DB[token]
        return None

    return token_data

def revoke_token(token: str) -> bool:
    """Отзывает (удаляет) токен"""
    if token in TOKENS_DB:
        del TOKENS_DB[token]
        return True
    return False

# Dependency для проверки аутентификации
async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> User:
    """Извлекает текущего пользователя из токена"""
    token = credentials.credentials
    token_data = validate_token(token)

    if not token_data:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or expired token",
            headers={"WWW-Authenticate": "Bearer"},
        )

    return User(
        id=token_data["user_id"],
        username=token_data["username"],
        role=token_data["role"]
    )

# Эндпоинты
@app.post("/auth/login", response_model=TokenResponse)
async def login(user_data: UserLogin):
    """Аутентификация пользователя и выдача токена"""
    user = USERS_DB.get(user_data.username)

    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid username or password"
        )

    password_hash = hash_password(user_data.password)
    if password_hash != user["password_hash"]:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid username or password"
        )

    # Создаём токен
    ttl = 3600  # 1 час
    token = create_token(
        user_id=user["id"],
        username=user_data.username,
        role=user["role"],
        ttl_seconds=ttl
    )

    return TokenResponse(
        access_token=token,
        expires_in=ttl
    )

@app.post("/auth/logout")
async def logout(credentials: HTTPAuthorizationCredentials = Depends(security)):
    """Выход — отзыв токена"""
    token = credentials.credentials

    if revoke_token(token):
        return {"message": "Successfully logged out"}

    raise HTTPException(
        status_code=status.HTTP_400_BAD_REQUEST,
        detail="Token not found"
    )

@app.get("/api/me")
async def get_me(current_user: User = Depends(get_current_user)):
    """Возвращает информацию о текущем пользователе"""
    return {
        "id": current_user.id,
        "username": current_user.username,
        "role": current_user.role
    }

@app.get("/api/admin")
async def admin_endpoint(current_user: User = Depends(get_current_user)):
    """Доступен только администраторам"""
    if current_user.role != "admin":
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Admin access required"
        )

    return {"message": "Welcome to admin panel", "user": current_user.username}

@app.get("/api/protected")
async def protected_endpoint(current_user: User = Depends(get_current_user)):
    """Защищённый эндпоинт"""
    return {
        "message": f"Hello, {current_user.username}!",
        "data": "Secret information"
    }
```

### Python (Flask)

```python
from flask import Flask, request, jsonify, g
from functools import wraps
import secrets
from datetime import datetime, timedelta

app = Flask(__name__)

# Хранилище токенов
tokens_db = {}

def generate_token():
    return secrets.token_urlsafe(32)

def token_required(f):
    """Декоратор для защиты эндпоинтов"""
    @wraps(f)
    def decorated(*args, **kwargs):
        auth_header = request.headers.get('Authorization')

        if not auth_header or not auth_header.startswith('Bearer '):
            return jsonify({"error": "Token required"}), 401

        token = auth_header.split(' ')[1]
        token_data = tokens_db.get(token)

        if not token_data:
            return jsonify({"error": "Invalid token"}), 401

        if datetime.utcnow() > token_data['expires_at']:
            del tokens_db[token]
            return jsonify({"error": "Token expired"}), 401

        g.current_user = token_data
        return f(*args, **kwargs)

    return decorated

@app.route('/auth/login', methods=['POST'])
def login():
    data = request.get_json()
    username = data.get('username')
    password = data.get('password')

    # Упрощённая проверка (в production — проверка через БД)
    if username == 'admin' and password == 'secret':
        token = generate_token()
        tokens_db[token] = {
            'user_id': 1,
            'username': username,
            'role': 'admin',
            'expires_at': datetime.utcnow() + timedelta(hours=1)
        }

        return jsonify({
            'access_token': token,
            'token_type': 'bearer',
            'expires_in': 3600
        })

    return jsonify({"error": "Invalid credentials"}), 401

@app.route('/auth/logout', methods=['POST'])
@token_required
def logout():
    auth_header = request.headers.get('Authorization')
    token = auth_header.split(' ')[1]

    if token in tokens_db:
        del tokens_db[token]

    return jsonify({"message": "Logged out successfully"})

@app.route('/api/protected')
@token_required
def protected():
    return jsonify({
        "message": f"Hello, {g.current_user['username']}!",
        "user_id": g.current_user['user_id']
    })

if __name__ == '__main__':
    app.run(debug=True)
```

### Node.js (Express)

```javascript
const express = require('express');
const crypto = require('crypto');
const app = express();

app.use(express.json());

// Хранилище токенов (в production — Redis)
const tokensDb = new Map();

// Генерация токена
function generateToken() {
    return crypto.randomBytes(32).toString('base64url');
}

// Middleware для проверки токена
function authMiddleware(req, res, next) {
    const authHeader = req.headers.authorization;

    if (!authHeader || !authHeader.startsWith('Bearer ')) {
        return res.status(401).json({ error: 'Token required' });
    }

    const token = authHeader.split(' ')[1];
    const tokenData = tokensDb.get(token);

    if (!tokenData) {
        return res.status(401).json({ error: 'Invalid token' });
    }

    if (Date.now() > tokenData.expiresAt) {
        tokensDb.delete(token);
        return res.status(401).json({ error: 'Token expired' });
    }

    req.user = tokenData;
    next();
}

// Эндпоинт логина
app.post('/auth/login', (req, res) => {
    const { username, password } = req.body;

    // Проверка учётных данных (упрощённо)
    if (username === 'admin' && password === 'secret') {
        const token = generateToken();
        const expiresIn = 3600; // 1 час

        tokensDb.set(token, {
            userId: 1,
            username: username,
            role: 'admin',
            expiresAt: Date.now() + expiresIn * 1000
        });

        return res.json({
            access_token: token,
            token_type: 'bearer',
            expires_in: expiresIn
        });
    }

    res.status(401).json({ error: 'Invalid credentials' });
});

// Эндпоинт logout
app.post('/auth/logout', authMiddleware, (req, res) => {
    const token = req.headers.authorization.split(' ')[1];
    tokensDb.delete(token);
    res.json({ message: 'Logged out successfully' });
});

// Защищённый эндпоинт
app.get('/api/protected', authMiddleware, (req, res) => {
    res.json({
        message: `Hello, ${req.user.username}!`,
        userId: req.user.userId
    });
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

### Клиентский код

```javascript
class AuthClient {
    constructor(baseUrl) {
        this.baseUrl = baseUrl;
        this.token = localStorage.getItem('access_token');
    }

    async login(username, password) {
        const response = await fetch(`${this.baseUrl}/auth/login`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({ username, password })
        });

        if (!response.ok) {
            throw new Error('Login failed');
        }

        const data = await response.json();
        this.token = data.access_token;
        localStorage.setItem('access_token', this.token);

        // Планируем обновление токена
        setTimeout(() => this.refreshToken(), (data.expires_in - 60) * 1000);

        return data;
    }

    async logout() {
        try {
            await this.authFetch('/auth/logout', { method: 'POST' });
        } finally {
            this.token = null;
            localStorage.removeItem('access_token');
        }
    }

    async authFetch(endpoint, options = {}) {
        if (!this.token) {
            throw new Error('Not authenticated');
        }

        const response = await fetch(`${this.baseUrl}${endpoint}`, {
            ...options,
            headers: {
                ...options.headers,
                'Authorization': `Bearer ${this.token}`,
                'Content-Type': 'application/json'
            }
        });

        if (response.status === 401) {
            // Токен истёк — пробуем обновить
            await this.refreshToken();
            return this.authFetch(endpoint, options);
        }

        return response;
    }

    async refreshToken() {
        // Реализация обновления токена
        // ...
    }
}

// Использование
const auth = new AuthClient('https://api.example.com');

await auth.login('admin', 'secret');
const response = await auth.authFetch('/api/protected');
const data = await response.json();
console.log(data);
```

## Хранение токенов на сервере

### Redis (рекомендуется для production)

```python
import redis
import json
from datetime import timedelta

class TokenStore:
    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis = redis.from_url(redis_url)
        self.prefix = "token:"

    def save_token(self, token: str, user_data: dict, ttl_seconds: int = 3600):
        """Сохраняет токен с TTL"""
        key = f"{self.prefix}{token}"
        self.redis.setex(key, ttl_seconds, json.dumps(user_data))

    def get_token(self, token: str) -> dict | None:
        """Получает данные по токену"""
        key = f"{self.prefix}{token}"
        data = self.redis.get(key)

        if data:
            return json.loads(data)
        return None

    def revoke_token(self, token: str) -> bool:
        """Удаляет токен"""
        key = f"{self.prefix}{token}"
        return self.redis.delete(key) > 0

    def revoke_all_user_tokens(self, user_id: int):
        """Отзывает все токены пользователя"""
        # Для этого нужен дополнительный индекс user_id -> tokens
        pattern = f"{self.prefix}*"
        for key in self.redis.scan_iter(pattern):
            data = self.redis.get(key)
            if data:
                token_data = json.loads(data)
                if token_data.get("user_id") == user_id:
                    self.redis.delete(key)

# Использование
store = TokenStore()
store.save_token("abc123", {"user_id": 1, "role": "admin"}, ttl_seconds=3600)
user_data = store.get_token("abc123")
```

### PostgreSQL

```python
from sqlalchemy import create_engine, Column, String, Integer, DateTime, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from datetime import datetime, timedelta
import secrets

Base = declarative_base()

class Token(Base):
    __tablename__ = 'tokens'

    id = Column(Integer, primary_key=True)
    token = Column(String(64), unique=True, nullable=False, index=True)
    user_id = Column(Integer, ForeignKey('users.id'), nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    expires_at = Column(DateTime, nullable=False)

    @classmethod
    def create(cls, user_id: int, ttl_seconds: int = 3600):
        return cls(
            token=secrets.token_urlsafe(32),
            user_id=user_id,
            expires_at=datetime.utcnow() + timedelta(seconds=ttl_seconds)
        )

    def is_valid(self) -> bool:
        return datetime.utcnow() < self.expires_at

# Очистка просроченных токенов (cron job)
def cleanup_expired_tokens(session):
    session.query(Token).filter(Token.expires_at < datetime.utcnow()).delete()
    session.commit()
```

## Плюсы Token-Based Auth

| Преимущество | Описание |
|--------------|----------|
| **Безопасность** | Пароль передаётся только один раз при логине |
| **Revocation** | Можно отозвать токен в любой момент |
| **Flexibility** | Разные токены могут иметь разные права |
| **Cross-domain** | Легко работает с разными доменами |
| **Mobile-friendly** | Удобно использовать в мобильных приложениях |
| **Logout** | Полноценный выход из системы |

## Минусы Token-Based Auth

| Недостаток | Описание |
|------------|----------|
| **Stateful** | Opaque токены требуют хранилища на сервере |
| **DB lookup** | Каждый запрос требует проверки в БД |
| **Масштабирование** | Нужно синхронизировать хранилище токенов |
| **Усложнение** | Больше кода, чем Basic Auth |
| **Token theft** | Украденный токен даёт полный доступ |

## Когда использовать

### Подходит для:

- Веб-приложений с пользовательскими сессиями
- Мобильных приложений
- SPA (Single Page Applications)
- Когда нужен полноценный logout
- Когда нужен контроль над активными сессиями

### НЕ подходит для:

- Простых API без состояния
- Machine-to-machine взаимодействия (лучше API keys)
- Микросервисов (лучше JWT)
- Когда важна stateless архитектура

## Вопросы безопасности

### 1. Хранение токенов на клиенте

```javascript
// ПЛОХО: localStorage уязвим к XSS
localStorage.setItem('token', token);

// ЛУЧШЕ: HttpOnly cookie (защита от XSS)
// Сервер устанавливает:
// Set-Cookie: token=abc123; HttpOnly; Secure; SameSite=Strict

// ЕЩЁ ЛУЧШЕ: В памяти + refresh через HttpOnly cookie
class SecureTokenStorage {
    #accessToken = null;

    setToken(token) {
        this.#accessToken = token;
    }

    getToken() {
        return this.#accessToken;
    }

    clear() {
        this.#accessToken = null;
    }
}
```

### 2. Защита от кражи токена

```python
from fastapi import Request

async def validate_token_with_context(
    request: Request,
    token: str
) -> bool:
    """Проверяет токен с учётом контекста"""
    token_data = get_token_from_db(token)

    if not token_data:
        return False

    # Проверяем IP (опционально, может мешать мобильным)
    if token_data.get("ip") != request.client.host:
        # Логируем подозрительную активность
        log_suspicious_activity(token, request)
        return False

    # Проверяем User-Agent
    if token_data.get("user_agent") != request.headers.get("user-agent"):
        log_suspicious_activity(token, request)
        return False

    return True
```

### 3. Ограничение количества активных токенов

```python
MAX_ACTIVE_TOKENS_PER_USER = 5

def create_token_with_limit(user_id: int) -> str:
    """Создаёт токен с ограничением количества"""
    active_tokens = get_active_tokens_for_user(user_id)

    if len(active_tokens) >= MAX_ACTIVE_TOKENS_PER_USER:
        # Удаляем самый старый токен
        oldest_token = min(active_tokens, key=lambda t: t.created_at)
        revoke_token(oldest_token.token)

    return create_new_token(user_id)
```

## Best Practices

1. **Используй криптографически стойкие токены** — `secrets.token_urlsafe(32)` минимум
2. **Устанавливай разумный TTL** — не более нескольких часов для access token
3. **Реализуй refresh tokens** — для обновления access token без повторного логина
4. **Храни токены в Redis** — быстрый доступ и автоматическое удаление по TTL
5. **Логируй использование токенов** — для аудита и обнаружения аномалий
6. **Ограничивай количество активных токенов** — защита от накопления сессий
7. **Привязывай токены к контексту** — IP, User-Agent, device fingerprint

## Типичные ошибки

### Ошибка 1: Использование предсказуемых токенов

```python
# ПЛОХО: предсказуемый токен
token = f"user_{user_id}_{timestamp}"

# ХОРОШО: криптографически случайный
token = secrets.token_urlsafe(32)
```

### Ошибка 2: Отсутствие TTL

```python
# ПЛОХО: токен живёт вечно
tokens_db[token] = {"user_id": user_id}

# ХОРОШО: токен с ограниченным сроком жизни
tokens_db[token] = {
    "user_id": user_id,
    "expires_at": datetime.utcnow() + timedelta(hours=1)
}
```

### Ошибка 3: Нет возможности отозвать все токены

```python
# ПЛОХО: нет связи токенов с пользователем
tokens = {"token1": {...}, "token2": {...}}

# ХОРОШО: индекс по user_id
user_tokens = {"user_1": ["token1", "token2"]}  # Для быстрого отзыва всех
```

## Refresh Tokens

Паттерн для продления сессии без повторного ввода пароля:

```python
class TokenPair:
    """Access token + Refresh token"""

    def __init__(self, user_id: int):
        self.access_token = secrets.token_urlsafe(32)
        self.refresh_token = secrets.token_urlsafe(64)
        self.user_id = user_id
        self.access_expires = datetime.utcnow() + timedelta(minutes=15)
        self.refresh_expires = datetime.utcnow() + timedelta(days=7)

@app.post("/auth/refresh")
async def refresh_tokens(refresh_token: str):
    """Обновляет пару токенов"""
    token_data = validate_refresh_token(refresh_token)

    if not token_data:
        raise HTTPException(status_code=401, detail="Invalid refresh token")

    # Отзываем старый refresh token (rotation)
    revoke_refresh_token(refresh_token)

    # Создаём новую пару
    new_pair = TokenPair(token_data["user_id"])
    save_tokens(new_pair)

    return {
        "access_token": new_pair.access_token,
        "refresh_token": new_pair.refresh_token,
        "expires_in": 900  # 15 минут
    }
```

## Резюме

Token-Based Authentication — это надёжный метод аутентификации, который решает многие проблемы Basic Auth. Он позволяет реализовать полноценный logout, контролировать активные сессии и не передавать пароль с каждым запросом. Основной компромисс — необходимость хранить состояние на сервере. Для stateless архитектуры рассмотри JWT.
