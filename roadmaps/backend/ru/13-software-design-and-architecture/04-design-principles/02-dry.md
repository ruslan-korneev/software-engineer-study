# DRY — Don't Repeat Yourself

## Определение

**DRY (Don't Repeat Yourself)** — принцип разработки программного обеспечения, направленный на уменьшение дублирования кода и информации. Был сформулирован Эндрю Хантом и Дэвидом Томасом в книге "The Pragmatic Programmer".

> "Every piece of knowledge must have a single, unambiguous, authoritative representation within a system."
>
> "Каждая часть знания должна иметь единственное, недвусмысленное, авторитетное представление в системе."

---

## Зачем нужен DRY

### Проблемы дублирования кода

1. **Сложность поддержки** — изменения нужно вносить в нескольких местах
2. **Риск ошибок** — легко забыть обновить одну из копий
3. **Увеличение объема кода** — сложнее читать и понимать
4. **Несогласованность** — разные копии могут "разойтись" со временем

### Преимущества DRY

- Единая точка изменений
- Меньше кода = меньше багов
- Проще тестирование
- Лучше читаемость

---

## Виды дублирования

### 1. Дублирование кода (Code Duplication)

Самый очевидный вид — копирование одинаковых блоков кода.

**Плохо:**

```python
def calculate_order_total(order):
    total = 0
    for item in order.items:
        total += item.price * item.quantity
    if order.discount:
        total = total - (total * order.discount / 100)
    total = total + (total * 0.2)  # VAT 20%
    return total


def calculate_cart_total(cart):
    total = 0
    for item in cart.items:
        total += item.price * item.quantity
    if cart.discount:
        total = total - (total * cart.discount / 100)
    total = total + (total * 0.2)  # VAT 20%
    return total
```

**Хорошо:**

```python
def calculate_total(items, discount=None):
    """Универсальный расчет суммы с учетом скидки и НДС"""
    subtotal = sum(item.price * item.quantity for item in items)

    if discount:
        subtotal -= subtotal * discount / 100

    # Добавляем НДС
    total = subtotal * 1.2
    return total


def calculate_order_total(order):
    return calculate_total(order.items, order.discount)


def calculate_cart_total(cart):
    return calculate_total(cart.items, cart.discount)
```

---

### 2. Дублирование логики (Logic Duplication)

Одинаковая логика, записанная разными способами.

**Плохо:**

```python
class UserValidator:
    def is_valid_email(self, email: str) -> bool:
        return "@" in email and "." in email.split("@")[1]


class CustomerValidator:
    def check_email_format(self, email: str) -> bool:
        parts = email.split("@")
        if len(parts) != 2:
            return False
        return "." in parts[1]
```

**Хорошо:**

```python
import re


def is_valid_email(email: str) -> bool:
    """Единственный источник истины для валидации email"""
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return bool(re.match(pattern, email))


class UserValidator:
    def validate_email(self, email: str) -> bool:
        return is_valid_email(email)


class CustomerValidator:
    def validate_email(self, email: str) -> bool:
        return is_valid_email(email)
```

---

### 3. Дублирование знаний (Knowledge Duplication)

Одна и та же информация хранится в нескольких местах.

**Плохо:**

```python
# config.py
DATABASE_URL = "postgresql://localhost:5432/mydb"

# docker-compose.yml
# services:
#   db:
#     environment:
#       - POSTGRES_DB=mydb
#       - POSTGRES_PORT=5432

# В коде
def get_connection():
    return connect("postgresql://localhost:5432/mydb")  # Дублирование!
```

**Хорошо:**

```python
import os

# Единственный источник истины — переменные окружения
DATABASE_URL = os.environ.get("DATABASE_URL")

def get_connection():
    return connect(DATABASE_URL)
```

---

### 4. Дублирование данных (Data Duplication)

**Плохо:**

```python
class Product:
    def __init__(self, name: str, price: float, quantity: int):
        self.name = name
        self.price = price
        self.quantity = quantity
        self.total_value = price * quantity  # Дублирование!


# total_value рассинхронизируется при изменении price или quantity
product = Product("Laptop", 1000, 5)
product.quantity = 10
print(product.total_value)  # Всё ещё 5000, а не 10000!
```

**Хорошо:**

```python
class Product:
    def __init__(self, name: str, price: float, quantity: int):
        self.name = name
        self.price = price
        self.quantity = quantity

    @property
    def total_value(self) -> float:
        """Вычисляемое свойство — всегда актуально"""
        return self.price * self.quantity


product = Product("Laptop", 1000, 5)
product.quantity = 10
print(product.total_value)  # Корректно: 10000
```

---

## Техники устранения дублирования

### 1. Извлечение метода/функции

```python
# До
def process_user(user):
    # Валидация email
    if not "@" in user.email:
        raise ValueError("Invalid email")
    if not "." in user.email.split("@")[1]:
        raise ValueError("Invalid email")
    # Сохранение
    db.save(user)


def process_customer(customer):
    # Валидация email (дублирование!)
    if not "@" in customer.email:
        raise ValueError("Invalid email")
    if not "." in customer.email.split("@")[1]:
        raise ValueError("Invalid email")
    # Отправка уведомления
    send_notification(customer)


# После
def validate_email(email: str):
    if not "@" in email:
        raise ValueError("Invalid email")
    if not "." in email.split("@")[1]:
        raise ValueError("Invalid email")


def process_user(user):
    validate_email(user.email)
    db.save(user)


def process_customer(customer):
    validate_email(customer.email)
    send_notification(customer)
```

---

### 2. Использование наследования и композиции

```python
# Композиция — предпочтительный способ
class EmailValidator:
    def validate(self, email: str) -> bool:
        return "@" in email and "." in email


class UserService:
    def __init__(self, email_validator: EmailValidator):
        self.email_validator = email_validator

    def create_user(self, email: str):
        if not self.email_validator.validate(email):
            raise ValueError("Invalid email")
        # создание пользователя


class CustomerService:
    def __init__(self, email_validator: EmailValidator):
        self.email_validator = email_validator

    def register_customer(self, email: str):
        if not self.email_validator.validate(email):
            raise ValueError("Invalid email")
        # регистрация клиента
```

---

### 3. Шаблоны и генерики

```python
from typing import TypeVar, Generic, List

T = TypeVar('T')


class Repository(Generic[T]):
    """Универсальный репозиторий для любых сущностей"""

    def __init__(self):
        self._storage: List[T] = []

    def add(self, entity: T) -> None:
        self._storage.append(entity)

    def get_all(self) -> List[T]:
        return self._storage.copy()

    def find(self, predicate) -> List[T]:
        return [e for e in self._storage if predicate(e)]


# Использование
user_repo = Repository[User]()
product_repo = Repository[Product]()
```

---

### 4. Константы и конфигурация

```python
# constants.py
class TaxRates:
    VAT = 0.20
    SALES_TAX = 0.08
    LUXURY_TAX = 0.15


class Limits:
    MAX_LOGIN_ATTEMPTS = 3
    SESSION_TIMEOUT_MINUTES = 30
    MAX_FILE_SIZE_MB = 10


# Использование
from constants import TaxRates, Limits

def calculate_price_with_tax(price: float) -> float:
    return price * (1 + TaxRates.VAT)


def check_file_size(size_mb: float) -> bool:
    return size_mb <= Limits.MAX_FILE_SIZE_MB
```

---

## Когда НЕ применять DRY

### 1. Случайное совпадение (Accidental Duplication)

```python
# Эти функции выглядят похоже, но имеют разные причины для изменения
def format_user_name(user):
    return f"{user.first_name} {user.last_name}"


def format_author_name(author):
    return f"{author.first_name} {author.last_name}"

# НЕ нужно объединять! Требования к форматированию имени пользователя
# и автора могут измениться независимо друг от друга
```

### 2. Преждевременная абстракция

```python
# Если код дублируется только 2 раза — возможно, ещё рано рефакторить
# Правило трёх: рефакторьте, когда дублирование появляется в третий раз
```

### 3. Разные контексты использования

```python
# OrderValidator и PaymentValidator могут иметь похожий код,
# но относятся к разным бизнес-доменам и будут развиваться независимо
```

---

## Best Practices

1. **Правило трёх** — рефакторьте при третьем появлении дублирования
2. **Единый источник истины** — конфигурация, константы, бизнес-правила
3. **Вычисляемые свойства** вместо хранения избыточных данных
4. **Переиспользуемые компоненты** — библиотеки, утилиты, общие модули
5. **Шаблоны проектирования** — Strategy, Template Method, Factory

## Типичные ошибки

1. **Чрезмерный DRY** — создание сложных абстракций ради устранения минимального дублирования
2. **Объединение несвязанного** — слияние кода, который похож случайно
3. **Игнорирование контекста** — DRY применим к знаниям и логике, не только к коду
4. **Premature DRY** — рефакторинг до понимания реальных паттернов использования

---

## Связь с другими принципами

- **SOLID (SRP)** — DRY помогает выделять ответственности в отдельные модули
- **KISS** — иногда небольшое дублирование проще, чем сложная абстракция
- **YAGNI** — не создавайте абстракции "на будущее"

> "Duplication is far cheaper than the wrong abstraction" — Sandi Metz
