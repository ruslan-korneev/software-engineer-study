# Entities

## Что такое Entity?

**Entity (Сущность)** — это объект в доменной модели, который обладает уникальной идентичностью и жизненным циклом. В отличие от Value Object, две Entity с одинаковыми атрибутами — это разные объекты, если у них разные идентификаторы.

> Концепция описана Эриком Эвансом в книге "Domain-Driven Design" и является одним из ключевых строительных блоков доменной модели.

## Ключевые характеристики

1. **Уникальная идентичность** — определяется ID, а не атрибутами
2. **Жизненный цикл** — создание, изменение, удаление
3. **Изменяемость** — состояние может меняться со временем
4. **Непрерывность** — сохраняет идентичность при изменениях
5. **Бизнес-поведение** — содержит бизнес-логику

## Сравнение: Entity vs Value Object

```python
from dataclasses import dataclass
from typing import Optional


# Entity — определяется идентичностью
class User:
    def __init__(self, id: int, name: str, email: str):
        self.id = id
        self.name = name
        self.email = email

    def __eq__(self, other):
        if not isinstance(other, User):
            return False
        return self.id == other.id  # Сравнение по ID!

    def __hash__(self):
        return hash(self.id)


# Два пользователя с одинаковыми данными, но разными ID — разные!
user1 = User(1, "Alice", "alice@example.com")
user2 = User(2, "Alice", "alice@example.com")
print(user1 == user2)  # False — разные ID

# Один пользователь, изменённые данные — тот же!
user1.name = "Alicia"
user3 = User(1, "Alicia", "alice@example.com")
print(user1 == user3)  # True — одинаковый ID


# Value Object — определяется значением
@dataclass(frozen=True)
class Email:
    value: str

email1 = Email("alice@example.com")
email2 = Email("alice@example.com")
print(email1 == email2)  # True — одинаковые значения
```

## Базовая реализация Entity

### Абстрактный базовый класс

```python
from abc import ABC
from typing import TypeVar, Generic, Optional
from uuid import UUID, uuid4


T = TypeVar("T")


class Entity(ABC, Generic[T]):
    """Базовый класс для всех сущностей."""

    def __init__(self, id: Optional[T] = None):
        self._id = id

    @property
    def id(self) -> Optional[T]:
        return self._id

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Entity):
            return False
        if self._id is None or other._id is None:
            return False
        return self._id == other._id

    def __hash__(self) -> int:
        if self._id is None:
            return id(self)  # Используем id объекта для новых сущностей
        return hash(self._id)

    def __repr__(self) -> str:
        return f"{self.__class__.__name__}(id={self._id})"


class EntityWithIntId(Entity[int]):
    """Сущность с целочисленным ID."""
    pass


class EntityWithUUID(Entity[UUID]):
    """Сущность с UUID."""

    def __init__(self, id: Optional[UUID] = None):
        super().__init__(id or uuid4())
```

### Пример: User Entity

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional, List
from uuid import UUID, uuid4


class User(EntityWithUUID):
    """Сущность пользователя."""

    def __init__(
        self,
        id: Optional[UUID] = None,
        username: str = "",
        email: str = "",
        password_hash: str = "",
        is_active: bool = True
    ):
        super().__init__(id)
        self._username = username
        self._email = email
        self._password_hash = password_hash
        self._is_active = is_active
        self._created_at = datetime.now()
        self._updated_at = datetime.now()

    # Properties для инкапсуляции
    @property
    def username(self) -> str:
        return self._username

    @property
    def email(self) -> str:
        return self._email

    @property
    def is_active(self) -> bool:
        return self._is_active

    @property
    def created_at(self) -> datetime:
        return self._created_at

    # Бизнес-методы
    def change_email(self, new_email: str) -> None:
        """Изменение email с валидацией."""
        if "@" not in new_email:
            raise ValueError("Invalid email format")
        self._email = new_email
        self._updated_at = datetime.now()

    def change_username(self, new_username: str) -> None:
        """Изменение имени пользователя."""
        if len(new_username) < 3:
            raise ValueError("Username must be at least 3 characters")
        self._username = new_username
        self._updated_at = datetime.now()

    def deactivate(self) -> None:
        """Деактивация пользователя."""
        if not self._is_active:
            raise ValueError("User is already inactive")
        self._is_active = False
        self._updated_at = datetime.now()

    def activate(self) -> None:
        """Активация пользователя."""
        if self._is_active:
            raise ValueError("User is already active")
        self._is_active = True
        self._updated_at = datetime.now()

    def verify_password(self, password: str) -> bool:
        """Проверка пароля."""
        import hashlib
        hash_attempt = hashlib.sha256(password.encode()).hexdigest()
        return hash_attempt == self._password_hash

    def change_password(self, old_password: str, new_password: str) -> None:
        """Изменение пароля."""
        if not self.verify_password(old_password):
            raise ValueError("Current password is incorrect")
        if len(new_password) < 8:
            raise ValueError("Password must be at least 8 characters")
        import hashlib
        self._password_hash = hashlib.sha256(new_password.encode()).hexdigest()
        self._updated_at = datetime.now()


