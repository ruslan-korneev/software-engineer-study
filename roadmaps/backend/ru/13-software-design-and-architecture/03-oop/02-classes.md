# Classes

## Классы в объектно-ориентированном программировании

Класс — это шаблон (blueprint) для создания объектов. Он определяет структуру данных (атрибуты) и поведение (методы), которыми будут обладать объекты этого класса.

---

## Основные концепции

### Класс vs Объект

- **Класс** — это описание (чертёж), определяющее свойства и поведение
- **Объект (экземпляр)** — конкретная реализация класса в памяти

```python
# Класс — это чертёж
class Car:
    def __init__(self, brand: str, model: str):
        self.brand = brand
        self.model = model

    def start(self) -> str:
        return f"{self.brand} {self.model} заводится"


# Объекты — конкретные машины
my_car = Car("Toyota", "Camry")
your_car = Car("BMW", "X5")

print(my_car.start())   # Toyota Camry заводится
print(your_car.start()) # BMW X5 заводится
```

---

## Анатомия класса в Python

### Атрибуты класса и экземпляра

```python
class Employee:
    # Атрибут класса — общий для всех экземпляров
    company_name = "TechCorp"
    employee_count = 0

    def __init__(self, name: str, salary: float):
        # Атрибуты экземпляра — уникальны для каждого объекта
        self.name = name
        self.salary = salary
        self._id = Employee.employee_count  # private by convention

        # Изменяем атрибут класса
        Employee.employee_count += 1

    @property
    def id(self) -> int:
        return self._id


# Атрибут класса доступен через класс и экземпляры
print(Employee.company_name)  # TechCorp

emp1 = Employee("Алиса", 80000)
emp2 = Employee("Борис", 90000)

print(emp1.company_name)  # TechCorp (доступ через экземпляр)
print(Employee.employee_count)  # 2
```

### Типы методов

```python
from datetime import datetime


class User:
    _instances: list["User"] = []

    def __init__(self, username: str, email: str):
        self.username = username
        self.email = email
        self.created_at = datetime.now()
        User._instances.append(self)

    # Обычный метод — работает с конкретным экземпляром
    def get_info(self) -> str:
        return f"{self.username} ({self.email})"

    # Метод класса — работает с классом, не с экземпляром
    @classmethod
    def get_all_users(cls) -> list["User"]:
        return cls._instances.copy()

    @classmethod
    def from_dict(cls, data: dict) -> "User":
        """Фабричный метод — альтернативный конструктор."""
        return cls(data["username"], data["email"])

    # Статический метод — не зависит ни от класса, ни от экземпляра
    @staticmethod
    def validate_email(email: str) -> bool:
        """Утилитарная функция, логически связанная с классом."""
        return "@" in email and "." in email


# Использование разных типов методов
user1 = User("alice", "alice@example.com")
user2 = User.from_dict({"username": "bob", "email": "bob@example.com"})

print(user1.get_info())  # Метод экземпляра
print(User.get_all_users())  # Метод класса
print(User.validate_email("test@test.com"))  # Статический метод
```

---

## Специальные методы (Magic Methods / Dunder Methods)

Python использует специальные методы для определения поведения объектов в различных контекстах.

### Основные магические методы

```python
from typing import Self


class Vector:
    """Двумерный вектор с поддержкой арифметических операций."""

    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y

    # Строковое представление для разработчиков
    def __repr__(self) -> str:
        return f"Vector({self.x}, {self.y})"

    # Строковое представление для пользователей
    def __str__(self) -> str:
        return f"({self.x}, {self.y})"

    # Сложение: vector1 + vector2
    def __add__(self, other: Self) -> Self:
        return Vector(self.x + other.x, self.y + other.y)

    # Вычитание: vector1 - vector2
    def __sub__(self, other: Self) -> Self:
        return Vector(self.x - other.x, self.y - other.y)

    # Умножение на скаляр: vector * 3
    def __mul__(self, scalar: float) -> Self:
        return Vector(self.x * scalar, self.y * scalar)

    # Умножение справа: 3 * vector
    def __rmul__(self, scalar: float) -> Self:
        return self.__mul__(scalar)

    # Сравнение на равенство
    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Vector):
            return NotImplemented
        return self.x == other.x and self.y == other.y

    # Хеширование (для использования в set и dict)
    def __hash__(self) -> int:
        return hash((self.x, self.y))

    # Абсолютное значение (длина вектора)
    def __abs__(self) -> float:
        return (self.x ** 2 + self.y ** 2) ** 0.5

    # Проверка на истинность
    def __bool__(self) -> bool:
        return self.x != 0 or self.y != 0


# Использование
v1 = Vector(3, 4)
v2 = Vector(1, 2)

print(repr(v1))     # Vector(3, 4)
print(str(v1))      # (3, 4)
print(v1 + v2)      # (4, 6)
print(v1 * 2)       # (6, 8)
print(2 * v1)       # (6, 8)
print(abs(v1))      # 5.0
print(v1 == Vector(3, 4))  # True
```

