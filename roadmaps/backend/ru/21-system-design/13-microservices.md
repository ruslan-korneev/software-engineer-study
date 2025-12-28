# Microservices

[prev: 12-application-layer](./12-application-layer.md) | [next: 14-service-discovery](./14-service-discovery.md)

---

## Что такое микросервисы

**Микросервисная архитектура** — это подход к разработке программного обеспечения, при котором приложение строится как набор небольших, независимых сервисов, каждый из которых:

- Выполняет одну бизнес-функцию
- Работает в собственном процессе
- Общается с другими сервисами через легковесные протоколы (HTTP/REST, gRPC, сообщения)
- Может быть развёрнут независимо от других сервисов
- Может быть написан на разных языках программирования

### Ключевые характеристики

| Характеристика | Описание |
|----------------|----------|
| **Независимость** | Каждый сервис может разрабатываться, тестироваться и деплоиться отдельно |
| **Ограниченный контекст** | Сервис владеет своими данными и бизнес-логикой |
| **Автономность** | Сбой одного сервиса не должен приводить к падению всей системы |
| **Децентрализация** | Нет единой точки управления данными или логикой |
| **Технологическая гибкость** | Разные сервисы могут использовать разные технологии |

### Пример структуры микросервисного приложения

```
E-commerce Platform
├── User Service          # Управление пользователями
├── Product Service       # Каталог товаров
├── Order Service         # Обработка заказов
├── Payment Service       # Платежи
├── Notification Service  # Уведомления (email, SMS, push)
├── Inventory Service     # Управление складом
├── Shipping Service      # Доставка
└── Analytics Service     # Аналитика и отчёты
```

---

## Сравнение с монолитной архитектурой

### Монолитная архитектура

```
┌─────────────────────────────────────────────┐
│              Monolithic Application         │
│  ┌─────────┬─────────┬─────────┬─────────┐  │
│  │  Users  │ Products│ Orders  │ Payments│  │
│  └─────────┴─────────┴─────────┴─────────┘  │
│  ┌─────────────────────────────────────────┐│
│  │         Shared Database                 ││
│  └─────────────────────────────────────────┘│
└─────────────────────────────────────────────┘
```

### Микросервисная архитектура

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  Users   │   │ Products │   │  Orders  │   │ Payments │
│ Service  │   │ Service  │   │ Service  │   │ Service  │
├──────────┤   ├──────────┤   ├──────────┤   ├──────────┤
│  Users   │   │ Products │   │  Orders  │   │ Payments │
│   DB     │   │   DB     │   │   DB     │   │   DB     │
└──────────┘   └──────────┘   └──────────┘   └──────────┘
      ↑              ↑              ↑              ↑
      └──────────────┴──────────────┴──────────────┘
                    API Gateway
```

### Сравнительная таблица

| Критерий | Монолит | Микросервисы |
|----------|---------|--------------|
| **Развёртывание** | Всё приложение целиком | Каждый сервис отдельно |
| **Масштабирование** | Вертикальное (весь монолит) | Горизонтальное (отдельные сервисы) |
| **Технологический стек** | Единый для всего | Разный для каждого сервиса |
| **Время разработки** | Быстрый старт | Дольше на начальном этапе |
| **Сложность** | Растёт экспоненциально | Распределённая сложность |
| **Отказоустойчивость** | Одна ошибка = падение всего | Изоляция сбоев |
| **Команды** | Одна большая команда | Маленькие автономные команды |

### Плюсы микросервисов

1. **Независимый деплой** — изменения в одном сервисе не требуют передеплоя всего приложения
2. **Масштабируемость** — можно масштабировать только нагруженные сервисы
3. **Технологическая свобода** — выбор лучшего инструмента для каждой задачи
4. **Отказоустойчивость** — сбой одного сервиса не роняет всю систему
5. **Организационная гибкость** — маленькие команды работают над отдельными сервисами
6. **Быстрые релизы** — короткий цикл от разработки до продакшена

### Минусы микросервисов

1. **Сложность** — распределённая система сложнее монолита
2. **Сетевые задержки** — каждый вызов между сервисами = сетевой запрос
3. **Eventual consistency** — данные не всегда консистентны
4. **Операционные затраты** — нужны инструменты для мониторинга, логирования, деплоя
5. **Сложность отладки** — трейсинг запросов через множество сервисов
6. **Дублирование** — общий код может дублироваться между сервисами

---

## Принципы проектирования микросервисов

### 1. Single Responsibility Principle (SRP)

Каждый сервис должен отвечать за одну бизнес-функцию и иметь одну причину для изменения.

```python
# Плохо: Order Service делает слишком много
class OrderService:
    def create_order(self, order_data):
        self.validate_user(order_data.user_id)       # Ответственность User Service
        self.check_inventory(order_data.items)        # Ответственность Inventory Service
        self.process_payment(order_data.payment)      # Ответственность Payment Service
        self.send_notification(order_data.user_id)    # Ответственность Notification Service
        return self.save_order(order_data)

