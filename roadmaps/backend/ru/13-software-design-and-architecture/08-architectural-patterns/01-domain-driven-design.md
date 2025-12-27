# Domain-Driven Design (DDD)

## Что такое Domain-Driven Design?

Domain-Driven Design (DDD) — это подход к разработке программного обеспечения, который фокусируется на моделировании предметной области (домена) бизнеса. DDD был представлен Эриком Эвансом в книге "Domain-Driven Design: Tackling Complexity in the Heart of Software" (2003).

Основная идея DDD — создание программного обеспечения, которое точно отражает бизнес-логику и терминологию предметной области.

```
┌─────────────────────────────────────────────────────────────┐
│                    Domain-Driven Design                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────┐    ┌─────────────────────┐        │
│  │   Стратегический    │    │    Тактический      │        │
│  │      уровень        │    │      уровень        │        │
│  ├─────────────────────┤    ├─────────────────────┤        │
│  │ • Bounded Context   │    │ • Entities          │        │
│  │ • Ubiquitous Lang   │    │ • Value Objects     │        │
│  │ • Context Map       │    │ • Aggregates        │        │
│  │ • Subdomains        │    │ • Repositories      │        │
│  │                     │    │ • Domain Services   │        │
│  │                     │    │ • Domain Events     │        │
│  └─────────────────────┘    └─────────────────────┘        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Стратегический уровень (Strategic Design)

Стратегический уровень DDD определяет, как структурировать всю систему на высоком уровне.

### Ubiquitous Language (Единый язык)

Единый язык — это общий словарь терминов, который используется как разработчиками, так и бизнес-экспертами. Все термины должны иметь одинаковое значение в коде, документации и разговорах.

```python
# Пример: термины из области электронной коммерции
# Единый язык: Order, Customer, Product, ShoppingCart, Checkout

class Order:
    """
    Заказ (Order) — оформленная покупка клиента.

    Термины единого языка:
    - place_order: размещение заказа
    - cancel_order: отмена заказа
    - order_total: общая сумма заказа
    - order_items: позиции заказа
    """

    def __init__(self, customer_id: str, items: list):
        self.customer_id = customer_id
        self.items = items  # order_items в терминах домена
        self.status = "pending"

    def place(self) -> None:
        """Размещение заказа"""
        if not self.items:
            raise ValueError("Cannot place an order with no items")
        self.status = "placed"

    def cancel(self) -> None:
        """Отмена заказа"""
        if self.status == "shipped":
            raise ValueError("Cannot cancel a shipped order")
        self.status = "cancelled"

    def calculate_total(self) -> float:
        """Расчёт общей суммы заказа"""
        return sum(item.price * item.quantity for item in self.items)
```

### Bounded Context (Ограниченный контекст)

Bounded Context — это логическая граница, внутри которой определённая доменная модель применяется последовательно. В разных контекстах одно и то же понятие может иметь разное значение.

```
┌──────────────────────────────────────────────────────────────────┐
│                        E-Commerce System                          │
├────────────────────┬────────────────────┬────────────────────────┤
│   Sales Context    │  Shipping Context  │   Billing Context      │
├────────────────────┼────────────────────┼────────────────────────┤
│                    │                    │                        │
│   ┌──────────┐    │   ┌──────────┐    │   ┌──────────┐         │
│   │  Order   │    │   │  Order   │    │   │  Order   │         │
│   │----------|    │   │----------|    │   │----------|         │
│   │ items    │    │   │ address  │    │   │ total    │         │
│   │ customer │    │   │ weight   │    │   │ payments │         │
│   │ discounts│    │   │ carrier  │    │   │ invoices │         │
│   └──────────┘    │   └──────────┘    │   └──────────┘         │
│                    │                    │                        │
│  Order = что       │  Order = куда      │  Order = сколько       │
│  покупают          │  доставить         │  платить               │
│                    │                    │                        │
└────────────────────┴────────────────────┴────────────────────────┘
```

```python
# Sales Context - Order содержит информацию о товарах
class SalesOrder:
    def __init__(self, order_id: str):
        self.order_id = order_id
        self.items: list[OrderItem] = []
        self.customer_id: str = ""
        self.discounts: list[Discount] = []

    def add_item(self, product_id: str, quantity: int, price: float):
        self.items.append(OrderItem(product_id, quantity, price))

    def apply_discount(self, discount: 'Discount'):
        self.discounts.append(discount)


