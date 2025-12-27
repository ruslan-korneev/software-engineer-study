# Domain Language

## Ubiquitous Language (Единый язык)

Ubiquitous Language (Единый язык) — это концепция из Domain-Driven Design (DDD), которая требует использования единой терминологии между разработчиками, бизнес-экспертами и в самом коде. Это мост между бизнесом и технической реализацией.

---

## Что такое Ubiquitous Language

### Основная идея

- **Один язык для всех**: разработчики, аналитики и доменные эксперты говорят одинаково
- **Код отражает бизнес**: имена классов, методов и переменных соответствуют терминам предметной области
- **Документация в коде**: код сам является документацией бизнес-логики

### Преимущества

1. **Улучшение коммуникации** между техническими и нетехническими участниками
2. **Снижение количества ошибок** из-за недопонимания требований
3. **Легче вносить изменения**, так как бизнес-логика явно выражена в коде
4. **Onboarding новых разработчиков** проще — код читается как бизнес-спецификация

---

## Примеры: плохо vs хорошо

### Пример 1: E-commerce

```python
# ПЛОХО: технический язык
class DataProcessor:
    def __init__(self):
        self.items = []
        self.flag = False

    def add(self, item_data: dict) -> None:
        self.items.append(item_data)

    def execute(self) -> dict:
        result = {"total": 0}
        for item in self.items:
            result["total"] += item["price"] * item["qty"]
        self.flag = True
        return result


# ХОРОШО: доменный язык
from decimal import Decimal
from dataclasses import dataclass
from enum import Enum
from typing import List


class OrderStatus(Enum):
    DRAFT = "draft"
    PLACED = "placed"
    CONFIRMED = "confirmed"
    SHIPPED = "shipped"
    DELIVERED = "delivered"


@dataclass(frozen=True)
class ProductId:
    value: str


@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str = "RUB"

    def add(self, other: "Money") -> "Money":
        return Money(self.amount + other.amount, self.currency)


@dataclass
class OrderLine:
    """Строка заказа — товар и его количество."""
    product_id: ProductId
    product_name: str
    unit_price: Money
    quantity: int

    @property
    def line_total(self) -> Money:
        return Money(self.unit_price.amount * self.quantity)


class ShoppingCart:
    """Корзина покупок."""

    def __init__(self, customer_id: str):
        self._customer_id = customer_id
        self._lines: List[OrderLine] = []

    def add_product(
        self,
        product_id: ProductId,
        product_name: str,
        price: Money,
        quantity: int = 1
    ) -> None:
        """Добавить товар в корзину."""
        line = OrderLine(product_id, product_name, price, quantity)
        self._lines.append(line)

    def remove_product(self, product_id: ProductId) -> None:
        """Удалить товар из корзины."""
        self._lines = [
            line for line in self._lines
            if line.product_id != product_id
        ]

    def update_quantity(self, product_id: ProductId, quantity: int) -> None:
        """Изменить количество товара."""
        if quantity <= 0:
            self.remove_product(product_id)
            return

        for line in self._lines:
            if line.product_id == product_id:
                line.quantity = quantity
                return

    @property
    def subtotal(self) -> Money:
        """Сумма товаров без доставки."""
        total = Money(Decimal("0"))
        for line in self._lines:
            total = total.add(line.line_total)
        return total

    def place_order(self) -> "Order":
        """Оформить заказ из корзины."""
        if not self._lines:
            raise ValueError("Cannot place order with empty cart")

        return Order.create_from_cart(self._customer_id, self._lines)


class Order:
    """Заказ — оформленная корзина."""

    def __init__(
        self,
        order_id: str,
        customer_id: str,
        lines: List[OrderLine],
        status: OrderStatus = OrderStatus.PLACED
    ):
        self._order_id = order_id
        self._customer_id = customer_id
        self._lines = lines
        self._status = status

    @classmethod
    def create_from_cart(cls, customer_id: str, lines: List[OrderLine]) -> "Order":
        import uuid
        return cls(
            order_id=str(uuid.uuid4()),
            customer_id=customer_id,
            lines=list(lines)
        )

    def confirm(self) -> None:
        """Подтвердить заказ."""
        if self._status != OrderStatus.PLACED:
            raise ValueError("Only placed orders can be confirmed")
        self._status = OrderStatus.CONFIRMED

    def ship(self, tracking_number: str) -> None:
        """Отправить заказ."""
        if self._status != OrderStatus.CONFIRMED:
            raise ValueError("Only confirmed orders can be shipped")
        self._status = OrderStatus.SHIPPED
        self._tracking_number = tracking_number

    def deliver(self) -> None:
        """Отметить доставку."""
        if self._status != OrderStatus.SHIPPED:
            raise ValueError("Only shipped orders can be delivered")
        self._status = OrderStatus.DELIVERED
```

