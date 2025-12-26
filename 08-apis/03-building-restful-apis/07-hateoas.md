# HATEOAS (Hypermedia as the Engine of Application State)

## Введение

**HATEOAS** — это ограничение архитектуры REST, согласно которому клиент взаимодействует с сервером полностью через гипермедиа, динамически предоставляемую сервером. Это высший уровень зрелости REST API (Level 3 по Richardson Maturity Model).

**Ключевая идея:** Клиенту не нужно знать структуру URL заранее. Всё, что ему нужно — это начальная точка входа. Дальнейшая навигация происходит через ссылки в ответах сервера.

## Простой пример

### Без HATEOAS:

```json
{
    "id": 123,
    "name": "John Doe",
    "email": "john@example.com",
    "status": "active"
}
```

Клиент должен **сам знать**:
- Как обновить пользователя? `PUT /users/123`
- Как удалить? `DELETE /users/123`
- Как получить заказы? `GET /users/123/orders`

### С HATEOAS:

```json
{
    "id": 123,
    "name": "John Doe",
    "email": "john@example.com",
    "status": "active",
    "_links": {
        "self": { "href": "/api/users/123" },
        "update": { "href": "/api/users/123", "method": "PUT" },
        "delete": { "href": "/api/users/123", "method": "DELETE" },
        "orders": { "href": "/api/users/123/orders" },
        "deactivate": { "href": "/api/users/123/deactivate", "method": "POST" }
    }
}
```

Клиент **получает все возможные действия** из ответа.

## Преимущества HATEOAS

### 1. Самодокументируемость
Ответ API сам говорит, что можно делать дальше.

### 2. Слабая связанность
Клиент не зависит от жёстко закодированных URL.

### 3. Эволюция API
Можно менять URL без ломки клиентов.

### 4. Контекстные действия
Разные действия доступны в разных состояниях.

```json
// Заказ в статусе "pending"
{
    "id": 456,
    "status": "pending",
    "_links": {
        "self": { "href": "/api/orders/456" },
        "cancel": { "href": "/api/orders/456/cancel", "method": "POST" },
        "pay": { "href": "/api/orders/456/pay", "method": "POST" }
    }
}

// Заказ в статусе "shipped"
{
    "id": 456,
    "status": "shipped",
    "_links": {
        "self": { "href": "/api/orders/456" },
        "track": { "href": "/api/orders/456/tracking" }
        // "cancel" недоступен — заказ уже отправлен!
    }
}
```

## Форматы гипермедиа

### 1. HAL (Hypertext Application Language)

Самый популярный формат для HATEOAS.

```json
{
    "id": 123,
    "name": "John Doe",
    "_links": {
        "self": { "href": "/api/users/123" },
        "orders": { "href": "/api/users/123/orders" }
    },
    "_embedded": {
        "latestOrder": {
            "id": 456,
            "total": 99.99,
            "_links": {
                "self": { "href": "/api/orders/456" }
            }
        }
    }
}
```

**Content-Type:** `application/hal+json`

### 2. JSON:API

Стандарт с детальной спецификацией.

```json
{
    "data": {
        "type": "users",
        "id": "123",
        "attributes": {
            "name": "John Doe",
            "email": "john@example.com"
        },
        "relationships": {
            "orders": {
                "links": {
                    "self": "/api/users/123/relationships/orders",
                    "related": "/api/users/123/orders"
                }
            }
        },
        "links": {
            "self": "/api/users/123"
        }
    },
    "links": {
        "self": "/api/users/123"
    }
}
```

**Content-Type:** `application/vnd.api+json`

### 3. JSON-LD (Linked Data)

Семантический формат с контекстом.

```json
{
    "@context": "https://schema.org/",
    "@type": "Person",
    "@id": "/api/users/123",
    "name": "John Doe",
    "email": "john@example.com",
    "knows": {
        "@id": "/api/users/456"
    }
}
```

### 4. Siren

Формат с actions для описания операций.

```json
{
    "class": ["order"],
    "properties": {
        "id": 456,
        "status": "pending",
        "total": 99.99
    },
    "entities": [
        {
            "class": ["items", "collection"],
            "rel": ["items"],
            "href": "/api/orders/456/items"
        }
    ],
    "actions": [
        {
            "name": "cancel-order",
            "title": "Cancel Order",
            "method": "POST",
            "href": "/api/orders/456/cancel",
            "type": "application/json",
            "fields": [
                { "name": "reason", "type": "text" }
            ]
        }
    ],
    "links": [
        { "rel": ["self"], "href": "/api/orders/456" }
    ]
}
```

