# Session-Based Authentication (Сессионная аутентификация)

## Что такое Session-Based Authentication?

**Session-Based Authentication** — это традиционный метод аутентификации, при котором сервер создаёт и хранит сессию пользователя после успешного входа. Клиент получает идентификатор сессии (session ID), который передаётся с каждым запросом через cookies.

## Как это работает

### Механизм сессионной аутентификации

1. Пользователь отправляет логин и пароль
2. Сервер проверяет учётные данные
3. Сервер создаёт сессию и сохраняет её в хранилище (память/Redis/БД)
4. Сервер отправляет session ID в cookie
5. Браузер автоматически отправляет cookie с каждым запросом
6. Сервер проверяет session ID и извлекает данные пользователя

### Диаграмма взаимодействия

```
Клиент (Браузер)                          Сервер              Session Store
       |                                     |                      |
       |  POST /login                        |                      |
       |  {username, password}               |                      |
       |------------------------------------>|                      |
       |                                     |                      |
       |                                     |  Проверка учётных    |
       |                                     |  данных              |
       |                                     |                      |
       |                                     |  Создание сессии     |
       |                                     |--------------------->|
       |                                     |                      |
       |  Set-Cookie: session_id=abc123      |  Сохранение          |
       |  200 OK                             |<---------------------|
       |<------------------------------------|                      |
       |                                     |                      |
       |  GET /api/profile                   |                      |
       |  Cookie: session_id=abc123          |                      |
       |------------------------------------>|                      |
       |                                     |                      |
       |                                     |  Получение сессии    |
       |                                     |--------------------->|
       |                                     |                      |
       |                                     |  Данные сессии       |
       |                                     |<---------------------|
       |                                     |                      |
       |  200 OK {profile data}              |                      |
       |<------------------------------------|                      |
       |                                     |                      |
       |  POST /logout                       |                      |
       |  Cookie: session_id=abc123          |                      |
       |------------------------------------>|                      |
       |                                     |                      |
       |                                     |  Удаление сессии     |
       |                                     |--------------------->|
       |                                     |                      |
       |  Set-Cookie: session_id=; Max-Age=0 |                      |
       |<------------------------------------|                      |
```

## Компоненты сессии

### Session ID

Уникальный идентификатор сессии — криптографически случайная строка.

```python
import secrets

# Генерация безопасного session ID
session_id = secrets.token_urlsafe(32)
# Пример: "dGhpcyBpcyBhIHNhbXBsZSBzZXNzaW9u"
```

### Session Data

Данные, хранящиеся на сервере:

```python
session_data = {
    "user_id": 123,
    "username": "john_doe",
    "role": "admin",
    "created_at": "2024-01-15T10:30:00Z",
    "last_activity": "2024-01-15T11:45:00Z",
    "ip_address": "192.168.1.100",
    "user_agent": "Mozilla/5.0...",
    "csrf_token": "random_csrf_token"
}
```

### Session Cookie

```http
Set-Cookie: session_id=abc123def456;
            Path=/;
            HttpOnly;
            Secure;
            SameSite=Strict;
            Max-Age=3600
```

## Практические примеры кода

### Python (Flask)

