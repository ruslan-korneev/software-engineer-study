# Architectural Boundaries

## Введение

Архитектурные границы (Architectural Boundaries) — это линии разделения между различными частями системы, которые защищают бизнес-логику от внешних изменений и позволяют развивать компоненты независимо друг от друга.

> "Архитектурные границы — это способ разделения системы на компоненты так, чтобы изменения в одних не влияли на другие"
> — Роберт Мартин, "Clean Architecture"

Правильно установленные границы позволяют:
- Изолировать изменения в рамках одного компонента
- Откладывать технические решения
- Тестировать компоненты независимо
- Заменять реализации без изменения бизнес-логики

## Типы архитектурных границ

### 1. Границы между слоями (Layer Boundaries)

Классическое разделение на слои: представление, бизнес-логика, данные.

```python
# Граница между слоями через абстракции

# === Слой домена (Domain Layer) ===
# domain/entities/order.py
from dataclasses import dataclass
from decimal import Decimal
from typing import List, Optional
from datetime import datetime


@dataclass
class OrderItem:
    product_id: int
    name: str
    price: Decimal
    quantity: int

    @property
    def subtotal(self) -> Decimal:
        return self.price * self.quantity


@dataclass
class Order:
    id: Optional[int]
    customer_id: int
    items: List[OrderItem]
    created_at: datetime
    status: str = "pending"

    @property
    def total(self) -> Decimal:
        return sum(item.subtotal for item in self.items)

    def can_be_cancelled(self) -> bool:
        return self.status in ("pending", "confirmed")

    def cancel(self) -> None:
        if not self.can_be_cancelled():
            raise ValueError(f"Cannot cancel order in {self.status} status")
        self.status = "cancelled"


# === Граница: порты (интерфейсы) ===
# domain/ports/repositories.py
from typing import Protocol, Optional, List


class OrderRepository(Protocol):
    """Порт для работы с хранилищем заказов."""

    def save(self, order: Order) -> Order:
        ...

    def find_by_id(self, order_id: int) -> Optional[Order]:
        ...

    def find_by_customer(self, customer_id: int) -> List[Order]:
        ...


class PaymentGateway(Protocol):
    """Порт для работы с платёжной системой."""

    def charge(self, amount: Decimal, payment_method: str) -> str:
        ...

    def refund(self, payment_id: str) -> bool:
        ...


# === Слой приложения (Application Layer) ===
# application/use_cases/cancel_order.py
from dataclasses import dataclass


@dataclass
class CancelOrderInput:
    order_id: int
    reason: str


@dataclass
class CancelOrderOutput:
    success: bool
    refund_id: Optional[str] = None
    message: str = ""


class CancelOrderUseCase:
    """Use case: отмена заказа.

    Зависит только от портов (абстракций), не от конкретных реализаций.
    """

    def __init__(
        self,
        order_repository: OrderRepository,
        payment_gateway: PaymentGateway
    ):
        self.orders = order_repository
        self.payments = payment_gateway

    def execute(self, input_data: CancelOrderInput) -> CancelOrderOutput:
        order = self.orders.find_by_id(input_data.order_id)

        if not order:
            return CancelOrderOutput(
                success=False,
                message="Order not found"
            )

        try:
            order.cancel()
        except ValueError as e:
            return CancelOrderOutput(success=False, message=str(e))

        # Возврат средств
        refund_id = None
        if order.payment_id:
            refund_id = self.payments.refund(order.payment_id)

        self.orders.save(order)

        return CancelOrderOutput(
            success=True,
            refund_id=refund_id,
            message=f"Order cancelled. Reason: {input_data.reason}"
        )


# === Слой инфраструктуры (Infrastructure Layer) ===
# infrastructure/persistence/sqlalchemy_order_repository.py
from sqlalchemy.orm import Session


class SQLAlchemyOrderRepository:
    """Конкретная реализация репозитория.

    Находится за границей, реализует порт OrderRepository.
    """

    def __init__(self, session: Session):
        self.session = session

    def save(self, order: Order) -> Order:
        model = self._to_model(order)
        self.session.add(model)
        self.session.commit()
        order.id = model.id
        return order

    def find_by_id(self, order_id: int) -> Optional[Order]:
        model = self.session.query(OrderModel).get(order_id)
        return self._to_entity(model) if model else None

    def _to_model(self, order: Order) -> "OrderModel":
        # Маппинг из доменной сущности в ORM-модель
        pass

    def _to_entity(self, model: "OrderModel") -> Order:
        # Маппинг из ORM-модели в доменную сущность
        pass
```

