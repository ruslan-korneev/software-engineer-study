# Classes (Классы)

## Что такое класс?

**Класс** — это шаблон (чертёж) для создания объектов. Определяет:
- **Атрибуты** — данные объекта
- **Методы** — функции объекта

## Базовый синтаксис

```python
class Dog:
    # Атрибут класса (общий для всех экземпляров)
    species = "Canis familiaris"

    # Конструктор (инициализатор)
    def __init__(self, name, age):
        # Атрибуты экземпляра (уникальны для каждого объекта)
        self.name = name
        self.age = age

    # Метод экземпляра
    def bark(self):
        return f"{self.name} говорит: Гав!"
```

## Создание объектов

```python
buddy = Dog("Buddy", 3)
max_dog = Dog("Max", 5)

print(buddy.name)      # Buddy
print(buddy.bark())    # Buddy говорит: Гав!
print(buddy.species)   # Canis familiaris
```

## Ключевые концепции

### 1. `self` — ссылка на текущий экземпляр

```python
class Counter:
    def __init__(self):
        self.count = 0

    def increment(self):
        self.count += 1
        return self.count

c1 = Counter()
c2 = Counter()
c1.increment()  # 1
c1.increment()  # 2
c2.increment()  # 1 (отдельный счётчик)
```

### 2. Атрибуты класса vs экземпляра

```python
class Employee:
    company = "TechCorp"  # Атрибут класса — один на всех

    def __init__(self, name):
        self.name = name  # Атрибут экземпляра — свой у каждого

Employee.company = "NewCorp"  # Меняется для ВСЕХ экземпляров
```

### 3. Приватные атрибуты (конвенция)

| Синтаксис | Название | Описание |
|-----------|----------|----------|
| `name` | Public | Доступен везде |
| `_name` | Protected | "Не трогай" (конвенция) |
| `__name` | Private | Name mangling → `_ClassName__name` |

```python
class BankAccount:
    def __init__(self, balance):
        self._balance = balance      # Protected
        self.__secret = "hidden"     # Private

acc = BankAccount(1000)
print(acc._balance)               # Работает (но не рекомендуется)
print(acc._BankAccount__secret)   # "hidden" (name mangling)
```

### 4. Property — контроль доступа

```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):
        """Getter"""
        return self._radius

    @radius.setter
    def radius(self, value):
        """Setter с валидацией"""
        if value < 0:
            raise ValueError("Радиус не может быть отрицательным")
        self._radius = value

    @property
    def area(self):
        """Вычисляемое свойство (только для чтения)"""
        return 3.14159 * self._radius ** 2

c = Circle(5)
print(c.radius)  # 5 (getter)
c.radius = 10    # setter
print(c.area)    # 314.159
```

## Резюме

- `class` создаёт новый тип данных
- `__init__` — конструктор, вызывается при создании объекта
- `self` — ссылка на текущий экземпляр
- Атрибуты класса — общие, атрибуты экземпляра — индивидуальные
- `_` и `__` — конвенции для "приватности"
- `@property` — геттеры/сеттеры в Pythonic стиле
