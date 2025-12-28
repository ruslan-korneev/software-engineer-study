# Abstract Base Classes (ABC)

[prev: 03-methods-dunder](./03-methods-dunder.md) | [next: 05-protocols](./05-protocols.md)
---

## Что такое ABC?

**Abstract Base Class (ABC)** — абстрактный базовый класс, который нельзя инстанцировать напрямую. Используется для определения интерфейса, который должны реализовать подклассы.

```python
from abc import ABC, abstractmethod

class Animal(ABC):
    @abstractmethod
    def speak(self):
        """Каждое животное должно уметь издавать звук"""
        pass

# ❌ Нельзя создать экземпляр абстрактного класса
animal = Animal()  # TypeError: Can't instantiate abstract class

# ✅ Можно создать подкласс с реализацией
class Dog(Animal):
    def speak(self):
        return "Гав!"

dog = Dog()
dog.speak()  # "Гав!"
```

## Зачем нужны ABC?

1. **Определение контракта** — явный интерфейс для подклассов
2. **Ранняя проверка** — ошибка при создании объекта, если не реализован метод
3. **Документация** — понятно какие методы обязательны
4. **Полиморфизм** — единый интерфейс для разных реализаций

```python
# Без ABC — ошибка только при вызове метода
class BadAnimal:
    pass

animal = BadAnimal()
animal.speak()  # AttributeError (поздняя ошибка)

# С ABC — ошибка при создании
class BadAnimal(Animal):
    pass  # Не реализован speak()

animal = BadAnimal()  # TypeError (ранняя ошибка)
```

## Создание ABC

### Базовый синтаксис

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float:
        """Вычисляет площадь фигуры"""
        pass

    @abstractmethod
    def perimeter(self) -> float:
        """Вычисляет периметр фигуры"""
        pass
```

### Реализация подкласса

```python
class Rectangle(Shape):
    def __init__(self, width: float, height: float):
        self.width = width
        self.height = height

    def area(self) -> float:
        return self.width * self.height

    def perimeter(self) -> float:
        return 2 * (self.width + self.height)

class Circle(Shape):
    def __init__(self, radius: float):
        self.radius = radius

    def area(self) -> float:
        from math import pi
        return pi * self.radius ** 2

    def perimeter(self) -> float:
        from math import pi
        return 2 * pi * self.radius
```

### Использование полиморфизма

```python
def total_area(shapes: list[Shape]) -> float:
    """Суммирует площади всех фигур"""
    return sum(shape.area() for shape in shapes)

shapes = [Rectangle(3, 4), Circle(5), Rectangle(2, 2)]
print(total_area(shapes))  # 98.54...
```

## Абстрактные свойства

```python
from abc import ABC, abstractmethod

class Vehicle(ABC):
    @property
    @abstractmethod
    def max_speed(self) -> int:
        """Максимальная скорость в км/ч"""
        pass

    @property
    @abstractmethod
    def fuel_type(self) -> str:
        """Тип топлива"""
        pass

class Car(Vehicle):
    @property
    def max_speed(self) -> int:
        return 200

    @property
    def fuel_type(self) -> str:
        return "gasoline"
```

## Абстрактные classmethod и staticmethod

```python
from abc import ABC, abstractmethod

class Serializable(ABC):
    @classmethod
    @abstractmethod
    def from_json(cls, data: str) -> 'Serializable':
        """Создаёт объект из JSON"""
        pass

    @abstractmethod
    def to_json(self) -> str:
        """Сериализует объект в JSON"""
        pass

class User(Serializable):
    def __init__(self, name: str, age: int):
        self.name = name
        self.age = age

    @classmethod
    def from_json(cls, data: str) -> 'User':
        import json
        d = json.loads(data)
        return cls(d['name'], d['age'])

    def to_json(self) -> str:
        import json
        return json.dumps({'name': self.name, 'age': self.age})
```

## ABC с реализацией по умолчанию

Абстрактные методы могут иметь реализацию по умолчанию:

```python
from abc import ABC, abstractmethod

class Logger(ABC):
    @abstractmethod
    def log(self, message: str) -> None:
        """Логирует сообщение. Подклассы могут вызвать super()."""
        print(f"[LOG] {message}")

class FileLogger(Logger):
    def __init__(self, filename: str):
        self.filename = filename

    def log(self, message: str) -> None:
        super().log(message)  # Вызываем базовую реализацию
        with open(self.filename, 'a') as f:
            f.write(f"{message}\n")
```

## Регистрация виртуальных подклассов

Можно "зарегистрировать" класс как подкласс ABC без наследования:

```python
from abc import ABC, abstractmethod

class Drawable(ABC):
    @abstractmethod
    def draw(self):
        pass

# Класс НЕ наследует Drawable
class ThirdPartyWidget:
    def draw(self):
        print("Drawing widget...")

# Регистрируем как виртуальный подкласс
Drawable.register(ThirdPartyWidget)

