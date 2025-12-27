# Client-Server Architecture

## Что такое клиент-серверная архитектура?

Клиент-серверная архитектура (Client-Server Architecture) — это распределённая архитектурная модель, в которой задачи распределяются между поставщиками услуг (серверами) и потребителями услуг (клиентами). Клиент отправляет запросы серверу, а сервер обрабатывает эти запросы и возвращает результаты.

> "Клиент-серверная модель — это фундамент современного интернета и большинства корпоративных приложений."

## Основные компоненты

```
┌─────────────────┐           ┌─────────────────┐
│     CLIENT      │           │     SERVER      │
│                 │           │                 │
│  ┌───────────┐  │  Request  │  ┌───────────┐  │
│  │    UI     │  │ ───────►  │  │  Business │  │
│  └───────────┘  │           │  │   Logic   │  │
│  ┌───────────┐  │  Response │  └───────────┘  │
│  │   Logic   │  │ ◄───────  │  ┌───────────┐  │
│  └───────────┘  │           │  │  Database │  │
│                 │           │  └───────────┘  │
└─────────────────┘           └─────────────────┘
        │                             │
        └──────────── Network ────────┘
```

### Клиент (Client)

Клиент — это программа или устройство, которое:
- Инициирует запросы к серверу
- Получает и обрабатывает ответы
- Предоставляет пользовательский интерфейс
- Может выполнять локальную обработку данных

### Сервер (Server)

Сервер — это программа или устройство, которое:
- Ожидает входящие запросы от клиентов
- Обрабатывает запросы
- Управляет общими ресурсами
- Возвращает результаты клиентам

## Типы клиент-серверной архитектуры

### 2-Tier Architecture (Двухуровневая)

```
┌──────────┐         ┌──────────┐
│  Client  │ ◄─────► │  Server  │
│  (UI +   │         │  (DB +   │
│  Logic)  │         │  Logic)  │
└──────────┘         └──────────┘
```

```python
# Пример: Клиент напрямую работает с базой данных

# client.py
import psycopg2

class DatabaseClient:
    """Толстый клиент с бизнес-логикой."""

    def __init__(self, connection_string: str):
        self.conn = psycopg2.connect(connection_string)

    def get_user_orders(self, user_id: int):
        """Клиент сам формирует SQL-запросы."""
        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT o.id, o.total_amount, o.status
            FROM orders o
            WHERE o.user_id = %s
            ORDER BY o.created_at DESC
        """, (user_id,))
        return cursor.fetchall()

    def calculate_order_total(self, order_id: int):
        """Бизнес-логика на стороне клиента."""
        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT oi.quantity, p.price
            FROM order_items oi
            JOIN products p ON oi.product_id = p.id
            WHERE oi.order_id = %s
        """, (order_id,))

        items = cursor.fetchall()
        total = sum(qty * price for qty, price in items)
        return total
```

### 3-Tier Architecture (Трёхуровневая)

```
┌──────────┐     ┌──────────────┐     ┌──────────┐
│  Client  │ ──► │  Application │ ──► │ Database │
│  (UI)    │ ◄── │    Server    │ ◄── │  Server  │
└──────────┘     └──────────────┘     └──────────┘
Presentation      Business Logic      Data Storage
    Tier              Tier               Tier
```

