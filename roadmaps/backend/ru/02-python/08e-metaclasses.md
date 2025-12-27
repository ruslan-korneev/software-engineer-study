# Метаклассы (Metaclasses)

## Что такое метакласс?

**Метакласс** — это "класс класса". Если объект является экземпляром класса, то класс является экземпляром метакласса.

```python
class MyClass:
    pass

obj = MyClass()

type(obj)        # <class 'MyClass'>   — obj является экземпляром MyClass
type(MyClass)    # <class 'type'>      — MyClass является экземпляром type
type(type)       # <class 'type'>      — type является экземпляром самого себя
```

## type — встроенный метакласс

`type` — это метакласс по умолчанию для всех классов. `type` можно использовать для создания классов динамически:

```python
# Стандартное определение
class Dog:
    def bark(self):
        return "Woof!"

# Эквивалент через type
def bark(self):
    return "Woof!"

Dog = type('Dog', (), {'bark': bark})

# type(name, bases, dict)
# - name: имя класса
# - bases: кортеж базовых классов
# - dict: словарь атрибутов
```

### Создание классов через type

```python
# Простой класс
Point = type('Point', (), {'x': 0, 'y': 0})

# С методами
def __init__(self, x, y):
    self.x = x
    self.y = y

def __repr__(self):
    return f"Point({self.x}, {self.y})"

Point = type('Point', (), {
    '__init__': __init__,
    '__repr__': __repr__
})

p = Point(1, 2)
print(p)  # Point(1, 2)
```

## Создание своего метакласса

```python
class MyMeta(type):
    def __new__(mcs, name, bases, namespace):
        print(f"Creating class: {name}")
        return super().__new__(mcs, name, bases, namespace)

class MyClass(metaclass=MyMeta):
    pass
# Выводит: Creating class: MyClass
```

### __new__ vs __init__ в метаклассах

```python
class MyMeta(type):
    def __new__(mcs, name, bases, namespace):
        """Создание объекта класса"""
        print(f"__new__: Creating {name}")
        # Можно модифицировать namespace перед созданием
        cls = super().__new__(mcs, name, bases, namespace)
        return cls

    def __init__(cls, name, bases, namespace):
        """Инициализация объекта класса"""
        print(f"__init__: Initializing {name}")
        super().__init__(name, bases, namespace)

    def __call__(cls, *args, **kwargs):
        """Вызывается при создании экземпляра класса"""
        print(f"__call__: Creating instance of {cls.__name__}")
        instance = super().__call__(*args, **kwargs)
        return instance

class MyClass(metaclass=MyMeta):
    def __init__(self, value):
        self.value = value

# При определении класса:
# __new__: Creating MyClass
# __init__: Initializing MyClass

obj = MyClass(42)
# __call__: Creating instance of MyClass
```

## Практические примеры

### Singleton через метакласс

```python
class SingletonMeta(type):
    _instances = {}

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            instance = super().__call__(*args, **kwargs)
            cls._instances[cls] = instance
        return cls._instances[cls]

class Database(metaclass=SingletonMeta):
    def __init__(self):
        self.connected = True
        print("Database connected")

db1 = Database()  # Database connected
db2 = Database()  # Ничего не выводит
print(db1 is db2)  # True
```

### Регистрация классов

```python
class PluginMeta(type):
    plugins = {}

    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        if name != 'Plugin':  # Не регистрируем базовый класс
            mcs.plugins[name] = cls
        return cls

class Plugin(metaclass=PluginMeta):
    pass

class AuthPlugin(Plugin):
    pass

class CachePlugin(Plugin):
    pass

print(PluginMeta.plugins)
# {'AuthPlugin': <class 'AuthPlugin'>, 'CachePlugin': <class 'CachePlugin'>}
```

### Автоматическое добавление методов

```python
class AutoReprMeta(type):
    def __new__(mcs, name, bases, namespace):
        # Добавляем __repr__ если его нет
        if '__repr__' not in namespace:
            def __repr__(self):
                attrs = ', '.join(f'{k}={v!r}' for k, v in self.__dict__.items())
                return f'{name}({attrs})'
            namespace['__repr__'] = __repr__

        return super().__new__(mcs, name, bases, namespace)

class Person(metaclass=AutoReprMeta):
    def __init__(self, name, age):
        self.name = name
        self.age = age

p = Person("Alice", 30)
print(p)  # Person(name='Alice', age=30)
```

### Валидация при создании класса

```python
class ValidatedMeta(type):
    def __new__(mcs, name, bases, namespace):
        # Проверяем, что все публичные методы имеют docstring
        for key, value in namespace.items():
            if callable(value) and not key.startswith('_'):
                if not value.__doc__:
                    raise TypeError(
                        f"Method '{key}' in class '{name}' must have a docstring"
                    )

        return super().__new__(mcs, name, bases, namespace)

class MyClass(metaclass=ValidatedMeta):
    def method(self):
        """This method has a docstring."""
        pass

    def bad_method(self):  # TypeError!
        pass
```

### Интерфейс через метакласс

```python
class InterfaceMeta(type):
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)

        # Пропускаем сам Interface
        if name == 'Interface':
            return cls

        # Проверяем реализацию требуемых методов
        for base in bases:
            if hasattr(base, '_required_methods'):
                for method in base._required_methods:
                    if method not in namespace:
                        raise TypeError(
                            f"Class '{name}' must implement '{method}'"
                        )
        return cls

class Interface(metaclass=InterfaceMeta):
    _required_methods = []

class Drawable(Interface):
    _required_methods = ['draw']

class Circle(Drawable):
    def draw(self):  # OK
        print("Drawing circle")

class Square(Drawable):  # TypeError: must implement 'draw'
    pass
```