# Проверки работают
widget = ThirdPartyWidget()
isinstance(widget, Drawable)  # True
issubclass(ThirdPartyWidget, Drawable)  # True
```

### Использование декоратора

```python
@Drawable.register
class AnotherWidget:
    def draw(self):
        print("Drawing another widget...")
```

### Когда использовать register

- Интеграция со сторонними библиотеками
- Duck typing с проверкой `isinstance()`
- Обратная совместимость

## Встроенные ABC из collections.abc

Python предоставляет множество полезных ABC:

```python
from collections.abc import (
    Iterable,      # Имеет __iter__
    Iterator,      # Имеет __iter__ и __next__
    Sequence,      # Неизменяемая последовательность (list, tuple)
    MutableSequence,  # Изменяемая последовательность
    Mapping,       # Отображение (dict)
    MutableMapping,   # Изменяемое отображение
    Set,           # Множество
    Callable,      # Вызываемый объект
    Hashable,      # Хешируемый объект
)

# Проверка интерфейса
isinstance([1, 2, 3], Sequence)     # True
isinstance({1, 2, 3}, Set)          # True
isinstance({'a': 1}, Mapping)       # True
isinstance(lambda x: x, Callable)   # True
```

### Создание своей последовательности

```python
from collections.abc import Sequence

class MyRange(Sequence):
    def __init__(self, start: int, stop: int):
        self._start = start
        self._stop = stop

    def __getitem__(self, index):
        if index >= len(self):
            raise IndexError
        return self._start + index

    def __len__(self):
        return max(0, self._stop - self._start)

# Автоматически получаем: __contains__, __iter__, __reversed__, index, count
r = MyRange(0, 5)
list(r)       # [0, 1, 2, 3, 4]
3 in r        # True
r.count(2)    # 1
```

## __subclasshook__ — автоматическая регистрация

Позволяет определить условия для автоматического признания класса подклассом:

```python
from abc import ABC, abstractmethod

class Closeable(ABC):
    @abstractmethod
    def close(self):
        pass

    @classmethod
    def __subclasshook__(cls, C):
        if cls is Closeable:
            if hasattr(C, 'close') and callable(getattr(C, 'close')):
                return True
        return NotImplemented

# Любой класс с методом close() — подкласс Closeable
class File:
    def close(self):
        print("Closing file")

isinstance(File(), Closeable)  # True (без register!)
```

## ABC vs Protocols

| ABC | Protocol |
|-----|----------|
| Явное наследование | Структурная типизация (duck typing) |
| Проверка при создании | Проверка статическими анализаторами |
| Runtime проверка | Нет runtime проверки (без @runtime_checkable) |
| Может иметь реализацию | Только сигнатуры методов |

```python
# ABC — явное наследование
class Shape(ABC):
    @abstractmethod
    def area(self) -> float: pass

class Circle(Shape):  # Явно наследует
    def area(self) -> float: ...

# Protocol — структурная типизация
from typing import Protocol

class Drawable(Protocol):
    def draw(self) -> None: ...

class Button:  # НЕ наследует Drawable
    def draw(self) -> None: ...

def render(d: Drawable) -> None:  # Button подходит!
    d.draw()
```

## Best Practices

### 1. Используйте ABC для общих интерфейсов

```python
# ✅ Хорошо — общий интерфейс для хранилищ
class Repository(ABC):
    @abstractmethod
    def save(self, entity): pass

    @abstractmethod
    def find(self, id): pass

    @abstractmethod
    def delete(self, id): pass

class PostgresRepository(Repository):
    # Реализация для PostgreSQL
    pass

class MongoRepository(Repository):
    # Реализация для MongoDB
    pass
```

### 2. Не злоупотребляйте абстракциями

```python
# ❌ Избыточно для простого случая
class Addable(ABC):
    @abstractmethod
    def add(self, a, b): pass

# ✅ Достаточно простой функции
def add(a, b):
    return a + b
```

### 3. Комбинируйте с дженериками

```python
from abc import ABC, abstractmethod
from typing import Generic, TypeVar

T = TypeVar('T')

class Repository(ABC, Generic[T]):
    @abstractmethod
    def save(self, entity: T) -> None: pass

    @abstractmethod
    def find(self, id: int) -> T | None: pass

class UserRepository(Repository['User']):
    def save(self, entity: 'User') -> None: ...
    def find(self, id: int) -> 'User | None': ...
```

## Q&A

**Q: Когда использовать ABC, а когда Protocol?**
A: ABC — для runtime проверок, когда нужна реализация по умолчанию или register. Protocol — для статической типизации и duck typing.

**Q: Можно ли наследовать от нескольких ABC?**
A: Да, это безопасно, так как ABC обычно не содержат состояния.

**Q: Чем ABC отличается от интерфейсов в Java?**
A: ABC могут содержать реализацию методов, состояние, и поддерживают множественное наследование.

**Q: Почему register() не проверяет реализацию методов?**
A: Это для совместимости с duck typing. Ответственность за корректность на разработчике.

---
[prev: 03-methods-dunder](./03-methods-dunder.md) | [next: 05-protocols](./05-protocols.md)