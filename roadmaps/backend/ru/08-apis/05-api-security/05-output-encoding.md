# Кодирование выходных данных (Output Encoding)

## Что такое Output Encoding?

**Output Encoding** — процесс преобразования данных в безопасный формат перед их отправкой клиенту. Защищает от атак, связанных с интерпретацией данных в неправильном контексте.

Даже если входные данные прошли валидацию, при выводе они могут быть интерпретированы как код.

---

## Типы атак, от которых защищает Output Encoding

### 1. XSS (Cross-Site Scripting)

Внедрение вредоносного JavaScript в страницу.

**Пример уязвимости:**
```python
# ПЛОХО: Данные пользователя вставляются без обработки
@app.get("/profile/{username}")
async def get_profile(username: str):
    user = await db.get_user(username)
    return f"<h1>Welcome, {user.name}!</h1>"
    # Если user.name = "<script>alert('XSS')</script>" — XSS!
```

### 2. Инъекция в JSON

Нарушение структуры JSON-ответа.

**Пример уязвимости:**
```python
# ПЛОХО: Ручное формирование JSON
name = 'John", "admin": true, "x": "'
return f'{{"name": "{name}"}}'
# Результат: {"name": "John", "admin": true, "x": ""}
```

### 3. HTTP Header Injection

Внедрение заголовков через данные.

**Пример:**
```
Location: /redirect?url=http://evil.com%0d%0aSet-Cookie:%20session=malicious
```

---

## Контексты кодирования

Разные контексты требуют разного кодирования!

| Контекст | Опасные символы | Метод кодирования |
|----------|-----------------|-------------------|
| HTML Body | `< > & " '` | HTML entity encoding |
| HTML Attribute | `" ' &` | HTML attribute encoding |
| JavaScript | `' " \ /` | JavaScript encoding |
| URL | Спецсимволы | URL encoding |
| CSS | `( ) ;` | CSS encoding |
| JSON | `" \ /` | JSON encoding |

---

## HTML Encoding

### Базовое HTML экранирование

```python
import html

def safe_html(text: str) -> str:
    """Экранирует HTML специальные символы"""
    return html.escape(text)

# Примеры:
safe_html("<script>alert('xss')</script>")
# Результат: "&lt;script&gt;alert(&#x27;xss&#x27;)&lt;/script&gt;"

safe_html('"><img src=x onerror=alert(1)>')
# Результат: "&quot;&gt;&lt;img src=x onerror=alert(1)&gt;"
```

### Использование в Jinja2

```python
from jinja2 import Environment, select_autoescape

# Автоматическое экранирование
env = Environment(
    autoescape=select_autoescape(['html', 'xml'])
)

template = env.from_string("<h1>Hello, {{ name }}!</h1>")
result = template.render(name="<script>alert('xss')</script>")
# Результат: <h1>Hello, &lt;script&gt;alert(&#39;xss&#39;)&lt;/script&gt;!</h1>
```

### Безопасное встраивание HTML (если нужен rich text)

```python
import bleach

def sanitize_rich_text(html_content: str) -> str:
    """Разрешает только безопасные теги"""
    allowed_tags = [
        'p', 'br', 'b', 'i', 'u', 'strong', 'em',
        'h1', 'h2', 'h3', 'h4',
        'ul', 'ol', 'li',
        'a', 'img'
    ]

    allowed_attrs = {
        'a': ['href', 'title'],
        'img': ['src', 'alt', 'width', 'height']
    }

    # Протоколы для ссылок
    allowed_protocols = ['http', 'https', 'mailto']

    return bleach.clean(
        html_content,
        tags=allowed_tags,
        attributes=allowed_attrs,
        protocols=allowed_protocols,
        strip=True  # Удаляем запрещённые теги вместо экранирования
    )

# Пример:
sanitize_rich_text('<p>Hello <script>evil()</script> world!</p>')
# Результат: '<p>Hello  world!</p>'

sanitize_rich_text('<a href="javascript:alert(1)">click</a>')
# Результат: '<a>click</a>'  # href удалён, т.к. протокол не разрешён
```

---

## JSON Encoding

### Правильная сериализация в JSON

```python
import json
from fastapi.responses import JSONResponse

# ПЛОХО: Ручное формирование JSON
def bad_json_response(user):
    return f'{{"name": "{user.name}", "bio": "{user.bio}"}}'
    # Уязвимо к injection!

# ХОРОШО: Использование json.dumps
def safe_json_response(user):
    return json.dumps({"name": user.name, "bio": user.bio})

# ЛУЧШЕ: FastAPI делает это автоматически
@app.get("/user/{user_id}")
async def get_user(user_id: str):
    user = await db.get_user(user_id)
    return {"name": user.name, "bio": user.bio}
    # FastAPI безопасно сериализует в JSON
```

