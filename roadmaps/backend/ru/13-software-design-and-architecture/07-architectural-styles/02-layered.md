# Layered Architecture

## Что такое слоистая (многослойная) архитектура?

Слоистая архитектура (Layered Architecture), также известная как N-tier архитектура — это один из самых распространённых архитектурных паттернов, в котором приложение разделяется на горизонтальные слои, каждый из которых выполняет определённую роль. Каждый слой зависит только от слоя, находящегося непосредственно под ним.

> "Слоистая архитектура — это фундамент, на котором строится большинство корпоративных приложений." — Мартин Фаулер

## Основные слои

```
┌─────────────────────────────────────────────────────┐
│              Presentation Layer                     │
│         (UI, Controllers, Views, API)               │
├─────────────────────────────────────────────────────┤
│              Application Layer                      │
│         (Use Cases, Application Services)           │
├─────────────────────────────────────────────────────┤
│               Domain Layer                          │
│         (Business Logic, Entities, Rules)           │
├─────────────────────────────────────────────────────┤
│            Infrastructure Layer                     │
│      (Database, External Services, Framework)       │
└─────────────────────────────────────────────────────┘
```

## Классическая трёхслойная архитектура

### Presentation Layer (Слой представления)

Отвечает за взаимодействие с пользователем:
- Отображение данных
- Обработка пользовательского ввода
- Валидация входных данных
- Преобразование данных для отображения

### Business Logic Layer (Слой бизнес-логики)

Содержит основную логику приложения:
- Бизнес-правила
- Валидация бизнес-данных
- Вычисления и обработка
- Оркестрация операций

### Data Access Layer (Слой доступа к данным)

Отвечает за работу с данными:
- CRUD операции
- Запросы к базе данных
- Маппинг данных
- Управление транзакциями

## Пример реализации на Python

### Структура проекта

```
layered_app/
├── presentation/
│   ├── __init__.py
│   ├── api/
│   │   ├── __init__.py
│   │   ├── users.py
│   │   └── orders.py
│   └── schemas/
│       ├── __init__.py
│       ├── user_schema.py
│       └── order_schema.py
├── business/
│   ├── __init__.py
│   ├── services/
│   │   ├── __init__.py
│   │   ├── user_service.py
│   │   └── order_service.py
│   └── validators/
│       ├── __init__.py
│       └── order_validator.py
├── data_access/
│   ├── __init__.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── order.py
│   └── repositories/
│       ├── __init__.py
│       ├── user_repository.py
│       └── order_repository.py
├── infrastructure/
│   ├── __init__.py
│   ├── database.py
│   └── config.py
└── main.py
```

### Data Access Layer

```python
# data_access/models/user.py
from sqlalchemy import Column, Integer, String, DateTime, Boolean
from sqlalchemy.orm import relationship
from datetime import datetime
from infrastructure.database import Base

class UserModel(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String(255), unique=True, index=True, nullable=False)
    hashed_password = Column(String(255), nullable=False)
    full_name = Column(String(255))
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)

    orders = relationship("OrderModel", back_populates="user")
```

```python
# data_access/models/order.py
from sqlalchemy import Column, Integer, Float, ForeignKey, DateTime, String
from sqlalchemy.orm import relationship
from datetime import datetime
from infrastructure.database import Base

class OrderModel(Base):
    __tablename__ = "orders"

    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    total_amount = Column(Float, nullable=False)
    status = Column(String(50), default="pending")
    created_at = Column(DateTime, default=datetime.utcnow)

    user = relationship("UserModel", back_populates="orders")
    items = relationship("OrderItemModel", back_populates="order")
```

