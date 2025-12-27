# Policy vs Detail

## Введение

Разделение политики (Policy) и деталей (Detail) — это фундаментальный архитектурный принцип, который помогает создавать гибкие и поддерживаемые системы. Этот принцип был подробно описан Робертом Мартином в книге "Clean Architecture".

**Политика (Policy)** — это бизнес-правила, которые определяют, *что* система должна делать. Это ядро приложения, его основная ценность.

**Детали (Detail)** — это технические решения о том, *как* система реализует политику. Это база данных, веб-фреймворк, UI, внешние сервисы.

> "Хорошая архитектура максимально откладывает принятие решений о деталях"
> — Роберт Мартин

## Почему это важно?

### 1. Независимость от технологий

```python
# Плохо: бизнес-логика привязана к деталям (SQLAlchemy)
from sqlalchemy.orm import Session
from sqlalchemy import create_engine

class OrderService:
    def __init__(self):
        self.engine = create_engine("postgresql://localhost/db")

    def create_order(self, user_id: int, items: list) -> dict:
        with Session(self.engine) as session:
            # Бизнес-логика смешана с деталями БД
            user = session.query(User).filter_by(id=user_id).first()
            if not user:
                raise ValueError("User not found")

            total = sum(item["price"] * item["quantity"] for item in items)

            # Применение скидки — это политика!
            if total > 1000:
                total *= 0.9  # 10% скидка

            order = Order(user_id=user_id, total=total)
            session.add(order)
            session.commit()
            return {"order_id": order.id, "total": total}


# Хорошо: политика отделена от деталей
# domain/order.py (ПОЛИТИКА)
from dataclasses import dataclass
from decimal import Decimal
from typing import List

@dataclass
class OrderItem:
    product_id: int
    price: Decimal
    quantity: int

    @property
    def subtotal(self) -> Decimal:
        return self.price * self.quantity


@dataclass
class Order:
    user_id: int
    items: List[OrderItem]
    discount_percentage: Decimal = Decimal("0")

    @property
    def subtotal(self) -> Decimal:
        return sum(item.subtotal for item in self.items)

    @property
    def discount(self) -> Decimal:
        return self.subtotal * self.discount_percentage / 100

    @property
    def total(self) -> Decimal:
        return self.subtotal - self.discount


# domain/discount_policy.py (ПОЛИТИКА)
from decimal import Decimal

class DiscountPolicy:
    """Политика расчёта скидок."""

    BULK_ORDER_THRESHOLD = Decimal("1000")
    BULK_ORDER_DISCOUNT = Decimal("10")  # 10%

    def calculate_discount(self, order_subtotal: Decimal) -> Decimal:
        """Возвращает процент скидки."""
        if order_subtotal > self.BULK_ORDER_THRESHOLD:
            return self.BULK_ORDER_DISCOUNT
        return Decimal("0")
```

### 2. Тестируемость

```python
# Политику легко тестировать без базы данных и внешних сервисов
import pytest
from decimal import Decimal
from domain.order import Order, OrderItem
from domain.discount_policy import DiscountPolicy


class TestDiscountPolicy:
    def test_no_discount_for_small_orders(self):
        policy = DiscountPolicy()
        discount = policy.calculate_discount(Decimal("500"))
        assert discount == Decimal("0")

    def test_bulk_discount_applied(self):
        policy = DiscountPolicy()
        discount = policy.calculate_discount(Decimal("1500"))
        assert discount == Decimal("10")


class TestOrder:
    def test_total_calculation(self):
        items = [
            OrderItem(product_id=1, price=Decimal("100"), quantity=2),
            OrderItem(product_id=2, price=Decimal("50"), quantity=3),
        ]
        order = Order(user_id=1, items=items)

        assert order.subtotal == Decimal("350")
        assert order.total == Decimal("350")

    def test_total_with_discount(self):
        items = [
            OrderItem(product_id=1, price=Decimal("100"), quantity=2),
        ]
        order = Order(user_id=1, items=items, discount_percentage=Decimal("10"))

        assert order.subtotal == Decimal("200")
        assert order.discount == Decimal("20")
        assert order.total == Decimal("180")
```

## Что относится к политике?

### Бизнес-правила

