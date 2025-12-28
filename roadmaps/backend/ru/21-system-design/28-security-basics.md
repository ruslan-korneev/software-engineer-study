# Основы безопасности (Security Basics)

[prev: 27-reliability-patterns](./27-reliability-patterns.md) | [next: 29-case-studies](./29-case-studies.md)

---

## Введение

Безопасность — критически важный аспект проектирования систем. Уязвимости приводят к утечкам данных, финансовым потерям и репутационному ущербу. В этом разделе рассмотрим основные концепции и практики безопасности backend-систем.

Три столпа информационной безопасности (CIA Triad):
- **Confidentiality (Конфиденциальность)** — данные доступны только авторизованным пользователям
- **Integrity (Целостность)** — данные не могут быть изменены без авторизации
- **Availability (Доступность)** — система доступна для легитимных пользователей

---

## Аутентификация (Authentication)

### Концепция

**Аутентификация** — процесс проверки личности пользователя. Отвечает на вопрос: "Кто вы?"

### Факторы аутентификации

```
┌─────────────────────────────────────────────────────────────────┐
│                   Authentication Factors                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Something you KNOW (Что-то, что вы знаете)                  │
│     - Пароль                                                     │
│     - PIN-код                                                    │
│     - Секретный вопрос                                          │
│                                                                  │
│  2. Something you HAVE (Что-то, что вы имеете)                  │
│     - Мобильный телефон (SMS, TOTP)                             │
│     - Hardware token (YubiKey)                                   │
│     - Smart card                                                 │
│                                                                  │
│  3. Something you ARE (Что-то, чем вы являетесь)                │
│     - Отпечаток пальца                                          │
│     - Распознавание лица                                        │
│     - Сканирование радужки глаза                                │
│                                                                  │
│  MFA = Multi-Factor Authentication                               │
│  Использование 2+ факторов для повышения безопасности           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Хранение паролей

```python
# НИКОГДА не храните пароли в открытом виде!

# ПЛОХО
def store_password_bad(password: str):
    db.save(password)  # Ужасно!

def store_password_also_bad(password: str):
    import hashlib
    hashed = hashlib.sha256(password.encode()).hexdigest()
    db.save(hashed)  # Плохо: без соли, уязвим к rainbow tables


# ПРАВИЛЬНО: используйте bcrypt, Argon2 или scrypt
import bcrypt
from argon2 import PasswordHasher


# Вариант 1: bcrypt
def hash_password_bcrypt(password: str) -> bytes:
    """Хеширование пароля с bcrypt."""
    salt = bcrypt.gensalt(rounds=12)  # 12 раундов — хороший баланс
    return bcrypt.hashpw(password.encode('utf-8'), salt)


def verify_password_bcrypt(password: str, hashed: bytes) -> bool:
    """Проверка пароля."""
    return bcrypt.checkpw(password.encode('utf-8'), hashed)


# Вариант 2: Argon2 (рекомендуется OWASP)
ph = PasswordHasher(
    time_cost=2,        # Количество итераций
    memory_cost=65536,  # 64 MB памяти
    parallelism=4       # Количество потоков
)


def hash_password_argon2(password: str) -> str:
    """Хеширование пароля с Argon2."""
    return ph.hash(password)


def verify_password_argon2(password: str, hashed: str) -> bool:
    """Проверка пароля с Argon2."""
    try:
        ph.verify(hashed, password)
        return True
    except Exception:
        return False


# Пример использования в регистрации
from pydantic import BaseModel, validator
import re


class UserRegistration(BaseModel):
    email: str
    password: str

    @validator('password')
    def validate_password(cls, v):
        if len(v) < 12:
            raise ValueError('Password must be at least 12 characters')
        if not re.search(r'[A-Z]', v):
            raise ValueError('Password must contain uppercase letter')
        if not re.search(r'[a-z]', v):
            raise ValueError('Password must contain lowercase letter')
        if not re.search(r'\d', v):
            raise ValueError('Password must contain digit')
        if not re.search(r'[!@#$%^&*(),.?":{}|<>]', v):
            raise ValueError('Password must contain special character')
        return v


def register_user(data: UserRegistration) -> dict:
    # Хешируем пароль перед сохранением
    hashed_password = hash_password_argon2(data.password)

    user = User(
        email=data.email,
        password_hash=hashed_password
    )
    db.save(user)

    return {"status": "registered"}
```

### Session-based Authentication

```python
"""
Session-based аутентификация:
1. Пользователь отправляет credentials
2. Сервер проверяет и создаёт session
3. Session ID хранится в cookie
4. При каждом запросе браузер отправляет cookie
5. Сервер проверяет session
"""

from flask import Flask, session, request, jsonify
from datetime import timedelta
import secrets

app = Flask(__name__)
app.secret_key = secrets.token_hex(32)
app.config['SESSION_COOKIE_SECURE'] = True      # Только HTTPS
app.config['SESSION_COOKIE_HTTPONLY'] = True    # Недоступен для JS
app.config['SESSION_COOKIE_SAMESITE'] = 'Lax'   # Защита от CSRF
app.config['PERMANENT_SESSION_LIFETIME'] = timedelta(hours=2)


@app.route('/login', methods=['POST'])
def login():
    data = request.json
    user = authenticate_user(data['email'], data['password'])

    if user:
        session['user_id'] = user.id
        session['authenticated'] = True
        session.permanent = True
        return jsonify({'status': 'success'})

    return jsonify({'error': 'Invalid credentials'}), 401


@app.route('/logout', methods=['POST'])
def logout():
    session.clear()
    return jsonify({'status': 'logged out'})


@app.route('/protected')
def protected():
    if not session.get('authenticated'):
        return jsonify({'error': 'Unauthorized'}), 401

    return jsonify({'data': 'secret'})


# Redis Session Store для масштабирования
from flask_session import Session
import redis

app.config['SESSION_TYPE'] = 'redis'
app.config['SESSION_REDIS'] = redis.Redis(host='localhost', port=6379)
Session(app)
```

---

## Авторизация (Authorization)

### Концепция

**Авторизация** — процесс проверки прав доступа. Отвечает на вопрос: "Что вы можете делать?"

### Модели авторизации

```
┌─────────────────────────────────────────────────────────────────┐
│                   Authorization Models                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. RBAC (Role-Based Access Control)                            │
│     User → Role → Permissions                                   │
│     Пример: Admin, Editor, Viewer                               │
│                                                                  │
│  2. ABAC (Attribute-Based Access Control)                       │
│     Решения на основе атрибутов:                                │
│     - User attributes (department, level)                       │
│     - Resource attributes (owner, classification)               │
│     - Environment (time, location)                              │
│                                                                  │
│  3. ACL (Access Control Lists)                                  │
│     Списки прав для каждого ресурса                             │
│     Resource → [(User, Permissions), ...]                       │
│                                                                  │
│  4. Policy-Based Access Control                                 │
│     Централизованные политики (OPA, Casbin)                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Реализация RBAC

