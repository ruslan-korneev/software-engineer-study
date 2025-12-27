# Coupling and Cohesion

## Введение

Связанность (Coupling) и сцепление (Cohesion) — это два фундаментальных понятия в проектировании программного обеспечения, которые определяют качество модульной структуры системы.

**Сцепление (Cohesion)** — мера того, насколько тесно связаны элементы внутри одного модуля. Высокое сцепление означает, что все части модуля работают вместе для достижения единой цели.

**Связанность (Coupling)** — мера зависимости между модулями. Низкая связанность означает, что модули можно изменять независимо друг от друга.

> **Цель хорошего дизайна: высокое сцепление внутри модулей и низкая связанность между модулями.**

## Сцепление (Cohesion)

### Типы сцепления (от худшего к лучшему)

#### 1. Случайное сцепление (Coincidental Cohesion)

Элементы модуля не связаны между собой — худший вид сцепления.

```python
# Плохо: случайный набор функций в одном модуле
# utils.py
import os
import smtplib
from PIL import Image

def calculate_tax(amount: float, rate: float) -> float:
    """Расчёт налога — финансовая операция."""
    return amount * rate

def send_email(to: str, subject: str, body: str) -> bool:
    """Отправка email — работа с почтой."""
    # ...
    pass

def resize_image(path: str, width: int, height: int) -> Image:
    """Изменение размера изображения — обработка графики."""
    # ...
    pass

def get_environment_variable(name: str) -> str:
    """Получение переменной окружения — конфигурация."""
    return os.getenv(name)
```

#### 2. Логическое сцепление (Logical Cohesion)

Элементы объединены по логическому признаку, но выполняют разные операции.

```python
# Плохо: модуль "всё, что связано с выводом"
# output.py
class OutputHandler:
    def output(self, data, output_type: str):
        if output_type == "console":
            print(data)
        elif output_type == "file":
            with open("output.txt", "w") as f:
                f.write(str(data))
        elif output_type == "email":
            self._send_email(data)
        elif output_type == "pdf":
            self._generate_pdf(data)

    def _send_email(self, data):
        # Логика email
        pass

    def _generate_pdf(self, data):
        # Логика PDF
        pass
```

#### 3. Временное сцепление (Temporal Cohesion)

Элементы выполняются в одно время (например, при инициализации).

```python
# Плохо: всё, что происходит при старте приложения
# startup.py
class ApplicationStartup:
    def initialize(self):
        # Разные несвязанные действия, объединённые только временем выполнения
        self._load_config()
        self._connect_database()
        self._initialize_cache()
        self._start_metrics_collector()
        self._send_startup_notification()
        self._cleanup_temp_files()

    def _load_config(self):
        pass

    def _connect_database(self):
        pass

    def _initialize_cache(self):
        pass

    def _start_metrics_collector(self):
        pass

    def _send_startup_notification(self):
        pass

    def _cleanup_temp_files(self):
        pass
```

#### 4. Процедурное сцепление (Procedural Cohesion)

Элементы выполняются в определённой последовательности.

```python
# Средне: шаги обработки заказа в одном модуле
# order_processor.py
class OrderProcessor:
    def process_order(self, order_data: dict):
        # Последовательность шагов
        validated_data = self._validate_order(order_data)
        inventory = self._check_inventory(validated_data)
        payment = self._process_payment(validated_data)
        shipment = self._create_shipment(validated_data)
        notification = self._send_confirmation(validated_data)
        return {"status": "completed"}
```

#### 5. Коммуникационное сцепление (Communicational Cohesion)

Элементы работают с одними и теми же данными.

```python
# Средне: все операции над пользователем
# user_operations.py
class UserOperations:
    def __init__(self, user_id: int):
        self.user = self._load_user(user_id)

    def _load_user(self, user_id: int):
        # Загрузка пользователя
        pass

    def get_user_orders(self):
        # Работает с self.user
        pass

    def get_user_profile(self):
        # Работает с self.user
        pass

    def update_user_preferences(self, prefs: dict):
        # Работает с self.user
        pass

    def calculate_user_statistics(self):
        # Работает с self.user
        pass
```

