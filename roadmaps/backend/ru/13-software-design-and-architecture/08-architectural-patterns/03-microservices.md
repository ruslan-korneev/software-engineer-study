# Microservices (Микросервисная архитектура)

## Что такое микросервисы?

Микросервисная архитектура — это подход к разработке программного обеспечения, при котором приложение строится как набор небольших, независимо развёртываемых сервисов. Каждый микросервис выполняет одну бизнес-функцию, имеет собственное хранилище данных и взаимодействует с другими сервисами через API.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Микросервисная архитектура                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│                           ┌─────────────┐                               │
│                           │ API Gateway │                               │
│                           └──────┬──────┘                               │
│                                  │                                       │
│          ┌───────────────────────┼───────────────────────┐              │
│          │                       │                       │              │
│          ▼                       ▼                       ▼              │
│   ┌─────────────┐         ┌─────────────┐         ┌─────────────┐      │
│   │   User      │         │   Order     │         │   Product   │      │
│   │  Service    │         │  Service    │         │  Service    │      │
│   └──────┬──────┘         └──────┬──────┘         └──────┬──────┘      │
│          │                       │                       │              │
│          ▼                       ▼                       ▼              │
│   ┌─────────────┐         ┌─────────────┐         ┌─────────────┐      │
│   │  User DB    │         │  Order DB   │         │ Product DB  │      │
│   │ (PostgreSQL)│         │  (MongoDB)  │         │   (Redis)   │      │
│   └─────────────┘         └─────────────┘         └─────────────┘      │
│                                                                          │
│   Каждый сервис:                                                        │
│   • Независимо развёртывается                                           │
│   • Имеет своё хранилище данных                                         │
│   • Общается через API (HTTP/gRPC/Message Queue)                        │
│   • Может использовать разные технологии                                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## Монолит vs Микросервисы

```
┌────────────────────────────────────────────────────────────────────────┐
│                    Монолит vs Микросервисы                              │
├──────────────────────────────┬─────────────────────────────────────────┤
│         Монолит              │            Микросервисы                 │
├──────────────────────────────┼─────────────────────────────────────────┤
│                              │                                         │
│  ┌────────────────────┐      │    ┌─────┐  ┌─────┐  ┌─────┐           │
│  │                    │      │    │ Svc │  │ Svc │  │ Svc │           │
│  │   User Module      │      │    │  A  │  │  B  │  │  C  │           │
│  │   ─────────────    │      │    └──┬──┘  └──┬──┘  └──┬──┘           │
│  │   Order Module     │      │       │        │        │              │
│  │   ─────────────    │      │    ┌──┴──┐  ┌──┴──┐  ┌──┴──┐           │
│  │   Product Module   │      │    │ DB  │  │ DB  │  │ DB  │           │
│  │                    │      │    └─────┘  └─────┘  └─────┘           │
│  └─────────┬──────────┘      │                                         │
│            │                 │                                         │
│     ┌──────┴──────┐          │  + Независимое развёртывание           │
│     │   Shared    │          │  + Масштабирование по частям           │
│     │   Database  │          │  + Изоляция отказов                    │
│     └─────────────┘          │  - Сложность инфраструктуры            │
│                              │  - Распределённые транзакции           │
│  + Простота разработки       │                                         │
│  + Простое развёртывание     │                                         │
│  - Сложность масштабирования │                                         │
│  - Один отказ = всё падает   │                                         │
│                              │                                         │
└──────────────────────────────┴─────────────────────────────────────────┘
```

## Принципы микросервисной архитектуры

### 1. Single Responsibility (Единственная ответственность)

Каждый сервис отвечает за одну бизнес-функцию.

