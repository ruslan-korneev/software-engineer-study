# Law of Demeter — Закон Деметры

## Определение

**Law of Demeter (LoD)**, также известный как **Принцип минимального знания** (Principle of Least Knowledge), гласит:

> "Объект должен общаться только со своими непосредственными друзьями и не разговаривать с незнакомцами."

Формально: метод объекта `M` может вызывать методы только у:
1. Самого объекта (`this`/`self`)
2. Объектов, переданных как параметры в `M`
3. Объектов, созданных внутри `M`
4. Прямых атрибутов объекта
5. Глобальных объектов, доступных в области видимости `M`

---

## Проблема "цепочек вызовов"

### Train Wreck (Крушение поезда)

Типичное нарушение LoD — длинные цепочки вызовов:

```python
# Нарушение Law of Demeter
customer.get_address().get_city().get_country().get_name()

# Это "крушение поезда" — код знает слишком много о внутренней структуре
order.get_customer().get_wallet().get_balance()
```

### Почему это плохо

1. **Высокая связность** — изменение в любом классе цепочки ломает код
2. **Хрупкость** — код зависит от деталей реализации
3. **Сложность тестирования** — нужно мокать всю цепочку
4. **Нарушение инкапсуляции** — внутренняя структура "вытекает" наружу

---

## Примеры

### 1. Базовый пример

**Нарушение LoD:**

```python
class Customer:
    def __init__(self, wallet: "Wallet"):
        self.wallet = wallet


class Wallet:
    def __init__(self, balance: float):
        self.balance = balance

    def deduct(self, amount: float) -> bool:
        if self.balance >= amount:
            self.balance -= amount
            return True
        return False


class Order:
    def __init__(self, customer: Customer, amount: float):
        self.customer = customer
        self.amount = amount


# Нарушение: Order лезет в кошелёк клиента напрямую
def process_payment(order: Order) -> bool:
    # Цепочка: order -> customer -> wallet -> balance
    if order.customer.wallet.balance >= order.amount:
        order.customer.wallet.balance -= order.amount
        return True
    return False
```

**Соблюдение LoD:**

```python
class Customer:
    def __init__(self, wallet: "Wallet"):
        self._wallet = wallet

    def can_afford(self, amount: float) -> bool:
        """Клиент сам знает, может ли заплатить"""
        return self._wallet.has_funds(amount)

    def pay(self, amount: float) -> bool:
        """Клиент сам управляет своим кошельком"""
        return self._wallet.deduct(amount)


class Wallet:
    def __init__(self, balance: float):
        self._balance = balance

    def has_funds(self, amount: float) -> bool:
        return self._balance >= amount

    def deduct(self, amount: float) -> bool:
        if self.has_funds(amount):
            self._balance -= amount
            return True
        return False


class Order:
    def __init__(self, customer: Customer, amount: float):
        self.customer = customer
        self.amount = amount


# Соблюдение: Order общается только с Customer
def process_payment(order: Order) -> bool:
    return order.customer.pay(order.amount)
```

---

### 2. Адреса и локации

**Нарушение LoD:**

```python
class Company:
    def __init__(self, address: "Address"):
        self.address = address


class Address:
    def __init__(self, city: "City"):
        self.city = city


class City:
    def __init__(self, country: "Country"):
        self.country = country


class Country:
    def __init__(self, name: str, tax_rate: float):
        self.name = name
        self.tax_rate = tax_rate


# Нарушение: слишком глубокое знание структуры
def calculate_tax(company: Company, amount: float) -> float:
    tax_rate = company.address.city.country.tax_rate
    return amount * tax_rate


def get_company_location(company: Company) -> str:
    return f"{company.address.city.country.name}"
```

**Соблюдение LoD:**