```python
# Современный подход: разделение на слои

# === Presentation Tier (Client) ===
# client.py
import httpx

class APIClient:
    """Тонкий клиент - только UI и HTTP запросы."""

    def __init__(self, base_url: str):
        self.base_url = base_url
        self.client = httpx.Client()

    def get_user_orders(self, user_id: int):
        response = self.client.get(
            f"{self.base_url}/users/{user_id}/orders"
        )
        return response.json()

    def create_order(self, user_id: int, items: list):
        response = self.client.post(
            f"{self.base_url}/orders",
            json={"user_id": user_id, "items": items}
        )
        return response.json()


# === Business Logic Tier (Application Server) ===
# server.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List

app = FastAPI()

class OrderItem(BaseModel):
    product_id: int
    quantity: int

class OrderCreate(BaseModel):
    user_id: int
    items: List[OrderItem]

@app.post("/orders")
async def create_order(order_data: OrderCreate):
    """Бизнес-логика на сервере приложений."""
    # Валидация
    user = await user_repository.get_by_id(order_data.user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")

    # Проверка бизнес-правил
    if not user.is_active:
        raise HTTPException(status_code=400, detail="User is inactive")

    # Расчёт суммы
    total = 0
    for item in order_data.items:
        product = await product_repository.get_by_id(item.product_id)
        if product.stock < item.quantity:
            raise HTTPException(
                status_code=400,
                detail=f"Insufficient stock for {product.name}"
            )
        total += product.price * item.quantity

    # Создание заказа
    order = await order_repository.create({
        "user_id": order_data.user_id,
        "total_amount": total,
        "status": "pending"
    })

    return {"order_id": order.id, "total": total}


# === Data Tier (Database) ===
# repository.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

class OrderRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def create(self, data: dict):
        order = Order(**data)
        self.session.add(order)
        await self.session.commit()
        return order

    async def get_by_user_id(self, user_id: int):
        result = await self.session.execute(
            select(Order).where(Order.user_id == user_id)
        )
        return result.scalars().all()
```

### N-Tier Architecture (Многоуровневая)

```
┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
│  Mobile   │  │    Web    │  │   API     │  │  Business │  │ Database  │
│   App     │─►│  Browser  │─►│  Gateway  │─►│  Services │─►│  Cluster  │
└───────────┘  └───────────┘  └───────────┘  └───────────┘  └───────────┘
```

## Модели взаимодействия клиент-сервер

### Request-Response (Запрос-Ответ)

```python
# Синхронная модель Request-Response

# server.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/api/users/{user_id}")
async def get_user(user_id: int):
    """Сервер обрабатывает запрос и возвращает ответ."""
    user = await fetch_user_from_db(user_id)
    return {"id": user.id, "name": user.name, "email": user.email}


# client.py
import requests

def get_user(user_id: int):
    """Клиент отправляет запрос и ждёт ответа."""
    response = requests.get(f"http://api.example.com/users/{user_id}")
    if response.status_code == 200:
        return response.json()
    raise Exception(f"Error: {response.status_code}")
```

### Push Model (Серверные уведомления)

```python
# Server-Sent Events (SSE) - сервер отправляет данные клиенту

# server.py
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio

app = FastAPI()

async def event_generator():
    """Генератор событий для отправки клиенту."""
    while True:
        # Получаем новые данные
        data = await get_latest_updates()
        if data:
            yield f"data: {json.dumps(data)}\n\n"
        await asyncio.sleep(1)

@app.get("/events")
async def stream_events():
    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream"
    )


# client.py (JavaScript)
"""
const eventSource = new EventSource('/events');
eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    updateUI(data);
};
"""
```

### Full-Duplex (WebSocket)

```python
# WebSocket - двунаправленная связь

# server.py
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import List

app = FastAPI()

class ConnectionManager:
    def __init__(self):
        self.active_connections: List[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)

    async def broadcast(self, message: str):
        for connection in self.active_connections:
            await connection.send_text(message)

manager = ConnectionManager()

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: str):
    await manager.connect(websocket)
    try:
        while True:
            # Получаем сообщение от клиента
            data = await websocket.receive_text()

            # Обрабатываем и отправляем ответ
            response = process_message(data)
            await websocket.send_text(response)

            # Или рассылаем всем подключённым клиентам
            await manager.broadcast(f"Client {client_id}: {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)


# client.py
import websockets
import asyncio

async def client():
    async with websockets.connect("ws://localhost:8000/ws/client1") as ws:
        # Отправляем сообщение
        await ws.send("Hello, Server!")

        # Получаем ответ
        response = await ws.recv()
        print(f"Received: {response}")

asyncio.run(client())
```

## Типы клиентов

### Thin Client (Тонкий клиент)

```python
# Тонкий клиент - минимум логики, вся обработка на сервере

class ThinClient:
    """Клиент только отображает данные."""

    def __init__(self, server_url: str):
        self.server_url = server_url

    def render_page(self, page_url: str):
        """Запрашивает готовую HTML страницу с сервера."""
        response = requests.get(f"{self.server_url}{page_url}")
        return response.text  # Готовый HTML

    def submit_form(self, form_data: dict):
        """Отправляет данные формы на сервер."""
        response = requests.post(
            f"{self.server_url}/submit",
            data=form_data
        )
        # Сервер возвращает новую страницу
        return response.text
```

