# OAuth 2.0

## Что такое OAuth 2.0?

**OAuth 2.0** — это открытый стандарт авторизации (RFC 6749), который позволяет приложениям получать ограниченный доступ к ресурсам пользователя на другом сервисе без передачи пароля. OAuth 2.0 решает проблему делегированного доступа: "Как разрешить приложению X доступ к моим данным на сервисе Y?"

## Ключевые концепции

### Роли в OAuth 2.0

| Роль | Описание | Пример |
|------|----------|--------|
| **Resource Owner** | Владелец ресурса (пользователь) | Пользователь Google |
| **Client** | Приложение, запрашивающее доступ | Твоё веб-приложение |
| **Authorization Server** | Сервер, выдающий токены | accounts.google.com |
| **Resource Server** | Сервер с защищёнными ресурсами | Google Calendar API |

### Типы токенов

| Токен | Назначение | Срок жизни |
|-------|------------|------------|
| **Access Token** | Доступ к ресурсам | Короткий (минуты-часы) |
| **Refresh Token** | Обновление access token | Длинный (дни-месяцы) |
| **ID Token** | Идентификация пользователя (OpenID Connect) | Короткий |

### Scopes (области доступа)

Scopes определяют, к каким ресурсам и операциям приложение получает доступ:

```
scope=read:user write:repo read:email
```

Примеры scopes:
- Google: `email`, `profile`, `calendar.readonly`
- GitHub: `repo`, `user:email`, `read:org`
- Twitter: `tweet.read`, `users.read`, `follows.write`

## Flows (потоки авторизации)

### 1. Authorization Code Flow (рекомендуемый)

Самый безопасный flow для серверных приложений.

```
    Пользователь              Client (Backend)          Authorization Server
         |                          |                           |
         |  1. Клик "Login with X"  |                           |
         |------------------------->|                           |
         |                          |                           |
         |  2. Redirect to Auth Server                          |
         |<-------------------------|                           |
         |                          |                           |
         |  3. Login + Consent      |                           |
         |-------------------------------------------------------->|
         |                          |                           |
         |  4. Redirect с auth code |                           |
         |<---------------------------------------------------------|
         |                          |                           |
         |  5. Redirect с code      |                           |
         |------------------------->|                           |
         |                          |                           |
         |                          |  6. Exchange code → token |
         |                          |-------------------------->|
         |                          |                           |
         |                          |  7. Access + Refresh tokens|
         |                          |<--------------------------|
         |                          |                           |
         |  8. Authenticated!       |                           |
         |<-------------------------|                           |
```

### 2. Authorization Code Flow + PKCE

Расширение для публичных клиентов (SPA, mobile apps).

```python
import hashlib
import base64
import secrets

def generate_pkce_pair():
    """Генерирует code_verifier и code_challenge"""
    # code_verifier: случайная строка 43-128 символов
    code_verifier = secrets.token_urlsafe(32)

    # code_challenge: SHA256 hash от verifier
    code_challenge = base64.urlsafe_b64encode(
        hashlib.sha256(code_verifier.encode()).digest()
    ).decode().rstrip('=')

    return code_verifier, code_challenge

# Использование
verifier, challenge = generate_pkce_pair()

# 1. При запросе авторизации отправляем challenge
auth_url = f"""
https://auth.example.com/authorize?
    response_type=code&
    client_id=YOUR_CLIENT_ID&
    redirect_uri=https://yourapp.com/callback&
    scope=read:user&
    code_challenge={challenge}&
    code_challenge_method=S256&
    state=random_state_string
"""

# 2. При обмене кода на токен отправляем verifier
token_request = {
    "grant_type": "authorization_code",
    "code": received_code,
    "redirect_uri": "https://yourapp.com/callback",
    "client_id": "YOUR_CLIENT_ID",
    "code_verifier": verifier  # Сервер проверит, что SHA256(verifier) == challenge
}
```

### 3. Client Credentials Flow

Для machine-to-machine взаимодействия (без пользователя).

```
    Client                          Authorization Server
       |                                    |
       |  POST /token                       |
       |  grant_type=client_credentials     |
       |  client_id=...                     |
       |  client_secret=...                 |
       |  scope=...                         |
       |----------------------------------->|
       |                                    |
       |  Access Token                      |
       |<-----------------------------------|
```