```python
class Company:
    def __init__(self, address: "Address"):
        self._address = address

    def get_tax_rate(self) -> float:
        """Компания знает свою налоговую ставку"""
        return self._address.get_tax_rate()

    def get_country_name(self) -> str:
        """Компания знает, в какой стране находится"""
        return self._address.get_country_name()


class Address:
    def __init__(self, city: "City"):
        self._city = city

    def get_tax_rate(self) -> float:
        return self._city.get_tax_rate()

    def get_country_name(self) -> str:
        return self._city.get_country_name()


class City:
    def __init__(self, country: "Country"):
        self._country = country

    def get_tax_rate(self) -> float:
        return self._country.tax_rate

    def get_country_name(self) -> str:
        return self._country.name


class Country:
    def __init__(self, name: str, tax_rate: float):
        self.name = name
        self.tax_rate = tax_rate


# Соблюдение: работаем только с Company
def calculate_tax(company: Company, amount: float) -> float:
    return amount * company.get_tax_rate()


def get_company_location(company: Company) -> str:
    return company.get_country_name()
```

---

### 3. Пользователи и настройки

**Нарушение LoD:**

```python
class UserService:
    def get_notification_email(self, user_id: int) -> str:
        user = self.repository.find_by_id(user_id)
        # Нарушение: цепочка вызовов
        return user.profile.settings.notifications.email

    def should_send_newsletter(self, user_id: int) -> bool:
        user = self.repository.find_by_id(user_id)
        return user.profile.settings.notifications.newsletter_enabled
```

**Соблюдение LoD:**

```python
class User:
    def __init__(self, profile: "Profile"):
        self._profile = profile

    def get_notification_email(self) -> str:
        return self._profile.get_notification_email()

    def wants_newsletter(self) -> bool:
        return self._profile.wants_newsletter()


class Profile:
    def __init__(self, settings: "Settings"):
        self._settings = settings

    def get_notification_email(self) -> str:
        return self._settings.get_notification_email()

    def wants_newsletter(self) -> bool:
        return self._settings.wants_newsletter()


class Settings:
    def __init__(self, notifications: "NotificationSettings"):
        self._notifications = notifications

    def get_notification_email(self) -> str:
        return self._notifications.email

    def wants_newsletter(self) -> bool:
        return self._notifications.newsletter_enabled


class NotificationSettings:
    def __init__(self, email: str, newsletter_enabled: bool):
        self.email = email
        self.newsletter_enabled = newsletter_enabled


class UserService:
    def get_notification_email(self, user_id: int) -> str:
        user = self.repository.find_by_id(user_id)
        return user.get_notification_email()  # Один вызов

    def should_send_newsletter(self, user_id: int) -> bool:
        user = self.repository.find_by_id(user_id)
        return user.wants_newsletter()  # Один вызов
```

---

### 4. Работа с заказами

**Нарушение LoD:**

```python
class OrderController:
    def ship_order(self, order_id: int):
        order = self.order_repository.find(order_id)

        # Много цепочек вызовов
        if order.customer.address.country.shipping_available:
            shipping_cost = order.items[0].product.category.shipping_rate
            carrier = order.customer.preferences.default_carrier.name
            # ...
```

**Соблюдение LoD:**

```python
class Order:
    def __init__(self, customer: "Customer", items: list):
        self._customer = customer
        self._items = items

    def can_be_shipped(self) -> bool:
        """Заказ знает, можно ли его отправить"""
        return self._customer.allows_shipping()

    def calculate_shipping_cost(self) -> float:
        """Заказ считает стоимость доставки"""
        return sum(item.get_shipping_cost() for item in self._items)

    def get_preferred_carrier(self) -> str:
        """Заказ знает предпочтительного перевозчика"""
        return self._customer.get_preferred_carrier()

    def ship(self) -> bool:
        """Команда на отправку"""
        if not self.can_be_shipped():
            return False
        # Логика отправки
        return True


class Customer:
    def allows_shipping(self) -> bool:
        return self._address.is_shipping_available()

    def get_preferred_carrier(self) -> str:
        return self._preferences.get_default_carrier()


class OrderController:
    def ship_order(self, order_id: int):
        order = self.order_repository.find(order_id)

        # Чистый интерфейс
        if order.can_be_shipped():
            shipping_cost = order.calculate_shipping_cost()
            carrier = order.get_preferred_carrier()
            order.ship()
```

---

## Исключения из правила

### 1. Data Transfer Objects (DTO)

