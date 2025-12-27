# Value Objects

## Что такое Value Object?

**Value Object (Объект-значение)** — это объект, который определяется своими атрибутами, а не уникальной идентичностью. Два Value Object с одинаковыми значениями считаются равными и взаимозаменяемыми. Value Objects неизменяемы (immutable).

> Концепция описана Эриком Эвансом в книге "Domain-Driven Design" и является фундаментальным строительным блоком доменной модели.

## Ключевые характеристики

1. **Определяется значением** — равенство по атрибутам, а не по ID
2. **Неизменяемость** — после создания не может быть изменён
3. **Заменяемость** — можно заменить другим с теми же значениями
4. **Самовалидация** — проверяет свои инварианты при создании
5. **Без побочных эффектов** — методы возвращают новые объекты

## Сравнение: Value Object vs Entity

```python
# Entity — определяется идентичностью
class User:
    def __init__(self, id: int, name: str):
        self.id = id
        self.name = name

user1 = User(1, "Alice")
user2 = User(1, "Alice")
print(user1 == user2)  # False (разные объекты!)
# Но логически это один и тот же пользователь (id=1)


# Value Object — определяется значением
from dataclasses import dataclass

@dataclass(frozen=True)
class Money:
    amount: int
    currency: str

money1 = Money(100, "USD")
money2 = Money(100, "USD")
print(money1 == money2)  # True (одинаковые значения!)
# $100 = $100, они взаимозаменяемы
```

## Базовая реализация

### С использованием dataclass (frozen)

```python
from dataclasses import dataclass
from typing import Optional
import re


@dataclass(frozen=True)
class Email:
    """Value Object для email адреса."""
    value: str

    def __post_init__(self):
        if not self._is_valid(self.value):
            raise ValueError(f"Invalid email: {self.value}")

    @staticmethod
    def _is_valid(email: str) -> bool:
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return bool(re.match(pattern, email))

    @property
    def domain(self) -> str:
        return self.value.split('@')[1]

    @property
    def local_part(self) -> str:
        return self.value.split('@')[0]


# Использование
email1 = Email("user@example.com")
email2 = Email("user@example.com")

print(email1 == email2)  # True
print(email1.domain)     # "example.com"

# Попытка изменить вызовет ошибку
# email1.value = "other@example.com"  # FrozenInstanceError
```

### Money Value Object

```python
from dataclasses import dataclass
from decimal import Decimal
from typing import Union


@dataclass(frozen=True)
class Money:
    """Value Object для денежных сумм."""
    amount: Decimal
    currency: str

    def __post_init__(self):
        # Конвертируем в Decimal если нужно
        if not isinstance(self.amount, Decimal):
            object.__setattr__(self, 'amount', Decimal(str(self.amount)))

        # Валидация
        if self.amount < 0:
            raise ValueError("Amount cannot be negative")

        if len(self.currency) != 3:
            raise ValueError("Currency must be 3-letter ISO code")

    def __add__(self, other: "Money") -> "Money":
        self._check_same_currency(other)
        return Money(self.amount + other.amount, self.currency)

    def __sub__(self, other: "Money") -> "Money":
        self._check_same_currency(other)
        result = self.amount - other.amount
        if result < 0:
            raise ValueError("Result cannot be negative")
        return Money(result, self.currency)

    def __mul__(self, multiplier: Union[int, float, Decimal]) -> "Money":
        return Money(self.amount * Decimal(str(multiplier)), self.currency)

    def __truediv__(self, divisor: Union[int, float, Decimal]) -> "Money":
        return Money(self.amount / Decimal(str(divisor)), self.currency)

    def _check_same_currency(self, other: "Money") -> None:
        if self.currency != other.currency:
            raise ValueError(
                f"Cannot operate on different currencies: "
                f"{self.currency} and {other.currency}"
            )

    def allocate(self, ratios: list[int]) -> list["Money"]:
        """Распределение суммы по пропорциям без потери центов."""
        total = sum(ratios)
        remainder = self.amount
        results = []

        for i, ratio in enumerate(ratios):
            share = self.amount * ratio // total
            results.append(Money(share, self.currency))
            remainder -= share

        # Распределяем остаток
        for i in range(int(remainder * 100)):
            idx = i % len(results)
            results[idx] = Money(
                results[idx].amount + Decimal("0.01"),
                self.currency
            )

        return results

    def __str__(self) -> str:
        return f"{self.amount:.2f} {self.currency}"


# Использование
price = Money(100, "USD")
tax = Money(8, "USD")
total = price + tax
print(total)  # 108.00 USD

# Скидка 10%
discounted = total * Decimal("0.9")
print(discounted)  # 97.20 USD

# Разделение счёта на троих
shares = Money(100, "USD").allocate([1, 1, 1])
for share in shares:
    print(share)  # 33.34, 33.33, 33.33
```

