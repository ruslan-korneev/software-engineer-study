# Use Cases / Interactors

## Что такое Use Case?

**Use Case (Вариант использования)** или **Interactor** — это паттерн, в котором каждая бизнес-операция инкапсулируется в отдельный класс. Use Case координирует работу доменных объектов и инфраструктуры для выполнения одного конкретного сценария использования системы.

> Концепция Use Cases/Interactors популяризирована Робертом Мартином (Uncle Bob) в книге "Clean Architecture" и является ключевым элементом Application Layer.

## Основная идея

Use Case — это оркестратор, который знает "что" делать, но не "как":

```
                 Presentation Layer
                        │
                        v
               ┌────────────────┐
               │    Use Case    │
               │   (Interactor) │
               └────────────────┘
                   /    │    \
                  /     │     \
                 v      v      v
           ┌─────┐  ┌─────┐  ┌─────────┐
           │Repo │  │Entity│ │Service  │
           └─────┘  └─────┘  └─────────┘
                 Domain Layer
```

## Зачем нужны Use Cases?

### Основные преимущества:

1. **Один класс — одна операция** — ясная ответственность
2. **Тестируемость** — легко тестировать изолированно
3. **Читаемость** — название класса описывает операцию
4. **Независимость** — не зависит от фреймворка
5. **Переиспользование** — можно вызывать из разных точек

## Базовая реализация

### Структура Use Case

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Generic, TypeVar

# Типы для входных и выходных данных
InputDTO = TypeVar("InputDTO")
OutputDTO = TypeVar("OutputDTO")


class UseCase(ABC, Generic[InputDTO, OutputDTO]):
    """Базовый абстрактный Use Case."""

    @abstractmethod
    def execute(self, input_dto: InputDTO) -> OutputDTO:
        """Выполнение use case."""
        raise NotImplementedError


# Альтернатива: протокол с __call__
class CallableUseCase(ABC, Generic[InputDTO, OutputDTO]):
    """Use Case как callable объект."""

    @abstractmethod
    def __call__(self, input_dto: InputDTO) -> OutputDTO:
        raise NotImplementedError
```

### Пример: Регистрация пользователя

```python
from dataclasses import dataclass
from typing import Optional
from datetime import datetime
import hashlib


# Input DTO
@dataclass
class RegisterUserInput:
    username: str
    email: str
    password: str


# Output DTO
@dataclass
class RegisterUserOutput:
    success: bool
    user_id: Optional[int] = None
    message: str = ""


# Порты (интерфейсы)
class UserRepository(ABC):
    @abstractmethod
    def find_by_email(self, email: str) -> Optional[User]:
        pass

    @abstractmethod
    def find_by_username(self, username: str) -> Optional[User]:
        pass

    @abstractmethod
    def save(self, user: User) -> User:
        pass


class EmailService(ABC):
    @abstractmethod
    def send_welcome_email(self, user: User) -> None:
        pass


# Use Case
class RegisterUserUseCase(UseCase[RegisterUserInput, RegisterUserOutput]):
    """Use Case: Регистрация нового пользователя."""

    def __init__(
        self,
        user_repository: UserRepository,
        email_service: EmailService
    ):
        self.user_repository = user_repository
        self.email_service = email_service

    def execute(self, input_dto: RegisterUserInput) -> RegisterUserOutput:
        # 1. Валидация
        if len(input_dto.username) < 3:
            return RegisterUserOutput(
                success=False,
                message="Username must be at least 3 characters"
            )

        if len(input_dto.password) < 8:
            return RegisterUserOutput(
                success=False,
                message="Password must be at least 8 characters"
            )

        # 2. Проверка уникальности
        existing_user = self.user_repository.find_by_email(input_dto.email)
        if existing_user:
            return RegisterUserOutput(
                success=False,
                message="Email already registered"
            )

        existing_user = self.user_repository.find_by_username(input_dto.username)
        if existing_user:
            return RegisterUserOutput(
                success=False,
                message="Username already taken"
            )

        # 3. Создание пользователя
        password_hash = hashlib.sha256(input_dto.password.encode()).hexdigest()
        user = User(
            username=input_dto.username,
            email=input_dto.email,
            password_hash=password_hash,
            is_active=True,
            created_at=datetime.now()
        )

        # 4. Сохранение
        saved_user = self.user_repository.save(user)

        # 5. Отправка email
        try:
            self.email_service.send_welcome_email(saved_user)
        except Exception:
            # Логируем ошибку, но не откатываем регистрацию
            pass

        return RegisterUserOutput(
            success=True,
            user_id=saved_user.id,
            message="User registered successfully"
        )
