# Event-Driven Architecture

## Введение

**Event-Driven Architecture (EDA)** — это архитектурный стиль, в котором компоненты системы взаимодействуют через события. Событие — это факт о чём-то, что произошло в системе (например, "заказ создан", "платёж обработан").

### Ключевые концепции

- **Event (событие)** — неизменяемая запись о факте, который произошёл
- **Event Producer** — компонент, генерирующий события
- **Event Consumer** — компонент, реагирующий на события
- **Event Broker** — посредник для передачи событий (Kafka, RabbitMQ)
- **Event Store** — хранилище событий

### Преимущества EDA

1. **Слабая связанность** — сервисы не знают друг о друге напрямую
2. **Масштабируемость** — легко добавлять новых consumers
3. **Гибкость** — новая функциональность без изменения существующего кода
4. **Аудит** — полная история изменений через события
5. **Resilience** — изоляция сбоев между компонентами

---

## Типы событий

### 1. Domain Events (События предметной области)

Отражают значимые изменения в бизнес-логике.

```python
from dataclasses import dataclass
from datetime import datetime
from typing import List
import uuid

@dataclass(frozen=True)  # Immutable
class DomainEvent:
    event_id: str
    occurred_at: datetime
    aggregate_id: str
    aggregate_type: str

@dataclass(frozen=True)
class OrderCreated(DomainEvent):
    customer_id: str
    items: List[dict]
    total_amount: float

@dataclass(frozen=True)
class OrderShipped(DomainEvent):
    tracking_number: str
    carrier: str

@dataclass(frozen=True)
class PaymentReceived(DomainEvent):
    payment_id: str
    amount: float
    method: str

# Создание события
event = OrderCreated(
    event_id=str(uuid.uuid4()),
    occurred_at=datetime.utcnow(),
    aggregate_id="order-123",
    aggregate_type="Order",
    customer_id="customer-456",
    items=[{"product_id": "prod-1", "quantity": 2}],
    total_amount=99.99
)
```

### 2. Integration Events (События интеграции)

Используются для коммуникации между bounded contexts или микросервисами.

```python
@dataclass(frozen=True)
class IntegrationEvent:
    event_id: str
    occurred_at: datetime
    source_service: str
    correlation_id: str  # Для трассировки

@dataclass(frozen=True)
class CustomerRegisteredIntegration(IntegrationEvent):
    customer_id: str
    email: str
    # Только необходимые поля для других сервисов
    # Не раскрываем внутренние детали

# Преобразование Domain Event в Integration Event
def to_integration_event(domain_event: OrderCreated) -> dict:
    return {
        "event_type": "order.created",
        "event_id": domain_event.event_id,
        "occurred_at": domain_event.occurred_at.isoformat(),
        "source_service": "order-service",
        "data": {
            "order_id": domain_event.aggregate_id,
            "customer_id": domain_event.customer_id,
            "total_amount": domain_event.total_amount
        }
    }
```

### 3. Event Notifications

Минимальное уведомление о том, что что-то произошло.

```python
# Только факт события, детали нужно получить отдельно
notification = {
    "event_type": "order.updated",
    "order_id": "order-123",
    "occurred_at": "2024-01-15T10:30:00Z"
}

# Consumer при необходимости запрашивает детали
async def handle_order_updated(notification):
    order_id = notification["order_id"]
    # Получаем актуальные данные
    order = await order_service.get_order(order_id)
    # Обрабатываем
```

---

## Event Sourcing

**Event Sourcing** — паттерн, при котором состояние системы сохраняется как последовательность событий, а не как текущий снимок.

### Традиционный подход vs Event Sourcing

```
Традиционный подход:
┌─────────────────────────────────────┐
│ orders table                        │
│ id: 123                             │
│ status: "shipped"                   │
│ total: 150.00                       │
│ updated_at: 2024-01-15              │
└─────────────────────────────────────┘
  └── Мы видим только текущее состояние

Event Sourcing:
┌─────────────────────────────────────┐
│ events table                        │
│ 1. OrderCreated { total: 100 }      │
│ 2. ItemAdded { product: X, +50 }    │
│ 3. PaymentReceived { amount: 150 }  │
│ 4. OrderShipped { tracking: ABC }   │
└─────────────────────────────────────┘
  └── Полная история всех изменений
```

### Реализация Event Sourcing