### Экранирование для встраивания в HTML

```python
import json
import html

def json_for_html_script(data: dict) -> str:
    """
    Безопасное встраивание JSON в <script> тег.
    Защита от </script> внутри данных.
    """
    json_str = json.dumps(data, ensure_ascii=False)

    # Экранируем символы, которые могут сломать <script>
    json_str = json_str.replace('</', '<\\/')
    json_str = json_str.replace('<!--', '<\\!--')

    return json_str

# Использование в шаблоне:
"""
<script>
    const data = {{ json_for_html_script(user_data) | safe }};
</script>
"""
```

---

## URL Encoding

```python
from urllib.parse import quote, quote_plus, urlencode

# Кодирование пути URL
path = quote("файл с пробелами.txt")
# Результат: "%D1%84%D0%B0%D0%B9%D0%BB%20%D1%81%20%D0%BF%D1%80%D0%BE%D0%B1%D0%B5%D0%BB%D0%B0%D0%BC%D0%B8.txt"

# Кодирование query string
query = quote_plus("search query")
# Результат: "search+query"

# Полная query string
params = urlencode({
    "q": "test & test",
    "page": 1,
    "sort": "name"
})
# Результат: "q=test+%26+test&page=1&sort=name"
```

### Безопасные редиректы

```python
from urllib.parse import urlparse, urljoin
from fastapi import HTTPException, Request
from fastapi.responses import RedirectResponse

ALLOWED_HOSTS = {'example.com', 'www.example.com'}

def safe_redirect(url: str, request: Request) -> RedirectResponse:
    """
    Безопасный редирект — только на разрешённые хосты.
    Защита от Open Redirect.
    """
    # Парсим URL
    parsed = urlparse(url)

    # Если это относительный путь — разрешаем
    if not parsed.netloc:
        # Убеждаемся, что путь не начинается с //
        if url.startswith('//'):
            raise HTTPException(status_code=400, detail="Invalid redirect URL")
        safe_url = urljoin(str(request.base_url), url)
        return RedirectResponse(url=safe_url)

    # Проверяем хост
    if parsed.netloc not in ALLOWED_HOSTS:
        raise HTTPException(status_code=400, detail="Redirect to external host not allowed")

    # Убеждаемся в безопасном протоколе
    if parsed.scheme not in ('http', 'https'):
        raise HTTPException(status_code=400, detail="Invalid URL scheme")

    return RedirectResponse(url=url)

@app.get("/redirect")
async def redirect_handler(url: str, request: Request):
    return safe_redirect(url, request)
```

---

## HTTP Headers Encoding

### Защита от Header Injection

```python
from fastapi import Response, HTTPException
import re

def safe_header_value(value: str) -> str:
    """
    Очищает значение для HTTP заголовка.
    Удаляет символы переноса строки.
    """
    # Удаляем CRLF и другие опасные символы
    cleaned = re.sub(r'[\r\n\x00]', '', value)
    return cleaned

@app.get("/download")
async def download_file(filename: str):
    # Валидация имени файла
    if not re.match(r'^[a-zA-Z0-9_.-]+$', filename):
        raise HTTPException(status_code=400, detail="Invalid filename")

    # Безопасное формирование заголовка
    safe_filename = safe_header_value(filename)

    response = Response(content=file_content)
    response.headers["Content-Disposition"] = f'attachment; filename="{safe_filename}"'
    return response
```

### Установка Security Headers

```python
from fastapi import FastAPI
from starlette.middleware.base import BaseHTTPMiddleware

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        response = await call_next(request)

        # Защита от XSS
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["X-Content-Type-Options"] = "nosniff"

        # Content Security Policy
        response.headers["Content-Security-Policy"] = (
            "default-src 'self'; "
            "script-src 'self' 'unsafe-inline'; "
            "style-src 'self' 'unsafe-inline'; "
            "img-src 'self' data: https:; "
            "font-src 'self'; "
            "connect-src 'self'; "
            "frame-ancestors 'none'; "
        )

        # Защита от clickjacking
        response.headers["X-Frame-Options"] = "DENY"

        # HSTS (только HTTPS)
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"

        # Referrer Policy
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"

        return response

app.add_middleware(SecurityHeadersMiddleware)
```

---

## Content-Type правильный

Всегда указывайте правильный Content-Type!

```python
from fastapi import Response
from fastapi.responses import JSONResponse, HTMLResponse, PlainTextResponse

# JSON — используйте JSONResponse
@app.get("/api/data")
async def get_json_data():
    return JSONResponse(
        content={"key": "value"},
        headers={"Content-Type": "application/json; charset=utf-8"}
    )

# HTML — используйте HTMLResponse
@app.get("/page")
async def get_html():
    return HTMLResponse(
        content="<h1>Hello</h1>",
        headers={"Content-Type": "text/html; charset=utf-8"}
    )

# Текст — используйте PlainTextResponse
@app.get("/text")
async def get_text():
    return PlainTextResponse(
        content="Plain text",
        headers={"Content-Type": "text/plain; charset=utf-8"}
    )
```

