# Dataclasses

## Что такое Dataclass?

**Dataclass** (PEP 557) — декоратор, который автоматически генерирует специальные методы для классов, хранящих данные: `__init__`, `__repr__`, `__eq__` и другие.

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

# Автоматически создаётся:
# - __init__(self, x: float, y: float)
# - __repr__(self) → "Point(x=1.0, y=2.0)"
# - __eq__(self, other) — сравнение по полям

p1 = Point(1.0, 2.0)
p2 = Point(1.0, 2.0)

print(p1)        # Point(x=1.0, y=2.0)
print(p1 == p2)  # True
```

## Сравнение с обычным классом

```python
# Без dataclass — много boilerplate
class PersonManual:
    def __init__(self, name: str, age: int, email: str):
        self.name = name
        self.age = age
        self.email = email

    def __repr__(self):
        return f"PersonManual(name={self.name!r}, age={self.age!r}, email={self.email!r})"

    def __eq__(self, other):
        if not isinstance(other, PersonManual):
            return NotImplemented
        return (self.name, self.age, self.email) == (other.name, other.age, other.email)

# С dataclass — кратко и понятно
@dataclass
class Person:
    name: str
    age: int
    email: str
```

## Параметры @dataclass

```python
@dataclass(
    init=True,          # Генерировать __init__
    repr=True,          # Генерировать __repr__
    eq=True,            # Генерировать __eq__
    order=False,        # Генерировать __lt__, __le__, __gt__, __ge__
    unsafe_hash=False,  # Генерировать __hash__ (опасно с mutable полями)
    frozen=False,       # Сделать неизменяемым
    match_args=True,    # Для pattern matching (Python 3.10+)
    kw_only=False,      # Все поля только по ключу (Python 3.10+)
    slots=False,        # Использовать __slots__ (Python 3.10+)
)
class MyClass:
    ...
```

## Значения по умолчанию

```python
from dataclasses import dataclass

@dataclass
class User:
    name: str
    age: int = 0           # Простое значение
    active: bool = True    # Простое значение

user = User("Alice")
print(user)  # User(name='Alice', age=0, active=True)
```

### Порядок полей

Поля без значений по умолчанию должны идти перед полями с значениями:

```python
@dataclass
class Wrong:
    name: str = "default"
    age: int  # ❌ Ошибка: non-default after default

@dataclass
class Correct:
    age: int
    name: str = "default"  # ✅ OK
```

## field() — расширенная настройка

```python
from dataclasses import dataclass, field

@dataclass
class Product:
    name: str
    price: float
    tags: list[str] = field(default_factory=list)  # Mutable default!
    internal_id: str = field(default="", repr=False)  # Не показывать в repr
    discount: float = field(default=0, compare=False)  # Не сравнивать

p1 = Product("Laptop", 999.99)
p2 = Product("Laptop", 999.99, discount=10)
print(p1 == p2)  # True (discount не сравнивается)
```

### Параметры field()

| Параметр | Описание |
|----------|----------|
| `default` | Значение по умолчанию |
| `default_factory` | Функция для создания mutable defaults |
| `repr` | Включать в __repr__ (default: True) |
| `compare` | Включать в сравнение (default: True) |
| `hash` | Включать в __hash__ (default: depends) |
| `init` | Включать в __init__ (default: True) |
| `kw_only` | Только keyword аргумент (Python 3.10+) |
| `metadata` | Произвольные метаданные |

### default_factory — важно для mutable типов!

```python
from dataclasses import dataclass, field

# ❌ Опасно — один список для всех экземпляров
@dataclass
class WrongList:
    items: list = []  # Ошибка!

# ✅ Правильно — новый список для каждого экземпляра
@dataclass
class CorrectList:
    items: list = field(default_factory=list)

# Для dict, set и других mutable
@dataclass
class Config:
    settings: dict = field(default_factory=dict)
    enabled_features: set = field(default_factory=set)
```

## __post_init__ — дополнительная инициализация

```python
from dataclasses import dataclass, field

@dataclass
class Rectangle:
    width: float
    height: float
    area: float = field(init=False)  # Не в __init__

    def __post_init__(self):
        self.area = self.width * self.height

rect = Rectangle(3.0, 4.0)
print(rect.area)  # 12.0
```

### Валидация в __post_init__

```python
from dataclasses import dataclass

@dataclass
class User:
    name: str
    age: int

    def __post_init__(self):
        if self.age < 0:
            raise ValueError("Age cannot be negative")
        if not self.name:
            raise ValueError("Name cannot be empty")

User("Alice", 30)   # OK
User("", 30)        # ValueError
User("Bob", -5)     # ValueError
```

## Frozen Dataclasses — неизменяемые

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Point:
    x: float
    y: float

p = Point(1.0, 2.0)
p.x = 3.0  # ❌ FrozenInstanceError

# Frozen dataclasses можно хешировать
points = {Point(1, 2), Point(3, 4)}
point_dict = {Point(0, 0): "origin"}
```

### Изменение frozen объекта

```python
from dataclasses import dataclass, replace

@dataclass(frozen=True)
class Config:
    host: str
    port: int

config = Config("localhost", 8080)
new_config = replace(config, port=9090)  # Создаёт новый объект
print(new_config)  # Config(host='localhost', port=9090)
```

## Наследование Dataclasses