#### 6. Последовательное сцепление (Sequential Cohesion)

Выход одного элемента является входом для другого.

```python
# Хорошо: конвейер обработки данных
# data_pipeline.py
class DataPipeline:
    def process(self, raw_data: bytes) -> dict:
        # Каждый шаг использует результат предыдущего
        decoded = self._decode(raw_data)
        validated = self._validate(decoded)
        transformed = self._transform(validated)
        enriched = self._enrich(transformed)
        return enriched

    def _decode(self, data: bytes) -> str:
        return data.decode("utf-8")

    def _validate(self, data: str) -> dict:
        import json
        return json.loads(data)

    def _transform(self, data: dict) -> dict:
        # Нормализация полей
        return {k.lower(): v for k, v in data.items()}

    def _enrich(self, data: dict) -> dict:
        data["processed_at"] = datetime.now().isoformat()
        return data
```

#### 7. Функциональное сцепление (Functional Cohesion)

Все элементы модуля вносят вклад в единственную, чётко определённую задачу — лучший вид сцепления.

```python
# Отлично: модуль для расчёта скидок
# discount_calculator.py
from decimal import Decimal
from dataclasses import dataclass
from typing import List


@dataclass
class DiscountRule:
    min_amount: Decimal
    percentage: Decimal


class DiscountCalculator:
    """Единственная ответственность: расчёт скидок."""

    def __init__(self, rules: List[DiscountRule]):
        self.rules = sorted(rules, key=lambda r: r.min_amount, reverse=True)

    def calculate(self, amount: Decimal) -> Decimal:
        """Рассчитать скидку для суммы."""
        percentage = self._find_applicable_discount(amount)
        return self._apply_percentage(amount, percentage)

    def _find_applicable_discount(self, amount: Decimal) -> Decimal:
        """Найти применимое правило скидки."""
        for rule in self.rules:
            if amount >= rule.min_amount:
                return rule.percentage
        return Decimal("0")

    def _apply_percentage(self, amount: Decimal, percentage: Decimal) -> Decimal:
        """Применить процент скидки к сумме."""
        return amount * percentage / Decimal("100")


# Отлично: модуль для валидации email
# email_validator.py
import re
from dataclasses import dataclass


@dataclass
class ValidationResult:
    is_valid: bool
    error: str | None = None


class EmailValidator:
    """Единственная ответственность: валидация email адресов."""

    EMAIL_PATTERN = re.compile(
        r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"
    )
    MAX_LENGTH = 254

    def validate(self, email: str) -> ValidationResult:
        """Валидировать email адрес."""
        if not email:
            return ValidationResult(False, "Email is required")

        if len(email) > self.MAX_LENGTH:
            return ValidationResult(False, f"Email too long (max {self.MAX_LENGTH})")

        if not self._matches_pattern(email):
            return ValidationResult(False, "Invalid email format")

        return ValidationResult(True)

    def _matches_pattern(self, email: str) -> bool:
        """Проверить соответствие паттерну."""
        return bool(self.EMAIL_PATTERN.match(email))
```

### Как улучшить сцепление