```python
from flask import Flask, session, request, jsonify, redirect
from functools import wraps
import secrets
import redis
import json
from datetime import datetime, timedelta

app = Flask(__name__)
app.secret_key = secrets.token_hex(32)

# Конфигурация сессий
app.config.update(
    SESSION_COOKIE_SECURE=True,      # Только HTTPS
    SESSION_COOKIE_HTTPONLY=True,    # Недоступна из JavaScript
    SESSION_COOKIE_SAMESITE='Strict', # Защита от CSRF
    PERMANENT_SESSION_LIFETIME=timedelta(hours=1)
)

# Имитация БД пользователей
USERS = {
    "admin": {"id": 1, "password": "secret123", "role": "admin"},
    "user": {"id": 2, "password": "userpass", "role": "user"}
}

def login_required(f):
    """Декоратор для защиты эндпоинтов"""
    @wraps(f)
    def decorated(*args, **kwargs):
        if 'user_id' not in session:
            return jsonify({"error": "Authentication required"}), 401
        return f(*args, **kwargs)
    return decorated

def admin_required(f):
    """Декоратор для проверки роли admin"""
    @wraps(f)
    def decorated(*args, **kwargs):
        if 'user_id' not in session:
            return jsonify({"error": "Authentication required"}), 401
        if session.get('role') != 'admin':
            return jsonify({"error": "Admin access required"}), 403
        return f(*args, **kwargs)
    return decorated

@app.route('/auth/login', methods=['POST'])
def login():
    """Вход в систему"""
    data = request.get_json()
    username = data.get('username')
    password = data.get('password')

    user = USERS.get(username)

    if not user or user['password'] != password:
        return jsonify({"error": "Invalid credentials"}), 401

    # Создаём сессию
    session.permanent = True
    session['user_id'] = user['id']
    session['username'] = username
    session['role'] = user['role']
    session['login_time'] = datetime.utcnow().isoformat()

    return jsonify({
        "message": "Login successful",
        "user": {"id": user['id'], "username": username, "role": user['role']}
    })

@app.route('/auth/logout', methods=['POST'])
@login_required
def logout():
    """Выход из системы"""
    session.clear()
    return jsonify({"message": "Logged out successfully"})

@app.route('/api/me')
@login_required
def get_me():
    """Информация о текущем пользователе"""
    return jsonify({
        "user_id": session['user_id'],
        "username": session['username'],
        "role": session['role']
    })

@app.route('/api/admin')
@admin_required
def admin_panel():
    """Админ-панель"""
    return jsonify({"message": "Welcome to admin panel"})

if __name__ == '__main__':
    app.run(debug=True, ssl_context='adhoc')  # HTTPS для разработки
```

### Python (Flask) с Redis Session Store

```python
from flask import Flask, session
from flask_session import Session
import redis

app = Flask(__name__)

# Конфигурация Redis Session Store
app.config.update(
    SECRET_KEY=secrets.token_hex(32),
    SESSION_TYPE='redis',
    SESSION_REDIS=redis.from_url('redis://localhost:6379'),
    SESSION_PERMANENT=True,
    PERMANENT_SESSION_LIFETIME=timedelta(hours=1),
    SESSION_COOKIE_SECURE=True,
    SESSION_COOKIE_HTTPONLY=True,
    SESSION_COOKIE_SAMESITE='Strict',
    SESSION_KEY_PREFIX='myapp:session:'
)

Session(app)

# Теперь сессии хранятся в Redis, а не в cookies
# Это позволяет:
# - Хранить большие объёмы данных
# - Легко инвалидировать сессии
# - Масштабировать приложение
```

### Python (FastAPI) с сессиями