### Контекстные менеджеры

```python
from typing import Self


class DatabaseConnection:
    """Пример класса с поддержкой контекстного менеджера."""

    def __init__(self, connection_string: str):
        self.connection_string = connection_string
        self.connected = False

    def connect(self) -> None:
        print(f"Подключение к {self.connection_string}")
        self.connected = True

    def disconnect(self) -> None:
        print("Отключение от базы данных")
        self.connected = False

    def __enter__(self) -> Self:
        """Вызывается при входе в блок with."""
        self.connect()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb) -> bool:
        """Вызывается при выходе из блока with."""
        self.disconnect()
        # Возвращаем False — не подавляем исключения
        return False

    def execute(self, query: str) -> list:
        if not self.connected:
            raise RuntimeError("Нет подключения к БД")
        print(f"Выполняем: {query}")
        return []


# Использование с контекстным менеджером
with DatabaseConnection("postgresql://localhost/mydb") as db:
    db.execute("SELECT * FROM users")
# Автоматическое отключение при выходе из блока
```

---

## Dataclasses — современный подход

Python 3.7+ предоставляет декоратор `@dataclass` для упрощения создания классов данных.

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional


@dataclass
class Product:
    """Простой dataclass с автоматически сгенерированными методами."""
    name: str
    price: float
    category: str
    in_stock: bool = True


@dataclass(frozen=True)  # Неизменяемый (immutable)
class Point:
    x: float
    y: float


@dataclass
class Order:
    """Dataclass с настройками по умолчанию и вычисляемыми полями."""
    customer_id: int
    items: list[Product] = field(default_factory=list)
    created_at: datetime = field(default_factory=datetime.now)
    notes: Optional[str] = None

    # Поле, исключённое из __init__ и сравнения
    _total_cache: Optional[float] = field(
        default=None,
        init=False,
        repr=False,
        compare=False
    )

    def __post_init__(self):
        """Вызывается после __init__ для дополнительной инициализации."""
        self._calculate_total()

    def _calculate_total(self) -> None:
        self._total_cache = sum(p.price for p in self.items)

    @property
    def total(self) -> float:
        if self._total_cache is None:
            self._calculate_total()
        return self._total_cache


# Использование
product1 = Product("Ноутбук", 50000, "Электроника")
product2 = Product("Мышь", 1500, "Электроника")

order = Order(
    customer_id=123,
    items=[product1, product2]
)

print(order)
print(f"Итого: {order.total}")

# frozen dataclass
p = Point(1.0, 2.0)
# p.x = 5.0  # Ошибка! Объект неизменяемый
```

---

## Наследование классов

### Простое наследование

```python
from abc import ABC, abstractmethod
from decimal import Decimal


class Payment(ABC):
    """Абстрактный базовый класс для платежей."""

    def __init__(self, amount: Decimal):
        self.amount = amount
        self.processed = False

    @abstractmethod
    def process(self) -> bool:
        """Обработка платежа — реализуется в подклассах."""
        pass

    def __str__(self) -> str:
        status = "обработан" if self.processed else "ожидает"
        return f"Платёж {self.amount} руб. — {status}"


class CreditCardPayment(Payment):
    def __init__(self, amount: Decimal, card_number: str):
        super().__init__(amount)
        self.card_number = card_number[-4:]  # только последние 4 цифры

    def process(self) -> bool:
        print(f"Обработка платежа картой ****{self.card_number}")
        self.processed = True
        return True


class CryptoPayment(Payment):
    def __init__(self, amount: Decimal, wallet_address: str):
        super().__init__(amount)
        self.wallet_address = wallet_address

    def process(self) -> bool:
        print(f"Обработка крипто-платежа на {self.wallet_address[:10]}...")
        self.processed = True
        return True