```

## Use Cases с Result Type

```python
from dataclasses import dataclass
from typing import TypeVar, Generic, Optional, Union

T = TypeVar("T")
E = TypeVar("E")


@dataclass
class Success(Generic[T]):
    value: T


@dataclass
class Failure(Generic[E]):
    error: E


Result = Union[Success[T], Failure[E]]


# Использование в Use Case
@dataclass
class CreateOrderError:
    code: str
    message: str


class CreateOrderUseCase:
    def __init__(
        self,
        order_repository: OrderRepository,
        product_repository: ProductRepository,
        user_repository: UserRepository
    ):
        self.order_repo = order_repository
        self.product_repo = product_repository
        self.user_repo = user_repository

    def execute(self, input_dto: CreateOrderInput) -> Result[Order, CreateOrderError]:
        # Валидация пользователя
        user = self.user_repo.find_by_id(input_dto.user_id)
        if not user:
            return Failure(CreateOrderError(
                code="USER_NOT_FOUND",
                message="User not found"
            ))

        if not user.is_active:
            return Failure(CreateOrderError(
                code="USER_INACTIVE",
                message="User is inactive"
            ))

        # Создание заказа
        order = Order(user_id=user.id, status="pending")

        for item in input_dto.items:
            product = self.product_repo.find_by_id(item.product_id)
            if not product:
                return Failure(CreateOrderError(
                    code="PRODUCT_NOT_FOUND",
                    message=f"Product {item.product_id} not found"
                ))

            if product.stock < item.quantity:
                return Failure(CreateOrderError(
                    code="INSUFFICIENT_STOCK",
                    message=f"Insufficient stock for {product.name}"
                ))

            order.add_item(product, item.quantity)
            product.decrease_stock(item.quantity)

        saved_order = self.order_repo.save(order)
        return Success(saved_order)


# Использование
result = use_case.execute(input_dto)

if isinstance(result, Success):
    print(f"Order created: {result.value.id}")
else:
    print(f"Error: {result.error.message}")
```

## Use Case с Unit of Work

```python
from typing import Protocol


class UnitOfWork(Protocol):
    users: UserRepository
    orders: OrderRepository
    products: ProductRepository

    def __enter__(self) -> "UnitOfWork":
        ...

    def __exit__(self, exc_type, exc_val, exc_tb) -> None:
        ...

    def commit(self) -> None:
        ...

    def rollback(self) -> None:
        ...


class PlaceOrderUseCase:
    """Use Case с Unit of Work для транзакционности."""

    def __init__(self, uow: UnitOfWork):
        self.uow = uow

    def execute(self, input_dto: PlaceOrderInput) -> PlaceOrderOutput:
        with self.uow:
            # Все операции в одной транзакции
            user = self.uow.users.find_by_id(input_dto.user_id)
            if not user:
                return PlaceOrderOutput(success=False, error="User not found")

            order = Order(user_id=user.id)

            for item in input_dto.items:
                product = self.uow.products.find_by_id(item.product_id)
                if not product:
                    return PlaceOrderOutput(
                        success=False,
                        error=f"Product {item.product_id} not found"
                    )

                order.add_item(product, item.quantity)
                product.stock -= item.quantity

            self.uow.orders.add(order)
            self.uow.commit()

            return PlaceOrderOutput(success=True, order_id=order.id)
```

## CQRS: Разделение команд и запросов

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Generic, TypeVar, List

# Команды (изменяют состояние)
class Command(ABC):
    pass


class CommandHandler(ABC, Generic[InputDTO]):
    @abstractmethod
    def handle(self, command: InputDTO) -> None:
        pass


# Запросы (только чтение)
class Query(ABC):
    pass


class QueryHandler(ABC, Generic[InputDTO, OutputDTO]):
    @abstractmethod
    def handle(self, query: InputDTO) -> OutputDTO:
        pass


# Пример команды
@dataclass
class CreateUserCommand(Command):
    username: str
    email: str
    password: str


class CreateUserHandler(CommandHandler[CreateUserCommand]):
    def __init__(self, user_repository: UserRepository):
        self.user_repository = user_repository

    def handle(self, command: CreateUserCommand) -> None:
        user = User(
            username=command.username,
            email=command.email,
            password_hash=self._hash_password(command.password)
        )
        self.user_repository.save(user)

    def _hash_password(self, password: str) -> str:
        import hashlib
        return hashlib.sha256(password.encode()).hexdigest()


# Пример запроса
@dataclass
class GetUserByIdQuery(Query):
    user_id: int


@dataclass
class UserDTO:
    id: int
    username: str
    email: str


class GetUserByIdHandler(QueryHandler[GetUserByIdQuery, UserDTO]):
    def __init__(self, user_repository: UserRepository):
        self.user_repository = user_repository

    def handle(self, query: GetUserByIdQuery) -> UserDTO:
        user = self.user_repository.find_by_id(query.user_id)
        if not user:
            raise UserNotFoundError(query.user_id)
        return UserDTO(
            id=user.id,
            username=user.username,
            email=user.email
        )
```

