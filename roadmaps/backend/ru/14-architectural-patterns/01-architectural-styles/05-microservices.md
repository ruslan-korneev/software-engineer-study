# Микросервисная архитектура (Microservices Architecture)

## Определение

**Микросервисная архитектура** — это подход к разработке, при котором приложение строится как набор небольших, независимо развёртываемых сервисов. Каждый сервис реализует конкретную бизнес-возможность (bounded context), имеет собственную базу данных и взаимодействует с другими сервисами через lightweight протоколы (REST, gRPC, messaging).

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        MICROSERVICES ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│      ┌─────────────────┐                                                │
│      │   API Gateway   │                                                │
│      └────────┬────────┘                                                │
│               │                                                          │
│   ┌───────────┼───────────┬───────────┬───────────┐                     │
│   │           │           │           │           │                     │
│   ▼           ▼           ▼           ▼           ▼                     │
│ ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐                        │
│ │User │   │Order│   │ Pay │   │ Inv │   │Notif│                        │
│ │Svc  │   │ Svc │   │ Svc │   │ Svc │   │ Svc │                        │
│ └──┬──┘   └──┬──┘   └──┬──┘   └──┬──┘   └──┬──┘                        │
│    │         │         │         │         │                            │
│    ▼         ▼         ▼         ▼         ▼                            │
│ ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐                        │
│ │ DB  │   │ DB  │   │ DB  │   │ DB  │   │ DB  │                        │
│ └─────┘   └─────┘   └─────┘   └─────┘   └─────┘                        │
│                                                                          │
│         ┌─────────────────────────────────────────┐                     │
│         │           Message Broker                │                     │
│         │        (Kafka / RabbitMQ)              │                     │
│         └─────────────────────────────────────────┘                     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

Термин популяризирован James Lewis и Martin Fowler в 2014 году.

## Ключевые характеристики

### 1. Характеристики микросервисов

| Характеристика | Описание |
|----------------|----------|
| **Single Responsibility** | Один сервис = одна бизнес-функция |
| **Независимый deployment** | Можно деплоить отдельно от других |
| **Decentralized Data** | Database per service |
| **Lightweight Communication** | REST, gRPC, async messaging |
| **Smart endpoints, dumb pipes** | Логика в сервисах, не в middleware |
| **Design for Failure** | Circuit breakers, retries, timeouts |
| **Polyglot** | Разные языки и технологии |
| **DevOps Culture** | CI/CD, автоматизация, мониторинг |

### 2. Организация по бизнес-возможностям

```
┌─────────────────────────────────────────────────────────────────┐
│              BOUNDED CONTEXTS (Domain-Driven Design)            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐    ┌─────────────────┐                     │
│  │   User Context  │    │  Order Context  │                     │
│  │  ┌───────────┐  │    │  ┌───────────┐  │                     │
│  │  │ User Svc  │  │    │  │ Order Svc │  │                     │
│  │  │ Auth Svc  │  │    │  │ Cart Svc  │  │                     │
│  │  │ Profile   │  │    │  │ Pricing   │  │                     │
│  │  └───────────┘  │    │  └───────────┘  │                     │
│  └─────────────────┘    └─────────────────┘                     │
│                                                                  │
│  ┌─────────────────┐    ┌─────────────────┐                     │
│  │ Payment Context │    │Shipping Context │                     │
│  │  ┌───────────┐  │    │  ┌───────────┐  │                     │
│  │  │Payment Svc│  │    │  │ Ship Svc  │  │                     │
│  │  │ Fraud Svc │  │    │  │ Track Svc │  │                     │
│  │  │ Billing   │  │    │  │ Warehouse │  │                     │
│  │  └───────────┘  │    │  └───────────┘  │                     │
│  └─────────────────┘    └─────────────────┘                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Коммуникационные паттерны

```
SYNCHRONOUS                         ASYNCHRONOUS
────────────────                    ────────────────

