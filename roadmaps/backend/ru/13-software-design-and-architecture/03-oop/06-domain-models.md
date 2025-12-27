# Domain Models

## Доменные модели

Доменная модель (Domain Model) — это концептуальная модель предметной области, которая включает как данные, так и поведение. Это один из ключевых паттернов в Domain-Driven Design (DDD), позволяющий точно отразить бизнес-логику в коде.

---

## Что такое доменная модель

### Определение

Доменная модель — это объектная модель, которая:
- Отражает сущности и процессы реального мира
- Инкапсулирует бизнес-правила и логику
- Не зависит от инфраструктуры (БД, UI, API)

### Доменная модель vs Анемичная модель

```python
# Анемичная модель — ПЛОХО
# Данные отделены от поведения

class OrderData:
    """Только данные, без логики."""
    def __init__(self):
        self.items: list = []
        self.status: str = "new"
        self.total: float = 0


class OrderService:
    """Вся логика в отдельном сервисе."""

    def add_item(self, order: OrderData, item: dict) -> None:
        order.items.append(item)
        order.total += item["price"] * item["quantity"]

    def submit(self, order: OrderData) -> None:
        if not order.items:
            raise ValueError("Cannot submit empty order")
        order.status = "submitted"


# Богатая доменная модель — ХОРОШО
# Данные и поведение вместе

from dataclasses import dataclass, field
from decimal import Decimal
from enum import Enum
from typing import List


class OrderStatus(Enum):
    NEW = "new"
    SUBMITTED = "submitted"
    PAID = "paid"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"


@dataclass
class OrderItem:
    product_id: str
    name: str
    price: Decimal
    quantity: int

    @property
    def subtotal(self) -> Decimal:
        return self.price * self.quantity


class Order:
    """Богатая доменная модель с инкапсулированной логикой."""

    def __init__(self, customer_id: str):
        self._customer_id = customer_id
        self._items: List[OrderItem] = []
        self._status = OrderStatus.NEW

    @property
    def customer_id(self) -> str:
        return self._customer_id

    @property
    def status(self) -> OrderStatus:
        return self._status

    @property
    def items(self) -> tuple[OrderItem, ...]:
        return tuple(self._items)  # Возвращаем immutable копию

    @property
    def total(self) -> Decimal:
        return sum(item.subtotal for item in self._items)

    def add_item(self, product_id: str, name: str,
                 price: Decimal, quantity: int) -> None:
        """Добавить товар в заказ."""
        if self._status != OrderStatus.NEW:
            raise ValueError("Cannot modify submitted order")
        if quantity <= 0:
            raise ValueError("Quantity must be positive")

        # Если товар уже есть — увеличиваем количество
        for item in self._items:
            if item.product_id == product_id:
                # Создаём новый item с обновлённым количеством
                self._items.remove(item)
                self._items.append(OrderItem(
                    product_id, name, price, item.quantity + quantity
                ))
                return

        self._items.append(OrderItem(product_id, name, price, quantity))

    def submit(self) -> None:
        """Отправить заказ на обработку."""
        if not self._items:
            raise ValueError("Cannot submit empty order")
        if self._status != OrderStatus.NEW:
            raise ValueError(f"Cannot submit order in status {self._status}")

        self._status = OrderStatus.SUBMITTED

    def cancel(self) -> None:
        """Отменить заказ."""
        if self._status in (OrderStatus.SHIPPED, OrderStatus.DELIVERED):
            raise ValueError("Cannot cancel shipped order")

        self._status = OrderStatus.CANCELLED
```

---

## Ключевые элементы доменной модели

### Entity (Сущность)

Объект с уникальной идентичностью, которая сохраняется на протяжении всего жизненного цикла.

