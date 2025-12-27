# Interfaces

## Интерфейсы в объектно-ориентированном программировании

Интерфейс — это контракт, который определяет набор методов, которые класс обязуется реализовать. Интерфейсы позволяют создавать слабосвязанные системы, где компоненты зависят от абстракций, а не от конкретных реализаций.

---

## Концепция интерфейса

### Что такое интерфейс

- **Контракт**: определяет, ЧТО должен уметь объект, но не КАК
- **Абстракция**: скрывает детали реализации
- **Точка расширения**: позволяет добавлять новые реализации без изменения существующего кода

### Интерфейс vs Абстрактный класс

| Аспект | Интерфейс | Абстрактный класс |
|--------|-----------|-------------------|
| Реализация | Только сигнатуры методов | Может содержать реализацию |
| Наследование | Множественное | Одиночное (в большинстве языков) |
| Состояние | Не содержит данных | Может содержать атрибуты |
| Назначение | Определяет возможности | Определяет общую базу |

---

## Интерфейсы в Python

Python не имеет ключевого слова `interface`, но предоставляет несколько способов создания интерфейсов.

### Абстрактные базовые классы (ABC)

```python
from abc import ABC, abstractmethod
from typing import Iterator


class Repository(ABC):
    """Интерфейс репозитория для работы с данными."""

    @abstractmethod
    def get(self, id: int) -> dict | None:
        """Получить запись по ID."""
        pass

    @abstractmethod
    def save(self, entity: dict) -> int:
        """Сохранить запись и вернуть ID."""
        pass

    @abstractmethod
    def delete(self, id: int) -> bool:
        """Удалить запись по ID."""
        pass

    @abstractmethod
    def find_all(self) -> Iterator[dict]:
        """Получить все записи."""
        pass


class PostgresRepository(Repository):
    """Реализация репозитория для PostgreSQL."""

    def __init__(self, connection_string: str):
        self.connection_string = connection_string

    def get(self, id: int) -> dict | None:
        # Логика работы с PostgreSQL
        print(f"PostgreSQL: SELECT * FROM table WHERE id = {id}")
        return {"id": id, "data": "..."}

    def save(self, entity: dict) -> int:
        print(f"PostgreSQL: INSERT INTO table ...")
        return 1

    def delete(self, id: int) -> bool:
        print(f"PostgreSQL: DELETE FROM table WHERE id = {id}")
        return True

    def find_all(self) -> Iterator[dict]:
        print("PostgreSQL: SELECT * FROM table")
        yield {"id": 1}
        yield {"id": 2}


class MongoRepository(Repository):
    """Реализация репозитория для MongoDB."""

    def __init__(self, uri: str, database: str):
        self.uri = uri
        self.database = database

    def get(self, id: int) -> dict | None:
        print(f"MongoDB: db.collection.findOne({{_id: {id}}})")
        return {"id": id, "data": "..."}

    def save(self, entity: dict) -> int:
        print("MongoDB: db.collection.insertOne(...)")
        return 1

    def delete(self, id: int) -> bool:
        print(f"MongoDB: db.collection.deleteOne({{_id: {id}}})")
        return True

    def find_all(self) -> Iterator[dict]:
        print("MongoDB: db.collection.find({})")
        yield {"id": 1}
        yield {"id": 2}


# Клиентский код работает с интерфейсом
class UserService:
    def __init__(self, repository: Repository):
        self.repository = repository

    def get_user(self, user_id: int) -> dict | None:
        return self.repository.get(user_id)


# Легко переключать реализации
postgres_repo = PostgresRepository("postgresql://localhost/db")
mongo_repo = MongoRepository("mongodb://localhost", "mydb")

service1 = UserService(postgres_repo)
service2 = UserService(mongo_repo)
```

### Протоколы (Protocol) — структурная типизация

Python 3.8+ поддерживает протоколы — способ определения интерфейсов через структурную типизацию (duck typing с проверкой типов).