```python
# user_service/main.py
# User Service — отвечает ТОЛЬКО за пользователей

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, EmailStr
from typing import Optional
import uuid

app = FastAPI(title="User Service")


class UserCreate(BaseModel):
    email: EmailStr
    username: str
    password: str


class User(BaseModel):
    id: str
    email: str
    username: str
    is_active: bool = True


# In-memory storage (в реальности — база данных)
users_db: dict[str, dict] = {}


@app.post("/users", response_model=User)
async def create_user(user_data: UserCreate):
    """Создание пользователя"""
    # Проверка уникальности email
    for user in users_db.values():
        if user["email"] == user_data.email:
            raise HTTPException(400, "Email already exists")

    user_id = str(uuid.uuid4())
    user = {
        "id": user_id,
        "email": user_data.email,
        "username": user_data.username,
        "password_hash": hash(user_data.password),  # Упрощённо
        "is_active": True
    }
    users_db[user_id] = user

    return User(**user)


@app.get("/users/{user_id}", response_model=User)
async def get_user(user_id: str):
    """Получение пользователя по ID"""
    if user_id not in users_db:
        raise HTTPException(404, "User not found")
    return User(**users_db[user_id])


@app.get("/users", response_model=list[User])
async def list_users():
    """Список всех пользователей"""
    return [User(**u) for u in users_db.values()]


@app.delete("/users/{user_id}")
async def delete_user(user_id: str):
    """Удаление пользователя"""
    if user_id not in users_db:
        raise HTTPException(404, "User not found")
    del users_db[user_id]
    return {"status": "deleted"}
```

### 2. Независимое хранилище данных (Database per Service)

Каждый сервис имеет собственную базу данных.

```python
# order_service/main.py
# Order Service — своя база данных MongoDB

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
from datetime import datetime
from enum import Enum
import httpx
import uuid

app = FastAPI(title="Order Service")

# Конфигурация
USER_SERVICE_URL = "http://user-service:8000"


class OrderStatus(str, Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"


class OrderItem(BaseModel):
    product_id: str
    quantity: int
    price: float


class OrderCreate(BaseModel):
    user_id: str
    items: list[OrderItem]


class Order(BaseModel):
    id: str
    user_id: str
    items: list[OrderItem]
    status: OrderStatus
    total: float
    created_at: datetime


# In-memory storage (в реальности — MongoDB)
orders_db: dict[str, dict] = {}


async def verify_user_exists(user_id: str) -> bool:
    """Проверка существования пользователя через User Service"""
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(f"{USER_SERVICE_URL}/users/{user_id}")
            return response.status_code == 200
        except httpx.RequestError:
            # Если сервис недоступен — используем fallback
            return True  # или кэшированные данные


@app.post("/orders", response_model=Order)
async def create_order(order_data: OrderCreate):
    """Создание заказа"""
    # Проверяем существование пользователя
    if not await verify_user_exists(order_data.user_id):
        raise HTTPException(400, "User not found")

    order_id = str(uuid.uuid4())
    total = sum(item.price * item.quantity for item in order_data.items)

    order = {
        "id": order_id,
        "user_id": order_data.user_id,
        "items": [item.dict() for item in order_data.items],
        "status": OrderStatus.PENDING,
        "total": total,
        "created_at": datetime.now()
    }
    orders_db[order_id] = order

    return Order(**order)


@app.get("/orders/{order_id}", response_model=Order)
async def get_order(order_id: str):
    """Получение заказа"""
    if order_id not in orders_db:
        raise HTTPException(404, "Order not found")
    return Order(**orders_db[order_id])


@app.get("/orders/user/{user_id}", response_model=list[Order])
async def get_user_orders(user_id: str):
    """Получение заказов пользователя"""
    user_orders = [
        Order(**o) for o in orders_db.values()
        if o["user_id"] == user_id
    ]
    return user_orders


@app.patch("/orders/{order_id}/status")
async def update_order_status(order_id: str, status: OrderStatus):
    """Обновление статуса заказа"""
    if order_id not in orders_db:
        raise HTTPException(404, "Order not found")

    orders_db[order_id]["status"] = status
    return Order(**orders_db[order_id])
```

### 3. Коммуникация между сервисами

#### Синхронная коммуникация (HTTP/gRPC)