### 2. Границы между компонентами (Component Boundaries)

Разделение системы на независимые компоненты/модули.

```
ecommerce/
├── orders/                 # Компонент заказов
│   ├── domain/
│   ├── application/
│   ├── infrastructure/
│   └── api/
│
├── payments/               # Компонент платежей
│   ├── domain/
│   ├── application/
│   ├── infrastructure/
│   └── api/
│
├── inventory/              # Компонент склада
│   ├── domain/
│   ├── application/
│   ├── infrastructure/
│   └── api/
│
└── shared/                 # Общий код
    ├── domain/             # Общие value objects
    └── infrastructure/     # Общая инфраструктура
```

```python
# Взаимодействие между компонентами через контракты

# === Компонент Orders ===
# orders/domain/ports/inventory_service.py
from typing import Protocol


class InventoryService(Protocol):
    """Контракт для взаимодействия со складом.

    Определён в компоненте Orders, реализован в компоненте Inventory.
    """

    def reserve(self, product_id: int, quantity: int) -> str:
        """Зарезервировать товар, вернуть ID резервации."""
        ...

    def release(self, reservation_id: str) -> bool:
        """Освободить резервацию."""
        ...

    def check_availability(self, product_id: int) -> int:
        """Проверить доступное количество."""
        ...


# orders/application/use_cases/create_order.py
class CreateOrderUseCase:
    def __init__(
        self,
        order_repository: OrderRepository,
        inventory_service: InventoryService,  # Порт для другого компонента
        payment_gateway: PaymentGateway
    ):
        self.orders = order_repository
        self.inventory = inventory_service
        self.payments = payment_gateway

    def execute(self, input_data: CreateOrderInput) -> CreateOrderOutput:
        # Проверка доступности на складе (вызов другого компонента)
        for item in input_data.items:
            available = self.inventory.check_availability(item.product_id)
            if available < item.quantity:
                return CreateOrderOutput(
                    success=False,
                    message=f"Not enough stock for product {item.product_id}"
                )

        # Создание заказа
        order = Order(
            id=None,
            customer_id=input_data.customer_id,
            items=[...],
            created_at=datetime.now()
        )

        # Резервирование товаров
        reservations = []
        for item in order.items:
            reservation_id = self.inventory.reserve(
                item.product_id,
                item.quantity
            )
            reservations.append(reservation_id)

        # Оплата
        try:
            payment_id = self.payments.charge(
                order.total,
                input_data.payment_method
            )
            order.payment_id = payment_id
        except PaymentError:
            # Откат резерваций при ошибке оплаты
            for reservation_id in reservations:
                self.inventory.release(reservation_id)
            raise

        self.orders.save(order)

        return CreateOrderOutput(success=True, order_id=order.id)


# === Компонент Inventory ===
# inventory/infrastructure/adapters/inventory_service_adapter.py
from orders.domain.ports.inventory_service import InventoryService
from inventory.application.use_cases.reserve_stock import ReserveStockUseCase


class InventoryServiceAdapter(InventoryService):
    """Адаптер, реализующий контракт Orders через Inventory use cases."""

    def __init__(
        self,
        reserve_use_case: ReserveStockUseCase,
        stock_repository: StockRepository
    ):
        self.reserve_use_case = reserve_use_case
        self.stock_repository = stock_repository

    def reserve(self, product_id: int, quantity: int) -> str:
        result = self.reserve_use_case.execute(
            ReserveStockInput(product_id=product_id, quantity=quantity)
        )
        return result.reservation_id

    def release(self, reservation_id: str) -> bool:
        # ...
        pass

    def check_availability(self, product_id: int) -> int:
        stock = self.stock_repository.find_by_product(product_id)
        return stock.available_quantity if stock else 0
```

