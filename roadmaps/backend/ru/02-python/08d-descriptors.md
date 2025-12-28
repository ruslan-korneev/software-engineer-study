# Дескрипторы (Descriptors)

[prev: 08c-dataclasses](./08c-dataclasses.md) | [next: 08e-metaclasses](./08e-metaclasses.md)
---

## Что такое дескриптор?

**Дескриптор** — объект, который определяет, как происходит доступ к атрибуту другого объекта. Реализует один или несколько методов протокола дескрипторов: `__get__`, `__set__`, `__delete__`.

```python
class Descriptor:
    def __get__(self, obj, objtype=None):
        """Вызывается при чтении атрибута"""
        pass

    def __set__(self, obj, value):
        """Вызывается при записи атрибута"""
        pass

    def __delete__(self, obj):
        """Вызывается при удалении атрибута"""
        pass
```

## Простой пример

```python
class VerboseDescriptor:
    def __get__(self, obj, objtype=None):
        print(f"__get__ called: obj={obj}, objtype={objtype}")
        return 42

    def __set__(self, obj, value):
        print(f"__set__ called: obj={obj}, value={value}")

    def __delete__(self, obj):
        print(f"__delete__ called: obj={obj}")

class MyClass:
    attr = VerboseDescriptor()

obj = MyClass()
obj.attr      # __get__ called: obj=<...>, objtype=<class 'MyClass'>
obj.attr = 10 # __set__ called: obj=<...>, value=10
del obj.attr  # __delete__ called: obj=<...>
```

## Data vs Non-Data Descriptors

### Data Descriptor (имеет __set__ или __delete__)

```python
class DataDescriptor:
    def __get__(self, obj, objtype=None):
        return "from descriptor"

    def __set__(self, obj, value):
        pass  # Перехватывает присваивание

class MyClass:
    attr = DataDescriptor()

obj = MyClass()
obj.__dict__['attr'] = "from instance"  # Добавляем в __dict__
print(obj.attr)  # "from descriptor" — дескриптор приоритетнее!
```

### Non-Data Descriptor (только __get__)

```python
class NonDataDescriptor:
    def __get__(self, obj, objtype=None):
        return "from descriptor"

class MyClass:
    attr = NonDataDescriptor()

obj = MyClass()
obj.__dict__['attr'] = "from instance"  # Добавляем в __dict__
print(obj.attr)  # "from instance" — __dict__ приоритетнее!
```

## Порядок поиска атрибутов

1. **Data descriptors** (класс и его родители)
2. **Instance `__dict__`**
3. **Non-data descriptors** (класс и его родители)
4. **`__getattr__`** (если определён)

```python
class DataDesc:
    def __get__(self, obj, objtype=None):
        return "data descriptor"
    def __set__(self, obj, value):
        pass

class NonDataDesc:
    def __get__(self, obj, objtype=None):
        return "non-data descriptor"

class MyClass:
    data = DataDesc()
    nondata = NonDataDesc()

obj = MyClass()
obj.__dict__['data'] = "instance data"
obj.__dict__['nondata'] = "instance nondata"

print(obj.data)     # "data descriptor" — приоритет дескриптора
print(obj.nondata)  # "instance nondata" — приоритет __dict__
```

## Практические примеры

### Валидирующий дескриптор

```python
class PositiveNumber:
    def __set_name__(self, owner, name):
        self.name = name
        self.private_name = f'__{name}'

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self.private_name, None)

    def __set__(self, obj, value):
        if value <= 0:
            raise ValueError(f"{self.name} must be positive, got {value}")
        setattr(obj, self.private_name, value)

class Product:
    price = PositiveNumber()
    quantity = PositiveNumber()

    def __init__(self, name: str, price: float, quantity: int):
        self.name = name
        self.price = price      # Валидация через дескриптор
        self.quantity = quantity

product = Product("Laptop", 999.99, 10)
product.price = -50  # ValueError: price must be positive
```

### Типизированный дескриптор

```python
class Typed:
    def __init__(self, expected_type):
        self.expected_type = expected_type

    def __set_name__(self, owner, name):
        self.name = name
        self.private_name = f'__{name}'

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self.private_name, None)

    def __set__(self, obj, value):
        if not isinstance(value, self.expected_type):
            raise TypeError(
                f"{self.name} must be {self.expected_type.__name__}, "
                f"got {type(value).__name__}"
            )
        setattr(obj, self.private_name, value)

class Person:
    name = Typed(str)
    age = Typed(int)

    def __init__(self, name: str, age: int):
        self.name = name
        self.age = age

person = Person("Alice", 30)
person.age = "thirty"  # TypeError: age must be int, got str
```

### Ленивое свойство (Lazy Property)

```python
class LazyProperty:
    def __init__(self, func):
        self.func = func
        self.name = func.__name__

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        # Вычисляем значение и кешируем в __dict__
        value = self.func(obj)
        obj.__dict__[self.name] = value  # Заменяем дескриптор в instance
        return value

class DataProcessor:
    def __init__(self, data):
        self.data = data

    @LazyProperty
    def processed_data(self):
        print("Processing data... (expensive operation)")
        return [x * 2 for x in self.data]

processor = DataProcessor([1, 2, 3])
print(processor.processed_data)  # Processing data... → [2, 4, 6]
print(processor.processed_data)  # [2, 4, 6] (кешировано, без "Processing")
```