┌───────┐  HTTP   ┌───────┐        ┌───────┐        ┌───────┐
│ Svc A │───────▶│ Svc B │        │ Svc A │        │ Svc B │
└───────┘        └───────┘        └───┬───┘        └───▲───┘
                                      │                │
REST API                              │   ┌────────┐   │
gRPC                                  └──▶│ Broker │───┘
GraphQL                                   └────────┘

                                   Event-Driven
                                   Message Queue
                                   Pub/Sub
```

### 4. Data Management

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATABASE PER SERVICE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │  User Svc    │    │  Order Svc   │    │  Product Svc │       │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘       │
│         │                   │                   │               │
│         ▼                   ▼                   ▼               │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │  PostgreSQL  │    │   MongoDB    │    │ Elasticsearch│       │
│  │   (Users)    │    │   (Orders)   │    │  (Products)  │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
│                                                                  │
│  Polyglot Persistence: каждый сервис выбирает подходящую БД    │
└─────────────────────────────────────────────────────────────────┘
```

## Когда использовать

### Идеальные сценарии

| Сценарий | Почему микросервисы подходят |
|----------|------------------------------|
| **Большие команды** (50+ разработчиков) | Независимые команды по сервисам |
| **Разные темпы развития модулей** | Независимый deployment |
| **Высокая нагрузка с разными SLA** | Масштабирование отдельных сервисов |
| **Polyglot требования** | Разные технологии для разных задач |
| **Continuous deployment** | Быстрые релизы без координации |
| **Устойчивость к сбоям** | Изоляция отказов |

### Типичный путь эволюции

```
Стадия 1: Монолит
─────────────────
• Startup / MVP
• Понимание домена
• Быстрая итерация

Стадия 2: Модульный монолит
───────────────────────────
• Выделение bounded contexts
• Подготовка к разделению
• Чёткие internal APIs

Стадия 3: Микросервисы
──────────────────────
• Разделение по доменам
• Database per service
• CI/CD для каждого сервиса
```

### Не подходит для

- Стартапы на ранней стадии
- Небольшие команды (< 10 человек)
- Простые приложения без сложного домена
- Системы с тесно связанными данными (требующие ACID)
- Команды без DevOps культуры

## Преимущества и недостатки

### Преимущества

| Преимущество | Описание |
|--------------|----------|
| **Независимый deployment** | Релиз одного сервиса без остальных |
| **Масштабируемость** | Горизонтальное масштабирование нужных сервисов |
| **Технологическая гибкость** | Выбор лучшего стека для каждой задачи |
| **Resilience** | Отказ одного сервиса не ломает всю систему |
| **Team autonomy** | Команды владеют сервисами end-to-end |
| **Faster time to market** | Параллельная разработка |

### Недостатки

| Недостаток | Описание |
|------------|----------|
| **Распределённая сложность** | Network latency, partial failures |
| **Data consistency** | Eventual consistency вместо ACID |
| **Operational overhead** | Много сервисов = много инфраструктуры |
| **Testing complexity** | Integration testing между сервисами |
| **Debugging difficulty** | Distributed tracing необходим |
| **Initial investment** | CI/CD, monitoring, service mesh |

## Примеры реализации

### Структура проекта (Python)

```
microservices-ecommerce/
├── services/
│   ├── user-service/
│   │   ├── src/
│   │   │   ├── main.py
│   │   │   ├── api/
│   │   │   ├── domain/
│   │   │   └── infrastructure/
│   │   ├── tests/
│   │   ├── Dockerfile
│   │   └── requirements.txt
│   │
│   ├── order-service/
│   │   ├── src/
│   │   ├── tests/
│   │   ├── Dockerfile
│   │   └── requirements.txt
│   │
│   ├── payment-service/
│   │   └── ...
│   │
│   └── notification-service/
│       └── ...
│
├── api-gateway/
│   └── ...
│
├── shared/
│   ├── proto/              # gRPC definitions
│   └── events/             # Event schemas
│
├── infrastructure/
│   ├── docker-compose.yaml
│   ├── kubernetes/
│   └── terraform/
│
└── docs/
```