# Shipping Context - Order содержит информацию о доставке
class ShippingOrder:
    def __init__(self, order_id: str):
        self.order_id = order_id
        self.delivery_address: str = ""
        self.weight: float = 0.0
        self.carrier: str = ""
        self.tracking_number: str = ""

    def assign_carrier(self, carrier: str):
        self.carrier = carrier

    def set_tracking(self, tracking_number: str):
        self.tracking_number = tracking_number


# Billing Context - Order содержит финансовую информацию
class BillingOrder:
    def __init__(self, order_id: str):
        self.order_id = order_id
        self.total_amount: float = 0.0
        self.payments: list['Payment'] = []
        self.invoices: list['Invoice'] = []

    def record_payment(self, payment: 'Payment'):
        self.payments.append(payment)

    def generate_invoice(self) -> 'Invoice':
        invoice = Invoice(self.order_id, self.total_amount)
        self.invoices.append(invoice)
        return invoice
```

### Context Map (Карта контекстов)

Context Map показывает отношения между различными Bounded Contexts и способы их интеграции.

```
┌─────────────────────────────────────────────────────────────────┐
│                         Context Map                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────┐         ┌─────────────┐                       │
│   │   Sales     │◄───────►│  Shipping   │                       │
│   │  Context    │   ACL   │  Context    │                       │
│   └──────┬──────┘         └─────────────┘                       │
│          │                                                       │
│          │ Shared Kernel                                         │
│          ▼                                                       │
│   ┌─────────────┐         ┌─────────────┐                       │
│   │  Billing    │◄───────►│  Inventory  │                       │
│   │  Context    │   OHS   │  Context    │                       │
│   └─────────────┘         └─────────────┘                       │
│                                                                  │
│   Типы отношений:                                               │
│   • ACL (Anti-Corruption Layer) — защитный слой                 │
│   • OHS (Open Host Service) — открытый сервис                   │
│   • Shared Kernel — общее ядро                                  │
│   • Customer/Supplier — клиент/поставщик                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Subdomains (Поддомены)

Поддомены — это части бизнес-домена, каждая со своей специализацией:

```python
# Core Domain (Основной домен) - главная ценность бизнеса
# Для e-commerce это может быть система рекомендаций
class RecommendationEngine:
    """Core Domain - уникальное конкурентное преимущество"""

    def get_personalized_recommendations(self, customer_id: str) -> list:
        # Сложная логика машинного обучения
        pass


# Supporting Subdomain (Поддерживающий поддомен)
# Необходим для работы Core Domain, но не уникален
class InventoryManagement:
    """Supporting Subdomain - поддержка основного бизнеса"""

    def check_availability(self, product_id: str) -> int:
        pass

    def reserve_stock(self, product_id: str, quantity: int) -> bool:
        pass


# Generic Subdomain (Общий поддомен)
# Стандартная функциональность, можно использовать готовые решения
class EmailNotification:
    """Generic Subdomain - стандартная функциональность"""

    def send_email(self, to: str, subject: str, body: str):
        # Можно использовать внешний сервис
        pass
```

## Тактический уровень (Tactical Design)

Тактический уровень DDD определяет строительные блоки для реализации доменной модели.

### Entity (Сущность)

Entity — объект с уникальной идентичностью, которая сохраняется во времени, даже если атрибуты меняются.

```python
from dataclasses import dataclass, field
from datetime import datetime
from uuid import uuid4
from typing import Optional


class Entity:
    """Базовый класс для сущностей"""

    def __init__(self, entity_id: Optional[str] = None):
        self._id = entity_id or str(uuid4())

    @property
    def id(self) -> str:
        return self._id

    def __eq__(self, other) -> bool:
        if not isinstance(other, Entity):
            return False
        return self._id == other._id

    def __hash__(self) -> int:
        return hash(self._id)


class Customer(Entity):
    """
    Сущность Customer.
    Идентичность определяется по customer_id,
    а не по имени или email.
    """

    def __init__(self, customer_id: str, name: str, email: str):
        super().__init__(customer_id)
        self.name = name
        self.email = email
        self.created_at = datetime.now()
        self.orders: list = []

    def change_email(self, new_email: str) -> None:
        """Email меняется, но клиент остаётся тем же"""
        if not self._is_valid_email(new_email):
            raise ValueError("Invalid email format")
        self.email = new_email

    def change_name(self, new_name: str) -> None:
        """Имя меняется, но клиент остаётся тем же"""
        if not new_name.strip():
            raise ValueError("Name cannot be empty")
        self.name = new_name

    def _is_valid_email(self, email: str) -> bool:
        return "@" in email and "." in email


# Два объекта Customer с одинаковым ID — это один клиент
customer1 = Customer("cust-123", "John", "john@example.com")
customer2 = Customer("cust-123", "John Smith", "john.smith@example.com")

print(customer1 == customer2)  # True - тот же клиент
```