```python
# domain/pricing.py
from decimal import Decimal
from dataclasses import dataclass
from datetime import datetime
from typing import Optional

@dataclass
class PricingRule:
    """Правило ценообразования."""
    min_quantity: int
    discount_percent: Decimal


class PricingPolicy:
    """Политика ценообразования — чистая бизнес-логика."""

    def __init__(self, rules: list[PricingRule]):
        self.rules = sorted(rules, key=lambda r: r.min_quantity, reverse=True)

    def calculate_unit_price(
        self,
        base_price: Decimal,
        quantity: int
    ) -> Decimal:
        """Расчёт цены за единицу с учётом количества."""
        for rule in self.rules:
            if quantity >= rule.min_quantity:
                discount = base_price * rule.discount_percent / 100
                return base_price - discount
        return base_price


# domain/inventory.py
class InventoryPolicy:
    """Политика управления запасами."""

    REORDER_THRESHOLD = 10
    SAFETY_STOCK = 5

    def should_reorder(self, current_stock: int, pending_orders: int) -> bool:
        """Определяет, нужно ли заказать пополнение."""
        available = current_stock - pending_orders
        return available <= self.REORDER_THRESHOLD

    def calculate_reorder_quantity(
        self,
        average_daily_sales: int,
        lead_time_days: int
    ) -> int:
        """Расчёт количества для заказа."""
        expected_demand = average_daily_sales * lead_time_days
        return expected_demand + self.SAFETY_STOCK
```

### Доменные сущности

```python
# domain/entities/user.py
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional
from enum import Enum

class UserStatus(Enum):
    PENDING = "pending"
    ACTIVE = "active"
    SUSPENDED = "suspended"


@dataclass
class User:
    """Доменная сущность пользователя — политика."""

    id: Optional[int]
    email: str
    name: str
    status: UserStatus = UserStatus.PENDING
    created_at: datetime = field(default_factory=datetime.now)

    def activate(self) -> None:
        """Активация пользователя."""
        if self.status == UserStatus.SUSPENDED:
            raise ValueError("Cannot activate suspended user")
        self.status = UserStatus.ACTIVE

    def suspend(self, reason: str) -> None:
        """Приостановка аккаунта."""
        if self.status != UserStatus.ACTIVE:
            raise ValueError("Can only suspend active users")
        self.status = UserStatus.SUSPENDED

    def can_place_order(self) -> bool:
        """Может ли пользователь делать заказы."""
        return self.status == UserStatus.ACTIVE


# domain/entities/payment.py
from decimal import Decimal
from enum import Enum

class PaymentStatus(Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"
    REFUNDED = "refunded"


@dataclass
class Payment:
    """Платёж — доменная сущность."""

    id: Optional[int]
    order_id: int
    amount: Decimal
    status: PaymentStatus = PaymentStatus.PENDING

    def can_refund(self) -> bool:
        """Можно ли сделать возврат."""
        return self.status == PaymentStatus.COMPLETED

    def process(self) -> None:
        """Начать обработку платежа."""
        if self.status != PaymentStatus.PENDING:
            raise ValueError(f"Cannot process payment in {self.status} status")
        self.status = PaymentStatus.PROCESSING

    def complete(self) -> None:
        """Завершить платёж успешно."""
        if self.status != PaymentStatus.PROCESSING:
            raise ValueError("Payment must be processing to complete")
        self.status = PaymentStatus.COMPLETED

    def fail(self) -> None:
        """Отметить платёж как неудачный."""
        if self.status != PaymentStatus.PROCESSING:
            raise ValueError("Payment must be processing to fail")
        self.status = PaymentStatus.FAILED
```

### Use Cases (Варианты использования)

