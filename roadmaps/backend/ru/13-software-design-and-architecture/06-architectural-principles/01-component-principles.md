# Component Principles

## Введение

Принципы проектирования компонентов (Component Principles) — это набор рекомендаций, определяющих, как группировать классы в компоненты и как эти компоненты должны взаимодействовать друг с другом. Эти принципы были сформулированы Робертом Мартином (Uncle Bob) и являются продолжением SOLID-принципов на уровне архитектуры.

Компонент — это единица развёртывания. В разных языках и экосистемах компоненты реализуются по-разному: модули в Python, пакеты в Java, gem'ы в Ruby, npm-пакеты в JavaScript.

Принципы делятся на две группы:
1. **Принципы связности компонентов** (Component Cohesion) — какие классы объединять в компонент
2. **Принципы сцепления компонентов** (Component Coupling) — как компоненты должны зависеть друг от друга

## Принципы связности компонентов

### REP — Reuse/Release Equivalence Principle

**Принцип эквивалентности повторного использования и выпуска**

> "Гранула повторного использования — это гранула выпуска"

Классы и модули, объединённые в компонент, должны выпускаться вместе. Если вы хотите повторно использовать код, он должен быть версионирован и выпущен как единое целое.

```python
# Плохо: случайный набор утилит в одном пакете
# utils/
#   string_helpers.py    - работа со строками
#   payment_gateway.py   - интеграция с платёжной системой
#   image_processor.py   - обработка изображений

# Хорошо: логически связанные классы в одном компоненте
# payment/
#   __init__.py
#   gateway.py
#   transaction.py
#   refund.py
#   exceptions.py

# payment/gateway.py
class PaymentGateway:
    """Основной класс для работы с платежами."""

    def __init__(self, api_key: str):
        self.api_key = api_key

    def process_payment(self, amount: float, card_token: str) -> "Transaction":
        # Обработка платежа
        pass


# payment/transaction.py
class Transaction:
    """Представляет транзакцию платежа."""

    def __init__(self, transaction_id: str, amount: float, status: str):
        self.transaction_id = transaction_id
        self.amount = amount
        self.status = status

    def refund(self) -> "Refund":
        # Создание возврата
        pass


# payment/refund.py
class Refund:
    """Представляет возврат средств."""

    def __init__(self, refund_id: str, transaction: Transaction):
        self.refund_id = refund_id
        self.transaction = transaction
```

**Практические рекомендации:**
- Используйте семантическое версионирование (semver)
- Ведите CHANGELOG
- Все классы в компоненте должны иметь общую цель

### CCP — Common Closure Principle

**Принцип общего закрытия**

> "Классы, которые изменяются по одной причине, должны быть объединены в один компонент"

Это применение принципа единственной ответственности (SRP) на уровне компонентов. Когда требование изменяется, изменения должны затрагивать минимальное количество компонентов.

```python
# Плохо: связанные классы разбросаны по разным пакетам
# models/user.py
class User:
    def __init__(self, name: str, email: str):
        self.name = name
        self.email = email

# validators/user_validator.py
class UserValidator:
    def validate(self, user):
        pass

# serializers/user_serializer.py
class UserSerializer:
    def serialize(self, user):
        pass


# Хорошо: всё, что связано с пользователем, в одном компоненте
# user/
#   __init__.py
#   model.py
#   validator.py
#   serializer.py
#   repository.py

# user/model.py
from dataclasses import dataclass
from typing import Optional

@dataclass
class User:
    id: Optional[int]
    name: str
    email: str
    is_active: bool = True


# user/validator.py
from .model import User

class UserValidator:
    def validate(self, user: User) -> list[str]:
        errors = []
        if not user.name:
            errors.append("Name is required")
        if not user.email or "@" not in user.email:
            errors.append("Valid email is required")
        return errors


# user/serializer.py
from .model import User

class UserSerializer:
    def to_dict(self, user: User) -> dict:
        return {
            "id": user.id,
            "name": user.name,
            "email": user.email,
            "is_active": user.is_active
        }

    def from_dict(self, data: dict) -> User:
        return User(
            id=data.get("id"),
            name=data["name"],
            email=data["email"],
            is_active=data.get("is_active", True)
        )
```