### Thick Client (Толстый клиент)

```python
# Толстый клиент - значительная часть логики на клиенте

class ThickClient:
    """Клиент с локальной бизнес-логикой."""

    def __init__(self, server_url: str):
        self.server_url = server_url
        self.local_cache = {}

    def get_products(self):
        """Получает данные и кэширует локально."""
        if "products" not in self.local_cache:
            response = requests.get(f"{self.server_url}/api/products")
            self.local_cache["products"] = response.json()
        return self.local_cache["products"]

    def calculate_cart_total(self, cart_items: list):
        """Расчёт выполняется локально."""
        products = self.get_products()
        total = 0
        for item in cart_items:
            product = next(
                p for p in products if p["id"] == item["product_id"]
            )
            total += product["price"] * item["quantity"]

            # Применяем скидки локально
            if item["quantity"] >= 10:
                total *= 0.9  # 10% скидка

        return total

    def validate_order(self, order_data: dict):
        """Валидация на стороне клиента."""
        errors = []
        if not order_data.get("email"):
            errors.append("Email is required")
        if not order_data.get("items"):
            errors.append("Cart is empty")
        return errors

    def submit_order(self, order_data: dict):
        """Отправка только валидных данных."""
        errors = self.validate_order(order_data)
        if errors:
            return {"success": False, "errors": errors}

        response = requests.post(
            f"{self.server_url}/api/orders",
            json=order_data
        )
        return response.json()
```

### Rich Client (Богатый клиент)

```python
# Rich Client - SPA приложение с полноценным состоянием

class RichClient:
    """Клиент типа SPA с локальным состоянием."""

    def __init__(self, api_url: str):
        self.api_url = api_url
        self.state = {
            "user": None,
            "products": [],
            "cart": [],
            "orders": []
        }
        self.subscribers = []

    def subscribe(self, callback):
        """Подписка на изменения состояния."""
        self.subscribers.append(callback)

    def notify(self):
        """Уведомление подписчиков об изменениях."""
        for callback in self.subscribers:
            callback(self.state)

    async def login(self, email: str, password: str):
        """Аутентификация и загрузка данных пользователя."""
        response = await self._post("/auth/login", {
            "email": email,
            "password": password
        })

        if response.get("token"):
            self.state["user"] = response["user"]
            self.token = response["token"]

            # Загружаем связанные данные
            await self._load_user_data()
            self.notify()

    async def _load_user_data(self):
        """Предзагрузка данных для оффлайн работы."""
        products = await self._get("/products")
        orders = await self._get("/orders")

        self.state["products"] = products
        self.state["orders"] = orders

    def add_to_cart(self, product_id: int, quantity: int):
        """Локальная операция без обращения к серверу."""
        product = next(
            p for p in self.state["products"]
            if p["id"] == product_id
        )

        self.state["cart"].append({
            "product": product,
            "quantity": quantity
        })
        self.notify()

    async def checkout(self):
        """Синхронизация корзины с сервером."""
        order_data = {
            "items": [
                {"product_id": item["product"]["id"], "quantity": item["quantity"]}
                for item in self.state["cart"]
            ]
        }

        response = await self._post("/orders", order_data)

        if response.get("success"):
            self.state["cart"] = []
            self.state["orders"].append(response["order"])
            self.notify()

        return response
```

## Примеры реализации

### REST API Server