```python
from enum import Enum
from dataclasses import dataclass
from functools import wraps
from typing import Set, List
from flask import g, jsonify


class Permission(Enum):
    # Посты
    CREATE_POST = "post:create"
    READ_POST = "post:read"
    UPDATE_POST = "post:update"
    DELETE_POST = "post:delete"

    # Пользователи
    READ_USER = "user:read"
    UPDATE_USER = "user:update"
    DELETE_USER = "user:delete"

    # Администрирование
    MANAGE_ROLES = "admin:manage_roles"
    VIEW_ANALYTICS = "admin:view_analytics"


@dataclass
class Role:
    name: str
    permissions: Set[Permission]


# Определение ролей
ROLES = {
    "viewer": Role(
        name="viewer",
        permissions={Permission.READ_POST, Permission.READ_USER}
    ),
    "editor": Role(
        name="editor",
        permissions={
            Permission.READ_POST, Permission.CREATE_POST,
            Permission.UPDATE_POST, Permission.READ_USER
        }
    ),
    "admin": Role(
        name="admin",
        permissions={perm for perm in Permission}  # Все права
    ),
}


class RBACService:
    def __init__(self):
        self.user_roles: dict = {}  # user_id -> List[str]

    def assign_role(self, user_id: int, role_name: str):
        if role_name not in ROLES:
            raise ValueError(f"Unknown role: {role_name}")

        if user_id not in self.user_roles:
            self.user_roles[user_id] = []

        if role_name not in self.user_roles[user_id]:
            self.user_roles[user_id].append(role_name)

    def get_user_permissions(self, user_id: int) -> Set[Permission]:
        role_names = self.user_roles.get(user_id, ["viewer"])
        permissions = set()

        for role_name in role_names:
            if role_name in ROLES:
                permissions.update(ROLES[role_name].permissions)

        return permissions

    def has_permission(self, user_id: int, permission: Permission) -> bool:
        return permission in self.get_user_permissions(user_id)


rbac = RBACService()


# Декоратор для проверки прав
def require_permission(*permissions: Permission):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            user_id = g.get('user_id')
            if not user_id:
                return jsonify({'error': 'Unauthorized'}), 401

            user_permissions = rbac.get_user_permissions(user_id)

            for perm in permissions:
                if perm not in user_permissions:
                    return jsonify({
                        'error': 'Forbidden',
                        'required': perm.value
                    }), 403

            return f(*args, **kwargs)
        return decorated_function
    return decorator


# Использование
@app.route('/posts', methods=['POST'])
@require_permission(Permission.CREATE_POST)
def create_post():
    # Только пользователи с правом CREATE_POST
    return jsonify({'status': 'created'})


@app.route('/admin/analytics')
@require_permission(Permission.VIEW_ANALYTICS)
def view_analytics():
    # Только администраторы
    return jsonify({'data': 'analytics'})
```

### Resource-based Authorization

```python
"""
Проверка прав на конкретный ресурс.
Пример: пользователь может редактировать только свои посты.
"""

from dataclasses import dataclass
from typing import Optional


@dataclass
class Post:
    id: int
    title: str
    content: str
    author_id: int


class PostService:
    def __init__(self, rbac: RBACService):
        self.rbac = rbac

    def can_update(self, user_id: int, post: Post) -> bool:
        # Админ может редактировать всё
        if self.rbac.has_permission(user_id, Permission.UPDATE_POST):
            if "admin" in self.rbac.user_roles.get(user_id, []):
                return True

        # Автор может редактировать свой пост
        return post.author_id == user_id

    def can_delete(self, user_id: int, post: Post) -> bool:
        # Только админ или автор
        if self.rbac.has_permission(user_id, Permission.DELETE_POST):
            if "admin" in self.rbac.user_roles.get(user_id, []):
                return True

        return post.author_id == user_id

    def update_post(self, user_id: int, post_id: int, data: dict) -> Post:
        post = db.get_post(post_id)
        if not post:
            raise NotFoundError("Post not found")

        if not self.can_update(user_id, post):
            raise ForbiddenError("Cannot update this post")

        post.title = data.get('title', post.title)
        post.content = data.get('content', post.content)
        db.save(post)

        return post
```

---

## OAuth 2.0

### Концепция

OAuth 2.0 — протокол авторизации, позволяющий приложениям получать ограниченный доступ к ресурсам пользователя без передачи пароля.

### Роли в OAuth 2.0

```
┌─────────────────────────────────────────────────────────────────┐
│                      OAuth 2.0 Roles                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Resource Owner (Владелец ресурса)                              │
│  └── Пользователь, чьи данные защищаются                        │
│                                                                  │
│  Client (Клиент)                                                │
│  └── Приложение, запрашивающее доступ                           │
│                                                                  │
│  Authorization Server (Сервер авторизации)                      │
│  └── Выдаёт токены после аутентификации                         │
│                                                                  │
│  Resource Server (Сервер ресурсов)                              │
│  └── Хранит защищённые ресурсы, проверяет токены                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Authorization Code Flow (рекомендуемый)

```
┌──────────┐                                   ┌─────────────────┐
│          │ (1) Authorization Request         │                 │
│          │ ────────────────────────────────► │                 │
│          │                                   │  Authorization  │
│ Resource │ (2) User authenticates            │     Server      │
│  Owner   │                                   │                 │
│ (Browser)│ (3) Authorization Code            │                 │
│          │ ◄──────────────────────────────── │                 │
└──────────┘                                   └─────────────────┘
      │                                               ▲
      │ (4) Code                                      │
      ▼                                               │
┌──────────┐                                          │
│          │ (5) Exchange Code for Token              │
│  Client  │ ─────────────────────────────────────────┘
│  (App)   │
│          │ (6) Access Token + Refresh Token
│          │ ◄─────────────────────────────────────────
└──────────┘
      │
      │ (7) Access Protected Resource
      ▼
┌─────────────────┐
│    Resource     │
│     Server      │
└─────────────────┘
```

### Реализация OAuth 2.0 сервера

```python
"""
Пример реализации OAuth 2.0 Authorization Server.
В продакшене используйте библиотеки: authlib, python-oauth2
"""