```python
from abc import ABC, abstractmethod
from typing import List, Type
from dataclasses import dataclass, field
import json

# Базовый класс для агрегата
class Aggregate(ABC):
    def __init__(self, aggregate_id: str):
        self.id = aggregate_id
        self.version = 0
        self._uncommitted_events: List[DomainEvent] = []

    def apply_event(self, event: DomainEvent):
        """Применяет событие к состоянию агрегата"""
        handler_name = f"_apply_{type(event).__name__}"
        handler = getattr(self, handler_name, None)
        if handler:
            handler(event)
        self.version += 1

    def load_from_history(self, events: List[DomainEvent]):
        """Восстанавливает состояние из истории событий"""
        for event in events:
            self.apply_event(event)

    def raise_event(self, event: DomainEvent):
        """Регистрирует новое событие"""
        self._uncommitted_events.append(event)
        self.apply_event(event)

    def get_uncommitted_events(self) -> List[DomainEvent]:
        return self._uncommitted_events.copy()

    def clear_uncommitted_events(self):
        self._uncommitted_events.clear()


# Пример: Order Aggregate
@dataclass
class OrderItem:
    product_id: str
    quantity: int
    price: float

class Order(Aggregate):
    def __init__(self, order_id: str):
        super().__init__(order_id)
        self.customer_id: str = ""
        self.items: List[OrderItem] = []
        self.status: str = ""
        self.total: float = 0.0

    # Команды (изменяют состояние через события)
    def create(self, customer_id: str, items: List[dict]):
        if self.status:
            raise ValueError("Order already exists")

        self.raise_event(OrderCreated(
            event_id=str(uuid.uuid4()),
            occurred_at=datetime.utcnow(),
            aggregate_id=self.id,
            aggregate_type="Order",
            customer_id=customer_id,
            items=items,
            total_amount=sum(i["price"] * i["quantity"] for i in items)
        ))

    def add_item(self, product_id: str, quantity: int, price: float):
        if self.status != "created":
            raise ValueError("Cannot add items to this order")

        self.raise_event(ItemAdded(
            event_id=str(uuid.uuid4()),
            occurred_at=datetime.utcnow(),
            aggregate_id=self.id,
            aggregate_type="Order",
            product_id=product_id,
            quantity=quantity,
            price=price
        ))

    def ship(self, tracking_number: str, carrier: str):
        if self.status != "paid":
            raise ValueError("Cannot ship unpaid order")

        self.raise_event(OrderShipped(
            event_id=str(uuid.uuid4()),
            occurred_at=datetime.utcnow(),
            aggregate_id=self.id,
            aggregate_type="Order",
            tracking_number=tracking_number,
            carrier=carrier
        ))

    # Обработчики событий (изменяют внутреннее состояние)
    def _apply_OrderCreated(self, event: OrderCreated):
        self.customer_id = event.customer_id
        self.items = [
            OrderItem(i["product_id"], i["quantity"], i["price"])
            for i in event.items
        ]
        self.total = event.total_amount
        self.status = "created"

    def _apply_ItemAdded(self, event):
        self.items.append(OrderItem(
            event.product_id,
            event.quantity,
            event.price
        ))
        self.total += event.price * event.quantity

    def _apply_OrderShipped(self, event: OrderShipped):
        self.status = "shipped"
```

### Event Store

```python
from typing import Optional
import json

class EventStore:
    def __init__(self, connection):
        self.connection = connection

    async def save_events(
        self,
        aggregate_id: str,
        events: List[DomainEvent],
        expected_version: int
    ):
        """Сохраняет события с оптимистичной блокировкой"""
        async with self.connection.transaction():
            # Проверяем версию для оптимистичной блокировки
            current_version = await self._get_current_version(aggregate_id)

            if current_version != expected_version:
                raise ConcurrencyError(
                    f"Expected version {expected_version}, "
                    f"but current is {current_version}"
                )

            # Сохраняем события
            for i, event in enumerate(events):
                await self.connection.execute("""
                    INSERT INTO events
                    (aggregate_id, version, event_type, event_data, occurred_at)
                    VALUES ($1, $2, $3, $4, $5)
                """,
                    aggregate_id,
                    expected_version + i + 1,
                    type(event).__name__,
                    json.dumps(self._serialize_event(event)),
                    event.occurred_at
                )

    async def load_events(
        self,
        aggregate_id: str,
        from_version: int = 0
    ) -> List[DomainEvent]:
        """Загружает события для агрегата"""
        rows = await self.connection.fetch("""
            SELECT event_type, event_data, version
            FROM events
            WHERE aggregate_id = $1 AND version > $2
            ORDER BY version
        """, aggregate_id, from_version)

        return [
            self._deserialize_event(row["event_type"], row["event_data"])
            for row in rows
        ]

    async def _get_current_version(self, aggregate_id: str) -> int:
        result = await self.connection.fetchval("""
            SELECT COALESCE(MAX(version), 0)
            FROM events
            WHERE aggregate_id = $1
        """, aggregate_id)
        return result

    def _serialize_event(self, event: DomainEvent) -> dict:
        return {
            "event_id": event.event_id,
            "occurred_at": event.occurred_at.isoformat(),
            "aggregate_id": event.aggregate_id,
            **{k: v for k, v in event.__dict__.items()
               if k not in ("event_id", "occurred_at", "aggregate_id", "aggregate_type")}
        }

    def _deserialize_event(self, event_type: str, data: dict) -> DomainEvent:
        event_classes = {
            "OrderCreated": OrderCreated,
            "ItemAdded": ItemAdded,
            "OrderShipped": OrderShipped,
        }
        cls = event_classes[event_type]
        return cls(**data)
```

### Repository для Event Sourcing

