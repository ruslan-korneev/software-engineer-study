# URI Design (Проектирование URI)

## Введение

**URI** (Uniform Resource Identifier) — это уникальный идентификатор ресурса в REST API. Правильное проектирование URI критически важно для создания интуитивно понятного, последовательного и удобного API.

```
https://api.example.com:443/v1/users/123/orders?status=pending&limit=10#section
└─┬─┘   └──────┬──────┘└┬┘└────────────┬────────┘└──────────┬─────────┘└───┬───┘
scheme       host      port          path                 query         fragment
```

## Основные принципы проектирования URI

### 1. Используйте существительные, а не глаголы

URI должен описывать **ресурс** (что), а не **действие** (как). Действие определяется HTTP-методом.

```
# Правильно
GET    /users          # Получить список пользователей
POST   /users          # Создать пользователя
GET    /users/123      # Получить пользователя с ID 123
PUT    /users/123      # Обновить пользователя
DELETE /users/123      # Удалить пользователя

# Неправильно
GET    /getUsers
POST   /createUser
GET    /getUserById/123
POST   /deleteUser/123
```

### 2. Используйте множественное число для коллекций

```
# Правильно
/users
/products
/orders
/categories

# Менее предпочтительно (но допустимо при консистентности)
/user
/product
/order
```

### 3. Иерархия ресурсов

Вложенные ресурсы отражают отношения между сущностями:

```
/users/{userId}/orders                 # Заказы пользователя
/users/{userId}/orders/{orderId}       # Конкретный заказ пользователя
/users/{userId}/orders/{orderId}/items # Товары в заказе

/organizations/{orgId}/teams           # Команды организации
/organizations/{orgId}/teams/{teamId}/members  # Члены команды
```

**Ограничение глубины вложенности:**

```
# Слишком глубоко (плохо)
/countries/{id}/states/{id}/cities/{id}/districts/{id}/streets/{id}/buildings/{id}

# Лучше использовать плоскую структуру
/buildings/{id}
/buildings?city={cityId}&district={districtId}
```

### 4. Используйте kebab-case для составных слов

```
# Правильно
/user-profiles
/order-items
/shipping-addresses
/credit-cards

# Неправильно
/userProfiles      # camelCase
/user_profiles     # snake_case
/UserProfiles      # PascalCase
```

### 5. Используйте строчные буквы

```
# Правильно
/api/users/123/orders

# Неправильно
/API/Users/123/Orders
/Api/USERS/123/orders
```

## Структура URI

### Базовый путь (Base Path)

```
https://api.example.com/v1
└─────────┬──────────┘ └┬┘
       base URL       version
```

**Варианты размещения API:**

```
# Поддомен
https://api.example.com/v1/users

# Путь
https://example.com/api/v1/users

# Отдельный домен
https://example-api.com/v1/users
```

### Идентификаторы ресурсов

```
# Числовые ID
/users/123
/products/456

# UUID
/users/550e8400-e29b-41d4-a716-446655440000

# Slug (человекочитаемые)
/articles/how-to-design-rest-api
/products/iphone-15-pro-max

# Составные ключи
/orders/2024/ORD-00123
```

### Query параметры

Используются для фильтрации, сортировки, пагинации:

```http
GET /api/products?category=electronics&brand=apple&price_min=100&price_max=1000

GET /api/users?role=admin&status=active&sort=-created_at&page=2&limit=20

GET /api/orders?created_after=2024-01-01&created_before=2024-12-31
```

**Типичные query параметры:**

| Параметр | Назначение | Пример |
|----------|------------|--------|
| `page`, `limit` | Пагинация | `?page=2&limit=20` |
| `offset`, `limit` | Пагинация | `?offset=40&limit=20` |
| `sort` | Сортировка | `?sort=-created_at,name` |
| `fields` | Выбор полей | `?fields=id,name,email` |
| `filter[field]` | Фильтрация | `?filter[status]=active` |
| `include` | Включение связей | `?include=orders,profile` |
| `q`, `search` | Поиск | `?q=john` |

## Специальные endpoints

### Действия над ресурсами (RPC-style)

Иногда нужны действия, которые не укладываются в CRUD:

```
# Вариант 1: Действие как под-ресурс
POST /users/123/activate
POST /orders/456/cancel
POST /payments/789/refund

# Вариант 2: Через query параметр
POST /users/123?action=activate

# Вариант 3: Отдельный ресурс для действия
POST /user-activations
{
    "user_id": 123
}
```

### Batch операции

```
# Batch создание
POST /users/batch
[
    {"name": "User 1", "email": "user1@example.com"},
    {"name": "User 2", "email": "user2@example.com"}
]

# Batch удаление
DELETE /users?ids=1,2,3,4,5

# Или через POST
POST /users/batch-delete
{
    "ids": [1, 2, 3, 4, 5]
}
```