# Хорошо: Order Service только управляет заказами
class OrderService:
    def create_order(self, order_data):
        # Вызывает другие сервисы через API
        user = self.user_client.get_user(order_data.user_id)
        inventory = self.inventory_client.reserve(order_data.items)

        order = Order(
            user_id=user.id,
            items=order_data.items,
            status="pending"
        )
        return self.order_repository.save(order)
```

### 2. Loose Coupling (Слабая связанность)

Сервисы должны минимально зависеть друг от друга. Изменения в одном сервисе не должны требовать изменений в других.

```python
# Плохо: Жёсткая связь через прямые вызовы
class OrderService:
    def __init__(self):
        self.payment_service = PaymentService()  # Прямая зависимость

    def complete_order(self, order_id):
        order = self.get_order(order_id)
        self.payment_service.process(order)  # Синхронный вызов

# Хорошо: Слабая связь через сообщения
class OrderService:
    def __init__(self, message_broker):
        self.message_broker = message_broker

    def complete_order(self, order_id):
        order = self.get_order(order_id)
        # Публикуем событие, не знаем кто его обработает
        self.message_broker.publish("order.completed", {
            "order_id": order_id,
            "amount": order.total,
            "user_id": order.user_id
        })
```

### 3. High Cohesion (Высокая связность)

Все компоненты внутри сервиса должны быть тесно связаны и работать вместе для достижения одной цели.

```python
# Хорошо: Высокая связность — все компоненты работают с заказами
# order_service/
# ├── models/
# │   ├── order.py
# │   └── order_item.py
# ├── repositories/
# │   └── order_repository.py
# ├── services/
# │   ├── order_service.py
# │   └── order_validator.py
# ├── events/
# │   ├── order_created.py
# │   └── order_completed.py
# └── api/
#     └── order_controller.py
```

### 4. Domain-Driven Design (DDD)

Сервисы строятся вокруг бизнес-доменов (Bounded Contexts).

```
E-commerce Bounded Contexts:

┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│   Sales Context │   │Inventory Context│   │ Shipping Context│
├─────────────────┤   ├─────────────────┤   ├─────────────────┤
│ • Order         │   │ • Stock         │   │ • Shipment      │
│ • Cart          │   │ • Warehouse     │   │ • Carrier       │
│ • Discount      │   │ • Reservation   │   │ • Tracking      │
└─────────────────┘   └─────────────────┘   └─────────────────┘
```

### 5. Design for Failure

Сервисы должны быть готовы к сбоям других сервисов.

```python
import time
from functools import wraps

# Circuit Breaker паттерн
class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failures = 0
        self.last_failure_time = None
        self.state = "closed"  # closed, open, half-open

    def __call__(self, func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            if self.state == "open":
                if time.time() - self.last_failure_time > self.recovery_timeout:
                    self.state = "half-open"
                else:
                    raise CircuitBreakerOpen("Service unavailable")

            try:
                result = func(*args, **kwargs)
                self.failures = 0
                self.state = "closed"
                return result
            except Exception as e:
                self.failures += 1
                self.last_failure_time = time.time()
                if self.failures >= self.failure_threshold:
                    self.state = "open"
                raise e
        return wrapper

# Использование
circuit_breaker = CircuitBreaker(failure_threshold=3, recovery_timeout=60)

@circuit_breaker
def call_payment_service(order):
    response = requests.post(f"{PAYMENT_SERVICE_URL}/process", json=order)
    return response.json()
```

---

## Паттерны коммуникации

### Синхронная коммуникация

Клиент отправляет запрос и ждёт ответа.

#### REST (HTTP/JSON)

```python
# User Service API
from fastapi import FastAPI, HTTPException

app = FastAPI()

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    user = await user_repository.find_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return {"id": user.id, "name": user.name, "email": user.email}

# Order Service вызывает User Service
import httpx

async def get_user_info(user_id: int):
    async with httpx.AsyncClient() as client:
        response = await client.get(f"http://user-service/users/{user_id}")
        if response.status_code == 404:
            raise UserNotFound(user_id)
        return response.json()
```

#### gRPC

Более эффективный протокол с бинарной сериализацией (Protocol Buffers).

```protobuf
// user.proto
syntax = "proto3";

package user;

service UserService {
    rpc GetUser(GetUserRequest) returns (User);
    rpc CreateUser(CreateUserRequest) returns (User);
    rpc ListUsers(ListUsersRequest) returns (stream User);
}

message GetUserRequest {
    int64 user_id = 1;
}

message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
    string created_at = 4;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
}

message ListUsersRequest {
    int32 page = 1;
    int32 page_size = 2;
}
```

```python
# gRPC Server (User Service)
import grpc
from concurrent import futures
import user_pb2
import user_pb2_grpc

class UserServicer(user_pb2_grpc.UserServiceServicer):
    async def GetUser(self, request, context):
        user = await user_repository.find_by_id(request.user_id)
        if not user:
            context.abort(grpc.StatusCode.NOT_FOUND, "User not found")
        return user_pb2.User(
            id=user.id,
            name=user.name,
            email=user.email
        )

