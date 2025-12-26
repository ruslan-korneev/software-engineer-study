# JWT (JSON Web Tokens)

## Что такое JWT?

**JWT (JSON Web Token)** — это открытый стандарт (RFC 7519) для создания токенов доступа, которые содержат данные в формате JSON. JWT — это самодостаточный (self-contained) токен, который несёт в себе всю необходимую информацию о пользователе и не требует обращения к базе данных для валидации.

## Структура JWT

JWT состоит из трёх частей, разделённых точками:

```
xxxxx.yyyyy.zzzzz
  |      |     |
  |      |     +-- Signature (Подпись)
  |      +-------- Payload (Полезная нагрузка)
  +--------------- Header (Заголовок)
```

### 1. Header (Заголовок)

Содержит метаданные о токене: тип и алгоритм подписи.

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### 2. Payload (Полезная нагрузка)

Содержит claims — утверждения о пользователе и дополнительные данные.

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "role": "admin",
  "iat": 1516239022,
  "exp": 1516242622
}
```

#### Стандартные claims (RFC 7519):

| Claim | Название | Описание |
|-------|----------|----------|
| `iss` | Issuer | Кто выпустил токен |
| `sub` | Subject | Идентификатор субъекта (обычно user_id) |
| `aud` | Audience | Для кого предназначен токен |
| `exp` | Expiration | Время истечения (Unix timestamp) |
| `nbf` | Not Before | Токен не валиден до этого времени |
| `iat` | Issued At | Когда токен был выпущен |
| `jti` | JWT ID | Уникальный идентификатор токена |

### 3. Signature (Подпись)

Создаётся путём подписания закодированных header и payload секретным ключом.

```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

### Пример реального JWT

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNTE2MjM5MDIyLCJleHAiOjE1MTYyNDI2MjJ9.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

## Как работает JWT

### Процесс аутентификации

```
Клиент                                         Сервер
   |                                              |
   |  POST /auth/login                            |
   |  {username, password}                        |
   |--------------------------------------------->|
   |                                              |  1. Проверка учётных данных
   |                                              |  2. Создание JWT с данными
   |  200 OK                                      |  3. Подпись JWT секретом
   |  {token: "eyJhbG..."}                        |
   |<---------------------------------------------|
   |                                              |
   |  GET /api/protected                          |
   |  Authorization: Bearer eyJhbG...             |
   |--------------------------------------------->|
   |                                              |  1. Декодирование JWT
   |                                              |  2. Проверка подписи
   |                                              |  3. Проверка exp
   |  200 OK                                      |  4. Извлечение данных
   |  {data: ...}                                 |
   |<---------------------------------------------|
```

## Алгоритмы подписи

### Симметричные (HMAC)

Один секретный ключ для создания и проверки подписи.

| Алгоритм | Описание |
|----------|----------|
| HS256 | HMAC с SHA-256 |
| HS384 | HMAC с SHA-384 |
| HS512 | HMAC с SHA-512 |

### Асимметричные (RSA, ECDSA)

Приватный ключ для подписи, публичный для проверки.

| Алгоритм | Описание |
|----------|----------|
| RS256 | RSA с SHA-256 |
| RS384 | RSA с SHA-384 |
| RS512 | RSA с SHA-512 |
| ES256 | ECDSA с P-256 и SHA-256 |
| ES384 | ECDSA с P-384 и SHA-384 |
| ES512 | ECDSA с P-521 и SHA-512 |

## Практические примеры кода

### Python (PyJWT) — базовый пример