### Value Object (Объект-значение)

Value Object — объект без идентичности, определяется только своими атрибутами. Value Objects должны быть неизменяемыми.

```python
from dataclasses import dataclass
from typing import Self


@dataclass(frozen=True)
class Money:
    """
    Value Object для денег.
    Неизменяемый, сравнивается по значению.
    """
    amount: float
    currency: str

    def __post_init__(self):
        if self.amount < 0:
            raise ValueError("Amount cannot be negative")
        if len(self.currency) != 3:
            raise ValueError("Currency must be 3 characters")

    def add(self, other: 'Money') -> 'Money':
        """Возвращает новый объект, не изменяет текущий"""
        if self.currency != other.currency:
            raise ValueError("Cannot add different currencies")
        return Money(self.amount + other.amount, self.currency)

    def multiply(self, factor: float) -> 'Money':
        return Money(self.amount * factor, self.currency)

    def __str__(self) -> str:
        return f"{self.amount:.2f} {self.currency}"


@dataclass(frozen=True)
class Address:
    """Value Object для адреса"""
    street: str
    city: str
    postal_code: str
    country: str

    def format_full(self) -> str:
        return f"{self.street}, {self.city}, {self.postal_code}, {self.country}"


@dataclass(frozen=True)
class Email:
    """Value Object для email с валидацией"""
    value: str

    def __post_init__(self):
        if not self._is_valid():
            raise ValueError(f"Invalid email: {self.value}")

    def _is_valid(self) -> bool:
        return "@" in self.value and "." in self.value.split("@")[1]

    @property
    def domain(self) -> str:
        return self.value.split("@")[1]


# Value Objects сравниваются по значению
money1 = Money(100.0, "USD")
money2 = Money(100.0, "USD")
print(money1 == money2)  # True - одинаковые значения

# Неизменяемость
total = money1.add(Money(50.0, "USD"))
print(money1)  # 100.00 USD - не изменился
print(total)   # 150.00 USD - новый объект
```

### Aggregate (Агрегат)

