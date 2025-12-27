# Basic Authentication (Базовая аутентификация)

## Что такое Basic Auth?

**Basic Authentication** — это простейший метод аутентификации HTTP, определённый в RFC 7617. При каждом запросе клиент передаёт учётные данные (логин и пароль) в заголовке `Authorization`, закодированные в Base64.

## Как это работает

### Механизм аутентификации

1. Клиент формирует строку `username:password`
2. Эта строка кодируется в Base64
3. Результат добавляется в заголовок `Authorization` с префиксом `Basic`
4. Сервер декодирует строку и проверяет учётные данные

```
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

### Диаграмма взаимодействия

```
Клиент                                    Сервер
   |                                          |
   |  GET /protected-resource                 |
   |  (без Authorization header)              |
   |----------------------------------------->|
   |                                          |
   |  401 Unauthorized                        |
   |  WWW-Authenticate: Basic realm="API"     |
   |<-----------------------------------------|
   |                                          |
   |  GET /protected-resource                 |
   |  Authorization: Basic base64(user:pass)  |
   |----------------------------------------->|
   |                                          |
   |  200 OK (или 403 если неверные данные)   |
   |<-----------------------------------------|
```

## Практические примеры кода

### Python (Flask)

```python
from flask import Flask, request, jsonify
from functools import wraps
import base64

app = Flask(__name__)

# Простая "база данных" пользователей
USERS = {
    "admin": "secret123",
    "user": "password456"
}

def check_auth(username, password):
    """Проверяет учётные данные пользователя"""
    return USERS.get(username) == password

def requires_auth(f):
    """Декоратор для защиты эндпоинтов"""
    @wraps(f)
    def decorated(*args, **kwargs):
        auth = request.authorization

        if not auth:
            return jsonify({"error": "Authorization required"}), 401, {
                'WWW-Authenticate': 'Basic realm="API"'
            }

        if not check_auth(auth.username, auth.password):
            return jsonify({"error": "Invalid credentials"}), 403

        return f(*args, **kwargs)
    return decorated

@app.route('/api/protected')
@requires_auth
def protected_resource():
    return jsonify({"message": "Welcome!", "user": request.authorization.username})

@app.route('/api/public')
def public_resource():
    return jsonify({"message": "This is public"})

if __name__ == '__main__':
    app.run(debug=True)
```

### Python (FastAPI)

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPBasic, HTTPBasicCredentials
import secrets

app = FastAPI()
security = HTTPBasic()

# Хранилище пользователей (в production — используй БД с хешированием)
USERS_DB = {
    "admin": "supersecret",
    "developer": "devpass123"
}

def verify_credentials(credentials: HTTPBasicCredentials = Depends(security)):
    """Проверяет учётные данные с защитой от timing attacks"""
    correct_password = USERS_DB.get(credentials.username)

    if correct_password is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid credentials",
            headers={"WWW-Authenticate": "Basic"},
        )

    # Используем secrets.compare_digest для защиты от timing attacks
    is_correct_password = secrets.compare_digest(
        credentials.password.encode("utf8"),
        correct_password.encode("utf8")
    )

    if not is_correct_password:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid credentials",
            headers={"WWW-Authenticate": "Basic"},
        )

    return credentials.username

@app.get("/api/protected")
def read_protected(username: str = Depends(verify_credentials)):
    return {"message": f"Hello, {username}!"}

@app.get("/api/admin")
def admin_only(username: str = Depends(verify_credentials)):
    if username != "admin":
        raise HTTPException(status_code=403, detail="Admin access required")
    return {"message": "Admin panel", "secrets": ["secret1", "secret2"]}
```

### Node.js (Express)