### 3. Границы между сервисами (Service Boundaries)

В микросервисной архитектуре границы проходят между отдельными сервисами.

```python
# Границы между сервисами через API

# === Order Service ===
# order_service/infrastructure/adapters/inventory_client.py
import httpx
from orders.domain.ports.inventory_service import InventoryService


class InventoryHttpClient(InventoryService):
    """HTTP-клиент для взаимодействия с Inventory Service.

    Реализует порт через HTTP-вызовы к другому сервису.
    """

    def __init__(self, base_url: str, timeout: int = 30):
        self.base_url = base_url
        self.timeout = timeout

    def reserve(self, product_id: int, quantity: int) -> str:
        response = httpx.post(
            f"{self.base_url}/api/v1/reservations",
            json={"product_id": product_id, "quantity": quantity},
            timeout=self.timeout
        )
        response.raise_for_status()
        return response.json()["reservation_id"]

    def release(self, reservation_id: str) -> bool:
        response = httpx.delete(
            f"{self.base_url}/api/v1/reservations/{reservation_id}",
            timeout=self.timeout
        )
        return response.status_code == 204

    def check_availability(self, product_id: int) -> int:
        response = httpx.get(
            f"{self.base_url}/api/v1/products/{product_id}/availability",
            timeout=self.timeout
        )
        response.raise_for_status()
        return response.json()["available_quantity"]


# === Anti-Corruption Layer (ACL) ===
# order_service/infrastructure/adapters/legacy_inventory_adapter.py
class LegacyInventoryAdapter(InventoryService):
    """Адаптер для устаревшей системы склада.

    Anti-Corruption Layer защищает домен от особенностей legacy-системы.
    """

    def __init__(self, legacy_client: "LegacyWarehouseClient"):
        self.legacy = legacy_client

    def reserve(self, product_id: int, quantity: int) -> str:
        # Legacy система использует другой формат
        legacy_sku = self._convert_to_legacy_sku(product_id)
        result = self.legacy.lock_inventory(
            sku=legacy_sku,
            qty=quantity,
            lock_type="SALES_ORDER"
        )
        return result["lock_reference"]

    def check_availability(self, product_id: int) -> int:
        legacy_sku = self._convert_to_legacy_sku(product_id)
        # Legacy возвращает данные в странном формате
        raw = self.legacy.get_stock_level(legacy_sku)
        # Преобразуем в наш формат
        return raw["total_qty"] - raw["reserved_qty"] - raw["damaged_qty"]

    def _convert_to_legacy_sku(self, product_id: int) -> str:
        # Маппинг между нашими ID и legacy SKU
        return f"SKU-{product_id:08d}"
```

### 4. Границы процессов (Process Boundaries)

Разделение синхронных и асинхронных операций.