```python
from fastapi import FastAPI, Request, Response, Depends, HTTPException
from starlette.middleware.sessions import SessionMiddleware
from pydantic import BaseModel
import secrets
import redis
import json
from datetime import datetime, timedelta
from typing import Optional

app = FastAPI()

# Добавляем middleware для сессий
app.add_middleware(
    SessionMiddleware,
    secret_key=secrets.token_hex(32),
    session_cookie="session_id",
    max_age=3600,
    https_only=True,
    same_site="strict"
)

# Redis для хранения сессий
redis_client = redis.from_url("redis://localhost:6379")
SESSION_PREFIX = "session:"
SESSION_TTL = 3600  # 1 час

class LoginRequest(BaseModel):
    username: str
    password: str

class SessionData(BaseModel):
    user_id: int
    username: str
    role: str
    created_at: str
    last_activity: str

# Хранилище сессий в Redis
class SessionStore:
    def __init__(self, redis_client, prefix: str = "session:", ttl: int = 3600):
        self.redis = redis_client
        self.prefix = prefix
        self.ttl = ttl

    def create(self, user_id: int, username: str, role: str) -> str:
        """Создаёт новую сессию"""
        session_id = secrets.token_urlsafe(32)
        session_data = {
            "user_id": user_id,
            "username": username,
            "role": role,
            "created_at": datetime.utcnow().isoformat(),
            "last_activity": datetime.utcnow().isoformat()
        }

        self.redis.setex(
            f"{self.prefix}{session_id}",
            self.ttl,
            json.dumps(session_data)
        )

        return session_id

    def get(self, session_id: str) -> Optional[dict]:
        """Получает данные сессии"""
        data = self.redis.get(f"{self.prefix}{session_id}")
        if data:
            session_data = json.loads(data)
            # Обновляем last_activity
            session_data["last_activity"] = datetime.utcnow().isoformat()
            self.redis.setex(
                f"{self.prefix}{session_id}",
                self.ttl,
                json.dumps(session_data)
            )
            return session_data
        return None

    def delete(self, session_id: str) -> bool:
        """Удаляет сессию"""
        return self.redis.delete(f"{self.prefix}{session_id}") > 0

    def delete_all_for_user(self, user_id: int):
        """Удаляет все сессии пользователя"""
        for key in self.redis.scan_iter(f"{self.prefix}*"):
            data = self.redis.get(key)
            if data:
                session_data = json.loads(data)
                if session_data.get("user_id") == user_id:
                    self.redis.delete(key)

session_store = SessionStore(redis_client)

# Имитация БД
USERS = {
    "admin": {"id": 1, "password": "secret123", "role": "admin"},
    "user": {"id": 2, "password": "userpass", "role": "user"}
}

def get_current_user(request: Request) -> SessionData:
    """Dependency для получения текущего пользователя"""
    session_id = request.cookies.get("session_id")

    if not session_id:
        raise HTTPException(status_code=401, detail="Not authenticated")

    session_data = session_store.get(session_id)

    if not session_data:
        raise HTTPException(status_code=401, detail="Session expired")

    return SessionData(**session_data)

def require_role(required_role: str):
    """Factory для проверки роли"""
    def role_checker(user: SessionData = Depends(get_current_user)):
        if user.role != required_role:
            raise HTTPException(status_code=403, detail=f"Role '{required_role}' required")
        return user
    return role_checker

@app.post("/auth/login")
async def login(request: LoginRequest, response: Response):
    """Вход в систему"""
    user = USERS.get(request.username)

    if not user or user["password"] != request.password:
        raise HTTPException(status_code=401, detail="Invalid credentials")

    # Создаём сессию
    session_id = session_store.create(
        user_id=user["id"],
        username=request.username,
        role=user["role"]
    )

    # Устанавливаем cookie
    response.set_cookie(
        key="session_id",
        value=session_id,
        max_age=SESSION_TTL,
        httponly=True,
        secure=True,
        samesite="strict"
    )

    return {"message": "Login successful", "user": {"username": request.username}}

@app.post("/auth/logout")
async def logout(request: Request, response: Response):
    """Выход из системы"""
    session_id = request.cookies.get("session_id")

    if session_id:
        session_store.delete(session_id)

    response.delete_cookie("session_id")
    return {"message": "Logged out successfully"}

@app.post("/auth/logout-all")
async def logout_all(
    request: Request,
    response: Response,
    user: SessionData = Depends(get_current_user)
):
    """Выход со всех устройств"""
    session_store.delete_all_for_user(user.user_id)
    response.delete_cookie("session_id")
    return {"message": "Logged out from all devices"}

@app.get("/api/me")
async def get_me(user: SessionData = Depends(get_current_user)):
    """Информация о текущем пользователе"""
    return {
        "user_id": user.user_id,
        "username": user.username,
        "role": user.role
    }

@app.get("/api/admin")
async def admin_panel(user: SessionData = Depends(require_role("admin"))):
    """Админ-панель"""
    return {"message": f"Welcome, admin {user.username}!"}
```

### Node.js (Express)

