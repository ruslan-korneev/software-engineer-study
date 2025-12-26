# URL, Query & Path Parameters

## Структура URL

**URL (Uniform Resource Locator)** — это адрес ресурса в интернете. Полная структура URL:

```
https://user:password@api.example.com:8080/v1/users/123?sort=name&limit=10#section
|_____|  |__________|  |_____________| |__| |_________| |________________| |______|
  |          |               |           |       |              |            |
схема   credentials        хост        порт    путь     query-параметры  fragment
```

### Компоненты URL

| Компонент | Описание | Пример |
|-----------|----------|--------|
| Scheme | Протокол | `https`, `http`, `ftp` |
| User:Password | Учетные данные (устаревший) | `user:pass` |
| Host | Доменное имя или IP | `api.example.com` |
| Port | Порт сервера | `8080` |
| Path | Путь к ресурсу | `/v1/users/123` |
| Query String | Параметры запроса | `?sort=name&limit=10` |
| Fragment | Якорь (не отправляется на сервер) | `#section` |

## Path Parameters (Параметры пути)

Path parameters — это динамические части URL-пути, которые идентифицируют конкретный ресурс.

### Синтаксис

```
/users/{user_id}
/users/{user_id}/orders/{order_id}
/categories/{category}/products/{product_id}
```

### Примеры в FastAPI

```python
from fastapi import FastAPI, Path
from typing import Optional

app = FastAPI()

# Простой path parameter
@app.get("/users/{user_id}")
def get_user(user_id: int):
    return {"user_id": user_id}

# Несколько path parameters
@app.get("/users/{user_id}/orders/{order_id}")
def get_order(user_id: int, order_id: int):
    return {"user_id": user_id, "order_id": order_id}

# С валидацией
@app.get("/items/{item_id}")
def get_item(
    item_id: int = Path(
        ...,
        title="Item ID",
        description="ID товара для получения",
        gt=0,  # greater than 0
        le=1000  # less than or equal 1000
    )
):
    return {"item_id": item_id}

# Enum значения
from enum import Enum

class ModelName(str, Enum):
    alexnet = "alexnet"
    resnet = "resnet"
    lenet = "lenet"

@app.get("/models/{model_name}")
def get_model(model_name: ModelName):
    return {"model": model_name.value}
```

### Запросы клиента

```python
import requests

# Простой path parameter
response = requests.get("https://api.example.com/users/123")

# Несколько параметров
response = requests.get("https://api.example.com/users/123/orders/456")

# Динамическое построение URL
user_id = 123
order_id = 456
url = f"https://api.example.com/users/{user_id}/orders/{order_id}"
response = requests.get(url)
```

## Query Parameters (Параметры запроса)

Query parameters — это именованные параметры после знака `?` в URL, разделенные `&`.

### Синтаксис

```
/users?page=1&limit=10&sort=name&order=asc
```

### Примеры в FastAPI

```python
from fastapi import FastAPI, Query
from typing import Optional, List

app = FastAPI()

# Обязательный query parameter
@app.get("/search")
def search(q: str):
    return {"query": q}

# Опциональный query parameter с дефолтом
@app.get("/users")
def get_users(
    page: int = 1,
    limit: int = 10,
    sort_by: Optional[str] = None
):
    return {
        "page": page,
        "limit": limit,
        "sort_by": sort_by
    }

# С валидацией
@app.get("/items")
def get_items(
    q: Optional[str] = Query(
        None,
        min_length=3,
        max_length=50,
        regex="^[a-zA-Z0-9]+$",
        description="Поисковый запрос"
    ),
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100)
):
    return {"q": q, "skip": skip, "limit": limit}

# Множественные значения (список)
@app.get("/products")
def get_products(
    category: List[str] = Query(None)
):
    # /products?category=electronics&category=books
    return {"categories": category}

# Алиасы
@app.get("/search-v2")
def search_v2(
    search_query: str = Query(..., alias="q"),
    page_number: int = Query(1, alias="page")
):
    # /search-v2?q=test&page=2
    return {"query": search_query, "page": page_number}
```

