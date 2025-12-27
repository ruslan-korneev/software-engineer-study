# Аутентификация в API (Authentication)

## Что такое аутентификация?

**Аутентификация** — это процесс подтверждения личности пользователя или системы. Отвечает на вопрос: "Кто вы?"

В отличие от **авторизации**, которая отвечает на вопрос "Что вам разрешено делать?", аутентификация только подтверждает идентичность.

---

## Методы аутентификации

### 1. Basic Authentication

Простейший метод — передача логина и пароля в заголовке.

```
Authorization: Basic base64(username:password)
```

**Пример:**
```python
import base64

# Кодирование
credentials = base64.b64encode(b"user:password").decode()
# Результат: "dXNlcjpwYXNzd29yZA=="

# Заголовок
headers = {"Authorization": f"Basic {credentials}"}
```

**Проблемы:**
- Пароль передаётся с каждым запросом
- Base64 — это НЕ шифрование, легко декодируется
- Обязательно использовать HTTPS

**Когда использовать:**
- Внутренние сервисы с ограниченным доступом
- Простые API без сложных требований

---

### 2. API Keys

Уникальный ключ, идентифицирующий клиента.

```
X-API-Key: sk_live_abc123xyz
```

**Реализация FastAPI:**

```python
from fastapi import FastAPI, Security, HTTPException
from fastapi.security import APIKeyHeader

app = FastAPI()
api_key_header = APIKeyHeader(name="X-API-Key")

# Хранение ключей (в production — база данных)
API_KEYS = {
    "sk_live_abc123": {"user_id": "user_1", "permissions": ["read", "write"]},
    "sk_live_xyz789": {"user_id": "user_2", "permissions": ["read"]},
}

async def verify_api_key(api_key: str = Security(api_key_header)):
    if api_key not in API_KEYS:
        raise HTTPException(status_code=401, detail="Invalid API key")
    return API_KEYS[api_key]

@app.get("/api/data")
async def get_data(api_info: dict = Security(verify_api_key)):
    return {"user_id": api_info["user_id"], "data": "secret"}
```

**Best Practices для API Keys:**
- Используйте префиксы для типов ключей: `sk_live_`, `sk_test_`, `pk_`
- Храните хеши ключей, не сами ключи
- Реализуйте ротацию ключей
- Логируйте использование ключей

---

### 3. JWT (JSON Web Tokens)

Самодостаточный токен, содержащий информацию о пользователе.

**Структура JWT:**
```
header.payload.signature
```

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4ifQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**Реализация FastAPI:**

```python
from datetime import datetime, timedelta
from jose import JWTError, jwt
from fastapi import FastAPI, Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from passlib.context import CryptContext
from pydantic import BaseModel

# Конфигурация
SECRET_KEY = "your-super-secret-key-keep-it-safe"  # В production — из env
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

class Token(BaseModel):
    access_token: str
    token_type: str

class TokenData(BaseModel):
    username: str | None = None

class User(BaseModel):
    username: str
    email: str
    disabled: bool = False

# Функции для работы с паролями
def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)

# Создание токена
def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

# Проверка токена
async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    credentials_exception = HTTPException(
        status_code=401,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    user = get_user_from_db(username)  # Ваша функция
    if user is None:
        raise credentials_exception
    return user

# Эндпоинт получения токена
@app.post("/token", response_model=Token)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user = authenticate_user(form_data.username, form_data.password)
    if not user:
        raise HTTPException(status_code=401, detail="Incorrect username or password")

    access_token = create_access_token(
        data={"sub": user.username},
        expires_delta=timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    )
    return {"access_token": access_token, "token_type": "bearer"}

# Защищённый эндпоинт
@app.get("/users/me")
async def read_users_me(current_user: User = Depends(get_current_user)):
    return current_user
```

**Важно о JWT:**
- Токены НЕ шифрованы — payload можно прочитать
- Подпись защищает от подделки, но не от чтения
- Не храните в JWT чувствительные данные (пароли, номера карт)
- Используйте короткое время жизни (15-30 минут)

---

### 4. OAuth 2.0

Протокол делегирования авторизации. Позволяет приложениям получать ограниченный доступ к ресурсам пользователя.

**Основные роли:**
- **Resource Owner** — пользователь
- **Client** — приложение, запрашивающее доступ
- **Authorization Server** — выдаёт токены
- **Resource Server** — API с защищёнными ресурсами

**Типы грантов (flows):**

