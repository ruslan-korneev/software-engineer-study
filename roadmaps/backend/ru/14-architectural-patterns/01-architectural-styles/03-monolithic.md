# Монолитная архитектура (Monolithic Architecture)

## Определение

**Монолитная архитектура** — это традиционный подход к разработке приложений, при котором вся функциональность системы объединена в единый, неделимый deployment unit. Весь код компилируется, тестируется и деплоится как одно целое.

```
┌─────────────────────────────────────────────────────────────────┐
│                     MONOLITHIC APPLICATION                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │    Users     │  │   Orders     │  │   Products   │           │
│  │    Module    │  │   Module     │  │    Module    │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  Payments    │  │  Inventory   │  │  Notifications│          │
│  │   Module     │  │   Module     │  │    Module    │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Shared Database                             │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    Single Deployment Unit
```

Термин "монолит" происходит от греческого "μόνος" (один) и "λίθος" (камень) — единый камень.

## Ключевые характеристики

### 1. Единый codebase и deployment

```
monolithic-app/
├── src/
│   ├── users/           # Все модули в одном репозитории
│   ├── orders/
│   ├── products/
│   ├── payments/
│   └── shared/          # Общий код
├── tests/
├── Dockerfile           # Один Docker image
├── requirements.txt     # Одни зависимости
└── deploy.yaml          # Один deployment
```

### 2. Shared memory space

```python
# Все модули работают в одном процессе
# Могут напрямую вызывать друг друга

from users.services import UserService
from orders.services import OrderService

# Синхронный вызов в том же процессе
user = UserService.get_user(user_id)
order = OrderService.create_order(user, items)
```

### 3. Single database

```
┌─────────────────────────────────────────────────────────┐
│                    PostgreSQL Database                   │
├─────────────────────────────────────────────────────────┤
│  users table    │  orders table   │  products table     │
│  payments table │  inventory table│  notifications table│
└─────────────────────────────────────────────────────────┘

Все модули используют одну БД с общей схемой
```

### 4. Типы монолитов

```
┌─────────────────────────────────────────────────────────────────┐
│                     TYPES OF MONOLITHS                          │
├───────────────────┬───────────────────┬─────────────────────────┤
│   Single-Process  │   Modular         │   Distributed           │
│     Monolith      │   Monolith        │     Monolith            │
├───────────────────┼───────────────────┼─────────────────────────┤
│ • Один процесс    │ • Модульная       │ • Несколько сервисов    │
│ • Всё в одном     │   структура       │ • Shared database       │
│ • Классический    │ • Границы модулей │ • Tight coupling        │
│   подход          │ • Можно разделить │ • "Big Ball of Mud"     │
└───────────────────┴───────────────────┴─────────────────────────┘
```

## Когда использовать

### Идеальные сценарии

| Сценарий | Почему монолит подходит |
|----------|------------------------|
| **Стартап / MVP** | Быстрая разработка, простота |
| **Небольшая команда** (< 10 человек) | Меньше координации |
| **Прототип** | Быстрая итерация |
| **Простой домен** | Нет сложных bounded contexts |
| **Неопределённые требования** | Легче рефакторить |
| **Начало нового проекта** | "Monolith First" подход |

### Monolith First Strategy (Martin Fowler)

```
Этап 1: Монолит
────────────────
Начните с монолита, чтобы:
• Понять домен
• Найти boundaries
• Быстро итерировать

Этап 2: Модульный монолит
─────────────────────────
Выделите чёткие модули:
• Определите bounded contexts
• Минимизируйте coupling
• Подготовьте к разделению

Этап 3: Микросервисы (если нужно)
─────────────────────────────────
Разделите, когда:
• Команда выросла
• Нужна независимая масштабируемость
• Чёткие domain boundaries
```

### Не подходит для

- Очень большие команды (> 50 разработчиков)
- Высоконагруженные системы с разными SLA для компонентов
- Системы с требованием polyglot persistence
- Организации с независимыми командами

## Преимущества и недостатки

### Преимущества