```python
# product_service/client.py
# Клиент для вызова других сервисов

import httpx
from typing import Optional
from pydantic import BaseModel


class ProductServiceClient:
    """Клиент для Product Service"""

    def __init__(self, base_url: str):
        self.base_url = base_url

    async def get_product(self, product_id: str) -> Optional[dict]:
        """Получить продукт по ID"""
        async with httpx.AsyncClient() as client:
            try:
                response = await client.get(
                    f"{self.base_url}/products/{product_id}",
                    timeout=5.0
                )
                if response.status_code == 200:
                    return response.json()
                return None
            except httpx.RequestError as e:
                # Логирование ошибки
                print(f"Error calling product service: {e}")
                return None

    async def check_availability(self, product_id: str, quantity: int) -> bool:
        """Проверить наличие товара"""
        async with httpx.AsyncClient() as client:
            try:
                response = await client.get(
                    f"{self.base_url}/products/{product_id}/availability",
                    params={"quantity": quantity},
                    timeout=5.0
                )
                if response.status_code == 200:
                    return response.json().get("available", False)
                return False
            except httpx.RequestError:
                return False


class UserServiceClient:
    """Клиент для User Service"""

    def __init__(self, base_url: str):
        self.base_url = base_url

    async def get_user(self, user_id: str) -> Optional[dict]:
        """Получить пользователя по ID"""
        async with httpx.AsyncClient() as client:
            try:
                response = await client.get(
                    f"{self.base_url}/users/{user_id}",
                    timeout=5.0
                )
                if response.status_code == 200:
                    return response.json()
                return None
            except httpx.RequestError:
                return None

    async def verify_user(self, user_id: str) -> bool:
        """Проверить существование пользователя"""
        user = await self.get_user(user_id)
        return user is not None
```

#### Асинхронная коммуникация (Message Queue)

```python
# events/publisher.py
# Публикация событий через RabbitMQ

import json
import aio_pika
from datetime import datetime
from typing import Any


class EventPublisher:
    """Издатель событий"""

    def __init__(self, amqp_url: str):
        self.amqp_url = amqp_url
        self.connection = None
        self.channel = None

    async def connect(self):
        """Подключение к RabbitMQ"""
        self.connection = await aio_pika.connect_robust(self.amqp_url)
        self.channel = await self.connection.channel()

    async def publish(self, event_type: str, data: dict, routing_key: str = ""):
        """Опубликовать событие"""
        if not self.channel:
            await self.connect()

        message = {
            "event_type": event_type,
            "data": data,
            "timestamp": datetime.now().isoformat(),
            "version": "1.0"
        }

        await self.channel.default_exchange.publish(
            aio_pika.Message(
                body=json.dumps(message).encode(),
                content_type="application/json"
            ),
            routing_key=routing_key or event_type
        )

    async def close(self):
        """Закрыть соединение"""
        if self.connection:
            await self.connection.close()


# Использование в Order Service
class OrderService:
    def __init__(self, event_publisher: EventPublisher):
        self.publisher = event_publisher

    async def create_order(self, order_data: dict) -> dict:
        # Создание заказа...
        order = self._save_order(order_data)

        # Публикация события
        await self.publisher.publish(
            event_type="order.created",
            data={
                "order_id": order["id"],
                "user_id": order["user_id"],
                "total": order["total"],
                "items": order["items"]
            }
        )

        return order

    async def cancel_order(self, order_id: str) -> dict:
        order = self._get_order(order_id)
        order["status"] = "cancelled"

        # Публикация события отмены
        await self.publisher.publish(
            event_type="order.cancelled",
            data={
                "order_id": order_id,
                "user_id": order["user_id"],
                "reason": "user_request"
            }
        )

        return order


# events/consumer.py
# Подписчик на события

class EventConsumer:
    """Потребитель событий"""

    def __init__(self, amqp_url: str):
        self.amqp_url = amqp_url
        self.handlers: dict = {}

    def on_event(self, event_type: str):
        """Декоратор для обработчиков событий"""
        def decorator(func):
            self.handlers[event_type] = func
            return func
        return decorator

    async def start(self, queue_name: str):
        """Начать прослушивание"""
        connection = await aio_pika.connect_robust(self.amqp_url)
        channel = await connection.channel()
        queue = await channel.declare_queue(queue_name, durable=True)

        async with queue.iterator() as queue_iter:
            async for message in queue_iter:
                async with message.process():
                    await self._handle_message(message)

    async def _handle_message(self, message):
        """Обработка сообщения"""
        data = json.loads(message.body.decode())
        event_type = data.get("event_type")

        if event_type in self.handlers:
            await self.handlers[event_type](data["data"])


# Inventory Service — подписчик на события заказов
inventory_consumer = EventConsumer("amqp://localhost")


@inventory_consumer.on_event("order.created")
async def handle_order_created(data: dict):
    """Обработка события создания заказа"""
    print(f"Reserving inventory for order {data['order_id']}")
    for item in data["items"]:
        await reserve_inventory(item["product_id"], item["quantity"])


@inventory_consumer.on_event("order.cancelled")
async def handle_order_cancelled(data: dict):
    """Обработка события отмены заказа"""
    print(f"Releasing inventory for order {data['order_id']}")
    # Освободить зарезервированный товар
    await release_inventory(data["order_id"])
```