```python
from typing import Protocol, runtime_checkable


@runtime_checkable
class Drawable(Protocol):
    """Протокол для объектов, которые можно нарисовать."""

    def draw(self) -> str:
        """Отрисовать объект и вернуть результат."""
        ...


class Circle:
    """Круг — реализует Drawable неявно."""

    def __init__(self, radius: float):
        self.radius = radius

    def draw(self) -> str:
        return f"○ (радиус: {self.radius})"


class Rectangle:
    """Прямоугольник — реализует Drawable неявно."""

    def __init__(self, width: float, height: float):
        self.width = width
        self.height = height

    def draw(self) -> str:
        return f"▭ ({self.width}x{self.height})"


class Button:
    """Кнопка — тоже реализует Drawable."""

    def __init__(self, text: str):
        self.text = text

    def draw(self) -> str:
        return f"[{self.text}]"


def render_all(items: list[Drawable]) -> None:
    """Функция работает с любыми Drawable объектами."""
    for item in items:
        print(item.draw())


# Использование
shapes: list[Drawable] = [
    Circle(5),
    Rectangle(10, 20),
    Button("Click me"),
]

render_all(shapes)

# Проверка протокола во время выполнения
print(isinstance(Circle(1), Drawable))  # True
print(isinstance("string", Drawable))   # False
```

### Сложные протоколы

```python
from typing import Protocol, TypeVar, Generic, Iterator


T = TypeVar("T")


class Comparable(Protocol):
    """Протокол для сравниваемых объектов."""

    def __lt__(self, other: "Comparable") -> bool:
        ...

    def __eq__(self, other: object) -> bool:
        ...


class Iterable(Protocol[T]):
    """Протокол для итерируемых объектов."""

    def __iter__(self) -> Iterator[T]:
        ...


class Sized(Protocol):
    """Протокол для объектов с размером."""

    def __len__(self) -> int:
        ...


class Collection(Iterable[T], Sized, Protocol[T]):
    """Комбинированный протокол — коллекция."""

    def __contains__(self, item: T) -> bool:
        ...


# Функция, принимающая любую коллекцию
def process_collection(items: Collection[str]) -> int:
    print(f"Размер: {len(items)}")
    for item in items:
        print(f"  - {item}")
    return len(items)


# Работает с list, set, tuple и любым другим,
# кто реализует нужные методы
process_collection(["a", "b", "c"])
process_collection({"x", "y", "z"})
```

---

## Принципы проектирования интерфейсов

### Interface Segregation Principle (ISP)

Клиенты не должны зависеть от методов, которые они не используют. Лучше много маленьких интерфейсов, чем один большой.

```python
from abc import ABC, abstractmethod


# Плохо: "жирный" интерфейс
class BadWorker(ABC):
    @abstractmethod
    def work(self) -> None:
        pass

    @abstractmethod
    def eat(self) -> None:
        pass

    @abstractmethod
    def sleep(self) -> None:
        pass


# Робот не ест и не спит, но вынужден реализовывать эти методы
class Robot(BadWorker):
    def work(self) -> None:
        print("Работаю...")

    def eat(self) -> None:
        pass  # Бессмысленная реализация

    def sleep(self) -> None:
        pass  # Бессмысленная реализация


# Хорошо: сегрегированные интерфейсы
class Workable(ABC):
    @abstractmethod
    def work(self) -> None:
        pass


class Eatable(ABC):
    @abstractmethod
    def eat(self) -> None:
        pass


class Sleepable(ABC):
    @abstractmethod
    def sleep(self) -> None:
        pass


class Human(Workable, Eatable, Sleepable):
    def work(self) -> None:
        print("Человек работает")

    def eat(self) -> None:
        print("Человек ест")

    def sleep(self) -> None:
        print("Человек спит")


class GoodRobot(Workable):  # Только нужный интерфейс
    def work(self) -> None:
        print("Робот работает 24/7")
```

### Dependency Inversion Principle (DIP)

Модули высокого уровня не должны зависеть от модулей низкого уровня. Оба должны зависеть от абстракций.