## Реализация

### Python (FastAPI) — HAL формат

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Dict, List, Optional, Any
from enum import Enum

app = FastAPI()

class OrderStatus(str, Enum):
    PENDING = "pending"
    PAID = "paid"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class Link(BaseModel):
    href: str
    method: str = "GET"
    title: Optional[str] = None

class HALResource(BaseModel):
    """Базовый класс для HAL ресурсов."""
    _links: Dict[str, Link] = {}
    _embedded: Optional[Dict[str, Any]] = None

    class Config:
        extra = "allow"

def build_user_links(user_id: int, user_status: str) -> Dict[str, Link]:
    """Строит ссылки для пользователя в зависимости от статуса."""
    links = {
        "self": Link(href=f"/api/users/{user_id}"),
        "orders": Link(href=f"/api/users/{user_id}/orders"),
        "update": Link(href=f"/api/users/{user_id}", method="PUT"),
    }

    if user_status == "active":
        links["deactivate"] = Link(
            href=f"/api/users/{user_id}/deactivate",
            method="POST",
            title="Deactivate user account"
        )
    elif user_status == "inactive":
        links["activate"] = Link(
            href=f"/api/users/{user_id}/activate",
            method="POST",
            title="Activate user account"
        )

    return links

def build_order_links(order_id: int, status: OrderStatus) -> Dict[str, Link]:
    """Строит ссылки для заказа в зависимости от статуса."""
    links = {
        "self": Link(href=f"/api/orders/{order_id}"),
        "items": Link(href=f"/api/orders/{order_id}/items"),
    }

    # Контекстные действия в зависимости от статуса
    if status == OrderStatus.PENDING:
        links["pay"] = Link(
            href=f"/api/orders/{order_id}/pay",
            method="POST",
            title="Pay for this order"
        )
        links["cancel"] = Link(
            href=f"/api/orders/{order_id}/cancel",
            method="POST",
            title="Cancel this order"
        )

    elif status == OrderStatus.PAID:
        links["refund"] = Link(
            href=f"/api/orders/{order_id}/refund",
            method="POST",
            title="Request a refund"
        )

    elif status == OrderStatus.SHIPPED:
        links["track"] = Link(
            href=f"/api/orders/{order_id}/tracking",
            title="Track shipment"
        )

    return links

# Имитация БД
users_db = {
    123: {"id": 123, "name": "John Doe", "email": "john@example.com", "status": "active"}
}
orders_db = {
    456: {"id": 456, "user_id": 123, "total": 99.99, "status": OrderStatus.PENDING}
}