```python
# До: низкое сцепление — класс делает слишком много
class UserManager:
    def create_user(self, data: dict):
        # Валидация
        if not data.get("email"):
            raise ValueError("Email required")
        # Хеширование пароля
        import hashlib
        password_hash = hashlib.sha256(data["password"].encode()).hexdigest()
        # Сохранение в БД
        self.db.execute("INSERT INTO users ...")
        # Отправка email
        import smtplib
        server = smtplib.SMTP("smtp.gmail.com", 587)
        server.send_message(...)
        # Логирование
        self.logger.info("User created")


# После: высокое сцепление — каждый класс делает одно дело
class UserValidator:
    """Только валидация пользователей."""

    def validate(self, data: dict) -> list[str]:
        errors = []
        if not data.get("email"):
            errors.append("Email required")
        if not data.get("password"):
            errors.append("Password required")
        return errors


class PasswordHasher:
    """Только хеширование паролей."""

    def hash(self, password: str) -> str:
        import hashlib
        return hashlib.sha256(password.encode()).hexdigest()

    def verify(self, password: str, hash: str) -> bool:
        return self.hash(password) == hash


class UserRepository:
    """Только работа с хранилищем пользователей."""

    def __init__(self, db):
        self.db = db

    def save(self, user: "User") -> "User":
        # SQL операции
        pass

    def find_by_email(self, email: str) -> "User | None":
        pass


class WelcomeEmailSender:
    """Только отправка приветственных писем."""

    def send(self, user: "User") -> bool:
        pass


class UserService:
    """Оркестрация создания пользователя."""

    def __init__(
        self,
        validator: UserValidator,
        hasher: PasswordHasher,
        repository: UserRepository,
        email_sender: WelcomeEmailSender
    ):
        self.validator = validator
        self.hasher = hasher
        self.repository = repository
        self.email_sender = email_sender

    def create_user(self, data: dict) -> "User":
        errors = self.validator.validate(data)
        if errors:
            raise ValueError(errors)

        user = User(
            email=data["email"],
            password_hash=self.hasher.hash(data["password"])
        )

        saved_user = self.repository.save(user)
        self.email_sender.send(saved_user)

        return saved_user
```

## Связанность (Coupling)

### Типы связанности (от худшего к лучшему)

#### 1. Связанность по содержимому (Content Coupling)

Один модуль напрямую обращается к внутренним данным другого — худший вид связанности.

```python
# Плохо: прямой доступ к внутренним данным
class Order:
    def __init__(self):
        self._items = []
        self._discount = 0

class OrderProcessor:
    def process(self, order: Order):
        # Прямой доступ к приватным атрибутам!
        order._items.append({"id": 1, "price": 100})
        order._discount = 10  # Изменение внутреннего состояния


# Хорошо: использование публичного интерфейса
class Order:
    def __init__(self):
        self._items = []
        self._discount = 0

    def add_item(self, item: dict):
        self._items.append(item)

    def apply_discount(self, percentage: int):
        self._discount = percentage

    @property
    def items(self) -> list:
        return self._items.copy()  # Возвращаем копию


class OrderProcessor:
    def process(self, order: Order):
        order.add_item({"id": 1, "price": 100})
        order.apply_discount(10)
```

#### 2. Связанность по общему состоянию (Common Coupling)

Модули используют общие глобальные данные.

```python
# Плохо: глобальное состояние
_current_user = None
_database_connection = None

class UserService:
    def get_current_user(self):
        global _current_user
        return _current_user

    def set_current_user(self, user):
        global _current_user
        _current_user = user


class OrderService:
    def create_order(self, items: list):
        global _current_user, _database_connection
        # Зависит от глобального состояния
        if not _current_user:
            raise ValueError("No user logged in")
        # Использует глобальное соединение
        _database_connection.execute("INSERT ...")


# Хорошо: явная передача зависимостей
class UserService:
    def __init__(self, session_store: "SessionStore"):
        self.session_store = session_store

    def get_current_user(self, session_id: str) -> "User":
        return self.session_store.get_user(session_id)


class OrderService:
    def __init__(self, db: "Database"):
        self.db = db

    def create_order(self, user: "User", items: list):
        # Явная зависимость, нет глобального состояния
        self.db.execute("INSERT ...", user_id=user.id)
```

#### 3. Связанность по внешнему ресурсу (External Coupling)

Модули зависят от конкретного внешнего ресурса (файл, URL, формат данных).