### User Service

```python
# services/user-service/src/main.py
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel, EmailStr
from typing import Optional
import httpx
import os

app = FastAPI(title="User Service", version="1.0.0")

# Configuration
ORDER_SERVICE_URL = os.getenv("ORDER_SERVICE_URL", "http://order-service:8001")
EVENT_BUS_URL = os.getenv("EVENT_BUS_URL", "http://rabbitmq:5672")

# Models
class UserCreate(BaseModel):
    email: EmailStr
    name: str

class User(BaseModel):
    id: int
    email: str
    name: str
    is_active: bool = True

# In-memory storage (use database in production)
users_db = {}
next_id = 1

# Event Publisher
async def publish_event(event_type: str, payload: dict):
    """Публикация события в message broker"""
    async with httpx.AsyncClient() as client:
        try:
            await client.post(
                f"{EVENT_BUS_URL}/events",
                json={"type": event_type, "payload": payload}
            )
        except Exception as e:
            print(f"Failed to publish event: {e}")

# API Endpoints
@app.post("/users", response_model=User, status_code=201)
async def create_user(data: UserCreate):
    global next_id

    # Check email uniqueness
    for user in users_db.values():
        if user["email"] == data.email:
            raise HTTPException(400, "Email already exists")

    user = {
        "id": next_id,
        "email": data.email,
        "name": data.name,
        "is_active": True
    }
    users_db[next_id] = user
    next_id += 1

    # Publish event for other services
    await publish_event("user.created", user)

    return User(**user)

@app.get("/users/{user_id}", response_model=User)
async def get_user(user_id: int):
    if user_id not in users_db:
        raise HTTPException(404, "User not found")
    return User(**users_db[user_id])

@app.get("/users/{user_id}/orders")
async def get_user_orders(user_id: int):
    """Получение заказов пользователя (вызов Order Service)"""
    if user_id not in users_db:
        raise HTTPException(404, "User not found")

    # Synchronous call to Order Service
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(
                f"{ORDER_SERVICE_URL}/orders",
                params={"user_id": user_id},
                timeout=5.0
            )
            response.raise_for_status()
            return response.json()
        except httpx.RequestError as e:
            raise HTTPException(503, f"Order Service unavailable: {e}")

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "user-service"}
```

### Order Service