### Пример 2: Банковская система

```python
# ПЛОХО: generic code
class Account:
    def __init__(self):
        self.value = 0
        self.active = True

    def process(self, amount: float, operation: str) -> bool:
        if operation == "add":
            self.value += amount
        elif operation == "remove":
            if self.value >= amount:
                self.value -= amount
            else:
                return False
        return True


# ХОРОШО: banking domain language
from datetime import datetime
from decimal import Decimal
from enum import Enum
from typing import List, Optional


class TransactionType(Enum):
    DEPOSIT = "deposit"
    WITHDRAWAL = "withdrawal"
    TRANSFER = "transfer"
    FEE = "fee"
    INTEREST = "interest"


@dataclass(frozen=True)
class AccountNumber:
    """Номер банковского счёта."""
    value: str

    def __post_init__(self):
        if len(self.value) != 20:  # Стандартный формат
            raise ValueError("Invalid account number format")


@dataclass
class Transaction:
    """Банковская транзакция."""
    transaction_id: str
    transaction_type: TransactionType
    amount: Money
    timestamp: datetime
    description: str
    reference: Optional[str] = None


class InsufficientFundsError(Exception):
    """Недостаточно средств на счёте."""
    pass


class AccountFrozenError(Exception):
    """Счёт заморожен."""
    pass


class BankAccount:
    """Банковский счёт."""

    MINIMUM_BALANCE = Money(Decimal("0"))
    DAILY_WITHDRAWAL_LIMIT = Money(Decimal("50000"))

    def __init__(
        self,
        account_number: AccountNumber,
        holder_name: str,
        initial_deposit: Money
    ):
        self._account_number = account_number
        self._holder_name = holder_name
        self._balance = initial_deposit
        self._is_frozen = False
        self._transactions: List[Transaction] = []
        self._opened_at = datetime.now()

        self._record_transaction(
            TransactionType.DEPOSIT,
            initial_deposit,
            "Initial deposit"
        )

    @property
    def account_number(self) -> AccountNumber:
        return self._account_number

    @property
    def balance(self) -> Money:
        return self._balance

    @property
    def is_frozen(self) -> bool:
        return self._is_frozen

    def deposit(self, amount: Money, description: str = "Cash deposit") -> None:
        """Внести средства на счёт."""
        self._ensure_not_frozen()

        if amount.amount <= 0:
            raise ValueError("Deposit amount must be positive")

        self._balance = self._balance.add(amount)
        self._record_transaction(TransactionType.DEPOSIT, amount, description)

    def withdraw(self, amount: Money, description: str = "Cash withdrawal") -> None:
        """Снять средства со счёта."""
        self._ensure_not_frozen()
        self._ensure_sufficient_funds(amount)
        self._ensure_within_daily_limit(amount)

        self._balance = self._balance.subtract(amount)
        self._record_transaction(TransactionType.WITHDRAWAL, amount, description)

    def transfer_to(
        self,
        recipient: "BankAccount",
        amount: Money,
        description: str = "Transfer"
    ) -> None:
        """Перевести средства на другой счёт."""
        self._ensure_not_frozen()
        recipient._ensure_not_frozen()
        self._ensure_sufficient_funds(amount)

        self._balance = self._balance.subtract(amount)
        recipient._balance = recipient._balance.add(amount)

        reference = f"TRF-{datetime.now().strftime('%Y%m%d%H%M%S')}"

        self._record_transaction(
            TransactionType.TRANSFER,
            amount,
            f"Transfer to {recipient.account_number.value}",
            reference
        )
        recipient._record_transaction(
            TransactionType.TRANSFER,
            amount,
            f"Transfer from {self.account_number.value}",
            reference
        )

    def freeze(self, reason: str) -> None:
        """Заморозить счёт."""
        self._is_frozen = True
        # В реальности здесь был бы аудит

    def unfreeze(self) -> None:
        """Разморозить счёт."""
        self._is_frozen = False

    def get_statement(
        self,
        from_date: datetime,
        to_date: datetime
    ) -> List[Transaction]:
        """Получить выписку по счёту."""
        return [
            t for t in self._transactions
            if from_date <= t.timestamp <= to_date
        ]

    def _ensure_not_frozen(self) -> None:
        if self._is_frozen:
            raise AccountFrozenError("Account is frozen")

    def _ensure_sufficient_funds(self, amount: Money) -> None:
        if self._balance.amount < amount.amount:
            raise InsufficientFundsError(
                f"Insufficient funds. Balance: {self._balance.amount}, "
                f"Requested: {amount.amount}"
            )

    def _ensure_within_daily_limit(self, amount: Money) -> None:
        today = datetime.now().date()
        today_withdrawals = sum(
            t.amount.amount for t in self._transactions
            if t.timestamp.date() == today
            and t.transaction_type == TransactionType.WITHDRAWAL
        )

        if today_withdrawals + amount.amount > self.DAILY_WITHDRAWAL_LIMIT.amount:
            raise ValueError("Daily withdrawal limit exceeded")

    def _record_transaction(
        self,
        transaction_type: TransactionType,
        amount: Money,
        description: str,
        reference: Optional[str] = None
    ) -> None:
        import uuid
        transaction = Transaction(
            transaction_id=str(uuid.uuid4()),
            transaction_type=transaction_type,
            amount=amount,
            timestamp=datetime.now(),
            description=description,
            reference=reference
        )
        self._transactions.append(transaction)
```

