# Microservices Architecture

## Введение

**Микросервисная архитектура** — это подход к разработке программного обеспечения, при котором приложение строится как набор небольших, независимых сервисов. Каждый сервис выполняет одну бизнес-функцию, работает в своём процессе и общается с другими через легковесные протоколы (HTTP/REST, gRPC, сообщения).

## Монолит vs Микросервисы

### Архитектурное сравнение

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              МОНОЛИТ                                        │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                         Единое приложение                            │  │
│   │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │  │
│   │  │  Users  │  │ Orders  │  │ Payment │  │Inventory│  │ Reports │   │  │
│   │  │ Module  │  │ Module  │  │ Module  │  │ Module  │  │ Module  │   │  │
│   │  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘   │  │
│   │       └────────────┴───────────┴────────────┴────────────┘         │  │
│   │                              │                                      │  │
│   │                     ┌────────▼────────┐                            │  │
│   │                     │   Общая База    │                            │  │
│   │                     │     Данных      │                            │  │
│   │                     └─────────────────┘                            │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│                           МИКРОСЕРВИСЫ                                      │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   │
│   │    User     │   │   Order     │   │   Payment   │   │  Inventory  │   │
│   │   Service   │   │   Service   │   │   Service   │   │   Service   │   │
│   │  ┌───────┐  │   │  ┌───────┐  │   │  ┌───────┐  │   │  ┌───────┐  │   │
│   │  │  API  │  │   │  │  API  │  │   │  │  API  │  │   │  │  API  │  │   │
│   │  └───────┘  │   │  └───────┘  │   │  └───────┘  │   │  └───────┘  │   │
│   │  ┌───────┐  │   │  ┌───────┐  │   │  ┌───────┐  │   │  ┌───────┐  │   │
│   │  │  DB   │  │   │  │  DB   │  │   │  │  DB   │  │   │  │  DB   │  │   │
│   │  └───────┘  │   │  └───────┘  │   │  └───────┘  │   │  └───────┘  │   │
│   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   │
│          │                 │                 │                 │          │
│          └─────────────────┼─────────────────┼─────────────────┘          │
│                            │                 │                             │
│                    ┌───────▼─────────────────▼───────┐                    │
│                    │      Message Broker / API GW     │                    │
│                    └─────────────────────────────────┘                    │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Сравнительная таблица

| Характеристика | Монолит | Микросервисы |
|----------------|---------|--------------|
| Развёртывание | Всё сразу | Независимое |
| Масштабирование | Вертикальное | Горизонтальное |
| Технологии | Единый стек | Полиглот |
| База данных | Общая | Изолированные |
| Команда | Централизованная | Автономные |
| Сложность | Низкая (изначально) | Высокая |
| Отказоустойчивость | Низкая | Высокая |

---

## Принципы микросервисной архитектуры

### 1. Single Responsibility (Единая ответственность)

Каждый сервис выполняет одну бизнес-функцию.

```python
# ❌ Плохо: сервис делает слишком много
class EcommerceService:
    def create_user(self, data): ...
    def process_order(self, order): ...
    def charge_payment(self, payment): ...
    def update_inventory(self, items): ...
    def send_notification(self, message): ...

# ✅ Хорошо: каждый сервис — одна ответственность

# user_service.py
class UserService:
    def create_user(self, data): ...
    def get_user(self, user_id): ...
    def update_user(self, user_id, data): ...

# order_service.py
class OrderService:
    def create_order(self, order_data): ...
    def get_order(self, order_id): ...
    def cancel_order(self, order_id): ...

# payment_service.py
class PaymentService:
    def process_payment(self, payment_data): ...
    def refund(self, payment_id): ...
```

### 2. Database per Service (Своя база для каждого сервиса)

```python
# Каждый сервис владеет своими данными

# User Service — PostgreSQL
class UserRepository:
    def __init__(self):
        self.db = PostgresConnection('users_db')

    async def save(self, user: User):
        await self.db.execute(
            "INSERT INTO users (id, email, name) VALUES ($1, $2, $3)",
            user.id, user.email, user.name
        )

# Order Service — MongoDB
class OrderRepository:
    def __init__(self):
        self.db = MongoClient('orders_db')

    async def save(self, order: Order):
        await self.db.orders.insert_one(order.dict())

# Inventory Service — Redis
class InventoryRepository:
    def __init__(self):
        self.redis = Redis('inventory_db')

    async def reserve(self, product_id: str, quantity: int):
        await self.redis.decrby(f'stock:{product_id}', quantity)
```