# gRPC Client (Order Service)
async def get_user_via_grpc(user_id: int):
    async with grpc.aio.insecure_channel('user-service:50051') as channel:
        stub = user_pb2_grpc.UserServiceStub(channel)
        response = await stub.GetUser(user_pb2.GetUserRequest(user_id=user_id))
        return response
```

### Сравнение REST и gRPC

| Критерий | REST | gRPC |
|----------|------|------|
| Формат данных | JSON (текст) | Protocol Buffers (бинарный) |
| Производительность | Медленнее | Быстрее (в 7-10 раз) |
| Streaming | Ограниченный | Полноценный bidirectional |
| Типизация | Слабая | Строгая (схема) |
| Браузеры | Нативная поддержка | Требует gRPC-Web |
| Отладка | Легко (curl, Postman) | Сложнее |

### Асинхронная коммуникация (Message Queues)

Сервисы общаются через брокер сообщений, не ожидая немедленного ответа.

```
┌────────────┐     ┌──────────────┐     ┌────────────────┐
│   Order    │────►│  Message     │────►│  Notification  │
│   Service  │     │  Broker      │     │    Service     │
└────────────┘     │  (RabbitMQ/  │     └────────────────┘
                   │   Kafka)     │
                   └──────────────┘
                          │
                          ▼
                   ┌────────────────┐
                   │   Inventory    │
                   │    Service     │
                   └────────────────┘
```

#### RabbitMQ (AMQP)

```python
# Publisher (Order Service)
import aio_pika

async def publish_order_created(order):
    connection = await aio_pika.connect_robust("amqp://rabbitmq:5672")
    async with connection:
        channel = await connection.channel()
        exchange = await channel.declare_exchange(
            "orders",
            aio_pika.ExchangeType.TOPIC
        )

        message = aio_pika.Message(
            body=json.dumps({
                "event": "order.created",
                "order_id": order.id,
                "user_id": order.user_id,
                "items": [item.to_dict() for item in order.items],
                "timestamp": datetime.utcnow().isoformat()
            }).encode(),
            content_type="application/json"
        )

        await exchange.publish(message, routing_key="order.created")

# Consumer (Inventory Service)
async def consume_order_events():
    connection = await aio_pika.connect_robust("amqp://rabbitmq:5672")
    async with connection:
        channel = await connection.channel()
        queue = await channel.declare_queue("inventory_orders", durable=True)

        await queue.bind("orders", routing_key="order.created")

        async with queue.iterator() as queue_iter:
            async for message in queue_iter:
                async with message.process():
                    data = json.loads(message.body.decode())
                    await reserve_inventory(data["items"])
```

#### Apache Kafka

```python
# Producer (Order Service)
from aiokafka import AIOKafkaProducer

async def publish_order_event(order, event_type):
    producer = AIOKafkaProducer(bootstrap_servers='kafka:9092')
    await producer.start()
    try:
        event = {
            "event_type": event_type,
            "order_id": order.id,
            "data": order.to_dict(),
            "timestamp": datetime.utcnow().isoformat()
        }
        await producer.send_and_wait(
            "orders",
            key=str(order.id).encode(),
            value=json.dumps(event).encode()
        )
    finally:
        await producer.stop()

# Consumer (Analytics Service)
from aiokafka import AIOKafkaConsumer

async def consume_order_events():
    consumer = AIOKafkaConsumer(
        'orders',
        bootstrap_servers='kafka:9092',
        group_id='analytics-group',
        auto_offset_reset='earliest'
    )
    await consumer.start()
    try:
        async for message in consumer:
            event = json.loads(message.value.decode())
            await process_analytics(event)
    finally:
        await consumer.stop()
```

### Event-Driven Architecture

Сервисы реагируют на события, а не на прямые вызовы.

```python
# Domain Events
class OrderCreated:
    def __init__(self, order_id, user_id, items, total):
        self.order_id = order_id
        self.user_id = user_id
        self.items = items
        self.total = total
        self.timestamp = datetime.utcnow()

class PaymentProcessed:
    def __init__(self, order_id, payment_id, status):
        self.order_id = order_id
        self.payment_id = payment_id
        self.status = status
        self.timestamp = datetime.utcnow()

# Event Handlers в разных сервисах
class InventoryEventHandler:
    async def handle_order_created(self, event: OrderCreated):
        for item in event.items:
            await self.inventory_service.reserve(item.product_id, item.quantity)

class NotificationEventHandler:
    async def handle_payment_processed(self, event: PaymentProcessed):
        if event.status == "success":
            await self.email_service.send_confirmation(event.order_id)
        else:
            await self.email_service.send_payment_failed(event.order_id)
```

---

## Паттерны декомпозиции

### 1. Decomposition by Business Capability

Разделение по бизнес-функциям организации.

```
Бизнес-возможности компании:
├── Управление продуктами
│   └── Product Service
├── Управление заказами
│   └── Order Service
├── Управление клиентами
│   └── Customer Service
├── Платежи
│   └── Payment Service
├── Доставка
│   └── Shipping Service
└── Маркетинг
    ├── Promotion Service
    └── Recommendation Service