---

## Построение Ubiquitous Language

### 1. Работа с доменными экспертами

```python
# Собираем термины из бесед с экспертами:
# - "Бронирование" -> Reservation
# - "Заселение" -> CheckIn
# - "Выселение" -> CheckOut
# - "Гость" -> Guest
# - "Номер" -> Room
# - "Стоимость за ночь" -> NightlyRate

class HotelReservationSystem:
    """
    Система бронирования отеля.

    Глоссарий:
    - Reservation: бронирование номера на определённые даты
    - Guest: гость, совершающий бронирование
    - Room: номер в отеле
    - CheckIn: процесс заселения гостя
    - CheckOut: процесс выселения гостя
    - NightlyRate: стоимость номера за одну ночь
    """
    pass


class Reservation:
    """
    Бронирование номера.

    Бизнес-правила:
    - Бронирование создаётся в статусе PENDING
    - Бронирование подтверждается после оплаты депозита
    - Гость может заселиться не раньше 14:00 дня заезда
    - Гость должен выселиться до 12:00 дня выезда
    """

    CHECK_IN_TIME = "14:00"
    CHECK_OUT_TIME = "12:00"

    def __init__(
        self,
        reservation_number: str,
        guest: "Guest",
        room: "Room",
        check_in_date: date,
        check_out_date: date
    ):
        self._reservation_number = reservation_number
        self._guest = guest
        self._room = room
        self._check_in_date = check_in_date
        self._check_out_date = check_out_date
        self._status = ReservationStatus.PENDING

    def confirm_with_deposit(self, deposit: Money) -> None:
        """Подтвердить бронирование с депозитом."""
        pass

    def check_in_guest(self) -> None:
        """Заселить гостя."""
        pass

    def check_out_guest(self) -> None:
        """Выселить гостя."""
        pass
```

### 2. Создание глоссария

```python
"""
ГЛОССАРИЙ ПРЕДМЕТНОЙ ОБЛАСТИ: СТРАХОВАЯ КОМПАНИЯ

ПОЛИС (Policy)
    Договор страхования между страховщиком и страхователем.
    Определяет условия, покрытие и стоимость.

СТРАХОВОЙ СЛУЧАЙ (Claim)
    Событие, при котором страхователь может получить выплату.

СТРАХОВАЯ ПРЕМИЯ (Premium)
    Сумма, которую страхователь платит за полис.

СТРАХОВОЕ ПОКРЫТИЕ (Coverage)
    Перечень рисков, застрахованных по полису.

ФРАНШИЗА (Deductible)
    Сумма, которую страхователь оплачивает сам при страховом случае.

СТРАХОВАТЕЛЬ (Policyholder)
    Лицо или организация, заключившая договор страхования.

ВЫГОДОПРИОБРЕТАТЕЛЬ (Beneficiary)
    Лицо, которое получит выплату при наступлении страхового случая.
"""


class InsurancePolicy:
    """Страховой полис."""

    def __init__(
        self,
        policy_number: str,
        policyholder: "Policyholder",
        coverage: "Coverage",
        premium: Money,
        deductible: Money,
        effective_date: date,
        expiration_date: date
    ):
        self._policy_number = policy_number
        self._policyholder = policyholder
        self._coverage = coverage
        self._premium = premium
        self._deductible = deductible
        self._effective_date = effective_date
        self._expiration_date = expiration_date

    def file_claim(self, incident_date: date, description: str) -> "Claim":
        """Заявить о страховом случае."""
        if not self.is_active_on(incident_date):
            raise ValueError("Policy was not active on incident date")
        return Claim(policy=self, incident_date=incident_date, description=description)

    def is_active_on(self, check_date: date) -> bool:
        """Проверить, активен ли полис на дату."""
        return self._effective_date <= check_date <= self._expiration_date

    def calculate_payout(self, claim_amount: Money) -> Money:
        """Рассчитать выплату с учётом франшизы."""
        if claim_amount.amount <= self._deductible.amount:
            return Money(Decimal("0"), claim_amount.currency)
        return claim_amount.subtract(self._deductible)
```