Aggregate — кластер связанных Entity и Value Objects, которые рассматриваются как единое целое. Aggregate имеет корень (Aggregate Root), через который происходит вся работа с агрегатом.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Order Aggregate                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌───────────────────────────────────────────────────────┐    │
│   │                  Order (Aggregate Root)                │    │
│   │   ───────────────────────────────────────────────────  │    │
│   │   - order_id: string                                   │    │
│   │   - customer_id: string                                │    │
│   │   - status: OrderStatus                                │    │
│   │   - items: list[OrderItem]                             │    │
│   │   - shipping_address: Address                          │    │
│   │   + add_item(item)                                     │    │
│   │   + remove_item(item_id)                               │    │
│   │   + place()                                            │    │
│   │   + cancel()                                           │    │
│   └───────────────────────────────────────────────────────┘    │
│              │                            │                      │
│              │ содержит                    │ содержит             │
│              ▼                            ▼                      │
│   ┌─────────────────────┐    ┌─────────────────────┐           │
│   │    OrderItem        │    │     Address         │           │
│   │    (Entity)         │    │   (Value Object)    │           │
│   │   ─────────────────  │    │   ─────────────────  │           │
│   │   - item_id         │    │   - street          │           │
│   │   - product_id      │    │   - city            │           │
│   │   - quantity        │    │   - postal_code     │           │
│   │   - price           │    │   - country         │           │
│   └─────────────────────┘    └─────────────────────┘           │
│                                                                  │
│   Правила:                                                      │
│   • Внешние объекты могут ссылаться только на корень            │
│   • Изменения внутри агрегата — только через корень             │
│   • Агрегат загружается и сохраняется как единое целое          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```python
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from typing import Optional
from uuid import uuid4


class OrderStatus(Enum):
    DRAFT = "draft"
    PLACED = "placed"
    PAID = "paid"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"


@dataclass
class OrderItem:
    """Entity внутри агрегата Order"""
    item_id: str
    product_id: str
    product_name: str
    quantity: int
    unit_price: Money

    @property
    def total_price(self) -> Money:
        return self.unit_price.multiply(self.quantity)

    def change_quantity(self, new_quantity: int) -> None:
        if new_quantity < 1:
            raise ValueError("Quantity must be at least 1")
        self.quantity = new_quantity


class Order:
    """
    Aggregate Root для заказа.
    Все изменения OrderItem происходят через Order.
    """

    def __init__(self, customer_id: str, shipping_address: Address):
        self._order_id = str(uuid4())
        self._customer_id = customer_id
        self._shipping_address = shipping_address
        self._items: list[OrderItem] = []
        self._status = OrderStatus.DRAFT
        self._created_at = datetime.now()
        self._domain_events: list = []

    @property
    def order_id(self) -> str:
        return self._order_id

    @property
    def status(self) -> OrderStatus:
        return self._status

    @property
    def items(self) -> tuple[OrderItem, ...]:
        """Возвращаем неизменяемую копию"""
        return tuple(self._items)

    @property
    def total(self) -> Money:
        if not self._items:
            return Money(0.0, "USD")
        total = self._items[0].total_price
        for item in self._items[1:]:
            total = total.add(item.total_price)
        return total

    # Бизнес-методы агрегата

    def add_item(self, product_id: str, product_name: str,
                 quantity: int, unit_price: Money) -> str:
        """Добавить товар в заказ"""
        self._ensure_draft_status()

        # Проверяем, есть ли уже такой товар
        existing_item = self._find_item_by_product(product_id)
        if existing_item:
            existing_item.change_quantity(existing_item.quantity + quantity)
            return existing_item.item_id

        item = OrderItem(
            item_id=str(uuid4()),
            product_id=product_id,
            product_name=product_name,
            quantity=quantity,
            unit_price=unit_price
        )
        self._items.append(item)
        return item.item_id

    def remove_item(self, item_id: str) -> None:
        """Удалить товар из заказа"""
        self._ensure_draft_status()

        item = self._find_item_by_id(item_id)
        if not item:
            raise ValueError(f"Item {item_id} not found")
        self._items.remove(item)

    def change_item_quantity(self, item_id: str, quantity: int) -> None:
        """Изменить количество товара"""
        self._ensure_draft_status()

        item = self._find_item_by_id(item_id)
        if not item:
            raise ValueError(f"Item {item_id} not found")
        item.change_quantity(quantity)

    def place(self) -> None:
        """Разместить заказ"""
        self._ensure_draft_status()

        if not self._items:
            raise ValueError("Cannot place an empty order")

        self._status = OrderStatus.PLACED
        self._domain_events.append(
            OrderPlacedEvent(self._order_id, self._customer_id, self.total)
        )

    def cancel(self) -> None:
        """Отменить заказ"""
        if self._status in (OrderStatus.SHIPPED, OrderStatus.DELIVERED):
            raise ValueError("Cannot cancel shipped or delivered order")

        self._status = OrderStatus.CANCELLED
        self._domain_events.append(
            OrderCancelledEvent(self._order_id, self._customer_id)
        )

    def change_shipping_address(self, new_address: Address) -> None:
        """Изменить адрес доставки"""
        if self._status not in (OrderStatus.DRAFT, OrderStatus.PLACED):
            raise ValueError("Cannot change address after payment")
        self._shipping_address = new_address

    # Приватные методы

    def _ensure_draft_status(self) -> None:
        if self._status != OrderStatus.DRAFT:
            raise ValueError("Can only modify draft orders")

    def _find_item_by_id(self, item_id: str) -> Optional[OrderItem]:
        return next((i for i in self._items if i.item_id == item_id), None)

    def _find_item_by_product(self, product_id: str) -> Optional[OrderItem]:
        return next((i for i in self._items if i.product_id == product_id), None)

    def collect_domain_events(self) -> list:
        """Забрать накопленные доменные события"""
        events = self._domain_events.copy()
        self._domain_events.clear()
        return events
```