### 3. API First Design

```python
# Сначала определяем контракт API

# openapi.yaml
"""
openapi: 3.0.0
info:
  title: Order Service API
  version: 1.0.0

paths:
  /orders:
    post:
      summary: Create new order
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        '201':
          description: Order created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'

components:
  schemas:
    CreateOrderRequest:
      type: object
      required:
        - customer_id
        - items
      properties:
        customer_id:
          type: string
        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItem'
"""

# Затем реализуем
from pydantic import BaseModel
from typing import List

class OrderItem(BaseModel):
    product_id: str
    quantity: int
    price: float

class CreateOrderRequest(BaseModel):
    customer_id: str
    items: List[OrderItem]

class Order(BaseModel):
    id: str
    customer_id: str
    items: List[OrderItem]
    total: float
    status: str
    created_at: datetime
```

---

## Паттерны коммуникации

### 1. Synchronous Communication (REST/gRPC)

```python
# REST клиент
import httpx

class UserServiceClient:
    def __init__(self, base_url: str):
        self.base_url = base_url

    async def get_user(self, user_id: str) -> dict:
        async with httpx.AsyncClient() as client:
            response = await client.get(
                f'{self.base_url}/users/{user_id}',
                timeout=5.0
            )
            response.raise_for_status()
            return response.json()

# gRPC клиент (более эффективный)
import grpc
from user_pb2_grpc import UserServiceStub
from user_pb2 import GetUserRequest

class UserGrpcClient:
    def __init__(self, address: str):
        self.channel = grpc.aio.insecure_channel(address)
        self.stub = UserServiceStub(self.channel)

    async def get_user(self, user_id: str):
        request = GetUserRequest(user_id=user_id)
        return await self.stub.GetUser(request)
```

### 2. Asynchronous Communication (Message Broker)

```python
# Публикация события
from aiokafka import AIOKafkaProducer
import json

class EventPublisher:
    def __init__(self, bootstrap_servers: str):
        self.producer = AIOKafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode()
        )

    async def publish(self, topic: str, event: dict):
        await self.producer.send_and_wait(topic, event)

# Order Service публикует событие
class OrderService:
    def __init__(self, repository, publisher: EventPublisher):
        self.repository = repository
        self.publisher = publisher

    async def create_order(self, data: CreateOrderRequest) -> Order:
        order = Order(
            id=str(uuid.uuid4()),
            customer_id=data.customer_id,
            items=data.items,
            total=sum(item.price * item.quantity for item in data.items),
            status='pending',
            created_at=datetime.utcnow()
        )

        await self.repository.save(order)

        # Публикуем событие для других сервисов
        await self.publisher.publish('orders', {
            'event_type': 'order.created',
            'order_id': order.id,
            'customer_id': order.customer_id,
            'items': [item.dict() for item in order.items],
            'total': order.total
        })

        return order

# Payment Service подписывается на событие
from aiokafka import AIOKafkaConsumer

class PaymentEventHandler:
    def __init__(self, payment_service, bootstrap_servers: str):
        self.payment_service = payment_service
        self.consumer = AIOKafkaConsumer(
            'orders',
            bootstrap_servers=bootstrap_servers,
            group_id='payment-service',
            value_deserializer=lambda v: json.loads(v.decode())
        )

    async def start(self):
        await self.consumer.start()
        async for message in self.consumer:
            event = message.value
            if event['event_type'] == 'order.created':
                await self.payment_service.process_order_payment(
                    order_id=event['order_id'],
                    amount=event['total']
                )
```

### 3. Service Mesh Pattern

```yaml
# Istio VirtualService для traffic management
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
    - order-service
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: order-service
            subset: canary
          weight: 100
    - route:
        - destination:
            host: order-service
            subset: stable
          weight: 100

---
# DestinationRule для circuit breaker
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

---

## Паттерны проектирования микросервисов

### 1. API Gateway Pattern

```python
# Централизованная точка входа