```python
from dataclasses import dataclass
from datetime import datetime
from uuid import UUID, uuid4


@dataclass
class UserId:
    """Value Object для идентификатора пользователя."""
    value: UUID

    @classmethod
    def generate(cls) -> "UserId":
        return cls(uuid4())


class User:
    """Entity — имеет уникальную идентичность."""

    def __init__(self, user_id: UserId, email: str, name: str):
        self._id = user_id
        self._email = email
        self._name = name
        self._created_at = datetime.now()
        self._is_active = True

    @property
    def id(self) -> UserId:
        return self._id

    @property
    def email(self) -> str:
        return self._email

    @property
    def name(self) -> str:
        return self._name

    @property
    def is_active(self) -> bool:
        return self._is_active

    def change_email(self, new_email: str) -> None:
        """Бизнес-правило: email можно менять."""
        if not self._is_active:
            raise ValueError("Cannot change email for inactive user")
        # Валидация email
        if "@" not in new_email:
            raise ValueError("Invalid email format")
        self._email = new_email

    def deactivate(self) -> None:
        """Деактивировать пользователя."""
        self._is_active = False

    def __eq__(self, other: object) -> bool:
        """Сущности равны, если равны их ID."""
        if not isinstance(other, User):
            return NotImplemented
        return self._id == other._id

    def __hash__(self) -> int:
        return hash(self._id)
```

### Value Object (Объект-значение)

Неизменяемый объект без идентичности, определяемый только своими атрибутами.

```python
from dataclasses import dataclass
from decimal import Decimal


@dataclass(frozen=True)
class Money:
    """Value Object для денежных сумм."""
    amount: Decimal
    currency: str

    def __post_init__(self):
        if self.amount < 0:
            raise ValueError("Amount cannot be negative")
        if len(self.currency) != 3:
            raise ValueError("Currency must be 3-letter code")

    def add(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError("Cannot add different currencies")
        return Money(self.amount + other.amount, self.currency)

    def subtract(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError("Cannot subtract different currencies")
        return Money(self.amount - other.amount, self.currency)

    def multiply(self, factor: Decimal) -> "Money":
        return Money(self.amount * factor, self.currency)


@dataclass(frozen=True)
class Address:
    """Value Object для адреса."""
    street: str
    city: str
    postal_code: str
    country: str

    def __str__(self) -> str:
        return f"{self.street}, {self.city}, {self.postal_code}, {self.country}"


@dataclass(frozen=True)
class DateRange:
    """Value Object для диапазона дат."""
    start: datetime
    end: datetime

    def __post_init__(self):
        if self.start > self.end:
            raise ValueError("Start date must be before end date")

    def contains(self, date: datetime) -> bool:
        return self.start <= date <= self.end

    def overlaps(self, other: "DateRange") -> bool:
        return self.start <= other.end and other.start <= self.end

    @property
    def duration_days(self) -> int:
        return (self.end - self.start).days
```

### Aggregate (Агрегат)

Кластер связанных объектов, рассматриваемых как единое целое. Агрегат имеет корень (Aggregate Root), через который происходит всё взаимодействие.