# Полиморфное использование
payments: list[Payment] = [
    CreditCardPayment(Decimal("1000"), "1234567890123456"),
    CryptoPayment(Decimal("5000"), "0x1234abcd..."),
]

for payment in payments:
    payment.process()
    print(payment)
```

### Множественное наследование и MRO

```python
class Flyable:
    def fly(self) -> str:
        return "Летит"


class Swimmable:
    def swim(self) -> str:
        return "Плывёт"


class Walkable:
    def walk(self) -> str:
        return "Идёт"


class Duck(Flyable, Swimmable, Walkable):
    """Утка умеет всё!"""

    def quack(self) -> str:
        return "Кря!"


class Penguin(Swimmable, Walkable):
    """Пингвин не умеет летать."""
    pass


duck = Duck()
print(duck.fly())   # Летит
print(duck.swim())  # Плывёт
print(duck.walk())  # Идёт

# Method Resolution Order (порядок разрешения методов)
print(Duck.__mro__)
# (<class 'Duck'>, <class 'Flyable'>, <class 'Swimmable'>,
#  <class 'Walkable'>, <class 'object'>)
```

---

## Связи между классами

### Ассоциация

```python
class Author:
    def __init__(self, name: str):
        self.name = name


class Book:
    """Книга связана с автором (ассоциация)."""

    def __init__(self, title: str, author: Author):
        self.title = title
        self.author = author  # ссылка на другой объект


author = Author("Лев Толстой")
book = Book("Война и мир", author)
```

### Агрегация

```python
class Engine:
    def __init__(self, horsepower: int):
        self.horsepower = horsepower


class Car:
    """Машина содержит двигатель, но двигатель может существовать отдельно."""

    def __init__(self, brand: str, engine: Engine):
        self.brand = brand
        self.engine = engine  # агрегация — слабая связь


engine = Engine(200)
car = Car("BMW", engine)
# engine существует независимо от car
```

### Композиция

```python
class Heart:
    def beat(self) -> str:
        return "Тук-тук"


class Human:
    """Человек содержит сердце — сердце не может существовать без человека."""

    def __init__(self, name: str):
        self.name = name
        self._heart = Heart()  # композиция — сильная связь

    def is_alive(self) -> bool:
        return self._heart is not None


# Heart создаётся внутри Human и не может существовать отдельно
```

---

## Best Practices

### Принцип единственной ответственности

```python
# Плохо — класс делает слишком много
class UserBad:
    def __init__(self, name: str, email: str):
        self.name = name
        self.email = email

    def save_to_database(self):
        pass  # работа с БД

    def send_email(self, message: str):
        pass  # отправка почты

    def generate_report(self):
        pass  # генерация отчёта


# Хорошо — разделение ответственности
class User:
    def __init__(self, name: str, email: str):
        self.name = name
        self.email = email


class UserRepository:
    def save(self, user: User) -> None:
        pass


class EmailService:
    def send(self, to: str, message: str) -> None:
        pass


class ReportGenerator:
    def generate_user_report(self, user: User) -> str:
        pass
```

### Предпочитайте композицию наследованию

```python
# Вместо глубокой иерархии наследования
# используйте композицию и делегирование

class Logger:
    def log(self, message: str) -> None:
        print(f"[LOG] {message}")


class Validator:
    def validate(self, data: dict) -> bool:
        return bool(data)


class UserService:
    """Использует композицию вместо наследования."""

    def __init__(self, logger: Logger, validator: Validator):
        self._logger = logger
        self._validator = validator

    def create_user(self, data: dict) -> None:
        if self._validator.validate(data):
            self._logger.log(f"Создание пользователя: {data}")
        else:
            self._logger.log("Ошибка валидации")
```

---

## Типичные ошибки

1. **God Class** — класс, который знает и делает слишком много
2. **Anemic Domain Model** — классы только с данными, без поведения
3. **Нарушение инкапсуляции** — прямой доступ к приватным атрибутам
4. **Избыточное наследование** — наследование ради переиспользования кода
5. **Изменяемые атрибуты класса** — общие списки/словари как атрибуты класса

```python
# Ошибка: изменяемый атрибут класса
class BadClass:
    items = []  # Общий для всех экземпляров!

    def add_item(self, item):
        self.items.append(item)


# Правильно: создавать в __init__
class GoodClass:
    def __init__(self):
        self.items = []  # Уникальный для каждого экземпляра

    def add_item(self, item):
        self.items.append(item)
```