### CRP — Common Reuse Principle

**Принцип общего повторного использования**

> "Не заставляйте пользователей компонента зависеть от того, что им не нужно"

Классы в компоненте должны использоваться вместе. Если вы используете один класс из компонента, вы должны использовать все (или большинство). Это обратная сторона CCP.

```python
# Плохо: компонент с несвязанными зависимостями
# reporting/
#   pdf_generator.py      - требует библиотеку reportlab
#   excel_exporter.py     - требует библиотеку openpyxl
#   email_sender.py       - требует библиотеку sendgrid
#   chart_builder.py      - требует библиотеку matplotlib

# Если нужен только PDF, приходится тянуть все зависимости!


# Хорошо: разделение на отдельные компоненты
# pdf_reports/
#   __init__.py
#   generator.py
#   templates.py

# excel_reports/
#   __init__.py
#   exporter.py
#   formatters.py

# notifications/
#   __init__.py
#   email_sender.py
#   templates.py


# pdf_reports/generator.py
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import A4

class PDFReportGenerator:
    def __init__(self, output_path: str):
        self.output_path = output_path

    def generate(self, data: dict) -> str:
        c = canvas.Canvas(self.output_path, pagesize=A4)
        # Генерация PDF
        c.drawString(100, 750, data.get("title", "Report"))
        c.save()
        return self.output_path


# excel_reports/exporter.py
from openpyxl import Workbook

class ExcelExporter:
    def export(self, data: list[dict], output_path: str) -> str:
        wb = Workbook()
        ws = wb.active

        if data:
            # Заголовки
            headers = list(data[0].keys())
            ws.append(headers)

            # Данные
            for row in data:
                ws.append(list(row.values()))

        wb.save(output_path)
        return output_path
```

## Принципы сцепления компонентов

### ADP — Acyclic Dependencies Principle

**Принцип ацикличности зависимостей**

> "В графе зависимостей компонентов не должно быть циклов"

Циклические зависимости создают проблемы при сборке, тестировании и развёртывании. Если компонент A зависит от B, а B зависит от A, изменение в любом из них требует пересборки обоих.

```python
# Плохо: циклическая зависимость
# orders/service.py
from customers.service import CustomerService  # orders -> customers

class OrderService:
    def __init__(self):
        self.customer_service = CustomerService()

    def create_order(self, customer_id: int, items: list):
        customer = self.customer_service.get_customer(customer_id)
        # ...


# customers/service.py
from orders.service import OrderService  # customers -> orders (ЦИКЛ!)

class CustomerService:
    def __init__(self):
        self.order_service = OrderService()

    def get_customer_orders(self, customer_id: int):
        return self.order_service.get_orders_by_customer(customer_id)


# Хорошо: разрыв цикла через инверсию зависимостей
# Создаём интерфейс в отдельном компоненте

# interfaces/order_provider.py
from abc import ABC, abstractmethod

class OrderProvider(ABC):
    @abstractmethod
    def get_orders_by_customer(self, customer_id: int) -> list:
        pass


# customers/service.py
from interfaces.order_provider import OrderProvider

class CustomerService:
    def __init__(self, order_provider: OrderProvider):
        self.order_provider = order_provider

    def get_customer(self, customer_id: int):
        # Получение клиента
        pass

    def get_customer_orders(self, customer_id: int):
        return self.order_provider.get_orders_by_customer(customer_id)


# orders/service.py
from interfaces.order_provider import OrderProvider
from customers.service import CustomerService

class OrderService(OrderProvider):
    def create_order(self, customer_id: int, items: list):
        # Создание заказа
        pass

    def get_orders_by_customer(self, customer_id: int) -> list:
        # Получение заказов клиента
        return []


# main.py - сборка зависимостей
order_service = OrderService()
customer_service = CustomerService(order_provider=order_service)
```

### SDP — Stable Dependencies Principle

**Принцип стабильных зависимостей**

> "Компонент должен зависеть только от более стабильных компонентов"

Стабильность компонента определяется количеством зависящих от него компонентов. Чем больше компонентов зависит от данного, тем он стабильнее (сложнее изменить).

