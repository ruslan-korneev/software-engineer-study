# Protocols (Структурная типизация)

[prev: 04-abc](./04-abc.md) | [next: 08a-paradigms](../08a-paradigms.md)
---

## Что такое Protocol?

**Protocol** (PEP 544) — механизм структурной типизации в Python. Позволяет определить интерфейс без явного наследования. Если объект имеет нужные методы — он соответствует протоколу.

```python
from typing import Protocol

class Drawable(Protocol):
    def draw(self) -> None: ...

# Класс НЕ наследует Drawable, но соответствует протоколу
class Circle:
    def draw(self) -> None:
        print("Drawing circle")

class Square:
    def draw(self) -> None:
        print("Drawing square")

def render(shape: Drawable) -> None:
    shape.draw()

# Оба работают — duck typing!
render(Circle())  # Drawing circle
render(Square())  # Drawing square
```

## Структурная vs Номинальная типизация

### Номинальная (ABC, наследование)

```python
from abc import ABC, abstractmethod

class Animal(ABC):
    @abstractmethod
    def speak(self) -> str: ...

class Dog(Animal):  # Явное наследование обязательно
    def speak(self) -> str:
        return "Woof!"

# Без наследования — ошибка типов
class Cat:
    def speak(self) -> str:
        return "Meow!"

def greet(animal: Animal) -> None:
    print(animal.speak())

greet(Cat())  # Ошибка mypy: Cat не является Animal
```

### Структурная (Protocol)

```python
from typing import Protocol

class Speaker(Protocol):
    def speak(self) -> str: ...

class Dog:  # Наследование НЕ нужно
    def speak(self) -> str:
        return "Woof!"

class Cat:  # Наследование НЕ нужно
    def speak(self) -> str:
        return "Meow!"

def greet(speaker: Speaker) -> None:
    print(speaker.speak())

greet(Dog())  # OK
greet(Cat())  # OK — Cat соответствует протоколу
```

## Создание протоколов

### Базовый синтаксис

```python
from typing import Protocol

class Closeable(Protocol):
    def close(self) -> None: ...

class Readable(Protocol):
    def read(self, n: int = -1) -> str: ...

class Writable(Protocol):
    def write(self, data: str) -> int: ...
```

### Протокол с атрибутами

```python
from typing import Protocol

class Named(Protocol):
    name: str  # Обязательный атрибут

class HasId(Protocol):
    id: int
    created_at: str

# Соответствует Named
class User:
    def __init__(self, name: str):
        self.name = name

def greet(entity: Named) -> str:
    return f"Hello, {entity.name}!"
```

### Протокол с property

```python
from typing import Protocol

class Sized(Protocol):
    @property
    def size(self) -> int: ...

class MyList:
    def __init__(self, items: list):
        self._items = items

    @property
    def size(self) -> int:
        return len(self._items)

def is_empty(obj: Sized) -> bool:
    return obj.size == 0
```

## Наследование протоколов

```python
from typing import Protocol

class Readable(Protocol):
    def read(self) -> str: ...

class Writable(Protocol):
    def write(self, data: str) -> None: ...

# Комбинированный протокол
class ReadWritable(Readable, Writable, Protocol):
    pass

# Или с дополнительными методами
class Stream(Readable, Writable, Protocol):
    def seek(self, position: int) -> None: ...
    def tell(self) -> int: ...
```

## @runtime_checkable

По умолчанию протоколы проверяются только статически. Для runtime-проверок нужен декоратор:

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Closeable(Protocol):
    def close(self) -> None: ...

class File:
    def close(self) -> None:
        print("Closing file")

class NotCloseable:
    pass

# Runtime проверки работают!
isinstance(File(), Closeable)       # True
isinstance(NotCloseable(), Closeable)  # False
```

### Ограничения @runtime_checkable

```python
@runtime_checkable
class HasLength(Protocol):
    def __len__(self) -> int: ...

# Проверяет только наличие метода, НЕ сигнатуру!
class BadLength:
    def __len__(self, x: int) -> str:  # Неправильная сигнатура
        return "wrong"

isinstance(BadLength(), HasLength)  # True (проверяет только имя!)
```

## Протоколы с Generic

```python
from typing import Protocol, TypeVar

T = TypeVar('T')
T_co = TypeVar('T_co', covariant=True)

class Container(Protocol[T]):
    def get(self) -> T: ...
    def set(self, value: T) -> None: ...

class Box:
    def __init__(self, value: int):
        self._value = value

    def get(self) -> int:
        return self._value

    def set(self, value: int) -> None:
        self._value = value

def process(container: Container[int]) -> int:
    return container.get() * 2

process(Box(5))  # 10
```

### Ковариантные и контравариантные протоколы

```python
from typing import Protocol, TypeVar

T_co = TypeVar('T_co', covariant=True)

class Reader(Protocol[T_co]):
    def read(self) -> T_co: ...

# Reader[Animal] принимает Reader[Dog] (ковариантность)
```

## Встроенные протоколы

Python имеет несколько встроенных протоколов:

```python
from typing import (
    SupportsInt,      # Имеет __int__
    SupportsFloat,    # Имеет __float__
    SupportsBytes,    # Имеет __bytes__
    SupportsAbs,      # Имеет __abs__
    SupportsRound,    # Имеет __round__
    SupportsIndex,    # Имеет __index__
    Hashable,         # Имеет __hash__
    Sized,            # Имеет __len__
    Iterable,         # Имеет __iter__
    Iterator,         # Имеет __iter__ и __next__
    Reversible,       # Имеет __reversed__
    Container,        # Имеет __contains__
)

def double(x: SupportsFloat) -> float:
    return float(x) * 2