| Преимущество | Описание |
|--------------|----------|
| **Простота разработки** | Один проект, одна IDE, один debug |
| **Простой deployment** | Одна сборка, один артефакт |
| **Производительность** | In-process вызовы, нет network latency |
| **Транзакции** | ACID транзакции через всю систему |
| **Простое тестирование** | E2E тесты без моков сервисов |
| **Рефакторинг** | Изменения в одном месте |
| **Низкий порог входа** | Легко понять архитектуру |

### Недостатки

| Недостаток | Описание |
|------------|----------|
| **Coupling** | Изменения могут затронуть всё приложение |
| **Scaling** | Только вертикальное масштабирование |
| **Deployment** | Любое изменение = полный redeploy |
| **Technology lock-in** | Один стек для всего приложения |
| **Reliability** | Баг в одном модуле может сломать всё |
| **Team scaling** | Сложно работать большим командам |
| **Build time** | Рост времени сборки с размером |

## Примеры реализации

### Структура модульного монолита (Python/FastAPI)

```
ecommerce/
├── src/
│   ├── __init__.py
│   ├── main.py                  # Entry point
│   ├── config.py                # Configuration
│   │
│   ├── users/                   # User module
│   │   ├── __init__.py
│   │   ├── router.py            # API endpoints
│   │   ├── service.py           # Business logic
│   │   ├── repository.py        # Data access
│   │   ├── models.py            # Domain models
│   │   └── schemas.py           # DTOs
│   │
│   ├── products/                # Product module
│   │   ├── __init__.py
│   │   ├── router.py
│   │   ├── service.py
│   │   ├── repository.py
│   │   ├── models.py
│   │   └── schemas.py
│   │
│   ├── orders/                  # Order module
│   │   ├── __init__.py
│   │   ├── router.py
│   │   ├── service.py
│   │   ├── repository.py
│   │   ├── models.py
│   │   └── schemas.py
│   │
│   ├── payments/                # Payment module
│   │   └── ...
│   │
│   └── shared/                  # Shared utilities
│       ├── database.py
│       ├── exceptions.py
│       └── middleware.py
│
├── tests/
├── alembic/                     # Migrations
├── Dockerfile
├── docker-compose.yaml
└── requirements.txt
```

### Main Application

```python
# src/main.py
from fastapi import FastAPI
from contextlib import asynccontextmanager

from src.shared.database import engine, Base
from src.users.router import router as users_router
from src.products.router import router as products_router
from src.orders.router import router as orders_router
from src.payments.router import router as payments_router

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    # Shutdown
    await engine.dispose()

app = FastAPI(
    title="E-Commerce Monolith",
    version="1.0.0",
    lifespan=lifespan
)

# Регистрация всех модулей
app.include_router(users_router, prefix="/api/users", tags=["users"])
app.include_router(products_router, prefix="/api/products", tags=["products"])
app.include_router(orders_router, prefix="/api/orders", tags=["orders"])
app.include_router(payments_router, prefix="/api/payments", tags=["payments"])

@app.get("/health")
def health_check():
    return {"status": "healthy"}
```

### User Module

