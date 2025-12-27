# Four Pillars of OOP

## Четыре столпа объектно-ориентированного программирования

Объектно-ориентированное программирование (ООП) основывается на четырёх фундаментальных принципах, которые определяют структуру и организацию кода. Эти принципы помогают создавать гибкие, масштабируемые и поддерживаемые системы.

---

## 1. Инкапсуляция (Encapsulation)

### Что это такое

Инкапсуляция — это механизм объединения данных и методов, работающих с этими данными, в единую сущность (класс), а также сокрытие внутренней реализации от внешнего мира.

### Основные идеи

- **Сокрытие данных**: внутреннее состояние объекта скрыто от внешнего доступа
- **Контролируемый доступ**: взаимодействие с объектом происходит через публичные методы (интерфейс)
- **Защита целостности**: объект сам контролирует корректность своего состояния

### Пример на Python

```python
class BankAccount:
    def __init__(self, owner: str, initial_balance: float = 0):
        self._owner = owner           # protected
        self.__balance = initial_balance  # private

    @property
    def balance(self) -> float:
        """Публичный доступ к балансу только для чтения."""
        return self.__balance

    @property
    def owner(self) -> str:
        return self._owner

    def deposit(self, amount: float) -> None:
        """Контролируемое пополнение счёта."""
        if amount <= 0:
            raise ValueError("Сумма пополнения должна быть положительной")
        self.__balance += amount

    def withdraw(self, amount: float) -> bool:
        """Контролируемое снятие со счёта."""
        if amount <= 0:
            raise ValueError("Сумма снятия должна быть положительной")
        if amount > self.__balance:
            return False  # недостаточно средств
        self.__balance -= amount
        return True


# Использование
account = BankAccount("Иван", 1000)
account.deposit(500)
print(account.balance)  # 1500

# account.__balance = -1000  # Ошибка! Прямой доступ запрещён
```

### Преимущества инкапсуляции

1. **Модульность**: изменения внутренней реализации не влияют на внешний код
2. **Безопасность**: защита от некорректного использования
3. **Простота использования**: пользователь работает с простым интерфейсом

---

## 2. Наследование (Inheritance)

### Что это такое

Наследование — это механизм, позволяющий создавать новые классы на основе существующих, наследуя их свойства и поведение.

### Виды наследования

- **Одиночное наследование**: класс наследует от одного родителя
- **Множественное наследование**: класс наследует от нескольких родителей (Python поддерживает)
- **Многоуровневое наследование**: цепочка наследования (A -> B -> C)

### Пример на Python

```python
from abc import ABC, abstractmethod
from datetime import datetime


class Employee(ABC):
    """Базовый класс для всех сотрудников."""

    def __init__(self, name: str, hire_date: datetime):
        self.name = name
        self.hire_date = hire_date

    @abstractmethod
    def calculate_salary(self) -> float:
        """Расчёт зарплаты — у каждого типа сотрудника своя логика."""
        pass

    def years_employed(self) -> int:
        """Общий метод для всех сотрудников."""
        return (datetime.now() - self.hire_date).days // 365


class FullTimeEmployee(Employee):
    """Штатный сотрудник с фиксированной зарплатой."""

    def __init__(self, name: str, hire_date: datetime, monthly_salary: float):
        super().__init__(name, hire_date)
        self.monthly_salary = monthly_salary

    def calculate_salary(self) -> float:
        return self.monthly_salary


class HourlyEmployee(Employee):
    """Почасовой сотрудник."""

    def __init__(self, name: str, hire_date: datetime,
                 hourly_rate: float, hours_worked: float):
        super().__init__(name, hire_date)
        self.hourly_rate = hourly_rate
        self.hours_worked = hours_worked

    def calculate_salary(self) -> float:
        overtime = max(0, self.hours_worked - 160)
        regular = min(self.hours_worked, 160)
        return (regular * self.hourly_rate +
                overtime * self.hourly_rate * 1.5)


# Использование
emp1 = FullTimeEmployee("Анна", datetime(2020, 1, 15), 80000)
emp2 = HourlyEmployee("Борис", datetime(2021, 6, 1), 500, 180)

print(f"{emp1.name}: {emp1.calculate_salary()}")  # Анна: 80000
print(f"{emp2.name}: {emp2.calculate_salary()}")  # Борис: 95000
```