| Grant Type | Когда использовать |
|------------|-------------------|
| Authorization Code | Веб-приложения с серверной частью |
| Authorization Code + PKCE | SPA и мобильные приложения |
| Client Credentials | Межсерверное взаимодействие (M2M) |
| Refresh Token | Продление сессии без повторной аутентификации |

**Authorization Code Flow:**

```
1. User → Client: Хочу войти через Google
2. Client → Auth Server: Redirect на страницу входа Google
3. User → Auth Server: Логин/пароль
4. Auth Server → Client: Authorization code
5. Client → Auth Server: Обмен code на access_token
6. Auth Server → Client: access_token + refresh_token
7. Client → Resource Server: Запрос с access_token
```

**Пример с PKCE (Python):**

```python
import hashlib
import base64
import secrets

# Генерация code_verifier и code_challenge
def generate_pkce_pair():
    code_verifier = secrets.token_urlsafe(32)
    code_challenge = base64.urlsafe_b64encode(
        hashlib.sha256(code_verifier.encode()).digest()
    ).decode().rstrip("=")
    return code_verifier, code_challenge

verifier, challenge = generate_pkce_pair()

# 1. Redirect пользователя на авторизацию
auth_url = (
    f"https://auth.example.com/authorize?"
    f"response_type=code&"
    f"client_id=your_client_id&"
    f"redirect_uri=https://yourapp.com/callback&"
    f"scope=read write&"
    f"code_challenge={challenge}&"
    f"code_challenge_method=S256"
)

# 2. После callback обмениваем code на токен
import httpx

async def exchange_code_for_token(code: str, verifier: str):
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "https://auth.example.com/token",
            data={
                "grant_type": "authorization_code",
                "code": code,
                "redirect_uri": "https://yourapp.com/callback",
                "client_id": "your_client_id",
                "code_verifier": verifier,
            }
        )
        return response.json()
```

---

### 5. Session-based Authentication

Классический подход — сервер хранит состояние сессии.

```python
from fastapi import FastAPI, Request, Response
from fastapi.responses import RedirectResponse
import uuid

app = FastAPI()

# Хранилище сессий (в production — Redis)
sessions: dict[str, dict] = {}

@app.post("/login")
async def login(response: Response, username: str, password: str):
    if not verify_credentials(username, password):
        raise HTTPException(status_code=401)

    session_id = str(uuid.uuid4())
    sessions[session_id] = {
        "user_id": username,
        "created_at": datetime.utcnow(),
    }

    response.set_cookie(
        key="session_id",
        value=session_id,
        httponly=True,      # Недоступно для JavaScript
        secure=True,        # Только HTTPS
        samesite="strict",  # Защита от CSRF
        max_age=3600,       # 1 час
    )
    return {"message": "Logged in"}

async def get_session(request: Request):
    session_id = request.cookies.get("session_id")
    if not session_id or session_id not in sessions:
        raise HTTPException(status_code=401)
    return sessions[session_id]
```

**JWT vs Sessions:**

| Критерий | JWT | Sessions |
|----------|-----|----------|
| Хранение состояния | Клиент (stateless) | Сервер (stateful) |
| Масштабирование | Проще | Требует shared storage |
| Отзыв токена | Сложно | Легко |
| Размер | Больше (~1KB) | Меньше (session ID) |
| Использование | Микросервисы, SPA | Монолиты, SSR |

---

## Refresh Tokens

Механизм продления сессии без повторного ввода пароля.

```python
from datetime import datetime, timedelta

ACCESS_TOKEN_EXPIRE = timedelta(minutes=15)
REFRESH_TOKEN_EXPIRE = timedelta(days=7)

def create_tokens(user_id: str) -> dict:
    access_token = create_access_token(
        data={"sub": user_id, "type": "access"},
        expires_delta=ACCESS_TOKEN_EXPIRE
    )
    refresh_token = create_access_token(
        data={"sub": user_id, "type": "refresh"},
        expires_delta=REFRESH_TOKEN_EXPIRE
    )

    # Сохраняем refresh token в БД для возможности отзыва
    save_refresh_token_to_db(user_id, refresh_token)

    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "token_type": "bearer"
    }

@app.post("/refresh")
async def refresh_tokens(refresh_token: str):
    try:
        payload = jwt.decode(refresh_token, SECRET_KEY, algorithms=[ALGORITHM])
        if payload.get("type") != "refresh":
            raise HTTPException(status_code=401, detail="Invalid token type")

        user_id = payload.get("sub")

        # Проверяем, не отозван ли токен
        if not is_refresh_token_valid(user_id, refresh_token):
            raise HTTPException(status_code=401, detail="Token revoked")

        # Ротация: выдаём новую пару токенов
        return create_tokens(user_id)

    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")
```