### Address Value Object

```python
from dataclasses import dataclass
from typing import Optional


@dataclass(frozen=True)
class Address:
    """Value Object для адреса."""
    street: str
    city: str
    country: str
    postal_code: str
    apartment: Optional[str] = None

    def __post_init__(self):
        if not self.street:
            raise ValueError("Street is required")
        if not self.city:
            raise ValueError("City is required")
        if not self.country:
            raise ValueError("Country is required")
        if not self.postal_code:
            raise ValueError("Postal code is required")

    def with_apartment(self, apartment: str) -> "Address":
        """Создаёт новый адрес с номером квартиры."""
        return Address(
            street=self.street,
            city=self.city,
            country=self.country,
            postal_code=self.postal_code,
            apartment=apartment
        )

    def format_single_line(self) -> str:
        """Форматирование в одну строку."""
        parts = [self.street]
        if self.apartment:
            parts.append(f"apt. {self.apartment}")
        parts.extend([self.city, self.postal_code, self.country])
        return ", ".join(parts)

    def format_multiline(self) -> str:
        """Форматирование в несколько строк."""
        lines = [self.street]
        if self.apartment:
            lines.append(f"Apartment {self.apartment}")
        lines.append(f"{self.city}, {self.postal_code}")
        lines.append(self.country)
        return "\n".join(lines)


# Использование
address = Address(
    street="123 Main St",
    city="New York",
    country="USA",
    postal_code="10001"
)

# Создание нового адреса (неизменяемость!)
address_with_apt = address.with_apartment("42")

print(address.format_single_line())
# 123 Main St, New York, 10001, USA

print(address_with_apt.format_multiline())
# 123 Main St
# Apartment 42
# New York, 10001
# USA
```

## DateRange Value Object

```python
from dataclasses import dataclass
from datetime import date, timedelta
from typing import Iterator


@dataclass(frozen=True)
class DateRange:
    """Value Object для диапазона дат."""
    start: date
    end: date

    def __post_init__(self):
        if self.start > self.end:
            raise ValueError("Start date must be before or equal to end date")

    @property
    def days(self) -> int:
        """Количество дней в диапазоне."""
        return (self.end - self.start).days + 1

    def contains(self, d: date) -> bool:
        """Проверяет, входит ли дата в диапазон."""
        return self.start <= d <= self.end

    def overlaps(self, other: "DateRange") -> bool:
        """Проверяет пересечение с другим диапазоном."""
        return self.start <= other.end and other.start <= self.end

    def intersection(self, other: "DateRange") -> "DateRange":
        """Возвращает пересечение диапазонов."""
        if not self.overlaps(other):
            raise ValueError("Ranges do not overlap")
        return DateRange(
            start=max(self.start, other.start),
            end=min(self.end, other.end)
        )

    def extend_by(self, days: int) -> "DateRange":
        """Расширяет диапазон на указанное количество дней."""
        return DateRange(
            start=self.start,
            end=self.end + timedelta(days=days)
        )

    def __iter__(self) -> Iterator[date]:
        """Итерация по датам диапазона."""
        current = self.start
        while current <= self.end:
            yield current
            current += timedelta(days=1)

    def __contains__(self, item: date) -> bool:
        return self.contains(item)


# Использование
vacation = DateRange(date(2024, 7, 1), date(2024, 7, 14))
conference = DateRange(date(2024, 7, 10), date(2024, 7, 12))

print(f"Vacation days: {vacation.days}")  # 14
print(f"Overlap: {vacation.overlaps(conference)}")  # True
print(date(2024, 7, 5) in vacation)  # True

# Итерация
for day in conference:
    print(day)
```

