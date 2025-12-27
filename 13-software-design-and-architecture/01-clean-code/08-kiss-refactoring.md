# KISS and Refactoring

## Принцип KISS

KISS (Keep It Simple, Stupid) — принцип проектирования, который гласит:

> **Большинство систем работают лучше, если они остаются простыми, а не усложняются.**

Простой код:
- Легче понять
- Легче поддерживать
- Содержит меньше багов
- Быстрее разрабатывается

## Признаки нарушения KISS

### 1. Преждевременная оптимизация

```python
# Плохо — оптимизация без необходимости
class UserCache:
    def __init__(self):
        self._cache = {}
        self._lru_order = []
        self._max_size = 1000
        self._hit_count = 0
        self._miss_count = 0
        self._lock = threading.Lock()

    def get(self, user_id):
        with self._lock:
            if user_id in self._cache:
                self._hit_count += 1
                self._update_lru(user_id)
                return self._cache[user_id]
            self._miss_count += 1
            return None


# Хорошо — простое решение (если достаточно)
from functools import lru_cache

@lru_cache(maxsize=1000)
def get_user(user_id):
    return db.users.find(user_id)
```

### 2. Избыточная абстракция

```python
# Плохо — слишком много слоёв
class UserServiceInterface(ABC):
    @abstractmethod
    def get_user(self, user_id): pass

class UserServiceImpl(UserServiceInterface):
    def __init__(self, user_repository_factory):
        self.repo = user_repository_factory.create()

class UserRepositoryFactory:
    def create(self):
        return UserRepository()

class UserRepository:
    def __init__(self, database_connection_provider):
        self.db = database_connection_provider.get_connection()

# Для простого приложения достаточно:
class UserService:
    def __init__(self, db):
        self.db = db

    def get_user(self, user_id):
        return self.db.users.find(user_id)
```

### 3. Паттерны ради паттернов

```python
# Плохо — паттерн Стратегия для простой задачи
class GreetingStrategy(ABC):
    @abstractmethod
    def greet(self, name): pass

class EnglishGreeting(GreetingStrategy):
    def greet(self, name):
        return f"Hello, {name}!"

class SpanishGreeting(GreetingStrategy):
    def greet(self, name):
        return f"Hola, {name}!"

class GreetingContext:
    def __init__(self, strategy):
        self.strategy = strategy

    def execute(self, name):
        return self.strategy.greet(name)


# Хорошо — простой словарь
GREETINGS = {
    'en': lambda name: f"Hello, {name}!",
    'es': lambda name: f"Hola, {name}!",
}

def greet(name, language='en'):
    return GREETINGS.get(language, GREETINGS['en'])(name)
```

### 4. Чрезмерная конфигурируемость

```python
# Плохо — слишком много опций
def send_email(
    to, subject, body,
    cc=None, bcc=None,
    priority='normal',
    encoding='utf-8',
    format='html',
    attachments=None,
    headers=None,
    retry_count=3,
    retry_delay=1,
    timeout=30,
    ssl=True,
    verify_ssl=True,
    track_opens=False,
    track_clicks=False,
    unsubscribe_link=None,
    template_id=None,
    template_variables=None,
    schedule_at=None,
    batch_id=None,
    custom_args=None
):
    pass


# Хорошо — разумные дефолты, расширение через объекты
@dataclass
class EmailMessage:
    to: str
    subject: str
    body: str
    cc: List[str] = field(default_factory=list)

def send_email(message: EmailMessage) -> None:
    # Простая реализация с разумными дефолтами
    pass
```

## Принципы простоты

### 1. YAGNI (You Aren't Gonna Need It)

```python
# Плохо — функциональность "на будущее"
class User:
    def __init__(self):
        self.name = ""
        self.email = ""
        # "Может понадобиться потом"
        self.facebook_id = None
        self.twitter_id = None
        self.linkedin_id = None
        self.instagram_id = None
        self.tiktok_id = None
        self.metadata = {}
        self.tags = []
        self.custom_fields = {}


# Хорошо — только то, что нужно сейчас
class User:
    def __init__(self, name: str, email: str):
        self.name = name
        self.email = email
```

### 2. Предпочитайте явное неявному

```python
# Плохо — "магия"
class SmartModel:
    def __getattr__(self, name):
        if name.startswith('get_'):
            field = name[4:]
            return lambda: self._data.get(field)
        elif name.startswith('set_'):
            field = name[4:]
            return lambda v: self._data.__setitem__(field, v)
        raise AttributeError(name)


# Хорошо — явные методы
class User:
    def __init__(self, name: str, email: str):
        self.name = name
        self.email = email

    def get_name(self) -> str:
        return self.name

    def set_name(self, name: str) -> None:
        self.name = name
```

### 3. Минимизируйте состояние