```javascript
const express = require('express');
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const { createClient } = require('redis');
const crypto = require('crypto');

const app = express();
app.use(express.json());

// Подключение к Redis
const redisClient = createClient({ url: 'redis://localhost:6379' });
redisClient.connect();

// Настройка сессий
app.use(session({
    store: new RedisStore({ client: redisClient }),
    secret: crypto.randomBytes(32).toString('hex'),
    resave: false,
    saveUninitialized: false,
    name: 'session_id',  // Имя cookie
    cookie: {
        secure: true,           // Только HTTPS
        httpOnly: true,         // Недоступна из JS
        sameSite: 'strict',     // Защита от CSRF
        maxAge: 3600000         // 1 час
    }
}));

// Имитация БД
const USERS = {
    admin: { id: 1, password: 'secret123', role: 'admin' },
    user: { id: 2, password: 'userpass', role: 'user' }
};

// Middleware для проверки аутентификации
function requireAuth(req, res, next) {
    if (!req.session.userId) {
        return res.status(401).json({ error: 'Authentication required' });
    }
    next();
}

function requireAdmin(req, res, next) {
    if (!req.session.userId) {
        return res.status(401).json({ error: 'Authentication required' });
    }
    if (req.session.role !== 'admin') {
        return res.status(403).json({ error: 'Admin access required' });
    }
    next();
}

// Эндпоинты
app.post('/auth/login', (req, res) => {
    const { username, password } = req.body;
    const user = USERS[username];

    if (!user || user.password !== password) {
        return res.status(401).json({ error: 'Invalid credentials' });
    }

    // Регенерируем session ID для защиты от session fixation
    req.session.regenerate((err) => {
        if (err) {
            return res.status(500).json({ error: 'Session error' });
        }

        req.session.userId = user.id;
        req.session.username = username;
        req.session.role = user.role;
        req.session.loginTime = new Date().toISOString();

        res.json({
            message: 'Login successful',
            user: { id: user.id, username, role: user.role }
        });
    });
});

app.post('/auth/logout', requireAuth, (req, res) => {
    req.session.destroy((err) => {
        if (err) {
            return res.status(500).json({ error: 'Logout failed' });
        }
        res.clearCookie('session_id');
        res.json({ message: 'Logged out successfully' });
    });
});

app.get('/api/me', requireAuth, (req, res) => {
    res.json({
        userId: req.session.userId,
        username: req.session.username,
        role: req.session.role
    });
});

app.get('/api/admin', requireAdmin, (req, res) => {
    res.json({ message: `Welcome, admin ${req.session.username}!` });
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

## Хранилища сессий

### 1. In-Memory (не для production)

```python
# Простое хранилище в памяти
sessions = {}

def create_session(user_id):
    session_id = secrets.token_urlsafe(32)
    sessions[session_id] = {"user_id": user_id}
    return session_id
```

**Проблемы:**
- Теряются при перезапуске
- Не работает с несколькими серверами
- Утечка памяти

### 2. Redis (рекомендуется)

```python
import redis

redis_client = redis.from_url("redis://localhost:6379")

def create_session(user_id, ttl=3600):
    session_id = secrets.token_urlsafe(32)
    redis_client.setex(
        f"session:{session_id}",
        ttl,
        json.dumps({"user_id": user_id})
    )
    return session_id

def get_session(session_id):
    data = redis_client.get(f"session:{session_id}")
    return json.loads(data) if data else None
```

**Преимущества:**
- Быстрый доступ
- Автоматический TTL
- Масштабируется
- Персистентность (опционально)

### 3. База данных

```python
from sqlalchemy import Column, String, Integer, DateTime, Text
from datetime import datetime, timedelta

class Session(Base):
    __tablename__ = 'sessions'

    id = Column(String(64), primary_key=True)
    user_id = Column(Integer, nullable=False, index=True)
    data = Column(Text)  # JSON
    created_at = Column(DateTime, default=datetime.utcnow)
    expires_at = Column(DateTime, nullable=False)

    @classmethod
    def create(cls, user_id, data=None, ttl_seconds=3600):
        return cls(
            id=secrets.token_urlsafe(32),
            user_id=user_id,
            data=json.dumps(data or {}),
            expires_at=datetime.utcnow() + timedelta(seconds=ttl_seconds)
        )
```

## Плюсы Session-Based Auth

| Преимущество | Описание |
|--------------|----------|
| **Revocation** | Легко инвалидировать сессию |
| **Logout** | Полноценный выход |
| **Контроль** | Видны все активные сессии |
| **Маленький cookie** | Только session ID |
| **Безопасность** | Данные на сервере, не у клиента |
| **Браузерная интеграция** | Cookies отправляются автоматически |

## Минусы Session-Based Auth

| Недостаток | Описание |
|------------|----------|
| **Stateful** | Требует хранилища на сервере |
| **Масштабирование** | Нужен shared session store |
| **Cross-domain** | Сложности с разными доменами |
| **Mobile** | Менее удобно для мобильных приложений |
| **DB lookup** | Каждый запрос требует проверки в хранилище |

## Когда использовать

### Подходит для:

- Традиционных веб-приложений
- Когда нужен полный контроль над сессиями
- Когда важен мгновенный logout
- Server-rendered приложений

### НЕ подходит для:

- Микросервисной архитектуры (лучше JWT)
- Stateless API
- Мобильных приложений (лучше токены)
- Cross-domain сценариев

## Вопросы безопасности

### 1. Session Fixation Attack

Атакующий навязывает свой session ID жертве.

```python
# ПЛОХО: используем существующий session ID после логина

