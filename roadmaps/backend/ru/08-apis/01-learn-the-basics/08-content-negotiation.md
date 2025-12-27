# Content Negotiation (Согласование контента)

## Что такое Content Negotiation

**Content Negotiation** — это механизм HTTP, позволяющий клиенту и серверу договориться о наиболее подходящем представлении ресурса. Один и тот же ресурс может быть представлен в разных форматах (JSON, XML, HTML), на разных языках, с разным сжатием.

## Типы согласования

### 1. Server-driven (Проактивное)

Сервер выбирает представление на основе заголовков запроса клиента.

```http
GET /api/users HTTP/1.1
Accept: application/json
Accept-Language: ru-RU
Accept-Encoding: gzip
```

### 2. Client-driven (Реактивное)

Сервер возвращает список доступных представлений, клиент выбирает.

```http
HTTP/1.1 300 Multiple Choices
Content-Type: text/html

<ul>
  <li><a href="/api/users.json">JSON</a></li>
  <li><a href="/api/users.xml">XML</a></li>
</ul>
```

### 3. Transparent (Прозрачное)

Промежуточный прокси выбирает представление.

## Заголовки согласования

### Accept — тип контента

Указывает предпочтительные MIME-типы ответа.

```http
# Только JSON
Accept: application/json

# JSON предпочтительнее, но XML тоже подойдет
Accept: application/json, application/xml;q=0.9

# Любой формат
Accept: */*

# HTML предпочтительнее, но можно и JSON
Accept: text/html, application/xhtml+xml, application/json;q=0.5
```

**Параметр качества (q)**: от 0 до 1, указывает предпочтение (по умолчанию 1).

```python
# Парсинг Accept заголовка
def parse_accept_header(accept: str) -> list:
    """Парсит Accept заголовок и возвращает отсортированный список типов."""
    types = []
    for item in accept.split(","):
        parts = item.strip().split(";")
        media_type = parts[0].strip()
        q = 1.0
        for part in parts[1:]:
            if part.strip().startswith("q="):
                q = float(part.strip()[2:])
        types.append((media_type, q))

    return sorted(types, key=lambda x: x[1], reverse=True)

# Пример
accept = "application/json, application/xml;q=0.9, */*;q=0.1"
print(parse_accept_header(accept))
# [('application/json', 1.0), ('application/xml', 0.9), ('*/*', 0.1)]
```

### Accept-Language — язык

Предпочтительные языки для ответа.

```http
# Русский предпочтительнее
Accept-Language: ru-RU, ru;q=0.9, en-US;q=0.8, en;q=0.7

# Только английский
Accept-Language: en

# Любой язык
Accept-Language: *
```

### Accept-Encoding — сжатие

Поддерживаемые алгоритмы сжатия.

```http
# Поддержка gzip, deflate и brotli
Accept-Encoding: gzip, deflate, br

# Только gzip
Accept-Encoding: gzip

# Без сжатия
Accept-Encoding: identity
```

### Accept-Charset — кодировка (устаревший)

Предпочтительные кодировки символов. Почти всегда используется UTF-8.

```http
Accept-Charset: utf-8, iso-8859-1;q=0.5
```

## Реализация в FastAPI

### Согласование типа контента

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse, Response
import xml.etree.ElementTree as ET

app = FastAPI()

def to_xml(data: dict) -> str:
    """Конвертирует dict в XML."""
    root = ET.Element("root")
    for key, value in data.items():
        child = ET.SubElement(root, key)
        child.text = str(value)
    return ET.tostring(root, encoding="unicode")

@app.get("/users/{user_id}")
def get_user(user_id: int, request: Request):
    user = {"id": user_id, "name": "John", "email": "john@example.com"}

    accept = request.headers.get("accept", "application/json")

    if "application/xml" in accept:
        xml_content = to_xml(user)
        return Response(content=xml_content, media_type="application/xml")
    elif "text/html" in accept:
        html = f"<h1>{user['name']}</h1><p>{user['email']}</p>"
        return Response(content=html, media_type="text/html")
    else:
        # По умолчанию JSON
        return JSONResponse(content=user)
