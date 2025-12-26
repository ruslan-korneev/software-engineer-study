# Cookies (Куки)

## Что такое Cookies

**Cookies (куки)** — это небольшие фрагменты данных, которые сервер отправляет браузеру, а браузер сохраняет и отправляет обратно с каждым последующим запросом. Куки позволяют "запоминать" информацию между HTTP-запросами, несмотря на stateless-природу HTTP.

## Как работают Cookies

```
1. Клиент отправляет запрос
   GET /login HTTP/1.1

2. Сервер отвечает с Set-Cookie
   HTTP/1.1 200 OK
   Set-Cookie: session_id=abc123; HttpOnly; Secure

3. Браузер сохраняет cookie

4. Браузер отправляет cookie с каждым последующим запросом
   GET /profile HTTP/1.1
   Cookie: session_id=abc123
```

## Создание Cookies (Set-Cookie)

### Синтаксис заголовка Set-Cookie

```http
Set-Cookie: name=value; Domain=example.com; Path=/; Expires=Wed, 15 Jan 2025 10:00:00 GMT; Max-Age=3600; HttpOnly; Secure; SameSite=Strict
```

### Атрибуты Cookie

| Атрибут | Описание |
|---------|----------|
| `name=value` | Имя и значение cookie (обязательно) |
| `Domain` | Домен, для которого cookie действительна |
| `Path` | Путь, для которого cookie действительна |
| `Expires` | Дата истечения срока действия |
| `Max-Age` | Время жизни в секундах |
| `HttpOnly` | Недоступна для JavaScript |
| `Secure` | Отправляется только по HTTPS |
| `SameSite` | Контроль отправки с cross-site запросами |

### Примеры в FastAPI

```python
from fastapi import FastAPI, Response, Cookie
from fastapi.responses import JSONResponse
from datetime import datetime, timedelta

app = FastAPI()

# Установка простой cookie
@app.post("/login")
def login(response: Response):
    response.set_cookie(
        key="session_id",
        value="abc123xyz"
    )
    return {"message": "Logged in"}

# Установка cookie с атрибутами безопасности
@app.post("/secure-login")
def secure_login(response: Response):
    response.set_cookie(
        key="session_id",
        value="abc123xyz",
        httponly=True,      # Недоступна для JS
        secure=True,        # Только HTTPS
        samesite="strict",  # Не отправляется с cross-site
        max_age=3600,       # 1 час
        path="/",           # Для всего сайта
        domain="example.com"
    )
    return {"message": "Logged in securely"}

# Установка cookie с датой истечения
@app.post("/remember-me")
def remember_me(response: Response):
    expires = datetime.utcnow() + timedelta(days=30)
    response.set_cookie(
        key="remember_token",
        value="long_lived_token",
        expires=expires.strftime("%a, %d %b %Y %H:%M:%S GMT"),
        httponly=True,
        secure=True
    )
    return {"message": "Remember me set"}

# Удаление cookie
@app.post("/logout")
def logout(response: Response):
    response.delete_cookie(key="session_id")
    # Или установка с прошедшей датой
    response.set_cookie(
        key="session_id",
        value="",
        max_age=0
    )
    return {"message": "Logged out"}
```

## Чтение Cookies

### На сервере (FastAPI)

```python
from fastapi import FastAPI, Cookie, Request
from typing import Optional

app = FastAPI()

# Через параметр Cookie
@app.get("/profile")
def get_profile(session_id: Optional[str] = Cookie(None)):
    if not session_id:
        return {"error": "Not authenticated"}
    return {"session_id": session_id}

# Через Request объект
@app.get("/all-cookies")
def get_all_cookies(request: Request):
    cookies = request.cookies
    return {"cookies": dict(cookies)}

# С дефолтным значением и валидацией
@app.get("/preferences")
def get_preferences(
    theme: str = Cookie("light"),
    language: str = Cookie("en")
):
    return {"theme": theme, "language": language}
```

### На клиенте (Python requests)

```python
import requests

# Отправка cookies
response = requests.get(
    "https://api.example.com/profile",
    cookies={"session_id": "abc123"}
)

# Получение cookies из ответа
response = requests.post("https://api.example.com/login", json={...})
print(response.cookies)  # RequestsCookieJar
print(response.cookies.get("session_id"))

# Использование сессии (автоматическое сохранение cookies)
session = requests.Session()

# Cookies сохраняются автоматически
session.post("https://api.example.com/login", json={...})

# Последующие запросы включают cookies
response = session.get("https://api.example.com/profile")

# Ручное добавление cookies в сессию
session.cookies.set("custom", "value")
```

## Типы Cookies

### 1. Session Cookies (Сессионные)

Удаляются при закрытии браузера. Не имеют `Expires` или `Max-Age`.

```python
response.set_cookie(
    key="session_id",
    value="temp_session",
    httponly=True
)
```

### 2. Persistent Cookies (Постоянные)

Сохраняются до указанной даты или времени.

```python
response.set_cookie(
    key="user_preferences",
    value="theme=dark",
    max_age=365 * 24 * 60 * 60  # 1 год
)
```