from fastapi import FastAPI, Request
import httpx

app = FastAPI()

SERVICES = {
    'users': 'http://user-service:8080',
    'orders': 'http://order-service:8080',
    'payments': 'http://payment-service:8080'
}

@app.api_route('/api/{service}/{path:path}', methods=['GET', 'POST', 'PUT', 'DELETE'])
async def gateway(request: Request, service: str, path: str):
    if service not in SERVICES:
        return {'error': 'Service not found'}, 404

    upstream = SERVICES[service]

    async with httpx.AsyncClient() as client:
        response = await client.request(
            method=request.method,
            url=f'{upstream}/{path}',
            headers={
                'X-Request-ID': request.headers.get('X-Request-ID'),
                'Authorization': request.headers.get('Authorization')
            },
            content=await request.body()
        )

    return Response(
        content=response.content,
        status_code=response.status_code
    )
```

### 2. Service Discovery

```python
# Consul-based service discovery

import consul
import random

class ServiceRegistry:
    def __init__(self, consul_host: str):
        self.consul = consul.Consul(host=consul_host)

    def register(self, name: str, port: int, health_check: str):
        """Регистрирует сервис в Consul."""
        self.consul.agent.service.register(
            name=name,
            service_id=f'{name}-{uuid.uuid4()}',
            port=port,
            check=consul.Check.http(
                url=health_check,
                interval='10s',
                timeout='5s'
            )
        )

    def discover(self, name: str) -> str:
        """Находит здоровый инстанс сервиса."""
        _, services = self.consul.health.service(name, passing=True)

        if not services:
            raise ServiceNotFoundError(f'No healthy instances of {name}')

        # Простой round-robin
        service = random.choice(services)
        address = service['Service']['Address']
        port = service['Service']['Port']

        return f'http://{address}:{port}'

class ServiceClient:
    def __init__(self, registry: ServiceRegistry):
        self.registry = registry

    async def call(self, service_name: str, path: str, **kwargs):
        base_url = self.registry.discover(service_name)
        async with httpx.AsyncClient() as client:
            return await client.request(
                url=f'{base_url}{path}',
                **kwargs
            )
```

### 3. Circuit Breaker Pattern

```python
import asyncio
from enum import Enum
from dataclasses import dataclass, field
from typing import Callable, Any
import time

class CircuitState(Enum):
    CLOSED = 'closed'
    OPEN = 'open'
    HALF_OPEN = 'half_open'

@dataclass
class CircuitBreaker:
    name: str
    failure_threshold: int = 5
    recovery_timeout: int = 30
    success_threshold: int = 2

    state: CircuitState = field(default=CircuitState.CLOSED)
    failures: int = field(default=0)
    successes: int = field(default=0)
    last_failure_time: float = field(default=0)

    async def execute(self, func: Callable, *args, **kwargs) -> Any:
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time >= self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                self.successes = 0
            else:
                raise CircuitBreakerOpen(f'{self.name} is open')

        try:
            result = await func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        if self.state == CircuitState.HALF_OPEN:
            self.successes += 1
            if self.successes >= self.success_threshold:
                self.state = CircuitState.CLOSED
                self.failures = 0
        else:
            self.failures = 0

    def _on_failure(self):
        self.failures += 1
        self.last_failure_time = time.time()

        if self.failures >= self.failure_threshold:
            self.state = CircuitState.OPEN

# Использование
class OrderService:
    def __init__(self, payment_client, user_client):
        self.payment_client = payment_client
        self.user_client = user_client
        self.payment_circuit = CircuitBreaker('payment-service')
        self.user_circuit = CircuitBreaker('user-service')

    async def create_order(self, data):
        # Получаем данные пользователя с circuit breaker
        user = await self.user_circuit.execute(
            self.user_client.get_user,
            data.customer_id
        )

        order = Order(...)

        # Обрабатываем оплату с circuit breaker
        try:
            payment = await self.payment_circuit.execute(
                self.payment_client.process_payment,
                order.id,
                order.total
            )
        except CircuitBreakerOpen:
            # Fallback: отложенная оплата
            order.status = 'pending_payment'
            await self.queue_payment(order)

        return order