```python
# Плохо: жёсткая привязка к конкретному файлу
class ConfigLoader:
    def load(self) -> dict:
        # Жёсткая зависимость от конкретного файла
        with open("/etc/app/config.json") as f:
            return json.load(f)


class UserService:
    def __init__(self):
        # Зависит от ConfigLoader и его внешнего ресурса
        self.config = ConfigLoader().load()


# Хорошо: абстракция над источником данных
from abc import ABC, abstractmethod

class ConfigSource(ABC):
    @abstractmethod
    def load(self) -> dict:
        pass


class FileConfigSource(ConfigSource):
    def __init__(self, path: str):
        self.path = path

    def load(self) -> dict:
        with open(self.path) as f:
            return json.load(f)


class EnvironmentConfigSource(ConfigSource):
    def __init__(self, prefix: str = "APP_"):
        self.prefix = prefix

    def load(self) -> dict:
        return {
            k[len(self.prefix):].lower(): v
            for k, v in os.environ.items()
            if k.startswith(self.prefix)
        }


class UserService:
    def __init__(self, config: dict):
        # Получает готовую конфигурацию
        self.config = config
```

#### 4. Связанность по управлению (Control Coupling)

Один модуль управляет поведением другого через флаги или параметры.

```python
# Плохо: флаг управляет поведением
class ReportGenerator:
    def generate(self, data: list, output_type: str):
        if output_type == "pdf":
            return self._generate_pdf(data)
        elif output_type == "excel":
            return self._generate_excel(data)
        elif output_type == "csv":
            return self._generate_csv(data)


class ReportService:
    def create_report(self, data: list, format: str):
        generator = ReportGenerator()
        # Управляет поведением через строковый параметр
        return generator.generate(data, format)


# Хорошо: полиморфизм вместо флагов
from abc import ABC, abstractmethod

class ReportFormatter(ABC):
    @abstractmethod
    def format(self, data: list) -> bytes:
        pass


class PDFFormatter(ReportFormatter):
    def format(self, data: list) -> bytes:
        # Генерация PDF
        pass


class ExcelFormatter(ReportFormatter):
    def format(self, data: list) -> bytes:
        # Генерация Excel
        pass


class CSVFormatter(ReportFormatter):
    def format(self, data: list) -> bytes:
        # Генерация CSV
        pass


class ReportService:
    def create_report(self, data: list, formatter: ReportFormatter) -> bytes:
        # Поведение определяется типом объекта, а не флагом
        return formatter.format(data)
```

#### 5. Связанность по структуре данных (Stamp Coupling)

Модули передают друг другу больше данных, чем нужно.

```python
# Плохо: передача всего объекта, когда нужна только часть
class User:
    def __init__(self):
        self.id = 1
        self.name = "John"
        self.email = "john@example.com"
        self.password_hash = "xxx"
        self.created_at = datetime.now()
        self.settings = {}
        self.orders = []


class EmailService:
    def send_welcome_email(self, user: User):
        # Нужен только email и name, но получает весь объект
        self._send(
            to=user.email,
            subject=f"Welcome, {user.name}!"
        )


# Хорошо: передача только необходимых данных
from dataclasses import dataclass

@dataclass
class EmailRecipient:
    email: str
    name: str


class EmailService:
    def send_welcome_email(self, recipient: EmailRecipient):
        self._send(
            to=recipient.email,
            subject=f"Welcome, {recipient.name}!"
        )


# Использование
user = User()
recipient = EmailRecipient(email=user.email, name=user.name)
email_service.send_welcome_email(recipient)
```

#### 6. Связанность по данным (Data Coupling)

Модули обмениваются только необходимыми данными через параметры — лучший вид связанности.