```python
# Асинхронное взаимодействие через события

# === Определение событий ===
# shared/domain/events.py
from dataclasses import dataclass
from datetime import datetime
from decimal import Decimal
from typing import List


@dataclass
class DomainEvent:
    occurred_at: datetime
    event_id: str


@dataclass
class OrderCreatedEvent(DomainEvent):
    order_id: int
    customer_id: int
    total: Decimal
    items: List[dict]


@dataclass
class OrderCancelledEvent(DomainEvent):
    order_id: int
    reason: str
    refund_amount: Decimal


@dataclass
class PaymentCompletedEvent(DomainEvent):
    payment_id: str
    order_id: int
    amount: Decimal


# === Порт для публикации событий ===
# shared/domain/ports/event_publisher.py
from typing import Protocol


class EventPublisher(Protocol):
    def publish(self, event: DomainEvent) -> None:
        ...


# === Use Case с публикацией событий ===
# orders/application/use_cases/create_order.py
class CreateOrderUseCase:
    def __init__(
        self,
        order_repository: OrderRepository,
        event_publisher: EventPublisher
    ):
        self.orders = order_repository
        self.events = event_publisher

    def execute(self, input_data: CreateOrderInput) -> CreateOrderOutput:
        order = Order(...)
        saved_order = self.orders.save(order)

        # Публикация события вместо прямого вызова
        self.events.publish(OrderCreatedEvent(
            event_id=str(uuid.uuid4()),
            occurred_at=datetime.now(),
            order_id=saved_order.id,
            customer_id=saved_order.customer_id,
            total=saved_order.total,
            items=[...]
        ))

        return CreateOrderOutput(success=True, order_id=saved_order.id)


# === Реализация через RabbitMQ ===
# infrastructure/messaging/rabbitmq_publisher.py
import json
import pika
from shared.domain.events import DomainEvent
from shared.domain.ports.event_publisher import EventPublisher


class RabbitMQEventPublisher(EventPublisher):
    def __init__(self, connection_url: str, exchange: str):
        self.connection = pika.BlockingConnection(
            pika.URLParameters(connection_url)
        )
        self.channel = self.connection.channel()
        self.exchange = exchange

        self.channel.exchange_declare(
            exchange=exchange,
            exchange_type='topic',
            durable=True
        )

    def publish(self, event: DomainEvent) -> None:
        routing_key = self._get_routing_key(event)
        body = self._serialize(event)

        self.channel.basic_publish(
            exchange=self.exchange,
            routing_key=routing_key,
            body=body,
            properties=pika.BasicProperties(
                delivery_mode=2,  # persistent
                content_type='application/json'
            )
        )

    def _get_routing_key(self, event: DomainEvent) -> str:
        # order.created, order.cancelled, payment.completed
        event_type = type(event).__name__
        return event_type.replace("Event", "").lower().replace("_", ".")

    def _serialize(self, event: DomainEvent) -> bytes:
        return json.dumps(asdict(event), default=str).encode()


# === Обработчик событий в другом сервисе ===
# notification_service/application/event_handlers/order_handlers.py
class OrderEventHandler:
    def __init__(self, email_sender: EmailSender):
        self.email_sender = email_sender

    def handle_order_created(self, event: OrderCreatedEvent):
        """Реакция на создание заказа."""
        self.email_sender.send(
            to=self._get_customer_email(event.customer_id),
            subject="Order Confirmation",
            template="order_confirmation",
            context={"order_id": event.order_id, "total": event.total}
        )

    def handle_order_cancelled(self, event: OrderCancelledEvent):
        """Реакция на отмену заказа."""
        self.email_sender.send(
            to=self._get_customer_email_by_order(event.order_id),
            subject="Order Cancelled",
            template="order_cancelled",
            context={"order_id": event.order_id, "reason": event.reason}
        )
```

## Паттерны установления границ

### 1. Ports and Adapters (Hexagonal Architecture)

```
                    ┌─────────────────────────────────┐
                    │         Primary Adapters        │
                    │   (Controllers, CLI, Events)    │
                    └────────────────┬────────────────┘
                                     │
                                     ▼
┌───────────────┐   ┌─────────────────────────────────┐   ┌───────────────┐
│   Secondary   │   │                                 │   │    Primary    │
│   Adapters    │◄──│         Application Core        │──►│     Ports     │
│  (DB, APIs)   │   │   (Domain + Application Layer)  │   │ (Interfaces)  │
└───────────────┘   └─────────────────────────────────┘   └───────────────┘
        ▲                                                          │
        │           ┌─────────────────────────────────┐            │
        └───────────│        Secondary Ports          │◄───────────┘
                    │        (Repositories)           │
                    └─────────────────────────────────┘
```

