# Безопасная обработка данных (Processing)

## Обзор

Безопасная обработка данных — это набор практик для защиты API на уровне бизнес-логики и обработки запросов. Даже при правильной аутентификации и валидации входных данных, ошибки в обработке могут привести к уязвимостям.

---

## 1. Защита всех эндпоинтов аутентификацией

### Проблема
Забытый незащищённый эндпоинт — частая причина утечек данных.

### Решение

```python
# FastAPI: глобальная зависимость для аутентификации
from fastapi import FastAPI, Depends, HTTPException
from fastapi.security import HTTPBearer

security = HTTPBearer()

# Публичные эндпоинты (whitelist)
PUBLIC_PATHS = ["/health", "/docs", "/openapi.json", "/auth/login"]

@app.middleware("http")
async def auth_middleware(request, call_next):
    if request.url.path not in PUBLIC_PATHS:
        auth_header = request.headers.get("Authorization")
        if not auth_header or not verify_token(auth_header):
            raise HTTPException(status_code=401, detail="Unauthorized")
    return await call_next(request)
```

### Best Practice
- Используйте **whitelist** для публичных эндпоинтов, а не blacklist
- Все новые эндпоинты должны требовать аутентификацию по умолчанию
- Регулярно проводите аудит эндпоинтов

---

## 2. UUID вместо auto-increment ID

### Проблема
Последовательные ID позволяют:
- Перебирать ресурсы (`/users/1`, `/users/2`, `/users/3`...)
- Узнать количество пользователей/заказов
- Предсказывать будущие ID

### Плохо

```
GET /api/users/1
GET /api/users/2
GET /api/orders/12345
```

### Хорошо

```
GET /api/users/550e8400-e29b-41d4-a716-446655440000
GET /api/orders/f47ac10b-58cc-4372-a567-0e02b2c3d479
```

### Реализация

```python
import uuid
from sqlalchemy import Column, String
from sqlalchemy.dialects.postgresql import UUID

class User(Base):
    __tablename__ = "users"

    # Используем UUID как первичный ключ
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)

    # Или храним оба: internal auto-increment + public UUID
    internal_id = Column(Integer, primary_key=True, autoincrement=True)
    public_id = Column(UUID(as_uuid=True), unique=True, default=uuid.uuid4, index=True)
```

### Важно
- Всегда проверяйте права доступа, даже с UUID!
- UUID не заменяет авторизацию

---

## 3. Защита от XXE (XML External Entity)

### Что такое XXE?
Атака через внешние сущности XML, позволяющая читать файлы сервера.

### Пример атаки

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<user>
  <name>&xxe;</name>
</user>
```

### Защита в Python

```python
import defusedxml.ElementTree as ET

# БЕЗОПАСНО: использует defusedxml
def parse_xml_safe(xml_string: str):
    return ET.fromstring(xml_string)

# ОПАСНО: стандартный парсер уязвим
# import xml.etree.ElementTree as ET  # НЕ ИСПОЛЬЗОВАТЬ!
```

### Защита при работе с YAML

```python
import yaml

# БЕЗОПАСНО: safe_load
data = yaml.safe_load(yaml_content)

# ОПАСНО: load без Loader
# data = yaml.load(yaml_content)  # НЕ ИСПОЛЬЗОВАТЬ!
```

### Общие правила
- Отключите DTD (Document Type Definition) полностью
- Используйте безопасные парсеры (`defusedxml`, `safe_load`)
- Если возможно, используйте JSON вместо XML

---

## 4. CDN для файловых загрузок

### Зачем?
- Защита основного сервера от нагрузки
- Изоляция потенциально вредоносных файлов
- Лучшая производительность

### Архитектура

```
User → API → Storage (S3/GCS) → CDN → User
         ↓
    Presigned URL
```

### Пример с AWS S3

```python
import boto3
from botocore.config import Config

s3_client = boto3.client(
    's3',
    config=Config(signature_version='s3v4')
)

def generate_upload_url(filename: str, content_type: str) -> str:
    """Генерирует presigned URL для загрузки файла напрямую в S3"""
    return s3_client.generate_presigned_url(
        'put_object',
        Params={
            'Bucket': 'my-uploads-bucket',
            'Key': f'uploads/{uuid.uuid4()}/{filename}',
            'ContentType': content_type,
        },
        ExpiresIn=3600  # URL действителен 1 час
    )

def generate_download_url(file_key: str) -> str:
    """Генерирует presigned URL для скачивания"""
    return s3_client.generate_presigned_url(
        'get_object',
        Params={
            'Bucket': 'my-uploads-bucket',
            'Key': file_key,
        },
        ExpiresIn=3600
    )
```

### Безопасность загрузок
- Проверяйте MIME-type файла
- Ограничивайте размер файла
- Сканируйте на вирусы (ClamAV)
- Не доверяйте расширению файла

---

## 5. Асинхронная обработка больших данных

### Проблема
Долгие операции блокируют HTTP-соединение и могут привести к таймаутам.

### Решение: очереди задач

```python
from celery import Celery
from fastapi import BackgroundTasks

