# SOLID - Принципы объектно-ориентированного проектирования

## Введение

SOLID — это акроним, объединяющий пять основных принципов объектно-ориентированного программирования и проектирования. Эти принципы были сформулированы Робертом Мартином (Uncle Bob) и помогают создавать гибкий, поддерживаемый и расширяемый код.

---

## S — Single Responsibility Principle (Принцип единственной ответственности)

### Определение

**Класс должен иметь только одну причину для изменения** — то есть отвечать только за одну задачу или функциональность.

### Почему это важно

- Упрощает понимание кода
- Облегчает тестирование
- Уменьшает связность между компонентами
- Снижает риск побочных эффектов при изменениях

### Плохой пример

```python
class User:
    def __init__(self, name: str, email: str):
        self.name = name
        self.email = email

    def save_to_database(self):
        # Логика сохранения в БД
        db.execute(f"INSERT INTO users (name, email) VALUES ('{self.name}', '{self.email}')")

    def send_welcome_email(self):
        # Логика отправки email
        email_service.send(self.email, "Welcome!", "Hello, " + self.name)

    def generate_report(self):
        # Логика генерации отчета
        return f"User Report: {self.name}, {self.email}"
```

Здесь класс `User` имеет три ответственности: хранение данных, работа с БД, отправка email и генерация отчетов.

### Хороший пример

```python
class User:
    """Только хранение данных пользователя"""
    def __init__(self, name: str, email: str):
        self.name = name
        self.email = email


class UserRepository:
    """Ответственность за работу с БД"""
    def save(self, user: User):
        db.execute(f"INSERT INTO users (name, email) VALUES ('{user.name}', '{user.email}')")

    def find_by_email(self, email: str) -> User:
        # Поиск пользователя
        pass


class EmailService:
    """Ответственность за отправку email"""
    def send_welcome_email(self, user: User):
        email_service.send(user.email, "Welcome!", f"Hello, {user.name}")


class UserReportGenerator:
    """Ответственность за генерацию отчетов"""
    def generate(self, user: User) -> str:
        return f"User Report: {user.name}, {user.email}"
```

---

## O — Open/Closed Principle (Принцип открытости/закрытости)

### Определение

**Программные сущности должны быть открыты для расширения, но закрыты для модификации.** То есть поведение можно расширить без изменения существующего кода.

### Почему это важно

- Защищает работающий код от изменений
- Позволяет добавлять новую функциональность без рисков
- Способствует использованию абстракций

### Плохой пример

```python
class DiscountCalculator:
    def calculate(self, customer_type: str, amount: float) -> float:
        if customer_type == "regular":
            return amount * 0.1
        elif customer_type == "premium":
            return amount * 0.2
        elif customer_type == "vip":
            return amount * 0.3
        # Каждый новый тип клиента требует изменения этого метода
        return 0
```

### Хороший пример

```python
from abc import ABC, abstractmethod


class DiscountStrategy(ABC):
    @abstractmethod
    def calculate(self, amount: float) -> float:
        pass


class RegularDiscount(DiscountStrategy):
    def calculate(self, amount: float) -> float:
        return amount * 0.1


class PremiumDiscount(DiscountStrategy):
    def calculate(self, amount: float) -> float:
        return amount * 0.2


class VIPDiscount(DiscountStrategy):
    def calculate(self, amount: float) -> float:
        return amount * 0.3


# Добавление нового типа скидки не требует изменения существующего кода
class StudentDiscount(DiscountStrategy):
    def calculate(self, amount: float) -> float:
        return amount * 0.15


class DiscountCalculator:
    def __init__(self, strategy: DiscountStrategy):
        self.strategy = strategy

    def calculate(self, amount: float) -> float:
        return self.strategy.calculate(amount)
```

---

## L — Liskov Substitution Principle (Принцип подстановки Барбары Лисков)

### Определение

**Объекты подклассов должны быть заменяемы объектами базового класса без нарушения корректности программы.** Подкласс не должен ослаблять предусловия или усиливать постусловия методов базового класса.

### Почему это важно

- Гарантирует корректность полиморфизма
- Позволяет безопасно использовать наследование
- Обеспечивает предсказуемое поведение

### Плохой пример

```python
class Rectangle:
    def __init__(self, width: int, height: int):
        self._width = width
        self._height = height

    @property
    def width(self) -> int:
        return self._width

    @width.setter
    def width(self, value: int):
        self._width = value

    @property
    def height(self) -> int:
        return self._height

    @height.setter
    def height(self, value: int):
        self._height = value

    def area(self) -> int:
        return self._width * self._height


class Square(Rectangle):
    def __init__(self, side: int):
        super().__init__(side, side)

    @Rectangle.width.setter
    def width(self, value: int):
        # Нарушение LSP: изменение ширины меняет и высоту
        self._width = value
        self._height = value

    @Rectangle.height.setter
    def height(self, value: int):
        self._width = value
        self._height = value


# Этот код работает неожиданно
def process_rectangle(rect: Rectangle):
    rect.width = 5
    rect.height = 10
    # Ожидаем area = 50
    assert rect.area() == 50  # Падает для Square!
```

