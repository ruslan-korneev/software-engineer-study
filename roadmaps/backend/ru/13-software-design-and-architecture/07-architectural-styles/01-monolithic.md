# Monolithic Architecture

## Что такое монолитная архитектура?

Монолитная архитектура (Monolithic Architecture) — это традиционный подход к проектированию программного обеспечения, при котором всё приложение разрабатывается, развёртывается и масштабируется как единое целое. Все компоненты системы — пользовательский интерфейс, бизнес-логика и доступ к данным — объединены в одном исполняемом файле или развёртываемом артефакте.

> "Монолит — это не антипаттерн. Это отправная точка для большинства систем." — Сэм Ньюман

## Структура монолитного приложения

```
┌─────────────────────────────────────────────────────┐
│                    МОНОЛИТ                          │
├─────────────────────────────────────────────────────┤
│  ┌───────────────────────────────────────────────┐  │
│  │           Presentation Layer                  │  │
│  │   (Web UI, REST API, CLI)                     │  │
│  └───────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────┐  │
│  │           Business Logic Layer                │  │
│  │   (Services, Domain Logic, Rules)             │  │
│  └───────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────┐  │
│  │           Data Access Layer                   │  │
│  │   (Repositories, ORM, Database Access)        │  │
│  └───────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────┤
│                   Database                          │
└─────────────────────────────────────────────────────┘
```

## Пример монолитного приложения на Python

### Структура проекта

```
ecommerce_monolith/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   ├── database.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── product.py
│   │   └── order.py
│   ├── services/
│   │   ├── __init__.py
│   │   ├── user_service.py
│   │   ├── product_service.py
│   │   ├── order_service.py
│   │   └── payment_service.py
│   ├── repositories/
│   │   ├── __init__.py
│   │   ├── user_repository.py
│   │   ├── product_repository.py
│   │   └── order_repository.py
│   └── api/
│       ├── __init__.py
│       ├── users.py
│       ├── products.py
│       └── orders.py
├── tests/
├── requirements.txt
└── Dockerfile
```

### Реализация

```python
# app/database.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base

DATABASE_URL = "postgresql://localhost:5432/ecommerce"

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

```python
# app/models/user.py
from sqlalchemy import Column, Integer, String, DateTime
from sqlalchemy.orm import relationship
from datetime import datetime
from app.database import Base

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True)
    hashed_password = Column(String)
    full_name = Column(String)
    created_at = Column(DateTime, default=datetime.utcnow)

    orders = relationship("Order", back_populates="user")
```

```python
# app/models/product.py
from sqlalchemy import Column, Integer, String, Float, Text
from app.database import Base

class Product(Base):
    __tablename__ = "products"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, index=True)
    description = Column(Text)
    price = Column(Float)
    stock_quantity = Column(Integer, default=0)
    category = Column(String, index=True)
```

```python
# app/models/order.py
from sqlalchemy import Column, Integer, Float, ForeignKey, DateTime, Enum
from sqlalchemy.orm import relationship
from datetime import datetime
import enum
from app.database import Base

class OrderStatus(enum.Enum):
    PENDING = "pending"
    PAID = "paid"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class Order(Base):
    __tablename__ = "orders"

    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id"))
    total_amount = Column(Float)
    status = Column(Enum(OrderStatus), default=OrderStatus.PENDING)
    created_at = Column(DateTime, default=datetime.utcnow)

    user = relationship("User", back_populates="orders")
    items = relationship("OrderItem", back_populates="order")
```

```python
# app/services/order_service.py
from typing import List, Optional
from sqlalchemy.orm import Session
from app.models.order import Order, OrderStatus
from app.models.product import Product
from app.services.payment_service import PaymentService

class OrderService:
    def __init__(self, db: Session):
        self.db = db
        self.payment_service = PaymentService()

    def create_order(self, user_id: int, items: List[dict]) -> Order:
        """Создание заказа с проверкой наличия товаров."""
        total_amount = 0.0

        # Проверяем наличие товаров и считаем сумму
        for item in items:
            product = self.db.query(Product).filter(
                Product.id == item["product_id"]
            ).first()

            if not product:
                raise ValueError(f"Product {item['product_id']} not found")

            if product.stock_quantity < item["quantity"]:
                raise ValueError(f"Insufficient stock for {product.name}")

            total_amount += product.price * item["quantity"]

        # Создаём заказ
        order = Order(
            user_id=user_id,
            total_amount=total_amount,
            status=OrderStatus.PENDING
        )
        self.db.add(order)

        # Обновляем остатки товаров
        for item in items:
            product = self.db.query(Product).filter(
                Product.id == item["product_id"]
            ).first()
            product.stock_quantity -= item["quantity"]

        self.db.commit()
        self.db.refresh(order)
        return order

    def process_payment(self, order_id: int, payment_info: dict) -> bool:
        """Обработка оплаты заказа."""
        order = self.db.query(Order).filter(Order.id == order_id).first()

        if not order:
            raise ValueError("Order not found")

        if order.status != OrderStatus.PENDING:
            raise ValueError("Order is not in pending status")

        # Обработка платежа
        payment_result = self.payment_service.process(
            amount=order.total_amount,
            payment_info=payment_info
        )

        if payment_result.success:
            order.status = OrderStatus.PAID
            self.db.commit()
            return True

        return False

    def get_user_orders(self, user_id: int) -> List[Order]:
        """Получение всех заказов пользователя."""
        return self.db.query(Order).filter(
            Order.user_id == user_id
        ).all()