```python
from dataclasses import dataclass, field
from datetime import datetime
from decimal import Decimal
from enum import Enum
from typing import List, Optional
from uuid import UUID, uuid4


class PaymentStatus(Enum):
    PENDING = "pending"
    COMPLETED = "completed"
    FAILED = "failed"
    REFUNDED = "refunded"


@dataclass(frozen=True)
class PaymentId:
    value: UUID

    @classmethod
    def generate(cls) -> "PaymentId":
        return cls(uuid4())


@dataclass(frozen=True)
class InvoiceId:
    value: UUID

    @classmethod
    def generate(cls) -> "InvoiceId":
        return cls(uuid4())


@dataclass
class Payment:
    """Часть агрегата Invoice — не может существовать отдельно."""
    id: PaymentId
    amount: Money
    status: PaymentStatus
    paid_at: Optional[datetime] = None

    def complete(self) -> None:
        if self.status != PaymentStatus.PENDING:
            raise ValueError("Can only complete pending payment")
        self.status = PaymentStatus.COMPLETED
        self.paid_at = datetime.now()


class Invoice:
    """Aggregate Root — точка входа для всех операций."""

    def __init__(self, invoice_id: InvoiceId, customer_id: str,
                 amount: Money, due_date: datetime):
        self._id = invoice_id
        self._customer_id = customer_id
        self._amount = amount
        self._due_date = due_date
        self._payments: List[Payment] = []
        self._created_at = datetime.now()

    @property
    def id(self) -> InvoiceId:
        return self._id

    @property
    def amount(self) -> Money:
        return self._amount

    @property
    def is_paid(self) -> bool:
        return self.paid_amount.amount >= self._amount.amount

    @property
    def paid_amount(self) -> Money:
        completed = [p for p in self._payments
                     if p.status == PaymentStatus.COMPLETED]
        if not completed:
            return Money(Decimal("0"), self._amount.currency)
        return sum((p.amount for p in completed[1:]), completed[0].amount)

    @property
    def remaining_amount(self) -> Money:
        return self._amount.subtract(self.paid_amount)

    def add_payment(self, amount: Money) -> Payment:
        """Добавить платёж к счёту."""
        if self.is_paid:
            raise ValueError("Invoice is already paid")
        if amount.currency != self._amount.currency:
            raise ValueError("Payment currency must match invoice currency")
        if amount.amount > self.remaining_amount.amount:
            raise ValueError("Payment exceeds remaining amount")

        payment = Payment(
            id=PaymentId.generate(),
            amount=amount,
            status=PaymentStatus.PENDING
        )
        self._payments.append(payment)
        return payment

    def complete_payment(self, payment_id: PaymentId) -> None:
        """Завершить платёж."""
        payment = self._find_payment(payment_id)
        payment.complete()

    def _find_payment(self, payment_id: PaymentId) -> Payment:
        for payment in self._payments:
            if payment.id == payment_id:
                return payment
        raise ValueError(f"Payment {payment_id} not found")
```

---

## Доменные сервисы

Когда операция не принадлежит ни одной сущности — используем доменный сервис.

```python
from abc import ABC, abstractmethod
from decimal import Decimal


class ExchangeRateProvider(ABC):
    """Интерфейс для получения курсов валют."""

    @abstractmethod
    def get_rate(self, from_currency: str, to_currency: str) -> Decimal:
        pass


class MoneyTransferService:
    """Доменный сервис — операция над несколькими агрегатами."""

    def __init__(self, exchange_rate_provider: ExchangeRateProvider):
        self._exchange_rate_provider = exchange_rate_provider

    def transfer(self, source_account: "Account",
                 target_account: "Account",
                 amount: Money) -> None:
        """Перевод денег между счетами."""

        # Проверка бизнес-правил
        if not source_account.is_active or not target_account.is_active:
            raise ValueError("Both accounts must be active")

        # Конвертация валюты если нужно
        target_amount = amount
        if amount.currency != target_account.currency:
            rate = self._exchange_rate_provider.get_rate(
                amount.currency,
                target_account.currency
            )
            target_amount = Money(
                amount.amount * rate,
                target_account.currency
            )

        # Выполнение операции
        source_account.withdraw(amount)
        target_account.deposit(target_amount)


class PricingService:
    """Доменный сервис для расчёта цен."""

    def calculate_order_price(
        self,
        order: Order,
        customer: "Customer",
        promotions: List["Promotion"]
    ) -> Money:
        """Сложная логика расчёта цены, включающая скидки."""

        base_price = order.total

        # Применяем скидку за лояльность
        loyalty_discount = self._calculate_loyalty_discount(
            base_price, customer
        )

        # Применяем промо-акции
        promo_discount = self._calculate_promo_discount(
            base_price, promotions
        )

        # Максимальная скидка не более 50%
        total_discount = min(
            loyalty_discount.add(promo_discount),
            base_price.multiply(Decimal("0.5"))
        )

        return base_price.subtract(total_discount)

    def _calculate_loyalty_discount(
        self, price: Money, customer: "Customer"
    ) -> Money:
        discount_rate = Decimal("0")
        if customer.loyalty_years >= 5:
            discount_rate = Decimal("0.15")
        elif customer.loyalty_years >= 2:
            discount_rate = Decimal("0.10")
        elif customer.loyalty_years >= 1:
            discount_rate = Decimal("0.05")

        return price.multiply(discount_rate)

    def _calculate_promo_discount(
        self, price: Money, promotions: List["Promotion"]
    ) -> Money:
        # Логика расчёта промо-скидок
        pass
```