### Хороший пример

```python
from abc import ABC, abstractmethod


class Shape(ABC):
    @abstractmethod
    def area(self) -> int:
        pass


class Rectangle(Shape):
    def __init__(self, width: int, height: int):
        self._width = width
        self._height = height

    def area(self) -> int:
        return self._width * self._height


class Square(Shape):
    def __init__(self, side: int):
        self._side = side

    def area(self) -> int:
        return self._side * self._side


# Теперь классы независимы и не нарушают LSP
```

---

## I — Interface Segregation Principle (Принцип разделения интерфейсов)

### Определение

**Клиенты не должны зависеть от интерфейсов, которые они не используют.** Лучше много специализированных интерфейсов, чем один универсальный.

### Почему это важно

- Уменьшает связность
- Упрощает реализацию интерфейсов
- Делает код более гибким

### Плохой пример

```python
from abc import ABC, abstractmethod


class Worker(ABC):
    @abstractmethod
    def work(self):
        pass

    @abstractmethod
    def eat(self):
        pass

    @abstractmethod
    def sleep(self):
        pass


class Robot(Worker):
    def work(self):
        print("Robot working")

    def eat(self):
        # Роботы не едят!
        raise NotImplementedError("Robots don't eat")

    def sleep(self):
        # Роботы не спят!
        raise NotImplementedError("Robots don't sleep")
```

### Хороший пример

```python
from abc import ABC, abstractmethod


class Workable(ABC):
    @abstractmethod
    def work(self):
        pass


class Eatable(ABC):
    @abstractmethod
    def eat(self):
        pass


class Sleepable(ABC):
    @abstractmethod
    def sleep(self):
        pass


class Human(Workable, Eatable, Sleepable):
    def work(self):
        print("Human working")

    def eat(self):
        print("Human eating")

    def sleep(self):
        print("Human sleeping")


class Robot(Workable):
    def work(self):
        print("Robot working")
```

---

## D — Dependency Inversion Principle (Принцип инверсии зависимостей)

### Определение

1. **Модули верхнего уровня не должны зависеть от модулей нижнего уровня. Оба должны зависеть от абстракций.**
2. **Абстракции не должны зависеть от деталей. Детали должны зависеть от абстракций.**

### Почему это важно

- Уменьшает связность между модулями
- Упрощает тестирование (легко подменять зависимости)
- Делает систему более гибкой

### Плохой пример

```python
class MySQLDatabase:
    def save(self, data: dict):
        print(f"Saving to MySQL: {data}")


class UserService:
    def __init__(self):
        # Жесткая зависимость от конкретной реализации
        self.database = MySQLDatabase()

    def create_user(self, name: str):
        self.database.save({"name": name})
```

### Хороший пример

```python
from abc import ABC, abstractmethod


class Database(ABC):
    @abstractmethod
    def save(self, data: dict):
        pass


class MySQLDatabase(Database):
    def save(self, data: dict):
        print(f"Saving to MySQL: {data}")


class PostgreSQLDatabase(Database):
    def save(self, data: dict):
        print(f"Saving to PostgreSQL: {data}")


class MongoDBDatabase(Database):
    def save(self, data: dict):
        print(f"Saving to MongoDB: {data}")


class UserService:
    def __init__(self, database: Database):
        # Зависимость от абстракции
        self.database = database

    def create_user(self, name: str):
        self.database.save({"name": name})


# Использование
mysql_service = UserService(MySQLDatabase())
postgres_service = UserService(PostgreSQLDatabase())
mongo_service = UserService(MongoDBDatabase())
```

---

## Best Practices

1. **Не применяйте SOLID везде бездумно** — принципы нужны для решения проблем, а не для создания сложности
2. **Начинайте с простого кода** — рефакторинг по SOLID делайте когда появляется реальная необходимость
3. **Используйте Dependency Injection** — это помогает соблюдать DIP
4. **Пишите тесты** — они помогут понять, насколько ваш код соответствует SOLID

## Типичные ошибки

1. **Чрезмерное дробление** — создание слишком мелких классов усложняет понимание кода
2. **Преждевременная абстракция** — создание интерфейсов без реальной необходимости
3. **Нарушение LSP при наследовании** — особенно при переопределении поведения родителя
4. **Игнорирование контекста** — SOLID не всегда уместен (простые скрипты, прототипы)

---

## Связь принципов SOLID

```
SRP ← помогает → ISP (маленькие классы = маленькие интерфейсы)
OCP ← требует → DIP (расширение через абстракции)
LSP ← обеспечивает → OCP (корректное наследование для полиморфизма)
ISP ← упрощает → LSP (меньше методов = меньше рисков нарушения)
```

Все принципы взаимосвязаны и дополняют друг друга для создания качественной архитектуры.