**Метрика нестабильности I:**
```
I = Fan-out / (Fan-in + Fan-out)

где:
- Fan-in: количество входящих зависимостей (кто зависит от нас)
- Fan-out: количество исходящих зависимостей (от кого зависим мы)
- I = 0: максимально стабильный компонент
- I = 1: максимально нестабильный компонент
```

```python
# Пример структуры с правильными зависимостями

# core/ (I ≈ 0, очень стабильный - от него зависят все)
#   domain/
#     entities.py
#     value_objects.py
#     exceptions.py

# infrastructure/ (I ≈ 0.5, средняя стабильность)
#   database/
#     repository.py
#   external/
#     api_client.py

# application/ (I ≈ 0.7, низкая стабильность)
#   services/
#     order_service.py
#     user_service.py

# presentation/ (I ≈ 1, максимально нестабильный)
#   api/
#     routes.py
#     controllers.py


# core/domain/entities.py (стабильный - никаких внешних зависимостей)
from dataclasses import dataclass
from decimal import Decimal

@dataclass
class Product:
    id: int
    name: str
    price: Decimal

    def apply_discount(self, percentage: Decimal) -> Decimal:
        discount = self.price * percentage / 100
        return self.price - discount


@dataclass
class OrderItem:
    product: Product
    quantity: int

    @property
    def total(self) -> Decimal:
        return self.product.price * self.quantity


# application/services/order_service.py (нестабильный - зависит от core)
from core.domain.entities import Product, OrderItem
from infrastructure.database.repository import OrderRepository

class OrderService:
    def __init__(self, repository: OrderRepository):
        self.repository = repository

    def calculate_total(self, items: list[OrderItem]) -> Decimal:
        return sum(item.total for item in items)
```

### SAP — Stable Abstractions Principle

**Принцип стабильных абстракций**

> "Стабильность компонента должна соответствовать его абстрактности"

Стабильные компоненты должны быть абстрактными (содержать интерфейсы), чтобы их стабильность не мешала расширению. Нестабильные компоненты должны быть конкретными (содержать реализации).

**Метрика абстрактности A:**
```
A = Количество абстрактных классов / Общее количество классов

- A = 0: полностью конкретный компонент
- A = 1: полностью абстрактный компонент
```

**Главная последовательность (Main Sequence):**
```
Идеальное соотношение: A + I = 1

Зона боли (Zone of Pain): A ≈ 0, I ≈ 0
  - Конкретный и стабильный
  - Сложно изменять, много зависимостей

Зона бесполезности (Zone of Uselessness): A ≈ 1, I ≈ 1
  - Абстрактный и нестабильный
  - Никто не использует
```

```python
# Пример соблюдения SAP

# core/interfaces/ (A = 1, I = 0 - стабильные абстракции)
from abc import ABC, abstractmethod
from typing import Optional, List

class UserRepository(ABC):
    """Абстрактный репозиторий пользователей."""

    @abstractmethod
    def find_by_id(self, user_id: int) -> Optional["User"]:
        pass

    @abstractmethod
    def find_by_email(self, email: str) -> Optional["User"]:
        pass

    @abstractmethod
    def save(self, user: "User") -> "User":
        pass

    @abstractmethod
    def delete(self, user_id: int) -> bool:
        pass


class NotificationService(ABC):
    """Абстрактный сервис уведомлений."""

    @abstractmethod
    def send(self, recipient: str, message: str) -> bool:
        pass


# infrastructure/persistence/ (A = 0, I = 1 - нестабильные конкретные реализации)
from sqlalchemy.orm import Session
from core.interfaces import UserRepository, User

class SQLAlchemyUserRepository(UserRepository):
    """Конкретная реализация репозитория на SQLAlchemy."""

    def __init__(self, session: Session):
        self.session = session

    def find_by_id(self, user_id: int) -> Optional[User]:
        return self.session.query(UserModel).filter_by(id=user_id).first()

    def find_by_email(self, email: str) -> Optional[User]:
        return self.session.query(UserModel).filter_by(email=email).first()

    def save(self, user: User) -> User:
        self.session.add(user)
        self.session.commit()
        return user

    def delete(self, user_id: int) -> bool:
        user = self.find_by_id(user_id)
        if user:
            self.session.delete(user)
            self.session.commit()
            return True
        return False


# infrastructure/notifications/
from core.interfaces import NotificationService

class EmailNotificationService(NotificationService):
    """Отправка уведомлений по email."""

    def __init__(self, smtp_host: str, smtp_port: int):
        self.smtp_host = smtp_host
        self.smtp_port = smtp_port

    def send(self, recipient: str, message: str) -> bool:
        # Отправка email
        return True


class SMSNotificationService(NotificationService):
    """Отправка уведомлений по SMS."""

    def __init__(self, api_key: str):
        self.api_key = api_key

    def send(self, recipient: str, message: str) -> bool:
        # Отправка SMS
        return True
```