```python
class OrderRepository:
    def __init__(self, event_store: EventStore):
        self.event_store = event_store

    async def get(self, order_id: str) -> Optional[Order]:
        events = await self.event_store.load_events(order_id)
        if not events:
            return None

        order = Order(order_id)
        order.load_from_history(events)
        return order

    async def save(self, order: Order):
        events = order.get_uncommitted_events()
        if not events:
            return

        expected_version = order.version - len(events)
        await self.event_store.save_events(
            order.id,
            events,
            expected_version
        )
        order.clear_uncommitted_events()

# Использование
async def create_order(customer_id: str, items: List[dict]):
    order_id = str(uuid.uuid4())
    order = Order(order_id)
    order.create(customer_id, items)

    await order_repository.save(order)

    # Публикуем события для других сервисов
    for event in order.get_uncommitted_events():
        await event_publisher.publish(event)

    return order_id
```

### Snapshots (Снимки)

Для оптимизации производительности при большом количестве событий.

```python
class SnapshotStore:
    async def save_snapshot(self, aggregate_id: str, aggregate: Aggregate):
        await self.connection.execute("""
            INSERT INTO snapshots (aggregate_id, version, state, created_at)
            VALUES ($1, $2, $3, NOW())
            ON CONFLICT (aggregate_id)
            DO UPDATE SET version = $2, state = $3, created_at = NOW()
        """,
            aggregate_id,
            aggregate.version,
            json.dumps(self._serialize_state(aggregate))
        )

    async def load_snapshot(self, aggregate_id: str) -> Optional[tuple]:
        row = await self.connection.fetchrow("""
            SELECT version, state FROM snapshots
            WHERE aggregate_id = $1
        """, aggregate_id)

        if row:
            return row["version"], row["state"]
        return None

class OrderRepositoryWithSnapshots:
    def __init__(
        self,
        event_store: EventStore,
        snapshot_store: SnapshotStore,
        snapshot_threshold: int = 100
    ):
        self.event_store = event_store
        self.snapshot_store = snapshot_store
        self.snapshot_threshold = snapshot_threshold

    async def get(self, order_id: str) -> Optional[Order]:
        order = Order(order_id)

        # Пытаемся загрузить snapshot
        snapshot = await self.snapshot_store.load_snapshot(order_id)
        if snapshot:
            version, state = snapshot
            self._restore_from_state(order, state)
            order.version = version

            # Загружаем только события после snapshot
            events = await self.event_store.load_events(order_id, version)
        else:
            events = await self.event_store.load_events(order_id)

        if not events and not snapshot:
            return None

        order.load_from_history(events)
        return order

    async def save(self, order: Order):
        events = order.get_uncommitted_events()
        if not events:
            return

        expected_version = order.version - len(events)
        await self.event_store.save_events(order.id, events, expected_version)

        # Создаём snapshot каждые N событий
        if order.version % self.snapshot_threshold == 0:
            await self.snapshot_store.save_snapshot(order.id, order)

        order.clear_uncommitted_events()
```

---

## CQRS (Command Query Responsibility Segregation)

**CQRS** — паттерн разделения операций чтения (Query) и записи (Command) на разные модели.

### Зачем нужен CQRS?

1. **Оптимизация чтения и записи независимо**
2. **Разные модели данных** для разных use cases
3. **Масштабирование** read и write сторон отдельно
4. **Упрощение сложных запросов** через денормализованные views

### Архитектура CQRS

```
                    ┌─────────────────┐
                    │     Client      │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
       ┌──────────────┐              ┌──────────────┐
       │   Commands   │              │   Queries    │
       │   (Write)    │              │   (Read)     │
       └──────┬───────┘              └──────┬───────┘
              │                             │
              ▼                             ▼
       ┌──────────────┐              ┌──────────────┐
       │ Write Model  │              │  Read Model  │
       │ (Normalized) │              │(Denormalized)│
       └──────┬───────┘              └──────────────┘
              │                             ▲
              │         Events              │
              └─────────────────────────────┘
```

### Реализация CQRS