```javascript
const express = require('express');
const app = express();

const USERS = {
    admin: 'secret123',
    user: 'password456'
};

function basicAuth(req, res, next) {
    const authHeader = req.headers.authorization;

    if (!authHeader || !authHeader.startsWith('Basic ')) {
        res.setHeader('WWW-Authenticate', 'Basic realm="API"');
        return res.status(401).json({ error: 'Authorization required' });
    }

    // Декодируем Base64
    const base64Credentials = authHeader.split(' ')[1];
    const credentials = Buffer.from(base64Credentials, 'base64').toString('utf8');
    const [username, password] = credentials.split(':');

    // Проверяем учётные данные
    if (USERS[username] && USERS[username] === password) {
        req.user = username;
        return next();
    }

    return res.status(403).json({ error: 'Invalid credentials' });
}

app.get('/api/protected', basicAuth, (req, res) => {
    res.json({ message: `Welcome, ${req.user}!` });
});

app.get('/api/public', (req, res) => {
    res.json({ message: 'Public endpoint' });
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

### Клиентский код (JavaScript)

```javascript
// Использование fetch с Basic Auth
async function fetchProtectedResource(username, password) {
    const credentials = btoa(`${username}:${password}`);

    const response = await fetch('https://api.example.com/protected', {
        method: 'GET',
        headers: {
            'Authorization': `Basic ${credentials}`,
            'Content-Type': 'application/json'
        }
    });

    if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
    }

    return await response.json();
}

// Использование axios
const axios = require('axios');

const response = await axios.get('https://api.example.com/protected', {
    auth: {
        username: 'admin',
        password: 'secret123'
    }
});
```

### cURL примеры

```bash
# Явное указание учётных данных
curl -u username:password https://api.example.com/protected

# Через заголовок Authorization
curl -H "Authorization: Basic $(echo -n 'username:password' | base64)" \
     https://api.example.com/protected

# Интерактивный ввод пароля (безопаснее)
curl -u username https://api.example.com/protected
```

## Плюсы Basic Auth

| Преимущество | Описание |
|--------------|----------|
| **Простота** | Очень легко реализовать на любом языке |
| **Универсальность** | Поддерживается всеми HTTP-клиентами и браузерами |
| **Stateless** | Сервер не хранит состояние сессии |
| **Стандарт** | Официально описан в RFC 7617 |
| **Совместимость** | Работает с любыми прокси и load balancer'ами |

## Минусы Basic Auth

| Недостаток | Описание |
|------------|----------|
| **Передача пароля** | Пароль передаётся при каждом запросе |
| **Base64 != шифрование** | Легко декодировать, это лишь кодирование |
| **Требует HTTPS** | Без TLS учётные данные передаются открыто |
| **Нет logout** | Невозможно "выйти" без закрытия браузера |
| **Нет expiration** | Учётные данные не истекают автоматически |
| **Уязвимость к replay** | Перехваченный заголовок можно переиспользовать |

## Когда использовать Basic Auth

### Подходит для:

- Внутренних API и микросервисов в защищённой сети
- Простых инструментов и скриптов
- Временных или тестовых решений
- API с низкими требованиями к безопасности
- Machine-to-machine коммуникации (в сочетании с TLS)

### НЕ подходит для:

- Публичных веб-приложений с пользователями
- Мобильных приложений (пароль хранится на устройстве)
- API, требующих высокого уровня безопасности
- Систем, где нужен granular access control
- Случаев, когда нужна возможность отзыва доступа

## Вопросы безопасности

### Обязательные меры

1. **Всегда используй HTTPS** — без TLS Basic Auth полностью небезопасен
2. **Хешируй пароли в БД** — используй bcrypt, Argon2 или PBKDF2
3. **Защита от timing attacks** — используй constant-time сравнение
4. **Rate limiting** — ограничивай количество попыток авторизации
5. **Логирование** — записывай неудачные попытки входа

### Пример хеширования паролей

```python
import bcrypt

def hash_password(password: str) -> bytes:
    """Хеширует пароль с использованием bcrypt"""
    salt = bcrypt.gensalt(rounds=12)
    return bcrypt.hashpw(password.encode('utf-8'), salt)