**Best Practices для Refresh Tokens:**
- Храните refresh tokens в httpOnly cookies
- Реализуйте ротацию токенов (каждый refresh выдаёт новую пару)
- Храните refresh tokens в БД для возможности отзыва
- Ограничивайте количество активных refresh tokens на пользователя

---

## Угрозы и защита

### 1. Brute Force

**Атака:** Перебор паролей.

**Защита:**
```python
from fastapi import HTTPException
from collections import defaultdict
import time

# Счётчик неудачных попыток
failed_attempts: dict[str, list[float]] = defaultdict(list)
MAX_ATTEMPTS = 5
LOCKOUT_TIME = 300  # 5 минут

async def check_brute_force(ip: str, username: str):
    key = f"{ip}:{username}"
    now = time.time()

    # Удаляем старые попытки
    failed_attempts[key] = [
        t for t in failed_attempts[key]
        if now - t < LOCKOUT_TIME
    ]

    if len(failed_attempts[key]) >= MAX_ATTEMPTS:
        raise HTTPException(
            status_code=429,
            detail=f"Too many attempts. Try again in {LOCKOUT_TIME} seconds"
        )

async def record_failed_attempt(ip: str, username: str):
    key = f"{ip}:{username}"
    failed_attempts[key].append(time.time())
```

### 2. Credential Stuffing

**Атака:** Использование утёкших паролей с других сайтов.

**Защита:**
- Проверка паролей через Have I Been Pwned API
- Требование уникальных паролей
- 2FA

```python
import hashlib
import httpx

async def check_password_breach(password: str) -> bool:
    """Проверяет, не был ли пароль скомпрометирован"""
    sha1_hash = hashlib.sha1(password.encode()).hexdigest().upper()
    prefix = sha1_hash[:5]
    suffix = sha1_hash[5:]

    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"https://api.pwnedpasswords.com/range/{prefix}"
        )

    for line in response.text.splitlines():
        hash_suffix, count = line.split(":")
        if hash_suffix == suffix:
            return True  # Пароль скомпрометирован

    return False
```

### 3. Token Theft

**Атака:** Кража токенов через XSS или перехват.

**Защита:**
- HttpOnly cookies для refresh tokens
- Короткий срок жизни access tokens
- Token binding (привязка к fingerprint клиента)

---

## Двухфакторная аутентификация (2FA)

### TOTP (Time-based One-Time Password)

```python
import pyotp
import qrcode
from io import BytesIO

def setup_2fa(user_email: str) -> tuple[str, bytes]:
    """Настройка 2FA для пользователя"""
    secret = pyotp.random_base32()

    # Генерация QR-кода для Google Authenticator
    totp = pyotp.TOTP(secret)
    provisioning_uri = totp.provisioning_uri(
        user_email,
        issuer_name="Your App"
    )

    qr = qrcode.make(provisioning_uri)
    buffer = BytesIO()
    qr.save(buffer, format="PNG")

    return secret, buffer.getvalue()

def verify_2fa(secret: str, code: str) -> bool:
    """Проверка кода 2FA"""
    totp = pyotp.TOTP(secret)
    return totp.verify(code, valid_window=1)  # ±30 секунд
```

---

## Best Practices

1. **Всегда используйте HTTPS** — без исключений
2. **Хешируйте пароли** — bcrypt, argon2, scrypt
3. **Не изобретайте криптографию** — используйте проверенные библиотеки
4. **Логируйте попытки аутентификации** — для обнаружения атак
5. **Реализуйте rate limiting** — защита от brute force
6. **Используйте 2FA** — особенно для админов
7. **Безопасный logout** — инвалидация всех токенов
8. **Регулярная ротация** — API keys, secrets

---

## Чек-лист безопасной аутентификации

- [ ] Пароли хешируются с использованием bcrypt/argon2
- [ ] JWT подписываются надёжным алгоритмом (RS256 или HS256 с длинным ключом)
- [ ] Refresh tokens хранятся безопасно и могут быть отозваны
- [ ] Реализована защита от brute force
- [ ] Настроен rate limiting на эндпоинтах аутентификации
- [ ] Используется HTTPS везде
- [ ] API keys имеют ограниченные права и срок действия
- [ ] Логируются все попытки входа
- [ ] Доступна 2FA для пользователей