### Поиск и фильтрация

```
# Простой поиск
GET /products/search?q=iphone

# Расширенный поиск (через POST для сложных запросов)
POST /products/search
{
    "query": "iphone",
    "filters": {
        "category": ["electronics", "phones"],
        "price": {"min": 500, "max": 1500},
        "in_stock": true
    },
    "sort": [{"field": "price", "order": "asc"}]
}
```

### Агрегации и статистика

```
GET /orders/stats
GET /users/count
GET /products/123/reviews/summary
GET /dashboard/metrics?period=monthly
```

## Примеры реализации

### Python (FastAPI)

```python
from fastapi import FastAPI, Query, Path, HTTPException
from typing import Optional, List
from enum import Enum

app = FastAPI()

class SortOrder(str, Enum):
    asc = "asc"
    desc = "desc"

# Коллекция ресурсов с фильтрацией
@app.get("/api/v1/products")
async def get_products(
    category: Optional[str] = Query(None, description="Filter by category"),
    brand: Optional[str] = Query(None, description="Filter by brand"),
    price_min: Optional[float] = Query(None, ge=0, description="Minimum price"),
    price_max: Optional[float] = Query(None, ge=0, description="Maximum price"),
    in_stock: Optional[bool] = Query(None, description="Filter by availability"),
    sort: Optional[str] = Query("created_at", description="Sort field"),
    order: SortOrder = Query(SortOrder.desc, description="Sort order"),
    page: int = Query(1, ge=1, description="Page number"),
    limit: int = Query(20, ge=1, le=100, description="Items per page")
):
    """
    Получить список продуктов с фильтрацией и пагинацией.

    Примеры:
    - /api/v1/products?category=electronics&sort=price&order=asc
    - /api/v1/products?price_min=100&price_max=500&in_stock=true
    """
    # Логика фильтрации и выборки
    return {
        "data": [],
        "pagination": {
            "page": page,
            "limit": limit,
            "total": 0
        }
    }

# Вложенные ресурсы
@app.get("/api/v1/users/{user_id}/orders")
async def get_user_orders(
    user_id: int = Path(..., description="User ID", ge=1),
    status: Optional[str] = Query(None, description="Order status"),
    page: int = Query(1, ge=1),
    limit: int = Query(20, ge=1, le=100)
):
    """Получить заказы конкретного пользователя."""
    return {"user_id": user_id, "orders": []}

@app.get("/api/v1/users/{user_id}/orders/{order_id}")
async def get_user_order(
    user_id: int = Path(..., ge=1),
    order_id: int = Path(..., ge=1)
):
    """Получить конкретный заказ пользователя."""
    return {"user_id": user_id, "order_id": order_id}

# Действие над ресурсом
@app.post("/api/v1/orders/{order_id}/cancel")
async def cancel_order(order_id: int = Path(..., ge=1)):
    """Отменить заказ."""
    return {"order_id": order_id, "status": "cancelled"}

# Поиск
@app.get("/api/v1/products/search")
async def search_products(
    q: str = Query(..., min_length=2, description="Search query"),
    category: Optional[str] = None,
    page: int = Query(1, ge=1),
    limit: int = Query(20, ge=1, le=100)
):
    """Поиск продуктов по ключевому слову."""
    return {"query": q, "results": []}

# Batch операции
@app.post("/api/v1/products/batch")
async def create_products_batch(products: List[dict]):
    """Создать несколько продуктов одним запросом."""
    return {"created": len(products)}

@app.delete("/api/v1/products")
async def delete_products_batch(
    ids: str = Query(..., description="Comma-separated product IDs")
):
    """Удалить несколько продуктов."""
    id_list = [int(id) for id in ids.split(",")]
    return {"deleted": id_list}
```

### Node.js (Express)