```python
# data_access/repositories/base_repository.py
from typing import Generic, TypeVar, Type, Optional, List
from sqlalchemy.orm import Session
from infrastructure.database import Base

ModelType = TypeVar("ModelType", bound=Base)

class BaseRepository(Generic[ModelType]):
    """Базовый репозиторий с CRUD операциями."""

    def __init__(self, model: Type[ModelType], db: Session):
        self.model = model
        self.db = db

    def get_by_id(self, id: int) -> Optional[ModelType]:
        return self.db.query(self.model).filter(self.model.id == id).first()

    def get_all(self, skip: int = 0, limit: int = 100) -> List[ModelType]:
        return self.db.query(self.model).offset(skip).limit(limit).all()

    def create(self, obj_data: dict) -> ModelType:
        db_obj = self.model(**obj_data)
        self.db.add(db_obj)
        self.db.commit()
        self.db.refresh(db_obj)
        return db_obj

    def update(self, id: int, obj_data: dict) -> Optional[ModelType]:
        db_obj = self.get_by_id(id)
        if db_obj:
            for key, value in obj_data.items():
                setattr(db_obj, key, value)
            self.db.commit()
            self.db.refresh(db_obj)
        return db_obj

    def delete(self, id: int) -> bool:
        db_obj = self.get_by_id(id)
        if db_obj:
            self.db.delete(db_obj)
            self.db.commit()
            return True
        return False
```

```python
# data_access/repositories/user_repository.py
from typing import Optional
from sqlalchemy.orm import Session
from data_access.models.user import UserModel
from data_access.repositories.base_repository import BaseRepository

class UserRepository(BaseRepository[UserModel]):
    def __init__(self, db: Session):
        super().__init__(UserModel, db)

    def get_by_email(self, email: str) -> Optional[UserModel]:
        return self.db.query(UserModel).filter(
            UserModel.email == email
        ).first()

    def get_active_users(self):
        return self.db.query(UserModel).filter(
            UserModel.is_active == True
        ).all()
```

```python
# data_access/repositories/order_repository.py
from typing import List
from sqlalchemy.orm import Session
from data_access.models.order import OrderModel
from data_access.repositories.base_repository import BaseRepository

class OrderRepository(BaseRepository[OrderModel]):
    def __init__(self, db: Session):
        super().__init__(OrderModel, db)

    def get_by_user_id(self, user_id: int) -> List[OrderModel]:
        return self.db.query(OrderModel).filter(
            OrderModel.user_id == user_id
        ).all()

    def get_by_status(self, status: str) -> List[OrderModel]:
        return self.db.query(OrderModel).filter(
            OrderModel.status == status
        ).all()

    def get_pending_orders(self) -> List[OrderModel]:
        return self.get_by_status("pending")
```

### Business Logic Layer

```python
# business/services/user_service.py
from typing import Optional, List
from dataclasses import dataclass
from datetime import datetime

@dataclass
class User:
    """Доменная сущность User."""
    id: Optional[int] = None
    email: str = ""
    full_name: str = ""
    is_active: bool = True
    created_at: Optional[datetime] = None

class UserService:
    """Сервис бизнес-логики для пользователей."""

    def __init__(self, user_repository):
        self.user_repository = user_repository

    def get_user(self, user_id: int) -> Optional[User]:
        user_model = self.user_repository.get_by_id(user_id)
        if user_model:
            return self._to_domain(user_model)
        return None

    def get_user_by_email(self, email: str) -> Optional[User]:
        user_model = self.user_repository.get_by_email(email)
        if user_model:
            return self._to_domain(user_model)
        return None

    def create_user(self, email: str, password: str, full_name: str) -> User:
        # Бизнес-правило: проверка уникальности email
        existing_user = self.user_repository.get_by_email(email)
        if existing_user:
            raise ValueError("User with this email already exists")

        # Бизнес-правило: валидация email
        if not self._is_valid_email(email):
            raise ValueError("Invalid email format")

        # Хеширование пароля (бизнес-логика)
        hashed_password = self._hash_password(password)

        user_data = {
            "email": email,
            "hashed_password": hashed_password,
            "full_name": full_name,
            "is_active": True
        }

        user_model = self.user_repository.create(user_data)
        return self._to_domain(user_model)

    def deactivate_user(self, user_id: int) -> Optional[User]:
        """Деактивация пользователя (мягкое удаление)."""
        user_model = self.user_repository.update(
            user_id,
            {"is_active": False}
        )
        if user_model:
            return self._to_domain(user_model)
        return None

    def _is_valid_email(self, email: str) -> bool:
        import re
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return re.match(pattern, email) is not None

    def _hash_password(self, password: str) -> str:
        import hashlib
        return hashlib.sha256(password.encode()).hexdigest()

    def _to_domain(self, model) -> User:
        return User(
            id=model.id,
            email=model.email,
            full_name=model.full_name,
            is_active=model.is_active,
            created_at=model.created_at
        )
```