```

### 4. Saga Pattern (Distributed Transactions)

```python
# Orchestration-based Saga

from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import List, Optional
from enum import Enum

class SagaStepStatus(Enum):
    PENDING = 'pending'
    COMPLETED = 'completed'
    FAILED = 'failed'
    COMPENSATED = 'compensated'

@dataclass
class SagaStep:
    name: str
    action: Callable
    compensation: Callable
    status: SagaStepStatus = SagaStepStatus.PENDING

class OrderSaga:
    """
    Saga для создания заказа:
    1. Reserve Inventory
    2. Process Payment
    3. Create Shipment
    4. Send Notification
    """

    def __init__(
        self,
        inventory_service,
        payment_service,
        shipping_service,
        notification_service
    ):
        self.inventory = inventory_service
        self.payment = payment_service
        self.shipping = shipping_service
        self.notification = notification_service

    async def execute(self, order: Order) -> bool:
        steps: List[SagaStep] = [
            SagaStep(
                name='reserve_inventory',
                action=lambda: self.inventory.reserve(order.items),
                compensation=lambda: self.inventory.release(order.items)
            ),
            SagaStep(
                name='process_payment',
                action=lambda: self.payment.charge(order.customer_id, order.total),
                compensation=lambda: self.payment.refund(order.payment_id)
            ),
            SagaStep(
                name='create_shipment',
                action=lambda: self.shipping.create(order.id, order.address),
                compensation=lambda: self.shipping.cancel(order.shipment_id)
            ),
            SagaStep(
                name='send_notification',
                action=lambda: self.notification.send(order.customer_id, 'Order confirmed'),
                compensation=lambda: None  # Нечего компенсировать
            )
        ]

        completed_steps = []

        for step in steps:
            try:
                await step.action()
                step.status = SagaStepStatus.COMPLETED
                completed_steps.append(step)
            except Exception as e:
                step.status = SagaStepStatus.FAILED

                # Компенсируем все выполненные шаги в обратном порядке
                for completed in reversed(completed_steps):
                    try:
                        await completed.compensation()
                        completed.status = SagaStepStatus.COMPENSATED
                    except Exception as comp_error:
                        # Логируем ошибку компенсации
                        logger.error(f'Compensation failed: {completed.name}')

                return False

        return True

# Choreography-based Saga (через события)
class OrderCreatedHandler:
    async def handle(self, event: OrderCreatedEvent):
        # Inventory Service подписывается на order.created
        try:
            await self.inventory.reserve(event.items)
            await self.publish(InventoryReservedEvent(order_id=event.order_id))
        except Exception:
            await self.publish(InventoryReservationFailedEvent(order_id=event.order_id))

class InventoryReservedHandler:
    async def handle(self, event: InventoryReservedEvent):
        # Payment Service подписывается на inventory.reserved
        try:
            await self.payment.charge(event.order_id)
            await self.publish(PaymentProcessedEvent(order_id=event.order_id))
        except Exception:
            await self.publish(PaymentFailedEvent(order_id=event.order_id))

class PaymentFailedHandler:
    async def handle(self, event: PaymentFailedEvent):
        # Компенсация: освобождаем резерв
        await self.inventory.release(event.order_id)
```

### 5. CQRS Pattern

```python
# Command side (Write)
class OrderCommandHandler:
    def __init__(self, event_store: EventStore):
        self.event_store = event_store

    async def handle_create_order(self, cmd: CreateOrderCommand):
        # Восстанавливаем агрегат из событий
        events = await self.event_store.get_events(f'order-{cmd.order_id}')
        order = Order()
        order.load_from_history(events)

        # Применяем команду
        order.create(cmd.customer_id, cmd.items)

        # Сохраняем новые события
        await self.event_store.append(
            stream_id=f'order-{order.id}',
            events=order.uncommitted_events,
            expected_version=order.version
        )