### 4. Resource Owner Password Credentials (устаревший)

Не рекомендуется — требует передачи пароля клиенту.

### 5. Implicit Flow (устаревший)

Заменён на Authorization Code + PKCE.

## Практические примеры кода

### Python (FastAPI) — OAuth2 Provider

```python
from fastapi import FastAPI, Depends, HTTPException, status, Form
from fastapi.security import OAuth2AuthorizationCodeBearer, OAuth2PasswordBearer
from pydantic import BaseModel
from datetime import datetime, timedelta
from typing import Optional
import secrets
import hashlib
import jwt

app = FastAPI()

# Конфигурация
SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30
REFRESH_TOKEN_EXPIRE_DAYS = 7

# Хранилища (в production — БД)
authorization_codes = {}
refresh_tokens = {}
clients = {
    "my-client-id": {
        "secret": "my-client-secret",
        "redirect_uris": ["http://localhost:3000/callback"],
        "scopes": ["read:user", "write:user"]
    }
}
users = {
    "user@example.com": {
        "id": "user-123",
        "password": "hashed_password",
        "name": "John Doe"
    }
}

# Модели
class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"
    expires_in: int
    refresh_token: Optional[str] = None
    scope: Optional[str] = None

class AuthorizationRequest(BaseModel):
    response_type: str
    client_id: str
    redirect_uri: str
    scope: str
    state: str
    code_challenge: Optional[str] = None
    code_challenge_method: Optional[str] = None

def create_access_token(user_id: str, scopes: list, expires_delta: timedelta):
    """Создаёт access token"""
    expire = datetime.utcnow() + expires_delta
    payload = {
        "sub": user_id,
        "scopes": scopes,
        "exp": expire,
        "iat": datetime.utcnow(),
        "type": "access"
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def create_refresh_token(user_id: str, scopes: list):
    """Создаёт refresh token"""
    token = secrets.token_urlsafe(32)
    refresh_tokens[token] = {
        "user_id": user_id,
        "scopes": scopes,
        "created_at": datetime.utcnow(),
        "expires_at": datetime.utcnow() + timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)
    }
    return token

def verify_pkce(code_verifier: str, code_challenge: str, method: str = "S256") -> bool:
    """Проверяет PKCE"""
    if method == "S256":
        computed = hashlib.sha256(code_verifier.encode()).digest()
        computed = secrets.base64.urlsafe_b64encode(computed).decode().rstrip('=')
        return computed == code_challenge
    elif method == "plain":
        return code_verifier == code_challenge
    return False

# Эндпоинт авторизации (упрощённый)
@app.get("/authorize")
async def authorize(
    response_type: str,
    client_id: str,
    redirect_uri: str,
    scope: str,
    state: str,
    code_challenge: Optional[str] = None,
    code_challenge_method: Optional[str] = "S256"
):
    """Authorization endpoint"""
    # Проверяем клиента
    client = clients.get(client_id)
    if not client:
        raise HTTPException(status_code=400, detail="Invalid client_id")

    if redirect_uri not in client["redirect_uris"]:
        raise HTTPException(status_code=400, detail="Invalid redirect_uri")

    # В реальности здесь показываем форму логина и consent screen
    # После успешной авторизации генерируем код

    auth_code = secrets.token_urlsafe(32)
    authorization_codes[auth_code] = {
        "client_id": client_id,
        "redirect_uri": redirect_uri,
        "scope": scope,
        "user_id": "user-123",  # После логина
        "code_challenge": code_challenge,
        "code_challenge_method": code_challenge_method,
        "created_at": datetime.utcnow(),
        "expires_at": datetime.utcnow() + timedelta(minutes=10)
    }

    # Редирект обратно к клиенту
    return {"redirect_to": f"{redirect_uri}?code={auth_code}&state={state}"}

# Эндпоинт обмена кода на токен
@app.post("/token", response_model=TokenResponse)
async def token(
    grant_type: str = Form(...),
    code: Optional[str] = Form(None),
    redirect_uri: Optional[str] = Form(None),
    client_id: str = Form(...),
    client_secret: Optional[str] = Form(None),
    code_verifier: Optional[str] = Form(None),
    refresh_token: Optional[str] = Form(None),
    scope: Optional[str] = Form(None)
):
    """Token endpoint"""

    # Проверяем клиента
    client = clients.get(client_id)
    if not client:
        raise HTTPException(status_code=400, detail="Invalid client")

    # Для confidential clients проверяем secret
    if client_secret and client_secret != client["secret"]:
        raise HTTPException(status_code=401, detail="Invalid client credentials")

    if grant_type == "authorization_code":
        # Обмен authorization code на токены
        code_data = authorization_codes.get(code)

        if not code_data:
            raise HTTPException(status_code=400, detail="Invalid authorization code")

        if datetime.utcnow() > code_data["expires_at"]:
            del authorization_codes[code]
            raise HTTPException(status_code=400, detail="Authorization code expired")

        if code_data["client_id"] != client_id:
            raise HTTPException(status_code=400, detail="Client mismatch")

        if code_data["redirect_uri"] != redirect_uri:
            raise HTTPException(status_code=400, detail="Redirect URI mismatch")

        # PKCE проверка
        if code_data.get("code_challenge"):
            if not code_verifier:
                raise HTTPException(status_code=400, detail="Code verifier required")
            if not verify_pkce(code_verifier, code_data["code_challenge"],
                              code_data.get("code_challenge_method", "S256")):
                raise HTTPException(status_code=400, detail="Invalid code verifier")

        # Удаляем использованный код
        del authorization_codes[code]

        # Создаём токены
        scopes = code_data["scope"].split()
        access_token = create_access_token(
            code_data["user_id"],
            scopes,
            timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
        )
        refresh_token_value = create_refresh_token(code_data["user_id"], scopes)

        return TokenResponse(
            access_token=access_token,
            expires_in=ACCESS_TOKEN_EXPIRE_MINUTES * 60,
            refresh_token=refresh_token_value,
            scope=code_data["scope"]
        )

    elif grant_type == "refresh_token":
        # Обновление access token
        token_data = refresh_tokens.get(refresh_token)

        if not token_data:
            raise HTTPException(status_code=400, detail="Invalid refresh token")

        if datetime.utcnow() > token_data["expires_at"]:
            del refresh_tokens[refresh_token]
            raise HTTPException(status_code=400, detail="Refresh token expired")

        # Создаём новый access token
        access_token = create_access_token(
            token_data["user_id"],
            token_data["scopes"],
            timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
        )

        # Опционально: ротация refresh token
        del refresh_tokens[refresh_token]
        new_refresh_token = create_refresh_token(
            token_data["user_id"],
            token_data["scopes"]
        )

        return TokenResponse(
            access_token=access_token,
            expires_in=ACCESS_TOKEN_EXPIRE_MINUTES * 60,
            refresh_token=new_refresh_token,
            scope=" ".join(token_data["scopes"])
        )

    elif grant_type == "client_credentials":
        # Machine-to-machine
        if not client_secret:
            raise HTTPException(status_code=401, detail="Client secret required")

        requested_scopes = scope.split() if scope else []

        access_token = create_access_token(
            f"client:{client_id}",
            requested_scopes,
            timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
        )

        return TokenResponse(
            access_token=access_token,
            expires_in=ACCESS_TOKEN_EXPIRE_MINUTES * 60,
            scope=scope
        )

    raise HTTPException(status_code=400, detail="Unsupported grant type")
```