def verify_password(password: str, hashed: bytes) -> bool:
    """Проверяет пароль против хеша"""
    return bcrypt.checkpw(password.encode('utf-8'), hashed)

# Пример использования
hashed = hash_password("mysecretpassword")
print(verify_password("mysecretpassword", hashed))  # True
print(verify_password("wrongpassword", hashed))      # False
```

### Защита от brute force

```python
from functools import wraps
from flask import request, jsonify
import time

# Простой in-memory rate limiter (в production используй Redis)
failed_attempts = {}
LOCKOUT_THRESHOLD = 5
LOCKOUT_TIME = 300  # 5 минут

def rate_limit_auth(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        ip = request.remote_addr

        if ip in failed_attempts:
            attempts, lockout_until = failed_attempts[ip]

            if time.time() < lockout_until:
                return jsonify({
                    "error": "Too many failed attempts. Try again later."
                }), 429

        return f(*args, **kwargs)
    return decorated

def record_failed_attempt(ip: str):
    """Записывает неудачную попытку входа"""
    if ip in failed_attempts:
        attempts, _ = failed_attempts[ip]
        attempts += 1
    else:
        attempts = 1

    if attempts >= LOCKOUT_THRESHOLD:
        failed_attempts[ip] = (attempts, time.time() + LOCKOUT_TIME)
    else:
        failed_attempts[ip] = (attempts, 0)

def clear_failed_attempts(ip: str):
    """Сбрасывает счётчик после успешного входа"""
    if ip in failed_attempts:
        del failed_attempts[ip]
```

## Best Practices

1. **Никогда не используй Basic Auth без HTTPS**
2. **Используй `secrets.compare_digest()` для сравнения паролей** — защита от timing attacks
3. **Храни пароли только в хешированном виде** — bcrypt, Argon2
4. **Реализуй rate limiting** — защита от brute force
5. **Логируй попытки аутентификации** — для аудита безопасности
6. **Не включай учётные данные в URL** — они могут попасть в логи
7. **Рассмотри альтернативы** — для серьёзных проектов используй JWT или OAuth2

## Типичные ошибки

### Ошибка 1: Хранение паролей в открытом виде

```python
# ПЛОХО: пароли хранятся как есть
USERS = {"admin": "password123"}

# ХОРОШО: пароли хешированы
USERS = {"admin": "$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/X4.G"}
```

### Ошибка 2: Сравнение строк напрямую (уязвимость к timing attack)

```python
# ПЛОХО: обычное сравнение
if password == stored_password:
    return True

# ХОРОШО: constant-time сравнение
import secrets
if secrets.compare_digest(password, stored_password):
    return True
```

### Ошибка 3: Отсутствие rate limiting

```python
# ПЛОХО: неограниченные попытки входа
def login(username, password):
    if check_credentials(username, password):
        return "Success"
    return "Failed"  # Атакующий может перебирать бесконечно

# ХОРОШО: с rate limiting
@rate_limit(max_attempts=5, window=300)
def login(username, password):
    ...
```

### Ошибка 4: Учётные данные в логах

```python
# ПЛОХО: логируем всё
logger.info(f"Auth attempt: {username}:{password}")

# ХОРОШО: не логируем пароли
logger.info(f"Auth attempt for user: {username}")
```

## Сравнение с другими методами

| Критерий | Basic Auth | Token Auth | JWT | OAuth2 |
|----------|------------|------------|-----|--------|
| Сложность | Низкая | Средняя | Средняя | Высокая |
| Безопасность | Низкая | Средняя | Высокая | Высокая |
| Stateless | Да | Зависит | Да | Зависит |
| Logout | Нет | Да | Сложно | Да |
| Подходит для API | Ограниченно | Да | Да | Да |

## Резюме

Basic Authentication — это простой, но ограниченный метод аутентификации. Используй его только для внутренних сервисов, быстрых прототипов или когда простота важнее безопасности. Для production-систем с пользователями рассмотри JWT, OAuth2 или session-based аутентификацию.