```

### Более продвинутая реализация

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse, Response
from typing import Callable, Dict
import json

app = FastAPI()

# Сериализаторы для разных форматов
SERIALIZERS: Dict[str, Callable] = {
    "application/json": lambda data: json.dumps(data),
    "application/xml": lambda data: to_xml(data),
    "text/csv": lambda data: to_csv(data),
}

def get_best_media_type(accept_header: str) -> str:
    """Выбирает лучший формат на основе Accept заголовка."""
    if not accept_header:
        return "application/json"

    accepted = parse_accept_header(accept_header)

    for media_type, _ in accepted:
        if media_type in SERIALIZERS:
            return media_type
        if media_type == "*/*":
            return "application/json"

    return None

@app.get("/data")
def get_data(request: Request):
    data = {"id": 1, "name": "Test", "value": 42}

    accept = request.headers.get("accept", "*/*")
    media_type = get_best_media_type(accept)

    if not media_type:
        raise HTTPException(
            status_code=406,
            detail=f"None of the requested formats are supported. "
                   f"Supported: {list(SERIALIZERS.keys())}"
        )

    serializer = SERIALIZERS[media_type]
    content = serializer(data)

    return Response(content=content, media_type=media_type)
```

### Согласование языка

```python
from fastapi import FastAPI, Request

app = FastAPI()

TRANSLATIONS = {
    "en": {"greeting": "Hello", "farewell": "Goodbye"},
    "ru": {"greeting": "Привет", "farewell": "До свидания"},
    "es": {"greeting": "Hola", "farewell": "Adiós"},
}

def get_best_language(accept_language: str) -> str:
    """Выбирает лучший язык на основе Accept-Language."""
    if not accept_language:
        return "en"

    for item in accept_language.split(","):
        lang = item.strip().split(";")[0].strip()
        # Берем основной язык (ru-RU -> ru)
        lang_code = lang.split("-")[0].lower()
        if lang_code in TRANSLATIONS:
            return lang_code

    return "en"

@app.get("/greeting")
def get_greeting(request: Request):
    accept_language = request.headers.get("accept-language", "en")
    lang = get_best_language(accept_language)

    return {
        "language": lang,
        "greeting": TRANSLATIONS[lang]["greeting"],
        "farewell": TRANSLATIONS[lang]["farewell"]
    }
```

### Сжатие ответов

```python
from fastapi import FastAPI
from fastapi.middleware.gzip import GZipMiddleware

app = FastAPI()

# Автоматическое сжатие ответов > 500 байт
app.add_middleware(GZipMiddleware, minimum_size=500)

@app.get("/large-data")
def get_large_data():
    # Большой ответ будет автоматически сжат
    return {"data": ["item"] * 1000}
```

## Content-Type в запросах

Клиент указывает формат отправляемых данных.

```python
from fastapi import FastAPI, Request, HTTPException
from pydantic import BaseModel
import json
import xml.etree.ElementTree as ET

app = FastAPI()

class User(BaseModel):
    name: str
    email: str

def parse_xml_user(xml_string: str) -> dict:
    """Парсит XML в dict."""
    root = ET.fromstring(xml_string)
    return {
        "name": root.find("name").text,
        "email": root.find("email").text
    }

@app.post("/users")
async def create_user(request: Request):
    content_type = request.headers.get("content-type", "")
    body = await request.body()

    if "application/json" in content_type:
        data = json.loads(body)
    elif "application/xml" in content_type:
        data = parse_xml_user(body.decode())
    elif "application/x-www-form-urlencoded" in content_type:
        form = await request.form()
        data = dict(form)
    else:
        raise HTTPException(
            status_code=415,
            detail=f"Unsupported content type: {content_type}"
        )

    user = User(**data)
    return {"created": user.dict()}
```

## Клиентская сторона

### Python requests

```python
import requests

# Запрос JSON
response = requests.get(
    "https://api.example.com/users",
    headers={"Accept": "application/json"}
)

# Запрос XML
response = requests.get(
    "https://api.example.com/users",
    headers={"Accept": "application/xml"}
)

# Запрос с предпочтением языка
response = requests.get(
    "https://api.example.com/content",
    headers={"Accept-Language": "ru-RU, ru;q=0.9, en;q=0.8"}
)

# Отправка данных в разных форматах
# JSON
response = requests.post(
    "https://api.example.com/users",
    json={"name": "John"},  # Автоматически Content-Type: application/json
)

# Form data
response = requests.post(
    "https://api.example.com/users",
    data={"name": "John"},  # Content-Type: application/x-www-form-urlencoded
)

# XML
response = requests.post(
    "https://api.example.com/users",
    data="<user><name>John</name></user>",
    headers={"Content-Type": "application/xml"}
)
```

## Версионирование через Content Negotiation

Альтернатива версионированию в URL.