### Python — OAuth2 Client (использование Google OAuth)

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import RedirectResponse
import httpx
import secrets

app = FastAPI()

# Конфигурация Google OAuth
GOOGLE_CLIENT_ID = "your-client-id.apps.googleusercontent.com"
GOOGLE_CLIENT_SECRET = "your-client-secret"
GOOGLE_REDIRECT_URI = "http://localhost:8000/callback"
GOOGLE_AUTH_URL = "https://accounts.google.com/o/oauth2/v2/auth"
GOOGLE_TOKEN_URL = "https://oauth2.googleapis.com/token"
GOOGLE_USERINFO_URL = "https://www.googleapis.com/oauth2/v2/userinfo"

# Хранилище state (в production — Redis)
oauth_states = {}

@app.get("/login/google")
async def login_google():
    """Начинает OAuth flow"""
    state = secrets.token_urlsafe(32)
    oauth_states[state] = {"created": True}

    params = {
        "client_id": GOOGLE_CLIENT_ID,
        "redirect_uri": GOOGLE_REDIRECT_URI,
        "response_type": "code",
        "scope": "openid email profile",
        "state": state,
        "access_type": "offline",  # Для получения refresh token
        "prompt": "consent"
    }

    auth_url = f"{GOOGLE_AUTH_URL}?" + "&".join(f"{k}={v}" for k, v in params.items())
    return RedirectResponse(auth_url)