```python
# src/users/models.py
from sqlalchemy import Column, Integer, String, Boolean, DateTime
from sqlalchemy.sql import func
from src.shared.database import Base

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    email = Column(String(255), unique=True, nullable=False)
    name = Column(String(100), nullable=False)
    hashed_password = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, server_default=func.now())


# src/users/repository.py
from typing import Optional
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from src.users.models import User

class UserRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def get_by_id(self, user_id: int) -> Optional[User]:
        result = await self.session.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()

    async def get_by_email(self, email: str) -> Optional[User]:
        result = await self.session.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()

    async def create(self, user: User) -> User:
        self.session.add(user)
        await self.session.commit()
        await self.session.refresh(user)
        return user


# src/users/service.py
from typing import Optional
from src.users.repository import UserRepository
from src.users.models import User
from src.users.schemas import UserCreate
from src.shared.security import hash_password

class UserService:
    def __init__(self, repository: UserRepository):
        self.repository = repository

    async def create_user(self, data: UserCreate) -> User:
        # Проверка уникальности email
        existing = await self.repository.get_by_email(data.email)
        if existing:
            raise ValueError("Email already registered")

        user = User(
            email=data.email,
            name=data.name,
            hashed_password=hash_password(data.password)
        )
        return await self.repository.create(user)

    async def get_user(self, user_id: int) -> Optional[User]:
        return await self.repository.get_by_id(user_id)


# src/users/router.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from src.shared.database import get_session
from src.users.service import UserService
from src.users.repository import UserRepository
from src.users.schemas import UserCreate, UserResponse

router = APIRouter()

def get_user_service(session: AsyncSession = Depends(get_session)) -> UserService:
    repository = UserRepository(session)
    return UserService(repository)

@router.post("/", response_model=UserResponse, status_code=201)
async def create_user(
    data: UserCreate,
    service: UserService = Depends(get_user_service)
):
    try:
        user = await service.create_user(data)
        return UserResponse.model_validate(user)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    service: UserService = Depends(get_user_service)
):
    user = await service.get_user(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return UserResponse.model_validate(user)
```

### Orders Module (с зависимостью от Users)

```python
# src/orders/service.py
from typing import List
from src.orders.repository import OrderRepository
from src.orders.models import Order, OrderItem
from src.orders.schemas import CreateOrderRequest
from src.users.service import UserService
from src.products.service import ProductService

class OrderService:
    """Order Service — координирует несколько модулей"""

    def __init__(
        self,
        order_repo: OrderRepository,
        user_service: UserService,
        product_service: ProductService
    ):
        self.order_repo = order_repo
        self.user_service = user_service
        self.product_service = product_service

    async def create_order(self, user_id: int, request: CreateOrderRequest) -> Order:
        # 1. Проверка пользователя (in-process вызов)
        user = await self.user_service.get_user(user_id)
        if not user:
            raise ValueError("User not found")
        if not user.is_active:
            raise ValueError("User is not active")

        # 2. Проверка и резервирование товаров
        items = []
        total_amount = 0

        for item_request in request.items:
            product = await self.product_service.get_product(item_request.product_id)
            if not product:
                raise ValueError(f"Product {item_request.product_id} not found")

            # Проверка наличия
            if product.stock < item_request.quantity:
                raise ValueError(f"Insufficient stock for {product.name}")

            # Резервирование
            await self.product_service.reserve_stock(
                product.id,
                item_request.quantity
            )

            items.append(OrderItem(
                product_id=product.id,
                quantity=item_request.quantity,
                price=product.price
            ))
            total_amount += product.price * item_request.quantity

        # 3. Создание заказа (в одной транзакции!)
        order = Order(
            user_id=user_id,
            items=items,
            total_amount=total_amount,
            status="pending"
        )

        return await self.order_repo.create(order)
```

### Shared Database Configuration

```python
# src/shared/database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import declarative_base
from src.config import settings

engine = create_async_engine(
    settings.DATABASE_URL,
    echo=settings.DEBUG,
    pool_size=20,
    max_overflow=10
)

AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)

Base = declarative_base()

async def get_session() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()
```

### Dockerfile для деплоя

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Application code
COPY src/ ./src/
COPY alembic/ ./alembic/
COPY alembic.ini .

# Expose port
EXPOSE 8000

# Run migrations and start server
CMD ["sh", "-c", "alembic upgrade head && uvicorn src.main:app --host 0.0.0.0 --port 8000"]
```

## Best practices и антипаттерны

### Best Practices

#### 1. Модульная структура
```python
# ХОРОШО: чёткие границы модулей
src/
├── users/           # Независимый модуль
├── orders/          # Зависит от users, products
├── products/        # Независимый модуль
└── shared/          # Только утилиты!

# Правило: модуль импортирует только shared и свои зависимости
```

#### 2. Внутренние интерфейсы
```python
# users/__init__.py — публичный API модуля
from .service import UserService
from .schemas import UserResponse, UserCreate

