# Объектно-ориентированное программирование (ООП)

## Что такое ООП?

**Объектно-ориентированное программирование** — это парадигма программирования, основанная на концепции "объектов", которые содержат данные (атрибуты) и код (методы). ООП позволяет моделировать реальный мир и создавать модульные, повторно используемые программы.

## Четыре столпа ООП

### 1. Инкапсуляция (Encapsulation)

Объединение данных и методов работы с ними в одном объекте, скрытие внутренней реализации:

```python
class BankAccount:
    def __init__(self, owner, initial_balance=0):
        self._owner = owner           # Protected (соглашение)
        self.__balance = initial_balance  # Private (name mangling)

    @property
    def balance(self):
        """Геттер для баланса"""
        return self.__balance

    @property
    def owner(self):
        """Геттер для владельца"""
        return self._owner

    def deposit(self, amount):
        """Публичный метод для пополнения"""
        if amount <= 0:
            raise ValueError("Сумма должна быть положительной")
        self.__balance += amount
        return self.__balance

    def withdraw(self, amount):
        """Публичный метод для снятия"""
        if amount <= 0:
            raise ValueError("Сумма должна быть положительной")
        if amount > self.__balance:
            raise ValueError("Недостаточно средств")
        self.__balance -= amount
        return self.__balance

# Использование
account = BankAccount("Иван", 1000)
print(account.balance)  # 1000 (через property)
account.deposit(500)
# account.__balance = 999999  # Не работает напрямую!
```

### 2. Наследование (Inheritance)

Создание новых классов на основе существующих:

```python
class Animal:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def speak(self):
        raise NotImplementedError("Подклассы должны реализовать этот метод")

    def describe(self):
        return f"{self.name}, возраст: {self.age}"

class Dog(Animal):
    def __init__(self, name, age, breed):
        super().__init__(name, age)
        self.breed = breed

    def speak(self):
        return "Гав!"

    def fetch(self):
        return f"{self.name} принёс палку!"

class Cat(Animal):
    def __init__(self, name, age, indoor=True):
        super().__init__(name, age)
        self.indoor = indoor

    def speak(self):
        return "Мяу!"

    def purr(self):
        return f"{self.name} мурлычет"

# Множественное наследование
class Pet:
    def __init__(self):
        self.vaccinated = False

    def vaccinate(self):
        self.vaccinated = True
        return "Питомец вакцинирован"

class DomesticDog(Dog, Pet):
    def __init__(self, name, age, breed, owner):
        Dog.__init__(self, name, age, breed)
        Pet.__init__(self)
        self.owner = owner

# MRO (Method Resolution Order)
print(DomesticDog.__mro__)
```

### 3. Полиморфизм (Polymorphism)

Способность объектов разных классов отвечать на одинаковые сообщения:

```python
from abc import ABC, abstractmethod
from typing import List

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

# Полиморфное использование
def calculate_total_area(shapes: List[Shape]) -> float:
    """Работает с любыми фигурами"""
    return sum(shape.area() for shape in shapes)

shapes = [
    Rectangle(10, 5),
    Circle(7),
    Rectangle(3, 4)
]

print(f"Общая площадь: {calculate_total_area(shapes)}")
```

### 4. Абстракция (Abstraction)

Выделение существенных характеристик объекта, скрытие несущественных деталей:

```python
from abc import ABC, abstractmethod

class PaymentProcessor(ABC):
    """Абстрактный класс для обработки платежей"""

    @abstractmethod
    def process_payment(self, amount: float) -> bool:
        """Обработать платёж"""
        pass

    @abstractmethod
    def refund(self, transaction_id: str) -> bool:
        """Вернуть средства"""
        pass

    def validate_amount(self, amount: float) -> bool:
        """Общая валидация для всех процессоров"""
        return amount > 0

class CreditCardProcessor(PaymentProcessor):
    def __init__(self, merchant_id: str):
        self.merchant_id = merchant_id

    def process_payment(self, amount: float) -> bool:
        if not self.validate_amount(amount):
            return False
        # Логика обработки кредитной карты
        print(f"Обработка платежа на {amount} через кредитную карту")
        return True

    def refund(self, transaction_id: str) -> bool:
        print(f"Возврат по транзакции {transaction_id}")
        return True

class PayPalProcessor(PaymentProcessor):
    def __init__(self, api_key: str):
        self.api_key = api_key

    def process_payment(self, amount: float) -> bool:
        if not self.validate_amount(amount):
            return False
        # Логика PayPal
        print(f"Обработка платежа на {amount} через PayPal")
        return True

    def refund(self, transaction_id: str) -> bool:
        print(f"PayPal возврат по {transaction_id}")
        return True

# Клиентский код работает с абстракцией
def checkout(processor: PaymentProcessor, amount: float):
    if processor.process_payment(amount):
        print("Платёж успешен!")
    else:
        print("Ошибка платежа")
```