```python
from dataclasses import dataclass


@dataclass
class UserDTO:
    id: int
    name: str
    email: str
    address: "AddressDTO"


@dataclass
class AddressDTO:
    street: str
    city: str
    country: str


# DTO — это просто структуры данных, не объекты с поведением
# Цепочки доступа допустимы
user_dto = get_user_dto(user_id)
city = user_dto.address.city  # Это нормально для DTO
```

### 2. Fluent Interface / Builder Pattern

```python
# Fluent interface возвращает self для цепочки
query = (
    QueryBuilder()
    .select("name", "email")
    .from_table("users")
    .where("age > 18")
    .order_by("name")
    .build()
)

# Каждый метод возвращает тот же объект — это не нарушение LoD
```

### 3. Standard Library

```python
# Работа со стандартными типами допустима
text = "  hello world  ".strip().upper().split()

# Это не нарушение, т.к. строка — примитив языка
```

---

## Facade для соблюдения LoD

```python
class OrderFacade:
    """Фасад скрывает сложную внутреннюю структуру"""

    def __init__(self, order: Order):
        self._order = order

    def get_customer_email(self) -> str:
        return self._order.customer.contact_info.email

    def get_shipping_address(self) -> str:
        addr = self._order.customer.address
        return f"{addr.street}, {addr.city}, {addr.country}"

    def get_total_with_tax(self) -> float:
        subtotal = self._order.calculate_subtotal()
        tax_rate = self._order.customer.address.country.tax_rate
        return subtotal * (1 + tax_rate)


# Клиентский код использует фасад
def send_order_confirmation(facade: OrderFacade):
    email = facade.get_customer_email()  # Один вызов
    address = facade.get_shipping_address()  # Один вызов
    total = facade.get_total_with_tax()  # Один вызов
    # Отправка...
```

---

## Метрики нарушения LoD

### Количество точек в вызове

```python
# 0 точек — отлично
self.do_something()

# 1 точка — хорошо
self.helper.process()

# 2+ точки — возможное нарушение
self.order.customer.address.city  # 3 точки — плохо
```

### Глубина вложенности

```python
# Плохо: глубокое знание структуры
def get_manager_email(employee):
    return employee.department.manager.contact.email

# Хорошо: делегирование
def get_manager_email(employee):
    return employee.get_manager_email()
```

---

## Best Practices

1. **Создавайте методы-делегаты** — пусть объект сам предоставляет нужные данные
2. **Используйте Facade** — для упрощения сложных интерфейсов
3. **Следуйте Tell, Don't Ask** — связанный принцип
4. **Различайте объекты и структуры** — DTO могут нарушать LoD
5. **Ограничивайте публичный API** — меньше методов = меньше связности

## Типичные ошибки

1. **Слишком много делегатов** — код превращается в "прокси-ад"
2. **Игнорирование для DTO** — применение LoD к простым структурам
3. **Жёсткое следование** — иногда цепочка допустима
4. **Создание методов-обёрток** только для LoD без реальной пользы

---

## Связь с другими принципами

| Принцип | Связь с LoD |
|---------|-------------|
| Encapsulation | LoD защищает инкапсуляцию |
| Tell, Don't Ask | Оба уменьшают связность |
| Single Responsibility | Объект отвечает за свои данные |
| Loose Coupling | LoD обеспечивает слабую связность |

---

## Рефакторинг к LoD

### Шаг 1: Найти цепочки вызовов

```python
# До рефакторинга
invoice.customer.address.country.tax_rate
```

### Шаг 2: Создать метод-делегат

```python
class Invoice:
    def get_tax_rate(self) -> float:
        return self._customer.get_tax_rate()
```

### Шаг 3: Пробросить через цепочку

```python
class Customer:
    def get_tax_rate(self) -> float:
        return self._address.get_tax_rate()


class Address:
    def get_tax_rate(self) -> float:
        return self._country.tax_rate
```

### Шаг 4: Использовать

```python
# После рефакторинга
invoice.get_tax_rate()
```

---

## Резюме

Law of Demeter учит нас:
- Минимизировать знание о внутренней структуре объектов
- Общаться только с непосредственными "друзьями"
- Использовать делегирование вместо цепочек вызовов
- Создавать более устойчивый к изменениям код

> "Only talk to your immediate friends" — Karl Lieberherr (автор закона)