## Составные Value Objects

```python
from dataclasses import dataclass
from decimal import Decimal


@dataclass(frozen=True)
class Currency:
    """Value Object для валюты."""
    code: str
    symbol: str
    decimal_places: int = 2

    def __post_init__(self):
        if len(self.code) != 3:
            raise ValueError("Currency code must be 3 letters")


# Предопределённые валюты
class Currencies:
    USD = Currency("USD", "$", 2)
    EUR = Currency("EUR", "u", 2)
    RUB = Currency("RUB", "r", 2)
    BTC = Currency("BTC", "B", 8)


@dataclass(frozen=True)
class Price:
    """Value Object для цены с валютой."""
    amount: Decimal
    currency: Currency

    def __post_init__(self):
        if not isinstance(self.amount, Decimal):
            object.__setattr__(self, 'amount', Decimal(str(self.amount)))

    def format(self) -> str:
        formatted = f"{self.amount:.{self.currency.decimal_places}f}"
        return f"{self.currency.symbol}{formatted}"

    def __str__(self) -> str:
        return self.format()


@dataclass(frozen=True)
class Product:
    """Сущность с Value Objects."""
    id: int
    name: str
    price: Price
    weight: "Weight"


@dataclass(frozen=True)
class Weight:
    """Value Object для веса."""
    value: Decimal
    unit: str = "kg"

    def to_grams(self) -> "Weight":
        if self.unit == "kg":
            return Weight(self.value * 1000, "g")
        return self

    def to_kg(self) -> "Weight":
        if self.unit == "g":
            return Weight(self.value / 1000, "kg")
        return self


# Использование
price = Price(Decimal("29.99"), Currencies.USD)
weight = Weight(Decimal("0.5"), "kg")

product = Product(
    id=1,
    name="Coffee Beans",
    price=price,
    weight=weight
)

print(product.price)  # $29.99
print(product.weight.to_grams())  # Weight(value=500, unit='g')
```

## Value Objects в SQLAlchemy

```python
from sqlalchemy import Column, Integer, String, Numeric, TypeDecorator
from sqlalchemy.orm import composite, declarative_base
from dataclasses import dataclass
from decimal import Decimal

Base = declarative_base()


@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str

    def __composite_values__(self):
        return self.amount, self.currency


@dataclass(frozen=True)
class Address:
    street: str
    city: str
    postal_code: str

    def __composite_values__(self):
        return self.street, self.city, self.postal_code


class Order(Base):
    __tablename__ = "orders"

    id = Column(Integer, primary_key=True)
    customer_name = Column(String(100))

    # Money как composite
    total_amount = Column(Numeric(10, 2))
    total_currency = Column(String(3))
    total = composite(Money, total_amount, total_currency)

    # Address как composite
    shipping_street = Column(String(200))
    shipping_city = Column(String(100))
    shipping_postal = Column(String(20))
    shipping_address = composite(
        Address,
        shipping_street,
        shipping_city,
        shipping_postal
    )


# Использование
order = Order(
    customer_name="Alice",
    total=Money(Decimal("99.99"), "USD"),
    shipping_address=Address("123 Main St", "NYC", "10001")
)

# SQLAlchemy автоматически сохраняет компоненты в отдельные колонки
# и восстанавливает Value Objects при загрузке
```

## Паттерны создания Value Objects

### Factory Method