### Запросы клиента

```python
import requests

# Простые query параметры
response = requests.get(
    "https://api.example.com/users",
    params={"page": 1, "limit": 10}
)
# URL: /users?page=1&limit=10

# Несколько значений для одного параметра
response = requests.get(
    "https://api.example.com/products",
    params={"category": ["electronics", "books"]}
)
# URL: /products?category=electronics&category=books

# Комбинация path и query параметров
response = requests.get(
    "https://api.example.com/users/123/orders",
    params={"status": "completed", "limit": 5}
)
# URL: /users/123/orders?status=completed&limit=5
```

## Path vs Query Parameters

### Когда использовать Path Parameters

- Идентификация конкретного ресурса
- Обязательные параметры
- Иерархические отношения

```python
# Хорошо
GET /users/123              # Конкретный пользователь
GET /users/123/orders/456   # Заказ конкретного пользователя
GET /categories/electronics/products  # Продукты в категории

# Плохо
GET /users?id=123           # ID лучше в path
```

### Когда использовать Query Parameters

- Фильтрация, сортировка, пагинация
- Опциональные параметры
- Множественные значения

```python
# Хорошо
GET /users?page=1&limit=10           # Пагинация
GET /users?status=active&role=admin  # Фильтрация
GET /users?sort=name&order=desc      # Сортировка
GET /products?category=a&category=b  # Множественные значения

# Плохо
GET /users/page/1/limit/10           # Пагинация лучше в query
```

## URL Encoding

Специальные символы в URL должны быть закодированы.

```python
from urllib.parse import quote, quote_plus, urlencode

# Кодирование отдельного значения
search = "hello world"
encoded = quote(search)  # "hello%20world"

# Кодирование для query string
encoded_plus = quote_plus(search)  # "hello+world"

# Полное кодирование параметров
params = {
    "q": "hello world",
    "filter": "price>100",
    "tag": "новый товар"
}
query_string = urlencode(params)
# "q=hello+world&filter=price%3E100&tag=%D0%BD%D0%BE%D0%B2%D1%8B%D0%B9+%D1%82%D0%BE%D0%B2%D0%B0%D1%80"

# requests делает кодирование автоматически
import requests
response = requests.get(
    "https://api.example.com/search",
    params={"q": "hello world", "tag": "новый"}
)
print(response.url)
# https://api.example.com/search?q=hello+world&tag=%D0%BD%D0%BE%D0%B2%D1%8B%D0%B9
```

### Таблица кодирования

| Символ | Закодировано |
|--------|--------------|
| Пробел | `%20` или `+` |
| `&` | `%26` |
| `=` | `%3D` |
| `?` | `%3F` |
| `/` | `%2F` |
| `#` | `%23` |
| `%` | `%25` |

## Request Body vs URL Parameters

### Когда использовать тело запроса

- Большие объемы данных
- Сложные структуры (вложенные объекты)
- Чувствительные данные
- POST, PUT, PATCH запросы

```python
# В теле запроса
POST /users
{
    "name": "John Doe",
    "email": "john@example.com",
    "address": {
        "city": "Moscow",
        "street": "Tverskaya"
    }
}
```

### Когда использовать URL параметры

- Простые значения
- Идентификаторы ресурсов
- Фильтрация и пагинация
- GET запросы (не должны иметь тело)

```python
# В URL
GET /users/123/orders?status=pending&page=1&limit=10
```

## Практический пример: API с разными типами параметров