```

### 2. Decomposition by Subdomain (DDD)

Разделение на основе поддоменов из Domain-Driven Design.

```
E-commerce Domain:

Core Subdomains (конкурентное преимущество):
├── Order Management
├── Pricing & Promotions
└── Recommendation Engine

Supporting Subdomains (важные, но не уникальные):
├── Inventory Management
├── Shipping
└── Customer Support

Generic Subdomains (типовые решения):
├── User Authentication
├── Payment Processing (можно интегрировать Stripe)
└── Email Notifications
```

### 3. Strangler Fig Pattern

Постепенная миграция с монолита на микросервисы.

```
Этап 1: Монолит
┌─────────────────────────────────────┐
│            Monolith                 │
│  [Users] [Products] [Orders] [Pay]  │
└─────────────────────────────────────┘

Этап 2: Выделение первого сервиса
┌────────────────┐    ┌─────────────────────────────┐
│  User Service  │◄───│         Monolith            │
└────────────────┘    │  [Products] [Orders] [Pay]  │
                      └─────────────────────────────┘

Этап 3: Продолжение миграции
┌────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  User Service  │  │ Product Service │  │     Monolith    │
└────────────────┘  └─────────────────┘  │ [Orders] [Pay]  │
                                         └─────────────────┘

Этап 4: Полная декомпозиция
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│  Users  │  │Products │  │ Orders  │  │ Payment │
└─────────┘  └─────────┘  └─────────┘  └─────────┘
```

```python
# API Gateway routing during migration
from fastapi import FastAPI, Request
import httpx

app = FastAPI()

# Правила маршрутизации
ROUTING_RULES = {
    "/users": "http://user-service",       # Новый микросервис
    "/products": "http://product-service", # Новый микросервис
    "/orders": "http://legacy-monolith",   # Всё ещё в монолите
    "/payments": "http://legacy-monolith", # Всё ещё в монолите
}

@app.api_route("/{path:path}", methods=["GET", "POST", "PUT", "DELETE"])
async def route_request(request: Request, path: str):
    # Определяем куда направить запрос
    for prefix, service_url in ROUTING_RULES.items():
        if f"/{path}".startswith(prefix):
            target_url = f"{service_url}/{path}"
            break
    else:
        target_url = f"http://legacy-monolith/{path}"

    async with httpx.AsyncClient() as client:
        response = await client.request(
            method=request.method,
            url=target_url,
            headers=dict(request.headers),
            content=await request.body()
        )
        return Response(
            content=response.content,
            status_code=response.status_code,
            headers=dict(response.headers)
        )
```

---

## Data Management в микросервисах

### Database per Service

Каждый сервис владеет своей базой данных. Это обеспечивает:
- Независимость схемы данных
- Свободу выбора типа БД (PostgreSQL, MongoDB, Redis)
- Изоляцию данных

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Order     │     │   Product   │     │   User      │
│   Service   │     │   Service   │     │   Service   │
├─────────────┤     ├─────────────┤     ├─────────────┤
│ PostgreSQL  │     │ MongoDB     │     │ PostgreSQL  │
│   orders    │     │  products   │     │   users     │
└─────────────┘     └─────────────┘     └─────────────┘
```

### Проблема: Как сделать JOIN между сервисами?

```python
# Плохо: Прямой доступ к БД другого сервиса (НЕЛЬЗЯ!)
def get_order_with_user(order_id):
    order = Order.query.get(order_id)
    # Нельзя напрямую обращаться к таблице users!
    user = db.session.execute("SELECT * FROM users WHERE id = :id", {"id": order.user_id})

# Хорошо: API Composition
async def get_order_with_user(order_id):
    order = await order_repository.find_by_id(order_id)
    user = await user_service_client.get_user(order.user_id)

    return {
        "order": order.to_dict(),
        "user": {
            "id": user["id"],
            "name": user["name"],
            "email": user["email"]
        }
    }
```

### Saga Pattern

Паттерн для управления распределёнными транзакциями без использования 2PC.

#### Choreography-based Saga

Каждый сервис публикует события и слушает события других сервисов.

```
Создание заказа:

OrderService          InventoryService       PaymentService        ShippingService
     │                       │                     │                     │
     │  OrderCreated         │                     │                     │
     │──────────────────────►│                     │                     │
     │                       │                     │                     │
     │                       │ InventoryReserved   │                     │
     │                       │────────────────────►│                     │
     │                       │                     │                     │
     │                       │                     │ PaymentProcessed    │
     │                       │                     │────────────────────►│
     │                       │                     │                     │
     │ OrderCompleted        │                     │                     │
     │◄──────────────────────┴─────────────────────┴─────────────────────│
```