@app.get("/callback")
async def oauth_callback(code: str, state: str):
    """Обрабатывает callback от Google"""
    # Проверяем state
    if state not in oauth_states:
        raise HTTPException(status_code=400, detail="Invalid state")

    del oauth_states[state]

    # Обмениваем code на токены
    async with httpx.AsyncClient() as client:
        token_response = await client.post(
            GOOGLE_TOKEN_URL,
            data={
                "grant_type": "authorization_code",
                "code": code,
                "redirect_uri": GOOGLE_REDIRECT_URI,
                "client_id": GOOGLE_CLIENT_ID,
                "client_secret": GOOGLE_CLIENT_SECRET
            }
        )

    if token_response.status_code != 200:
        raise HTTPException(status_code=400, detail="Failed to get tokens")

    tokens = token_response.json()
    access_token = tokens["access_token"]

    # Получаем информацию о пользователе
    async with httpx.AsyncClient() as client:
        userinfo_response = await client.get(
            GOOGLE_USERINFO_URL,
            headers={"Authorization": f"Bearer {access_token}"}
        )

    if userinfo_response.status_code != 200:
        raise HTTPException(status_code=400, detail="Failed to get user info")

    user_info = userinfo_response.json()

    # Здесь создаём или обновляем пользователя в БД
    # И создаём сессию или JWT для нашего приложения

    return {
        "message": "Successfully authenticated!",
        "user": user_info,
        "tokens": {
            "access_token": access_token,
            "refresh_token": tokens.get("refresh_token")
        }
    }
```

### Node.js — OAuth2 с Passport.js

```javascript
const express = require('express');
const passport = require('passport');
const GoogleStrategy = require('passport-google-oauth20').Strategy;
const session = require('express-session');

const app = express();

// Настройка сессий
app.use(session({
    secret: 'your-session-secret',
    resave: false,
    saveUninitialized: false
}));

app.use(passport.initialize());
app.use(passport.session());

// Сериализация пользователя
passport.serializeUser((user, done) => done(null, user));
passport.deserializeUser((user, done) => done(null, user));

// Настройка Google Strategy
passport.use(new GoogleStrategy({
    clientID: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackURL: 'http://localhost:3000/auth/google/callback'
}, async (accessToken, refreshToken, profile, done) => {
    // Здесь можно сохранить пользователя в БД
    const user = {
        id: profile.id,
        email: profile.emails[0].value,
        name: profile.displayName,
        accessToken,
        refreshToken
    };

    return done(null, user);
}));

// Маршруты
app.get('/auth/google',
    passport.authenticate('google', {
        scope: ['profile', 'email'],
        accessType: 'offline',
        prompt: 'consent'
    })
);

app.get('/auth/google/callback',
    passport.authenticate('google', { failureRedirect: '/login' }),
    (req, res) => {
        // Успешная аутентификация
        res.redirect('/dashboard');
    }
);

app.get('/dashboard', (req, res) => {
    if (!req.isAuthenticated()) {
        return res.redirect('/login');
    }
    res.json({ user: req.user });
});

app.get('/logout', (req, res) => {
    req.logout(() => {
        res.redirect('/');
    });
});