## __init_subclass__ — альтернатива метаклассам

Python 3.6+ предоставляет более простой способ для многих задач:

```python
class Plugin:
    plugins = {}

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        Plugin.plugins[cls.__name__] = cls

class AuthPlugin(Plugin):
    pass

class CachePlugin(Plugin):
    pass

print(Plugin.plugins)
# {'AuthPlugin': <class 'AuthPlugin'>, 'CachePlugin': <class 'CachePlugin'>}
```

### __init_subclass__ с параметрами

```python
class Validator:
    required_fields = []

    def __init_subclass__(cls, required_fields=None, **kwargs):
        super().__init_subclass__(**kwargs)
        if required_fields:
            cls.required_fields = required_fields

    def validate(self):
        for field in self.required_fields:
            if not hasattr(self, field):
                raise ValueError(f"Missing required field: {field}")

class User(Validator, required_fields=['name', 'email']):
    def __init__(self, name, email):
        self.name = name
        self.email = email

class Product(Validator, required_fields=['sku', 'price']):
    pass
```

## Метаклассы vs __init_subclass__

| Критерий | Метакласс | __init_subclass__ |
|----------|-----------|-------------------|
| Сложность | Высокая | Низкая |
| Контроль создания класса | Полный | Только после создания |
| Модификация namespace | Да | Нет |
| Контроль создания экземпляров | Да (__call__) | Нет |
| Конфликты | Возможны | Нет |
| Читаемость | Низкая | Высокая |

### Когда использовать метаклассы

```python
# 1. Нужно модифицировать namespace до создания класса
class ModifyingMeta(type):
    def __new__(mcs, name, bases, namespace):
        namespace['extra_attr'] = 42  # До создания класса
        return super().__new__(mcs, name, bases, namespace)

# 2. Нужно контролировать создание экземпляров
class ControlledMeta(type):
    def __call__(cls, *args, **kwargs):
        # Проверки перед созданием экземпляра
        return super().__call__(*args, **kwargs)

# 3. Нужна сложная логика наследования
# 4. Реализация ORM, фреймворков
```

### Когда использовать __init_subclass__

```python
# 1. Регистрация подклассов
# 2. Добавление атрибутов к подклассам
# 3. Валидация подклассов
# 4. Большинство случаев, где раньше нужен был метакласс
```

## Примеры из реального мира

### Django ORM (упрощённо)

```python
class ModelMeta(type):
    def __new__(mcs, name, bases, namespace):
        fields = {}
        for key, value in namespace.items():
            if isinstance(value, Field):
                fields[key] = value

        namespace['_fields'] = fields
        return super().__new__(mcs, name, bases, namespace)

class Field:
    def __init__(self, field_type):
        self.field_type = field_type

class Model(metaclass=ModelMeta):
    pass

class User(Model):
    name = Field(str)
    age = Field(int)

print(User._fields)  # {'name': Field, 'age': Field}
```

### Enum (упрощённо)

```python
class EnumMeta(type):
    def __new__(mcs, name, bases, namespace):
        members = {}
        for key, value in list(namespace.items()):
            if not key.startswith('_') and not callable(value):
                members[key] = value

        namespace['_members'] = members
        cls = super().__new__(mcs, name, bases, namespace)

        # Делаем класс итерируемым
        def __iter__(cls):
            return iter(cls._members.items())
        cls.__iter__ = classmethod(__iter__)

        return cls

class Color(metaclass=EnumMeta):
    RED = 1
    GREEN = 2
    BLUE = 3

for name, value in Color:
    print(f"{name} = {value}")
```

## Конфликты метаклассов

```python
class Meta1(type):
    pass

class Meta2(type):
    pass

class Base1(metaclass=Meta1):
    pass

class Base2(metaclass=Meta2):
    pass

# ❌ TypeError: metaclass conflict
class Child(Base1, Base2):
    pass

# ✅ Решение — общий метакласс
class CombinedMeta(Meta1, Meta2):
    pass

class Child(Base1, Base2, metaclass=CombinedMeta):
    pass
```

## Best Practices

### 1. Предпочитайте __init_subclass__

```python
# ✅ Просто и понятно
class Plugin:
    def __init_subclass__(cls, **kwargs):
        register(cls)

# ❌ Избыточно для простых задач
class PluginMeta(type):
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        register(cls)
        return cls
```

### 2. Документируйте метаклассы

```python
class RegistryMeta(type):
    """Метакласс для автоматической регистрации классов.

    Все подклассы автоматически добавляются в registry.
    Используется для plugin системы.
    """
```

### 3. Не злоупотребляйте

> "Metaclasses are deeper magic than 99% of users should ever worry about."
> — Tim Peters

```python
# ❌ Метакласс для добавления одного метода
# ✅ Используйте миксин или декоратор класса
```

## Q&A

**Q: Когда нужны метаклассы?**
A: Фреймворки (Django, SQLAlchemy), реализация паттернов (Singleton), генерация кода. Для обычного кода — почти никогда.

**Q: Чем метакласс отличается от декоратора класса?**
A: Метакласс работает при создании класса, контролирует весь процесс. Декоратор — после создания, модифицирует готовый класс.

**Q: Можно ли у класса быть два метакласса?**
A: Нет, но можно создать метакласс, наследующий от обоих.

**Q: __init_subclass__ вызывается для самого класса?**
A: Нет, только для подклассов. Для самого класса нужен метакласс.