```python
# Order Service
class OrderService:
    async def create_order(self, order_data):
        order = Order(**order_data)
        order.status = "pending"
        await self.order_repository.save(order)

        # Публикуем событие
        await self.event_bus.publish(OrderCreated(
            order_id=order.id,
            items=order.items,
            total=order.total
        ))
        return order

    async def handle_payment_processed(self, event: PaymentProcessed):
        order = await self.order_repository.find_by_id(event.order_id)
        if event.status == "success":
            order.status = "paid"
        else:
            order.status = "payment_failed"
            # Запускаем компенсирующую транзакцию
            await self.event_bus.publish(OrderCancelled(order_id=order.id))
        await self.order_repository.save(order)

# Inventory Service
class InventoryService:
    async def handle_order_created(self, event: OrderCreated):
        try:
            for item in event.items:
                await self.reserve_item(item.product_id, item.quantity)
            await self.event_bus.publish(InventoryReserved(order_id=event.order_id))
        except InsufficientStock:
            await self.event_bus.publish(InventoryReservationFailed(order_id=event.order_id))

    async def handle_order_cancelled(self, event: OrderCancelled):
        # Компенсирующая транзакция — освобождаем резерв
        await self.release_reservation(event.order_id)
```

#### Orchestration-based Saga

Центральный оркестратор управляет последовательностью шагов.

```python
class OrderSagaOrchestrator:
    def __init__(self, inventory_client, payment_client, shipping_client):
        self.inventory_client = inventory_client
        self.payment_client = payment_client
        self.shipping_client = shipping_client

    async def execute(self, order: Order) -> SagaResult:
        saga_log = SagaLog(order_id=order.id)

        try:
            # Шаг 1: Резервирование товаров
            reservation = await self.inventory_client.reserve(order.items)
            saga_log.add_step("inventory_reserved", reservation.id)

            # Шаг 2: Обработка платежа
            payment = await self.payment_client.process(
                amount=order.total,
                user_id=order.user_id
            )
            saga_log.add_step("payment_processed", payment.id)

            # Шаг 3: Создание доставки
            shipment = await self.shipping_client.create(
                order_id=order.id,
                address=order.shipping_address
            )
            saga_log.add_step("shipment_created", shipment.id)

            return SagaResult(success=True, order_id=order.id)

        except Exception as e:
            # Компенсирующие транзакции в обратном порядке
            await self.compensate(saga_log)
            return SagaResult(success=False, error=str(e))

    async def compensate(self, saga_log: SagaLog):
        for step in reversed(saga_log.steps):
            if step.name == "shipment_created":
                await self.shipping_client.cancel(step.reference_id)
            elif step.name == "payment_processed":
                await self.payment_client.refund(step.reference_id)
            elif step.name == "inventory_reserved":
                await self.inventory_client.release(step.reference_id)
```

### CQRS (Command Query Responsibility Segregation)

Разделение операций чтения и записи.

```
                    ┌─────────────────┐
                    │    API Layer    │
                    └────────┬────────┘
                             │
           ┌─────────────────┴─────────────────┐
           │                                   │
           ▼                                   ▼
    ┌─────────────┐                    ┌─────────────┐
    │  Commands   │                    │   Queries   │
    │   (Write)   │                    │   (Read)    │
    └──────┬──────┘                    └──────┬──────┘
           │                                   │
           ▼                                   ▼
    ┌─────────────┐     Events      ┌─────────────┐
    │  Write DB   │ ───────────────►│   Read DB   │
    │ (Postgres)  │                 │   (Elastic) │
    └─────────────┘                 └─────────────┘
```

```python
# Commands (Write Model)
class CreateOrderCommand:
    user_id: int
    items: List[OrderItem]

class OrderCommandHandler:
    async def handle_create_order(self, command: CreateOrderCommand):
        order = Order(
            user_id=command.user_id,
            items=command.items,
            status="pending"
        )
        await self.write_repository.save(order)

        # Публикуем событие для синхронизации Read Model
        await self.event_bus.publish(OrderCreatedEvent(order))
        return order.id

# Queries (Read Model)
class OrderQueryService:
    def __init__(self, elasticsearch_client):
        self.es = elasticsearch_client

    async def search_orders(self, filters: OrderFilters):
        query = self.build_query(filters)
        result = await self.es.search(index="orders", body=query)
        return [hit["_source"] for hit in result["hits"]["hits"]]

    async def get_order_summary(self, user_id: int):
        # Денормализованные данные для быстрого чтения
        return await self.es.get(index="order_summaries", id=user_id)

# Event Handler для синхронизации Read Model
class OrderReadModelUpdater:
    async def handle_order_created(self, event: OrderCreatedEvent):
        await self.es.index(
            index="orders",
            id=event.order_id,
            body={
                "order_id": event.order_id,
                "user_id": event.user_id,
                "items": event.items,
                "status": event.status,
                "created_at": event.timestamp
            }
        )
```

---

## Проблемы и вызовы

### 1. Распределённые транзакции

В монолите ACID-транзакции просты:

```python
# Монолит: одна транзакция
with db.transaction():
    order = create_order(data)
    update_inventory(order.items)
    process_payment(order)
    # Всё откатится при ошибке
```

В микросервисах это невозможно — используйте Saga Pattern.

### 2. Eventual Consistency

Данные между сервисами становятся консистентными не мгновенно.

```python
# Проблема: Race condition
# 1. User Service обновляет email
# 2. Order Service читает старый email (данные ещё не синхронизированы)

# Решение 1: Включать нужные данные в событие
class OrderCreatedEvent:
    order_id: int
    user_email: str  # Email на момент создания заказа
    # ...

# Решение 2: Идемпотентные операции
async def handle_order_created(event):
    if await self.already_processed(event.event_id):
        return  # Пропускаем дубликат
    await self.process(event)
    await self.mark_processed(event.event_id)
```

### 3. Сетевые сбои

```python
# Retry с exponential backoff
import asyncio
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10)
)
async def call_payment_service(order):
    async with httpx.AsyncClient(timeout=5.0) as client:
        response = await client.post(
            f"{PAYMENT_SERVICE_URL}/process",
            json=order.to_dict()
        )
        response.raise_for_status()
        return response.json()

# Fallback стратегия
async def process_order_with_fallback(order):
    try:
        result = await call_payment_service(order)
    except Exception:
        # Fallback: сохраняем для последующей обработки
        await queue.enqueue("pending_payments", order)
        result = {"status": "pending", "message": "Payment will be processed later"}
    return result
```

### 4. Distributed Tracing

Отслеживание запроса через множество сервисов.

```python
# Использование OpenTelemetry
from opentelemetry import trace
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter

# Настройка трейсинга
tracer = trace.get_tracer(__name__)

# Инструментирование FastAPI
FastAPIInstrumentor.instrument_app(app)

# Ручное добавление спанов
@app.post("/orders")
async def create_order(order_data: OrderCreate):
    with tracer.start_as_current_span("create_order") as span:
        span.set_attribute("user_id", order_data.user_id)

        # Вложенный спан для вызова внешнего сервиса
        with tracer.start_as_current_span("call_inventory_service"):
            await inventory_client.reserve(order_data.items)

        with tracer.start_as_current_span("call_payment_service"):
            await payment_client.process(order_data.payment)

        return {"order_id": order.id}
```

### 5. Data Consistency при сбоях

```python
# Outbox Pattern — гарантия доставки событий
class OrderService:
    async def create_order(self, order_data):
        async with self.db.transaction():
            # 1. Сохраняем заказ
            order = Order(**order_data)
            await self.order_repository.save(order)

            # 2. Сохраняем событие в outbox (в той же транзакции!)
            event = OutboxEvent(
                aggregate_type="Order",
                aggregate_id=order.id,
                event_type="OrderCreated",
                payload=order.to_dict()
            )
            await self.outbox_repository.save(event)

        return order

# Отдельный процесс читает outbox и публикует события
class OutboxProcessor:
    async def process(self):
        while True:
            events = await self.outbox_repository.get_pending()
            for event in events:
                try:
                    await self.message_broker.publish(
                        event.event_type,
                        event.payload
                    )
                    event.status = "published"
                    await self.outbox_repository.update(event)
                except Exception as e:
                    event.retry_count += 1
                    await self.outbox_repository.update(event)
            await asyncio.sleep(1)
```

---

## Инструменты и технологии

### Docker — контейнеризация

```dockerfile
# Dockerfile для Python микросервиса
FROM python:3.11-slim

WORKDIR /app

# Установка зависимостей
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Копирование кода
COPY . .

# Запуск
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml для локальной разработки
version: '3.8'

services:
  user-service:
    build: ./user-service
    ports:
      - "8001:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@user-db:5432/users
    depends_on:
      - user-db
      - rabbitmq

  order-service:
    build: ./order-service
    ports:
      - "8002:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@order-db:5432/orders
      - USER_SERVICE_URL=http://user-service:8000
    depends_on:
      - order-db
      - rabbitmq

  user-db:
    image: postgres:15
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=users

  order-db:
    image: postgres:15
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=orders

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"
```

### Kubernetes — оркестрация

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: myregistry/order-service:v1.2.3
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: order-service-secrets
              key: database-url
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 3
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
  - port: 80
    targetPort: 8000
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Service Mesh (Istio)

Управление трафиком между сервисами на уровне инфраструктуры.

```yaml
# VirtualService для canary deployment
apiVersion: networking.istio.io/v1alpha3
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
        subset: v2
  - route:
    - destination:
        host: order-service
        subset: v1
      weight: 90
    - destination:
        host: order-service
        subset: v2
      weight: 10
---
# DestinationRule для circuit breaker
apiVersion: networking.istio.io/v1alpha3
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
      maxEjectionPercent: 100
```

### API Gateway