```python
# Полный пример Hexagonal Architecture

# === Домен ===
# domain/order.py
@dataclass
class Order:
    id: Optional[int]
    customer_id: int
    items: List[OrderItem]
    status: OrderStatus

    def confirm(self) -> None:
        if self.status != OrderStatus.PENDING:
            raise DomainError("Can only confirm pending orders")
        self.status = OrderStatus.CONFIRMED


# === Primary Port (входящий) ===
# application/ports/incoming/order_service.py
from typing import Protocol


class OrderService(Protocol):
    """Primary port — определяет, что система может делать."""

    def create_order(
        self,
        customer_id: int,
        items: List[dict]
    ) -> Order:
        ...

    def confirm_order(self, order_id: int) -> Order:
        ...

    def cancel_order(self, order_id: int, reason: str) -> Order:
        ...


# === Secondary Ports (исходящие) ===
# application/ports/outgoing/repositories.py
class OrderRepository(Protocol):
    """Secondary port — определяет, что система требует."""

    def save(self, order: Order) -> Order:
        ...

    def find_by_id(self, order_id: int) -> Optional[Order]:
        ...


# application/ports/outgoing/notifications.py
class OrderNotifier(Protocol):
    def notify_order_confirmed(self, order: Order) -> None:
        ...


# === Реализация порта (Use Case) ===
# application/services/order_service_impl.py
class OrderServiceImpl(OrderService):
    """Реализация primary port."""

    def __init__(
        self,
        repository: OrderRepository,
        notifier: OrderNotifier
    ):
        self.repository = repository
        self.notifier = notifier

    def create_order(
        self,
        customer_id: int,
        items: List[dict]
    ) -> Order:
        order = Order(
            id=None,
            customer_id=customer_id,
            items=[OrderItem(**item) for item in items],
            status=OrderStatus.PENDING
        )
        return self.repository.save(order)

    def confirm_order(self, order_id: int) -> Order:
        order = self.repository.find_by_id(order_id)
        if not order:
            raise OrderNotFoundError(order_id)

        order.confirm()
        saved = self.repository.save(order)
        self.notifier.notify_order_confirmed(saved)

        return saved


# === Primary Adapter (REST API) ===
# adapters/incoming/rest/order_controller.py
from fastapi import APIRouter, Depends, HTTPException

router = APIRouter()


@router.post("/orders")
async def create_order(
    request: CreateOrderRequest,
    service: OrderService = Depends(get_order_service)
):
    """Primary adapter — преобразует HTTP в вызовы порта."""
    try:
        order = service.create_order(
            customer_id=request.customer_id,
            items=[item.dict() for item in request.items]
        )
        return OrderResponse.from_domain(order)
    except DomainError as e:
        raise HTTPException(status_code=400, detail=str(e))


# === Secondary Adapter (PostgreSQL) ===
# adapters/outgoing/persistence/postgres_order_repository.py
class PostgresOrderRepository(OrderRepository):
    """Secondary adapter — реализует исходящий порт."""

    def __init__(self, session: Session):
        self.session = session

    def save(self, order: Order) -> Order:
        model = OrderModel.from_domain(order)
        self.session.add(model)
        self.session.commit()
        return model.to_domain()


# === Secondary Adapter (Email) ===
# adapters/outgoing/notifications/email_notifier.py
class EmailOrderNotifier(OrderNotifier):
    """Secondary adapter для уведомлений."""

    def __init__(self, email_client: EmailClient):
        self.email_client = email_client

    def notify_order_confirmed(self, order: Order) -> None:
        self.email_client.send(
            to=self._get_customer_email(order.customer_id),
            subject="Order Confirmed",
            body=f"Your order #{order.id} has been confirmed!"
        )
```

### 2. Clean Architecture

```
┌────────────────────────────────────────────────────────────┐
│                    Frameworks & Drivers                     │
│  (Web, UI, DB, Devices, External Interfaces)               │
├────────────────────────────────────────────────────────────┤
│                  Interface Adapters                         │
│  (Controllers, Gateways, Presenters)                       │
├────────────────────────────────────────────────────────────┤
│               Application Business Rules                    │
│  (Use Cases)                                               │
├────────────────────────────────────────────────────────────┤
│               Enterprise Business Rules                     │
│  (Entities)                                                │
└────────────────────────────────────────────────────────────┘
```

```python
# Структура проекта Clean Architecture
"""
order_management/
├── domain/                      # Enterprise Business Rules
│   ├── entities/
│   │   ├── order.py
│   │   └── order_item.py
│   ├── value_objects/
│   │   ├── money.py
│   │   └── order_status.py
│   ├── services/
│   │   └── pricing_service.py
│   └── exceptions.py
│
├── application/                 # Application Business Rules
│   ├── use_cases/
│   │   ├── create_order.py
│   │   ├── confirm_order.py
│   │   └── cancel_order.py
│   ├── dto/
│   │   ├── order_dto.py
│   │   └── order_item_dto.py
│   └── interfaces/
│       ├── repositories.py
│       └── services.py
│
├── interface_adapters/          # Interface Adapters
│   ├── controllers/
│   │   └── order_controller.py
│   ├── presenters/
│   │   └── order_presenter.py
│   └── gateways/
│       └── order_gateway.py
│
└── frameworks_drivers/          # Frameworks & Drivers
    ├── web/
    │   └── fastapi_app.py
    ├── database/
    │   ├── sqlalchemy_models.py
    │   └── sqlalchemy_repository.py
    └── external/
        └── payment_client.py
"""
```