```python
from dataclasses import dataclass
from decimal import Decimal
from typing import Optional


@dataclass(frozen=True)
class PhoneNumber:
    country_code: str
    number: str

    @classmethod
    def from_string(cls, phone: str) -> "PhoneNumber":
        """Парсинг телефона из строки."""
        # Упрощённый парсинг
        phone = phone.replace(" ", "").replace("-", "")
        if phone.startswith("+"):
            country_code = phone[1:2]
            number = phone[2:]
        else:
            country_code = "1"  # Default US
            number = phone
        return cls(country_code, number)

    @classmethod
    def us(cls, number: str) -> "PhoneNumber":
        """Создание американского номера."""
        return cls("1", number)

    @classmethod
    def ru(cls, number: str) -> "PhoneNumber":
        """Создание российского номера."""
        return cls("7", number)


phone1 = PhoneNumber.from_string("+1 555-1234")
phone2 = PhoneNumber.us("5551234")
phone3 = PhoneNumber.ru("9161234567")
```

### Builder для сложных Value Objects

```python
from dataclasses import dataclass
from typing import Optional, List


@dataclass(frozen=True)
class PersonName:
    first_name: str
    last_name: str
    middle_name: Optional[str] = None
    prefix: Optional[str] = None  # Mr., Dr., etc.
    suffix: Optional[str] = None  # Jr., PhD, etc.

    def full_name(self) -> str:
        parts = []
        if self.prefix:
            parts.append(self.prefix)
        parts.append(self.first_name)
        if self.middle_name:
            parts.append(self.middle_name)
        parts.append(self.last_name)
        if self.suffix:
            parts.append(self.suffix)
        return " ".join(parts)


class PersonNameBuilder:
    """Builder для PersonName."""

    def __init__(self, first_name: str, last_name: str):
        self._first_name = first_name
        self._last_name = last_name
        self._middle_name: Optional[str] = None
        self._prefix: Optional[str] = None
        self._suffix: Optional[str] = None

    def with_middle_name(self, middle: str) -> "PersonNameBuilder":
        self._middle_name = middle
        return self

    def with_prefix(self, prefix: str) -> "PersonNameBuilder":
        self._prefix = prefix
        return self

    def with_suffix(self, suffix: str) -> "PersonNameBuilder":
        self._suffix = suffix
        return self

    def build(self) -> PersonName:
        return PersonName(
            first_name=self._first_name,
            last_name=self._last_name,
            middle_name=self._middle_name,
            prefix=self._prefix,
            suffix=self._suffix
        )


# Использование
name = (PersonNameBuilder("John", "Doe")
        .with_middle_name("William")
        .with_prefix("Dr.")
        .with_suffix("PhD")
        .build())

print(name.full_name())  # Dr. John William Doe PhD
```

## Сравнение подходов

| Аспект | Value Object | Entity |
|--------|--------------|--------|
| **Идентичность** | По значениям | По ID |
| **Изменяемость** | Неизменяемый | Изменяемый |
| **Жизненный цикл** | Нет | Есть |
| **Пример** | Email, Money | User, Order |

## Плюсы и минусы

### Преимущества

1. **Безопасность** — неизменяемость предотвращает баги
2. **Самодокументирование** — код явно выражает намерение
3. **Инкапсуляция** — валидация и логика в одном месте
4. **Переиспользование** — можно использовать везде
5. **Тестируемость** — легко тестировать изолированно

### Недостатки

1. **Overhead** — создание новых объектов при изменении
2. **Память** — много мелких объектов
3. **Сложность** — для простых случаев избыточно

## Когда использовать?

### Используйте Value Objects для:
- Денежных сумм
- Дат и временных интервалов
- Адресов
- Email, телефонов
- Координат
- Единиц измерения
- Любых доменных концепций без идентичности

### Не используйте для:
- Объектов с уникальной идентичностью
- Объектов с жизненным циклом

## Лучшие практики

1. **Всегда frozen=True** — гарантируйте неизменяемость
2. **Валидация в __post_init__** — fail fast
3. **Методы возвращают новые объекты** — не мутируют
4. **Богатое поведение** — инкапсулируйте логику
5. **Маленькие и focused** — одна концепция на объект

## Заключение

Value Objects — это мощный инструмент для моделирования предметной области. Они делают код безопаснее за счёт неизменяемости, яснее за счёт самодокументирования и надёжнее за счёт встроенной валидации. В Python dataclasses с `frozen=True` — идеальный способ их реализации.