__all__ = ["UserService", "UserResponse", "UserCreate"]

# Другие модули используют только публичный API
from users import UserService  # OK
from users.repository import UserRepository  # Нежелательно!
```

#### 3. Feature flags для постепенного rollout
```python
# src/shared/features.py
from src.config import settings

class FeatureFlags:
    @property
    def new_payment_flow(self) -> bool:
        return settings.FEATURE_NEW_PAYMENT_FLOW

    @property
    def async_notifications(self) -> bool:
        return settings.FEATURE_ASYNC_NOTIFICATIONS

features = FeatureFlags()

# Использование
if features.new_payment_flow:
    await process_payment_v2(order)
else:
    await process_payment_v1(order)
```

#### 4. Health checks и observability
```python
@app.get("/health")
async def health_check(session: AsyncSession = Depends(get_session)):
    checks = {
        "database": await check_database(session),
        "redis": await check_redis(),
        "external_api": await check_external_api()
    }
    healthy = all(checks.values())
    return {
        "status": "healthy" if healthy else "unhealthy",
        "checks": checks
    }
```

### Антипаттерны

#### 1. Big Ball of Mud
```python
# ПЛОХО: всё связано со всем
class OrderService:
    def create_order(self):
        # Прямой доступ к таблицам других модулей
        user = db.execute("SELECT * FROM users WHERE...")
        product = db.execute("SELECT * FROM products WHERE...")
        payment = db.execute("INSERT INTO payments...")

        # Логика рассыпана по всему коду
        send_email(user.email)  # Notification logic in Order
        update_inventory()       # Inventory logic in Order
        create_invoice()         # Billing logic in Order
```

#### 2. Circular dependencies
```python
# ПЛОХО: циклические зависимости
# users/service.py
from orders.service import OrderService

class UserService:
    def delete_user(self, user_id):
        OrderService().cancel_user_orders(user_id)

# orders/service.py
from users.service import UserService

class OrderService:
    def create_order(self, user_id):
        UserService().validate_user(user_id)

# РЕШЕНИЕ: используйте events или dependency injection
```

#### 3. Shared mutable state
```python
# ПЛОХО: глобальное изменяемое состояние
global_cache = {}

class UserService:
    def get_user(self, user_id):
        if user_id in global_cache:
            return global_cache[user_id]
        # Race conditions, memory leaks!

# ХОРОШО: инъекция зависимости кэша
class UserService:
    def __init__(self, cache: Cache):
        self._cache = cache
```

## Связанные паттерны

| Паттерн | Связь |
|---------|-------|
| **Layered Architecture** | Часто используется внутри монолита |
| **Modular Monolith** | Эволюция классического монолита |
| **Microservices** | Альтернатива / следующий этап |
| **Strangler Fig Pattern** | Постепенная миграция из монолита |
| **Domain-Driven Design** | Помогает определить boundaries |
| **CQRS** | Может применяться внутри монолита |

## Ресурсы для изучения

### Книги
- **"Monolith to Microservices"** — Sam Newman
- **"Building Microservices"** — Sam Newman (глава о монолитах)
- **"Fundamentals of Software Architecture"** — Mark Richards, Neal Ford

### Статьи
- [MonolithFirst — Martin Fowler](https://martinfowler.com/bliki/MonolithFirst.html)
- [Modular Monolith — Kamil Grzybek](https://www.kamilgrzybek.com/design/modular-monolith-primer/)
- [The Majestic Monolith — DHH](https://m.signalvnoise.com/the-majestic-monolith/)

### Видео
- "Modular Monoliths" — Simon Brown
- "The Art of Scalability" — GOTO Conference

---

## Резюме

Монолитная архитектура — это не антипаттерн, а обоснованный выбор для многих ситуаций:

- **Простота** — одна кодовая база, один деплой
- **Производительность** — in-process вызовы
- **Транзакции** — ACID через всю систему
- **Подходит для** — стартапов, небольших команд, MVP

**Правило**: начинайте с монолита (Monolith First), переходите к микросервисам, когда это действительно нужно.