### Repository (Репозиторий)

Repository — абстракция для работы с коллекцией агрегатов. Скрывает детали хранения данных.

```python
from abc import ABC, abstractmethod
from typing import Optional


class OrderRepository(ABC):
    """
    Абстрактный репозиторий для Order.
    Работает только с агрегатами целиком.
    """

    @abstractmethod
    def save(self, order: Order) -> None:
        """Сохранить агрегат"""
        pass

    @abstractmethod
    def get_by_id(self, order_id: str) -> Optional[Order]:
        """Получить агрегат по ID"""
        pass

    @abstractmethod
    def get_by_customer(self, customer_id: str) -> list[Order]:
        """Получить все заказы клиента"""
        pass

    @abstractmethod
    def delete(self, order_id: str) -> None:
        """Удалить агрегат"""
        pass


class InMemoryOrderRepository(OrderRepository):
    """In-memory реализация для тестирования"""

    def __init__(self):
        self._orders: dict[str, Order] = {}

    def save(self, order: Order) -> None:
        self._orders[order.order_id] = order

    def get_by_id(self, order_id: str) -> Optional[Order]:
        return self._orders.get(order_id)

    def get_by_customer(self, customer_id: str) -> list[Order]:
        return [
            order for order in self._orders.values()
            if order._customer_id == customer_id
        ]

    def delete(self, order_id: str) -> None:
        if order_id in self._orders:
            del self._orders[order_id]


class SQLAlchemyOrderRepository(OrderRepository):
    """Реализация с использованием SQLAlchemy"""

    def __init__(self, session):
        self._session = session

    def save(self, order: Order) -> None:
        # Преобразуем доменную модель в модель БД
        db_order = self._to_db_model(order)
        self._session.merge(db_order)
        self._session.commit()

    def get_by_id(self, order_id: str) -> Optional[Order]:
        db_order = self._session.query(OrderModel).get(order_id)
        if db_order:
            return self._to_domain_model(db_order)
        return None

    def get_by_customer(self, customer_id: str) -> list[Order]:
        db_orders = self._session.query(OrderModel)\
            .filter(OrderModel.customer_id == customer_id)\
            .all()
        return [self._to_domain_model(o) for o in db_orders]

    def delete(self, order_id: str) -> None:
        self._session.query(OrderModel)\
            .filter(OrderModel.order_id == order_id)\
            .delete()
        self._session.commit()

    def _to_db_model(self, order: Order):
        # Маппинг доменной модели в модель БД
        pass

    def _to_domain_model(self, db_order) -> Order:
        # Маппинг модели БД в доменную модель
        pass
```

### Domain Service (Доменный сервис)

Domain Service содержит бизнес-логику, которая не принадлежит ни одной сущности.

```python
class PricingService:
    """
    Доменный сервис для расчёта цен.
    Логика не принадлежит ни Order, ни Product.
    """

    def __init__(self, discount_repository, tax_service):
        self._discount_repository = discount_repository
        self._tax_service = tax_service

    def calculate_order_total(self, order: Order,
                               discount_code: Optional[str] = None) -> Money:
        """Рассчитать итоговую стоимость заказа"""
        subtotal = order.total

        # Применить скидку
        if discount_code:
            discount = self._discount_repository.get_by_code(discount_code)
            if discount and discount.is_valid():
                subtotal = discount.apply(subtotal)

        # Добавить налоги
        tax = self._tax_service.calculate_tax(subtotal, order._shipping_address)

        return subtotal.add(tax)


class TransferService:
    """
    Доменный сервис для перевода денег между счетами.
    Операция затрагивает два агрегата.
    """

    def __init__(self, account_repository):
        self._account_repository = account_repository

    def transfer(self, from_account_id: str,
                 to_account_id: str,
                 amount: Money) -> None:
        from_account = self._account_repository.get_by_id(from_account_id)
        to_account = self._account_repository.get_by_id(to_account_id)

        if not from_account or not to_account:
            raise ValueError("Account not found")

        from_account.withdraw(amount)
        to_account.deposit(amount)

        self._account_repository.save(from_account)
        self._account_repository.save(to_account)
```