```python
# ==================== COMMAND SIDE ====================

from abc import ABC, abstractmethod

# Команды
@dataclass
class CreateOrderCommand:
    customer_id: str
    items: List[dict]

@dataclass
class AddItemCommand:
    order_id: str
    product_id: str
    quantity: int

@dataclass
class ShipOrderCommand:
    order_id: str
    tracking_number: str
    carrier: str

# Command Handler
class CommandHandler(ABC):
    @abstractmethod
    async def handle(self, command) -> None:
        pass

class CreateOrderHandler(CommandHandler):
    def __init__(
        self,
        order_repository: OrderRepository,
        event_publisher: EventPublisher
    ):
        self.order_repository = order_repository
        self.event_publisher = event_publisher

    async def handle(self, command: CreateOrderCommand) -> str:
        # Создаём агрегат
        order_id = str(uuid.uuid4())
        order = Order(order_id)
        order.create(command.customer_id, command.items)

        # Сохраняем
        await self.order_repository.save(order)

        # Публикуем события
        for event in order.get_uncommitted_events():
            await self.event_publisher.publish(event)

        return order_id

class ShipOrderHandler(CommandHandler):
    def __init__(
        self,
        order_repository: OrderRepository,
        event_publisher: EventPublisher
    ):
        self.order_repository = order_repository
        self.event_publisher = event_publisher

    async def handle(self, command: ShipOrderCommand):
        # Загружаем агрегат
        order = await self.order_repository.get(command.order_id)
        if not order:
            raise OrderNotFound(command.order_id)

        # Выполняем команду
        order.ship(command.tracking_number, command.carrier)

        # Сохраняем
        await self.order_repository.save(order)

        # Публикуем события
        for event in order.get_uncommitted_events():
            await self.event_publisher.publish(event)


# Command Bus
class CommandBus:
    def __init__(self):
        self._handlers: dict = {}

    def register(self, command_type: Type, handler: CommandHandler):
        self._handlers[command_type] = handler

    async def dispatch(self, command) -> any:
        handler = self._handlers.get(type(command))
        if not handler:
            raise ValueError(f"No handler for {type(command)}")
        return await handler.handle(command)


# ==================== QUERY SIDE ====================

# Read Models (View Models)
@dataclass
class OrderSummary:
    order_id: str
    customer_name: str
    status: str
    total: float
    item_count: int
    created_at: datetime

@dataclass
class OrderDetails:
    order_id: str
    customer: dict
    items: List[dict]
    status: str
    total: float
    shipping: Optional[dict]
    timeline: List[dict]

# Query
@dataclass
class GetOrderQuery:
    order_id: str

@dataclass
class ListOrdersQuery:
    customer_id: Optional[str] = None
    status: Optional[str] = None
    limit: int = 20
    offset: int = 0

# Query Handler
class QueryHandler(ABC):
    @abstractmethod
    async def handle(self, query):
        pass

class GetOrderQueryHandler(QueryHandler):
    def __init__(self, read_db):
        self.read_db = read_db

    async def handle(self, query: GetOrderQuery) -> Optional[OrderDetails]:
        row = await self.read_db.fetchrow("""
            SELECT
                o.order_id,
                o.customer_id,
                o.customer_name,
                o.customer_email,
                o.status,
                o.total,
                o.tracking_number,
                o.carrier,
                o.created_at,
                o.updated_at,
                json_agg(json_build_object(
                    'product_id', oi.product_id,
                    'product_name', oi.product_name,
                    'quantity', oi.quantity,
                    'price', oi.price
                )) as items
            FROM order_read_model o
            LEFT JOIN order_items_read_model oi ON o.order_id = oi.order_id
            WHERE o.order_id = $1
            GROUP BY o.order_id
        """, query.order_id)

        if not row:
            return None

        return OrderDetails(
            order_id=row["order_id"],
            customer={
                "id": row["customer_id"],
                "name": row["customer_name"],
                "email": row["customer_email"]
            },
            items=row["items"],
            status=row["status"],
            total=row["total"],
            shipping={
                "tracking_number": row["tracking_number"],
                "carrier": row["carrier"]
            } if row["tracking_number"] else None,
            timeline=await self._get_timeline(query.order_id)
        )

    async def _get_timeline(self, order_id: str) -> List[dict]:
        rows = await self.read_db.fetch("""
            SELECT event_type, occurred_at, details
            FROM order_timeline
            WHERE order_id = $1
            ORDER BY occurred_at
        """, order_id)
        return [dict(row) for row in rows]

class ListOrdersQueryHandler(QueryHandler):
    def __init__(self, read_db):
        self.read_db = read_db

    async def handle(self, query: ListOrdersQuery) -> List[OrderSummary]:
        conditions = []
        params = []

        if query.customer_id:
            conditions.append(f"customer_id = ${len(params) + 1}")
            params.append(query.customer_id)

        if query.status:
            conditions.append(f"status = ${len(params) + 1}")
            params.append(query.status)

        where_clause = "WHERE " + " AND ".join(conditions) if conditions else ""

        rows = await self.read_db.fetch(f"""
            SELECT order_id, customer_name, status, total, item_count, created_at
            FROM order_read_model
            {where_clause}
            ORDER BY created_at DESC
            LIMIT ${len(params) + 1} OFFSET ${len(params) + 2}
        """, *params, query.limit, query.offset)

        return [OrderSummary(**dict(row)) for row in rows]


# Query Bus
class QueryBus:
    def __init__(self):
        self._handlers: dict = {}

    def register(self, query_type: Type, handler: QueryHandler):
        self._handlers[query_type] = handler

    async def dispatch(self, query):
        handler = self._handlers.get(type(query))
        if not handler:
            raise ValueError(f"No handler for {type(query)}")
        return await handler.handle(query)
```

### Проекции (Projections)

Обновление Read Model на основе событий.