```python
# Отлично: минимальная связанность
class TaxCalculator:
    """Принимает только необходимые данные."""

    def calculate(self, amount: Decimal, rate: Decimal) -> Decimal:
        return amount * rate / Decimal("100")


class PriceFormatter:
    """Принимает только значение для форматирования."""

    def format(self, amount: Decimal, currency: str = "USD") -> str:
        return f"{currency} {amount:.2f}"


class InvoiceService:
    def __init__(
        self,
        tax_calculator: TaxCalculator,
        formatter: PriceFormatter
    ):
        self.tax_calculator = tax_calculator
        self.formatter = formatter

    def calculate_total(
        self,
        subtotal: Decimal,
        tax_rate: Decimal
    ) -> str:
        # Передаёт только необходимые данные
        tax = self.tax_calculator.calculate(subtotal, tax_rate)
        total = subtotal + tax
        return self.formatter.format(total)
```

#### 7. Отсутствие связанности (No Coupling)

Модули полностью независимы — идеальный, но редкодостижимый случай.

```python
# Идеально: полностью независимые утилиты
# string_utils.py
def capitalize_words(text: str) -> str:
    return " ".join(word.capitalize() for word in text.split())


# math_utils.py
def factorial(n: int) -> int:
    if n <= 1:
        return 1
    return n * factorial(n - 1)


# Эти модули никак не связаны друг с другом
```

### Метрики связанности

```python
# Afferent Coupling (Ca) — входящие зависимости
# Сколько других модулей зависит от данного модуля

# Efferent Coupling (Ce) — исходящие зависимости
# От скольких модулей зависит данный модуль

# Instability (I) = Ce / (Ca + Ce)
# I = 0: полностью стабильный (много зависят от нас)
# I = 1: полностью нестабильный (мы зависим от многих)


# Пример анализа
class PaymentService:  # Ce = 3 (зависит от 3 модулей)
    def __init__(
        self,
        gateway: "PaymentGateway",      # 1
        repository: "PaymentRepository", # 2
        notifier: "PaymentNotifier"      # 3
    ):
        pass


# Если от PaymentService зависят 5 других модулей (Ca = 5):
# I = 3 / (5 + 3) = 0.375 — относительно стабильный
```

## Практические примеры

### Рефакторинг: от высокой связанности к низкой

```python
# До: высокая связанность
class OrderController:
    def create_order(self, request):
        # Прямая зависимость от конкретной БД
        from mysql.connector import connect
        conn = connect(host="localhost", database="shop")
        cursor = conn.cursor()

        # Прямая зависимость от конкретного платёжного провайдера
        import stripe
        stripe.api_key = "sk_test_xxx"

        # Бизнес-логика смешана с инфраструктурой
        user_id = request.user_id
        cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")
        user = cursor.fetchone()

        total = sum(item["price"] for item in request.items)

        # Прямой вызов внешнего сервиса
        payment = stripe.PaymentIntent.create(amount=int(total * 100))

        cursor.execute("INSERT INTO orders ...")
        conn.commit()

        # Прямая отправка email
        import smtplib
        server = smtplib.SMTP("smtp.gmail.com", 587)
        server.send_message(...)

        return {"order_id": cursor.lastrowid}


# После: низкая связанность через абстракции
from typing import Protocol
from dataclasses import dataclass
from decimal import Decimal


# Абстракции (порты)
class UserRepository(Protocol):
    def find_by_id(self, user_id: int) -> "User | None": ...


class OrderRepository(Protocol):
    def save(self, order: "Order") -> "Order": ...


class PaymentProcessor(Protocol):
    def process(self, amount: Decimal) -> "PaymentResult": ...


class NotificationSender(Protocol):
    def send_order_confirmation(self, order: "Order") -> None: ...


# Доменная модель
@dataclass
class OrderItem:
    product_id: int
    price: Decimal
    quantity: int


@dataclass
class Order:
    id: int | None
    user_id: int
    items: list[OrderItem]

    @property
    def total(self) -> Decimal:
        return sum(item.price * item.quantity for item in self.items)


# Use Case — чистая бизнес-логика
class CreateOrderUseCase:
    def __init__(
        self,
        user_repository: UserRepository,
        order_repository: OrderRepository,
        payment_processor: PaymentProcessor,
        notification_sender: NotificationSender
    ):
        self.users = user_repository
        self.orders = order_repository
        self.payments = payment_processor
        self.notifications = notification_sender

    def execute(self, user_id: int, items: list[dict]) -> Order:
        user = self.users.find_by_id(user_id)
        if not user:
            raise ValueError("User not found")

        order = Order(
            id=None,
            user_id=user_id,
            items=[
                OrderItem(
                    product_id=i["product_id"],
                    price=Decimal(str(i["price"])),
                    quantity=i["quantity"]
                )
                for i in items
            ]
        )

        self.payments.process(order.total)
        saved_order = self.orders.save(order)
        self.notifications.send_order_confirmation(saved_order)

        return saved_order


# Контроллер — тонкий адаптер
class OrderController:
    def __init__(self, create_order: CreateOrderUseCase):
        self.create_order = create_order

    def handle_create_order(self, request) -> dict:
        order = self.create_order.execute(
            user_id=request.user_id,
            items=request.items
        )
        return {"order_id": order.id}
```