```python
# business/services/order_service.py
from typing import Optional, List
from dataclasses import dataclass
from datetime import datetime
from decimal import Decimal

@dataclass
class OrderItem:
    product_id: int
    quantity: int
    price: float

@dataclass
class Order:
    """Доменная сущность Order."""
    id: Optional[int] = None
    user_id: int = 0
    items: List[OrderItem] = None
    total_amount: float = 0.0
    status: str = "pending"
    created_at: Optional[datetime] = None

class OrderService:
    """Сервис бизнес-логики для заказов."""

    # Бизнес-константы
    MIN_ORDER_AMOUNT = 10.0
    MAX_ITEMS_PER_ORDER = 50
    VALID_STATUSES = ["pending", "confirmed", "shipped", "delivered", "cancelled"]

    def __init__(self, order_repository, user_repository, product_repository):
        self.order_repository = order_repository
        self.user_repository = user_repository
        self.product_repository = product_repository

    def create_order(self, user_id: int, items: List[dict]) -> Order:
        """Создание заказа с применением бизнес-правил."""
        # Бизнес-правило 1: Проверка существования пользователя
        user = self.user_repository.get_by_id(user_id)
        if not user:
            raise ValueError("User not found")

        # Бизнес-правило 2: Проверка активности пользователя
        if not user.is_active:
            raise ValueError("User account is deactivated")

        # Бизнес-правило 3: Ограничение количества товаров
        if len(items) > self.MAX_ITEMS_PER_ORDER:
            raise ValueError(f"Maximum {self.MAX_ITEMS_PER_ORDER} items per order")

        # Бизнес-правило 4: Расчёт суммы заказа
        total_amount = self._calculate_total(items)

        # Бизнес-правило 5: Минимальная сумма заказа
        if total_amount < self.MIN_ORDER_AMOUNT:
            raise ValueError(f"Minimum order amount is {self.MIN_ORDER_AMOUNT}")

        # Бизнес-правило 6: Проверка наличия товаров
        self._validate_stock(items)

        order_data = {
            "user_id": user_id,
            "total_amount": total_amount,
            "status": "pending"
        }

        order_model = self.order_repository.create(order_data)
        return self._to_domain(order_model)

    def update_status(self, order_id: int, new_status: str) -> Order:
        """Обновление статуса заказа с валидацией переходов."""
        # Бизнес-правило: Валидация статуса
        if new_status not in self.VALID_STATUSES:
            raise ValueError(f"Invalid status: {new_status}")

        order = self.order_repository.get_by_id(order_id)
        if not order:
            raise ValueError("Order not found")

        # Бизнес-правило: Валидация перехода статусов
        if not self._is_valid_status_transition(order.status, new_status):
            raise ValueError(
                f"Cannot transition from {order.status} to {new_status}"
            )

        updated_order = self.order_repository.update(
            order_id,
            {"status": new_status}
        )
        return self._to_domain(updated_order)

    def _is_valid_status_transition(self, current: str, new: str) -> bool:
        """Матрица допустимых переходов статусов."""
        allowed_transitions = {
            "pending": ["confirmed", "cancelled"],
            "confirmed": ["shipped", "cancelled"],
            "shipped": ["delivered"],
            "delivered": [],
            "cancelled": []
        }
        return new in allowed_transitions.get(current, [])

    def _calculate_total(self, items: List[dict]) -> float:
        total = 0.0
        for item in items:
            product = self.product_repository.get_by_id(item["product_id"])
            if product:
                total += product.price * item["quantity"]
        return total

    def _validate_stock(self, items: List[dict]) -> None:
        for item in items:
            product = self.product_repository.get_by_id(item["product_id"])
            if not product:
                raise ValueError(f"Product {item['product_id']} not found")
            if product.stock < item["quantity"]:
                raise ValueError(f"Insufficient stock for product {product.name}")

    def _to_domain(self, model) -> Order:
        return Order(
            id=model.id,
            user_id=model.user_id,
            total_amount=model.total_amount,
            status=model.status,
            created_at=model.created_at
        )
```