---

## Правила именования

### 1. Используйте термины домена

```python
# Плохо
class DataManager:
    def process_data(self, data): ...

# Хорошо
class OrderProcessor:
    def process_order(self, order: Order) -> OrderConfirmation: ...
```

### 2. Избегайте технических терминов в доменном слое

```python
# Плохо
class CustomerDAO:
    def insert_customer_record(self, customer_dto): ...

# Хорошо
class CustomerRepository:
    def save(self, customer: Customer) -> None: ...
```

### 3. Методы должны отражать бизнес-операции

```python
# Плохо
class Order:
    def set_status(self, status): ...
    def update_data(self, data): ...

# Хорошо
class Order:
    def place(self) -> None: ...
    def cancel(self, reason: str) -> None: ...
    def ship(self, tracking_number: str) -> None: ...
    def deliver(self) -> None: ...
```

### 4. Исключения с бизнес-смыслом

```python
# Плохо
raise Exception("Error processing order")
raise ValueError("Invalid data")

# Хорошо
class OrderAlreadyShippedException(Exception):
    """Заказ уже отправлен и не может быть изменён."""
    pass

class InsufficientInventoryException(Exception):
    """Недостаточно товара на складе."""
    pass

class PaymentDeclinedException(Exception):
    """Платёж отклонён платёжной системой."""
    pass
```

---

## Bounded Contexts и язык

Разные контексты могут иметь разные значения для одних и тех же терминов.

```python
# Контекст ПРОДАЖИ
# "Продукт" — то, что покупает клиент
class Product:
    """Продукт в каталоге магазина."""
    def __init__(self, sku: str, name: str, price: Money):
        self.sku = sku
        self.name = name
        self.price = price


# Контекст СКЛАД
# "Продукт" — единица хранения на складе
class InventoryItem:
    """Товар на складе."""
    def __init__(self, sku: str, location: str, quantity: int):
        self.sku = sku
        self.location = location  # Место на складе
        self.quantity = quantity
        self.reorder_level = 10   # Минимум для заказа

    def is_low_stock(self) -> bool:
        return self.quantity <= self.reorder_level


# Контекст ДОСТАВКА
# "Продукт" — посылка для отправки
class Shipment:
    """Отправление для доставки."""
    def __init__(self, tracking_number: str, items: List[str], weight: float):
        self.tracking_number = tracking_number
        self.items = items  # SKU товаров
        self.weight = weight
        self.status = ShipmentStatus.PENDING
```

---

## Best Practices

1. **Документируйте глоссарий** в коде и отдельно
2. **Проводите event storming** с доменными экспертами
3. **Рефакторьте при изменении понимания** — язык эволюционирует
4. **Используйте единый язык везде**: код, документация, коммуникация
5. **Избегайте синонимов** — один термин = одно понятие

---

## Типичные ошибки

1. **Технический язык в домене**
   ```python
   # Плохо
   class UserDTO: ...
   class OrderFactory: ...
   class PaymentStrategy: ...

   # Хорошо (если это доменные классы)
   class Customer: ...
   class Order: ...
   class PaymentMethod: ...
   ```

2. **Generic имена**
   ```python
   # Плохо
   class Manager: ...
   class Handler: ...
   class Processor: ...

   # Хорошо
   class LoanOfficer: ...
   class ClaimAdjuster: ...
   class PaymentProcessor: ...
   ```

3. **Несогласованность терминов**
   ```python
   # Плохо: разные имена для одного понятия
   class Customer: ...  # в одном модуле
   class Client: ...    # в другом модуле
   class User: ...      # в третьем модуле

   # Хорошо: единый термин
   class Customer: ...  # везде
   ```

4. **Игнорирование контекста**
   ```python
   # Плохо: один класс для всех контекстов
   class Product:
       # Поля для продаж
       price: Money
       # Поля для склада
       location: str
       # Поля для доставки
       weight: float

   # Хорошо: отдельные модели для контекстов
   # sales/product.py
   class Product: ...
   # warehouse/inventory_item.py
   class InventoryItem: ...
   # shipping/package.py
   class Package: ...
   ```