```python
from abc import ABC, abstractmethod


# Интерфейсы (абстракции)
class EmailSender(ABC):
    @abstractmethod
    def send(self, to: str, subject: str, body: str) -> bool:
        pass


class Logger(ABC):
    @abstractmethod
    def log(self, message: str) -> None:
        pass


# Низкоуровневые реализации
class SmtpEmailSender(EmailSender):
    def send(self, to: str, subject: str, body: str) -> bool:
        print(f"SMTP: Отправка письма на {to}")
        return True


class SendGridEmailSender(EmailSender):
    def send(self, to: str, subject: str, body: str) -> bool:
        print(f"SendGrid API: Отправка письма на {to}")
        return True


class FileLogger(Logger):
    def log(self, message: str) -> None:
        print(f"[FILE] {message}")


class CloudLogger(Logger):
    def log(self, message: str) -> None:
        print(f"[CLOUD] {message}")


# Высокоуровневый модуль зависит только от абстракций
class NotificationService:
    def __init__(self, email_sender: EmailSender, logger: Logger):
        self.email_sender = email_sender
        self.logger = logger

    def notify(self, user_email: str, message: str) -> None:
        self.logger.log(f"Отправка уведомления: {user_email}")
        success = self.email_sender.send(
            to=user_email,
            subject="Уведомление",
            body=message
        )
        if success:
            self.logger.log("Уведомление отправлено успешно")
        else:
            self.logger.log("Ошибка отправки уведомления")


# Легко менять реализации без изменения NotificationService
service = NotificationService(
    email_sender=SendGridEmailSender(),
    logger=CloudLogger()
)
service.notify("user@example.com", "Привет!")
```

---

## Паттерны с использованием интерфейсов

### Стратегия (Strategy)

```python
from abc import ABC, abstractmethod
from decimal import Decimal


class PricingStrategy(ABC):
    """Интерфейс стратегии ценообразования."""

    @abstractmethod
    def calculate_price(self, base_price: Decimal, quantity: int) -> Decimal:
        pass


class RegularPricing(PricingStrategy):
    def calculate_price(self, base_price: Decimal, quantity: int) -> Decimal:
        return base_price * quantity


class BulkPricing(PricingStrategy):
    """Скидка при оптовой покупке."""

    def __init__(self, discount_percent: int, min_quantity: int):
        self.discount = Decimal(discount_percent) / 100
        self.min_quantity = min_quantity

    def calculate_price(self, base_price: Decimal, quantity: int) -> Decimal:
        total = base_price * quantity
        if quantity >= self.min_quantity:
            total *= (1 - self.discount)
        return total


class LoyaltyPricing(PricingStrategy):
    """Скидка для постоянных клиентов."""

    def __init__(self, discount_percent: int):
        self.discount = Decimal(discount_percent) / 100

    def calculate_price(self, base_price: Decimal, quantity: int) -> Decimal:
        return base_price * quantity * (1 - self.discount)


class ShoppingCart:
    def __init__(self, pricing_strategy: PricingStrategy):
        self.strategy = pricing_strategy
        self.items: list[tuple[str, Decimal, int]] = []

    def add_item(self, name: str, price: Decimal, quantity: int = 1) -> None:
        self.items.append((name, price, quantity))

    def set_pricing_strategy(self, strategy: PricingStrategy) -> None:
        """Стратегию можно менять во время выполнения."""
        self.strategy = strategy

    def total(self) -> Decimal:
        return sum(
            self.strategy.calculate_price(price, qty)
            for _, price, qty in self.items
        )


# Использование
cart = ShoppingCart(RegularPricing())
cart.add_item("Товар A", Decimal("100"), 5)
print(f"Обычная цена: {cart.total()}")  # 500

cart.set_pricing_strategy(BulkPricing(10, 3))
print(f"Оптовая цена: {cart.total()}")  # 450

cart.set_pricing_strategy(LoyaltyPricing(15))
print(f"Цена для постоянного клиента: {cart.total()}")  # 425
```

### Адаптер (Adapter)