```yaml
# Kong API Gateway configuration
services:
  - name: user-service
    url: http://user-service:8000
    routes:
      - name: user-routes
        paths:
          - /api/users
        strip_path: true

  - name: order-service
    url: http://order-service:8000
    routes:
      - name: order-routes
        paths:
          - /api/orders
        strip_path: true

plugins:
  - name: rate-limiting
    config:
      minute: 100
      policy: local

  - name: jwt
    config:
      secret_is_base64: false
      claims_to_verify:
        - exp

  - name: cors
    config:
      origins:
        - "*"
      methods:
        - GET
        - POST
        - PUT
        - DELETE
```

### Observability Stack

```yaml
# Prometheus + Grafana + Jaeger
version: '3.8'

services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin

  jaeger:
    image: jaegertracing/all-in-one
    ports:
      - "16686:16686"  # UI
      - "6831:6831/udp"  # Thrift

  loki:
    image: grafana/loki
    ports:
      - "3100:3100"
```

---

## Best Practices

### 1. Правильные границы сервисов

```
✅ Хорошо:
- User Service: регистрация, аутентификация, профили
- Order Service: заказы, история заказов
- Notification Service: email, SMS, push

❌ Плохо:
- User-Order-Service: слишком много ответственности
- Email-Service, SMS-Service, Push-Service: слишком мелко
```

### 2. API Versioning

```python
from fastapi import FastAPI, APIRouter

app = FastAPI()

# Версионирование через URL
v1_router = APIRouter(prefix="/api/v1")
v2_router = APIRouter(prefix="/api/v2")

@v1_router.get("/users/{user_id}")
async def get_user_v1(user_id: int):
    return {"id": user_id, "name": "John"}

@v2_router.get("/users/{user_id}")
async def get_user_v2(user_id: int):
    return {
        "id": user_id,
        "name": "John",
        "email": "john@example.com",  # Новое поле
        "created_at": "2024-01-01"
    }

app.include_router(v1_router)
app.include_router(v2_router)
```

### 3. Health Checks

```python
from fastapi import FastAPI
from enum import Enum

app = FastAPI()

class HealthStatus(str, Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    UNHEALTHY = "unhealthy"

@app.get("/health")
async def health_check():
    """Liveness probe — сервис жив?"""
    return {"status": "ok"}

@app.get("/ready")
async def readiness_check():
    """Readiness probe — сервис готов принимать трафик?"""
    checks = {}

    # Проверка базы данных
    try:
        await database.execute("SELECT 1")
        checks["database"] = HealthStatus.HEALTHY
    except Exception:
        checks["database"] = HealthStatus.UNHEALTHY

    # Проверка Redis
    try:
        await redis.ping()
        checks["redis"] = HealthStatus.HEALTHY
    except Exception:
        checks["redis"] = HealthStatus.UNHEALTHY

    # Определяем общий статус
    if all(s == HealthStatus.HEALTHY for s in checks.values()):
        status = HealthStatus.HEALTHY
    elif any(s == HealthStatus.UNHEALTHY for s in checks.values()):
        status = HealthStatus.UNHEALTHY
    else:
        status = HealthStatus.DEGRADED

    return {"status": status, "checks": checks}
```

### 4. Graceful Shutdown

```python
import signal
import asyncio
from contextlib import asynccontextmanager
from fastapi import FastAPI

shutdown_event = asyncio.Event()

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    await database.connect()
    await message_broker.connect()

    yield

    # Shutdown
    print("Graceful shutdown initiated...")

    # Прекращаем принимать новые запросы
    shutdown_event.set()

    # Даём время на завершение текущих запросов
    await asyncio.sleep(5)

    # Закрываем соединения
    await message_broker.disconnect()
    await database.disconnect()

    print("Shutdown complete")

app = FastAPI(lifespan=lifespan)

# Обработка сигналов
def handle_signal(sig):
    shutdown_event.set()

signal.signal(signal.SIGTERM, handle_signal)
signal.signal(signal.SIGINT, handle_signal)
```

### 5. Централизованная конфигурация

```python
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    # Сервис
    service_name: str = "order-service"
    environment: str = "development"
    debug: bool = False

    # База данных
    database_url: str
    database_pool_size: int = 10

    # Внешние сервисы
    user_service_url: str
    payment_service_url: str

    # Message broker
    rabbitmq_url: str

    # Безопасность
    jwt_secret: str
    jwt_algorithm: str = "HS256"

    class Config:
        env_file = ".env"

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

---

## Типичные ошибки

### 1. Распределённый монолит

```python
# ❌ Плохо: Сервисы сильно связаны
class OrderService:
    def create_order(self, data):
        user = user_service.get_user(data.user_id)  # Синхронный вызов
        products = product_service.get_products(data.items)  # Синхронный вызов
        inventory = inventory_service.check(data.items)  # Синхронный вызов
        # Если любой сервис упал — всё падает

# ✅ Хорошо: Асинхронная коммуникация
class OrderService:
    def create_order(self, data):
        order = Order(status="pending", **data)
        self.order_repository.save(order)
        self.event_bus.publish(OrderCreated(order))
        return order  # Быстрый ответ, обработка асинхронно