```

```python
# app/api/orders.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List
from pydantic import BaseModel
from app.database import get_db
from app.services.order_service import OrderService

router = APIRouter(prefix="/orders", tags=["orders"])

class OrderItemCreate(BaseModel):
    product_id: int
    quantity: int

class OrderCreate(BaseModel):
    items: List[OrderItemCreate]

class OrderResponse(BaseModel):
    id: int
    user_id: int
    total_amount: float
    status: str

    class Config:
        from_attributes = True

@router.post("/", response_model=OrderResponse)
def create_order(
    order_data: OrderCreate,
    user_id: int,  # В реальности из JWT токена
    db: Session = Depends(get_db)
):
    service = OrderService(db)
    try:
        order = service.create_order(
            user_id=user_id,
            items=[item.dict() for item in order_data.items]
        )
        return order
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.get("/", response_model=List[OrderResponse])
def get_orders(
    user_id: int,
    db: Session = Depends(get_db)
):
    service = OrderService(db)
    return service.get_user_orders(user_id)
```

```python
# app/main.py
from fastapi import FastAPI
from app.api import users, products, orders
from app.database import engine, Base

# Создание таблиц
Base.metadata.create_all(bind=engine)

app = FastAPI(
    title="E-Commerce Monolith",
    description="Монолитное приложение электронной коммерции",
    version="1.0.0"
)

# Подключение роутеров
app.include_router(users.router)
app.include_router(products.router)
app.include_router(orders.router)

@app.get("/health")
def health_check():
    return {"status": "healthy"}
```

## Преимущества монолитной архитектуры

### 1. Простота разработки

```python
# Всё в одном месте - легко отлаживать
def process_order(order_id: int):
    order = get_order(order_id)           # Та же кодовая база
    user = get_user(order.user_id)        # Та же кодовая база
    products = get_order_products(order)  # Та же кодовая база

    # Можно легко поставить breakpoint и отладить
    result = calculate_shipping(user.address, products)
    return result
```

### 2. Простота развёртывания

```dockerfile
# Dockerfile - один артефакт для деплоя
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 3. Простота тестирования

```python
# Интеграционные тесты без моков сетевых вызовов
def test_full_order_flow(db_session):
    # Создаём пользователя
    user = create_user(db_session, email="test@example.com")

    # Создаём продукт
    product = create_product(db_session, name="Test", price=100, stock=10)

    # Создаём заказ
    order_service = OrderService(db_session)
    order = order_service.create_order(
        user_id=user.id,
        items=[{"product_id": product.id, "quantity": 2}]
    )

    # Проверяем результат
    assert order.total_amount == 200
    assert product.stock_quantity == 8
```

### 4. Производительность внутренних вызовов

```python
# Вызовы между модулями - просто вызовы функций
# Нет сетевой задержки, сериализации/десериализации

class OrderService:
    def create_order(self, user_id: int, items: List[dict]):
        # Прямой вызов - наносекунды
        user = self.user_service.get_user(user_id)

        # В микросервисах это был бы HTTP запрос - миллисекунды
        # response = requests.get(f"http://user-service/users/{user_id}")
```

## Недостатки монолитной архитектуры

### 1. Сложность масштабирования

```
Проблема: Нужно масштабировать только обработку заказов,
но приходится масштабировать весь монолит

┌─────────────────────────────────────────┐
│              Load Balancer              │
└─────────────────────────────────────────┘
         │         │         │
         ▼         ▼         ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐
    │ Монолит │ │ Монолит │ │ Монолит │
    │ (Всё)   │ │ (Всё)   │ │ (Всё)   │
    └─────────┘ └─────────┘ └─────────┘

Каждая копия содержит ВСЕ модули,
даже те, которые не нужно масштабировать
```

### 2. Долгий цикл развёртывания

```python
# Изменение одной строки требует пересборки всего приложения

# До изменения
def calculate_discount(price: float) -> float:
    return price * 0.1

# После изменения - нужен полный редеплой
def calculate_discount(price: float) -> float:
    return price * 0.15  # Изменили 0.1 на 0.15
```