# ХОРОШО: регенерируем session ID после логина
@app.route('/login', methods=['POST'])
def login():
    if authenticate(username, password):
        # Регенерация session ID
        old_session_data = dict(session)
        session.clear()
        session.regenerate()  # Новый ID

        # Восстанавливаем данные
        session.update(old_session_data)
        session['user_id'] = user_id
```

### 2. Session Hijacking

Кража session ID через XSS или network sniffing.

```python
# Защита: HttpOnly + Secure + SameSite
app.config.update(
    SESSION_COOKIE_HTTPONLY=True,   # Недоступна из JavaScript
    SESSION_COOKIE_SECURE=True,     # Только HTTPS
    SESSION_COOKIE_SAMESITE='Strict'  # Защита от CSRF
)
```

### 3. CSRF (Cross-Site Request Forgery)

```python
from flask_wtf.csrf import CSRFProtect

csrf = CSRFProtect(app)

# Или вручную
def generate_csrf_token():
    if 'csrf_token' not in session:
        session['csrf_token'] = secrets.token_urlsafe(32)
    return session['csrf_token']

def validate_csrf_token(token):
    return token == session.get('csrf_token')

# В формах
@app.route('/transfer', methods=['POST'])
def transfer():
    if not validate_csrf_token(request.form.get('csrf_token')):
        return "CSRF attack detected", 403
    # ...
```

### 4. Session Timeout

```python
from datetime import datetime, timedelta

SESSION_TIMEOUT = timedelta(minutes=30)
ABSOLUTE_TIMEOUT = timedelta(hours=8)

def check_session_timeout():
    """Проверяет таймауты сессии"""
    if 'user_id' not in session:
        return False

    now = datetime.utcnow()

    # Idle timeout (неактивность)
    last_activity = datetime.fromisoformat(session.get('last_activity', now.isoformat()))
    if now - last_activity > SESSION_TIMEOUT:
        session.clear()
        return False

    # Absolute timeout (максимальное время жизни)
    login_time = datetime.fromisoformat(session.get('login_time', now.isoformat()))
    if now - login_time > ABSOLUTE_TIMEOUT:
        session.clear()
        return False

    # Обновляем last_activity
    session['last_activity'] = now.isoformat()
    return True
```

## Best Practices

1. **Регенерируй session ID после логина** — защита от session fixation
2. **Используй HttpOnly cookies** — защита от XSS
3. **Используй Secure flag** — только HTTPS
4. **Устанавливай SameSite=Strict** — защита от CSRF
5. **Реализуй idle и absolute timeout** — автоматическое завершение сессий
6. **Храни сессии в Redis** — производительность и масштабируемость
7. **Логируй аутентификационные события** — для аудита
8. **Ограничивай количество активных сессий** — защита от накопления

## Типичные ошибки

### Ошибка 1: Хранение sensitive данных в cookie

```python
# ПЛОХО: данные в cookie
response.set_cookie('user_data', json.dumps(user_info))

# ХОРОШО: только session ID в cookie, данные на сервере
session['user_data'] = user_info
```

### Ошибка 2: Предсказуемый session ID

```python
# ПЛОХО: предсказуемый ID
session_id = f"user_{user_id}_{timestamp}"

# ХОРОШО: криптографически случайный
session_id = secrets.token_urlsafe(32)
```

### Ошибка 3: Нет регенерации после логина

```python
# ПЛОХО: тот же session ID до и после логина
session['user_id'] = user_id

# ХОРОШО: новый session ID
session.regenerate()
session['user_id'] = user_id
```

### Ошибка 4: Отсутствие таймаутов

```python
# ПЛОХО: сессия живёт вечно
session.permanent = True

# ХОРОШО: разумный TTL
session.permanent = True
app.permanent_session_lifetime = timedelta(hours=1)
```

## Session vs JWT

| Критерий | Session | JWT |
|----------|---------|-----|
| Хранение | Сервер | Клиент |
| Stateless | Нет | Да |
| Отзыв | Простой | Сложный |
| Масштабирование | Shared store | Нет зависимостей |
| Размер | Маленький cookie | Большой токен |
| Cross-domain | Сложно | Легко |
| Logout | Надёжный | Требует blacklist |

## Резюме

Session-Based Authentication — это проверенный временем подход для традиционных веб-приложений. Он обеспечивает надёжный контроль над сессиями и простой logout. Главное — правильно настроить cookies (HttpOnly, Secure, SameSite) и использовать надёжное хранилище (Redis). Для микросервисов и stateless API рассмотри JWT.