# Query side (Read)
class OrderQueryHandler:
    def __init__(self, read_db):
        self.read_db = read_db

    async def get_order(self, order_id: str) -> OrderReadModel:
        # Читаем из оптимизированной read model
        return await self.read_db.orders.find_one({'_id': order_id})

    async def get_customer_orders(
        self,
        customer_id: str,
        page: int = 1,
        limit: int = 20
    ) -> List[OrderReadModel]:
        return await self.read_db.orders.find(
            {'customer_id': customer_id}
        ).sort('created_at', -1).skip((page - 1) * limit).limit(limit).to_list()

# Projector (синхронизация read model)
class OrderProjector:
    def __init__(self, read_db):
        self.read_db = read_db

    async def project_order_created(self, event: OrderCreatedEvent):
        await self.read_db.orders.insert_one({
            '_id': event.order_id,
            'customer_id': event.customer_id,
            'items': event.items,
            'total': event.total,
            'status': 'pending',
            'created_at': event.timestamp
        })

    async def project_order_shipped(self, event: OrderShippedEvent):
        await self.read_db.orders.update_one(
            {'_id': event.order_id},
            {'$set': {'status': 'shipped', 'shipped_at': event.timestamp}}
        )
```

---

## Deployment Patterns

### 1. Container per Service

```yaml
# docker-compose.yml
version: '3.8'

services:
  user-service:
    build: ./user-service
    ports:
      - "8081:8080"
    environment:
      - DATABASE_URL=postgres://user-db:5432/users
      - KAFKA_BROKERS=kafka:9092
    depends_on:
      - user-db
      - kafka

  order-service:
    build: ./order-service
    ports:
      - "8082:8080"
    environment:
      - DATABASE_URL=mongodb://order-db:27017/orders
      - KAFKA_BROKERS=kafka:9092
      - USER_SERVICE_URL=http://user-service:8080
    depends_on:
      - order-db
      - kafka
      - user-service

  payment-service:
    build: ./payment-service
    ports:
      - "8083:8080"
    environment:
      - DATABASE_URL=postgres://payment-db:5432/payments
      - KAFKA_BROKERS=kafka:9092
    depends_on:
      - payment-db
      - kafka

  user-db:
    image: postgres:15
    environment:
      - POSTGRES_DB=users
      - POSTGRES_PASSWORD=secret

  order-db:
    image: mongo:6

  payment-db:
    image: postgres:15
    environment:
      - POSTGRES_DB=payments
      - POSTGRES_PASSWORD=secret

  kafka:
    image: confluentinc/cp-kafka:latest
    environment:
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
```

### 2. Kubernetes Deployment

```yaml
# user-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  labels:
    app: user-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user-service
          image: myregistry/user-service:v1.2.0
          ports:
            - containerPort: 8080
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: user-db-secret
                  key: url
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## Best Practices

### 1. Health Checks

```python
from fastapi import FastAPI
from enum import Enum

class HealthStatus(Enum):
    HEALTHY = 'healthy'
    UNHEALTHY = 'unhealthy'
    DEGRADED = 'degraded'

app = FastAPI()

@app.get('/health/live')
async def liveness():
    """Kubernetes liveness probe — приложение живо?"""
    return {'status': 'ok'}

@app.get('/health/ready')
async def readiness():
    """Kubernetes readiness probe — готово принимать трафик?"""
    checks = {
        'database': await check_database(),
        'kafka': await check_kafka(),
        'redis': await check_redis()
    }

    all_healthy = all(c['status'] == HealthStatus.HEALTHY for c in checks.values())

    if not all_healthy:
        return JSONResponse(
            status_code=503,
            content={'status': 'not ready', 'checks': checks}
        )

    return {'status': 'ready', 'checks': checks}

async def check_database():
    try:
        await db.execute('SELECT 1')
        return {'status': HealthStatus.HEALTHY}
    except Exception as e:
        return {'status': HealthStatus.UNHEALTHY, 'error': str(e)}
```

### 2. Distributed Tracing