### Domain Event (Доменное событие)

Domain Event — это запись о том, что что-то важное произошло в домене.

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Any


@dataclass(frozen=True)
class DomainEvent:
    """Базовый класс для доменных событий"""
    occurred_at: datetime = field(default_factory=datetime.now)

    @property
    def event_type(self) -> str:
        return self.__class__.__name__


@dataclass(frozen=True)
class OrderPlacedEvent(DomainEvent):
    """Событие: заказ размещён"""
    order_id: str
    customer_id: str
    total: Money


@dataclass(frozen=True)
class OrderCancelledEvent(DomainEvent):
    """Событие: заказ отменён"""
    order_id: str
    customer_id: str


@dataclass(frozen=True)
class PaymentReceivedEvent(DomainEvent):
    """Событие: платёж получен"""
    order_id: str
    payment_id: str
    amount: Money


class DomainEventPublisher:
    """Издатель доменных событий"""

    def __init__(self):
        self._handlers: dict[str, list] = {}

    def subscribe(self, event_type: str, handler) -> None:
        if event_type not in self._handlers:
            self._handlers[event_type] = []
        self._handlers[event_type].append(handler)

    def publish(self, event: DomainEvent) -> None:
        event_type = event.event_type
        if event_type in self._handlers:
            for handler in self._handlers[event_type]:
                handler(event)


# Использование событий
publisher = DomainEventPublisher()

# Подписчики на события
def send_order_confirmation(event: OrderPlacedEvent):
    print(f"Sending confirmation for order {event.order_id}")

def update_inventory(event: OrderPlacedEvent):
    print(f"Updating inventory for order {event.order_id}")

def notify_customer_cancellation(event: OrderCancelledEvent):
    print(f"Notifying customer about cancellation of {event.order_id}")

publisher.subscribe("OrderPlacedEvent", send_order_confirmation)
publisher.subscribe("OrderPlacedEvent", update_inventory)
publisher.subscribe("OrderCancelledEvent", notify_customer_cancellation)
```

### Factory (Фабрика)

Factory инкапсулирует сложную логику создания агрегатов.

```python
class OrderFactory:
    """Фабрика для создания заказов"""

    def __init__(self, customer_repository, product_repository):
        self._customer_repository = customer_repository
        self._product_repository = product_repository

    def create_order(self, customer_id: str,
                     shipping_address: Address,
                     items: list[dict]) -> Order:
        """
        Создать заказ с валидацией.

        items: [{"product_id": "...", "quantity": 1}, ...]
        """
        # Проверяем существование клиента
        customer = self._customer_repository.get_by_id(customer_id)
        if not customer:
            raise ValueError(f"Customer {customer_id} not found")

        # Создаём заказ
        order = Order(customer_id, shipping_address)

        # Добавляем товары с проверкой
        for item_data in items:
            product = self._product_repository.get_by_id(item_data["product_id"])
            if not product:
                raise ValueError(f"Product {item_data['product_id']} not found")

            if not product.is_available(item_data["quantity"]):
                raise ValueError(f"Product {product.name} is not available")

            order.add_item(
                product_id=product.product_id,
                product_name=product.name,
                quantity=item_data["quantity"],
                unit_price=product.price
            )

        return order

    def create_repeat_order(self, original_order_id: str,
                            order_repository) -> Order:
        """Создать повторный заказ на основе предыдущего"""
        original = order_repository.get_by_id(original_order_id)
        if not original:
            raise ValueError("Original order not found")

        new_order = Order(original._customer_id, original._shipping_address)

        for item in original.items:
            product = self._product_repository.get_by_id(item.product_id)
            if product and product.is_available(item.quantity):
                new_order.add_item(
                    product_id=item.product_id,
                    product_name=item.product_name,
                    quantity=item.quantity,
                    unit_price=product.price  # Используем текущую цену
                )

        return new_order