```python
class OrderProjection:
    """Обновляет Read Model на основе событий"""

    def __init__(self, read_db, customer_service):
        self.read_db = read_db
        self.customer_service = customer_service

    async def handle_event(self, event: DomainEvent):
        handler_name = f"_handle_{type(event).__name__}"
        handler = getattr(self, handler_name, None)
        if handler:
            await handler(event)

    async def _handle_OrderCreated(self, event: OrderCreated):
        # Получаем данные о клиенте
        customer = await self.customer_service.get_customer(event.customer_id)

        # Вставляем в read model
        await self.read_db.execute("""
            INSERT INTO order_read_model
            (order_id, customer_id, customer_name, customer_email,
             status, total, item_count, created_at, updated_at)
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $8)
        """,
            event.aggregate_id,
            event.customer_id,
            customer["name"],
            customer["email"],
            "created",
            event.total_amount,
            len(event.items),
            event.occurred_at
        )

        # Вставляем items
        for item in event.items:
            product = await self.product_service.get_product(item["product_id"])
            await self.read_db.execute("""
                INSERT INTO order_items_read_model
                (order_id, product_id, product_name, quantity, price)
                VALUES ($1, $2, $3, $4, $5)
            """,
                event.aggregate_id,
                item["product_id"],
                product["name"],
                item["quantity"],
                item["price"]
            )

        # Добавляем в timeline
        await self._add_timeline_event(
            event.aggregate_id,
            "OrderCreated",
            event.occurred_at,
            {"total": event.total_amount}
        )

    async def _handle_OrderShipped(self, event: OrderShipped):
        await self.read_db.execute("""
            UPDATE order_read_model
            SET status = 'shipped',
                tracking_number = $2,
                carrier = $3,
                updated_at = $4
            WHERE order_id = $1
        """,
            event.aggregate_id,
            event.tracking_number,
            event.carrier,
            event.occurred_at
        )

        await self._add_timeline_event(
            event.aggregate_id,
            "OrderShipped",
            event.occurred_at,
            {
                "tracking_number": event.tracking_number,
                "carrier": event.carrier
            }
        )

    async def _add_timeline_event(
        self,
        order_id: str,
        event_type: str,
        occurred_at: datetime,
        details: dict
    ):
        await self.read_db.execute("""
            INSERT INTO order_timeline
            (order_id, event_type, occurred_at, details)
            VALUES ($1, $2, $3, $4)
        """, order_id, event_type, occurred_at, json.dumps(details))


# Projection Runner (Consumer)
class ProjectionRunner:
    def __init__(self, projections: List, event_consumer):
        self.projections = projections
        self.event_consumer = event_consumer

    async def run(self):
        async for event in self.event_consumer.consume():
            for projection in self.projections:
                try:
                    await projection.handle_event(event)
                except Exception as e:
                    logger.error(f"Projection error: {e}")
                    # Решаем: retry, skip, или dead letter
```

---

## Saga Pattern

**Saga** — паттерн для управления распределёнными транзакциями через последовательность локальных транзакций с компенсирующими действиями.

### Проблема

```
Традиционная транзакция (невозможна в микросервисах):
BEGIN TRANSACTION
    1. Резервировать товар в Inventory Service
    2. Списать деньги в Payment Service
    3. Создать доставку в Shipping Service
COMMIT  ← Нельзя! Разные базы данных!
```

### Решение: Saga

```
Saga:
1. Order Service: Создать заказ
   ↓ успех
2. Inventory Service: Резервировать товар
   ↓ успех
3. Payment Service: Списать деньги
   ↓ ОШИБКА!
   ↓
   Компенсация:
   - Inventory Service: Освободить товар
   - Order Service: Отменить заказ
```

### Типы Saga

#### 1. Choreography (Хореография)

Сервисы координируются через события. Нет центрального координатора.

```python
# ==================== Order Service ====================
class OrderService:
    async def create_order(self, customer_id: str, items: List[dict]):
        order = Order.create(customer_id, items)
        await self.order_repository.save(order)

        # Публикуем событие — начало саги
        await self.event_publisher.publish(OrderCreated(
            order_id=order.id,
            customer_id=customer_id,
            items=items,
            total=order.total
        ))

        return order.id

    # Слушаем события от других сервисов
    async def on_payment_completed(self, event: PaymentCompleted):
        order = await self.order_repository.get(event.order_id)
        order.mark_paid()
        await self.order_repository.save(order)

    async def on_payment_failed(self, event: PaymentFailed):
        order = await self.order_repository.get(event.order_id)
        order.cancel(reason=event.reason)
        await self.order_repository.save(order)

        # Публикуем событие для отката в Inventory
        await self.event_publisher.publish(OrderCancelled(
            order_id=order.id,
            reason=event.reason
        ))


# ==================== Inventory Service ====================
class InventoryService:
    async def on_order_created(self, event: OrderCreated):
        try:
            reservation = await self.reserve_items(
                event.order_id,
                event.items
            )

            await self.event_publisher.publish(InventoryReserved(
                order_id=event.order_id,
                reservation_id=reservation.id
            ))
        except InsufficientStock as e:
            await self.event_publisher.publish(InventoryReservationFailed(
                order_id=event.order_id,
                reason=str(e)
            ))

    async def on_order_cancelled(self, event: OrderCancelled):
        # Компенсирующее действие
        await self.release_reservation(event.order_id)

    async def on_order_shipped(self, event: OrderShipped):
        # Подтверждаем списание
        await self.confirm_reservation(event.order_id)


# ==================== Payment Service ====================
class PaymentService:
    async def on_inventory_reserved(self, event: InventoryReserved):
        try:
            payment = await self.charge_customer(
                event.order_id,
                event.amount
            )

            await self.event_publisher.publish(PaymentCompleted(
                order_id=event.order_id,
                payment_id=payment.id
            ))
        except PaymentError as e:
            await self.event_publisher.publish(PaymentFailed(
                order_id=event.order_id,
                reason=str(e)
            ))

    async def on_order_cancelled(self, event: OrderCancelled):
        # Компенсирующее действие — возврат денег
        await self.refund_payment(event.order_id)
```