### 3. Bounded Context (DDD)

```python
# Разные контексты могут иметь разные модели одной сущности

# === Контекст Sales (продажи) ===
# sales/domain/customer.py
@dataclass
class Customer:
    """Клиент в контексте продаж."""
    id: int
    name: str
    credit_limit: Decimal
    discount_percentage: Decimal

    def can_place_order(self, amount: Decimal) -> bool:
        return amount <= self.credit_limit


# === Контекст Shipping (доставка) ===
# shipping/domain/customer.py
@dataclass
class Customer:
    """Клиент в контексте доставки — другая модель!"""
    id: int
    name: str
    shipping_address: Address
    preferred_carrier: str

    def get_delivery_instructions(self) -> str:
        return self.shipping_address.delivery_notes


# === Контекст Billing (биллинг) ===
# billing/domain/customer.py
@dataclass
class Customer:
    """Клиент в контексте биллинга."""
    id: int
    legal_name: str
    tax_id: str
    billing_address: Address
    payment_terms: int  # дней на оплату


# === Context Map: взаимодействие контекстов ===
# shared/context_map.py
"""
                    ┌─────────────────┐
                    │     Sales       │
                    │   (Upstream)    │
                    └────────┬────────┘
                             │
                    Customer Created Event
                             │
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│    Shipping     │ │    Billing      │ │   Marketing     │
│  (Downstream)   │ │  (Downstream)   │ │  (Downstream)   │
└─────────────────┘ └─────────────────┘ └─────────────────┘

Паттерны интеграции:
- Sales ↔ Shipping: Customer-Supplier (Sales диктует контракт)
- Sales ↔ Billing: Partnership (совместная разработка контракта)
- Sales → Marketing: Published Language (общий язык через события)
"""


# === Published Language (общие события) ===
# shared/integration_events/customer_events.py
@dataclass
class CustomerCreatedIntegrationEvent:
    """Событие, понятное всем контекстам."""
    customer_id: int
    email: str
    name: str
    created_at: datetime


# === Anti-Corruption Layer в downstream контексте ===
# shipping/infrastructure/acl/sales_customer_translator.py
class SalesCustomerTranslator:
    """ACL: переводит модель Sales в модель Shipping."""

    def translate(
        self,
        event: CustomerCreatedIntegrationEvent,
        address_service: AddressService
    ) -> "shipping.Customer":
        # Получаем адрес из другого источника
        address = address_service.get_default_address(event.customer_id)

        return shipping_domain.Customer(
            id=event.customer_id,
            name=event.name,
            shipping_address=address,
            preferred_carrier="standard"
        )
```

## Способы пересечения границ

### 1. Локальные вызовы (In-Process)

```python
# Простейший случай — вызов через интерфейс
class OrderController:
    def __init__(self, order_service: OrderService):
        self.order_service = order_service

    def create_order(self, request: Request):
        # Прямой вызов метода
        order = self.order_service.create_order(
            customer_id=request.customer_id,
            items=request.items
        )
        return OrderResponse.from_domain(order)
```

### 2. Сетевые вызовы (Remote Procedure Call)

```python
# REST API
class RemoteOrderService(OrderService):
    def __init__(self, base_url: str):
        self.base_url = base_url

    def create_order(self, customer_id: int, items: List[dict]) -> Order:
        response = httpx.post(
            f"{self.base_url}/orders",
            json={"customer_id": customer_id, "items": items}
        )
        response.raise_for_status()
        return Order(**response.json())


# gRPC
class GrpcOrderService(OrderService):
    def __init__(self, channel: grpc.Channel):
        self.stub = OrderServiceStub(channel)

    def create_order(self, customer_id: int, items: List[dict]) -> Order:
        request = CreateOrderRequest(
            customer_id=customer_id,
            items=[OrderItemProto(**item) for item in items]
        )
        response = self.stub.CreateOrder(request)
        return self._to_domain(response)
```