### 3. First-party Cookies

Устанавливаются текущим доменом.

```python
# На сайте example.com
Set-Cookie: session=abc; Domain=example.com
```

### 4. Third-party Cookies

Устанавливаются другим доменом (для рекламы, аналитики). Блокируются современными браузерами.

```python
# На сайте example.com, cookie от analytics.com
Set-Cookie: tracker=xyz; Domain=analytics.com
```

## Атрибуты безопасности

### HttpOnly

Защищает от XSS-атак. Cookie недоступна через `document.cookie` в JavaScript.

```python
# Хорошо для session cookies
response.set_cookie(
    key="session_id",
    value="secret",
    httponly=True  # JS не может прочитать
)
```

```javascript
// В браузере
console.log(document.cookie);  // session_id не будет видна
```

### Secure

Cookie отправляется только по HTTPS.

```python
# Обязательно для production
response.set_cookie(
    key="auth_token",
    value="secret",
    secure=True  # Только HTTPS
)
```

### SameSite

Контролирует отправку cookies с cross-site запросами. Защищает от CSRF.

```python
# Strict - не отправляется с любых cross-site запросов
response.set_cookie(key="session", value="abc", samesite="strict")

# Lax - отправляется только с "безопасными" cross-site (GET навигация)
response.set_cookie(key="session", value="abc", samesite="lax")

# None - отправляется всегда (требует Secure=true)
response.set_cookie(key="session", value="abc", samesite="none", secure=True)
```

| SameSite | Безопасный GET (ссылки) | Формы POST | AJAX/fetch |
|----------|-------------------------|------------|------------|
| Strict   | Нет                     | Нет        | Нет        |
| Lax      | Да                      | Нет        | Нет        |
| None     | Да                      | Да         | Да         |

## Domain и Path

### Domain

Определяет, для каких доменов cookie действительна.

```python
# Cookie для example.com и всех поддоменов (api.example.com, www.example.com)
response.set_cookie(key="user", value="john", domain=".example.com")

# Cookie только для api.example.com
response.set_cookie(key="api_token", value="xyz", domain="api.example.com")

# Без указания domain - только для текущего домена
response.set_cookie(key="session", value="abc")
```

### Path

Определяет, для каких путей cookie отправляется.

```python
# Cookie для всего сайта
response.set_cookie(key="global", value="all", path="/")

# Cookie только для /api
response.set_cookie(key="api_session", value="xyz", path="/api")

# Cookie для /admin
response.set_cookie(key="admin_token", value="secret", path="/admin")
```

## Практические сценарии

### 1. Аутентификация с сессиями

```python
from fastapi import FastAPI, Response, Cookie, HTTPException
from typing import Optional
import secrets

app = FastAPI()

# Хранилище сессий (в production - Redis или база данных)
sessions = {}

@app.post("/login")
def login(username: str, password: str, response: Response):
    # Проверка учетных данных
    if username == "admin" and password == "secret":
        session_id = secrets.token_urlsafe(32)
        sessions[session_id] = {"user_id": 1, "username": username}

        response.set_cookie(
            key="session_id",
            value=session_id,
            httponly=True,
            secure=True,
            samesite="lax",
            max_age=3600
        )
        return {"message": "Logged in"}

    raise HTTPException(status_code=401, detail="Invalid credentials")

@app.get("/protected")
def protected(session_id: Optional[str] = Cookie(None)):
    if not session_id or session_id not in sessions:
        raise HTTPException(status_code=401, detail="Not authenticated")

    user = sessions[session_id]
    return {"message": f"Hello, {user['username']}"}

@app.post("/logout")
def logout(response: Response, session_id: Optional[str] = Cookie(None)):
    if session_id and session_id in sessions:
        del sessions[session_id]

    response.delete_cookie(key="session_id")
    return {"message": "Logged out"}
```

### 2. Remember Me функциональность

```python
import secrets
from datetime import datetime, timedelta

remember_tokens = {}

@app.post("/login")
def login(
    username: str,
    password: str,
    remember_me: bool = False,
    response: Response = None
):
    # Проверка учетных данных
    user_id = authenticate(username, password)

    # Session cookie
    session_id = create_session(user_id)
    response.set_cookie(
        key="session_id",
        value=session_id,
        httponly=True,
        secure=True,
        samesite="lax"
    )

    # Remember me cookie (долгоживущая)
    if remember_me:
        remember_token = secrets.token_urlsafe(32)
        remember_tokens[remember_token] = {
            "user_id": user_id,
            "expires": datetime.utcnow() + timedelta(days=30)
        }
        response.set_cookie(
            key="remember_token",
            value=remember_token,
            httponly=True,
            secure=True,
            samesite="lax",
            max_age=30 * 24 * 60 * 60  # 30 дней
        )

    return {"message": "Logged in"}

@app.get("/auto-login")
def auto_login(
    session_id: Optional[str] = Cookie(None),
    remember_token: Optional[str] = Cookie(None),
    response: Response = None
):
    # Проверяем session
    if session_id and is_valid_session(session_id):
        return {"message": "Already logged in"}

    # Проверяем remember token
    if remember_token and remember_token in remember_tokens:
        token_data = remember_tokens[remember_token]
        if token_data["expires"] > datetime.utcnow():
            # Создаем новую сессию
            session_id = create_session(token_data["user_id"])
            response.set_cookie(
                key="session_id",
                value=session_id,
                httponly=True,
                secure=True
            )
            return {"message": "Auto-logged in"}

    raise HTTPException(status_code=401, detail="Please login")
```