```python
# Полноценный REST API сервер

from fastapi import FastAPI, HTTPException, Depends, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from pydantic import BaseModel
from typing import List, Optional
from datetime import datetime
import jwt

app = FastAPI(title="E-Commerce API")
security = HTTPBearer()

# === Модели ===

class ProductResponse(BaseModel):
    id: int
    name: str
    price: float
    stock: int
    category: str

class OrderItemCreate(BaseModel):
    product_id: int
    quantity: int

class OrderCreate(BaseModel):
    items: List[OrderItemCreate]
    shipping_address: str

class OrderResponse(BaseModel):
    id: int
    total_amount: float
    status: str
    created_at: datetime
    items: List[dict]

# === Аутентификация ===

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security)
):
    """Извлечение пользователя из JWT токена."""
    try:
        payload = jwt.decode(
            credentials.credentials,
            SECRET_KEY,
            algorithms=["HS256"]
        )
        user_id = payload.get("user_id")
        if user_id is None:
            raise HTTPException(status_code=401)

        user = await user_repository.get_by_id(user_id)
        if user is None:
            raise HTTPException(status_code=401)

        return user
    except jwt.PyJWTError:
        raise HTTPException(status_code=401)

# === Эндпоинты ===

@app.get("/products", response_model=List[ProductResponse])
async def list_products(
    category: Optional[str] = None,
    min_price: Optional[float] = None,
    max_price: Optional[float] = None,
    skip: int = 0,
    limit: int = 20
):
    """Получение списка продуктов с фильтрацией."""
    filters = {}
    if category:
        filters["category"] = category
    if min_price:
        filters["min_price"] = min_price
    if max_price:
        filters["max_price"] = max_price

    products = await product_repository.get_filtered(
        filters=filters,
        skip=skip,
        limit=limit
    )
    return products

@app.get("/products/{product_id}", response_model=ProductResponse)
async def get_product(product_id: int):
    """Получение конкретного продукта."""
    product = await product_repository.get_by_id(product_id)
    if not product:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Product not found"
        )
    return product

@app.post("/orders", response_model=OrderResponse, status_code=status.HTTP_201_CREATED)
async def create_order(
    order_data: OrderCreate,
    current_user = Depends(get_current_user)
):
    """Создание заказа."""
    # Проверка товаров и расчёт суммы
    total_amount = 0
    order_items = []

    for item in order_data.items:
        product = await product_repository.get_by_id(item.product_id)
        if not product:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail=f"Product {item.product_id} not found"
            )

        if product.stock < item.quantity:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail=f"Insufficient stock for {product.name}"
            )

        item_total = product.price * item.quantity
        total_amount += item_total

        order_items.append({
            "product_id": product.id,
            "product_name": product.name,
            "quantity": item.quantity,
            "price": product.price,
            "total": item_total
        })

    # Создание заказа
    order = await order_repository.create({
        "user_id": current_user.id,
        "total_amount": total_amount,
        "status": "pending",
        "shipping_address": order_data.shipping_address,
        "items": order_items
    })

    # Обновление остатков
    for item in order_data.items:
        await product_repository.decrease_stock(
            item.product_id,
            item.quantity
        )

    return order

@app.get("/orders", response_model=List[OrderResponse])
async def list_orders(
    current_user = Depends(get_current_user),
    status: Optional[str] = None
):
    """Получение списка заказов пользователя."""
    filters = {"user_id": current_user.id}
    if status:
        filters["status"] = status

    orders = await order_repository.get_filtered(filters)
    return orders

@app.get("/orders/{order_id}", response_model=OrderResponse)
async def get_order(
    order_id: int,
    current_user = Depends(get_current_user)
):
    """Получение конкретного заказа."""
    order = await order_repository.get_by_id(order_id)

    if not order:
        raise HTTPException(status_code=404, detail="Order not found")

    if order.user_id != current_user.id:
        raise HTTPException(status_code=403, detail="Access denied")

    return order
```

### Клиент с обработкой ошибок