**Диаграмма Choreography:**

```
┌──────────┐    ┌───────────┐    ┌─────────┐    ┌──────────┐
│  Order   │    │ Inventory │    │ Payment │    │ Shipping │
└────┬─────┘    └─────┬─────┘    └────┬────┘    └────┬─────┘
     │                │               │              │
     │ OrderCreated   │               │              │
     │───────────────▶│               │              │
     │                │               │              │
     │                │ InventoryReserved            │
     │                │──────────────▶│              │
     │                │               │              │
     │                │               │ PaymentCompleted
     │◀───────────────│───────────────│              │
     │                │               │              │
     │ OrderConfirmed │               │              │
     │───────────────▶│───────────────│─────────────▶│
     │                │               │              │
```

#### 2. Orchestration (Оркестрация)

Центральный координатор (Orchestrator) управляет сагой.

```python
from enum import Enum
from dataclasses import dataclass, field
from typing import Optional

class SagaState(Enum):
    STARTED = "started"
    INVENTORY_RESERVED = "inventory_reserved"
    PAYMENT_COMPLETED = "payment_completed"
    SHIPPING_CREATED = "shipping_created"
    COMPLETED = "completed"
    COMPENSATING = "compensating"
    FAILED = "failed"

@dataclass
class OrderSagaData:
    order_id: str
    customer_id: str
    items: List[dict]
    total: float
    state: SagaState = SagaState.STARTED
    reservation_id: Optional[str] = None
    payment_id: Optional[str] = None
    shipment_id: Optional[str] = None
    failure_reason: Optional[str] = None


class OrderSagaOrchestrator:
    """Центральный координатор саги"""

    def __init__(
        self,
        saga_repository,
        inventory_service,
        payment_service,
        shipping_service,
        order_service
    ):
        self.saga_repository = saga_repository
        self.inventory_service = inventory_service
        self.payment_service = payment_service
        self.shipping_service = shipping_service
        self.order_service = order_service

    async def start_saga(
        self,
        order_id: str,
        customer_id: str,
        items: List[dict],
        total: float
    ):
        saga = OrderSagaData(
            order_id=order_id,
            customer_id=customer_id,
            items=items,
            total=total
        )
        await self.saga_repository.save(saga)
        await self._execute_step(saga)

    async def _execute_step(self, saga: OrderSagaData):
        """Выполняет следующий шаг саги"""
        try:
            if saga.state == SagaState.STARTED:
                await self._reserve_inventory(saga)

            elif saga.state == SagaState.INVENTORY_RESERVED:
                await self._process_payment(saga)

            elif saga.state == SagaState.PAYMENT_COMPLETED:
                await self._create_shipment(saga)

            elif saga.state == SagaState.SHIPPING_CREATED:
                await self._complete_saga(saga)

        except Exception as e:
            await self._start_compensation(saga, str(e))

    async def _reserve_inventory(self, saga: OrderSagaData):
        result = await self.inventory_service.reserve(
            saga.order_id,
            saga.items
        )

        saga.reservation_id = result.reservation_id
        saga.state = SagaState.INVENTORY_RESERVED
        await self.saga_repository.save(saga)
        await self._execute_step(saga)

    async def _process_payment(self, saga: OrderSagaData):
        result = await self.payment_service.charge(
            saga.order_id,
            saga.customer_id,
            saga.total
        )

        saga.payment_id = result.payment_id
        saga.state = SagaState.PAYMENT_COMPLETED
        await self.saga_repository.save(saga)
        await self._execute_step(saga)

    async def _create_shipment(self, saga: OrderSagaData):
        result = await self.shipping_service.create_shipment(
            saga.order_id,
            saga.items
        )

        saga.shipment_id = result.shipment_id
        saga.state = SagaState.SHIPPING_CREATED
        await self.saga_repository.save(saga)
        await self._execute_step(saga)

    async def _complete_saga(self, saga: OrderSagaData):
        await self.order_service.confirm_order(saga.order_id)
        saga.state = SagaState.COMPLETED
        await self.saga_repository.save(saga)

    async def _start_compensation(self, saga: OrderSagaData, reason: str):
        """Запускает компенсирующие действия"""
        saga.state = SagaState.COMPENSATING
        saga.failure_reason = reason
        await self.saga_repository.save(saga)

        # Компенсация в обратном порядке
        if saga.shipment_id:
            await self.shipping_service.cancel_shipment(saga.shipment_id)

        if saga.payment_id:
            await self.payment_service.refund(saga.payment_id)

        if saga.reservation_id:
            await self.inventory_service.release(saga.reservation_id)

        await self.order_service.cancel_order(saga.order_id, reason)

        saga.state = SagaState.FAILED
        await self.saga_repository.save(saga)


# Использование с Temporal (рекомендуется для production)
from temporalio import workflow, activity
from temporalio.common import RetryPolicy

@activity.defn
async def reserve_inventory(order_id: str, items: List[dict]) -> str:
    # Вызов Inventory Service
    result = await inventory_client.reserve(order_id, items)
    return result.reservation_id

@activity.defn
async def release_inventory(reservation_id: str):
    await inventory_client.release(reservation_id)

@activity.defn
async def charge_payment(order_id: str, amount: float) -> str:
    result = await payment_client.charge(order_id, amount)
    return result.payment_id

@activity.defn
async def refund_payment(payment_id: str):
    await payment_client.refund(payment_id)

@workflow.defn
class OrderSagaWorkflow:
    @workflow.run
    async def run(self, order_id: str, items: List[dict], total: float):
        reservation_id = None
        payment_id = None

        try:
            # Шаг 1: Резервирование
            reservation_id = await workflow.execute_activity(
                reserve_inventory,
                args=[order_id, items],
                start_to_close_timeout=timedelta(seconds=30),
                retry_policy=RetryPolicy(maximum_attempts=3)
            )

            # Шаг 2: Оплата
            payment_id = await workflow.execute_activity(
                charge_payment,
                args=[order_id, total],
                start_to_close_timeout=timedelta(seconds=30),
                retry_policy=RetryPolicy(maximum_attempts=3)
            )

            return {"status": "completed", "payment_id": payment_id}

        except Exception as e:
            # Компенсация
            if payment_id:
                await workflow.execute_activity(
                    refund_payment,
                    args=[payment_id],
                    start_to_close_timeout=timedelta(seconds=30)
                )

            if reservation_id:
                await workflow.execute_activity(
                    release_inventory,
                    args=[reservation_id],
                    start_to_close_timeout=timedelta(seconds=30)
                )

            return {"status": "failed", "reason": str(e)}
```