double(5)       # OK — int имеет __float__
double(3.14)    # OK
double("5.0")   # Ошибка — str не имеет __float__
```

## Callback протоколы

```python
from typing import Protocol

class Handler(Protocol):
    def __call__(self, event: str) -> None: ...

def on_click(event: str) -> None:
    print(f"Clicked: {event}")

class ClickHandler:
    def __call__(self, event: str) -> None:
        print(f"Handler: {event}")

def register(handler: Handler) -> None:
    handler("button_press")

register(on_click)      # Функция
register(ClickHandler())  # Callable объект
```

## Практические примеры

### Repository Pattern

```python
from typing import Protocol, TypeVar, Generic

T = TypeVar('T')
ID = TypeVar('ID')

class Repository(Protocol[T, ID]):
    def get(self, id: ID) -> T | None: ...
    def save(self, entity: T) -> None: ...
    def delete(self, id: ID) -> None: ...
    def list(self) -> list[T]: ...

# Любая реализация с нужными методами подходит
class InMemoryUserRepo:
    def __init__(self):
        self._users: dict[int, 'User'] = {}

    def get(self, id: int) -> 'User | None':
        return self._users.get(id)

    def save(self, entity: 'User') -> None:
        self._users[entity.id] = entity

    def delete(self, id: int) -> None:
        self._users.pop(id, None)

    def list(self) -> list['User']:
        return list(self._users.values())
```

### Dependency Injection

```python
from typing import Protocol

class EmailSender(Protocol):
    def send(self, to: str, subject: str, body: str) -> None: ...

class UserService:
    def __init__(self, email_sender: EmailSender):
        self._email = email_sender

    def register_user(self, email: str) -> None:
        # ... создание пользователя ...
        self._email.send(email, "Welcome!", "Thanks for registering")

# Легко подменить для тестов
class FakeEmailSender:
    def __init__(self):
        self.sent: list[tuple] = []

    def send(self, to: str, subject: str, body: str) -> None:
        self.sent.append((to, subject, body))

# В продакшене
class SMTPEmailSender:
    def send(self, to: str, subject: str, body: str) -> None:
        # Реальная отправка email
        pass
```

### Plugin System

```python
from typing import Protocol

class Plugin(Protocol):
    name: str

    def initialize(self) -> None: ...
    def execute(self, data: dict) -> dict: ...
    def cleanup(self) -> None: ...

def load_plugins(plugins: list[Plugin]) -> None:
    for plugin in plugins:
        print(f"Loading {plugin.name}")
        plugin.initialize()
```

## Protocol vs ABC: когда что использовать

| Критерий | Protocol | ABC |
|----------|----------|-----|
| Сторонние классы | ✅ Работает | ❌ Нужен register |
| Duck typing | ✅ Естественный | ❌ Явное наследование |
| Runtime isinstance | Нужен @runtime_checkable | ✅ Работает |
| Реализация по умолчанию | ❌ Нет | ✅ Есть |
| Проверка при создании | ❌ Нет | ✅ Есть |
| Статический анализ | ✅ Отлично | ✅ Хорошо |

### Когда использовать Protocol

```python
# 1. Интеграция со сторонними библиотеками
class JSONEncoder(Protocol):
    def encode(self, obj: object) -> str: ...

# 2. Duck typing интерфейсы
class Comparable(Protocol):
    def __lt__(self, other: 'Comparable') -> bool: ...

# 3. Callback-и и стратегии
class Validator(Protocol):
    def validate(self, value: str) -> bool: ...
```

### Когда использовать ABC

```python
# 1. Нужна реализация по умолчанию
class BaseHandler(ABC):
    @abstractmethod
    def handle(self, request): pass

    def log(self, message):  # Реализация по умолчанию
        print(f"[LOG] {message}")

# 2. Нужна проверка при создании объекта
class Shape(ABC):
    @abstractmethod
    def area(self) -> float: pass

# 3. Требуется явный контракт в рамках проекта
```

## Best Practices

### 1. Делайте протоколы минимальными

```python
# ✅ Один метод — один протокол
class Readable(Protocol):
    def read(self) -> str: ...

class Writable(Protocol):
    def write(self, data: str) -> None: ...

# ❌ Слишком много в одном
class FileOperations(Protocol):
    def read(self) -> str: ...
    def write(self, data: str) -> None: ...
    def seek(self, pos: int) -> None: ...
    def close(self) -> None: ...
```

### 2. Используйте Protocol для публичного API

```python
# Библиотека определяет протокол
class Logger(Protocol):
    def info(self, message: str) -> None: ...
    def error(self, message: str) -> None: ...

# Пользователь реализует как хочет
def configure_app(logger: Logger) -> None:
    logger.info("App configured")
```

### 3. Документируйте контракт

```python
class Serializable(Protocol):
    """Объекты, которые можно сериализовать в JSON.

    Реализации должны гарантировать, что:
    - to_json() возвращает валидный JSON
    - from_json(to_json(obj)) создаёт эквивалентный объект
    """
    def to_json(self) -> str: ...

    @classmethod
    def from_json(cls, data: str) -> 'Serializable': ...
```

## Q&A

**Q: Protocol проверяет типы методов?**
A: Статически — да (mypy проверяет). В runtime (@runtime_checkable) — только имена методов.

**Q: Можно ли наследовать от Protocol?**
A: Да, но не обязательно. Наследование явно показывает намерение реализовать протокол.

**Q: Работает ли Protocol с __init__?**
A: Нет, `__init__` не входит в структурную проверку. Используйте classmethod или factory.

**Q: Как Protocol работает с приватными методами?**
A: Протоколы обычно описывают публичный интерфейс. Приватные методы не должны быть частью протокола.

---
[prev: 04-abc](./04-abc.md) | [next: 08a-paradigms](../08a-paradigms.md)