```http
# Версия в Accept заголовке
Accept: application/vnd.api+json; version=2
Accept: application/vnd.myapi.v2+json

# Кастомный заголовок
X-API-Version: 2
```

```python
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()

@app.get("/users")
def get_users(request: Request):
    accept = request.headers.get("accept", "")

    # Парсим версию из Accept
    if "version=2" in accept or "v2" in accept:
        return get_users_v2()
    elif "version=1" in accept or "v1" in accept:
        return get_users_v1()
    else:
        # По умолчанию последняя версия
        return get_users_v2()

def get_users_v1():
    return {"version": 1, "users": ["user1", "user2"]}

def get_users_v2():
    return {"version": 2, "data": {"users": [{"id": 1}, {"id": 2}]}}
```

## Ответы сервера

### Vary заголовок

Указывает, по каким заголовкам кэш должен различать ответы.

```http
HTTP/1.1 200 OK
Content-Type: application/json
Vary: Accept, Accept-Language, Accept-Encoding

{"data": "..."}
```

```python
from fastapi import FastAPI
from fastapi.responses import Response

app = FastAPI()

@app.get("/data")
def get_data():
    response = Response(content='{"data": "..."}', media_type="application/json")
    response.headers["Vary"] = "Accept, Accept-Language"
    return response
```

### 406 Not Acceptable

Сервер не может удовлетворить Accept заголовок.

```python
from fastapi import HTTPException

SUPPORTED_TYPES = ["application/json", "application/xml"]

@app.get("/resource")
def get_resource(request: Request):
    accept = request.headers.get("accept", "*/*")

    if not any(t in accept for t in SUPPORTED_TYPES) and "*/*" not in accept:
        raise HTTPException(
            status_code=406,
            detail={
                "error": "Not Acceptable",
                "supported_types": SUPPORTED_TYPES
            }
        )
```

### 415 Unsupported Media Type

Сервер не понимает Content-Type запроса.

```python
SUPPORTED_CONTENT_TYPES = ["application/json", "application/xml"]

@app.post("/data")
async def post_data(request: Request):
    content_type = request.headers.get("content-type", "")

    if not any(t in content_type for t in SUPPORTED_CONTENT_TYPES):
        raise HTTPException(
            status_code=415,
            detail={
                "error": "Unsupported Media Type",
                "supported_types": SUPPORTED_CONTENT_TYPES
            }
        )
```

## Best Practices

### 1. Всегда указывайте Content-Type в ответе

```python
@app.get("/data")
def get_data():
    return Response(
        content='{"data": "value"}',
        media_type="application/json"  # Явно указываем тип
    )
```

### 2. Имейте разумное значение по умолчанию

```python
def get_best_format(accept: str) -> str:
    # Если клиент не указал или принимает все
    if not accept or "*/*" in accept:
        return "application/json"  # Разумное значение по умолчанию
    # ...
```

### 3. Используйте Vary для правильного кэширования

```python
@app.get("/content")
def get_content(request: Request):
    response = get_negotiated_response(request)
    response.headers["Vary"] = "Accept, Accept-Language"
    return response
```

### 4. Документируйте поддерживаемые форматы

```python
from fastapi import FastAPI

app = FastAPI(
    title="My API",
    description="""
    ## Supported formats

    This API supports the following content types:
    - `application/json` (default)
    - `application/xml`
    - `text/csv`

    Use the `Accept` header to request specific format.
    """
)
```

### 5. Валидируйте Content-Type входящих данных

```python
@app.post("/users")
async def create_user(request: Request):
    content_type = request.headers.get("content-type", "")

    if "application/json" not in content_type:
        raise HTTPException(
            status_code=415,
            detail="Content-Type must be application/json"
        )
```

## Типичные ошибки

1. **Игнорирование Accept заголовка**
   ```python
   # Плохо - всегда JSON
   return JSONResponse(data)

   # Хорошо - учитываем предпочтения клиента
   return negotiate_response(request, data)
   ```

2. **Отсутствие 406/415 ответов**
   ```python
   # Плохо - молча возвращаем JSON
   return JSONResponse(data)

   # Хорошо - сообщаем о неподдерживаемом формате
   raise HTTPException(status_code=406)
   ```

3. **Забытый Vary заголовок**
   ```python
   # Может вызвать проблемы с кэшированием
   # Один и тот же URL, разный контент

   # Хорошо
   response.headers["Vary"] = "Accept"
   ```

4. **Несогласованность форматов**
   ```python
   # Плохо - ошибки в JSON, успех в XML
   # Все ответы должны быть в согласованном формате
   ```