```python
# application/use_cases/place_order.py
from dataclasses import dataclass
from decimal import Decimal
from typing import Protocol

from domain.entities.order import Order, OrderItem
from domain.discount_policy import DiscountPolicy


# Порты — абстракции для деталей
class OrderRepository(Protocol):
    def save(self, order: Order) -> Order: ...
    def find_by_id(self, order_id: int) -> Order | None: ...


class UserRepository(Protocol):
    def find_by_id(self, user_id: int) -> "User | None": ...


class PaymentGateway(Protocol):
    def charge(self, amount: Decimal, payment_method: str) -> str: ...


@dataclass
class PlaceOrderInput:
    user_id: int
    items: list[dict]
    payment_method: str


@dataclass
class PlaceOrderOutput:
    order_id: int
    total: Decimal
    payment_id: str


class PlaceOrderUseCase:
    """Вариант использования: размещение заказа.

    Это политика — оркестрация бизнес-правил.
    Зависит от абстракций (Protocol), а не от конкретных реализаций.
    """

    def __init__(
        self,
        order_repository: OrderRepository,
        user_repository: UserRepository,
        payment_gateway: PaymentGateway,
        discount_policy: DiscountPolicy
    ):
        self.order_repository = order_repository
        self.user_repository = user_repository
        self.payment_gateway = payment_gateway
        self.discount_policy = discount_policy

    def execute(self, input_data: PlaceOrderInput) -> PlaceOrderOutput:
        # Получить пользователя
        user = self.user_repository.find_by_id(input_data.user_id)
        if not user:
            raise ValueError("User not found")

        if not user.can_place_order():
            raise ValueError("User cannot place orders")

        # Создать заказ
        items = [
            OrderItem(
                product_id=item["product_id"],
                price=Decimal(str(item["price"])),
                quantity=item["quantity"]
            )
            for item in input_data.items
        ]

        order = Order(user_id=user.id, items=items)

        # Применить политику скидок
        discount = self.discount_policy.calculate_discount(order.subtotal)
        order.discount_percentage = discount

        # Провести оплату
        payment_id = self.payment_gateway.charge(
            order.total,
            input_data.payment_method
        )

        # Сохранить заказ
        saved_order = self.order_repository.save(order)

        return PlaceOrderOutput(
            order_id=saved_order.id,
            total=order.total,
            payment_id=payment_id
        )
```

## Что относится к деталям?

### База данных

```python
# infrastructure/persistence/sqlalchemy/models.py (ДЕТАЛЬ)
from sqlalchemy import Column, Integer, String, Numeric, ForeignKey, Enum
from sqlalchemy.orm import relationship, declarative_base

Base = declarative_base()


class UserModel(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    email = Column(String(255), unique=True, nullable=False)
    name = Column(String(255), nullable=False)
    status = Column(String(50), default="pending")

    orders = relationship("OrderModel", back_populates="user")


class OrderModel(Base):
    __tablename__ = "orders"

    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    total = Column(Numeric(10, 2), nullable=False)
    discount_percentage = Column(Numeric(5, 2), default=0)

    user = relationship("UserModel", back_populates="orders")
    items = relationship("OrderItemModel", back_populates="order")


# infrastructure/persistence/sqlalchemy/repositories.py (ДЕТАЛЬ)
from sqlalchemy.orm import Session
from domain.entities.order import Order, OrderItem
from application.use_cases.place_order import OrderRepository


class SQLAlchemyOrderRepository(OrderRepository):
    """Конкретная реализация репозитория — это деталь."""

    def __init__(self, session: Session):
        self.session = session

    def save(self, order: Order) -> Order:
        model = OrderModel(
            user_id=order.user_id,
            total=order.total,
            discount_percentage=order.discount_percentage
        )
        self.session.add(model)
        self.session.commit()
        order.id = model.id
        return order

    def find_by_id(self, order_id: int) -> Order | None:
        model = self.session.query(OrderModel).filter_by(id=order_id).first()
        if not model:
            return None
        return self._to_domain(model)

    def _to_domain(self, model: OrderModel) -> Order:
        items = [
            OrderItem(
                product_id=item.product_id,
                price=item.price,
                quantity=item.quantity
            )
            for item in model.items
        ]
        return Order(
            id=model.id,
            user_id=model.user_id,
            items=items,
            discount_percentage=model.discount_percentage
        )
```

### Веб-фреймворк

```python
# infrastructure/web/fastapi/routes.py (ДЕТАЛЬ)
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel
from decimal import Decimal

from application.use_cases.place_order import PlaceOrderUseCase, PlaceOrderInput
from infrastructure.dependencies import get_place_order_use_case

router = APIRouter(prefix="/orders", tags=["orders"])


class OrderItemRequest(BaseModel):
    product_id: int
    price: float
    quantity: int


class PlaceOrderRequest(BaseModel):
    items: list[OrderItemRequest]
    payment_method: str


class PlaceOrderResponse(BaseModel):
    order_id: int
    total: float
    payment_id: str


@router.post("/", response_model=PlaceOrderResponse)
async def place_order(
    request: PlaceOrderRequest,
    user_id: int,  # Из аутентификации
    use_case: PlaceOrderUseCase = Depends(get_place_order_use_case)
):
    """REST endpoint — это деталь доставки."""
    try:
        result = use_case.execute(PlaceOrderInput(
            user_id=user_id,
            items=[item.dict() for item in request.items],
            payment_method=request.payment_method
        ))
        return PlaceOrderResponse(
            order_id=result.order_id,
            total=float(result.total),
            payment_id=result.payment_id
        )
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
```