## Use Cases в FastAPI

```python
from fastapi import FastAPI, Depends, HTTPException
from pydantic import BaseModel
from typing import List, Optional

app = FastAPI()


# Pydantic модели для API
class CreateOrderRequest(BaseModel):
    user_id: int
    items: List[OrderItemRequest]


class OrderItemRequest(BaseModel):
    product_id: int
    quantity: int


class CreateOrderResponse(BaseModel):
    order_id: int
    total: float
    status: str


# Dependency Injection
def get_create_order_use_case() -> CreateOrderUseCase:
    """Фабрика для Use Case с зависимостями."""
    session = get_db_session()
    uow = SQLAlchemyUnitOfWork(session)
    return CreateOrderUseCase(uow)


# Endpoint
@app.post("/orders", response_model=CreateOrderResponse)
def create_order(
    request: CreateOrderRequest,
    use_case: CreateOrderUseCase = Depends(get_create_order_use_case)
):
    """Создание заказа через Use Case."""
    input_dto = CreateOrderInput(
        user_id=request.user_id,
        items=[
            OrderItemInput(product_id=item.product_id, quantity=item.quantity)
            for item in request.items
        ]
    )

    result = use_case.execute(input_dto)

    if isinstance(result, Failure):
        raise HTTPException(
            status_code=400,
            detail=result.error.message
        )

    order = result.value
    return CreateOrderResponse(
        order_id=order.id,
        total=order.total,
        status=order.status
    )
```

## Организация Use Cases

### Структура проекта

```
application/
├── use_cases/
│   ├── __init__.py
│   ├── base.py           # Базовые классы
│   ├── users/
│   │   ├── __init__.py
│   │   ├── register_user.py
│   │   ├── login_user.py
│   │   ├── update_profile.py
│   │   └── deactivate_user.py
│   ├── orders/
│   │   ├── __init__.py
│   │   ├── create_order.py
│   │   ├── cancel_order.py
│   │   ├── ship_order.py
│   │   └── get_order.py
│   └── products/
│       ├── __init__.py
│       ├── create_product.py
│       ├── update_stock.py
│       └── search_products.py
├── ports/
│   ├── __init__.py
│   ├── repositories.py   # Интерфейсы репозиториев
│   └── services.py       # Интерфейсы внешних сервисов
└── dto/
    ├── __init__.py
    ├── user_dto.py
    └── order_dto.py
```

### Композитный Use Case

```python
from dataclasses import dataclass
from typing import List


@dataclass
class CheckoutInput:
    user_id: int
    cart_id: int
    payment_method: str
    shipping_address_id: int


@dataclass
class CheckoutOutput:
    success: bool
    order_id: Optional[int] = None
    payment_id: Optional[str] = None
    error: Optional[str] = None


class CheckoutUseCase:
    """
    Композитный Use Case, объединяющий несколько операций.
    """

    def __init__(
        self,
        create_order_use_case: CreateOrderUseCase,
        process_payment_use_case: ProcessPaymentUseCase,
        send_confirmation_use_case: SendConfirmationUseCase,
        uow: UnitOfWork
    ):
        self.create_order = create_order_use_case
        self.process_payment = process_payment_use_case
        self.send_confirmation = send_confirmation_use_case
        self.uow = uow

    def execute(self, input_dto: CheckoutInput) -> CheckoutOutput:
        with self.uow:
            # 1. Создаём заказ из корзины
            order_result = self.create_order.execute(
                CreateOrderFromCartInput(
                    user_id=input_dto.user_id,
                    cart_id=input_dto.cart_id,
                    shipping_address_id=input_dto.shipping_address_id
                )
            )

            if not order_result.success:
                return CheckoutOutput(success=False, error=order_result.error)

            order = order_result.order

            # 2. Обрабатываем оплату
            payment_result = self.process_payment.execute(
                ProcessPaymentInput(
                    order_id=order.id,
                    amount=order.total,
                    method=input_dto.payment_method
                )
            )

            if not payment_result.success:
                # Откатываем заказ
                order.cancel()
                return CheckoutOutput(success=False, error=payment_result.error)

            # 3. Подтверждаем заказ
            order.confirm(payment_result.payment_id)
            self.uow.commit()

            # 4. Отправляем уведомление (асинхронно, не в транзакции)
            try:
                self.send_confirmation.execute(
                    SendConfirmationInput(order_id=order.id)
                )
            except Exception:
                # Логируем, но не откатываем
                pass

            return CheckoutOutput(
                success=True,
                order_id=order.id,
                payment_id=payment_result.payment_id
            )
```

