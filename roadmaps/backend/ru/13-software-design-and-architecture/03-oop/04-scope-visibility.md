# Scope and Visibility

## Область видимости и модификаторы доступа

Область видимости (scope) определяет, где в коде доступна переменная или атрибут. Модификаторы доступа (visibility modifiers) контролируют доступ к членам класса извне.

---

## Уровни видимости в ООП

### Классические модификаторы доступа

| Модификатор | Описание | Python-конвенция |
|-------------|----------|------------------|
| **public** | Доступен отовсюду | `name` |
| **protected** | Доступен в классе и подклассах | `_name` |
| **private** | Доступен только внутри класса | `__name` |

---

## Модификаторы доступа в Python

Python использует соглашения об именовании вместо жёстких ограничений доступа. Это философия "мы все взрослые люди" (we're all consenting adults).

### Public (публичные члены)

```python
class User:
    def __init__(self, name: str, email: str):
        # Публичные атрибуты — доступны везде
        self.name = name
        self.email = email

    # Публичный метод
    def get_display_name(self) -> str:
        return f"{self.name} <{self.email}>"


user = User("Иван", "ivan@example.com")
print(user.name)  # Прямой доступ
print(user.email)  # Прямой доступ
print(user.get_display_name())  # Вызов метода
```

### Protected (защищённые члены)

Одно подчёркивание `_` — соглашение, означающее "для внутреннего использования".

```python
class BankAccount:
    def __init__(self, account_number: str, balance: float):
        self.account_number = account_number
        self._balance = balance  # Protected: не трогай извне!

    def _validate_amount(self, amount: float) -> bool:
        """Protected метод — используется только внутри класса и подклассов."""
        return amount > 0

    def deposit(self, amount: float) -> None:
        if self._validate_amount(amount):
            self._balance += amount

    @property
    def balance(self) -> float:
        return self._balance


class SavingsAccount(BankAccount):
    def __init__(self, account_number: str, balance: float, interest_rate: float):
        super().__init__(account_number, balance)
        self._interest_rate = interest_rate

    def add_interest(self) -> None:
        # Подкласс может обращаться к protected членам родителя
        interest = self._balance * self._interest_rate
        self._balance += interest


account = SavingsAccount("123", 1000, 0.05)
account.add_interest()

# Технически доступно, но нарушает соглашение!
print(account._balance)  # Работает, но IDE выдаст предупреждение
```

### Private (приватные члены)

Двойное подчёркивание `__` активирует механизм name mangling.

```python
class SecureData:
    def __init__(self, public_data: str, secret: str):
        self.public_data = public_data
        self.__secret = secret  # Private: name mangling

    def __private_method(self) -> str:
        """Private метод — недоступен напрямую извне."""
        return f"Секрет: {self.__secret}"

    def reveal_secret(self, password: str) -> str | None:
        if password == "correct_password":
            return self.__private_method()
        return None


data = SecureData("открытые данные", "суперсекрет")

print(data.public_data)  # Работает
# print(data.__secret)  # AttributeError!

# Name mangling: Python переименовывает __secret в _SecureData__secret
print(data._SecureData__secret)  # Работает, но это хак!

# Правильный доступ — через публичный метод
print(data.reveal_secret("correct_password"))
```

### Name Mangling в деталях

```python
class Parent:
    def __init__(self):
        self.__value = "parent_value"

    def get_value(self) -> str:
        return self.__value


class Child(Parent):
    def __init__(self):
        super().__init__()
        self.__value = "child_value"  # Это другой атрибут!

    def get_child_value(self) -> str:
        return self.__value


child = Child()
print(child.get_value())       # parent_value (из Parent)
print(child.get_child_value()) # child_value (из Child)

# Под капотом создаются разные атрибуты:
print(child._Parent__value)  # parent_value
print(child._Child__value)   # child_value
```

---

## Properties — контролируемый доступ

Properties позволяют создать публичный интерфейс с контролем доступа.

```python
class Temperature:
    def __init__(self, celsius: float = 0):
        self._celsius = celsius

    @property
    def celsius(self) -> float:
        """Геттер для температуры в Цельсиях."""
        return self._celsius

    @celsius.setter
    def celsius(self, value: float) -> None:
        """Сеттер с валидацией."""
        if value < -273.15:
            raise ValueError("Температура ниже абсолютного нуля невозможна")
        self._celsius = value

    @property
    def fahrenheit(self) -> float:
        """Вычисляемое свойство — только для чтения."""
        return self._celsius * 9/5 + 32

    @fahrenheit.setter
    def fahrenheit(self, value: float) -> None:
        """Можно задать через Фаренгейт."""
        self.celsius = (value - 32) * 5/9

    @property
    def kelvin(self) -> float:
        return self._celsius + 273.15


temp = Temperature(25)
print(f"{temp.celsius}°C = {temp.fahrenheit}°F = {temp.kelvin}K")

temp.fahrenheit = 100
print(f"{temp.celsius:.1f}°C")  # 37.8°C

# temp.celsius = -300  # ValueError!
```

### Read-only properties

```python
class Circle:
    def __init__(self, radius: float):
        self._radius = radius

    @property
    def radius(self) -> float:
        return self._radius

    @radius.setter
    def radius(self, value: float) -> None:
        if value <= 0:
            raise ValueError("Радиус должен быть положительным")
        self._radius = value

    @property
    def area(self) -> float:
        """Только для чтения — нет сеттера."""
        import math
        return math.pi * self._radius ** 2

    @property
    def circumference(self) -> float:
        """Только для чтения."""
        import math
        return 2 * math.pi * self._radius


circle = Circle(5)
print(f"Площадь: {circle.area:.2f}")
# circle.area = 100  # AttributeError: can't set attribute
```

---

## Дескрипторы — продвинутый контроль доступа

Дескрипторы — мощный механизм для создания повторно используемой логики доступа.

```python
class PositiveNumber:
    """Дескриптор для положительных чисел."""

    def __set_name__(self, owner, name):
        self.name = name
        self.private_name = f"_desc_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self.private_name, 0)

    def __set__(self, obj, value):
        if value <= 0:
            raise ValueError(f"{self.name} должен быть положительным")
        setattr(obj, self.private_name, value)


class ValidatedString:
    """Дескриптор для строк с валидацией."""

    def __init__(self, max_length: int):
        self.max_length = max_length

    def __set_name__(self, owner, name):
        self.name = name
        self.private_name = f"_desc_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self.private_name, "")

    def __set__(self, obj, value):
        if not isinstance(value, str):
            raise TypeError(f"{self.name} должен быть строкой")
        if len(value) > self.max_length:
            raise ValueError(f"{self.name} не должен превышать {self.max_length} символов")
        setattr(obj, self.private_name, value)


class Product:
    name = ValidatedString(100)
    price = PositiveNumber()
    quantity = PositiveNumber()

    def __init__(self, name: str, price: float, quantity: int):
        self.name = name
        self.price = price
        self.quantity = quantity

    @property
    def total_value(self) -> float:
        return self.price * self.quantity


product = Product("Ноутбук", 50000, 10)
print(f"{product.name}: {product.total_value}")

# product.price = -100  # ValueError: price должен быть положительным
# product.name = "x" * 200  # ValueError: name не должен превышать 100 символов
```

---

## Области видимости переменных

### LEGB Rule

Python ищет переменные в порядке: Local -> Enclosing -> Global -> Built-in.

```python
x = "global"  # Global scope

def outer():
    x = "enclosing"  # Enclosing scope

    def inner():
        x = "local"  # Local scope
        print(f"Inner: {x}")  # local

    inner()
    print(f"Outer: {x}")  # enclosing

outer()
print(f"Global: {x}")  # global


# Изменение внешних переменных
def counter():
    count = 0  # Enclosing

    def increment():
        nonlocal count  # Ссылка на enclosing переменную
        count += 1
        return count

    return increment


counter_func = counter()
print(counter_func())  # 1
print(counter_func())  # 2
print(counter_func())  # 3
```

### Замыкания и области видимости

```python
def make_multiplier(factor: float):
    """Фабрика функций-множителей."""

    def multiplier(x: float) -> float:
        # factor берётся из enclosing scope
        return x * factor

    return multiplier


double = make_multiplier(2)
triple = make_multiplier(3)

print(double(5))  # 10
print(triple(5))  # 15

# Замыкание "помнит" factor
print(double.__closure__[0].cell_contents)  # 2
```

---

## Видимость в модулях

### Публичный API модуля

```python
# mymodule.py

# Публичные функции и классы
def public_function():
    """Часть публичного API."""
    pass

class PublicClass:
    """Часть публичного API."""
    pass


# Приватные (внутренние) функции
def _internal_helper():
    """Для внутреннего использования."""
    pass

class _InternalClass:
    """Для внутреннего использования."""
    pass


# Определение публичного API через __all__
__all__ = ["public_function", "PublicClass"]
```

### Использование `__all__`

```python
# __all__ контролирует, что экспортируется при "from module import *"

# utils.py
__all__ = ["format_date", "parse_date"]

def format_date(date):
    return str(date)

def parse_date(string):
    return string

def _internal_function():
    """Не экспортируется."""
    pass

# Не в __all__, но доступна при явном импорте
def helper():
    pass


# main.py
from utils import *  # Импортирует только format_date и parse_date
from utils import helper  # Явный импорт работает
from utils import _internal_function  # Тоже работает (но не рекомендуется)
```

---

## Видимость в контексте тестирования

```python
class PaymentProcessor:
    """Класс с разными уровнями доступа для тестирования."""

    def __init__(self, api_key: str):
        self.__api_key = api_key  # Private
        self._retry_count = 3     # Protected

    def process_payment(self, amount: float) -> bool:
        """Публичный метод — основной API."""
        return self._validate_and_send(amount)

    def _validate_and_send(self, amount: float) -> bool:
        """Protected — можно переопределить в тестах."""
        if not self._validate_amount(amount):
            return False
        return self.__send_to_gateway(amount)

    def _validate_amount(self, amount: float) -> bool:
        """Protected — легко тестировать."""
        return amount > 0

    def __send_to_gateway(self, amount: float) -> bool:
        """Private — скрыт от тестов, но можно замокать через name mangling."""
        print(f"Sending {amount} to gateway...")
        return True


# В тестах:
class TestablePaymentProcessor(PaymentProcessor):
    """Подкласс для тестирования."""

    def _validate_and_send(self, amount: float) -> bool:
        """Переопределяем protected метод для тестов."""
        # Не отправляем реальные запросы в тестах
        return self._validate_amount(amount)


# Или используем mock
from unittest.mock import patch

processor = PaymentProcessor("test_key")

# Мокаем приватный метод через name mangling
with patch.object(processor, "_PaymentProcessor__send_to_gateway", return_value=True):
    result = processor.process_payment(100)
    assert result is True
```

---

## Best Practices

### 1. Используйте минимально необходимый уровень доступа

```python
class GoodDesign:
    def __init__(self):
        # Только то, что действительно нужно скрыть
        self.__secret_key = "..."  # Реально приватное
        self._cache = {}           # Внутренняя реализация
        self.name = ""             # Публичный атрибут
```

### 2. Предпочитайте properties прямому доступу

```python
class Rectangle:
    def __init__(self, width: float, height: float):
        self._width = width
        self._height = height

    @property
    def width(self) -> float:
        return self._width

    @width.setter
    def width(self, value: float) -> None:
        if value <= 0:
            raise ValueError("Ширина должна быть положительной")
        self._width = value

    # Аналогично для height
```

### 3. Документируйте ожидаемое использование

```python
class APIClient:
    """
    Клиент для работы с API.

    Публичные методы:
        - get(endpoint) - GET запрос
        - post(endpoint, data) - POST запрос

    Внутренние методы (_prefix):
        - Не использовать напрямую
        - Могут измениться без предупреждения
    """

    def __init__(self, base_url: str):
        self._base_url = base_url
        self._session = None

    def get(self, endpoint: str) -> dict:
        """Публичный API для GET запросов."""
        return self._request("GET", endpoint)

    def _request(self, method: str, endpoint: str) -> dict:
        """Внутренний метод — не для прямого использования."""
        pass
```

---

## Типичные ошибки

1. **Избыточные геттеры и сеттеры**
   ```python
   # Плохо: бессмысленные геттеры/сеттеры
   class Bad:
       def __init__(self):
           self._x = 0

       def get_x(self): return self._x
       def set_x(self, v): self._x = v

   # Хорошо: просто публичный атрибут
   class Good:
       def __init__(self):
           self.x = 0
   ```

2. **Использование __ для всего**
   ```python
   # Плохо: всё приватное без причины
   class Paranoid:
       def __init__(self):
           self.__a = 1
           self.__b = 2
           self.__c = 3

   # Хорошо: только действительно секретные данные
   class Reasonable:
       def __init__(self):
           self.a = 1
           self._b = 2      # Внутренняя реализация
           self.__secret = 3  # Действительно секретное
   ```

3. **Игнорирование соглашений**
   ```python
   # Плохо: обращение к protected/private извне
   obj._internal_method()  # Не делайте так
   obj._ClassName__private  # Тем более так

   # Хорошо: используйте публичный API
   obj.public_method()
   ```

4. **Утечка внутреннего состояния**
   ```python
   class Leaky:
       def __init__(self):
           self._items = []

       def get_items(self):
           return self._items  # Утечка! Можно изменить извне

   class Safe:
       def __init__(self):
           self._items = []

       def get_items(self):
           return self._items.copy()  # Возвращаем копию
   ```