```python
from abc import ABC, abstractmethod


# Целевой интерфейс, который ожидает наш код
class PaymentProcessor(ABC):
    @abstractmethod
    def process_payment(self, amount: float, currency: str) -> bool:
        pass


# Существующая система, которую нужно адаптировать
class LegacyPaymentSystem:
    """Старая платёжная система с другим интерфейсом."""

    def make_payment(self, amount_in_cents: int) -> dict:
        print(f"Legacy: обработка {amount_in_cents} центов")
        return {"status": "success", "transaction_id": "12345"}


class LegacyPaymentAdapter(PaymentProcessor):
    """Адаптер, приводящий старый интерфейс к новому."""

    def __init__(self, legacy_system: LegacyPaymentSystem):
        self.legacy = legacy_system

    def process_payment(self, amount: float, currency: str) -> bool:
        # Конвертируем в формат старой системы
        if currency != "USD":
            raise ValueError("Legacy система поддерживает только USD")

        amount_in_cents = int(amount * 100)
        result = self.legacy.make_payment(amount_in_cents)
        return result["status"] == "success"


# Клиентский код работает с единым интерфейсом
def checkout(processor: PaymentProcessor, amount: float) -> None:
    if processor.process_payment(amount, "USD"):
        print("Оплата прошла успешно!")
    else:
        print("Ошибка оплаты")


# Используем старую систему через адаптер
legacy = LegacyPaymentSystem()
adapter = LegacyPaymentAdapter(legacy)
checkout(adapter, 99.99)
```

---

## Callable интерфейсы

В Python функции — это объекты первого класса. Можно использовать callable как интерфейс.

```python
from typing import Callable, Protocol


# Callable как тип
Handler = Callable[[str], str]


def uppercase_handler(message: str) -> str:
    return message.upper()


def reverse_handler(message: str) -> str:
    return message[::-1]


def process_message(message: str, handler: Handler) -> str:
    return handler(message)


print(process_message("hello", uppercase_handler))  # HELLO
print(process_message("hello", reverse_handler))    # olleh


# Протокол для callable объектов
class MessageHandler(Protocol):
    def __call__(self, message: str) -> str:
        ...


class PrefixHandler:
    def __init__(self, prefix: str):
        self.prefix = prefix

    def __call__(self, message: str) -> str:
        return f"{self.prefix}: {message}"


handler = PrefixHandler("INFO")
print(process_message("System started", handler))  # INFO: System started
```

---

## Best Practices

### 1. Называйте интерфейсы по возможностям

```python
# Хорошо: описывают возможности
class Readable(Protocol):
    def read(self) -> bytes: ...

class Writable(Protocol):
    def write(self, data: bytes) -> int: ...

class Closeable(Protocol):
    def close(self) -> None: ...
```

### 2. Держите интерфейсы маленькими

```python
# Один метод — один интерфейс (если это имеет смысл)
class Validator(Protocol):
    def validate(self, data: dict) -> bool: ...

class Serializer(Protocol):
    def serialize(self, obj: object) -> bytes: ...
```

### 3. Программируйте на уровне интерфейсов

```python
# Плохо: зависимость от конкретной реализации
def save_user(user: dict, repo: PostgresRepository) -> None:
    repo.save(user)

# Хорошо: зависимость от интерфейса
def save_user(user: dict, repo: Repository) -> None:
    repo.save(user)
```

---

## Типичные ошибки

1. **Слишком большие интерфейсы**
   - Нарушение ISP: клиенты вынуждены зависеть от ненужных методов

2. **Маркерные интерфейсы без методов**
   - Пустые интерфейсы — обычно признак проблемы в дизайне

3. **Интерфейсы, копирующие реализацию**
   - Интерфейс не должен быть 1:1 копией конкретного класса

4. **Смешивание интерфейса и реализации**
   ```python
   # Плохо: интерфейс содержит логику
   class BadRepository(ABC):
       @abstractmethod
       def get(self, id: int) -> dict:
           pass

       def get_or_create(self, id: int, default: dict) -> dict:
           result = self.get(id)
           if result is None:
               self.save(default)
               return default
           return result
   ```

5. **Зависимость от конкретных классов вместо интерфейсов**
   - Усложняет тестирование и расширение системы