### Внешние сервисы

```python
# infrastructure/external/stripe_gateway.py (ДЕТАЛЬ)
import stripe
from decimal import Decimal
from application.use_cases.place_order import PaymentGateway


class StripePaymentGateway(PaymentGateway):
    """Интеграция со Stripe — это деталь."""

    def __init__(self, api_key: str):
        stripe.api_key = api_key

    def charge(self, amount: Decimal, payment_method: str) -> str:
        # Конвертируем в центы для Stripe
        amount_cents = int(amount * 100)

        payment_intent = stripe.PaymentIntent.create(
            amount=amount_cents,
            currency="usd",
            payment_method=payment_method,
            confirm=True
        )

        return payment_intent.id


# infrastructure/external/paypal_gateway.py (ДЕТАЛЬ)
import paypalrestsdk
from decimal import Decimal
from application.use_cases.place_order import PaymentGateway


class PayPalPaymentGateway(PaymentGateway):
    """Интеграция с PayPal — альтернативная деталь."""

    def __init__(self, client_id: str, client_secret: str):
        paypalrestsdk.configure({
            "mode": "sandbox",
            "client_id": client_id,
            "client_secret": client_secret
        })

    def charge(self, amount: Decimal, payment_method: str) -> str:
        payment = paypalrestsdk.Payment({
            "intent": "sale",
            "payer": {"payment_method": "paypal"},
            "transactions": [{
                "amount": {
                    "total": str(amount),
                    "currency": "USD"
                }
            }]
        })

        if payment.create():
            return payment.id
        raise Exception(payment.error)
```

## Правило зависимостей

```
                    ┌─────────────────┐
                    │   Presentation  │  (детали)
                    │   Web, CLI, UI  │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │   Application   │
                    │   Use Cases     │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │     Domain      │  (политика)
                    │ Entities, Rules │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  Infrastructure │  (детали)
                    │   DB, External  │
                    └─────────────────┘

Зависимости направлены ВНУТРЬ, к политике.
Политика НЕ знает о деталях.
```

```python
# Правильное направление зависимостей

# domain/ — не импортирует ничего из infrastructure или presentation
# domain/discount_policy.py
class DiscountPolicy:
    # Чистая бизнес-логика, никаких внешних зависимостей
    pass


# application/ — импортирует только из domain
# application/use_cases/place_order.py
from domain.discount_policy import DiscountPolicy  # OK
from domain.entities.order import Order  # OK
# from infrastructure.persistence import OrderRepository  # НЕЛЬЗЯ!


# infrastructure/ — импортирует из domain и application
# infrastructure/persistence/repositories.py
from domain.entities.order import Order  # OK
from application.use_cases.place_order import OrderRepository  # OK (интерфейс)


# presentation/ — импортирует из application
# presentation/api/routes.py
from application.use_cases.place_order import PlaceOrderUseCase  # OK
```

## Инверсия зависимостей на практике

```python
# Проблема: Use Case зависит от конкретной реализации
class PlaceOrderUseCase:
    def __init__(self):
        # Жёсткая зависимость от деталей!
        self.repository = SQLAlchemyOrderRepository()
        self.payment = StripePaymentGateway()


# Решение: Use Case зависит от абстракций
from typing import Protocol


class OrderRepository(Protocol):
    """Абстракция репозитория — определена в application слое."""
    def save(self, order: Order) -> Order: ...
    def find_by_id(self, order_id: int) -> Order | None: ...


class PaymentGateway(Protocol):
    """Абстракция платёжного шлюза."""
    def charge(self, amount: Decimal, payment_method: str) -> str: ...


class PlaceOrderUseCase:
    def __init__(
        self,
        repository: OrderRepository,  # Абстракция
        payment: PaymentGateway       # Абстракция
    ):
        self.repository = repository
        self.payment = payment


# Сборка зависимостей происходит на уровне инфраструктуры
# infrastructure/dependencies.py
from sqlalchemy.orm import Session
from infrastructure.persistence.repositories import SQLAlchemyOrderRepository
from infrastructure.external.stripe_gateway import StripePaymentGateway
from application.use_cases.place_order import PlaceOrderUseCase


def get_place_order_use_case(session: Session) -> PlaceOrderUseCase:
    return PlaceOrderUseCase(
        repository=SQLAlchemyOrderRepository(session),
        payment=StripePaymentGateway(api_key="sk_test_xxx")
    )
```