## Паттерны микросервисов

### API Gateway

```python
# api_gateway/main.py
# Единая точка входа для всех запросов

from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse
import httpx
from typing import Optional

app = FastAPI(title="API Gateway")

# Конфигурация сервисов
SERVICES = {
    "users": "http://user-service:8000",
    "orders": "http://order-service:8001",
    "products": "http://product-service:8002",
    "payments": "http://payment-service:8003",
}


class ServiceRouter:
    """Маршрутизация запросов к сервисам"""

    def __init__(self):
        self.client = httpx.AsyncClient(timeout=30.0)

    async def route(self, service: str, path: str,
                    method: str, headers: dict, body: Optional[bytes] = None):
        """Маршрутизация запроса к сервису"""
        if service not in SERVICES:
            raise HTTPException(404, f"Service {service} not found")

        url = f"{SERVICES[service]}{path}"

        # Фильтруем заголовки
        forward_headers = {
            k: v for k, v in headers.items()
            if k.lower() not in ["host", "content-length"]
        }

        response = await self.client.request(
            method=method,
            url=url,
            headers=forward_headers,
            content=body
        )

        return response


router = ServiceRouter()


# Аутентификация
async def verify_token(request: Request) -> Optional[dict]:
    """Проверка JWT токена"""
    auth_header = request.headers.get("Authorization")
    if not auth_header or not auth_header.startswith("Bearer "):
        return None

    token = auth_header[7:]
    # Валидация токена (упрощённо)
    # В реальности — вызов Auth Service или проверка JWT
    return {"user_id": "123", "roles": ["user"]}


# Rate Limiting
from collections import defaultdict
from datetime import datetime

request_counts = defaultdict(list)


async def check_rate_limit(client_ip: str) -> bool:
    """Проверка лимита запросов"""
    now = datetime.now()
    minute_ago = now.timestamp() - 60

    # Удаляем старые записи
    request_counts[client_ip] = [
        ts for ts in request_counts[client_ip]
        if ts > minute_ago
    ]

    # Проверяем лимит (100 запросов в минуту)
    if len(request_counts[client_ip]) >= 100:
        return False

    request_counts[client_ip].append(now.timestamp())
    return True


@app.api_route("/{service}/{path:path}", methods=["GET", "POST", "PUT", "DELETE", "PATCH"])
async def gateway(service: str, path: str, request: Request):
    """Основной обработчик gateway"""

    # Rate limiting
    client_ip = request.client.host
    if not await check_rate_limit(client_ip):
        raise HTTPException(429, "Too many requests")

    # Аутентификация для защищённых эндпоинтов
    if service in ["orders", "payments"]:
        user = await verify_token(request)
        if not user:
            raise HTTPException(401, "Unauthorized")

    # Маршрутизация к сервису
    body = await request.body() if request.method in ["POST", "PUT", "PATCH"] else None

    try:
        response = await router.route(
            service=service,
            path=f"/{path}",
            method=request.method,
            headers=dict(request.headers),
            body=body
        )

        return JSONResponse(
            content=response.json() if response.content else None,
            status_code=response.status_code
        )
    except httpx.RequestError as e:
        raise HTTPException(503, f"Service unavailable: {service}")
```

### Service Discovery