@app.get("/api/users/{user_id}")
async def get_user(user_id: int):
    user = users_db.get(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")

    return {
        **user,
        "_links": build_user_links(user_id, user["status"])
    }

@app.get("/api/orders/{order_id}")
async def get_order(order_id: int):
    order = orders_db.get(order_id)
    if not order:
        raise HTTPException(status_code=404, detail="Order not found")

    return {
        **order,
        "_links": build_order_links(order_id, order["status"])
    }

@app.post("/api/orders/{order_id}/pay")
async def pay_order(order_id: int):
    order = orders_db.get(order_id)
    if not order:
        raise HTTPException(status_code=404, detail="Order not found")

    if order["status"] != OrderStatus.PENDING:
        raise HTTPException(
            status_code=422,
            detail=f"Cannot pay order in status: {order['status']}"
        )

    order["status"] = OrderStatus.PAID

    return {
        **order,
        "_links": build_order_links(order_id, order["status"])
    }

@app.get("/api/users/{user_id}/orders")
async def get_user_orders(user_id: int):
    user_orders = [o for o in orders_db.values() if o["user_id"] == user_id]

    return {
        "_links": {
            "self": {"href": f"/api/users/{user_id}/orders"},
            "user": {"href": f"/api/users/{user_id}"}
        },
        "_embedded": {
            "orders": [
                {
                    **order,
                    "_links": build_order_links(order["id"], order["status"])
                }
                for order in user_orders
            ]
        },
        "count": len(user_orders)
    }
```

### Node.js (Express) — HAL формат

```javascript
const express = require('express');
const app = express();
app.use(express.json());

// HAL Helper
class HALResource {
    constructor(data) {
        Object.assign(this, data);
        this._links = {};
        this._embedded = {};
    }

    addLink(rel, href, options = {}) {
        this._links[rel] = { href, ...options };
        return this;
    }

    embed(rel, resources) {
        this._embedded[rel] = resources;
        return this;
    }

    toJSON() {
        const result = { ...this };
        if (Object.keys(this._embedded).length === 0) {
            delete result._embedded;
        }
        return result;
    }
}

// Link builders
function buildOrderLinks(order) {
    const links = {
        self: { href: `/api/orders/${order.id}` },
        items: { href: `/api/orders/${order.id}/items` }
    };

    switch (order.status) {
        case 'pending':
            links.pay = {
                href: `/api/orders/${order.id}/pay`,
                method: 'POST',
                title: 'Pay for this order'
            };
            links.cancel = {
                href: `/api/orders/${order.id}/cancel`,
                method: 'POST',
                title: 'Cancel this order'
            };
            break;
        case 'paid':
            links.refund = {
                href: `/api/orders/${order.id}/refund`,
                method: 'POST',
                title: 'Request a refund'
            };
            break;
        case 'shipped':
            links.track = {
                href: `/api/orders/${order.id}/tracking`,
                title: 'Track shipment'
            };
            break;
    }

    return links;
}

// Данные
const orders = new Map([
    [1, { id: 1, product: 'Laptop', total: 999.99, status: 'pending' }],
    [2, { id: 2, product: 'Mouse', total: 29.99, status: 'shipped' }]
]);

// Routes
app.get('/api/orders/:id', (req, res) => {
    const order = orders.get(parseInt(req.params.id));

    if (!order) {
        return res.status(404).json({ error: 'Order not found' });
    }

    res.json({
        ...order,
        _links: buildOrderLinks(order)
    });
});

app.post('/api/orders/:id/pay', (req, res) => {
    const order = orders.get(parseInt(req.params.id));

    if (!order) {
        return res.status(404).json({ error: 'Order not found' });
    }

    if (order.status !== 'pending') {
        return res.status(422).json({
            error: `Cannot pay order in status: ${order.status}`
        });
    }

    order.status = 'paid';

    res.json({
        ...order,
        _links: buildOrderLinks(order)
    });
});

app.get('/api/orders', (req, res) => {
    const orderList = Array.from(orders.values()).map(order => ({
        ...order,
        _links: buildOrderLinks(order)
    }));

    res.json({
        _links: {
            self: { href: '/api/orders' },
            create: { href: '/api/orders', method: 'POST' }
        },
        _embedded: {
            orders: orderList
        },
        count: orderList.length
    });
});

// Entry point
app.get('/api', (req, res) => {
    res.json({
        _links: {
            self: { href: '/api' },
            orders: { href: '/api/orders' },
            users: { href: '/api/users' },
            products: { href: '/api/products' }
        },
        version: '1.0.0'
    });
});

app.listen(3000);
```

## Пагинация с HATEOAS

```json
{
    "_links": {
        "self": { "href": "/api/orders?page=2&limit=20" },
        "first": { "href": "/api/orders?page=1&limit=20" },
        "prev": { "href": "/api/orders?page=1&limit=20" },
        "next": { "href": "/api/orders?page=3&limit=20" },
        "last": { "href": "/api/orders?page=10&limit=20" }
    },
    "_embedded": {
        "orders": [
            { "id": 21, "total": 50.00, "_links": { "self": { "href": "/api/orders/21" } } },
            { "id": 22, "total": 75.00, "_links": { "self": { "href": "/api/orders/22" } } }
        ]
    },
    "page": 2,
    "limit": 20,
    "total": 200
}
```

```python
@app.get("/api/orders")
async def list_orders(page: int = 1, limit: int = 20):
    total = len(orders_db)
    total_pages = (total + limit - 1) // limit

    # Пагинация
    start = (page - 1) * limit
    end = start + limit
    page_orders = list(orders_db.values())[start:end]

    links = {
        "self": {"href": f"/api/orders?page={page}&limit={limit}"},
        "first": {"href": f"/api/orders?page=1&limit={limit}"},
        "last": {"href": f"/api/orders?page={total_pages}&limit={limit}"}
    }

    if page > 1:
        links["prev"] = {"href": f"/api/orders?page={page-1}&limit={limit}"}
    if page < total_pages:
        links["next"] = {"href": f"/api/orders?page={page+1}&limit={limit}"}

    return {
        "_links": links,
        "_embedded": {
            "orders": [
                {
                    **order,
                    "_links": build_order_links(order["id"], order["status"])
                }
                for order in page_orders
            ]
        },
        "page": page,
        "limit": limit,
        "total": total
    }
```

## Примеры из реальных API

### GitHub API

```json
{
    "id": 1,
    "name": "Hello-World",
    "full_name": "octocat/Hello-World",
    "html_url": "https://github.com/octocat/Hello-World",
    "url": "https://api.github.com/repos/octocat/Hello-World",
    "commits_url": "https://api.github.com/repos/octocat/Hello-World/commits{/sha}",
    "issues_url": "https://api.github.com/repos/octocat/Hello-World/issues{/number}",
    "pulls_url": "https://api.github.com/repos/octocat/Hello-World/pulls{/number}"
}
```

### PayPal API

```json
{
    "id": "WH-123",
    "status": "PENDING",
    "amount": {
        "value": "100.00",
        "currency": "USD"
    },
    "links": [
        {
            "href": "https://api.paypal.com/v1/payments/payment/WH-123",
            "rel": "self",
            "method": "GET"
        },
        {
            "href": "https://api.paypal.com/v1/payments/payment/WH-123/execute",
            "rel": "execute",
            "method": "POST"
        }
    ]
}
```

### Amazon API Gateway

```json
{
    "id": "abc123",
    "name": "MyAPI",
    "_links": {
        "self": { "href": "/restapis/abc123" },
        "resources": { "href": "/restapis/abc123/resources" },
        "stages": { "href": "/restapis/abc123/stages" },
        "deployments": { "href": "/restapis/abc123/deployments" }
    }
}
```

## Клиентская работа с HATEOAS

```javascript
// Клиент не знает URL заранее — он следует ссылкам

async function processOrder(entryPoint) {
    // 1. Получаем точку входа
    const api = await fetch(entryPoint).then(r => r.json());

    // 2. Переходим к заказам
    const ordersUrl = api._links.orders.href;
    const orders = await fetch(ordersUrl).then(r => r.json());

    // 3. Находим заказ для оплаты
    const pendingOrder = orders._embedded.orders.find(
        o => o._links.pay
    );

    if (pendingOrder) {
        // 4. Оплачиваем, используя ссылку из ответа
        const payLink = pendingOrder._links.pay;
        const result = await fetch(payLink.href, {
            method: payLink.method || 'POST'
        }).then(r => r.json());

        console.log('Order paid:', result);

        // 5. Теперь у нас другие доступные действия
        if (result._links.refund) {
            console.log('Refund available at:', result._links.refund.href);
        }
    }
}

processOrder('https://api.example.com');
```

## Когда НЕ использовать HATEOAS

1. **Простые CRUD API** — overhead не оправдан
2. **Внутренние микросервисы** — сервисы знают друг друга
3. **Высоконагруженные API** — лишние данные в ответах
4. **Мобильные приложения** — жёсткие контракты предпочтительнее

## Best Practices

### Что делать:

1. **Используйте стандартные форматы:**
   - HAL для простых случаев
   - JSON:API для сложных API

2. **Включайте `self` ссылку всегда:**
   ```json
   "_links": { "self": { "href": "/api/resource/123" } }
   ```

3. **Контекстные действия:**
   - Показывайте только доступные в текущем состоянии

4. **Шаблоны URL (URI Templates):**
   ```json
   "search": { "href": "/api/users{?name,email}", "templated": true }
   ```

5. **Описывайте методы:**
   ```json
   "delete": { "href": "/api/users/123", "method": "DELETE" }
   ```

### Типичные ошибки:

1. **Избыточные ссылки:**
   - Не нужно включать все возможные ссылки

2. **Хардкод URL в клиенте:**
   - Это нарушает принцип HATEOAS

3. **Отсутствие документации:**
   - Даже с HATEOAS нужно объяснять семантику ссылок

## Заключение

HATEOAS — это мощный принцип для создания гибких и самодокументирующихся API:

- **Динамические ссылки** позволяют менять API без ломки клиентов
- **Контекстные действия** показывают только то, что доступно
- **Стандартные форматы** (HAL, JSON:API) упрощают реализацию

Однако это усложняет разработку и не всегда необходимо. Используйте HATEOAS для публичных API, где слабая связанность критически важна.