## Best Practices

### 1. Откладывайте решения о деталях

```python
# Начните с политики, используйте заглушки для деталей
class InMemoryOrderRepository(OrderRepository):
    """Заглушка для разработки и тестирования."""

    def __init__(self):
        self._orders: dict[int, Order] = {}
        self._next_id = 1

    def save(self, order: Order) -> Order:
        order.id = self._next_id
        self._orders[order.id] = order
        self._next_id += 1
        return order

    def find_by_id(self, order_id: int) -> Order | None:
        return self._orders.get(order_id)


class FakePaymentGateway(PaymentGateway):
    """Заглушка для платежей."""

    def charge(self, amount: Decimal, payment_method: str) -> str:
        return f"fake_payment_{payment_method}"
```

### 2. Тестируйте политику изолированно

```python
# tests/unit/test_place_order.py
import pytest
from decimal import Decimal
from application.use_cases.place_order import PlaceOrderUseCase, PlaceOrderInput
from domain.discount_policy import DiscountPolicy


class TestPlaceOrder:
    def test_order_with_bulk_discount(self):
        # Arrange
        repository = InMemoryOrderRepository()
        user_repository = InMemoryUserRepository()
        user_repository.save(User(id=1, email="test@test.com", name="Test"))

        payment = FakePaymentGateway()
        policy = DiscountPolicy()

        use_case = PlaceOrderUseCase(
            order_repository=repository,
            user_repository=user_repository,
            payment_gateway=payment,
            discount_policy=policy
        )

        # Act
        result = use_case.execute(PlaceOrderInput(
            user_id=1,
            items=[{"product_id": 1, "price": 500, "quantity": 3}],
            payment_method="card_xxx"
        ))

        # Assert
        assert result.total == Decimal("1350")  # 1500 - 10% скидка
```

### 3. Группируйте детали по механизму доставки

```
infrastructure/
├── persistence/          # Детали хранения данных
│   ├── sqlalchemy/
│   ├── mongodb/
│   └── redis/
├── external/             # Внешние сервисы
│   ├── stripe/
│   ├── sendgrid/
│   └── twilio/
├── web/                  # Веб-фреймворки
│   ├── fastapi/
│   └── flask/
└── messaging/            # Очереди сообщений
    ├── rabbitmq/
    └── kafka/
```

## Типичные ошибки

### 1. Доменные сущности знают о БД

```python
# Плохо
from sqlalchemy import Column, Integer, String
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class User(Base):  # Доменная сущность привязана к SQLAlchemy!
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String)

    def can_place_order(self) -> bool:
        return self.status == "active"
```

### 2. Бизнес-логика в контроллерах

```python
# Плохо
@router.post("/orders")
async def create_order(request: OrderRequest, db: Session = Depends(get_db)):
    # Бизнес-логика в контроллере!
    total = sum(item.price * item.quantity for item in request.items)
    if total > 1000:
        total *= 0.9

    order = OrderModel(total=total)
    db.add(order)
    db.commit()
    return {"order_id": order.id}
```

### 3. Use Case зависит от конкретного фреймворка

```python
# Плохо
from fastapi import Request  # Use Case знает о FastAPI!

class PlaceOrderUseCase:
    def execute(self, request: Request):  # Зависимость от фреймворка!
        data = await request.json()
        # ...
```

## Заключение

Разделение политики и деталей — это ключ к созданию гибкой архитектуры:

1. **Политика** — бизнес-правила, доменные сущности, use cases
2. **Детали** — БД, веб-фреймворк, UI, внешние сервисы
3. **Зависимости** направлены от деталей к политике
4. **Политика** не знает о деталях благодаря инверсии зависимостей
5. **Детали** можно менять без изменения бизнес-логики

Следуя этому принципу, вы сможете:
- Легко менять базу данных или фреймворк
- Тестировать бизнес-логику изолированно
- Откладывать технические решения
- Поддерживать систему долгие годы