```python
# service_discovery/registry.py
# Реестр сервисов

from typing import Dict, List, Optional
from dataclasses import dataclass, field
from datetime import datetime, timedelta
import random


@dataclass
class ServiceInstance:
    """Экземпляр сервиса"""
    service_name: str
    instance_id: str
    host: str
    port: int
    health_check_url: str
    last_heartbeat: datetime = field(default_factory=datetime.now)
    metadata: dict = field(default_factory=dict)


class ServiceRegistry:
    """Реестр сервисов (in-memory версия)"""

    def __init__(self, heartbeat_timeout: int = 30):
        self.services: Dict[str, Dict[str, ServiceInstance]] = {}
        self.heartbeat_timeout = timedelta(seconds=heartbeat_timeout)

    def register(self, instance: ServiceInstance) -> None:
        """Регистрация сервиса"""
        if instance.service_name not in self.services:
            self.services[instance.service_name] = {}

        self.services[instance.service_name][instance.instance_id] = instance
        print(f"Registered: {instance.service_name}/{instance.instance_id}")

    def deregister(self, service_name: str, instance_id: str) -> None:
        """Отмена регистрации"""
        if service_name in self.services:
            if instance_id in self.services[service_name]:
                del self.services[service_name][instance_id]
                print(f"Deregistered: {service_name}/{instance_id}")

    def heartbeat(self, service_name: str, instance_id: str) -> bool:
        """Обновление heartbeat"""
        if service_name in self.services:
            if instance_id in self.services[service_name]:
                self.services[service_name][instance_id].last_heartbeat = datetime.now()
                return True
        return False

    def get_instances(self, service_name: str) -> List[ServiceInstance]:
        """Получить все экземпляры сервиса"""
        if service_name not in self.services:
            return []

        now = datetime.now()
        healthy = [
            instance for instance in self.services[service_name].values()
            if now - instance.last_heartbeat < self.heartbeat_timeout
        ]
        return healthy

    def get_instance(self, service_name: str) -> Optional[ServiceInstance]:
        """Получить один экземпляр (round-robin)"""
        instances = self.get_instances(service_name)
        if not instances:
            return None
        return random.choice(instances)  # Упрощённый load balancing


# Клиент для работы с Service Discovery
class ServiceDiscoveryClient:
    """Клиент для обнаружения сервисов"""

    def __init__(self, registry: ServiceRegistry):
        self.registry = registry
        self.cache: Dict[str, List[ServiceInstance]] = {}

    def get_service_url(self, service_name: str) -> Optional[str]:
        """Получить URL сервиса"""
        instance = self.registry.get_instance(service_name)
        if instance:
            return f"http://{instance.host}:{instance.port}"
        return None

    def get_all_urls(self, service_name: str) -> List[str]:
        """Получить все URL сервиса"""
        instances = self.registry.get_instances(service_name)
        return [f"http://{i.host}:{i.port}" for i in instances]
```

### Circuit Breaker

```python
# resilience/circuit_breaker.py
# Паттерн Circuit Breaker для отказоустойчивости

from enum import Enum
from datetime import datetime, timedelta
from typing import Callable, Any
import asyncio


class CircuitState(Enum):
    CLOSED = "closed"      # Нормальная работа
    OPEN = "open"          # Прерывание запросов
    HALF_OPEN = "half_open"  # Тестовые запросы


class CircuitBreaker:
    """
    Circuit Breaker предотвращает каскадные отказы.
    """

    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: int = 30,
        success_threshold: int = 2
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = timedelta(seconds=recovery_timeout)
        self.success_threshold = success_threshold

        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time: datetime = None

    async def call(self, func: Callable, *args, **kwargs) -> Any:
        """Выполнить функцию через circuit breaker"""

        if self.state == CircuitState.OPEN:
            if self._should_attempt_reset():
                self.state = CircuitState.HALF_OPEN
                self.success_count = 0
            else:
                raise CircuitOpenError("Circuit is open")

        try:
            result = await func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise e

    def _should_attempt_reset(self) -> bool:
        """Проверить, пора ли попробовать восстановление"""
        if self.last_failure_time is None:
            return True
        return datetime.now() - self.last_failure_time >= self.recovery_timeout

    def _on_success(self):
        """Обработка успешного вызова"""
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.success_threshold:
                self.state = CircuitState.CLOSED
                self.failure_count = 0
        else:
            self.failure_count = 0

    def _on_failure(self):
        """Обработка неудачного вызова"""
        self.failure_count += 1
        self.last_failure_time = datetime.now()

        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.OPEN
        elif self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN


class CircuitOpenError(Exception):
    """Ошибка: circuit breaker открыт"""
    pass


# Использование Circuit Breaker
class ResilientServiceClient:
    """Клиент с Circuit Breaker"""

    def __init__(self, base_url: str):
        self.base_url = base_url
        self.circuit_breaker = CircuitBreaker(
            failure_threshold=5,
            recovery_timeout=30
        )

    async def get_user(self, user_id: str) -> dict:
        """Получить пользователя с защитой от отказов"""
        try:
            return await self.circuit_breaker.call(
                self._fetch_user, user_id
            )
        except CircuitOpenError:
            # Fallback: вернуть кэшированные данные или default
            return self._get_cached_user(user_id) or {"id": user_id, "name": "Unknown"}

    async def _fetch_user(self, user_id: str) -> dict:
        """Внутренний метод для запроса"""
        import httpx
        async with httpx.AsyncClient() as client:
            response = await client.get(
                f"{self.base_url}/users/{user_id}",
                timeout=5.0
            )
            response.raise_for_status()
            return response.json()

    def _get_cached_user(self, user_id: str) -> dict:
        # Логика получения из кэша
        return None
```