```python
# services/order-service/src/main.py
from fastapi import FastAPI, HTTPException, BackgroundTasks
from pydantic import BaseModel
from typing import List, Optional
from enum import Enum
from datetime import datetime
import httpx
import os
import uuid

app = FastAPI(title="Order Service", version="1.0.0")

# Configuration
USER_SERVICE_URL = os.getenv("USER_SERVICE_URL", "http://user-service:8000")
PAYMENT_SERVICE_URL = os.getenv("PAYMENT_SERVICE_URL", "http://payment-service:8002")

class OrderStatus(str, Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    PAID = "paid"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class OrderItem(BaseModel):
    product_id: int
    quantity: int
    price: float

class CreateOrderRequest(BaseModel):
    user_id: int
    items: List[OrderItem]

class Order(BaseModel):
    id: str
    user_id: int
    items: List[OrderItem]
    total: float
    status: OrderStatus
    created_at: datetime

# Storage
orders_db = {}

# Service clients with Circuit Breaker
class ServiceClient:
    def __init__(self, base_url: str, timeout: float = 5.0):
        self.base_url = base_url
        self.timeout = timeout
        self._failures = 0
        self._circuit_open = False

    async def get(self, path: str) -> dict:
        if self._circuit_open:
            raise HTTPException(503, "Circuit breaker is open")

        try:
            async with httpx.AsyncClient() as client:
                response = await client.get(
                    f"{self.base_url}{path}",
                    timeout=self.timeout
                )
                response.raise_for_status()
                self._failures = 0
                return response.json()
        except Exception as e:
            self._failures += 1
            if self._failures >= 5:
                self._circuit_open = True
            raise HTTPException(503, f"Service unavailable: {e}")

user_client = ServiceClient(USER_SERVICE_URL)

# API Endpoints
@app.post("/orders", response_model=Order, status_code=201)
async def create_order(request: CreateOrderRequest, background_tasks: BackgroundTasks):
    # 1. Validate user exists (sync call)
    try:
        user = await user_client.get(f"/users/{request.user_id}")
    except HTTPException:
        raise HTTPException(400, "Invalid user")

    # 2. Calculate total
    total = sum(item.price * item.quantity for item in request.items)

    # 3. Create order
    order = Order(
        id=str(uuid.uuid4()),
        user_id=request.user_id,
        items=request.items,
        total=total,
        status=OrderStatus.PENDING,
        created_at=datetime.utcnow()
    )
    orders_db[order.id] = order

    # 4. Trigger payment asynchronously
    background_tasks.add_task(initiate_payment, order)

    return order

async def initiate_payment(order: Order):
    """Асинхронная инициация платежа"""
    try:
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{PAYMENT_SERVICE_URL}/payments",
                json={
                    "order_id": order.id,
                    "amount": order.total,
                    "user_id": order.user_id
                }
            )
            if response.status_code == 201:
                orders_db[order.id].status = OrderStatus.CONFIRMED
    except Exception as e:
        print(f"Payment initiation failed: {e}")

@app.get("/orders", response_model=List[Order])
async def get_orders(user_id: Optional[int] = None):
    orders = list(orders_db.values())
    if user_id:
        orders = [o for o in orders if o.user_id == user_id]
    return orders

@app.get("/orders/{order_id}", response_model=Order)
async def get_order(order_id: str):
    if order_id not in orders_db:
        raise HTTPException(404, "Order not found")
    return orders_db[order_id]

@app.patch("/orders/{order_id}/status")
async def update_order_status(order_id: str, status: OrderStatus):
    if order_id not in orders_db:
        raise HTTPException(404, "Order not found")

    orders_db[order_id].status = status
    return orders_db[order_id]

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "order-service"}
```

### Event-Driven Communication

```python
# services/notification-service/src/main.py
from fastapi import FastAPI
import aio_pika
import json
import asyncio
import os

app = FastAPI(title="Notification Service")

RABBITMQ_URL = os.getenv("RABBITMQ_URL", "amqp://guest:guest@rabbitmq:5672/")

class EventConsumer:
    def __init__(self, rabbitmq_url: str):
        self.rabbitmq_url = rabbitmq_url
        self.connection = None
        self.channel = None

    async def connect(self):
        self.connection = await aio_pika.connect_robust(self.rabbitmq_url)
        self.channel = await self.connection.channel()

    async def subscribe(self, queue_name: str, handler):
        queue = await self.channel.declare_queue(queue_name, durable=True)
        await queue.consume(handler)

consumer = EventConsumer(RABBITMQ_URL)

# Event Handlers
async def handle_user_created(message: aio_pika.IncomingMessage):
    async with message.process():
        event = json.loads(message.body)
        user = event["payload"]
        print(f"Sending welcome email to {user['email']}")
        # await send_email(user['email'], "Welcome!", "...")

async def handle_order_created(message: aio_pika.IncomingMessage):
    async with message.process():
        event = json.loads(message.body)
        order = event["payload"]
        print(f"Sending order confirmation for order {order['id']}")
        # await send_email(...)

async def handle_payment_completed(message: aio_pika.IncomingMessage):
    async with message.process():
        event = json.loads(message.body)
        payment = event["payload"]
        print(f"Sending payment receipt for order {payment['order_id']}")
        # await send_email(...)

@app.on_event("startup")
async def startup():
    await consumer.connect()
    await consumer.subscribe("user.created", handle_user_created)
    await consumer.subscribe("order.created", handle_order_created)
    await consumer.subscribe("payment.completed", handle_payment_completed)

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "notification-service"}
```