```python
from fastapi import FastAPI, Path, Query, Body
from pydantic import BaseModel
from typing import Optional, List
from datetime import date

app = FastAPI()

class OrderFilter(BaseModel):
    min_amount: Optional[float] = None
    max_amount: Optional[float] = None

class Order(BaseModel):
    product_id: int
    quantity: int

# Комбинация всех типов параметров
@app.get("/users/{user_id}/orders")
def get_user_orders(
    # Path parameter - идентификация ресурса
    user_id: int = Path(..., gt=0, description="ID пользователя"),

    # Query parameters - фильтрация и пагинация
    status: Optional[str] = Query(
        None,
        enum=["pending", "completed", "cancelled"]
    ),
    created_after: Optional[date] = Query(None),
    created_before: Optional[date] = Query(None),
    sort_by: str = Query("created_at"),
    order: str = Query("desc", regex="^(asc|desc)$"),
    page: int = Query(1, ge=1),
    limit: int = Query(10, ge=1, le=100)
):
    return {
        "user_id": user_id,
        "filters": {
            "status": status,
            "created_after": created_after,
            "created_before": created_before
        },
        "sorting": {"by": sort_by, "order": order},
        "pagination": {"page": page, "limit": limit}
    }

# POST с path и body параметрами
@app.post("/users/{user_id}/orders")
def create_order(
    user_id: int = Path(..., gt=0),
    order: Order = Body(...)
):
    return {
        "user_id": user_id,
        "order": order.dict()
    }
```

### Клиентские запросы

```python
import requests

# GET с path и query параметрами
response = requests.get(
    "https://api.example.com/users/123/orders",
    params={
        "status": "pending",
        "created_after": "2024-01-01",
        "page": 1,
        "limit": 20
    }
)

# POST с path и body
response = requests.post(
    "https://api.example.com/users/123/orders",
    json={
        "product_id": 456,
        "quantity": 2
    }
)
```

## Best Practices

### 1. Используйте множественное число для коллекций

```python
# Хорошо
GET /users        # Коллекция
GET /users/123    # Элемент

# Плохо
GET /user
GET /user/123
```

### 2. Используйте lowercase и дефисы

```python
# Хорошо
GET /user-profiles
GET /order-items

# Плохо
GET /userProfiles
GET /order_items
```

### 3. Не включайте действия в URL

```python
# Плохо
POST /users/create
GET /users/123/delete

# Хорошо
POST /users
DELETE /users/123
```

### 4. Версионирование API в path

```python
# Хорошо
GET /v1/users
GET /v2/users

# Альтернатива через заголовок
GET /users
Accept: application/vnd.api+json; version=2
```

### 5. Вложенность не более 2-3 уровней

```python
# Хорошо
GET /users/123/orders
GET /orders/456/items

# Плохо (слишком глубоко)
GET /users/123/orders/456/items/789/reviews
```

### 6. Пагинация для списков

```python
# Offset-based
GET /users?page=2&limit=20

# Cursor-based (лучше для больших данных)
GET /users?cursor=abc123&limit=20
```

## Типичные ошибки

1. **Использование query параметров для идентификации**
   ```python
   # Плохо
   GET /users?id=123

   # Хорошо
   GET /users/123
   ```

2. **Чувствительные данные в URL**
   ```python
   # Плохо (попадает в логи!)
   GET /login?username=john&password=secret

   # Хорошо
   POST /login
   {"username": "john", "password": "secret"}
   ```

3. **Слишком длинные URL**
   ```python
   # URL имеет ограничение (~2000 символов для браузеров)
   # Для сложных фильтров используйте POST
   POST /users/search
   {
       "filters": {...сложные фильтры...}
   }
   ```

4. **Отсутствие валидации параметров**
   ```python
   # Всегда валидируйте
   @app.get("/users/{user_id}")
   def get_user(user_id: int = Path(..., gt=0)):
       pass

   @app.get("/items")
   def get_items(limit: int = Query(10, ge=1, le=100)):
       pass
   ```

5. **Игнорирование URL encoding**
   ```python
   # Плохо
   url = f"/search?q={user_input}"

   # Хорошо
   from urllib.parse import quote
   url = f"/search?q={quote(user_input)}"

   # Или лучше
   requests.get("/search", params={"q": user_input})
   ```