from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import Optional, List
import secrets
import hashlib
import base64
from urllib.parse import urlencode
from fastapi import FastAPI, HTTPException, Request, Form
from fastapi.responses import RedirectResponse

app = FastAPI()


@dataclass
class OAuthClient:
    client_id: str
    client_secret_hash: str
    redirect_uris: List[str]
    scopes: List[str]
    name: str


@dataclass
class AuthorizationCode:
    code: str
    client_id: str
    redirect_uri: str
    scopes: List[str]
    user_id: int
    code_challenge: Optional[str]  # PKCE
    code_challenge_method: Optional[str]
    expires_at: datetime


@dataclass
class AccessToken:
    token: str
    client_id: str
    user_id: int
    scopes: List[str]
    expires_at: datetime


# Хранилища (в реальности — база данных)
clients_db: dict[str, OAuthClient] = {}
codes_db: dict[str, AuthorizationCode] = {}
tokens_db: dict[str, AccessToken] = {}


class OAuthServer:
    def __init__(self):
        self.code_expiry = timedelta(minutes=10)
        self.token_expiry = timedelta(hours=1)

    def register_client(
        self,
        name: str,
        redirect_uris: List[str],
        scopes: List[str]
    ) -> tuple[str, str]:
        """Регистрация OAuth клиента."""
        client_id = secrets.token_urlsafe(16)
        client_secret = secrets.token_urlsafe(32)
        client_secret_hash = hashlib.sha256(client_secret.encode()).hexdigest()

        client = OAuthClient(
            client_id=client_id,
            client_secret_hash=client_secret_hash,
            redirect_uris=redirect_uris,
            scopes=scopes,
            name=name
        )
        clients_db[client_id] = client

        return client_id, client_secret

    def authorize(
        self,
        client_id: str,
        redirect_uri: str,
        scopes: List[str],
        user_id: int,
        state: str,
        code_challenge: Optional[str] = None,
        code_challenge_method: Optional[str] = None
    ) -> str:
        """Создание authorization code."""
        client = clients_db.get(client_id)
        if not client:
            raise ValueError("Unknown client")

        if redirect_uri not in client.redirect_uris:
            raise ValueError("Invalid redirect_uri")

        # Проверяем запрошенные scopes
        for scope in scopes:
            if scope not in client.scopes:
                raise ValueError(f"Invalid scope: {scope}")

        # Генерируем код
        code = secrets.token_urlsafe(32)

        auth_code = AuthorizationCode(
            code=code,
            client_id=client_id,
            redirect_uri=redirect_uri,
            scopes=scopes,
            user_id=user_id,
            code_challenge=code_challenge,
            code_challenge_method=code_challenge_method,
            expires_at=datetime.utcnow() + self.code_expiry
        )
        codes_db[code] = auth_code

        # Формируем redirect URL
        params = {"code": code, "state": state}
        return f"{redirect_uri}?{urlencode(params)}"

    def exchange_code(
        self,
        code: str,
        client_id: str,
        client_secret: str,
        redirect_uri: str,
        code_verifier: Optional[str] = None
    ) -> dict:
        """Обмен authorization code на access token."""
        auth_code = codes_db.get(code)
        if not auth_code:
            raise ValueError("Invalid authorization code")

        if auth_code.expires_at < datetime.utcnow():
            del codes_db[code]
            raise ValueError("Authorization code expired")

        if auth_code.client_id != client_id:
            raise ValueError("Client mismatch")

        if auth_code.redirect_uri != redirect_uri:
            raise ValueError("Redirect URI mismatch")

        # Проверяем client_secret
        client = clients_db.get(client_id)
        secret_hash = hashlib.sha256(client_secret.encode()).hexdigest()
        if secret_hash != client.client_secret_hash:
            raise ValueError("Invalid client credentials")

        # PKCE verification
        if auth_code.code_challenge:
            if not code_verifier:
                raise ValueError("Code verifier required")

            if auth_code.code_challenge_method == "S256":
                challenge = base64.urlsafe_b64encode(
                    hashlib.sha256(code_verifier.encode()).digest()
                ).rstrip(b'=').decode()
            else:
                challenge = code_verifier

            if challenge != auth_code.code_challenge:
                raise ValueError("Invalid code verifier")

        # Создаём токен
        access_token = secrets.token_urlsafe(32)
        refresh_token = secrets.token_urlsafe(32)

        token = AccessToken(
            token=access_token,
            client_id=client_id,
            user_id=auth_code.user_id,
            scopes=auth_code.scopes,
            expires_at=datetime.utcnow() + self.token_expiry
        )
        tokens_db[access_token] = token

        # Удаляем использованный код
        del codes_db[code]

        return {
            "access_token": access_token,
            "token_type": "Bearer",
            "expires_in": int(self.token_expiry.total_seconds()),
            "refresh_token": refresh_token,
            "scope": " ".join(auth_code.scopes)
        }

    def validate_token(self, token: str) -> Optional[AccessToken]:
        """Валидация access token."""
        access_token = tokens_db.get(token)
        if not access_token:
            return None

        if access_token.expires_at < datetime.utcnow():
            del tokens_db[token]
            return None

        return access_token


oauth_server = OAuthServer()


# API Endpoints

@app.get("/oauth/authorize")
async def authorize_endpoint(
    client_id: str,
    redirect_uri: str,
    response_type: str,
    scope: str,
    state: str,
    code_challenge: Optional[str] = None,
    code_challenge_method: Optional[str] = None
):
    """Authorization endpoint — показывает форму согласия."""
    if response_type != "code":
        raise HTTPException(400, "Unsupported response type")

    # В реальности здесь показываем форму согласия пользователю
    # После согласия вызываем authorize()
    return {"message": "Show consent form", "client_id": client_id}


@app.post("/oauth/authorize")
async def authorize_submit(
    client_id: str = Form(...),
    redirect_uri: str = Form(...),
    scope: str = Form(...),
    state: str = Form(...),
    code_challenge: Optional[str] = Form(None),
    code_challenge_method: Optional[str] = Form(None),
    user_id: int = Form(...)  # Из сессии после логина
):
    """Обработка согласия пользователя."""
    try:
        redirect_url = oauth_server.authorize(
            client_id=client_id,
            redirect_uri=redirect_uri,
            scopes=scope.split(),
            user_id=user_id,
            state=state,
            code_challenge=code_challenge,
            code_challenge_method=code_challenge_method
        )
        return RedirectResponse(redirect_url, status_code=302)
    except ValueError as e:
        raise HTTPException(400, str(e))