### API Gateway

```python
# api-gateway/main.py
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse
import httpx
import os

app = FastAPI(title="API Gateway")

# Service routing
SERVICES = {
    "users": os.getenv("USER_SERVICE_URL", "http://user-service:8000"),
    "orders": os.getenv("ORDER_SERVICE_URL", "http://order-service:8001"),
    "payments": os.getenv("PAYMENT_SERVICE_URL", "http://payment-service:8002"),
}

@app.api_route("/{service}/{path:path}", methods=["GET", "POST", "PUT", "PATCH", "DELETE"])
async def proxy(service: str, path: str, request: Request):
    """Проксирование запросов к микросервисам"""
    if service not in SERVICES:
        raise HTTPException(404, f"Service '{service}' not found")

    service_url = SERVICES[service]
    url = f"{service_url}/{path}"

    # Forward request
    async with httpx.AsyncClient() as client:
        try:
            response = await client.request(
                method=request.method,
                url=url,
                headers=dict(request.headers),
                content=await request.body(),
                params=dict(request.query_params),
                timeout=30.0
            )
            return JSONResponse(
                content=response.json(),
                status_code=response.status_code
            )
        except httpx.RequestError as e:
            raise HTTPException(503, f"Service unavailable: {e}")

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "api-gateway"}
```

### Docker Compose для локальной разработки

```yaml
# infrastructure/docker-compose.yaml
version: '3.8'

services:
  api-gateway:
    build: ../api-gateway
    ports:
      - "8080:8080"
    environment:
      - USER_SERVICE_URL=http://user-service:8000
      - ORDER_SERVICE_URL=http://order-service:8001
      - PAYMENT_SERVICE_URL=http://payment-service:8002
    depends_on:
      - user-service
      - order-service
      - payment-service

  user-service:
    build: ../services/user-service
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@user-db:5432/users
      - RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672/
    depends_on:
      - user-db
      - rabbitmq

  order-service:
    build: ../services/order-service
    ports:
      - "8001:8001"
    environment:
      - DATABASE_URL=mongodb://order-db:27017/orders
      - USER_SERVICE_URL=http://user-service:8000
      - PAYMENT_SERVICE_URL=http://payment-service:8002
    depends_on:
      - order-db
      - user-service

  payment-service:
    build: ../services/payment-service
    ports:
      - "8002:8002"
    environment:
      - DATABASE_URL=postgresql://user:pass@payment-db:5432/payments
      - RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672/
    depends_on:
      - payment-db
      - rabbitmq

  notification-service:
    build: ../services/notification-service
    environment:
      - RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672/
    depends_on:
      - rabbitmq

  # Databases
  user-db:
    image: postgres:15
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=users

  order-db:
    image: mongo:6

  payment-db:
    image: postgres:15
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=payments

  # Infrastructure
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "15672:15672"

  # Observability
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
```

## Best practices и антипаттерны

### Best Practices

#### 1. Design for Failure
```python
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=30)
async def call_external_service(url: str):
    async with httpx.AsyncClient() as client:
        response = await client.get(url, timeout=5.0)
        return response.json()

# Retry with exponential backoff
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10)
)
async def call_with_retry(url: str):
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        response.raise_for_status()
        return response.json()
```

#### 2. Distributed Tracing
```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.exporter.jaeger.thrift import JaegerExporter

# Setup
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger",
    agent_port=6831,
)

@app.post("/orders")
async def create_order(request: CreateOrderRequest):
    with tracer.start_as_current_span("create_order") as span:
        span.set_attribute("user_id", request.user_id)

        with tracer.start_as_current_span("validate_user"):
            user = await user_client.get(f"/users/{request.user_id}")

        with tracer.start_as_current_span("save_order"):
            order = await save_order(request)

        return order
```