---

## Защита API Response

### Структурированные ответы

```python
from pydantic import BaseModel
from typing import Generic, TypeVar, Optional, List

T = TypeVar('T')

class APIResponse(BaseModel, Generic[T]):
    success: bool
    data: Optional[T] = None
    error: Optional[str] = None

    class Config:
        # Автоматически конвертирует объекты в JSON-serializable формат
        json_encoders = {
            # Кастомные энкодеры
        }

class UserResponse(BaseModel):
    id: str
    name: str
    email: str
    # Не включаем чувствительные данные!
    # password_hash: str  # НИКОГДА!

@app.get("/users/{user_id}", response_model=APIResponse[UserResponse])
async def get_user(user_id: str):
    user = await db.get_user(user_id)

    # Pydantic фильтрует лишние поля
    return APIResponse(
        success=True,
        data=UserResponse(
            id=user.id,
            name=user.name,
            email=user.email
        )
    )
```

### Фильтрация чувствительных данных

```python
from pydantic import BaseModel, Field

class User(BaseModel):
    id: str
    username: str
    email: str
    password_hash: str = Field(..., exclude=True)  # Исключается из JSON
    api_key: str = Field(..., exclude=True)

    class Config:
        # Или через schema_extra
        fields = {
            'password_hash': {'exclude': True},
            'api_key': {'exclude': True}
        }

# Альтернатива: разные модели для разных контекстов
class UserDB(BaseModel):
    """Модель для БД — полная"""
    id: str
    username: str
    email: str
    password_hash: str
    api_key: str

class UserPublic(BaseModel):
    """Модель для API — публичная"""
    id: str
    username: str

class UserPrivate(BaseModel):
    """Модель для владельца — приватная"""
    id: str
    username: str
    email: str

@app.get("/users/{user_id}", response_model=UserPublic)
async def get_user_public(user_id: str):
    user = await db.get_user(user_id)
    return user  # Pydantic преобразует к UserPublic

@app.get("/users/me", response_model=UserPrivate)
async def get_current_user(current_user: User = Depends(get_current_user)):
    return current_user  # Больше информации для владельца
```

---

## Кодирование логов

Логи тоже должны быть безопасными!

```python
import logging
import json

class SafeFormatter(logging.Formatter):
    """Форматтер, экранирующий спецсимволы в логах"""

    def format(self, record):
        # Экранируем сообщение
        if isinstance(record.msg, str):
            # Заменяем CRLF на видимые символы
            record.msg = record.msg.replace('\n', '\\n').replace('\r', '\\r')

        # Экранируем аргументы
        if record.args:
            record.args = tuple(
                self._escape_arg(arg) for arg in record.args
            )

        return super().format(record)

    def _escape_arg(self, arg):
        if isinstance(arg, str):
            return arg.replace('\n', '\\n').replace('\r', '\\r')
        return arg

# Для JSON логов
class JSONLogFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "message": record.getMessage(),
            "module": record.module,
        }

        # Добавляем extra поля
        if hasattr(record, 'user_id'):
            log_data['user_id'] = record.user_id
        if hasattr(record, 'request_id'):
            log_data['request_id'] = record.request_id

        # JSON.dumps безопасно сериализует
        return json.dumps(log_data, ensure_ascii=False)
```

---

## Best Practices

1. **Кодируйте для контекста** — HTML, JSON, URL требуют разного кодирования
2. **Используйте фреймворки** — FastAPI/Jinja2 делают кодирование автоматически
3. **Никогда не формируйте JSON/HTML вручную** — используйте библиотеки
4. **Указывайте Content-Type** — браузер должен знать тип контента
5. **Устанавливайте Security Headers** — CSP, X-XSS-Protection
6. **Разделяйте модели** — разные модели для разных контекстов вывода
7. **Не раскрывайте лишнего** — фильтруйте чувствительные поля

---

## Чек-лист Output Encoding

- [ ] HTML данные экранируются через `html.escape()` или Jinja2 autoescape
- [ ] JSON формируется через `json.dumps()`, не конкатенацией строк
- [ ] URL параметры кодируются через `urllib.parse`
- [ ] HTTP заголовки очищаются от CRLF
- [ ] Установлен Content-Type для всех ответов
- [ ] Настроены Security Headers (CSP, X-XSS-Protection, etc.)
- [ ] Response models не включают чувствительные поля
- [ ] Редиректы проверяют разрешённые хосты
- [ ] Логи экранируют спецсимволы