@app.post("/oauth/token")
async def token_endpoint(
    grant_type: str = Form(...),
    code: str = Form(None),
    redirect_uri: str = Form(None),
    client_id: str = Form(...),
    client_secret: str = Form(...),
    code_verifier: Optional[str] = Form(None)
):
    """Token endpoint — обмен кода на токен."""
    if grant_type != "authorization_code":
        raise HTTPException(400, "Unsupported grant type")

    try:
        tokens = oauth_server.exchange_code(
            code=code,
            client_id=client_id,
            client_secret=client_secret,
            redirect_uri=redirect_uri,
            code_verifier=code_verifier
        )
        return tokens
    except ValueError as e:
        raise HTTPException(400, str(e))
```

---

## JWT (JSON Web Tokens)

### Концепция

JWT — компактный, самодостаточный способ передачи информации между сторонами в виде JSON-объекта. Информация верифицируется с помощью цифровой подписи.

### Структура JWT

```
┌──────────────────────────────────────────────────────────────┐
│                        JWT Structure                          │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Header.Payload.Signature                                     │
│                                                               │
│  ┌─────────────────┐                                         │
│  │     Header      │  Base64Url encoded                      │
│  │ {               │                                          │
│  │   "alg": "HS256",  ← Алгоритм подписи                    │
│  │   "typ": "JWT"     ← Тип токена                          │
│  │ }               │                                          │
│  └────────┬────────┘                                         │
│           │                                                   │
│  ┌────────▼────────┐                                         │
│  │     Payload     │  Base64Url encoded                      │
│  │ {               │                                          │
│  │   "sub": "1234", ← Subject (user ID)                      │
│  │   "name": "John", ← Custom claim                          │
│  │   "iat": 1516239022, ← Issued at                         │
│  │   "exp": 1516242622  ← Expiration                        │
│  │ }               │                                          │
│  └────────┬────────┘                                         │
│           │                                                   │
│  ┌────────▼────────┐                                         │
│  │    Signature    │                                         │
│  │ HMACSHA256(      │                                         │
│  │   base64UrlEncode(header) + "." +                         │
│  │   base64UrlEncode(payload),                               │
│  │   secret                                                   │
│  │ )                │                                         │
│  └─────────────────┘                                         │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Реализация JWT

```python
import jwt
from datetime import datetime, timedelta
from typing import Optional, Dict, Any
from dataclasses import dataclass
from fastapi import FastAPI, Depends, HTTPException, Header
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

app = FastAPI()


@dataclass
class JWTConfig:
    secret_key: str
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 30
    refresh_token_expire_days: int = 7
    issuer: str = "my-app"


config = JWTConfig(
    secret_key="your-super-secret-key-min-32-chars!",  # В продакшене из env
    algorithm="HS256"
)


class JWTService:
    def __init__(self, config: JWTConfig):
        self.config = config

    def create_access_token(
        self,
        user_id: int,
        roles: list[str],
        additional_claims: Dict[str, Any] = None
    ) -> str:
        """Создание access token."""
        now = datetime.utcnow()
        expire = now + timedelta(minutes=self.config.access_token_expire_minutes)

        payload = {
            "sub": str(user_id),
            "roles": roles,
            "type": "access",
            "iat": now,
            "exp": expire,
            "iss": self.config.issuer
        }

        if additional_claims:
            payload.update(additional_claims)

        return jwt.encode(
            payload,
            self.config.secret_key,
            algorithm=self.config.algorithm
        )

    def create_refresh_token(self, user_id: int) -> str:
        """Создание refresh token."""
        now = datetime.utcnow()
        expire = now + timedelta(days=self.config.refresh_token_expire_days)

        payload = {
            "sub": str(user_id),
            "type": "refresh",
            "iat": now,
            "exp": expire,
            "iss": self.config.issuer
        }

        return jwt.encode(
            payload,
            self.config.secret_key,
            algorithm=self.config.algorithm
        )

    def decode_token(self, token: str) -> Optional[Dict[str, Any]]:
        """Декодирование и валидация токена."""
        try:
            payload = jwt.decode(
                token,
                self.config.secret_key,
                algorithms=[self.config.algorithm],
                issuer=self.config.issuer
            )
            return payload
        except jwt.ExpiredSignatureError:
            raise ValueError("Token has expired")
        except jwt.InvalidTokenError as e:
            raise ValueError(f"Invalid token: {e}")

    def refresh_tokens(self, refresh_token: str) -> Dict[str, str]:
        """Обновление пары токенов."""
        payload = self.decode_token(refresh_token)

        if payload.get("type") != "refresh":
            raise ValueError("Invalid token type")

        user_id = int(payload["sub"])

        # В реальности проверяем, не отозван ли refresh token
        # и получаем актуальные роли из БД
        user = get_user_by_id(user_id)

        return {
            "access_token": self.create_access_token(user_id, user.roles),
            "refresh_token": self.create_refresh_token(user_id)
        }


jwt_service = JWTService(config)


# Dependency для получения текущего пользователя
security = HTTPBearer()


@dataclass
class CurrentUser:
    id: int
    roles: list[str]


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> CurrentUser:
    """Dependency для получения текущего пользователя из JWT."""
    try:
        payload = jwt_service.decode_token(credentials.credentials)

        if payload.get("type") != "access":
            raise HTTPException(401, "Invalid token type")

        return CurrentUser(
            id=int(payload["sub"]),
            roles=payload.get("roles", [])
        )
    except ValueError as e:
        raise HTTPException(401, str(e))


def require_roles(*required_roles: str):
    """Dependency для проверки ролей."""
    async def check_roles(user: CurrentUser = Depends(get_current_user)):
        for role in required_roles:
            if role not in user.roles:
                raise HTTPException(403, f"Role '{role}' required")
        return user
    return check_roles


# API endpoints

@app.post("/auth/login")
async def login(email: str, password: str):
    """Аутентификация и получение токенов."""
    user = authenticate_user(email, password)
    if not user:
        raise HTTPException(401, "Invalid credentials")

    return {
        "access_token": jwt_service.create_access_token(user.id, user.roles),
        "refresh_token": jwt_service.create_refresh_token(user.id),
        "token_type": "Bearer"
    }


@app.post("/auth/refresh")
async def refresh(refresh_token: str):
    """Обновление токенов."""
    try:
        tokens = jwt_service.refresh_tokens(refresh_token)
        return {**tokens, "token_type": "Bearer"}
    except ValueError as e:
        raise HTTPException(401, str(e))


@app.get("/users/me")
async def get_me(user: CurrentUser = Depends(get_current_user)):
    """Получение информации о текущем пользователе."""
    return {"user_id": user.id, "roles": user.roles}


@app.get("/admin/users")
async def list_users(user: CurrentUser = Depends(require_roles("admin"))):
    """Только для администраторов."""
    return {"users": []}
```