### Логирующий дескриптор

```python
import logging

class LoggedAccess:
    def __set_name__(self, owner, name):
        self.name = name
        self.private_name = f'__{name}'

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        value = getattr(obj, self.private_name, None)
        logging.info(f"Accessing {self.name}: {value}")
        return value

    def __set__(self, obj, value):
        logging.info(f"Setting {self.name} to {value}")
        setattr(obj, self.private_name, value)

class Account:
    balance = LoggedAccess()

    def __init__(self, initial_balance):
        self.balance = initial_balance
```

## __set_name__ — получение имени атрибута

Python 3.6+ вызывает `__set_name__` при создании класса:

```python
class Descriptor:
    def __set_name__(self, owner, name):
        print(f"Descriptor assigned to {owner.__name__}.{name}")
        self.name = name

class MyClass:
    attr = Descriptor()  # Выводит: Descriptor assigned to MyClass.attr
```

## Как работает @property

`@property` — это дескриптор! Вот упрощённая реализация:

```python
class Property:
    def __init__(self, fget=None, fset=None, fdel=None):
        self.fget = fget
        self.fset = fset
        self.fdel = fdel

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        if self.fget is None:
            raise AttributeError("unreadable attribute")
        return self.fget(obj)

    def __set__(self, obj, value):
        if self.fset is None:
            raise AttributeError("can't set attribute")
        self.fset(obj, value)

    def __delete__(self, obj):
        if self.fdel is None:
            raise AttributeError("can't delete attribute")
        self.fdel(obj)

    def setter(self, fset):
        return Property(self.fget, fset, self.fdel)

    def deleter(self, fdel):
        return Property(self.fget, self.fset, fdel)

# Использование (как встроенный @property)
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @Property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("Radius must be positive")
        self._radius = value
```

## Как работают методы класса

Функции — это non-data дескрипторы:

```python
class Function:
    def __get__(self, obj, objtype=None):
        if obj is None:
            return self  # Доступ через класс
        # Возвращаем bound method
        return lambda *args, **kwargs: self(obj, *args, **kwargs)

# Поэтому методы автоматически получают self!
class MyClass:
    def method(self):
        pass

obj = MyClass()
obj.method  # Возвращает bound method, где self = obj
```

## Дескрипторы с хранением данных

### Хранение в instance __dict__

```python
class Descriptor:
    def __set_name__(self, owner, name):
        self.private_name = f'__{name}'

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self.private_name, None)

    def __set__(self, obj, value):
        setattr(obj, self.private_name, value)
```

### Хранение в дескрипторе (WeakKeyDictionary)

```python
from weakref import WeakKeyDictionary

class Descriptor:
    def __init__(self):
        self.values = WeakKeyDictionary()

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return self.values.get(obj)

    def __set__(self, obj, value):
        self.values[obj] = value
```

## Дескрипторы в слотах

```python
class Slots:
    __slots__ = ('x', 'y')  # Слоты — это дескрипторы!

obj = Slots()
obj.x = 10
print(type(Slots.x))  # <class 'member_descriptor'>
```

## Best Practices

### 1. Используйте __set_name__

```python
# ✅ Автоматическое получение имени
class Validator:
    def __set_name__(self, owner, name):
        self.name = name

# ❌ Ручное указание имени
class Validator:
    def __init__(self, name):
        self.name = name  # Легко ошибиться
```

### 2. Храните данные в instance, не в дескрипторе

```python
# ✅ Данные в instance (через __dict__ или private attr)
class GoodDescriptor:
    def __set_name__(self, owner, name):
        self.private = f'__{name}'

    def __get__(self, obj, objtype=None):
        return getattr(obj, self.private, None)

# ❌ Данные в дескрипторе (проблемы с GC)
class BadDescriptor:
    def __init__(self):
        self.values = {}  # Утечка памяти!
```

### 3. Обрабатывайте доступ через класс

```python
class Descriptor:
    def __get__(self, obj, objtype=None):
        if obj is None:
            return self  # Доступ через класс
        return ...  # Доступ через instance
```

## Q&A

**Q: Когда использовать дескрипторы вместо @property?**
A: Когда нужна переиспользуемая логика для нескольких атрибутов. Property — для одного атрибута.

**Q: Почему data descriptor приоритетнее instance __dict__?**
A: Это позволяет дескрипторам полностью контролировать доступ к атрибуту, включая перехват присваивания.

**Q: Можно ли использовать дескрипторы с __slots__?**
A: Да, но нужно хранить данные не в instance __dict__ (его нет при __slots__), а через WeakKeyDictionary.

**Q: Дескрипторы работают с classmethod/staticmethod?**
A: classmethod и staticmethod сами являются дескрипторами. Их можно комбинировать с другими дескрипторами.

---
[prev: 08c-dataclasses](./08c-dataclasses.md) | [next: 08e-metaclasses](./08e-metaclasses.md)