## Практический пример: структура проекта

```
ecommerce/
├── core/                           # Стабильный, абстрактный (I≈0, A≈0.8)
│   ├── __init__.py
│   ├── domain/
│   │   ├── entities/
│   │   │   ├── user.py
│   │   │   ├── product.py
│   │   │   └── order.py
│   │   └── value_objects/
│   │       ├── money.py
│   │       └── email.py
│   └── interfaces/
│       ├── repositories.py
│       └── services.py
│
├── application/                    # Средняя стабильность (I≈0.5, A≈0.3)
│   ├── __init__.py
│   ├── use_cases/
│   │   ├── create_order.py
│   │   ├── process_payment.py
│   │   └── send_notification.py
│   └── services/
│       ├── order_service.py
│       └── user_service.py
│
├── infrastructure/                 # Нестабильный, конкретный (I≈0.8, A≈0.1)
│   ├── __init__.py
│   ├── persistence/
│   │   ├── sqlalchemy/
│   │   │   ├── models.py
│   │   │   └── repositories.py
│   │   └── redis/
│   │       └── cache.py
│   ├── external/
│   │   ├── payment_gateway.py
│   │   └── email_provider.py
│   └── config/
│       └── settings.py
│
└── presentation/                   # Максимально нестабильный (I≈1, A≈0)
    ├── __init__.py
    ├── api/
    │   ├── routes.py
    │   └── schemas.py
    └── cli/
        └── commands.py
```

## Best Practices

### 1. Регулярный анализ зависимостей

```python
# Используйте инструменты для визуализации зависимостей
# pip install pydeps
# pydeps mypackage --max-bacon=2 -o dependencies.svg
```

### 2. Документирование публичного API компонента

```python
# mycomponent/__init__.py
"""
Компонент для работы с платежами.

Публичный API:
- PaymentGateway: основной класс для обработки платежей
- Transaction: представление транзакции
- PaymentError: базовое исключение

Пример использования:
    from mycomponent import PaymentGateway

    gateway = PaymentGateway(api_key="xxx")
    transaction = gateway.process(amount=100.0, card_token="tok_xxx")
"""

from .gateway import PaymentGateway
from .transaction import Transaction
from .exceptions import PaymentError

__all__ = ["PaymentGateway", "Transaction", "PaymentError"]
```

### 3. Версионирование API компонента

```python
# При изменении публичного API
# v1/ - старая версия (deprecated)
# v2/ - новая версия

# Или использование __version__
__version__ = "2.0.0"
```

## Типичные ошибки

### 1. Монолитные компоненты

```python
# Плохо: всё в одном огромном компоненте
# app/
#   models.py      # 50+ моделей
#   services.py    # 30+ сервисов
#   utils.py       # 100+ утилит
```

### 2. Циклические зависимости

```python
# Плохо
# a.py: from b import B
# b.py: from a import A

# Решение: вынести общие абстракции
```

### 3. Нестабильные зависимости стабильных компонентов

```python
# Плохо: core зависит от infrastructure
# core/domain/user.py
from infrastructure.database import get_session  # Нарушение SDP!
```

## Заключение

Принципы проектирования компонентов помогают создавать модульные, поддерживаемые системы. Ключевые моменты:

1. **REP**: Выпускайте связанные классы вместе
2. **CCP**: Группируйте классы, меняющиеся по одной причине
3. **CRP**: Не создавайте ненужных зависимостей
4. **ADP**: Избегайте циклических зависимостей
5. **SDP**: Зависьте от стабильного
6. **SAP**: Стабильное должно быть абстрактным

Баланс между этими принципами — ключ к хорошей архитектуре.