### JWT vs Sessions

```
┌─────────────────────────────────────────────────────────────────┐
│                    JWT vs Sessions                               │
├────────────────────┬────────────────────────────────────────────┤
│                    │                                             │
│  Sessions          │  JWT                                        │
│  ─────────         │  ───                                        │
│                    │                                             │
│  ✓ Можно отозвать  │  ✗ Нельзя отозвать до истечения            │
│    немедленно      │    (нужен blocklist)                       │
│                    │                                             │
│  ✗ Требует хранили │  ✓ Stateless — не нужно хранилище          │
│    ще (Redis)      │                                             │
│                    │                                             │
│  ✓ Меньше payload  │  ✗ Большой размер токена                   │
│                    │                                             │
│  ✗ Сложнее масшта  │  ✓ Легко масштабировать                    │
│    бировать        │                                             │
│                    │                                             │
│  Лучше для:        │  Лучше для:                                 │
│  - Web-приложений  │  - API                                      │
│  - Когда нужен     │  - Микросервисов                           │
│    быстрый logout  │  - Mobile apps                              │
│                    │  - Распределённых систем                   │
│                    │                                             │
└────────────────────┴────────────────────────────────────────────┘
```

---

## HTTPS и TLS

### Концепция

HTTPS (HTTP Secure) использует TLS (Transport Layer Security) для шифрования трафика между клиентом и сервером.

### TLS Handshake

```
Client                                              Server
   │                                                   │
   │────────── 1. ClientHello ────────────────────────►│
   │           (supported cipher suites, random)       │
   │                                                   │
   │◄───────── 2. ServerHello ─────────────────────────│
   │           (chosen cipher suite, random)           │
   │                                                   │
   │◄───────── 3. Certificate ─────────────────────────│
   │           (server's public key)                   │
   │                                                   │
   │◄───────── 4. ServerHelloDone ─────────────────────│
   │                                                   │
   │────────── 5. ClientKeyExchange ──────────────────►│
   │           (pre-master secret encrypted with       │
   │            server's public key)                   │
   │                                                   │
   │────────── 6. ChangeCipherSpec ───────────────────►│
   │                                                   │
   │────────── 7. Finished ───────────────────────────►│
   │                                                   │
   │◄───────── 8. ChangeCipherSpec ────────────────────│
   │                                                   │
   │◄───────── 9. Finished ────────────────────────────│
   │                                                   │
   │◄─────────── Encrypted Data ──────────────────────►│
```

### Настройка HTTPS в Python

```python
"""
Настройка HTTPS для FastAPI/Uvicorn.
"""

# 1. Генерация самоподписанного сертификата (для разработки)
# openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes

# 2. Запуск с SSL
# uvicorn main:app --ssl-keyfile=key.pem --ssl-certfile=cert.pem

# 3. В продакшене используйте Let's Encrypt + Nginx/Caddy

# Nginx конфигурация
"""
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Современные настройки TLS
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;

    # HSTS
    add_header Strict-Transport-Security "max-age=63072000" always;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Редирект HTTP → HTTPS
server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}
"""


# Проверка HTTPS в приложении
from fastapi import FastAPI, Request
from starlette.middleware.httpsredirect import HTTPSRedirectMiddleware

app = FastAPI()

# Принудительный редирект на HTTPS
app.add_middleware(HTTPSRedirectMiddleware)


# Проверка, что запрос пришёл по HTTPS
@app.middleware("http")
async def check_https(request: Request, call_next):
    # X-Forwarded-Proto устанавливается reverse proxy
    if request.headers.get("X-Forwarded-Proto") != "https":
        if not request.url.scheme == "https":
            # В продакшене можно редиректить
            pass

    response = await call_next(request)
    return response
```

### Security Headers

```python
from fastapi import FastAPI
from starlette.middleware.base import BaseHTTPMiddleware


class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        response = await call_next(request)

        # Предотвращает атаки через clickjacking
        response.headers["X-Frame-Options"] = "DENY"

        # Предотвращает MIME-sniffing
        response.headers["X-Content-Type-Options"] = "nosniff"

        # XSS Protection (для старых браузеров)
        response.headers["X-XSS-Protection"] = "1; mode=block"

        # HSTS — принудительный HTTPS
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"

        # Контроль информации в Referer
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"

        # Content Security Policy
        response.headers["Content-Security-Policy"] = (
            "default-src 'self'; "
            "script-src 'self'; "
            "style-src 'self' 'unsafe-inline'; "
            "img-src 'self' data: https:; "
            "font-src 'self'; "
            "connect-src 'self'; "
            "frame-ancestors 'none';"
        )

        # Permissions Policy
        response.headers["Permissions-Policy"] = (
            "geolocation=(), microphone=(), camera=()"
        )

        return response


app = FastAPI()
app.add_middleware(SecurityHeadersMiddleware)
```

---

## CORS (Cross-Origin Resource Sharing)

### Концепция

CORS — механизм, позволяющий браузерам делать запросы к другим доменам. По умолчанию браузеры блокируют cross-origin запросы из соображений безопасности.

### Как работает CORS