### Presentation Layer

```python
# presentation/schemas/user_schema.py
from pydantic import BaseModel, EmailStr
from datetime import datetime
from typing import Optional

class UserCreate(BaseModel):
    email: EmailStr
    password: str
    full_name: str

class UserResponse(BaseModel):
    id: int
    email: str
    full_name: str
    is_active: bool
    created_at: datetime

    class Config:
        from_attributes = True

class UserUpdate(BaseModel):
    full_name: Optional[str] = None
```

```python
# presentation/schemas/order_schema.py
from pydantic import BaseModel, validator
from datetime import datetime
from typing import List, Optional

class OrderItemCreate(BaseModel):
    product_id: int
    quantity: int

    @validator('quantity')
    def quantity_must_be_positive(cls, v):
        if v <= 0:
            raise ValueError('Quantity must be positive')
        return v

class OrderCreate(BaseModel):
    items: List[OrderItemCreate]

    @validator('items')
    def items_not_empty(cls, v):
        if not v:
            raise ValueError('Order must have at least one item')
        return v

class OrderResponse(BaseModel):
    id: int
    user_id: int
    total_amount: float
    status: str
    created_at: datetime

    class Config:
        from_attributes = True

class OrderStatusUpdate(BaseModel):
    status: str
```

```python
# presentation/api/users.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
from presentation.schemas.user_schema import UserCreate, UserResponse, UserUpdate
from business.services.user_service import UserService
from data_access.repositories.user_repository import UserRepository
from infrastructure.database import get_db

router = APIRouter(prefix="/users", tags=["users"])

def get_user_service(db: Session = Depends(get_db)) -> UserService:
    """Фабрика для создания UserService с зависимостями."""
    user_repository = UserRepository(db)
    return UserService(user_repository)

@router.post("/", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
def create_user(
    user_data: UserCreate,
    service: UserService = Depends(get_user_service)
):
    """Создание нового пользователя."""
    try:
        user = service.create_user(
            email=user_data.email,
            password=user_data.password,
            full_name=user_data.full_name
        )
        return user
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.get("/{user_id}", response_model=UserResponse)
def get_user(
    user_id: int,
    service: UserService = Depends(get_user_service)
):
    """Получение пользователя по ID."""
    user = service.get_user(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
def deactivate_user(
    user_id: int,
    service: UserService = Depends(get_user_service)
):
    """Деактивация пользователя."""
    result = service.deactivate_user(user_id)
    if not result:
        raise HTTPException(status_code=404, detail="User not found")
```

```python
# presentation/api/orders.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
from typing import List
from presentation.schemas.order_schema import (
    OrderCreate, OrderResponse, OrderStatusUpdate
)
from business.services.order_service import OrderService
from data_access.repositories.order_repository import OrderRepository
from data_access.repositories.user_repository import UserRepository
from data_access.repositories.product_repository import ProductRepository
from infrastructure.database import get_db

router = APIRouter(prefix="/orders", tags=["orders"])

def get_order_service(db: Session = Depends(get_db)) -> OrderService:
    """Фабрика для создания OrderService с зависимостями."""
    return OrderService(
        order_repository=OrderRepository(db),
        user_repository=UserRepository(db),
        product_repository=ProductRepository(db)
    )

@router.post("/", response_model=OrderResponse, status_code=status.HTTP_201_CREATED)
def create_order(
    user_id: int,
    order_data: OrderCreate,
    service: OrderService = Depends(get_order_service)
):
    """Создание нового заказа."""
    try:
        order = service.create_order(
            user_id=user_id,
            items=[item.dict() for item in order_data.items]
        )
        return order
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.patch("/{order_id}/status", response_model=OrderResponse)
def update_order_status(
    order_id: int,
    status_data: OrderStatusUpdate,
    service: OrderService = Depends(get_order_service)
):
    """Обновление статуса заказа."""
    try:
        order = service.update_status(order_id, status_data.status)
        return order
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
```