### 3. Асинхронные сообщения (Messaging)

```python
# Публикация события
class CreateOrderUseCase:
    def execute(self, input_data: CreateOrderInput) -> Order:
        order = self._create_order(input_data)

        # Асинхронное уведомление других систем
        self.message_bus.publish(
            "order.created",
            OrderCreatedMessage(
                order_id=order.id,
                total=order.total
            )
        )

        return order


# Подписка на события
class InventoryEventHandler:
    @subscribe("order.created")
    def handle_order_created(self, message: OrderCreatedMessage):
        # Обработка в другом процессе/сервисе
        self.reservation_service.reserve_items(message.order_id)
```

## Best Practices

### 1. Защита доменного слоя

```python
# Домен не должен зависеть от внешних слоёв
# domain/order.py
from dataclasses import dataclass
from decimal import Decimal

# НЕ импортируем ничего из infrastructure или application!
# from sqlalchemy import ...  # НЕЛЬЗЯ
# from fastapi import ...     # НЕЛЬЗЯ

@dataclass
class Order:
    """Чистая доменная сущность."""
    id: int | None
    items: list
    total: Decimal
```

### 2. Dependency Rule

```python
# Зависимости направлены ВНУТРЬ
# Внешние слои зависят от внутренних

# infrastructure/repository.py
from domain.order import Order  # OK: infra → domain
from domain.ports.repository import OrderRepository  # OK

# domain/order.py
# from infrastructure.models import OrderModel  # НЕЛЬЗЯ: domain → infra
```

### 3. Использование DTO на границах

```python
# Не передавайте доменные объекты через границы
# application/dto/order_dto.py
@dataclass
class OrderDTO:
    id: int
    customer_id: int
    total: float
    status: str

    @classmethod
    def from_domain(cls, order: Order) -> "OrderDTO":
        return cls(
            id=order.id,
            customer_id=order.customer_id,
            total=float(order.total),
            status=order.status.value
        )


# controllers/order_controller.py
@router.get("/orders/{order_id}")
async def get_order(order_id: int) -> OrderDTO:
    order = order_service.get_by_id(order_id)
    return OrderDTO.from_domain(order)  # DTO на выходе
```

## Типичные ошибки

### 1. Утечка доменных объектов

```python
# Плохо: доменная сущность возвращается напрямую в API
@router.get("/orders/{order_id}")
async def get_order(order_id: int) -> Order:  # Утечка домена!
    return order_repository.find_by_id(order_id)


# Хорошо: используем DTO
@router.get("/orders/{order_id}")
async def get_order(order_id: int) -> OrderResponse:
    order = order_repository.find_by_id(order_id)
    return OrderResponse.from_domain(order)
```

### 2. Нарушение направления зависимостей

```python
# Плохо: домен знает о БД
# domain/order.py
from sqlalchemy.orm import Session  # Нарушение!

class Order:
    def save(self, session: Session):
        session.add(self)
```

### 3. Анемичные границы

```python
# Плохо: граница есть, но она бесполезна
class OrderRepository(Protocol):
    def save(self, order: Order) -> Order: ...


class SQLOrderRepository(OrderRepository):
    def save(self, order: Order) -> Order:
        # Просто проксирует вызовы — нет реальной изоляции
        return self.session.add(order)
```

## Заключение

Архитектурные границы — это инструмент управления сложностью:

1. **Слои**: разделяют представление, логику и данные
2. **Компоненты**: изолируют бизнес-возможности
3. **Сервисы**: обеспечивают независимое развёртывание
4. **Процессы**: разделяют синхронные и асинхронные операции

Ключевые принципы:
- Dependency Rule: зависимости направлены внутрь
- Порты и адаптеры: абстракции на границах
- DTO: передача данных через границы
- ACL: защита от внешних изменений

Правильные границы обеспечивают:
- Независимость компонентов
- Возможность замены реализаций
- Изолированное тестирование
- Эволюцию системы