```
┌─────────────────────────────────────────────────────────────────┐
│                         CORS Flow                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Simple Request (GET, POST с простыми заголовками)              │
│  ──────────────────────────────────────────────────              │
│                                                                  │
│  Browser ────────────────────► Server                           │
│          GET /api/data                                           │
│          Origin: https://frontend.com                           │
│                                                                  │
│  Browser ◄─────────────────── Server                            │
│          Access-Control-Allow-Origin: https://frontend.com      │
│          Data...                                                 │
│                                                                  │
│                                                                  │
│  Preflight Request (PUT, DELETE, custom headers)                │
│  ───────────────────────────────────────────────                │
│                                                                  │
│  Browser ────────────────────► Server                           │
│          OPTIONS /api/data                                       │
│          Origin: https://frontend.com                           │
│          Access-Control-Request-Method: PUT                     │
│          Access-Control-Request-Headers: Authorization          │
│                                                                  │
│  Browser ◄─────────────────── Server                            │
│          Access-Control-Allow-Origin: https://frontend.com      │
│          Access-Control-Allow-Methods: PUT, POST, DELETE        │
│          Access-Control-Allow-Headers: Authorization            │
│          Access-Control-Max-Age: 86400                          │
│                                                                  │
│  Browser ────────────────────► Server                           │
│          PUT /api/data                                           │
│          Origin: https://frontend.com                           │
│          Authorization: Bearer ...                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Настройка CORS

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# Разрешённые origins
ALLOWED_ORIGINS = [
    "https://myapp.com",
    "https://admin.myapp.com",
]

# Для разработки
if DEBUG:
    ALLOWED_ORIGINS.extend([
        "http://localhost:3000",
        "http://localhost:8080",
    ])

app.add_middleware(
    CORSMiddleware,
    allow_origins=ALLOWED_ORIGINS,      # Список разрешённых origins
    allow_credentials=True,              # Разрешить cookies
    allow_methods=["GET", "POST", "PUT", "DELETE", "PATCH"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
    expose_headers=["X-Request-ID"],     # Заголовки, доступные клиенту
    max_age=86400,                       # Кэш preflight на 24 часа
)


# ВАЖНО: Никогда не используйте в продакшене!
# allow_origins=["*"]  # Разрешает все домены — уязвимо!
# allow_credentials=True + allow_origins=["*"]  # Ещё хуже!


# Динамическая проверка origin
from starlette.middleware.base import BaseHTTPMiddleware


class DynamicCORSMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, allowed_origin_patterns: list):
        super().__init__(app)
        self.patterns = [re.compile(p) for p in allowed_origin_patterns]

    async def dispatch(self, request, call_next):
        origin = request.headers.get("origin")

        # Проверяем preflight
        if request.method == "OPTIONS":
            response = Response()
            response.status_code = 200
        else:
            response = await call_next(request)

        # Проверяем origin
        if origin and self._is_allowed(origin):
            response.headers["Access-Control-Allow-Origin"] = origin
            response.headers["Access-Control-Allow-Credentials"] = "true"
            response.headers["Access-Control-Allow-Methods"] = "GET, POST, PUT, DELETE"
            response.headers["Access-Control-Allow-Headers"] = "Authorization, Content-Type"

        return response

    def _is_allowed(self, origin: str) -> bool:
        return any(p.match(origin) for p in self.patterns)


# Разрешаем все поддомены myapp.com
app.add_middleware(
    DynamicCORSMiddleware,
    allowed_origin_patterns=[
        r"https://.*\.myapp\.com",
        r"https://myapp\.com"
    ]
)
```

---

## OWASP Top 10

### Обзор уязвимостей

```
┌─────────────────────────────────────────────────────────────────┐
│                    OWASP Top 10 (2021)                          │
├────┬────────────────────────────────────────────────────────────┤
│ A01│ Broken Access Control                                      │
│    │ Обход контроля доступа, IDOR                               │
├────┼────────────────────────────────────────────────────────────┤
│ A02│ Cryptographic Failures                                     │
│    │ Слабое шифрование, хранение паролей                        │
├────┼────────────────────────────────────────────────────────────┤
│ A03│ Injection                                                   │
│    │ SQL, NoSQL, OS Command, LDAP injection                     │
├────┼────────────────────────────────────────────────────────────┤
│ A04│ Insecure Design                                            │
│    │ Ошибки проектирования, отсутствие threat modeling         │
├────┼────────────────────────────────────────────────────────────┤
│ A05│ Security Misconfiguration                                  │
│    │ Дефолтные пароли, открытые порты, verbose errors          │
├────┼────────────────────────────────────────────────────────────┤
│ A06│ Vulnerable Components                                      │
│    │ Устаревшие библиотеки с известными уязвимостями           │
├────┼────────────────────────────────────────────────────────────┤
│ A07│ Authentication Failures                                    │
│    │ Слабые пароли, brute force, session fixation              │
├────┼────────────────────────────────────────────────────────────┤
│ A08│ Software and Data Integrity Failures                       │
│    │ Непроверенные обновления, insecure deserialization        │
├────┼────────────────────────────────────────────────────────────┤
│ A09│ Security Logging and Monitoring Failures                   │
│    │ Отсутствие логирования, alerting                          │
├────┼────────────────────────────────────────────────────────────┤
│ A10│ Server-Side Request Forgery (SSRF)                         │
│    │ Сервер делает запросы к внутренним ресурсам              │
└────┴────────────────────────────────────────────────────────────┘
```

---

## Защита от инъекций

### SQL Injection

```python
# УЯЗВИМО!
def get_user_vulnerable(user_id: str):
    query = f"SELECT * FROM users WHERE id = {user_id}"
    # Ввод: "1; DROP TABLE users;--"
    # Результат: SELECT * FROM users WHERE id = 1; DROP TABLE users;--
    return db.execute(query)


# БЕЗОПАСНО: Параметризованные запросы
def get_user_safe(user_id: int):
    query = "SELECT * FROM users WHERE id = %s"
    return db.execute(query, (user_id,))


# SQLAlchemy — ORM автоматически экранирует
from sqlalchemy import select
from models import User

def get_user_orm(user_id: int, session):
    return session.execute(
        select(User).where(User.id == user_id)
    ).scalar_one_or_none()


# Pydantic для валидации входных данных
from pydantic import BaseModel, validator

class UserQuery(BaseModel):
    user_id: int  # Автоматически проверит, что это число

    @validator('user_id')
    def validate_user_id(cls, v):
        if v < 1:
            raise ValueError('Invalid user_id')
        return v
```

### NoSQL Injection

```python
# MongoDB — УЯЗВИМО!
def find_user_vulnerable(username: str):
    # Ввод: {"$ne": ""}
    # Найдёт всех пользователей!
    return db.users.find_one({"username": username})


# БЕЗОПАСНО: проверка типа
def find_user_safe(username: str):
    if not isinstance(username, str):
        raise ValueError("Username must be a string")

    # Явно приводим к строке
    return db.users.find_one({"username": str(username)})


# Использование схем (Pydantic + beanie/odmantic)
from beanie import Document
from pydantic import Field

class User(Document):
    username: str = Field(..., min_length=3, max_length=50, regex="^[a-zA-Z0-9_]+$")

    class Settings:
        name = "users"


async def find_user(username: str) -> User:
    # Pydantic валидирует username автоматически
    return await User.find_one(User.username == username)
```

### Command Injection