#### 3. API Versioning
```python
# URL versioning
app_v1 = FastAPI(prefix="/v1")
app_v2 = FastAPI(prefix="/v2")

@app_v1.get("/users/{user_id}")
async def get_user_v1(user_id: int):
    return {"id": user_id, "name": "John"}

@app_v2.get("/users/{user_id}")
async def get_user_v2(user_id: int):
    return {"user_id": user_id, "full_name": "John Doe", "metadata": {}}
```

### Антипаттерны

#### 1. Distributed Monolith
```python
# ПЛОХО: сервисы тесно связаны
class OrderService:
    async def create_order(self, request):
        # Синхронные вызовы ко многим сервисам
        user = await self.user_service.get_user(request.user_id)
        products = await self.product_service.get_products(request.product_ids)
        inventory = await self.inventory_service.check_availability(products)
        pricing = await self.pricing_service.calculate_price(products)
        # Если любой сервис недоступен — всё ломается

# ХОРОШО: минимальные синхронные зависимости
class OrderService:
    async def create_order(self, request):
        # Только необходимая валидация
        user = await self.user_service.validate_user(request.user_id)

        # Остальное через события
        order = await self.save_order(request)
        await self.publish_event("order.created", order)
        return order
```

#### 2. Shared Database
```
ПЛОХО: Shared database between services
┌──────────┐  ┌──────────┐  ┌──────────┐
│  User    │  │  Order   │  │ Payment  │
│  Service │  │  Service │  │ Service  │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │
     └─────────────┼─────────────┘
                   ▼
            ┌──────────────┐
            │   Shared DB  │  ← Coupling!
            └──────────────┘

ХОРОШО: Database per service
┌──────────┐  ┌──────────┐  ┌──────────┐
│  User    │  │  Order   │  │ Payment  │
│  Service │  │  Service │  │ Service  │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │
     ▼             ▼             ▼
┌────────┐   ┌────────┐   ┌────────┐
│User DB │   │Order DB│   │Pay DB  │
└────────┘   └────────┘   └────────┘
```

#### 3. Nano services (слишком мелкие)
```
ПЛОХО: Слишком мелкие сервисы
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│Validate│ │ Create │ │ Save   │ │ Notify │
│ User   │ │ Order  │ │ Order  │ │ User   │
└────────┘ └────────┘ └────────┘ └────────┘

ХОРОШО: Сервис = bounded context
┌─────────────────────────────────────────┐
│              Order Service              │
│  (validate, create, save, update, ...)  │
└─────────────────────────────────────────┘
```

## Связанные паттерны

| Паттерн | Описание |
|---------|----------|
| **API Gateway** | Единая точка входа |
| **Service Mesh** | Инфраструктурный слой для коммуникации |
| **Circuit Breaker** | Защита от каскадных сбоев |
| **Saga Pattern** | Распределённые транзакции |
| **CQRS** | Разделение read/write |
| **Event Sourcing** | Хранение событий |
| **Sidecar Pattern** | Вспомогательные функции в отдельном контейнере |

## Ресурсы для изучения

### Книги
- **"Building Microservices"** — Sam Newman
- **"Microservices Patterns"** — Chris Richardson
- **"Domain-Driven Design"** — Eric Evans

### Статьи
- [Microservices — Martin Fowler](https://martinfowler.com/articles/microservices.html)
- [12-Factor App](https://12factor.net/)
- [microservices.io](https://microservices.io/)

### Курсы и видео
- "Microservices Architecture" — GOTO Conferences
- "Decomposing Monolith" — Sam Newman

---

## Резюме

Микросервисная архитектура — это мощный подход для масштабируемых систем:

- **Ключевые принципы**: независимый deployment, database per service, lightweight communication
- **Преимущества**: масштабируемость, resilience, team autonomy
- **Сложности**: distributed complexity, operational overhead
- **Подходит для**: больших команд, высоконагруженных систем

**Правило**: не начинайте с микросервисов. Используйте "Monolith First" и разделяйте, когда это действительно нужно.