### Паттерны для снижения связанности

#### 1. Dependency Injection

```python
# Инъекция зависимостей через конструктор
class OrderService:
    def __init__(
        self,
        repository: OrderRepository,
        payment: PaymentProcessor
    ):
        self.repository = repository
        self.payment = payment


# Фабрика для создания сервисов
class ServiceFactory:
    def __init__(self, config: dict):
        self.config = config

    def create_order_service(self) -> OrderService:
        return OrderService(
            repository=SQLOrderRepository(self.config["database"]),
            payment=StripePaymentProcessor(self.config["stripe_key"])
        )
```

#### 2. Event-Driven Architecture

```python
# Слабая связанность через события
from dataclasses import dataclass
from typing import Callable


@dataclass
class OrderCreatedEvent:
    order_id: int
    user_id: int
    total: Decimal


class EventBus:
    def __init__(self):
        self._handlers: dict[type, list[Callable]] = {}

    def subscribe(self, event_type: type, handler: Callable):
        if event_type not in self._handlers:
            self._handlers[event_type] = []
        self._handlers[event_type].append(handler)

    def publish(self, event):
        handlers = self._handlers.get(type(event), [])
        for handler in handlers:
            handler(event)


# Модули не знают друг о друге — только об EventBus
class OrderService:
    def __init__(self, event_bus: EventBus):
        self.event_bus = event_bus

    def create_order(self, user_id: int, items: list) -> Order:
        order = Order(user_id=user_id, items=items)
        # Публикуем событие вместо прямого вызова
        self.event_bus.publish(OrderCreatedEvent(
            order_id=order.id,
            user_id=user_id,
            total=order.total
        ))
        return order


class EmailService:
    def __init__(self, event_bus: EventBus):
        event_bus.subscribe(OrderCreatedEvent, self._on_order_created)

    def _on_order_created(self, event: OrderCreatedEvent):
        # Реагируем на событие
        self._send_confirmation_email(event.order_id)


class AnalyticsService:
    def __init__(self, event_bus: EventBus):
        event_bus.subscribe(OrderCreatedEvent, self._on_order_created)

    def _on_order_created(self, event: OrderCreatedEvent):
        # Другой модуль реагирует на то же событие
        self._track_purchase(event.total)
```

#### 3. Mediator Pattern

```python
# Посредник координирует взаимодействие
from abc import ABC, abstractmethod


class Command(ABC):
    pass


class CommandHandler(ABC):
    @abstractmethod
    def handle(self, command: Command):
        pass


@dataclass
class CreateOrderCommand(Command):
    user_id: int
    items: list


class Mediator:
    def __init__(self):
        self._handlers: dict[type, CommandHandler] = {}

    def register(self, command_type: type, handler: CommandHandler):
        self._handlers[command_type] = handler

    def send(self, command: Command):
        handler = self._handlers.get(type(command))
        if handler:
            return handler.handle(command)
        raise ValueError(f"No handler for {type(command)}")


class CreateOrderHandler(CommandHandler):
    def handle(self, command: CreateOrderCommand):
        # Обработка команды
        pass


# Использование
mediator = Mediator()
mediator.register(CreateOrderCommand, CreateOrderHandler())

# Вызывающий код не знает о конкретном обработчике
result = mediator.send(CreateOrderCommand(user_id=1, items=[]))
```