```python
import subprocess
import shlex

# УЯЗВИМО!
def run_command_vulnerable(filename: str):
    # Ввод: "file.txt; rm -rf /"
    os.system(f"cat {filename}")


# БЕЗОПАСНО: используйте subprocess с shell=False
def run_command_safe(filename: str):
    # Валидация имени файла
    if not filename.replace(".", "").replace("_", "").replace("-", "").isalnum():
        raise ValueError("Invalid filename")

    # shell=False предотвращает интерпретацию shell-метасимволов
    result = subprocess.run(
        ["cat", filename],
        capture_output=True,
        text=True,
        shell=False
    )
    return result.stdout


# Ещё безопаснее: не используйте shell вообще
def read_file_safe(filename: str) -> str:
    # Проверяем, что файл в разрешённой директории
    base_dir = Path("/app/uploads").resolve()
    file_path = (base_dir / filename).resolve()

    if not str(file_path).startswith(str(base_dir)):
        raise ValueError("Path traversal detected")

    with open(file_path) as f:
        return f.read()
```

### XSS (Cross-Site Scripting)

```python
from markupsafe import escape
from fastapi import FastAPI
from fastapi.responses import HTMLResponse

app = FastAPI()


# УЯЗВИМО!
@app.get("/greet-vulnerable", response_class=HTMLResponse)
def greet_vulnerable(name: str):
    # Ввод: <script>alert('XSS')</script>
    return f"<h1>Hello, {name}!</h1>"


# БЕЗОПАСНО: экранирование
@app.get("/greet-safe", response_class=HTMLResponse)
def greet_safe(name: str):
    # escape() преобразует < > & " ' в HTML entities
    safe_name = escape(name)
    return f"<h1>Hello, {safe_name}!</h1>"


# Лучше: использовать шаблонизатор
from jinja2 import Environment, select_autoescape

env = Environment(
    autoescape=select_autoescape(['html', 'xml'])  # Автоэкранирование
)


# API: всегда возвращайте JSON
@app.get("/api/greet")
def greet_api(name: str):
    # JSON автоматически экранируется
    return {"message": f"Hello, {name}!"}


# Content-Type заголовок
@app.get("/data")
def get_data():
    response = JSONResponse(content={"data": "value"})
    response.headers["Content-Type"] = "application/json"
    response.headers["X-Content-Type-Options"] = "nosniff"
    return response
```

### CSRF (Cross-Site Request Forgery)

```python
"""
CSRF защита для form-based приложений.
Для API с JWT токенами CSRF обычно не нужен.
"""

import secrets
from fastapi import FastAPI, Request, Form, HTTPException
from fastapi.responses import HTMLResponse
from starlette.middleware.sessions import SessionMiddleware

app = FastAPI()
app.add_middleware(SessionMiddleware, secret_key="secret-key")


def generate_csrf_token(request: Request) -> str:
    """Генерация CSRF токена."""
    if "csrf_token" not in request.session:
        request.session["csrf_token"] = secrets.token_urlsafe(32)
    return request.session["csrf_token"]


def validate_csrf_token(request: Request, token: str) -> bool:
    """Валидация CSRF токена."""
    session_token = request.session.get("csrf_token")
    if not session_token or not secrets.compare_digest(session_token, token):
        return False
    return True


@app.get("/form", response_class=HTMLResponse)
async def get_form(request: Request):
    token = generate_csrf_token(request)
    return f"""
    <form method="POST" action="/submit">
        <input type="hidden" name="csrf_token" value="{token}">
        <input type="text" name="data">
        <button type="submit">Submit</button>
    </form>
    """


@app.post("/submit")
async def submit_form(
    request: Request,
    csrf_token: str = Form(...),
    data: str = Form(...)
):
    if not validate_csrf_token(request, csrf_token):
        raise HTTPException(403, "Invalid CSRF token")

    return {"status": "success", "data": data}


# Для SPA с API: используйте SameSite cookies
"""
Set-Cookie: session=abc123; SameSite=Strict; Secure; HttpOnly
"""


# Double Submit Cookie Pattern
from fastapi import Cookie, Header


@app.post("/api/action")
async def api_action(
    csrf_cookie: str = Cookie(None, alias="csrf_token"),
    csrf_header: str = Header(None, alias="X-CSRF-Token")
):
    """
    Клиент должен отправить одинаковый токен в cookie и header.
    Атакующий не может прочитать cookie другого домена.
    """
    if not csrf_cookie or not csrf_header:
        raise HTTPException(403, "CSRF tokens required")

    if not secrets.compare_digest(csrf_cookie, csrf_header):
        raise HTTPException(403, "CSRF tokens mismatch")

    return {"status": "success"}
```

---

## Дополнительные практики безопасности

### Безопасное логирование

```python
import logging
from typing import Any
import re

logger = logging.getLogger(__name__)


class SensitiveDataFilter(logging.Filter):
    """Фильтр для маскирования чувствительных данных в логах."""

    PATTERNS = [
        (r'"password"\s*:\s*"[^"]*"', '"password": "[REDACTED]"'),
        (r'"token"\s*:\s*"[^"]*"', '"token": "[REDACTED]"'),
        (r'"credit_card"\s*:\s*"\d{16}"', '"credit_card": "[REDACTED]"'),
        (r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b', '[EMAIL]'),
    ]

    def filter(self, record: logging.LogRecord) -> bool:
        if isinstance(record.msg, str):
            for pattern, replacement in self.PATTERNS:
                record.msg = re.sub(pattern, replacement, record.msg)
        return True


# Применяем фильтр
handler = logging.StreamHandler()
handler.addFilter(SensitiveDataFilter())
logger.addHandler(handler)


# Безопасное логирование
def log_request(user_id: int, action: str, details: dict):
    """Логирование без чувствительных данных."""
    safe_details = {
        k: v for k, v in details.items()
        if k not in {'password', 'token', 'credit_card', 'ssn'}
    }

    logger.info(
        "User action",
        extra={
            "user_id": user_id,
            "action": action,
            "details": safe_details
        }
    )
```

### Rate Limiting для защиты от brute force