```python
from dataclasses import dataclass

@dataclass
class Person:
    name: str
    age: int

@dataclass
class Employee(Person):
    department: str
    salary: float

emp = Employee("Alice", 30, "Engineering", 75000)
print(emp)  # Employee(name='Alice', age=30, department='Engineering', salary=75000)
```

### С значениями по умолчанию

```python
@dataclass
class Base:
    x: int
    y: int = 0

@dataclass
class Derived(Base):
    z: int = 0  # ✅ OK — после полей с default

# Если нужно обязательное поле в наследнике
@dataclass
class Derived2(Base):
    z: int = field(default=0)  # Workaround
```

## Сравнение и сортировка

```python
from dataclasses import dataclass

@dataclass(order=True)
class Version:
    major: int
    minor: int
    patch: int

v1 = Version(1, 2, 3)
v2 = Version(1, 3, 0)
v3 = Version(2, 0, 0)

print(v1 < v2)  # True (сравнение по полям слева направо)
print(sorted([v3, v1, v2]))  # [Version(1, 2, 3), Version(1, 3, 0), Version(2, 0, 0)]
```

### Кастомный порядок сравнения

```python
from dataclasses import dataclass, field

@dataclass(order=True)
class Student:
    sort_index: float = field(init=False, repr=False)
    name: str
    grade: float

    def __post_init__(self):
        self.sort_index = -self.grade  # Сортировка по убыванию оценки

students = [
    Student("Alice", 85),
    Student("Bob", 92),
    Student("Charlie", 78)
]
print(sorted(students))  # Bob (92), Alice (85), Charlie (78)
```

## Slots (Python 3.10+)

```python
from dataclasses import dataclass

@dataclass(slots=True)
class Point:
    x: float
    y: float

# Экономит память, быстрее доступ к атрибутам
# Но нельзя добавлять новые атрибуты динамически
```

## Keyword-only (Python 3.10+)

```python
from dataclasses import dataclass, field

# Все поля keyword-only
@dataclass(kw_only=True)
class Config:
    host: str
    port: int
    debug: bool = False

config = Config(host="localhost", port=8080)  # ✅
config = Config("localhost", 8080)  # ❌ TypeError

# Отдельные поля keyword-only
@dataclass
class Mixed:
    positional: int
    keyword_only: str = field(kw_only=True)
```

## Конвертация

### В словарь/кортеж

```python
from dataclasses import dataclass, asdict, astuple

@dataclass
class Person:
    name: str
    age: int

person = Person("Alice", 30)

d = asdict(person)   # {'name': 'Alice', 'age': 30}
t = astuple(person)  # ('Alice', 30)
```

### Из словаря

```python
from dataclasses import dataclass

@dataclass
class Person:
    name: str
    age: int

data = {'name': 'Alice', 'age': 30}
person = Person(**data)
```

## Dataclass vs NamedTuple vs TypedDict

| Критерий | dataclass | NamedTuple | TypedDict |
|----------|-----------|------------|-----------|
| Изменяемость | Да (frozen=False) | Нет | Да |
| Наследование | Да | Ограничено | Да |
| Методы | Да | Да | Нет |
| Хеширование | frozen=True | Да | Нет |
| JSON | Через asdict() | Через _asdict() | Напрямую |

```python
from dataclasses import dataclass
from typing import NamedTuple, TypedDict

# Dataclass — наиболее гибкий
@dataclass
class PersonDC:
    name: str
    age: int

# NamedTuple — неизменяемый, как кортеж
class PersonNT(NamedTuple):
    name: str
    age: int

# TypedDict — типизированный словарь
class PersonTD(TypedDict):
    name: str
    age: int
```

## Практические примеры

### Конфигурация

```python
from dataclasses import dataclass, field
from pathlib import Path

@dataclass
class DatabaseConfig:
    host: str = "localhost"
    port: int = 5432
    database: str = "app"
    user: str = "postgres"
    password: str = field(default="", repr=False)  # Не показывать пароль

@dataclass
class AppConfig:
    debug: bool = False
    log_level: str = "INFO"
    database: DatabaseConfig = field(default_factory=DatabaseConfig)
```

### DTO (Data Transfer Object)

```python
from dataclasses import dataclass, asdict
import json

@dataclass
class UserDTO:
    id: int
    username: str
    email: str
    is_active: bool = True

    def to_json(self) -> str:
        return json.dumps(asdict(self))

    @classmethod
    def from_json(cls, json_str: str) -> 'UserDTO':
        return cls(**json.loads(json_str))
```

### Immutable Value Objects

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Money:
    amount: int  # В копейках
    currency: str = "RUB"

    def __add__(self, other: 'Money') -> 'Money':
        if self.currency != other.currency:
            raise ValueError("Cannot add different currencies")
        return Money(self.amount + other.amount, self.currency)

price = Money(10000, "RUB")
tax = Money(2000, "RUB")
total = price + tax  # Money(amount=12000, currency='RUB')
```

## Q&A

**Q: Когда использовать dataclass, а когда обычный класс?**
A: Dataclass — для классов, которые в основном хранят данные. Обычный класс — когда нужна сложная логика инициализации или поведение важнее данных.

**Q: Можно ли использовать dataclass с Pydantic?**
A: Да, но Pydantic предпочитает свой BaseModel. Для валидации лучше использовать Pydantic напрямую.

**Q: Как сделать поле приватным?**
A: Используйте _ или __ в имени. Dataclass не имеет специальной поддержки приватности.

**Q: Dataclass работает с @property?**
A: Да, но property не будет в автогенерированных методах. Добавляйте через `__post_init__` или вручную.