```python
# Плохо — много состояния
class OrderProcessor:
    def __init__(self):
        self.current_order = None
        self.items = []
        self.total = 0
        self.discount = 0
        self.tax = 0
        self.is_processed = False

    def set_order(self, order):
        self.current_order = order
        self.items = order.items
        self.total = 0
        self.discount = 0
        self.tax = 0
        self.is_processed = False

    def calculate_total(self):
        self.total = sum(item.price for item in self.items)

    def apply_discount(self, percent):
        self.discount = self.total * percent / 100

    def calculate_tax(self):
        self.tax = (self.total - self.discount) * 0.2

    def process(self):
        self.calculate_total()
        self.apply_discount(10)
        self.calculate_tax()
        self.is_processed = True
        return self.total - self.discount + self.tax


# Хорошо — чистые функции, минимум состояния
def process_order(order: Order) -> ProcessedOrder:
    subtotal = calculate_subtotal(order.items)
    discount = calculate_discount(subtotal, percent=10)
    taxable = subtotal - discount
    tax = calculate_tax(taxable, rate=0.2)
    total = taxable + tax

    return ProcessedOrder(
        order=order,
        subtotal=subtotal,
        discount=discount,
        tax=tax,
        total=total
    )


def calculate_subtotal(items: List[Item]) -> Decimal:
    return sum(item.price for item in items)


def calculate_discount(amount: Decimal, percent: int) -> Decimal:
    return amount * percent / 100


def calculate_tax(amount: Decimal, rate: float) -> Decimal:
    return amount * Decimal(str(rate))
```

## Что такое рефакторинг?

Рефакторинг — это изменение внутренней структуры кода без изменения его внешнего поведения.

**Цели рефакторинга:**
- Улучшить читаемость
- Упростить поддержку
- Устранить дублирование
- Подготовить к добавлению новых функций

## Когда рефакторить?

### Правило трёх (Rule of Three)

1. **Первый раз** — просто напишите код
2. **Второй раз** — допустите дублирование, но отметьте
3. **Третий раз** — рефакторьте

### Признаки необходимости рефакторинга (Code Smells)

1. **Дублирование кода**
2. **Длинные методы**
3. **Большие классы**
4. **Длинные списки параметров**
5. **Разбросанные изменения**
6. **Зависть к чужим данным**
7. **Примитивная одержимость**

## Техники рефакторинга

### 1. Extract Method (Извлечение метода)

```python
# До
def print_order(order):
    print("=" * 40)
    print("ORDER")
    print("=" * 40)

    total = 0
    for item in order.items:
        line_total = item.price * item.quantity
        total += line_total
        print(f"{item.name}: {item.quantity} x ${item.price} = ${line_total}")

    print("-" * 40)
    print(f"Total: ${total}")
    print("=" * 40)


# После
def print_order(order):
    print_header()
    total = print_items(order.items)
    print_footer(total)


def print_header():
    print("=" * 40)
    print("ORDER")
    print("=" * 40)


def print_items(items) -> Decimal:
    total = Decimal(0)
    for item in items:
        total += print_item(item)
    return total


def print_item(item) -> Decimal:
    line_total = item.price * item.quantity
    print(f"{item.name}: {item.quantity} x ${item.price} = ${line_total}")
    return line_total


def print_footer(total):
    print("-" * 40)
    print(f"Total: ${total}")
    print("=" * 40)
```

### 2. Extract Class (Извлечение класса)

```python
# До
class Person:
    def __init__(self):
        self.name = ""
        self.street = ""
        self.city = ""
        self.zip_code = ""
        self.phone_area_code = ""
        self.phone_number = ""

    def get_full_address(self):
        return f"{self.street}, {self.city} {self.zip_code}"

    def get_phone(self):
        return f"({self.phone_area_code}) {self.phone_number}"


# После
@dataclass
class Address:
    street: str
    city: str
    zip_code: str

    def full(self) -> str:
        return f"{self.street}, {self.city} {self.zip_code}"


@dataclass
class PhoneNumber:
    area_code: str
    number: str

    def full(self) -> str:
        return f"({self.area_code}) {self.number}"


@dataclass
class Person:
    name: str
    address: Address
    phone: PhoneNumber
```

### 3. Replace Conditional with Polymorphism

```python
# До
class Employee:
    def calculate_pay(self):
        if self.type == 'hourly':
            return self.hours * self.hourly_rate
        elif self.type == 'salaried':
            return self.monthly_salary
        elif self.type == 'commission':
            return self.base_salary + self.sales * self.commission_rate
        else:
            raise ValueError(f"Unknown employee type: {self.type}")


# После
class Employee(ABC):
    @abstractmethod
    def calculate_pay(self) -> Decimal:
        pass


class HourlyEmployee(Employee):
    def __init__(self, hours: int, hourly_rate: Decimal):
        self.hours = hours
        self.hourly_rate = hourly_rate

    def calculate_pay(self) -> Decimal:
        return self.hours * self.hourly_rate


class SalariedEmployee(Employee):
    def __init__(self, monthly_salary: Decimal):
        self.monthly_salary = monthly_salary

    def calculate_pay(self) -> Decimal:
        return self.monthly_salary


class CommissionEmployee(Employee):
    def __init__(self, base_salary: Decimal, sales: Decimal, rate: Decimal):
        self.base_salary = base_salary
        self.sales = sales
        self.commission_rate = rate

    def calculate_pay(self) -> Decimal:
        return self.base_salary + self.sales * self.commission_rate
```