```python
from datetime import datetime, timedelta
from typing import Optional
import redis

redis_client = redis.Redis()


class LoginRateLimiter:
    """Защита от brute force атак на login."""

    def __init__(
        self,
        max_attempts: int = 5,
        lockout_duration: int = 900,  # 15 минут
        window_size: int = 300         # 5 минут
    ):
        self.max_attempts = max_attempts
        self.lockout_duration = lockout_duration
        self.window_size = window_size

    def _get_key(self, identifier: str) -> str:
        return f"login_attempts:{identifier}"

    def _get_lockout_key(self, identifier: str) -> str:
        return f"login_lockout:{identifier}"

    async def is_locked(self, identifier: str) -> bool:
        """Проверка блокировки."""
        return bool(await redis_client.get(self._get_lockout_key(identifier)))

    async def record_attempt(self, identifier: str, success: bool):
        """Запись попытки входа."""
        if success:
            # Успешный вход — сбрасываем счётчик
            await redis_client.delete(self._get_key(identifier))
            return

        # Неудачная попытка
        key = self._get_key(identifier)
        pipe = redis_client.pipeline()
        pipe.incr(key)
        pipe.expire(key, self.window_size)
        results = await pipe.execute()

        attempts = results[0]

        if attempts >= self.max_attempts:
            # Блокируем аккаунт
            await redis_client.setex(
                self._get_lockout_key(identifier),
                self.lockout_duration,
                "locked"
            )
            # Отправляем уведомление пользователю
            await notify_user_of_lockout(identifier)

    async def get_remaining_attempts(self, identifier: str) -> int:
        """Оставшееся количество попыток."""
        attempts = await redis_client.get(self._get_key(identifier))
        current = int(attempts) if attempts else 0
        return max(0, self.max_attempts - current)


rate_limiter = LoginRateLimiter()


async def login(email: str, password: str) -> dict:
    # Проверяем блокировку
    if await rate_limiter.is_locked(email):
        raise HTTPException(
            429,
            "Account temporarily locked. Try again later."
        )

    user = await authenticate_user(email, password)

    if not user:
        await rate_limiter.record_attempt(email, success=False)
        remaining = await rate_limiter.get_remaining_attempts(email)
        raise HTTPException(
            401,
            f"Invalid credentials. {remaining} attempts remaining."
        )

    await rate_limiter.record_attempt(email, success=True)
    return create_tokens(user)
```

### Secure Configuration

```python
from pydantic import BaseSettings, SecretStr, validator
from typing import List
import secrets


class SecuritySettings(BaseSettings):
    # Секретные ключи
    secret_key: SecretStr
    jwt_secret: SecretStr
    database_password: SecretStr

    # CORS
    allowed_origins: List[str] = []

    # Cookie settings
    cookie_secure: bool = True
    cookie_httponly: bool = True
    cookie_samesite: str = "Lax"

    # Password policy
    min_password_length: int = 12
    require_special_chars: bool = True

    # Rate limiting
    rate_limit_requests: int = 100
    rate_limit_window: int = 60

    # Session
    session_timeout_minutes: int = 30

    @validator('secret_key', 'jwt_secret')
    def validate_secret_strength(cls, v):
        if len(v.get_secret_value()) < 32:
            raise ValueError('Secret key must be at least 32 characters')
        return v

    @validator('allowed_origins')
    def validate_origins(cls, v):
        for origin in v:
            if origin == "*":
                raise ValueError('Wildcard origin not allowed in production')
        return v

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"


settings = SecuritySettings()


# Генерация безопасных секретов
def generate_secure_secret(length: int = 32) -> str:
    return secrets.token_urlsafe(length)


# Использование: python -c "from config import generate_secure_secret; print(generate_secure_secret())"
```

---

## Чеклист безопасности

### Перед релизом

```markdown
## Аутентификация
- [ ] Пароли хешируются с Argon2/bcrypt
- [ ] Реализована защита от brute force
- [ ] MFA доступна для критичных аккаунтов
- [ ] Session timeout настроен

## Авторизация
- [ ] Проверка прав на каждом endpoint
- [ ] RBAC/ABAC реализован правильно
- [ ] IDOR уязвимости проверены

## Данные
- [ ] HTTPS везде (включая внутренние сервисы)
- [ ] Чувствительные данные зашифрованы at rest
- [ ] PII данные обрабатываются согласно GDPR

## Инъекции
- [ ] SQL injection — параметризованные запросы
- [ ] XSS — автоэкранирование
- [ ] Command injection — нет shell=True
- [ ] CSRF protection для форм

## Конфигурация
- [ ] Debug mode выключен
- [ ] Секреты в env переменных, не в коде
- [ ] Security headers настроены
- [ ] CORS правильно ограничен

## Мониторинг
- [ ] Логирование аутентификации
- [ ] Алерты на подозрительную активность
- [ ] Чувствительные данные не логируются

## Зависимости
- [ ] Все пакеты обновлены
- [ ] Уязвимости проверены (pip-audit, safety)
- [ ] Lockfile используется
```

---

## Полезные инструменты

### Анализ уязвимостей

```bash
# Python packages
pip install safety pip-audit bandit

# Проверка уязвимостей в зависимостях
safety check
pip-audit

# Статический анализ кода
bandit -r src/

# OWASP Dependency Check
# https://owasp.org/www-project-dependency-check/
```

### Тестирование безопасности

```python
# pytest-security
import pytest


def test_password_hashing():
    """Пароль не должен храниться в открытом виде."""
    user = create_user("test@test.com", "password123")
    assert user.password_hash != "password123"
    assert verify_password("password123", user.password_hash)


def test_sql_injection():
    """SQL injection должен быть предотвращён."""
    malicious_input = "'; DROP TABLE users;--"
    # Не должно вызывать ошибку или дропать таблицу
    result = get_user_by_name(malicious_input)
    assert result is None


def test_rate_limiting():
    """Rate limiting должен работать."""
    client = TestClient(app)

    for _ in range(100):
        client.get("/api/resource")

    response = client.get("/api/resource")
    assert response.status_code == 429


def test_authentication_required():
    """Защищённые endpoints требуют аутентификации."""
    client = TestClient(app)

    response = client.get("/api/protected")
    assert response.status_code == 401

    response = client.get("/api/protected", headers={"Authorization": "Bearer invalid"})
    assert response.status_code == 401
```

---

## Заключение

Безопасность — это не feature, а непрерывный процесс:

1. **Security by Design** — думайте о безопасности на этапе проектирования
2. **Defense in Depth** — несколько уровней защиты
3. **Least Privilege** — минимально необходимые права
4. **Fail Secure** — при ошибке система должна быть в безопасном состоянии
5. **Keep Updated** — регулярно обновляйте зависимости

Ресурсы для изучения:
- OWASP Cheat Sheet Series
- OWASP Testing Guide
- CWE/SANS Top 25
- NIST Cybersecurity Framework