## Правила взаимодействия между слоями

```
┌─────────────────────────────────────────────────────────┐
│                    Presentation                         │
│    Знает о: Business Layer                              │
│    НЕ знает о: Data Access Layer                        │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    Business Logic                       │
│    Знает о: Data Access Layer (через интерфейсы)        │
│    НЕ знает о: Presentation Layer                       │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    Data Access                          │
│    Знает о: Infrastructure (Database)                   │
│    НЕ знает о: Business, Presentation                   │
└─────────────────────────────────────────────────────────┘
```

### Закрытые слои (Closed Layers)

```python
# Каждый запрос должен пройти через все слои
# Presentation -> Business -> Data Access

# ПРАВИЛЬНО: Presentation вызывает Business
@router.get("/users/{user_id}")
def get_user(user_id: int, service: UserService = Depends(get_user_service)):
    return service.get_user(user_id)

# НЕПРАВИЛЬНО: Presentation напрямую вызывает Data Access
@router.get("/users/{user_id}")
def get_user(user_id: int, db: Session = Depends(get_db)):
    return db.query(UserModel).filter(UserModel.id == user_id).first()
```

### Открытые слои (Open Layers)

```python
# Иногда допускается пропуск слоёв для утилитных операций

# Слой утилит - открытый, может использоваться любым слоем
class DateUtils:
    @staticmethod
    def format_date(date: datetime) -> str:
        return date.strftime("%Y-%m-%d")

# Используется и в Presentation, и в Business
```

## Преимущества слоистой архитектуры

### 1. Разделение ответственности

```python
# Каждый слой отвечает за своё

# Presentation - только преобразование данных
class UserResponse(BaseModel):
    id: int
    email: str
    display_name: str  # Форматирование для отображения

# Business - только бизнес-логика
class UserService:
    def can_place_order(self, user_id: int) -> bool:
        user = self.get_user(user_id)
        return user.is_active and user.has_valid_payment_method

# Data Access - только работа с данными
class UserRepository:
    def get_by_id(self, user_id: int):
        return self.db.query(UserModel).filter(UserModel.id == user_id).first()
```

### 2. Простота тестирования

```python
# Каждый слой тестируется независимо

# Тест Business Layer с моком Data Access
def test_create_order_with_inactive_user():
    mock_user_repo = Mock()
    mock_user_repo.get_by_id.return_value = UserModel(is_active=False)

    service = OrderService(
        order_repository=Mock(),
        user_repository=mock_user_repo,
        product_repository=Mock()
    )

    with pytest.raises(ValueError, match="User account is deactivated"):
        service.create_order(user_id=1, items=[{"product_id": 1, "quantity": 1}])
```

### 3. Простота замены реализации

```python
# Можно заменить репозиторий без изменения бизнес-логики

class UserRepositoryInterface(ABC):
    @abstractmethod
    def get_by_id(self, user_id: int) -> Optional[UserModel]:
        pass

# PostgreSQL реализация
class PostgresUserRepository(UserRepositoryInterface):
    def get_by_id(self, user_id: int):
        return self.db.query(UserModel).filter(UserModel.id == user_id).first()

# MongoDB реализация
class MongoUserRepository(UserRepositoryInterface):
    def get_by_id(self, user_id: int):
        return self.collection.find_one({"_id": user_id})

# Сервис работает с любой реализацией
class UserService:
    def __init__(self, user_repository: UserRepositoryInterface):
        self.user_repository = user_repository
```

## Недостатки слоистой архитектуры

### 1. Sinkhole Anti-pattern