```

### 2. Отсутствие Circuit Breaker

```python
# ❌ Плохо: Каскадный сбой
async def get_order_details(order_id):
    order = await order_repository.find(order_id)
    user = await user_service.get(order.user_id)  # Если user-service упал — мы зависаем
    return {"order": order, "user": user}

# ✅ Хорошо: Graceful degradation
@circuit_breaker(failure_threshold=5)
async def get_user_safe(user_id):
    return await user_service.get(user_id)

async def get_order_details(order_id):
    order = await order_repository.find(order_id)
    try:
        user = await get_user_safe(order.user_id)
    except CircuitBreakerOpen:
        user = {"id": order.user_id, "name": "Unknown"}  # Fallback
    return {"order": order, "user": user}
```

### 3. Общая база данных

```
❌ Плохо:
┌─────────────┐     ┌─────────────┐
│   Service A │     │   Service B │
└──────┬──────┘     └──────┬──────┘
       │                   │
       └─────────┬─────────┘
                 │
         ┌───────▼───────┐
         │ Shared Database│
         └───────────────┘

✅ Хорошо:
┌─────────────┐     ┌─────────────┐
│   Service A │     │   Service B │
├─────────────┤     ├─────────────┤
│  Database A │     │  Database B │
└─────────────┘     └─────────────┘
```

### 4. Слишком мелкие сервисы (Nano-services)

```
❌ Плохо: Слишком много накладных расходов
- UserRegistrationService
- UserLoginService
- UserProfileService
- UserPasswordResetService
- UserEmailVerificationService

✅ Хорошо: Разумная гранулярность
- UserService (все операции с пользователями)
```

### 5. Отсутствие идемпотентности

```python
# ❌ Плохо: Повторный запрос создаёт дубликат
@app.post("/payments")
async def process_payment(payment: PaymentRequest):
    return await payment_service.process(payment)

# ✅ Хорошо: Идемпотентность через idempotency key
@app.post("/payments")
async def process_payment(
    payment: PaymentRequest,
    idempotency_key: str = Header(...)
):
    # Проверяем, не обрабатывали ли мы уже этот запрос
    existing = await cache.get(f"payment:{idempotency_key}")
    if existing:
        return existing

    result = await payment_service.process(payment)
    await cache.set(f"payment:{idempotency_key}", result, ttl=86400)
    return result
```

---

## Когда использовать микросервисы

### Используйте микросервисы когда:

1. **Большая команда** (50+ разработчиков) — независимые команды могут работать параллельно
2. **Разные требования к масштабированию** — одни части системы нагружены больше других
3. **Разные темпы изменений** — часть системы меняется часто, часть — редко
4. **Технологическое разнообразие необходимо** — разные части требуют разных технологий
5. **Высокие требования к отказоустойчивости** — критично изолировать сбои

### НЕ используйте микросервисы когда:

1. **Маленькая команда** (< 10 человек) — накладные расходы перевесят преимущества
2. **Новый проект без понимания домена** — сначала разберитесь в бизнесе
3. **Простое приложение** — микросервисы добавят ненужную сложность
4. **Нет DevOps культуры** — нужны инструменты и навыки для управления
5. **Tight budget** — инфраструктура микросервисов дороже

### Рекомендуемый путь

```
1. Начните с монолита
   └── Быстрый старт, понимание домена

2. Модульный монолит
   └── Чёткие границы между модулями внутри одного приложения

3. Постепенная декомпозиция (Strangler Fig)
   └── Выделяйте микросервисы по мере необходимости

4. Полноценные микросервисы
   └── Когда команда и инфраструктура готовы
```

```python
# Модульный монолит — промежуточный этап
# project/
# ├── users/
# │   ├── models.py
# │   ├── services.py
# │   ├── api.py
# │   └── __init__.py  # Публичный интерфейс модуля
# ├── orders/
# │   ├── models.py
# │   ├── services.py
# │   ├── api.py
# │   └── __init__.py
# ├── payments/
# │   └── ...
# └── main.py

# users/__init__.py — публичный API модуля
from .services import UserService
from .models import User

# orders/services.py — использует только публичный API
from users import UserService  # ✅ Через публичный интерфейс
# from users.models import User  # ❌ Прямой доступ к внутренностям
```

---

## Заключение

Микросервисная архитектура — мощный инструмент для построения масштабируемых и гибких систем, но она приносит значительную сложность. Ключевые моменты:

1. **Не начинайте с микросервисов** — сначала монолит, потом декомпозиция
2. **Правильные границы** — по бизнес-доменам, не по техническим слоям
3. **Database per Service** — каждый сервис владеет своими данными
4. **Асинхронная коммуникация** — снижает связанность и повышает устойчивость
5. **Design for Failure** — Circuit Breaker, Retry, Fallback
6. **Observability** — логи, метрики, трейсинг критически важны
7. **Автоматизация** — CI/CD, Infrastructure as Code, контейнеризация

Микросервисы — это не цель, а средство. Используйте их только когда преимущества перевешивают сложность.