### Проблемы наследования

1. **Жёсткая связь**: изменения в родителе влияют на всех потомков
2. **Хрупкий базовый класс**: сложно предвидеть все последствия изменений
3. **Нарушение инкапсуляции**: наследник может зависеть от деталей реализации

---

## 3. Полиморфизм (Polymorphism)

### Что это такое

Полиморфизм — это способность объектов разных классов отвечать на одинаковые сообщения (вызовы методов) по-разному. Это позволяет писать код, который работает с объектами разных типов единообразно.

### Виды полиморфизма

1. **Подтиповый (наследование)**: объекты подклассов можно использовать там, где ожидается родительский класс
2. **Параметрический (generics)**: функции/классы, работающие с разными типами данных
3. **Ad-hoc (перегрузка)**: разные реализации для разных типов аргументов

### Пример на Python

```python
from abc import ABC, abstractmethod
from typing import Protocol


# Подтиповый полиморфизм через абстрактный класс
class Shape(ABC):
    @abstractmethod
    def area(self) -> float:
        pass

    @abstractmethod
    def perimeter(self) -> float:
        pass


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
        import math
        return math.pi * self.radius ** 2

    def perimeter(self) -> float:
        import math
        return 2 * math.pi * self.radius


# Полиморфизм через протоколы (структурная типизация)
class Drawable(Protocol):
    def draw(self) -> str:
        ...


class Button:
    def draw(self) -> str:
        return "Drawing a button"


class TextBox:
    def draw(self) -> str:
        return "Drawing a text box"


def render(element: Drawable) -> None:
    print(element.draw())


# Полиморфное использование
def total_area(shapes: list[Shape]) -> float:
    """Работает с любыми фигурами единообразно."""
    return sum(shape.area() for shape in shapes)


shapes = [Rectangle(10, 5), Circle(7), Rectangle(3, 3)]
print(f"Общая площадь: {total_area(shapes):.2f}")

# Структурный полиморфизм
render(Button())   # Drawing a button
render(TextBox())  # Drawing a text box
```

### Duck Typing в Python

```python
class Duck:
    def quack(self):
        return "Кря-кря!"

class Person:
    def quack(self):
        return "Человек имитирует утку"

def make_it_quack(thing):
    """Не важно, что это — главное, чтобы умело крякать."""
    print(thing.quack())

make_it_quack(Duck())    # Кря-кря!
make_it_quack(Person())  # Человек имитирует утку
```

---

## 4. Абстракция (Abstraction)

### Что это такое

Абстракция — это выделение существенных характеристик объекта и игнорирование несущественных деталей. Это позволяет создавать упрощённые модели сложных систем.

### Уровни абстракции

1. **Абстрактные классы**: определяют общий интерфейс и частичную реализацию
2. **Интерфейсы/Протоколы**: определяют только контракт без реализации
3. **Высокоуровневые модули**: скрывают сложность низкоуровневых операций

### Пример на Python