```python
# Проблема: Слои просто передают данные без обработки

# Presentation Layer
@router.get("/users/{user_id}")
def get_user(user_id: int, service: UserService = Depends(get_user_service)):
    return service.get_user(user_id)  # Просто передаёт дальше

# Business Layer
class UserService:
    def get_user(self, user_id: int):
        return self.repository.get_by_id(user_id)  # Просто передаёт дальше

# Решение: Если >80% запросов - sinkhole, пересмотрите архитектуру
```

### 2. Жёсткая связность между слоями

```python
# Изменение в одном слое может потребовать изменений в других

# Добавили поле в модель
class UserModel(Base):
    # ...
    phone_number = Column(String)  # Новое поле

# Нужно обновить все слои:
# - Data Access: UserRepository
# - Business: UserService, User domain entity
# - Presentation: UserResponse, UserCreate schemas
```

## Варианты слоистой архитектуры

### 3-Layer Architecture

```
┌─────────────────┐
│  Presentation   │
├─────────────────┤
│ Business Logic  │
├─────────────────┤
│  Data Access    │
└─────────────────┘
```

### 4-Layer Architecture (с Application Layer)

```
┌─────────────────┐
│  Presentation   │
├─────────────────┤
│   Application   │  <- Use Cases, Orchestration
├─────────────────┤
│     Domain      │  <- Business Rules
├─────────────────┤
│ Infrastructure  │
└─────────────────┘
```

### N-Tier Architecture

```
┌─────────────────┐
│   Client Tier   │  <- Browser, Mobile App
├─────────────────┤
│ Presentation    │  <- Web Server
├─────────────────┤
│   Application   │  <- App Server
├─────────────────┤
│     Data        │  <- Database Server
└─────────────────┘
```

## Best Practices

### 1. Используйте Dependency Injection

```python
# Внедрение зависимостей для гибкости

class OrderService:
    def __init__(
        self,
        order_repository: OrderRepositoryInterface,
        user_repository: UserRepositoryInterface,
        notification_service: NotificationServiceInterface
    ):
        self.order_repository = order_repository
        self.user_repository = user_repository
        self.notification_service = notification_service
```

### 2. Определите чёткие интерфейсы между слоями

```python
# Интерфейсы определяют контракт между слоями

from abc import ABC, abstractmethod

class OrderServiceInterface(ABC):
    @abstractmethod
    def create_order(self, user_id: int, items: List[dict]) -> Order:
        pass

    @abstractmethod
    def get_order(self, order_id: int) -> Optional[Order]:
        pass

    @abstractmethod
    def update_status(self, order_id: int, status: str) -> Order:
        pass
```

### 3. Избегайте утечки абстракций

```python
# ПЛОХО: Бизнес-слой зависит от деталей Data Access
class UserService:
    def get_users(self, skip: int, limit: int):
        # Знает о пагинации SQLAlchemy
        return self.repository.db.query(UserModel).offset(skip).limit(limit).all()

# ХОРОШО: Бизнес-слой работает с абстракциями
class UserService:
    def get_users(self, page: int, page_size: int):
        return self.repository.get_paginated(page, page_size)
```

## Когда использовать?

### Подходит для:

1. **Корпоративных приложений** со сложной бизнес-логикой
2. **Команд с разделением ролей** (frontend/backend/DBA)
3. **Долгосрочных проектов** с предсказуемым развитием
4. **Систем с чёткими требованиями** к разделению ответственности

### Не подходит для:

1. **Простых CRUD приложений** (избыточная сложность)
2. **Микросервисов** (предпочтительнее вертикальная декомпозиция)
3. **Высоконагруженных систем** (дополнительные накладные расходы)
4. **Прототипов и MVP** (слишком много boilerplate)

## Заключение

Слоистая архитектура — это проверенный временем паттерн, который обеспечивает чёткое разделение ответственности и упрощает поддержку кода. Однако важно помнить о потенциальных проблемах, таких как sinkhole anti-pattern, и применять этот паттерн осознанно, исходя из реальных потребностей проекта.