# Использование
user = User(
    username="alice",
    email="alice@example.com",
    password_hash="hashed_password"
)

user.change_email("alice.new@example.com")
user.deactivate()
```

## Entity с Value Objects

```python
from dataclasses import dataclass
from typing import Optional, List
from uuid import UUID, uuid4
from decimal import Decimal


# Value Objects
@dataclass(frozen=True)
class Email:
    value: str

    def __post_init__(self):
        if "@" not in self.value:
            raise ValueError(f"Invalid email: {self.value}")


@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str

    def __add__(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError("Cannot add different currencies")
        return Money(self.amount + other.amount, self.currency)


@dataclass(frozen=True)
class Address:
    street: str
    city: str
    country: str
    postal_code: str


# Entity с Value Objects
class Customer(EntityWithUUID):
    """Сущность клиента с Value Objects."""

    def __init__(
        self,
        id: Optional[UUID] = None,
        name: str = "",
        email: Email = None,
        billing_address: Address = None,
        shipping_address: Address = None
    ):
        super().__init__(id)
        self._name = name
        self._email = email
        self._billing_address = billing_address
        self._shipping_address = shipping_address or billing_address

    @property
    def name(self) -> str:
        return self._name

    @property
    def email(self) -> Email:
        return self._email

    @property
    def billing_address(self) -> Address:
        return self._billing_address

    @property
    def shipping_address(self) -> Address:
        return self._shipping_address

    def change_email(self, new_email: Email) -> None:
        """Изменение email (принимает Value Object)."""
        self._email = new_email

    def update_billing_address(self, address: Address) -> None:
        """Обновление платёжного адреса."""
        self._billing_address = address

    def update_shipping_address(self, address: Address) -> None:
        """Обновление адреса доставки."""
        self._shipping_address = address

    def use_billing_as_shipping(self) -> None:
        """Использовать платёжный адрес для доставки."""
        self._shipping_address = self._billing_address


# Использование
customer = Customer(
    name="Alice Smith",
    email=Email("alice@example.com"),
    billing_address=Address(
        street="123 Main St",
        city="New York",
        country="USA",
        postal_code="10001"
    )
)

customer.use_billing_as_shipping()
customer.change_email(Email("alice.new@example.com"))
```

## Aggregate Root

Aggregate Root — это Entity, которая является корнем агрегата и контролирует доступ к вложенным сущностям:

```python
from dataclasses import dataclass, field
from typing import List, Optional
from uuid import UUID, uuid4
from datetime import datetime
from decimal import Decimal


@dataclass(frozen=True)
class OrderItem:
    """Value Object для позиции заказа."""
    product_id: UUID
    product_name: str
    quantity: int
    unit_price: Decimal

    @property
    def total(self) -> Decimal:
        return self.unit_price * self.quantity


class Order(EntityWithUUID):
    """Aggregate Root для заказа."""

    def __init__(
        self,
        id: Optional[UUID] = None,
        customer_id: UUID = None,
        status: str = "draft"
    ):
        super().__init__(id)
        self._customer_id = customer_id
        self._status = status
        self._items: List[OrderItem] = []
        self._created_at = datetime.now()
        self._submitted_at: Optional[datetime] = None

    @property
    def customer_id(self) -> UUID:
        return self._customer_id

    @property
    def status(self) -> str:
        return self._status

    @property
    def items(self) -> List[OrderItem]:
        return self._items.copy()  # Возвращаем копию для защиты

    @property
    def total(self) -> Decimal:
        return sum(item.total for item in self._items)

    @property
    def item_count(self) -> int:
        return len(self._items)

    # Бизнес-методы через Aggregate Root
    def add_item(
        self,
        product_id: UUID,
        product_name: str,
        quantity: int,
        unit_price: Decimal
    ) -> None:
        """Добавление позиции в заказ."""
        if self._status != "draft":
            raise ValueError("Cannot modify submitted order")

        if quantity <= 0:
            raise ValueError("Quantity must be positive")

        # Проверяем, есть ли уже такой продукт
        for i, item in enumerate(self._items):
            if item.product_id == product_id:
                # Создаём новый item с увеличенным количеством
                self._items[i] = OrderItem(
                    product_id=product_id,
                    product_name=product_name,
                    quantity=item.quantity + quantity,
                    unit_price=unit_price
                )
                return

        # Добавляем новый item
        self._items.append(OrderItem(
            product_id=product_id,
            product_name=product_name,
            quantity=quantity,
            unit_price=unit_price
        ))

    def remove_item(self, product_id: UUID) -> None:
        """Удаление позиции из заказа."""
        if self._status != "draft":
            raise ValueError("Cannot modify submitted order")

        self._items = [
            item for item in self._items
            if item.product_id != product_id
        ]

    def submit(self) -> None:
        """Отправка заказа."""
        if self._status != "draft":
            raise ValueError("Order is already submitted")

        if not self._items:
            raise ValueError("Cannot submit empty order")

        self._status = "submitted"
        self._submitted_at = datetime.now()

    def cancel(self) -> None:
        """Отмена заказа."""
        if self._status == "cancelled":
            raise ValueError("Order is already cancelled")

        if self._status == "shipped":
            raise ValueError("Cannot cancel shipped order")

        self._status = "cancelled"

    def ship(self) -> None:
        """Отправка заказа."""
        if self._status != "submitted":
            raise ValueError("Only submitted orders can be shipped")

        self._status = "shipped"


# Использование
order = Order(customer_id=uuid4())
order.add_item(
    product_id=uuid4(),
    product_name="Laptop",
    quantity=1,
    unit_price=Decimal("999.99")
)
order.add_item(
    product_id=uuid4(),
    product_name="Mouse",
    quantity=2,
    unit_price=Decimal("29.99")
)

print(f"Order total: ${order.total}")  # $1059.97
order.submit()
order.ship()
```

## Доменные события в Entity

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import List, Optional
from uuid import UUID, uuid4
from abc import ABC


@dataclass
class DomainEvent(ABC):
    """Базовое доменное событие."""
    occurred_at: datetime = field(default_factory=datetime.now)
    entity_id: UUID = None


@dataclass
class UserRegistered(DomainEvent):
    """Событие регистрации пользователя."""
    username: str = ""
    email: str = ""


@dataclass
class UserEmailChanged(DomainEvent):
    """Событие изменения email."""
    old_email: str = ""
    new_email: str = ""


@dataclass
class UserDeactivated(DomainEvent):
    """Событие деактивации пользователя."""
    reason: str = ""


class EventSourcedEntity(EntityWithUUID):
    """Сущность с поддержкой доменных событий."""

    def __init__(self, id: Optional[UUID] = None):
        super().__init__(id)
        self._domain_events: List[DomainEvent] = []

    @property
    def domain_events(self) -> List[DomainEvent]:
        return self._domain_events.copy()

    def add_domain_event(self, event: DomainEvent) -> None:
        event.entity_id = self.id
        self._domain_events.append(event)

    def clear_domain_events(self) -> List[DomainEvent]:
        events = self._domain_events.copy()
        self._domain_events.clear()
        return events


class User(EventSourcedEntity):
    """Сущность пользователя с доменными событиями."""

    def __init__(
        self,
        id: Optional[UUID] = None,
        username: str = "",
        email: str = "",
        is_active: bool = True
    ):
        super().__init__(id)
        self._username = username
        self._email = email
        self._is_active = is_active

    @classmethod
    def register(cls, username: str, email: str) -> "User":
        """Фабричный метод для регистрации."""
        user = cls(
            username=username,
            email=email,
            is_active=True
        )
        user.add_domain_event(UserRegistered(
            username=username,
            email=email
        ))
        return user

    def change_email(self, new_email: str) -> None:
        old_email = self._email
        self._email = new_email
        self.add_domain_event(UserEmailChanged(
            old_email=old_email,
            new_email=new_email
        ))

    def deactivate(self, reason: str = "") -> None:
        self._is_active = False
        self.add_domain_event(UserDeactivated(reason=reason))


# Использование
user = User.register("alice", "alice@example.com")
user.change_email("alice.new@example.com")
user.deactivate("User requested")

# Получаем и обрабатываем события
events = user.clear_domain_events()
for event in events:
    print(f"Event: {event}")
    # Отправляем в message bus, сохраняем в event store и т.д.
```

## Entity в SQLAlchemy

```python
from sqlalchemy import Column, Integer, String, Boolean, DateTime, ForeignKey
from sqlalchemy.orm import declarative_base, relationship
from sqlalchemy.dialects.postgresql import UUID as PGUUID
from datetime import datetime
from uuid import uuid4

Base = declarative_base()


class UserEntity(Base):
    """ORM модель для User Entity."""
    __tablename__ = "users"

    id = Column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)
    username = Column(String(50), unique=True, nullable=False)
    email = Column(String(100), unique=True, nullable=False)
    password_hash = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.now)
    updated_at = Column(DateTime, default=datetime.now, onupdate=datetime.now)

    # Связи
    orders = relationship("OrderEntity", back_populates="customer")

    def __eq__(self, other):
        if not isinstance(other, UserEntity):
            return False
        return self.id == other.id

    def __hash__(self):
        return hash(self.id)


class OrderEntity(Base):
    """ORM модель для Order Entity (Aggregate Root)."""
    __tablename__ = "orders"

    id = Column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)
    customer_id = Column(PGUUID(as_uuid=True), ForeignKey("users.id"))
    status = Column(String(20), default="draft")
    created_at = Column(DateTime, default=datetime.now)
    submitted_at = Column(DateTime, nullable=True)

    customer = relationship("UserEntity", back_populates="orders")
    items = relationship("OrderItemEntity", back_populates="order", cascade="all, delete-orphan")

    def __eq__(self, other):
        if not isinstance(other, OrderEntity):
            return False
        return self.id == other.id

    def __hash__(self):
        return hash(self.id)
```

## Сравнение подходов

| Аспект | Entity | Value Object |
|--------|--------|--------------|
| **Идентичность** | По ID | По значению |
| **Изменяемость** | Изменяемый | Неизменяемый |
| **Жизненный цикл** | Есть | Нет |
| **Персистентность** | Отдельная таблица | Часть таблицы родителя |
| **Пример** | User, Order | Email, Address, Money |

## Плюсы и минусы

### Преимущества

1. **Уникальность** — каждый объект уникален и отслеживаем
2. **Бизнес-логика** — инкапсулирует поведение
3. **Жизненный цикл** — отражает реальные объекты домена
4. **Инварианты** — защищает свои правила

### Недостатки

1. **Сложность** — требует правильной реализации equals/hashCode
2. **Изменяемость** — сложнее тестировать и отслеживать
3. **Персистентность** — требует маппинга в БД
4. **Идентификаторы** — нужна стратегия генерации ID

## Когда использовать?

### Используйте Entity для:
- Объектов с уникальной идентичностью (User, Order, Product)
- Объектов с жизненным циклом
- Объектов, которые нужно отслеживать во времени
- Aggregate Roots

### Не используйте для:
- Простых значений (используйте Value Objects)
- Объектов без идентичности
- Неизменяемых концепций

## Лучшие практики

1. **Правильное equals/hashCode** — по ID, не по атрибутам
2. **Инкапсуляция** — скрывайте состояние за методами
3. **Валидация** — проверяйте инварианты в методах
4. **Доменные события** — уведомляйте о важных изменениях
5. **Aggregate Root** — контролируйте доступ к вложенным сущностям

## Заключение

Entity — это центральный строительный блок доменной модели. Правильное моделирование сущностей с их бизнес-логикой, инвариантами и жизненным циклом — ключ к созданию богатой доменной модели в стиле Domain-Driven Design. Сочетание Entity и Value Objects позволяет создавать выразительный и безопасный код.