### Saga Pattern

```python
# saga/orchestrator.py
# Saga для распределённых транзакций

from abc import ABC, abstractmethod
from dataclasses import dataclass
from enum import Enum
from typing import List, Optional
import asyncio


class SagaStepStatus(Enum):
    PENDING = "pending"
    COMPLETED = "completed"
    FAILED = "failed"
    COMPENSATED = "compensated"


@dataclass
class SagaStep:
    """Шаг саги"""
    name: str
    execute: callable
    compensate: callable
    status: SagaStepStatus = SagaStepStatus.PENDING


class SagaOrchestrator:
    """
    Оркестратор саги для управления распределёнными транзакциями.
    """

    def __init__(self, saga_id: str):
        self.saga_id = saga_id
        self.steps: List[SagaStep] = []
        self.completed_steps: List[SagaStep] = []

    def add_step(self, name: str, execute: callable, compensate: callable):
        """Добавить шаг саги"""
        self.steps.append(SagaStep(name, execute, compensate))

    async def execute(self, context: dict) -> dict:
        """Выполнить сагу"""
        for step in self.steps:
            try:
                print(f"Executing step: {step.name}")
                result = await step.execute(context)
                context.update(result or {})
                step.status = SagaStepStatus.COMPLETED
                self.completed_steps.append(step)
            except Exception as e:
                print(f"Step {step.name} failed: {e}")
                step.status = SagaStepStatus.FAILED
                await self._compensate(context)
                raise SagaFailedException(f"Saga failed at step {step.name}: {e}")

        return context

    async def _compensate(self, context: dict):
        """Компенсация выполненных шагов"""
        print("Starting compensation...")

        for step in reversed(self.completed_steps):
            try:
                print(f"Compensating step: {step.name}")
                await step.compensate(context)
                step.status = SagaStepStatus.COMPENSATED
            except Exception as e:
                print(f"Compensation failed for {step.name}: {e}")
                # Логирование для ручного вмешательства


class SagaFailedException(Exception):
    pass


# Пример: Saga для создания заказа
class OrderCreationSaga:
    """Saga для создания заказа"""

    def __init__(self, order_service, inventory_service, payment_service):
        self.order_service = order_service
        self.inventory_service = inventory_service
        self.payment_service = payment_service

    async def execute(self, order_data: dict) -> dict:
        saga = SagaOrchestrator(f"order-{order_data['order_id']}")

        # Шаг 1: Создать заказ
        saga.add_step(
            name="create_order",
            execute=lambda ctx: self._create_order(ctx),
            compensate=lambda ctx: self._cancel_order(ctx)
        )

        # Шаг 2: Зарезервировать товары
        saga.add_step(
            name="reserve_inventory",
            execute=lambda ctx: self._reserve_inventory(ctx),
            compensate=lambda ctx: self._release_inventory(ctx)
        )

        # Шаг 3: Обработать платёж
        saga.add_step(
            name="process_payment",
            execute=lambda ctx: self._process_payment(ctx),
            compensate=lambda ctx: self._refund_payment(ctx)
        )

        return await saga.execute(order_data)

    async def _create_order(self, ctx: dict) -> dict:
        order = await self.order_service.create(ctx)
        return {"order_id": order["id"]}

    async def _cancel_order(self, ctx: dict):
        await self.order_service.cancel(ctx["order_id"])

    async def _reserve_inventory(self, ctx: dict) -> dict:
        reservation = await self.inventory_service.reserve(
            ctx["items"],
            ctx["order_id"]
        )
        return {"reservation_id": reservation["id"]}

    async def _release_inventory(self, ctx: dict):
        await self.inventory_service.release(ctx["reservation_id"])

    async def _process_payment(self, ctx: dict) -> dict:
        payment = await self.payment_service.charge(
            ctx["user_id"],
            ctx["total"],
            ctx["order_id"]
        )
        return {"payment_id": payment["id"]}

    async def _refund_payment(self, ctx: dict):
        await self.payment_service.refund(ctx["payment_id"])
```