```python
import jwt
from datetime import datetime, timedelta

SECRET_KEY = "your-256-bit-secret"
ALGORITHM = "HS256"

def create_jwt(user_id: int, role: str, expires_delta: timedelta = None) -> str:
    """Создаёт JWT токен"""
    if expires_delta is None:
        expires_delta = timedelta(hours=1)

    payload = {
        "sub": str(user_id),
        "role": role,
        "iat": datetime.utcnow(),
        "exp": datetime.utcnow() + expires_delta
    }

    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def decode_jwt(token: str) -> dict:
    """Декодирует и проверяет JWT токен"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except jwt.ExpiredSignatureError:
        raise ValueError("Token has expired")
    except jwt.InvalidTokenError as e:
        raise ValueError(f"Invalid token: {e}")

# Использование
token = create_jwt(user_id=123, role="admin")
print(f"Token: {token}")

decoded = decode_jwt(token)
print(f"Decoded: {decoded}")
```

### Python (FastAPI) — полная реализация

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from pydantic import BaseModel
from datetime import datetime, timedelta
from typing import Optional
import jwt

app = FastAPI()
security = HTTPBearer()

# Конфигурация
SECRET_KEY = "your-super-secret-key-change-in-production"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30
REFRESH_TOKEN_EXPIRE_DAYS = 7

# Модели
class UserLogin(BaseModel):
    username: str
    password: str

class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"

class TokenPayload(BaseModel):
    sub: str
    role: str
    type: str  # "access" или "refresh"
    exp: datetime

# Имитация БД пользователей
USERS_DB = {
    "admin": {"id": 1, "password": "secret123", "role": "admin"},
    "user": {"id": 2, "password": "userpass", "role": "user"}
}

def create_token(user_id: int, role: str, token_type: str, expires_delta: timedelta) -> str:
    """Создаёт JWT токен"""
    expire = datetime.utcnow() + expires_delta

    payload = {
        "sub": str(user_id),
        "role": role,
        "type": token_type,
        "iat": datetime.utcnow(),
        "exp": expire
    }

    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def create_access_token(user_id: int, role: str) -> str:
    """Создаёт access token"""
    return create_token(
        user_id=user_id,
        role=role,
        token_type="access",
        expires_delta=timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    )

def create_refresh_token(user_id: int, role: str) -> str:
    """Создаёт refresh token"""
    return create_token(
        user_id=user_id,
        role=role,
        token_type="refresh",
        expires_delta=timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)
    )