```python
from abc import ABC, abstractmethod
from typing import Iterator


# Абстракция хранилища данных
class UserRepository(ABC):
    """Абстрактный репозиторий — скрывает детали хранения."""

    @abstractmethod
    def get_by_id(self, user_id: int) -> dict | None:
        pass

    @abstractmethod
    def save(self, user: dict) -> int:
        pass

    @abstractmethod
    def delete(self, user_id: int) -> bool:
        pass

    @abstractmethod
    def find_all(self) -> Iterator[dict]:
        pass


# Конкретная реализация — PostgreSQL
class PostgresUserRepository(UserRepository):
    def __init__(self, connection_string: str):
        self.connection_string = connection_string
        # В реальности здесь было бы подключение к БД

    def get_by_id(self, user_id: int) -> dict | None:
        # SQL запрос к PostgreSQL
        print(f"SELECT * FROM users WHERE id = {user_id}")
        return {"id": user_id, "name": "User"}

    def save(self, user: dict) -> int:
        print(f"INSERT INTO users ... VALUES ...")
        return 1

    def delete(self, user_id: int) -> bool:
        print(f"DELETE FROM users WHERE id = {user_id}")
        return True

    def find_all(self) -> Iterator[dict]:
        print("SELECT * FROM users")
        yield from [{"id": 1}, {"id": 2}]


# Конкретная реализация — In-Memory (для тестов)
class InMemoryUserRepository(UserRepository):
    def __init__(self):
        self._storage: dict[int, dict] = {}
        self._next_id = 1

    def get_by_id(self, user_id: int) -> dict | None:
        return self._storage.get(user_id)

    def save(self, user: dict) -> int:
        user_id = self._next_id
        user["id"] = user_id
        self._storage[user_id] = user
        self._next_id += 1
        return user_id

    def delete(self, user_id: int) -> bool:
        if user_id in self._storage:
            del self._storage[user_id]
            return True
        return False

    def find_all(self) -> Iterator[dict]:
        yield from self._storage.values()


# Бизнес-логика работает с абстракцией
class UserService:
    def __init__(self, repository: UserRepository):
        self.repository = repository  # Dependency Injection

    def get_user(self, user_id: int) -> dict | None:
        return self.repository.get_by_id(user_id)

    def create_user(self, name: str, email: str) -> int:
        return self.repository.save({"name": name, "email": email})


# Легко переключаться между реализациями
service = UserService(InMemoryUserRepository())  # для тестов
# service = UserService(PostgresUserRepository("..."))  # для продакшена
```

---

## Взаимосвязь принципов

```
┌─────────────────────────────────────────────────────────────────┐
│                         АБСТРАКЦИЯ                              │
│        (определяет ЧТО должен делать объект)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐    ┌─────────────────┐                    │
│  │  ИНКАПСУЛЯЦИЯ   │    │  НАСЛЕДОВАНИЕ   │                    │
│  │  (скрывает КАК) │    │  (переиспользу- │                    │
│  │                 │    │  ет реализацию) │                    │
│  └────────┬────────┘    └────────┬────────┘                    │
│           │                      │                              │
│           └──────────┬───────────┘                              │
│                      │                                          │
│              ┌───────▼───────┐                                  │
│              │  ПОЛИМОРФИЗМ  │                                  │
│              │  (позволяет   │                                  │
│              │  работать     │                                  │
│              │  единообразно)│                                  │
│              └───────────────┘                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Best Practices

### Инкапсуляция
- Делайте поля приватными по умолчанию
- Предоставляйте минимально необходимый публичный интерфейс
- Используйте свойства (properties) для контролируемого доступа

### Наследование
- Предпочитайте композицию наследованию
- Наследуйте только когда есть отношение "является" (is-a)
- Не наследуйте ради переиспользования кода — для этого есть композиция

### Полиморфизм
- Программируйте на уровне интерфейсов, а не реализаций
- Используйте протоколы для слабой связанности
- Избегайте проверок типов (isinstance) — это признак нарушения полиморфизма

### Абстракция
- Выделяйте правильный уровень абстракции для вашей задачи
- Не абстрагируйте преждевременно
- Следуйте принципу Dependency Inversion: зависьте от абстракций

---

## Типичные ошибки

1. **Нарушение инкапсуляции через геттеры/сеттеры для всего**
   - Если у вас геттер и сеттер для каждого поля — инкапсуляции нет

2. **Глубокие иерархии наследования**
   - Более 2-3 уровней — признак проблемы в дизайне

3. **Использование наследования для переиспользования кода**
   - `class Stack(ArrayList)` — неправильно! Stack не является ArrayList

4. **Абстракции, протекающие деталями реализации**
   - Когда для использования абстракции нужно знать её реализацию

5. **Проверки типов вместо полиморфизма**
   ```python
   # Плохо
   if isinstance(shape, Circle):
       do_circle_stuff()
   elif isinstance(shape, Rectangle):
       do_rectangle_stuff()

   # Хорошо
   shape.do_stuff()  # полиморфный вызов
   ```
