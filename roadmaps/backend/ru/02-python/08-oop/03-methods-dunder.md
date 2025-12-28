# Methods and Dunder (Методы и магические методы)

[prev: 02-inheritance](./02-inheritance.md) | [next: 04-abc](./04-abc.md)
---

## Типы методов

### 1. Instance Methods (Методы экземпляра)

Обычные методы, принимают `self`:

```python
class Dog:
    def bark(self):
        return f"{self.name}: Гав!"
```

### 2. Class Methods (`@classmethod`)

Принимают класс (`cls`). Используются для альтернативных конструкторов:

```python
class Date:
    def __init__(self, year, month, day):
        self.year = year
        self.month = month
        self.day = day

    @classmethod
    def from_string(cls, date_string):
        year, month, day = map(int, date_string.split('-'))
        return cls(year, month, day)

    @classmethod
    def today(cls):
        import datetime
        t = datetime.date.today()
        return cls(t.year, t.month, t.day)

date1 = Date(2024, 12, 25)
date2 = Date.from_string("2024-12-25")
date3 = Date.today()
```

### 3. Static Methods (`@staticmethod`)

Не принимают ни `self`, ни `cls`:

```python
class Math:
    @staticmethod
    def add(a, b):
        return a + b

Math.add(2, 3)  # 5
```

### Сравнение

| Тип | Декоратор | Первый аргумент | Доступ к |
|-----|-----------|-----------------|----------|
| Instance | — | `self` | Экземпляру и классу |
| Class | `@classmethod` | `cls` | Только к классу |
| Static | `@staticmethod` | — | Ни к чему |

---

## Dunder Methods (Магические методы)

**Dunder** = **D**ouble **UNDER**score (`__method__`)

### 1. Создание и инициализация

```python
class User:
    def __new__(cls, *args, **kwargs):
        """Создаёт объект (редко переопределяют)"""
        return super().__new__(cls)

    def __init__(self, name):
        """Инициализирует объект"""
        self.name = name

    def __del__(self):
        """Вызывается при удалении"""
        print(f"Goodbye, {self.name}")
```

### 2. Строковое представление

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __str__(self):
        """Для пользователя: print(), str()"""
        return f"Точка ({self.x}, {self.y})"

    def __repr__(self):
        """Для разработчика: отладка, repr()"""
        return f"Point({self.x}, {self.y})"

p = Point(3, 4)
print(p)  # Точка (3, 4)
repr(p)   # Point(3, 4)
```

### 3. Сравнение

| Метод | Оператор |
|-------|----------|
| `__eq__` | `==` |
| `__ne__` | `!=` |
| `__lt__` | `<` |
| `__le__` | `<=` |
| `__gt__` | `>` |
| `__ge__` | `>=` |

```python
class Money:
    def __init__(self, amount):
        self.amount = amount

    def __eq__(self, other):
        return self.amount == other.amount

    def __lt__(self, other):
        return self.amount < other.amount
```

### 4. Арифметические операции

| Метод | Оператор |
|-------|----------|
| `__add__` | `+` |
| `__sub__` | `-` |
| `__mul__` | `*` |
| `__truediv__` | `/` |
| `__floordiv__` | `//` |
| `__mod__` | `%` |
| `__pow__` | `**` |

```python
class Vector:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __add__(self, other):
        return Vector(self.x + other.x, self.y + other.y)

    def __mul__(self, scalar):
        return Vector(self.x * scalar, self.y * scalar)

v1 = Vector(1, 2)
v2 = Vector(3, 4)
v1 + v2  # Vector(4, 6)
v1 * 3   # Vector(3, 6)
```

### 5. Контейнерные методы

| Метод | Использование |
|-------|---------------|
| `__len__` | `len(obj)` |
| `__getitem__` | `obj[key]` |
| `__setitem__` | `obj[key] = value` |
| `__delitem__` | `del obj[key]` |
| `__contains__` | `item in obj` |
| `__iter__` | `for x in obj` |

```python
class Playlist:
    def __init__(self):
        self._songs = []

    def __len__(self):
        return len(self._songs)

    def __getitem__(self, index):
        return self._songs[index]

    def __contains__(self, item):
        return item in self._songs

    def __iter__(self):
        return iter(self._songs)
```

### 6. Контекстный менеджер

```python
class FileManager:
    def __init__(self, filename, mode):
        self.filename = filename
        self.mode = mode

    def __enter__(self):
        """Вход в with"""
        self.file = open(self.filename, self.mode)
        return self.file

    def __exit__(self, exc_type, exc_val, exc_tb):
        """Выход из with"""
        self.file.close()
        return False  # Не подавлять исключения

with FileManager("test.txt", "w") as f:
    f.write("Hello!")
```

### 7. Callable объекты

```python
class Multiplier:
    def __init__(self, factor):
        self.factor = factor

    def __call__(self, value):
        """Объект можно вызвать как функцию"""
        return value * self.factor

double = Multiplier(2)
double(5)  # 10
```

### 8. Хеширование

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __hash__(self):
        """Для использования в set/dict"""
        return hash((self.x, self.y))

    def __eq__(self, other):
        return self.x == other.x and self.y == other.y

points = {Point(1, 2), Point(3, 4)}
```

## Резюме

| Категория | Методы |
|-----------|--------|
| Жизненный цикл | `__new__`, `__init__`, `__del__` |
| Строки | `__str__`, `__repr__` |
| Сравнение | `__eq__`, `__lt__`, `__gt__`, ... |
| Арифметика | `__add__`, `__sub__`, `__mul__`, ... |
| Контейнер | `__len__`, `__getitem__`, `__iter__`, ... |
| Контекст | `__enter__`, `__exit__` |
| Вызов | `__call__` |
| Хеш | `__hash__` |

---
[prev: 02-inheritance](./02-inheritance.md) | [next: 04-abc](./04-abc.md)