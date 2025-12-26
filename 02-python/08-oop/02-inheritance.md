# Inheritance (Наследование)

## Что такое наследование?

Механизм создания нового класса на основе существующего.

- **Parent/Base/Superclass** — родительский класс
- **Child/Derived/Subclass** — дочерний класс

## Базовый синтаксис

```python
class Animal:
    def __init__(self, name):
        self.name = name

    def speak(self):
        return "Какой-то звук"

class Dog(Animal):  # Dog наследует от Animal
    def speak(self):  # Переопределяем метод
        return f"{self.name}: Гав!"

dog = Dog("Шарик")
print(dog.name)     # Шарик (унаследовано)
print(dog.speak())  # Шарик: Гав! (переопределено)
```

## `super()` — вызов родительского метода

```python
class Animal:
    def __init__(self, name, age):
        self.name = name
        self.age = age

class Dog(Animal):
    def __init__(self, name, age, breed):
        super().__init__(name, age)  # Вызов родительского __init__
        self.breed = breed           # Добавляем своё

dog = Dog("Рекс", 5, "Овчарка")
print(dog.name, dog.breed)  # Рекс Овчарка
```

## Множественное наследование

```python
class Flyable:
    def fly(self):
        return "Летит!"

class Swimmable:
    def swim(self):
        return "Плывёт!"

class Duck(Animal, Flyable, Swimmable):
    def speak(self):
        return f"{self.name}: Кря!"

duck = Duck("Дональд")
duck.fly()   # Летит!
duck.swim()  # Плывёт!
```

### MRO (Method Resolution Order)

Порядок поиска методов при множественном наследовании:

```python
print(Duck.__mro__)
# (Duck, Animal, Flyable, Swimmable, object)
```

Python использует алгоритм C3 Linearization.

## Проверка наследования

```python
dog = Dog("Рекс", 5, "Овчарка")

# isinstance — объект является экземпляром класса?
isinstance(dog, Dog)     # True
isinstance(dog, Animal)  # True (родитель тоже)

# issubclass — класс наследует от другого?
issubclass(Dog, Animal)  # True
issubclass(Animal, Dog)  # False
```

## Абстрактные классы (ABC)

Заставляют дочерние классы реализовать определённые методы:

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self):
        pass

    @abstractmethod
    def perimeter(self):
        pass

class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def area(self):
        return self.width * self.height

    def perimeter(self):
        return 2 * (self.width + self.height)

# Shape()  # TypeError! Нельзя создать экземпляр ABC
rect = Rectangle(5, 3)
rect.area()  # 15
```

## Резюме

| Концепция | Описание |
|-----------|----------|
| `class Child(Parent)` | Наследование от Parent |
| `super()` | Вызов метода родителя |
| Множественное наследование | `class C(A, B)` |
| MRO | Порядок разрешения методов |
| `isinstance(obj, Class)` | Проверка типа объекта |
| `issubclass(A, B)` | Проверка наследования классов |
| `ABC`, `@abstractmethod` | Абстрактные классы и методы |

## Q&A

### В чём разница между `ABC` и `metaclass=ABCMeta`?

**Функционально — никакой.** `ABC` — это просто класс-помощник:

```python
# Внутри модуля abc
class ABC(metaclass=ABCMeta):
    pass
```

| | `class Shape(ABC)` | `class Shape(metaclass=ABCMeta)` |
|---|---|---|
| Читаемость | Проще | Более явно |
| Когда использовать | В 99% случаев | При комбинации с другим метаклассом |

```python
# ABCMeta нужен для комбинации метаклассов
class CombinedMeta(ABCMeta, MyMeta):
    pass

class Shape(metaclass=CombinedMeta):
    ...
```

### Почему в `@abstractmethod` используется `pass`, а не `raise NotImplementedError`?

**`pass` достаточно**, потому что `@abstractmethod` уже гарантирует ошибку при создании объекта:

```python
shape = Shape()  # TypeError: Can't instantiate abstract class Shape
```

Тело метода никогда не выполнится — `pass` просто заглушка для синтаксиса.

**`raise NotImplementedError`** — опционально, как страховка при вызове через `super()`.

### В чём разница между `NotImplemented` и `NotImplementedError`?

| | `NotImplemented` | `NotImplementedError` |
|---|---|---|
| Тип | Singleton (не исключение!) | Исключение |
| Использование | `return NotImplemented` | `raise NotImplementedError` |
| Назначение | Для бинарных операций (`__eq__`, `__add__`) | Для нереализованных методов |

```python
# NotImplemented — для бинарных операций
class Vector:
    def __eq__(self, other):
        if not isinstance(other, Vector):
            return NotImplemented  # Python попробует other.__eq__(self)
        return self.x == other.x

# NotImplementedError — для методов без @abstractmethod
class Shape:
    def area(self):
        raise NotImplementedError("Subclasses must implement area()")
```

**Важно:** `raise NotImplemented` — ошибка! Это не исключение.