### 3. Технологическая однородность

```python
# Весь монолит на одном стеке
# Нельзя использовать Node.js для real-time части
# и Python для ML части в одном приложении

# Всё должно быть на Python
class RealTimeNotifications:
    # Хотели бы использовать Node.js/Socket.io
    pass

class MLRecommendations:
    # Python подходит, но зависим от версии фреймворка
    pass
```

### 4. Риск полного отказа

```python
# Ошибка в одном модуле может "уронить" всё приложение

class PaymentService:
    def process(self, amount: float):
        # Утечка памяти здесь...
        self.cache = {}
        for i in range(1000000):
            self.cache[i] = "x" * 1000  # Memory leak!

        # ...выведет из строя весь монолит
```

## Паттерны организации монолита

### Модульный монолит

```python
# Структура модульного монолита
monolith/
├── modules/
│   ├── users/
│   │   ├── __init__.py
│   │   ├── api.py
│   │   ├── services.py
│   │   ├── models.py
│   │   └── repository.py
│   ├── products/
│   │   ├── __init__.py
│   │   ├── api.py
│   │   ├── services.py
│   │   ├── models.py
│   │   └── repository.py
│   └── orders/
│       ├── __init__.py
│       ├── api.py
│       ├── services.py
│       ├── models.py
│       └── repository.py
├── shared/
│   ├── database.py
│   └── events.py
└── main.py
```

```python
# modules/orders/services.py
from modules.users.services import UserService
from modules.products.services import ProductService

class OrderService:
    def __init__(self):
        # Взаимодействие через интерфейсы, а не напрямую с БД
        self.user_service = UserService()
        self.product_service = ProductService()

    def create_order(self, user_id: int, product_ids: List[int]):
        # Используем публичные методы других модулей
        user = self.user_service.get_by_id(user_id)
        products = self.product_service.get_by_ids(product_ids)

        # Бизнес-логика заказа
        return self._create(user, products)
```

## Когда использовать монолитную архитектуру?

### Подходит для:

1. **Стартапов и MVP** - быстрый запуск
2. **Небольших команд** (до 5-10 разработчиков)
3. **Простых доменов** с чёткими границами
4. **Приложений с низкой нагрузкой**

### Не подходит для:

1. **Больших команд** (более 20 разработчиков)
2. **Высоконагруженных систем** с разными требованиями к масштабированию
3. **Систем с разными циклами разработки** для разных частей
4. **Приложений, требующих разные технологии**

## Эволюция монолита

```
Этап 1: Простой монолит
┌─────────────────┐
│     Монолит     │
│  (Всё вместе)   │
└─────────────────┘

Этап 2: Модульный монолит
┌─────────────────┐
│   ┌───┐ ┌───┐   │
│   │ A │ │ B │   │
│   └───┘ └───┘   │
│   ┌───┐ ┌───┐   │
│   │ C │ │ D │   │
│   └───┘ └───┘   │
└─────────────────┘

Этап 3: Микросервисы (при необходимости)
┌───┐ ┌───┐ ┌───┐ ┌───┐
│ A │ │ B │ │ C │ │ D │
└───┘ └───┘ └───┘ └───┘
```

## Best Practices

### 1. Чёткие границы модулей

```python
# Хорошо: модули общаются через интерфейсы
class UserServiceInterface(ABC):
    @abstractmethod
    def get_user(self, user_id: int) -> User:
        pass

# Плохо: прямой доступ к данным другого модуля
def get_order_with_user(order_id: int, db: Session):
    # Напрямую лезем в таблицу users
    return db.query(Order).join(User).filter(Order.id == order_id).first()
```

### 2. Единая точка конфигурации

```python
# config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    redis_url: str
    jwt_secret: str
    debug: bool = False

    class Config:
        env_file = ".env"

settings = Settings()
```

### 3. Централизованная обработка ошибок

```python
# exceptions.py
class AppException(Exception):
    def __init__(self, message: str, code: str):
        self.message = message
        self.code = code

class NotFoundError(AppException):
    def __init__(self, resource: str, id: int):
        super().__init__(f"{resource} with id {id} not found", "NOT_FOUND")

# main.py
@app.exception_handler(AppException)
async def app_exception_handler(request, exc):
    return JSONResponse(
        status_code=400,
        content={"error": exc.code, "message": exc.message}
    )
```

## Заключение

Монолитная архитектура — это проверенный временем подход, который отлично подходит для начала разработки большинства проектов. Ключ к успеху — поддержание чистой структуры кода с чёткими границами модулей, что позволит при необходимости эволюционировать в сторону микросервисов без полной переписывания системы.

Помните: "Если вы не можете построить хорошо структурированный монолит, что заставляет вас думать, что микросервисы помогут?" — Саймон Браун