## Docker и Kubernetes для микросервисов

### Docker Compose

```yaml
# docker-compose.yml

version: '3.8'

services:
  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    environment:
      - USER_SERVICE_URL=http://user-service:8000
      - ORDER_SERVICE_URL=http://order-service:8001
      - PRODUCT_SERVICE_URL=http://product-service:8002
    depends_on:
      - user-service
      - order-service
      - product-service

  user-service:
    build: ./user-service
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:password@user-db:5432/users
      - RABBITMQ_URL=amqp://rabbitmq:5672
    depends_on:
      - user-db
      - rabbitmq

  order-service:
    build: ./order-service
    ports:
      - "8001:8001"
    environment:
      - MONGODB_URL=mongodb://order-db:27017/orders
      - RABBITMQ_URL=amqp://rabbitmq:5672
      - USER_SERVICE_URL=http://user-service:8000
    depends_on:
      - order-db
      - rabbitmq

  product-service:
    build: ./product-service
    ports:
      - "8002:8002"
    environment:
      - REDIS_URL=redis://product-db:6379
    depends_on:
      - product-db

  # Базы данных
  user-db:
    image: postgres:15
    environment:
      - POSTGRES_DB=users
      - POSTGRES_PASSWORD=password
    volumes:
      - user-db-data:/var/lib/postgresql/data

  order-db:
    image: mongo:6
    volumes:
      - order-db-data:/data/db

  product-db:
    image: redis:7
    volumes:
      - product-db-data:/data

  # Message Broker
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "15672:15672"  # Management UI
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq

volumes:
  user-db-data:
  order-db-data:
  product-db-data:
  rabbitmq-data:
```

## Плюсы и минусы микросервисов

### Плюсы

1. **Независимое развёртывание** — обновление одного сервиса не затрагивает другие
2. **Масштабирование** — можно масштабировать только нужные сервисы
3. **Изоляция отказов** — падение одного сервиса не роняет всю систему
4. **Технологическая гибкость** — разные сервисы могут использовать разные технологии
5. **Организационная гибкость** — команды могут работать независимо
6. **Повторное использование** — сервисы можно использовать в разных продуктах

### Минусы

1. **Сложность инфраструктуры** — нужны дополнительные инструменты (оркестрация, мониторинг)
2. **Распределённые транзакции** — сложнее поддерживать консистентность данных
3. **Сетевая латентность** — межсервисное взаимодействие медленнее, чем in-process
4. **Сложность тестирования** — интеграционное тестирование сложнее
5. **Операционная сложность** — нужны DevOps навыки для управления
6. **Дублирование кода** — общая логика может дублироваться

## Когда использовать микросервисы

### Используйте микросервисы когда:

- Большая команда (10+ разработчиков)
- Разные части системы имеют разные требования к масштабированию
- Нужна высокая доступность отдельных компонентов
- Разные части системы развиваются с разной скоростью
- Команды географически распределены
- Есть зрелая DevOps культура

### Не используйте микросервисы когда:

- Маленькая команда (< 5 человек)
- Стартап или MVP
- Нет опыта с распределёнными системами
- Нет инфраструктуры для оркестрации
- Домен ещё не понят достаточно хорошо
- Простое приложение без сложных требований

## Заключение

Микросервисная архитектура — мощный подход для построения сложных, масштабируемых систем. Однако она приносит значительную сложность и требует зрелой инфраструктуры и команды. Начинайте с монолита и переходите к микросервисам по мере роста системы и команды.