app.listen(3000);
```

## Плюсы OAuth 2.0

| Преимущество | Описание |
|--------------|----------|
| **Делегирование** | Пользователь не передаёт пароль приложению |
| **Granular access** | Scopes ограничивают доступ |
| **Отзыв доступа** | Пользователь может отозвать разрешения |
| **SSO** | Single Sign-On из коробки |
| **Стандарт** | Широко поддерживается |

## Минусы OAuth 2.0

| Недостаток | Описание |
|------------|----------|
| **Сложность** | Много flows и параметров |
| **Security зависит от реализации** | Легко сделать ошибку |
| **Не для аутентификации** | OAuth2 — авторизация, не аутентификация |
| **Token theft** | Украденный access token даёт полный доступ |

## Когда использовать

### Подходит для:

- "Login with Google/Facebook/GitHub"
- Интеграции с third-party сервисами
- Делегированного доступа к API
- SSO между приложениями

### НЕ подходит для:

- Простой аутентификации (используй OpenID Connect)
- First-party приложений (можно проще)
- Machine-to-machine без пользователя (Client Credentials достаточно)

## Вопросы безопасности

### 1. Всегда используй state parameter

```python
# ПЛОХО: нет state
auth_url = f"{AUTH_URL}?client_id={client_id}&redirect_uri={redirect}"

# ХОРОШО: state защищает от CSRF
state = secrets.token_urlsafe(32)
session['oauth_state'] = state
auth_url = f"{AUTH_URL}?client_id={client_id}&redirect_uri={redirect}&state={state}"

# При callback проверяем state
if request.args.get('state') != session.pop('oauth_state', None):
    raise SecurityError("Invalid state")
```

### 2. Используй PKCE для публичных клиентов

```python
# Для SPA и mobile apps PKCE обязателен
code_verifier = secrets.token_urlsafe(32)
code_challenge = base64.urlsafe_b64encode(
    hashlib.sha256(code_verifier.encode()).digest()
).rstrip(b'=').decode()

# code_challenge отправляется при авторизации
# code_verifier — при обмене кода на токен
```

### 3. Валидируй redirect_uri

```python
# Сервер авторизации должен строго проверять redirect_uri
allowed_redirects = ["https://app.example.com/callback"]

if redirect_uri not in allowed_redirects:
    raise SecurityError("Invalid redirect_uri")

# Не используй wildcard и relative URLs!
```

### 4. Храни токены безопасно

```javascript
// ПЛОХО: access token в localStorage
localStorage.setItem('access_token', token);

// ЛУЧШЕ:
// - Access token в памяти
// - Refresh token в HttpOnly cookie
```

## Best Practices

1. **Используй Authorization Code + PKCE** — даже для confidential clients
2. **Минимальные scopes** — запрашивай только необходимые разрешения
3. **Валидируй state** — защита от CSRF
4. **Короткий TTL для access tokens** — минимизация ущерба при краже
5. **Используй refresh token rotation** — каждый refresh выдаёт новый refresh token
6. **Проверяй redirect_uri** — точное совпадение, без wildcards
7. **Логируй OAuth события** — для аудита и обнаружения аномалий

## Типичные ошибки

### Ошибка 1: Отсутствие PKCE для SPA

```javascript
// ПЛОХО: Authorization Code без PKCE
const authUrl = `${AUTH_URL}?response_type=code&client_id=${clientId}`;

// ХОРОШО: с PKCE
const codeVerifier = generateRandomString(43);
const codeChallenge = await sha256(codeVerifier);
const authUrl = `${AUTH_URL}?response_type=code&client_id=${clientId}&code_challenge=${codeChallenge}&code_challenge_method=S256`;
```

### Ошибка 2: Использование Implicit Flow

```javascript
// УСТАРЕЛО: Implicit flow (token в URL)
// response_type=token

// АКТУАЛЬНО: Authorization Code + PKCE
// response_type=code
```

### Ошибка 3: Хранение client_secret на клиенте

```javascript
// ПЛОХО: secret в frontend коде
const clientSecret = "my-secret-key"; // Видно всем!

// ХОРОШО: secret только на backend
// Frontend использует PKCE без secret
```

## OAuth 2.0 vs OpenID Connect

| Аспект | OAuth 2.0 | OpenID Connect |
|--------|-----------|----------------|
| Назначение | Авторизация | Аутентификация + авторизация |
| Кто пользователь? | Не определено | ID Token с информацией |
| Стандартные scopes | Нет | `openid`, `profile`, `email` |
| UserInfo endpoint | Нет | Да |
| Основа | — | Надстройка над OAuth 2.0 |

## Резюме

OAuth 2.0 — это стандарт для делегированной авторизации. Используй его для интеграции с внешними сервисами и "Login with X". Для аутентификации добавь OpenID Connect. Главное — правильно выбрать flow (Authorization Code + PKCE) и соблюдать security best practices.