```python
from opentelemetry import trace
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Настройка трассировки
trace.set_tracer_provider(TracerProvider())
jaeger_exporter = JaegerExporter(
    agent_host_name='jaeger',
    agent_port=6831
)
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)

tracer = trace.get_tracer(__name__)

class OrderService:
    async def create_order(self, data):
        with tracer.start_as_current_span('create_order') as span:
            span.set_attribute('customer_id', data.customer_id)

            # Вызов другого сервиса — трейс передаётся автоматически
            with tracer.start_as_current_span('get_user'):
                user = await self.user_client.get_user(data.customer_id)

            with tracer.start_as_current_span('save_order'):
                order = await self.repository.save(data)

            span.set_attribute('order_id', order.id)
            return order
```

### 3. Centralized Configuration

```python
# Использование Consul для конфигурации

import consul
from pydantic import BaseSettings

class Settings(BaseSettings):
    service_name: str = 'order-service'
    consul_host: str = 'localhost'

    class Config:
        env_prefix = 'APP_'

class ConfigManager:
    def __init__(self, settings: Settings):
        self.consul = consul.Consul(host=settings.consul_host)
        self.prefix = f'config/{settings.service_name}/'
        self._cache = {}

    def get(self, key: str, default=None):
        full_key = f'{self.prefix}{key}'

        if full_key in self._cache:
            return self._cache[full_key]

        _, data = self.consul.kv.get(full_key)
        if data:
            value = data['Value'].decode()
            self._cache[full_key] = value
            return value

        return default

    def watch(self, key: str, callback):
        """Следит за изменениями конфигурации."""
        index = None
        while True:
            index, data = self.consul.kv.get(
                f'{self.prefix}{key}',
                index=index,
                wait='5m'
            )
            if data:
                callback(data['Value'].decode())
```

---

## Типичные ошибки

### 1. Distributed Monolith

```python
# ❌ Плохо: сервисы сильно связаны
class OrderService:
    async def create_order(self, data):
        # Синхронные вызовы ко всем сервисам
        user = await self.user_service.get_user(data.customer_id)
        inventory = await self.inventory_service.check(data.items)
        payment = await self.payment_service.charge(user, data.total)
        shipping = await self.shipping_service.create(data)
        notification = await self.notification_service.send(user)
        # Если один сервис упадёт — всё сломается

# ✅ Хорошо: асинхронная коммуникация
class OrderService:
    async def create_order(self, data):
        order = Order(...)
        await self.repository.save(order)
        # Публикуем событие — другие сервисы обработают асинхронно
        await self.publisher.publish('order.created', order)
        return order
```

### 2. Shared Database

```python
# ❌ Плохо: общая база данных
# User Service и Order Service обращаются к одной БД
# Изменения в схеме ломают оба сервиса

# ✅ Хорошо: своя база для каждого сервиса
# Order Service хранит только order_id и customer_id (не всю информацию о пользователе)
# Если нужны данные пользователя — запрашивает через API или события
```

### 3. No Backward Compatibility

```python
# ❌ Плохо: ломающие изменения API
# v1: GET /users/{id} -> {"id": "123", "name": "John"}
# v2: GET /users/{id} -> {"user_id": "123", "full_name": "John"} # Сломано!

# ✅ Хорошо: обратная совместимость
# v1: GET /users/{id} -> {"id": "123", "name": "John"}
# v2: GET /users/{id} -> {"id": "123", "name": "John", "user_id": "123", "full_name": "John"}
# Старые поля сохранены + новые добавлены
```

---

## Когда использовать микросервисы

### Подходит для:

- Большие команды (>10 разработчиков)
- Сложные домены с чёткими границами
- Необходимость независимого масштабирования
- Разные требования к технологиям
- Частые релизы отдельных компонентов

### НЕ подходит для:

- Стартапы на ранней стадии
- Маленькие команды (<5 человек)
- Простые приложения
- Прототипы и MVP
- Когда нет опыта в DevOps

---

## Заключение

Микросервисная архитектура — мощный инструмент для построения масштабируемых систем, но требует:

1. **Зрелой инфраструктуры** — CI/CD, мониторинг, оркестрация
2. **Опытной команды** — понимание распределённых систем
3. **Правильного проектирования** — чёткие границы сервисов
4. **Культуры DevOps** — автоматизация всего

Начинайте с монолита и выделяйте микросервисы по мере роста системы и команды.