celery_app = Celery('tasks', broker='redis://localhost:6379')

@celery_app.task
def process_large_file(file_id: str):
    """Обработка файла в фоновом режиме"""
    # Длительная операция
    pass

@app.post("/api/files/{file_id}/process")
async def start_processing(file_id: str):
    # Запускаем задачу и сразу возвращаем ответ
    task = process_large_file.delay(file_id)
    return {
        "status": "processing",
        "task_id": task.id,
        "check_status_url": f"/api/tasks/{task.id}"
    }

@app.get("/api/tasks/{task_id}")
async def get_task_status(task_id: str):
    task = celery_app.AsyncResult(task_id)
    return {
        "task_id": task_id,
        "status": task.status,
        "result": task.result if task.ready() else None
    }
```

### Паттерны для больших данных
- **Пагинация** — разбиение на страницы
- **Стриминг** — потоковая передача данных
- **Webhooks** — уведомление о готовности
- **Polling** — клиент проверяет статус

---

## 6. Debug mode OFF в production

### Почему это критично?

Debug mode раскрывает:
- Stack traces с путями к файлам
- Переменные окружения
- SQL-запросы
- Внутреннюю структуру приложения

### Проверка настроек

```python
# settings.py
import os

DEBUG = os.getenv("DEBUG", "false").lower() == "true"
ENVIRONMENT = os.getenv("ENVIRONMENT", "production")

# Проверка при старте
if ENVIRONMENT == "production" and DEBUG:
    raise RuntimeError("DEBUG mode is enabled in production!")
```

### FastAPI

```python
from fastapi import FastAPI

app = FastAPI(
    debug=False,  # Всегда False в production
    docs_url=None if ENVIRONMENT == "production" else "/docs",
    redoc_url=None if ENVIRONMENT == "production" else "/redoc",
)
```

### Django

```python
# settings.py
DEBUG = False  # НИКОГДА True в production

# Кастомные страницы ошибок
ALLOWED_HOSTS = ['api.example.com']
```

### Чек-лист для production
- [ ] `DEBUG = False`
- [ ] Отключена документация API (`/docs`, `/swagger`)
- [ ] Кастомные страницы ошибок (не stack traces)
- [ ] Логи ошибок идут в мониторинг, а не пользователю

---

## 7. Non-executable stacks

### Что это?
Защита памяти от выполнения кода из областей данных (DEP/NX bit).

### Уровни защиты

| Уровень | Технология | Описание |
|---------|------------|----------|
| OS | DEP/NX | Запрет выполнения из стека/кучи |
| Compiler | Stack canaries | Обнаружение переполнения буфера |
| Runtime | ASLR | Рандомизация адресов памяти |

### Для Python/Node.js
Эти языки управляют памятью автоматически, но:
- Используйте актуальные версии интерпретатора
- Обновляйте native-расширения (numpy, pillow, etc.)
- Не используйте `eval()`, `exec()` с пользовательскими данными

### Опасный код

```python
# НИКОГДА не делайте так!
user_input = request.query_params.get("code")
eval(user_input)  # Remote Code Execution!

exec(f"result = {user_input}")  # RCE!
```

### Безопасные альтернативы

```python
# Для математических выражений
import ast

def safe_eval(expression: str) -> float:
    """Безопасное вычисление математических выражений"""
    allowed_nodes = (ast.Expression, ast.Num, ast.BinOp,
                     ast.Add, ast.Sub, ast.Mult, ast.Div)

    tree = ast.parse(expression, mode='eval')

    for node in ast.walk(tree):
        if not isinstance(node, allowed_nodes):
            raise ValueError(f"Недопустимая операция: {type(node)}")

    return eval(compile(tree, '<string>', 'eval'))
```

---

## Чек-лист безопасной обработки

- [ ] Все эндпоинты защищены аутентификацией (whitelist для публичных)
- [ ] Используются UUID вместо sequential ID в URL
- [ ] XML/YAML парсятся безопасными библиотеками
- [ ] Файлы загружаются через CDN/S3, не через API-сервер
- [ ] Долгие операции выполняются асинхронно
- [ ] Debug mode отключён в production
- [ ] Нет использования `eval()`/`exec()` с пользовательскими данными
- [ ] Регулярно обновляются зависимости

---

## Типичные ошибки

| Ошибка | Последствие | Решение |
|--------|-------------|---------|
| Sequential IDs в URL | IDOR, enumeration | UUID |
| `yaml.load()` без safe | RCE | `yaml.safe_load()` |
| `DEBUG=True` в prod | Information disclosure | Environment checks |
| Синхронная обработка | DoS, таймауты | Celery/очереди |
| `eval(user_input)` | RCE | AST parsing, whitelist |