### Сравнение Choreography vs Orchestration

| Аспект | Choreography | Orchestration |
|--------|--------------|---------------|
| Связанность | Слабая | Средняя (через оркестратор) |
| Сложность | Растёт с количеством сервисов | Централизована |
| Single Point of Failure | Нет | Оркестратор |
| Видимость процесса | Распределена | Централизована |
| Добавление шагов | Сложнее | Проще |
| Тестирование | Сложнее | Проще |
| Рекомендуется | Простые саги (2-3 шага) | Сложные саги (4+ шага) |

---

## Паттерны обработки событий

### 1. Event Replay

```python
class EventReplayService:
    """Переигрывание событий для восстановления состояния"""

    async def rebuild_projection(
        self,
        projection: Projection,
        from_position: int = 0
    ):
        # Получаем все события с позиции
        async for event in self.event_store.read_all(from_position):
            await projection.handle_event(event)

    async def rebuild_aggregate(self, aggregate_id: str) -> Aggregate:
        events = await self.event_store.load_events(aggregate_id)
        aggregate = Order(aggregate_id)
        aggregate.load_from_history(events)
        return aggregate
```

### 2. Event Versioning

```python
# Версионирование событий для обратной совместимости
class EventUpgrader:
    def upgrade(self, event_data: dict, from_version: int) -> dict:
        if from_version < 2:
            # Добавляем новое поле
            event_data["currency"] = "USD"

        if from_version < 3:
            # Переименовываем поле
            if "total" in event_data:
                event_data["total_amount"] = event_data.pop("total")

        return event_data

# При чтении событий
async def load_events(self, aggregate_id: str) -> List[DomainEvent]:
    rows = await self.connection.fetch("""
        SELECT event_type, event_data, version, schema_version
        FROM events WHERE aggregate_id = $1
        ORDER BY version
    """, aggregate_id)

    events = []
    for row in rows:
        data = row["event_data"]
        if row["schema_version"] < CURRENT_SCHEMA_VERSION:
            data = self.upgrader.upgrade(data, row["schema_version"])

        events.append(self._deserialize(row["event_type"], data))

    return events
```

### 3. Idempotent Event Handling

```python
class IdempotentEventHandler:
    def __init__(self, handler, processed_events_store):
        self.handler = handler
        self.processed_store = processed_events_store

    async def handle(self, event: DomainEvent):
        # Проверяем, обрабатывали ли уже
        if await self.processed_store.is_processed(event.event_id):
            logger.info(f"Event {event.event_id} already processed, skipping")
            return

        # Обрабатываем
        await self.handler.handle(event)

        # Помечаем как обработанное
        await self.processed_store.mark_processed(event.event_id)

class ProcessedEventsStore:
    async def is_processed(self, event_id: str) -> bool:
        result = await self.redis.sismember("processed_events", event_id)
        return bool(result)

    async def mark_processed(self, event_id: str):
        await self.redis.sadd("processed_events", event_id)
        # TTL для очистки старых записей
        await self.redis.expire("processed_events", 86400 * 7)
```