### 4. Replace Magic Numbers with Constants

```python
# До
def calculate_circle_area(radius):
    return 3.14159 * radius ** 2

def is_adult(age):
    return age >= 18

def get_discount(total):
    if total > 100:
        return total * 0.1
    return 0


# После
import math

ADULT_AGE = 18
DISCOUNT_THRESHOLD = 100
DISCOUNT_RATE = 0.1


def calculate_circle_area(radius: float) -> float:
    return math.pi * radius ** 2


def is_adult(age: int) -> bool:
    return age >= ADULT_AGE


def get_discount(total: Decimal) -> Decimal:
    if total > DISCOUNT_THRESHOLD:
        return total * DISCOUNT_RATE
    return Decimal(0)
```

### 5. Introduce Parameter Object

```python
# До
def search_users(
    name: str = None,
    email: str = None,
    min_age: int = None,
    max_age: int = None,
    status: str = None,
    created_after: datetime = None,
    created_before: datetime = None,
    limit: int = 100,
    offset: int = 0
):
    pass


# После
@dataclass
class UserSearchCriteria:
    name: Optional[str] = None
    email: Optional[str] = None
    min_age: Optional[int] = None
    max_age: Optional[int] = None
    status: Optional[str] = None
    created_after: Optional[datetime] = None
    created_before: Optional[datetime] = None


@dataclass
class Pagination:
    limit: int = 100
    offset: int = 0


def search_users(
    criteria: UserSearchCriteria,
    pagination: Pagination = None
) -> List[User]:
    pagination = pagination or Pagination()
    # ...
```

### 6. Replace Temp with Query

```python
# До
def get_total_price(order):
    base_price = order.quantity * order.item_price
    discount = max(0, order.quantity - 100) * order.item_price * 0.05
    shipping = min(base_price * 0.1, 100)
    return base_price - discount + shipping


# После
def get_total_price(order):
    return get_base_price(order) - get_discount(order) + get_shipping(order)


def get_base_price(order):
    return order.quantity * order.item_price


def get_discount(order):
    return max(0, order.quantity - 100) * order.item_price * 0.05


def get_shipping(order):
    return min(get_base_price(order) * 0.1, 100)
```

## Безопасный рефакторинг

### 1. Сначала тесты

```python
# Перед рефакторингом — убедитесь, что тесты проходят
def test_calculate_total():
    order = Order(items=[
        Item(price=10, quantity=2),
        Item(price=5, quantity=3)
    ])
    assert calculate_total(order) == 35


def test_calculate_total_with_discount():
    order = Order(items=[Item(price=100, quantity=1)])
    order.discount = 10
    assert calculate_total(order) == 90
```

### 2. Маленькие шаги

```python
# Шаг 1: Извлечь метод
def calculate_total(order):
    subtotal = calculate_subtotal(order.items)  # Новый метод
    # ... остальной код


# Шаг 2: Проверить тесты

# Шаг 3: Извлечь следующий метод
def calculate_total(order):
    subtotal = calculate_subtotal(order.items)
    discount = calculate_discount(subtotal, order.discount)  # Ещё один
    # ...


# Шаг 4: Снова проверить тесты
```

### 3. Один рефакторинг за раз

```python
# НЕ делайте так:
# - Извлечь класс
# - Переименовать переменные
# - Изменить интерфейс
# - Добавить новую функцию
# Всё одним коммитом!

# Делайте так:
# Коммит 1: Extract class Address from Person
# Коммит 2: Rename 'addr' to 'address' in UserService
# Коммит 3: Add validation to Address
```

## Best Practices

### 1. Рефакторьте регулярно

```
┌─────────────────────────────────────────┐
│        Red → Green → Refactor           │
│                                         │
│  1. Напишите падающий тест (Red)       │
│  2. Сделайте тест зелёным (Green)      │
│  3. Улучшите код (Refactor)            │
│  4. Повторите                          │
└─────────────────────────────────────────┘
```

### 2. Используйте IDE

Современные IDE автоматизируют рефакторинг:
- Rename (переименование)
- Extract Method/Variable/Class
- Inline
- Move
- Change Signature

### 3. Документируйте решения

```python
# Почему не используем паттерн X?
#
# Рассматривали применение Strategy для обработки платежей,
# но текущие требования не предполагают добавление новых
# методов оплаты. Если появятся — отрефакторим.
#
# Ticket: PROJ-123
# Date: 2024-01-15

def process_payment(order: Order) -> None:
    # Простая реализация для единственного метода
    stripe.charge(order.total)
```

## Заключение

KISS и рефакторинг — взаимодополняющие практики:

1. **KISS** — пишите простой код изначально
2. **Рефакторинг** — упрощайте код по мере развития

**Ключевые принципы:**
- Простота важнее "умности"
- Рефакторьте маленькими шагами
- Тесты — страховка при рефакторинге
- Не оптимизируйте преждевременно
- Код пишется для людей, не для машин

Помните: **простой код, который работает, лучше сложного кода, который "может понадобиться"**.