```

## Application Service (Сервис приложения)

Application Service координирует работу доменных объектов и не содержит бизнес-логики.

```python
class OrderApplicationService:
    """
    Сервис приложения для работы с заказами.
    Координирует репозитории, фабрики и доменные сервисы.
    """

    def __init__(self,
                 order_repository: OrderRepository,
                 order_factory: OrderFactory,
                 pricing_service: PricingService,
                 event_publisher: DomainEventPublisher):
        self._order_repository = order_repository
        self._order_factory = order_factory
        self._pricing_service = pricing_service
        self._event_publisher = event_publisher

    def create_order(self, customer_id: str,
                     shipping_address_data: dict,
                     items: list[dict]) -> str:
        """Создать новый заказ"""
        shipping_address = Address(
            street=shipping_address_data["street"],
            city=shipping_address_data["city"],
            postal_code=shipping_address_data["postal_code"],
            country=shipping_address_data["country"]
        )

        order = self._order_factory.create_order(
            customer_id, shipping_address, items
        )

        self._order_repository.save(order)
        return order.order_id

    def place_order(self, order_id: str) -> None:
        """Разместить заказ"""
        order = self._order_repository.get_by_id(order_id)
        if not order:
            raise ValueError("Order not found")

        order.place()
        self._order_repository.save(order)

        # Публикуем доменные события
        for event in order.collect_domain_events():
            self._event_publisher.publish(event)

    def cancel_order(self, order_id: str) -> None:
        """Отменить заказ"""
        order = self._order_repository.get_by_id(order_id)
        if not order:
            raise ValueError("Order not found")

        order.cancel()
        self._order_repository.save(order)

        for event in order.collect_domain_events():
            self._event_publisher.publish(event)

    def get_order_total(self, order_id: str,
                        discount_code: Optional[str] = None) -> Money:
        """Получить итоговую стоимость заказа"""
        order = self._order_repository.get_by_id(order_id)
        if not order:
            raise ValueError("Order not found")

        return self._pricing_service.calculate_order_total(order, discount_code)
```

## Архитектура с DDD

```
┌─────────────────────────────────────────────────────────────────┐
│                      Presentation Layer                          │
│              (Controllers, API Endpoints, Views)                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Application Layer                           │
│     (Application Services, DTOs, Command/Query Handlers)        │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Domain Layer                              │
│   (Entities, Value Objects, Aggregates, Domain Services,        │
│    Domain Events, Repository Interfaces)                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Infrastructure Layer                         │
│    (Repository Implementations, External Services, ORM,         │
│     Message Queues, Email Services)                              │
└─────────────────────────────────────────────────────────────────┘
```

## Плюсы и минусы DDD

### Плюсы

1. **Глубокое понимание домена** — команда глубоко понимает бизнес
2. **Единый язык** — улучшает коммуникацию между разработчиками и бизнесом
3. **Модульность** — Bounded Contexts создают чёткие границы модулей
4. **Гибкость** — легко адаптировать модель к изменениям бизнеса
5. **Тестируемость** — доменная логика изолирована и легко тестируется

### Минусы

1. **Сложность** — требует значительных усилий на изучение
2. **Время** — создание модели занимает много времени
3. **Overhead** — избыточен для простых приложений
4. **Кривая обучения** — команда должна освоить концепции DDD
5. **Требует экспертов** — нужны доменные эксперты для работы

## Когда использовать DDD

### Используйте DDD когда:

- Сложная бизнес-логика с множеством правил
- Долгоживущий проект с активным развитием
- Есть доступ к доменным экспертам
- Несколько команд работают над разными частями системы
- Бизнес-логика важнее технических деталей

### Не используйте DDD когда:

- Простое CRUD-приложение
- Прототип или MVP
- Нет доступа к доменным экспертам
- Маленькая команда, маленький проект
- Техническая сложность важнее бизнес-логики

## Практические рекомендации

1. **Начните с Event Storming** — визуальный метод для исследования домена
2. **Итеративно улучшайте модель** — модель эволюционирует со временем
3. **Не все контексты требуют DDD** — Generic Subdomains можно делать проще
4. **Фокус на Core Domain** — инвестируйте время в главную ценность бизнеса
5. **Избегайте анемичных моделей** — бизнес-логика должна быть в доменных объектах

## Заключение

Domain-Driven Design — мощный подход для создания сложных бизнес-приложений. Он требует значительных инвестиций в обучение и проектирование, но окупается в долгосрочных проектах с богатой бизнес-логикой. Ключ к успеху — тесное сотрудничество с доменными экспертами и итеративное развитие модели.