---

## Доменные события

События, которые произошли в домене и могут быть интересны другим частям системы.

```python
from dataclasses import dataclass
from datetime import datetime
from typing import List, Callable
from uuid import UUID, uuid4


@dataclass(frozen=True)
class DomainEvent:
    """Базовый класс для доменных событий."""
    event_id: UUID
    occurred_at: datetime

    @classmethod
    def create(cls, **kwargs) -> "DomainEvent":
        return cls(
            event_id=uuid4(),
            occurred_at=datetime.now(),
            **kwargs
        )


@dataclass(frozen=True)
class OrderPlaced(DomainEvent):
    order_id: str
    customer_id: str
    total: Decimal


@dataclass(frozen=True)
class OrderShipped(DomainEvent):
    order_id: str
    tracking_number: str


@dataclass(frozen=True)
class PaymentReceived(DomainEvent):
    invoice_id: str
    payment_id: str
    amount: Decimal


class AggregateRoot:
    """Базовый класс для корней агрегатов с поддержкой событий."""

    def __init__(self):
        self._domain_events: List[DomainEvent] = []

    def add_domain_event(self, event: DomainEvent) -> None:
        self._domain_events.append(event)

    def clear_domain_events(self) -> List[DomainEvent]:
        events = self._domain_events.copy()
        self._domain_events.clear()
        return events

    @property
    def domain_events(self) -> tuple[DomainEvent, ...]:
        return tuple(self._domain_events)


class Order(AggregateRoot):
    """Заказ с поддержкой доменных событий."""

    def __init__(self, order_id: str, customer_id: str):
        super().__init__()
        self._order_id = order_id
        self._customer_id = customer_id
        self._items: List[OrderItem] = []
        self._status = OrderStatus.NEW

    def place(self) -> None:
        """Разместить заказ."""
        if not self._items:
            raise ValueError("Cannot place empty order")

        self._status = OrderStatus.SUBMITTED

        # Публикуем доменное событие
        self.add_domain_event(OrderPlaced.create(
            order_id=self._order_id,
            customer_id=self._customer_id,
            total=self.total
        ))

    def ship(self, tracking_number: str) -> None:
        """Отправить заказ."""
        if self._status != OrderStatus.PAID:
            raise ValueError("Order must be paid before shipping")

        self._status = OrderStatus.SHIPPED

        self.add_domain_event(OrderShipped.create(
            order_id=self._order_id,
            tracking_number=tracking_number
        ))
```

---

## Инварианты и валидация

Бизнес-правила, которые всегда должны соблюдаться.