### 3. Пользовательские предпочтения

```python
@app.post("/preferences")
def set_preferences(
    theme: str = "light",
    language: str = "en",
    response: Response = None
):
    response.set_cookie(
        key="theme",
        value=theme,
        max_age=365 * 24 * 60 * 60,
        samesite="lax"
    )
    response.set_cookie(
        key="language",
        value=language,
        max_age=365 * 24 * 60 * 60,
        samesite="lax"
    )
    return {"theme": theme, "language": language}

@app.get("/")
def home(
    theme: str = Cookie("light"),
    language: str = Cookie("en")
):
    return {
        "message": "Welcome",
        "theme": theme,
        "language": language
    }
```

## Cookies в API

### Cookies vs Authorization Header

| Аспект | Cookies | Authorization Header |
|--------|---------|---------------------|
| Автоматическая отправка | Да | Нет |
| CSRF защита | Требуется | Встроенная |
| Работа с JS | Ограничена (HttpOnly) | Полная |
| Cross-origin | Ограничено | Свободно |
| Mobile apps | Сложнее | Проще |

### Рекомендации

```python
# Для веб-приложений с SSR - cookies
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Strict

# Для SPA и мобильных приложений - токены в заголовке
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

# Гибридный подход
@app.get("/resource")
def get_resource(
    session_id: Optional[str] = Cookie(None),
    authorization: Optional[str] = Header(None)
):
    if session_id:
        user = get_user_from_session(session_id)
    elif authorization:
        user = get_user_from_token(authorization)
    else:
        raise HTTPException(status_code=401)

    return {"data": "..."}
```

## Best Practices

### 1. Всегда используйте атрибуты безопасности

```python
# Production cookie
response.set_cookie(
    key="session_id",
    value=session_id,
    httponly=True,      # Защита от XSS
    secure=True,        # Только HTTPS
    samesite="strict",  # Защита от CSRF
    max_age=3600
)
```

### 2. Подписывайте cookies

```python
from itsdangerous import URLSafeTimedSerializer
from fastapi import FastAPI

SECRET_KEY = "your-secret-key"
serializer = URLSafeTimedSerializer(SECRET_KEY)

@app.post("/set-signed")
def set_signed_cookie(user_id: int, response: Response):
    # Подписываем значение
    signed_value = serializer.dumps({"user_id": user_id})
    response.set_cookie(key="user_data", value=signed_value, httponly=True)
    return {"message": "Cookie set"}

@app.get("/get-signed")
def get_signed_cookie(user_data: Optional[str] = Cookie(None)):
    if not user_data:
        raise HTTPException(status_code=401)

    try:
        # Проверяем подпись (max_age в секундах)
        data = serializer.loads(user_data, max_age=3600)
        return {"user_id": data["user_id"]}
    except:
        raise HTTPException(status_code=401, detail="Invalid cookie")
```

### 3. Минимизируйте данные в cookies

```python
# Плохо - много данных
Set-Cookie: user={"id":1,"name":"John","email":"john@example.com","role":"admin"}

# Хорошо - только идентификатор
Set-Cookie: session_id=abc123
# Остальные данные храните на сервере
```

### 4. Устанавливайте правильный срок жизни

```python
# Сессионная cookie (удаляется при закрытии браузера)
response.set_cookie(key="session", value="...")

# Краткосрочная (1 час)
response.set_cookie(key="csrf_token", value="...", max_age=3600)

# Долгосрочная (30 дней)
response.set_cookie(key="remember_me", value="...", max_age=30*24*60*60)
```

## Типичные ошибки

1. **Отсутствие HttpOnly**
   ```python
   # Плохо - XSS может украсть cookie
   response.set_cookie(key="session", value="secret")

   # Хорошо
   response.set_cookie(key="session", value="secret", httponly=True)
   ```

2. **Отсутствие Secure в production**
   ```python
   # Плохо - cookie может быть перехвачена
   response.set_cookie(key="session", value="secret")

   # Хорошо
   response.set_cookie(key="session", value="secret", secure=True)
   ```

3. **Хранение чувствительных данных**
   ```python
   # Плохо
   response.set_cookie(key="password", value="secret123")

   # Хорошо - храните только идентификатор сессии
   response.set_cookie(key="session_id", value="abc123")
   ```

4. **Игнорирование SameSite**
   ```python
   # Уязвимо к CSRF
   response.set_cookie(key="session", value="...")

   # Защищено от CSRF
   response.set_cookie(key="session", value="...", samesite="strict")
   ```