```python
# Надёжный HTTP клиент с обработкой ошибок

import httpx
from typing import Optional, Dict, Any
from dataclasses import dataclass
import time

@dataclass
class APIResponse:
    success: bool
    data: Optional[Dict[str, Any]] = None
    error: Optional[str] = None
    status_code: int = 0

class ResilientAPIClient:
    """API клиент с повторными попытками и обработкой ошибок."""

    def __init__(
        self,
        base_url: str,
        timeout: float = 30.0,
        max_retries: int = 3,
        retry_delay: float = 1.0
    ):
        self.base_url = base_url
        self.timeout = timeout
        self.max_retries = max_retries
        self.retry_delay = retry_delay
        self.client = httpx.Client(timeout=timeout)
        self.token: Optional[str] = None

    def set_token(self, token: str):
        """Установка токена аутентификации."""
        self.token = token

    def _get_headers(self) -> Dict[str, str]:
        """Формирование заголовков запроса."""
        headers = {"Content-Type": "application/json"}
        if self.token:
            headers["Authorization"] = f"Bearer {self.token}"
        return headers

    def _make_request(
        self,
        method: str,
        endpoint: str,
        **kwargs
    ) -> APIResponse:
        """Выполнение HTTP запроса с повторными попытками."""
        url = f"{self.base_url}{endpoint}"
        headers = self._get_headers()

        for attempt in range(self.max_retries):
            try:
                response = self.client.request(
                    method,
                    url,
                    headers=headers,
                    **kwargs
                )

                if response.status_code >= 500:
                    # Ошибка сервера - повторяем
                    if attempt < self.max_retries - 1:
                        time.sleep(self.retry_delay * (attempt + 1))
                        continue

                return APIResponse(
                    success=200 <= response.status_code < 300,
                    data=response.json() if response.content else None,
                    status_code=response.status_code
                )

            except httpx.TimeoutException:
                if attempt < self.max_retries - 1:
                    time.sleep(self.retry_delay * (attempt + 1))
                    continue
                return APIResponse(
                    success=False,
                    error="Request timeout",
                    status_code=0
                )

            except httpx.ConnectError:
                return APIResponse(
                    success=False,
                    error="Connection error",
                    status_code=0
                )

        return APIResponse(
            success=False,
            error="Max retries exceeded",
            status_code=0
        )

    def get(self, endpoint: str, params: Optional[Dict] = None) -> APIResponse:
        return self._make_request("GET", endpoint, params=params)

    def post(self, endpoint: str, data: Dict) -> APIResponse:
        return self._make_request("POST", endpoint, json=data)

    def put(self, endpoint: str, data: Dict) -> APIResponse:
        return self._make_request("PUT", endpoint, json=data)

    def delete(self, endpoint: str) -> APIResponse:
        return self._make_request("DELETE", endpoint)


# Использование
client = ResilientAPIClient("https://api.example.com")

# Аутентификация
login_response = client.post("/auth/login", {
    "email": "user@example.com",
    "password": "password123"
})

if login_response.success:
    client.set_token(login_response.data["token"])

    # Получение данных
    products = client.get("/products", params={"category": "electronics"})
    if products.success:
        for product in products.data:
            print(f"{product['name']}: ${product['price']}")
    else:
        print(f"Error: {products.error}")
```

## Преимущества клиент-серверной архитектуры

### 1. Централизованное управление

```python
# Все данные хранятся на сервере
# Легко управлять доступом и делать резервные копии

class CentralizedDataServer:
    def __init__(self):
        self.access_log = []

    async def handle_request(self, request, user):
        # Централизованный контроль доступа
        if not self.check_permissions(user, request.resource):
            raise PermissionError("Access denied")

        # Логирование всех операций
        self.access_log.append({
            "user": user.id,
            "action": request.action,
            "resource": request.resource,
            "timestamp": datetime.now()
        })

        return await self.process_request(request)
```

### 2. Масштабируемость

```
                    ┌─────────────┐
                    │   Client 1  │
                    └──────┬──────┘
                           │
┌─────────────┐     ┌──────▼──────┐     ┌─────────────┐
│   Client 2  │────►│    Load     │────►│  Server 1   │
└─────────────┘     │   Balancer  │     └─────────────┘
                    └──────┬──────┘     ┌─────────────┐
┌─────────────┐            │     ├─────►│  Server 2   │
│   Client 3  │────────────┘           └─────────────┘
└─────────────┘                         ┌─────────────┐
                                   └────►│  Server 3   │
                                        └─────────────┘
```

### 3. Безопасность

```python
# Сервер контролирует все операции с данными

class SecureServer:
    def __init__(self):
        self.rate_limiter = RateLimiter()
        self.auth_service = AuthService()

    async def process_request(self, request):
        # Rate limiting
        if not self.rate_limiter.allow(request.client_ip):
            raise TooManyRequestsError()

        # Аутентификация
        user = await self.auth_service.verify_token(request.token)
        if not user:
            raise UnauthorizedError()

        # Авторизация
        if not self.check_permissions(user, request.action):
            raise ForbiddenError()

        # Валидация входных данных
        validated_data = self.validate_input(request.data)

        # Обработка запроса
        return await self.handle_action(user, request.action, validated_data)
```