```javascript
const express = require('express');
const router = express.Router();

// Middleware для парсинга query параметров
const parseFilters = (req, res, next) => {
    req.filters = {
        page: parseInt(req.query.page) || 1,
        limit: Math.min(parseInt(req.query.limit) || 20, 100),
        sort: req.query.sort || 'created_at',
        order: req.query.order || 'desc'
    };
    next();
};

// GET /api/v1/products
router.get('/products', parseFilters, async (req, res) => {
    const { category, brand, price_min, price_max, in_stock } = req.query;
    const { page, limit, sort, order } = req.filters;

    // Построение запроса к БД
    const query = {};
    if (category) query.category = category;
    if (brand) query.brand = brand;
    if (price_min || price_max) {
        query.price = {};
        if (price_min) query.price.$gte = parseFloat(price_min);
        if (price_max) query.price.$lte = parseFloat(price_max);
    }
    if (in_stock !== undefined) query.in_stock = in_stock === 'true';

    // Выполнение запроса (pseudo-code)
    // const products = await Product.find(query)
    //     .sort({ [sort]: order === 'asc' ? 1 : -1 })
    //     .skip((page - 1) * limit)
    //     .limit(limit);

    res.json({
        data: [],
        pagination: { page, limit, total: 0 }
    });
});

// GET /api/v1/users/:userId/orders
router.get('/users/:userId/orders', parseFilters, async (req, res) => {
    const { userId } = req.params;
    const { status } = req.query;

    res.json({
        user_id: userId,
        orders: []
    });
});

// GET /api/v1/users/:userId/orders/:orderId
router.get('/users/:userId/orders/:orderId', async (req, res) => {
    const { userId, orderId } = req.params;

    res.json({
        user_id: userId,
        order_id: orderId
    });
});

// POST /api/v1/orders/:orderId/cancel
router.post('/orders/:orderId/cancel', async (req, res) => {
    const { orderId } = req.params;

    res.json({
        order_id: orderId,
        status: 'cancelled'
    });
});

// GET /api/v1/products/search
router.get('/products/search', parseFilters, async (req, res) => {
    const { q } = req.query;

    if (!q || q.length < 2) {
        return res.status(400).json({
            error: 'Search query must be at least 2 characters'
        });
    }

    res.json({
        query: q,
        results: []
    });
});

module.exports = router;
```

## Примеры из реальных API

### GitHub API

```
# Репозитории пользователя
GET /users/{username}/repos

# Конкретный репозиторий
GET /repos/{owner}/{repo}

# Issues репозитория
GET /repos/{owner}/{repo}/issues
GET /repos/{owner}/{repo}/issues/{issue_number}

# Комментарии к issue
GET /repos/{owner}/{repo}/issues/{issue_number}/comments

# Поиск
GET /search/repositories?q=language:python+stars:>1000
```

### Stripe API

```
# Клиенты
GET    /v1/customers
POST   /v1/customers
GET    /v1/customers/{id}
POST   /v1/customers/{id}   # Обновление (не PUT!)
DELETE /v1/customers/{id}

# Платежи
GET  /v1/charges
POST /v1/charges
GET  /v1/charges/{id}

# Возвраты (отдельный ресурс)
POST /v1/refunds
GET  /v1/refunds/{id}
```

### Twilio API

```
# Аккаунты
GET /2010-04-01/Accounts/{AccountSid}

# Сообщения
GET  /2010-04-01/Accounts/{AccountSid}/Messages
POST /2010-04-01/Accounts/{AccountSid}/Messages
GET  /2010-04-01/Accounts/{AccountSid}/Messages/{MessageSid}
```

## Best Practices

### Что делать:

1. **Будьте консистентны:**
   - Выберите конвенцию и следуйте ей везде
   - `/users`, `/products`, `/orders` (не `/user`, `/products`, `/order-list`)

2. **Используйте понятные имена:**
   - `/shipping-addresses` вместо `/sa` или `/ship-addr`

3. **Документируйте формат идентификаторов:**
   - Укажите, используете ли вы числовые ID, UUID, slug

4. **Ограничивайте глубину вложенности:**
   - Максимум 2-3 уровня: `/users/{id}/orders/{id}`

5. **Используйте query параметры для опциональных данных:**
   - Фильтры, сортировка, пагинация, расширение ответа

### Типичные ошибки:

1. **Глаголы в URI:**
   ```
   # Плохо
   GET /getProductById/123
   POST /createNewUser

   # Хорошо
   GET /products/123
   POST /users
   ```

2. **Несогласованность:**
   ```
   # Плохо — микс стилей
   /user-profiles
   /productCategories
   /Order_Items

   # Хорошо — единый стиль
   /user-profiles
   /product-categories
   /order-items
   ```

3. **Слишком глубокая вложенность:**
   ```
   # Плохо
   /companies/1/departments/2/teams/3/employees/4/tasks/5

   # Лучше
   /tasks/5
   /tasks?employee_id=4&team_id=3
   ```

4. **Расширения файлов:**
   ```
   # Плохо
   /users.json
   /products.xml

   # Хорошо — формат через заголовок Accept
   GET /users
   Accept: application/json
   ```

5. **Trailing slash непоследовательность:**
   ```
   # Выберите один вариант и используйте везде
   /users     # без trailing slash
   /users/    # с trailing slash
   ```

## Заключение

Хорошо спроектированные URI делают API интуитивно понятным и простым в использовании. Следуйте принципам:

- Существительные, не глаголы
- Множественное число для коллекций
- Иерархия для связанных ресурсов
- Query параметры для фильтрации
- Консистентность во всём API

Правильный дизайн URI — это инвестиция, которая окупается удобством использования и поддержки API.