def decode_token(token: str, expected_type: str = "access") -> TokenPayload:
    """Декодирует и валидирует токен"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])

        if payload.get("type") != expected_type:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail=f"Invalid token type. Expected {expected_type}"
            )

        return TokenPayload(**payload)

    except jwt.ExpiredSignatureError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token has expired"
        )
    except jwt.InvalidTokenError as e:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail=f"Invalid token: {str(e)}"
        )

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> TokenPayload:
    """Dependency для получения текущего пользователя"""
    token = credentials.credentials
    return decode_token(token, expected_type="access")

def require_role(required_role: str):
    """Factory для проверки роли"""
    async def role_checker(current_user: TokenPayload = Depends(get_current_user)):
        if current_user.role != required_role:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Role '{required_role}' required"
            )
        return current_user
    return role_checker

# Эндпоинты
@app.post("/auth/login", response_model=TokenResponse)
async def login(user_data: UserLogin):
    """Аутентификация и выдача токенов"""
    user = USERS_DB.get(user_data.username)

    if not user or user["password"] != user_data.password:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid username or password"
        )

    access_token = create_access_token(user["id"], user["role"])
    refresh_token = create_refresh_token(user["id"], user["role"])

    return TokenResponse(
        access_token=access_token,
        refresh_token=refresh_token
    )

@app.post("/auth/refresh", response_model=TokenResponse)
async def refresh_tokens(
    credentials: HTTPAuthorizationCredentials = Depends(security)
):
    """Обновление токенов"""
    token = credentials.credentials
    payload = decode_token(token, expected_type="refresh")

    # Создаём новую пару токенов
    access_token = create_access_token(int(payload.sub), payload.role)
    refresh_token = create_refresh_token(int(payload.sub), payload.role)

    return TokenResponse(
        access_token=access_token,
        refresh_token=refresh_token
    )

@app.get("/api/me")
async def get_me(current_user: TokenPayload = Depends(get_current_user)):
    """Информация о текущем пользователе"""
    return {
        "user_id": current_user.sub,
        "role": current_user.role
    }

@app.get("/api/admin")
async def admin_only(current_user: TokenPayload = Depends(require_role("admin"))):
    """Только для администраторов"""
    return {"message": "Welcome, admin!", "user_id": current_user.sub}

@app.get("/api/protected")
async def protected_route(current_user: TokenPayload = Depends(get_current_user)):
    """Защищённый маршрут"""
    return {
        "message": f"Hello, user {current_user.sub}!",
        "role": current_user.role
    }
```

### Асимметричная подпись (RS256)

```python
import jwt
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.backends import default_backend

# Генерация ключей (делается один раз)
def generate_rsa_keys():
    private_key = rsa.generate_private_key(
        public_exponent=65537,
        key_size=2048,
        backend=default_backend()
    )

    private_pem = private_key.private_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PrivateFormat.PKCS8,
        encryption_algorithm=serialization.NoEncryption()
    )

    public_pem = private_key.public_key().public_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PublicFormat.SubjectPublicKeyInfo
    )

    return private_pem, public_pem

# Сохранённые ключи (в production — из файлов/vault)
PRIVATE_KEY = """-----BEGIN PRIVATE KEY-----
MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQ...
-----END PRIVATE KEY-----"""

PUBLIC_KEY = """-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...
-----END PUBLIC KEY-----"""

def create_jwt_rs256(user_id: int, role: str) -> str:
    """Создаёт JWT с RS256"""
    payload = {
        "sub": str(user_id),
        "role": role,
        "iat": datetime.utcnow(),
        "exp": datetime.utcnow() + timedelta(hours=1)
    }

    return jwt.encode(payload, PRIVATE_KEY, algorithm="RS256")

def decode_jwt_rs256(token: str) -> dict:
    """Декодирует JWT с RS256"""
    return jwt.decode(token, PUBLIC_KEY, algorithms=["RS256"])
```

### Node.js (jsonwebtoken)

```javascript
const jwt = require('jsonwebtoken');
const express = require('express');

const app = express();
app.use(express.json());

const SECRET_KEY = 'your-256-bit-secret';
const ACCESS_TOKEN_EXPIRE = '30m';
const REFRESH_TOKEN_EXPIRE = '7d';

// Создание токенов
function createAccessToken(userId, role) {
    return jwt.sign(
        { sub: userId, role, type: 'access' },
        SECRET_KEY,
        { expiresIn: ACCESS_TOKEN_EXPIRE }
    );
}

function createRefreshToken(userId, role) {
    return jwt.sign(
        { sub: userId, role, type: 'refresh' },
        SECRET_KEY,
        { expiresIn: REFRESH_TOKEN_EXPIRE }
    );
}

// Middleware для проверки JWT
function authenticateToken(req, res, next) {
    const authHeader = req.headers.authorization;

    if (!authHeader || !authHeader.startsWith('Bearer ')) {
        return res.status(401).json({ error: 'Token required' });
    }

    const token = authHeader.split(' ')[1];

    try {
        const decoded = jwt.verify(token, SECRET_KEY);

        if (decoded.type !== 'access') {
            return res.status(401).json({ error: 'Invalid token type' });
        }

        req.user = decoded;
        next();
    } catch (error) {
        if (error.name === 'TokenExpiredError') {
            return res.status(401).json({ error: 'Token expired' });
        }
        return res.status(401).json({ error: 'Invalid token' });
    }
}

// Middleware для проверки роли
function requireRole(role) {
    return (req, res, next) => {
        if (req.user.role !== role) {
            return res.status(403).json({ error: `Role '${role}' required` });
        }
        next();
    };
}

// Эндпоинты
app.post('/auth/login', (req, res) => {
    const { username, password } = req.body;

    // Проверка учётных данных (упрощённо)
    if (username === 'admin' && password === 'secret') {
        const accessToken = createAccessToken(1, 'admin');
        const refreshToken = createRefreshToken(1, 'admin');

        res.json({
            access_token: accessToken,
            refresh_token: refreshToken,
            token_type: 'bearer'
        });
    } else {
        res.status(401).json({ error: 'Invalid credentials' });
    }
});

app.post('/auth/refresh', (req, res) => {
    const { refresh_token } = req.body;

    try {
        const decoded = jwt.verify(refresh_token, SECRET_KEY);

        if (decoded.type !== 'refresh') {
            return res.status(401).json({ error: 'Invalid token type' });
        }

        const accessToken = createAccessToken(decoded.sub, decoded.role);
        const newRefreshToken = createRefreshToken(decoded.sub, decoded.role);

        res.json({
            access_token: accessToken,
            refresh_token: newRefreshToken,
            token_type: 'bearer'
        });
    } catch (error) {
        res.status(401).json({ error: 'Invalid refresh token' });
    }
});

app.get('/api/protected', authenticateToken, (req, res) => {
    res.json({ message: `Hello, user ${req.user.sub}!`, role: req.user.role });
});

app.get('/api/admin', authenticateToken, requireRole('admin'), (req, res) => {
    res.json({ message: 'Admin panel', userId: req.user.sub });
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

## Плюсы JWT

| Преимущество | Описание |
|--------------|----------|
| **Stateless** | Не требует хранения на сервере |
| **Самодостаточность** | Содержит все данные для авторизации |
| **Масштабируемость** | Любой сервер может проверить токен |
| **Микросервисы** | Идеально подходит для распределённых систем |
| **Производительность** | Нет обращений к БД для валидации |
| **Cross-domain** | Легко работает между доменами |

## Минусы JWT

| Недостаток | Описание |
|------------|----------|
| **Размер** | Больше, чем opaque токены |
| **Нельзя отозвать** | Токен валиден до истечения exp |
| **Чувствительные данные** | Payload легко декодируется |
| **Сложность ротации секрета** | При смене ключа все токены инвалидируются |
| **Нет logout** | Требует дополнительных механизмов (blacklist) |

## Когда использовать JWT

### Подходит для:

- Микросервисной архитектуры
- Stateless API
- Распределённых систем
- SSO (Single Sign-On)
- Когда нужна масштабируемость без shared state

### НЕ подходит для:

- Когда нужен мгновенный отзыв токенов
- Для хранения чувствительных данных
- Долгоживущих сессий без refresh механизма
- Когда размер токена критичен

## Вопросы безопасности

### 1. Никогда не храни чувствительные данные в payload

```python
# ПЛОХО: пароль в токене
payload = {"user_id": 1, "password": "secret123"}

# ХОРОШО: только необходимые данные
payload = {"sub": "1", "role": "admin"}
```

### 2. Всегда проверяй алгоритм

```python
# ПЛОХО: принимаем любой алгоритм
jwt.decode(token, SECRET_KEY, algorithms=["HS256", "none"])

# ХОРОШО: явно указываем разрешённые алгоритмы
jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
```

### 3. Blacklist для отзыва токенов

```python
import redis

# Redis для хранения отозванных токенов
blacklist = redis.Redis()

def revoke_token(token: str):
    """Добавляет токен в blacklist"""
    payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    jti = payload.get("jti")
    exp = payload.get("exp")

    # Храним до истечения токена
    ttl = exp - int(datetime.utcnow().timestamp())
    if ttl > 0:
        blacklist.setex(f"blacklist:{jti}", ttl, "revoked")

def is_token_revoked(token: str) -> bool:
    """Проверяет, отозван ли токен"""
    payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    jti = payload.get("jti")
    return blacklist.exists(f"blacklist:{jti}")

# Использование при валидации
def validate_token(token: str):
    payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])

    if is_token_revoked(token):
        raise ValueError("Token has been revoked")

    return payload
```

### 4. Защита от algorithm confusion

```python
# УЯЗВИМОСТЬ: "none" algorithm
# Атакующий может создать токен без подписи

# ЗАЩИТА: явный whitelist алгоритмов
ALLOWED_ALGORITHMS = ["HS256"]

def safe_decode(token: str):
    # Сначала проверяем header без верификации
    header = jwt.get_unverified_header(token)

    if header.get("alg") not in ALLOWED_ALGORITHMS:
        raise ValueError(f"Algorithm {header.get('alg')} not allowed")

    return jwt.decode(token, SECRET_KEY, algorithms=ALLOWED_ALGORITHMS)
```

### 5. Короткий TTL для access tokens

```python
# Рекомендуемые значения
ACCESS_TOKEN_EXPIRE = timedelta(minutes=15)   # Короткий
REFRESH_TOKEN_EXPIRE = timedelta(days=7)      # Длинный

# Access token обновляется через refresh token
# Это минимизирует окно уязвимости при краже access token
```

## Best Practices

1. **Используй короткий TTL для access tokens** — 15-30 минут максимум
2. **Реализуй refresh tokens** — для обновления access token
3. **Добавляй jti (JWT ID)** — для возможности отзыва
4. **Не храни чувствительные данные** — payload легко читается
5. **Проверяй audience (aud)** — защита от token confusion
6. **Используй асимметричные ключи в микросервисах** — каждый сервис имеет только публичный ключ
7. **Ротация секретов** — периодически меняй ключи подписи
8. **HTTPS only** — JWT не шифруется, только подписывается

## Типичные ошибки

### Ошибка 1: Слишком длинный TTL

```python
# ПЛОХО: токен живёт неделю
exp = datetime.utcnow() + timedelta(days=7)

# ХОРОШО: короткий access token + refresh
access_exp = datetime.utcnow() + timedelta(minutes=15)
refresh_exp = datetime.utcnow() + timedelta(days=7)
```

### Ошибка 2: Хранение токена в localStorage

```javascript
// ПЛОХО: уязвимо к XSS
localStorage.setItem('token', jwt);

// ЛУЧШЕ: HttpOnly cookie для refresh token
// Access token в памяти
let accessToken = null;  // В переменной, не в storage
```

### Ошибка 3: Отсутствие проверки типа токена

```python
# ПЛОХО: refresh token используется как access
payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
# Проверки типа нет!

# ХОРОШО: проверяем тип
payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
if payload.get("type") != "access":
    raise ValueError("Invalid token type")
```

### Ошибка 4: Игнорирование issuer и audience

```python
# ПЛОХО: не проверяем iss и aud
payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])

# ХОРОШО: полная проверка
payload = jwt.decode(
    token,
    SECRET_KEY,
    algorithms=["HS256"],
    issuer="https://auth.myapp.com",
    audience="https://api.myapp.com"
)
```

## JWT vs Session-based Auth

| Критерий | JWT | Session |
|----------|-----|---------|
| Хранение | Клиент | Сервер |
| Stateless | Да | Нет |
| Масштабирование | Легко | Требует shared storage |
| Отзыв | Сложно | Легко |
| Размер | Большой | Малый (session ID) |
| Производительность | Нет DB lookup | DB lookup |
| Микросервисы | Идеально | Сложно |

## Refresh Token Rotation

Паттерн для повышения безопасности:

```python
# При каждом refresh:
# 1. Проверяем refresh token
# 2. Инвалидируем старый refresh token
# 3. Выдаём новый refresh token

# Это защищает от повторного использования украденного refresh token
# Если атакующий использует украденный token, легитимный пользователь
# получит ошибку при следующем refresh и узнает о компрометации
```

## Резюме

JWT — это мощный инструмент для stateless аутентификации, особенно в микросервисной архитектуре. Однако он требует правильной реализации: короткий TTL, refresh tokens, проверка алгоритмов и claims. Помни, что JWT нельзя "выйти" — используй blacklist или короткий TTL для минимизации рисков.