## Best Practices

### 1. Следуйте принципу единственной ответственности

```python
# Один класс — одна причина для изменения
class UserValidator:
    """Только валидация."""
    pass

class UserRepository:
    """Только хранение."""
    pass

class UserNotifier:
    """Только уведомления."""
    pass
```

### 2. Программируйте на уровне интерфейсов

```python
# Зависьте от абстракций, не от реализаций
from typing import Protocol

class Logger(Protocol):
    def log(self, message: str) -> None: ...


class FileLogger:
    def log(self, message: str) -> None:
        with open("app.log", "a") as f:
            f.write(message + "\n")


class ConsoleLogger:
    def log(self, message: str) -> None:
        print(message)


class OrderService:
    def __init__(self, logger: Logger):  # Зависит от Protocol
        self.logger = logger
```

### 3. Используйте Dependency Injection

```python
# Не создавайте зависимости внутри класса
class OrderService:
    # Плохо
    def __init__(self):
        self.repository = SQLOrderRepository()  # Жёсткая зависимость

    # Хорошо
    def __init__(self, repository: OrderRepository):
        self.repository = repository  # Инъекция
```

## Типичные ошибки

### 1. God Object — объект, знающий всё

```python
# Плохо: один класс делает всё
class ApplicationManager:
    def create_user(self): pass
    def delete_user(self): pass
    def create_order(self): pass
    def process_payment(self): pass
    def send_email(self): pass
    def generate_report(self): pass
    def backup_database(self): pass
```

### 2. Feature Envy — зависть к функциональности

```python
# Плохо: метод больше использует данные другого класса
class Order:
    def __init__(self):
        self.items = []


class OrderPrinter:
    def print(self, order: Order):
        # Слишком много обращений к Order
        for item in order.items:
            print(f"{item.name}: {item.price}")
        print(f"Total: {sum(i.price for i in order.items)}")


# Хорошо: логика в правильном классе
class Order:
    def __init__(self):
        self.items = []

    def get_summary(self) -> str:
        lines = [f"{item.name}: {item.price}" for item in self.items]
        lines.append(f"Total: {self.total}")
        return "\n".join(lines)

    @property
    def total(self) -> Decimal:
        return sum(item.price for item in self.items)
```

### 3. Shotgun Surgery — изменение размазано по системе

```python
# Плохо: добавление нового типа платежа требует изменений везде
# payment_processor.py
if payment_type == "card": ...
elif payment_type == "paypal": ...
elif payment_type == "crypto": ...  # Новый тип — изменение здесь

# order_service.py
if payment_type == "crypto": ...  # И здесь

# reporting.py
if payment_type == "crypto": ...  # И здесь

# Хорошо: новый тип — один новый класс
class CryptoPayment(PaymentMethod):
    def process(self, amount: Decimal) -> PaymentResult:
        pass
```

## Заключение

Правильное управление сцеплением и связанностью — основа качественной архитектуры:

1. **Высокое сцепление**: элементы модуля тесно связаны единой целью
2. **Низкая связанность**: модули минимально зависят друг от друга
3. **Абстракции**: используйте интерфейсы для снижения связанности
4. **DI**: внедряйте зависимости, не создавайте их
5. **События**: используйте event-driven для слабой связанности
6. **SRP**: один модуль — одна ответственность

Эти принципы обеспечивают:
- Простоту изменения отдельных модулей
- Лёгкость тестирования
- Возможность параллельной разработки
- Переиспользование кода