```python
from dataclasses import dataclass
from datetime import datetime, date
from decimal import Decimal
from typing import List


class Reservation:
    """Бронирование с инвариантами."""

    MAX_GUESTS = 10
    MIN_STAY_DAYS = 1
    MAX_STAY_DAYS = 30

    def __init__(
        self,
        room_id: str,
        guest_name: str,
        check_in: date,
        check_out: date,
        guests_count: int
    ):
        # Валидация инвариантов при создании
        self._validate_dates(check_in, check_out)
        self._validate_guests_count(guests_count)

        self._room_id = room_id
        self._guest_name = guest_name
        self._check_in = check_in
        self._check_out = check_out
        self._guests_count = guests_count
        self._is_cancelled = False

    def _validate_dates(self, check_in: date, check_out: date) -> None:
        """Инвариант: даты должны быть корректными."""
        if check_in >= check_out:
            raise ValueError("Check-out must be after check-in")

        stay_days = (check_out - check_in).days
        if stay_days < self.MIN_STAY_DAYS:
            raise ValueError(f"Minimum stay is {self.MIN_STAY_DAYS} day(s)")
        if stay_days > self.MAX_STAY_DAYS:
            raise ValueError(f"Maximum stay is {self.MAX_STAY_DAYS} days")

        if check_in < date.today():
            raise ValueError("Cannot book in the past")

    def _validate_guests_count(self, guests_count: int) -> None:
        """Инвариант: количество гостей ограничено."""
        if guests_count <= 0:
            raise ValueError("Guests count must be positive")
        if guests_count > self.MAX_GUESTS:
            raise ValueError(f"Maximum guests is {self.MAX_GUESTS}")

    def extend_stay(self, new_check_out: date) -> None:
        """Продлить бронирование."""
        if self._is_cancelled:
            raise ValueError("Cannot extend cancelled reservation")

        self._validate_dates(self._check_in, new_check_out)
        self._check_out = new_check_out

    def cancel(self) -> None:
        """Отменить бронирование."""
        if self._is_cancelled:
            raise ValueError("Reservation is already cancelled")

        # Бизнес-правило: нельзя отменить в день заезда
        if date.today() >= self._check_in:
            raise ValueError("Cannot cancel on or after check-in date")

        self._is_cancelled = True

    @property
    def stay_duration(self) -> int:
        return (self._check_out - self._check_in).days
```

---

## Best Practices

### 1. Keep it focused

```python
# Хорошо: модель отвечает только за свою область
class Product:
    def __init__(self, name: str, price: Money):
        self._name = name
        self._price = price

    def apply_discount(self, percent: Decimal) -> Money:
        """Продукт знает, как применить скидку к себе."""
        return self._price.multiply(1 - percent / 100)
```

### 2. Use Value Objects for concepts

```python
# Вместо примитивов используйте Value Objects
@dataclass(frozen=True)
class Email:
    value: str

    def __post_init__(self):
        if "@" not in self.value:
            raise ValueError("Invalid email")


@dataclass(frozen=True)
class PhoneNumber:
    country_code: str
    number: str

    def format(self) -> str:
        return f"+{self.country_code} {self.number}"
```

### 3. Protect invariants

```python
class BankAccount:
    def __init__(self, account_id: str, initial_balance: Money):
        self._id = account_id
        self._balance = initial_balance

    def withdraw(self, amount: Money) -> None:
        # Инвариант: баланс не может быть отрицательным
        if amount.amount > self._balance.amount:
            raise ValueError("Insufficient funds")
        self._balance = self._balance.subtract(amount)
```

---

## Типичные ошибки

1. **Анемичная модель**
   - Данные в одном месте, логика в сервисах
   - Нарушение инкапсуляции

2. **Протечка инфраструктуры**
   ```python
   # Плохо: доменная модель знает о БД
   class User:
       def save(self):
           db.execute("INSERT INTO users...")

   # Хорошо: инфраструктура отдельно
   class UserRepository:
       def save(self, user: User) -> None:
           # Работа с БД
   ```

3. **Примитивная одержимость**
   ```python
   # Плохо: email как строка
   def create_user(email: str): ...

   # Хорошо: email как Value Object
   def create_user(email: Email): ...
   ```

4. **Публичные сеттеры**
   ```python
   # Плохо: можно изменить статус напрямую
   order.status = OrderStatus.PAID

   # Хорошо: изменение через методы с валидацией
   order.mark_as_paid(payment_id)
   ```