## Принципы SOLID

### S - Single Responsibility Principle (Принцип единственной ответственности)

Класс должен иметь только одну причину для изменения:

```python
# Плохо: класс делает слишком много
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email

    def save_to_database(self):
        # Логика БД
        pass

    def send_email(self, message):
        # Логика отправки email
        pass

    def generate_report(self):
        # Логика генерации отчёта
        pass

# Хорошо: разделение ответственности
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email

class UserRepository:
    def save(self, user: User):
        pass

class EmailService:
    def send(self, user: User, message: str):
        pass

class UserReportGenerator:
    def generate(self, user: User):
        pass
```

### O - Open/Closed Principle (Принцип открытости/закрытости)

Классы должны быть открыты для расширения, но закрыты для модификации:

```python
# Плохо: нужно модифицировать класс для добавления новых типов
class DiscountCalculator:
    def calculate(self, customer_type: str, amount: float) -> float:
        if customer_type == "regular":
            return amount * 0.1
        elif customer_type == "premium":
            return amount * 0.2
        elif customer_type == "vip":
            return amount * 0.3
        # Добавление нового типа требует изменения класса!

# Хорошо: расширяемая структура
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

# Добавление нового типа не требует изменения существующего кода
class StudentDiscount(DiscountStrategy):
    def calculate(self, amount: float) -> float:
        return amount * 0.15
```

### L - Liskov Substitution Principle (Принцип подстановки Лисков)

Объекты подклассов должны быть взаимозаменяемы с объектами базового класса:

```python
# Плохо: нарушение LSP
class Bird:
    def fly(self):
        return "Летит"

class Penguin(Bird):
    def fly(self):
        raise Exception("Пингвины не летают!")  # Нарушение!

# Хорошо: правильная иерархия
class Bird(ABC):
    @abstractmethod
    def move(self):
        pass

class FlyingBird(Bird):
    def move(self):
        return "Летит"

    def fly(self):
        return "Летит высоко"

class SwimmingBird(Bird):
    def move(self):
        return "Плывёт"

    def swim(self):
        return "Плывёт быстро"

class Sparrow(FlyingBird):
    pass

class Penguin(SwimmingBird):
    pass
```

### I - Interface Segregation Principle (Принцип разделения интерфейса)

Клиенты не должны зависеть от интерфейсов, которые они не используют:

```python
# Плохо: "толстый" интерфейс
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
        return "Работает"

    def eat(self):
        raise NotImplementedError()  # Робот не ест!

    def sleep(self):
        raise NotImplementedError()  # Робот не спит!

# Хорошо: разделённые интерфейсы
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
        return "Человек работает"

    def eat(self):
        return "Человек ест"

    def sleep(self):
        return "Человек спит"

class Robot(Workable):
    def work(self):
        return "Робот работает"
```

### D - Dependency Inversion Principle (Принцип инверсии зависимостей)

Высокоуровневые модули не должны зависеть от низкоуровневых. Оба должны зависеть от абстракций:

```python
# Плохо: прямая зависимость от конкретной реализации
class MySQLDatabase:
    def connect(self):
        return "MySQL connected"

    def query(self, sql):
        return f"MySQL: {sql}"

class UserService:
    def __init__(self):
        self.db = MySQLDatabase()  # Жёсткая зависимость!

    def get_user(self, user_id):
        return self.db.query(f"SELECT * FROM users WHERE id = {user_id}")

# Хорошо: зависимость от абстракции
class Database(ABC):
    @abstractmethod
    def connect(self):
        pass

    @abstractmethod
    def query(self, sql: str):
        pass

class MySQLDatabase(Database):
    def connect(self):
        return "MySQL connected"

    def query(self, sql: str):
        return f"MySQL: {sql}"

class PostgreSQLDatabase(Database):
    def connect(self):
        return "PostgreSQL connected"

    def query(self, sql: str):
        return f"PostgreSQL: {sql}"

class UserService:
    def __init__(self, db: Database):  # Инъекция зависимости
        self.db = db

    def get_user(self, user_id):
        return self.db.query(f"SELECT * FROM users WHERE id = {user_id}")

# Использование
mysql_service = UserService(MySQLDatabase())
postgres_service = UserService(PostgreSQLDatabase())
```

## Паттерны проектирования (краткий обзор)

### Порождающие паттерны

```python
# Singleton - единственный экземпляр
class Singleton:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

# Factory Method - фабричный метод
class AnimalFactory:
    @staticmethod
    def create(animal_type: str) -> Animal:
        if animal_type == "dog":
            return Dog("Бобик", 3, "Дворняга")
        elif animal_type == "cat":
            return Cat("Мурка", 2)
        raise ValueError(f"Неизвестный тип: {animal_type}")
```

### Структурные паттерны

```python
# Decorator - декоратор
class Coffee:
    def cost(self):
        return 100

class MilkDecorator:
    def __init__(self, coffee):
        self._coffee = coffee

    def cost(self):
        return self._coffee.cost() + 20

class SugarDecorator:
    def __init__(self, coffee):
        self._coffee = coffee

    def cost(self):
        return self._coffee.cost() + 10

coffee = SugarDecorator(MilkDecorator(Coffee()))
print(coffee.cost())  # 130
```

### Поведенческие паттерны

```python
# Observer - наблюдатель
class Subject:
    def __init__(self):
        self._observers = []

    def attach(self, observer):
        self._observers.append(observer)

    def notify(self, message):
        for observer in self._observers:
            observer.update(message)

class Observer:
    def update(self, message):
        print(f"Получено: {message}")
```

## Best Practices

### 1. Композиция вместо наследования

```python
# Часто лучше
class Engine:
    def start(self):
        return "Двигатель запущен"

class Car:
    def __init__(self):
        self.engine = Engine()  # Композиция

    def start(self):
        return self.engine.start()
```

### 2. Используйте property для контроля доступа

```python
class Temperature:
    def __init__(self):
        self._celsius = 0

    @property
    def celsius(self):
        return self._celsius

    @celsius.setter
    def celsius(self, value):
        if value < -273.15:
            raise ValueError("Температура ниже абсолютного нуля!")
        self._celsius = value

    @property
    def fahrenheit(self):
        return self._celsius * 9/5 + 32
```

### 3. Используйте __slots__ для оптимизации памяти

```python
class Point:
    __slots__ = ['x', 'y']

    def __init__(self, x, y):
        self.x = x
        self.y = y
```

## Типичные ошибки

### 1. Бог-объект (God Object)

```python
# Плохо: класс делает всё
class Application:
    def handle_user_login(self): pass
    def process_payment(self): pass
    def send_email(self): pass
    def generate_report(self): pass
    def backup_database(self): pass
    # ... 100+ методов
```

### 2. Глубокая иерархия наследования

```python
# Плохо: слишком много уровней
class A: pass
class B(A): pass
class C(B): pass
class D(C): pass
class E(D): pass  # 5 уровней - слишком много!
```

### 3. Нарушение инкапсуляции

```python
# Плохо: прямой доступ к внутренним данным
class User:
    def __init__(self):
        self.data = {}  # Публичный доступ к внутренней структуре

user = User()
user.data["password"] = "123"  # Опасно!
```

## Заключение

ООП — мощная парадигма для моделирования сложных систем. Ключ к успеху — правильное применение принципов SOLID и выбор подходящих паттернов проектирования. Помните, что ООП — это инструмент, и не каждая задача требует его использования.