## Тестирование Use Cases

```python
import pytest
from unittest.mock import Mock, MagicMock


class TestRegisterUserUseCase:
    def setup_method(self):
        self.user_repository = Mock(spec=UserRepository)
        self.email_service = Mock(spec=EmailService)
        self.use_case = RegisterUserUseCase(
            user_repository=self.user_repository,
            email_service=self.email_service
        )

    def test_successful_registration(self):
        # Arrange
        self.user_repository.find_by_email.return_value = None
        self.user_repository.find_by_username.return_value = None
        self.user_repository.save.return_value = User(
            id=1,
            username="alice",
            email="alice@example.com"
        )

        input_dto = RegisterUserInput(
            username="alice",
            email="alice@example.com",
            password="password123"
        )

        # Act
        result = self.use_case.execute(input_dto)

        # Assert
        assert result.success is True
        assert result.user_id == 1
        self.user_repository.save.assert_called_once()
        self.email_service.send_welcome_email.assert_called_once()

    def test_email_already_exists(self):
        # Arrange
        existing_user = User(id=1, email="alice@example.com")
        self.user_repository.find_by_email.return_value = existing_user

        input_dto = RegisterUserInput(
            username="alice",
            email="alice@example.com",
            password="password123"
        )

        # Act
        result = self.use_case.execute(input_dto)

        # Assert
        assert result.success is False
        assert "already registered" in result.message
        self.user_repository.save.assert_not_called()

    def test_password_too_short(self):
        # Arrange
        input_dto = RegisterUserInput(
            username="alice",
            email="alice@example.com",
            password="short"
        )

        # Act
        result = self.use_case.execute(input_dto)

        # Assert
        assert result.success is False
        assert "Password" in result.message
```

## Сравнение подходов

| Аспект | Use Case | Transaction Script | Service Layer |
|--------|----------|-------------------|---------------|
| **Один класс** | Одна операция | Много методов | Много методов |
| **Тестируемость** | Отличная | Средняя | Средняя |
| **SRP** | Соблюдается | Частично | Нарушается |
| **Сложность** | Выше | Ниже | Средняя |

## Плюсы и минусы

### Преимущества

1. **Читаемость** — название класса = бизнес-операция
2. **Тестируемость** — легко тестировать изолированно
3. **SRP** — один класс, одна ответственность
4. **Независимость** — не зависит от фреймворка
5. **Явные зависимости** — через конструктор

### Недостатки

1. **Много классов** — один класс на операцию
2. **Boilerplate** — много однотипного кода
3. **Сложность** — для простых случаев избыточно

## Когда использовать?

### Рекомендуется:
- Clean Architecture
- Сложные бизнес-операции
- Когда нужна высокая тестируемость
- Микросервисы
- Долгоживущие проекты

### Не рекомендуется:
- Простые CRUD-операции
- Прототипы и MVP
- Очень маленькие проекты

## Лучшие практики

1. **Один Use Case — одна операция** — не смешивайте
2. **Чёткие DTO** — вход и выход через DTO
3. **Dependency Injection** — инжектируйте зависимости
4. **Без побочных эффектов** — не зависьте от глобального состояния
5. **Тестируйте изолированно** — мокайте зависимости

## Заключение

Use Cases / Interactors — это мощный паттерн для организации бизнес-логики в сложных приложениях. Они обеспечивают ясное разделение ответственности, отличную тестируемость и независимость от фреймворка. В сочетании с Clean Architecture и DDD, Use Cases становятся центральным элементом Application Layer, координирующим работу всей системы.