---

## Best Practices

### 1. Проектирование событий

```python
# ✅ Правильно: события содержат всю необходимую информацию
@dataclass(frozen=True)
class OrderShipped(DomainEvent):
    order_id: str
    customer_email: str  # Для отправки уведомления
    customer_name: str
    shipping_address: dict
    tracking_number: str
    carrier: str
    items: List[dict]  # Для деталей в уведомлении

# ❌ Неправильно: требуется дополнительный запрос
@dataclass(frozen=True)
class OrderShipped(DomainEvent):
    order_id: str
    # Consumer должен запросить данные у Order Service
```

### 2. Ordering и Partitioning

```python
# Kafka: гарантия порядка внутри partition
producer.send(
    topic='orders',
    key=order_id.encode(),  # Все события заказа в одной partition
    value=event_data
)

# Важно: связанные события должны иметь одинаковый ключ
# Чтобы сохранить порядок
```

### 3. Обработка ошибок в проекциях

```python
class ResilientProjectionRunner:
    async def run(self):
        while True:
            try:
                async for event in self.consumer.consume():
                    try:
                        await self.projection.handle_event(event)
                        await self.consumer.commit()
                    except TransientError:
                        # Retry с backoff
                        await asyncio.sleep(self.backoff)
                        raise  # Consumer перечитает событие
                    except PermanentError as e:
                        # Логируем и пропускаем
                        logger.error(f"Cannot process event: {e}")
                        await self.dead_letter.send(event, error=str(e))
                        await self.consumer.commit()
            except Exception as e:
                logger.error(f"Consumer error: {e}")
                await asyncio.sleep(5)
```

### 4. Мониторинг

```python
from prometheus_client import Counter, Histogram, Gauge

# Метрики для Event Sourcing
events_stored = Counter(
    'events_stored_total',
    'Total events stored',
    ['aggregate_type', 'event_type']
)

event_processing_time = Histogram(
    'event_processing_seconds',
    'Time to process event',
    ['handler']
)

projection_lag = Gauge(
    'projection_lag_events',
    'Number of events projection is behind',
    ['projection']
)

saga_state = Gauge(
    'saga_state',
    'Current state of sagas',
    ['saga_type', 'state']
)
```

---

## Типичные ошибки

### 1. Синхронная обработка событий

```python
# ❌ Неправильно: синхронная публикация в транзакции
async def create_order(order_data):
    async with db.transaction():
        order = await db.insert(order_data)
        await event_publisher.publish(OrderCreated(...))  # Если упадёт?
        return order

# ✅ Правильно: Transactional Outbox
async def create_order(order_data):
    async with db.transaction():
        order = await db.insert(order_data)
        # Событие сохраняется в той же транзакции
        await db.insert_outbox_event(OrderCreated(...))
        return order

# Отдельный процесс публикует события из outbox
```

### 2. Циклические зависимости в Choreography

```python
# ❌ Неправильно: A → B → A создаёт цикл
# Order Service публикует OrderCreated
# Payment Service обрабатывает → публикует PaymentRequested
# Order Service обрабатывает → публикует OrderUpdated
# ... бесконечный цикл

# ✅ Правильно: чёткое разделение ответственности
# Или использовать Orchestration для сложных саг
```

### 3. Потеря событий

```python
# ❌ Неправильно: события теряются при сбое
async def handle_event(event):
    await process(event)
    await consumer.commit()  # Если process() упал — событие потеряно

# ✅ Правильно: at-least-once с идемпотентностью
async def handle_event(event):
    if await is_already_processed(event.id):
        await consumer.commit()
        return

    await process(event)
    await mark_processed(event.id)
    await consumer.commit()
```

### 4. Блокирующий replay

```python
# ❌ Неправильно: блокируем систему на время replay
async def rebuild_all():
    events = await event_store.load_all()  # Миллионы событий!
    for event in events:
        await projection.handle(event)

# ✅ Правильно: инкрементальный replay
async def rebuild_incremental():
    last_position = await get_last_position()

    while True:
        events = await event_store.load_batch(
            from_position=last_position,
            batch_size=1000
        )
        if not events:
            break

        for event in events:
            await projection.handle(event)

        last_position = events[-1].position
        await save_last_position(last_position)
```

---

## Заключение

Event-Driven Architecture — мощный подход для построения масштабируемых и гибких систем:

1. **Event Sourcing** — храните события вместо состояния для полной истории и аудита
2. **CQRS** — разделяйте модели чтения и записи для оптимизации
3. **Saga** — управляйте распределёнными транзакциями через компенсации
4. **Choreography** — для простых сценариев с минимальной связанностью
5. **Orchestration** — для сложных бизнес-процессов с центральным контролем

Ключевые принципы:
- События неизменяемы (immutable)
- Обработчики идемпотентны
- Проектируйте события для consumers, не producers
- Используйте Transactional Outbox для надёжности
- Мониторьте lag проекций и состояние саг