## Недостатки клиент-серверной архитектуры

### 1. Единая точка отказа

```python
# Проблема: если сервер недоступен, система не работает

# Решение: репликация и failover
class HighAvailabilityClient:
    def __init__(self, server_urls: list):
        self.servers = server_urls
        self.current_server_index = 0

    def _get_server(self):
        return self.servers[self.current_server_index]

    def _failover(self):
        """Переключение на резервный сервер."""
        self.current_server_index = (
            (self.current_server_index + 1) % len(self.servers)
        )

    def request(self, endpoint: str, **kwargs):
        for _ in range(len(self.servers)):
            try:
                response = requests.get(
                    f"{self._get_server()}{endpoint}",
                    timeout=5,
                    **kwargs
                )
                return response
            except requests.exceptions.RequestException:
                self._failover()

        raise Exception("All servers unavailable")
```

### 2. Сетевая зависимость

```python
# Проблема: сетевые задержки влияют на производительность

# Решение: кэширование на клиенте
class CachingClient:
    def __init__(self, server_url: str, cache_ttl: int = 300):
        self.server_url = server_url
        self.cache = {}
        self.cache_ttl = cache_ttl

    def get_with_cache(self, endpoint: str):
        cache_key = endpoint
        cached = self.cache.get(cache_key)

        if cached and time.time() - cached["timestamp"] < self.cache_ttl:
            return cached["data"]

        response = requests.get(f"{self.server_url}{endpoint}")
        data = response.json()

        self.cache[cache_key] = {
            "data": data,
            "timestamp": time.time()
        }

        return data
```

## Best Practices

### 1. Используйте версионирование API

```python
# Версионирование в URL
@app.get("/api/v1/users/{user_id}")
async def get_user_v1(user_id: int):
    return {"id": user_id, "name": "John"}

@app.get("/api/v2/users/{user_id}")
async def get_user_v2(user_id: int):
    return {"id": user_id, "name": "John", "metadata": {}}
```

### 2. Реализуйте Circuit Breaker

```python
from enum import Enum
import time

class CircuitState(Enum):
    CLOSED = "closed"      # Нормальная работа
    OPEN = "open"          # Сервер недоступен
    HALF_OPEN = "half_open"  # Проверка доступности

class CircuitBreaker:
    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: int = 30
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failures = 0
        self.state = CircuitState.CLOSED
        self.last_failure_time = None

    def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
            else:
                raise Exception("Circuit breaker is open")

        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise e

    def _on_success(self):
        self.failures = 0
        self.state = CircuitState.CLOSED

    def _on_failure(self):
        self.failures += 1
        self.last_failure_time = time.time()
        if self.failures >= self.failure_threshold:
            self.state = CircuitState.OPEN
```

### 3. Используйте идемпотентные операции

```python
# Идемпотентность - повторный запрос даёт тот же результат

@app.put("/orders/{order_id}/status")
async def update_order_status(
    order_id: int,
    status: str,
    idempotency_key: str = Header(None)
):
    # Проверяем, был ли уже обработан этот запрос
    if idempotency_key:
        cached_result = await cache.get(f"idempotency:{idempotency_key}")
        if cached_result:
            return cached_result

    # Обрабатываем запрос
    result = await order_service.update_status(order_id, status)

    # Сохраняем результат
    if idempotency_key:
        await cache.set(
            f"idempotency:{idempotency_key}",
            result,
            ttl=3600
        )

    return result
```

## Когда использовать?

### Подходит для:

1. **Веб-приложений** с централизованными данными
2. **Корпоративных систем** с контролем доступа
3. **API для мобильных приложений**
4. **Систем с общими ресурсами** (базы данных, файлы)

### Не подходит для:

1. **Оффлайн-приложений** без постоянного интернета
2. **P2P систем** без центрального контроля
3. **Высокодецентрализованных систем**
4. **Real-time игр** с высокими требованиями к задержке

## Заключение

Клиент-серверная архитектура остаётся основой большинства современных приложений. Её простота, масштабируемость и возможность централизованного управления делают её идеальным выбором для веб-приложений, API и корпоративных систем. При правильной реализации с учётом обработки ошибок, кэширования и отказоустойчивости, эта архитектура обеспечивает надёжную основу для любого масштаба приложений.
