# Многослойная архитектура (Layered Architecture / N-tier Architecture)

## Определение

**Многослойная архитектура** (Layered Architecture) — это архитектурный паттерн, организующий приложение в виде горизонтальных слоёв, каждый из которых выполняет определённую роль. Каждый слой зависит только от нижележащего слоя и предоставляет сервисы вышележащему.

Также известна как **N-tier Architecture** (многоуровневая архитектура), где N — количество уровней (обычно 3-4).

```
┌─────────────────────────────────────────────────┐
│           Presentation Layer (UI)               │  ← Пользовательский интерфейс
├─────────────────────────────────────────────────┤
│           Application Layer (Services)          │  ← Бизнес-логика приложения
├─────────────────────────────────────────────────┤
│           Domain Layer (Business Logic)         │  ← Доменная модель
├─────────────────────────────────────────────────┤
│           Data Access Layer (Persistence)       │  ← Работа с данными
├─────────────────────────────────────────────────┤
│           Database / External Services          │  ← Хранилище данных
└─────────────────────────────────────────────────┘
```

## Ключевые характеристики

### 1. Разделение ответственности (Separation of Concerns)
Каждый слой отвечает за конкретную функциональность:
- **Presentation Layer** — отображение данных, обработка пользовательского ввода
- **Application Layer** — координация use cases, оркестрация
- **Domain Layer** — бизнес-правила, доменная логика
- **Data Access Layer** — CRUD операции, работа с БД

### 2. Направление зависимостей
```
UI → Application → Domain → Data Access → Database

Правило: зависимости направлены ТОЛЬКО вниз
```

### 3. Изоляция слоёв
- **Closed layers** (закрытые) — запрос проходит через каждый слой
- **Open layers** (открытые) — можно пропустить слой

### 4. Абстракции между слоями
Слои взаимодействуют через интерфейсы, а не конкретные реализации.

## Когда использовать

### Идеальные сценарии
- **Традиционные бизнес-приложения** (ERP, CRM, банковские системы)
- **Небольшие и средние проекты** с командой до 10-15 человек
- **Приложения с чёткой доменной логикой**
- **MVP и стартапы** на ранних стадиях
- **Проекты с требованиями к maintainability**
- **Команды с разным уровнем опыта** — легко понять структуру

### Не подходит для
- Высоконагруженных систем с требованием горизонтального масштабирования
- Real-time приложений с низкой задержкой
- Систем с очень разнородной функциональностью
- Микросервисных архитектур

## Преимущества и недостатки

### Преимущества

| Преимущество | Описание |
|--------------|----------|
| **Простота понимания** | Интуитивная структура, легко объяснить новичкам |
| **Тестируемость** | Каждый слой можно тестировать изолированно |
| **Maintainability** | Изменения в одном слое не затрагивают другие |
| **Повторное использование** | Слои можно переиспользовать в разных проектах |
| **Параллельная разработка** | Команды могут работать над разными слоями |
| **Стандартизация** | Широко известный паттерн, много инструментов |

### Недостатки

| Недостаток | Описание |
|------------|----------|
| **Monolithic deployment** | Всё приложение деплоится как единое целое |
| **Sinkhole anti-pattern** | Запросы проходят через все слои без обработки |
| **Performance overhead** | Маппинг данных между слоями |
| **Сложность масштабирования** | Нельзя масштабировать отдельные слои |
| **Жёсткая связность** | Изменения в БД могут затронуть все слои |

## Примеры реализации

### Структура проекта (Python/FastAPI)

```
src/
├── presentation/           # Presentation Layer
│   ├── api/
│   │   ├── __init__.py
│   │   ├── users.py       # REST endpoints
│   │   ├── orders.py
│   │   └── auth.py
│   ├── schemas/           # Request/Response DTOs
│   │   ├── user_schema.py
│   │   └── order_schema.py
│   └── middleware/
│       └── auth_middleware.py
│
├── application/            # Application Layer
│   ├── services/
│   │   ├── user_service.py
│   │   └── order_service.py
│   ├── dto/               # Data Transfer Objects
│   │   └── user_dto.py
│   └── interfaces/        # Port interfaces
│       └── repository_interfaces.py
│
├── domain/                 # Domain Layer
│   ├── entities/
│   │   ├── user.py
│   │   └── order.py
│   ├── value_objects/
│   │   ├── email.py
│   │   └── money.py
│   ├── services/          # Domain services
│   │   └── pricing_service.py
│   └── exceptions/
│       └── domain_exceptions.py
│
├── infrastructure/         # Data Access Layer
│   ├── repositories/
│   │   ├── user_repository.py
│   │   └── order_repository.py
│   ├── orm/
│   │   └── models.py
│   ├── external/
│   │   └── payment_gateway.py
│   └── config/
│       └── database.py
│
└── main.py
```

### Пример кода

#### Domain Layer — Entity

```python
# domain/entities/user.py
from dataclasses import dataclass
from datetime import datetime
from typing import Optional
from domain.value_objects.email import Email

@dataclass
class User:
    """Доменная сущность пользователя"""
    id: Optional[int]
    email: Email
    name: str
    created_at: datetime
    is_active: bool = True

    def deactivate(self) -> None:
        """Деактивация пользователя — бизнес-правило"""
        if not self.is_active:
            raise ValueError("User is already deactivated")
        self.is_active = False

    def can_place_order(self) -> bool:
        """Проверка возможности создать заказ"""
        return self.is_active
```

#### Domain Layer — Value Object

```python
# domain/value_objects/email.py
import re
from dataclasses import dataclass

@dataclass(frozen=True)
class Email:
    """Value Object для email с валидацией"""
    value: str

    def __post_init__(self):
        if not self._is_valid(self.value):
            raise ValueError(f"Invalid email: {self.value}")

    @staticmethod
    def _is_valid(email: str) -> bool:
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return bool(re.match(pattern, email))
```

#### Application Layer — Service

```python
# application/services/user_service.py
from typing import Optional
from application.interfaces.repository_interfaces import IUserRepository
from application.dto.user_dto import CreateUserDTO, UserResponseDTO
from domain.entities.user import User
from domain.value_objects.email import Email
from datetime import datetime

class UserService:
    """Application Service — координирует use cases"""

    def __init__(self, user_repository: IUserRepository):
        self._user_repository = user_repository

    def create_user(self, dto: CreateUserDTO) -> UserResponseDTO:
        """Use case: создание пользователя"""
        # 1. Проверка уникальности email
        existing = self._user_repository.find_by_email(dto.email)
        if existing:
            raise ValueError("Email already exists")

        # 2. Создание доменной сущности
        user = User(
            id=None,
            email=Email(dto.email),
            name=dto.name,
            created_at=datetime.utcnow()
        )

        # 3. Сохранение через репозиторий
        saved_user = self._user_repository.save(user)

        # 4. Преобразование в DTO для ответа
        return UserResponseDTO.from_entity(saved_user)

    def get_user(self, user_id: int) -> Optional[UserResponseDTO]:
        """Use case: получение пользователя"""
        user = self._user_repository.find_by_id(user_id)
        if not user:
            return None
        return UserResponseDTO.from_entity(user)
```

#### Infrastructure Layer — Repository

```python
# infrastructure/repositories/user_repository.py
from typing import Optional
from sqlalchemy.orm import Session
from application.interfaces.repository_interfaces import IUserRepository
from domain.entities.user import User
from domain.value_objects.email import Email
from infrastructure.orm.models import UserModel

class SQLAlchemyUserRepository(IUserRepository):
    """Конкретная реализация репозитория с SQLAlchemy"""

    def __init__(self, session: Session):
        self._session = session

    def find_by_id(self, user_id: int) -> Optional[User]:
        model = self._session.query(UserModel).filter_by(id=user_id).first()
        return self._to_entity(model) if model else None

    def find_by_email(self, email: str) -> Optional[User]:
        model = self._session.query(UserModel).filter_by(email=email).first()
        return self._to_entity(model) if model else None

    def save(self, user: User) -> User:
        model = self._to_model(user)
        self._session.add(model)
        self._session.commit()
        self._session.refresh(model)
        return self._to_entity(model)

    def _to_entity(self, model: UserModel) -> User:
        """Маппинг ORM модели в доменную сущность"""
        return User(
            id=model.id,
            email=Email(model.email),
            name=model.name,
            created_at=model.created_at,
            is_active=model.is_active
        )

    def _to_model(self, entity: User) -> UserModel:
        """Маппинг доменной сущности в ORM модель"""
        return UserModel(
            id=entity.id,
            email=entity.email.value,
            name=entity.name,
            created_at=entity.created_at,
            is_active=entity.is_active
        )
```

#### Presentation Layer — API

```python
# presentation/api/users.py
from fastapi import APIRouter, Depends, HTTPException
from presentation.schemas.user_schema import UserCreateRequest, UserResponse
from application.services.user_service import UserService
from application.dto.user_dto import CreateUserDTO

router = APIRouter(prefix="/users", tags=["users"])

@router.post("/", response_model=UserResponse)
def create_user(
    request: UserCreateRequest,
    user_service: UserService = Depends(get_user_service)
):
    """Endpoint создания пользователя"""
    try:
        dto = CreateUserDTO(email=request.email, name=request.name)
        result = user_service.create_user(dto)
        return UserResponse.from_dto(result)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.get("/{user_id}", response_model=UserResponse)
def get_user(
    user_id: int,
    user_service: UserService = Depends(get_user_service)
):
    """Endpoint получения пользователя"""
    result = user_service.get_user(user_id)
    if not result:
        raise HTTPException(status_code=404, detail="User not found")
    return UserResponse.from_dto(result)
```

## Best practices и антипаттерны

### Best Practices

#### 1. Dependency Injection
```python
# Правильно: инъекция зависимостей
class UserService:
    def __init__(self, repository: IUserRepository):
        self._repository = repository

# Неправильно: создание зависимости внутри
class UserService:
    def __init__(self):
        self._repository = SQLAlchemyUserRepository()  # Жёсткая связь!
```

#### 2. Интерфейсы между слоями
```python
# application/interfaces/repository_interfaces.py
from abc import ABC, abstractmethod

class IUserRepository(ABC):
    @abstractmethod
    def find_by_id(self, user_id: int) -> Optional[User]:
        pass

    @abstractmethod
    def save(self, user: User) -> User:
        pass
```

#### 3. DTO для передачи данных между слоями
```python
# Не передавайте доменные сущности в Presentation Layer
# Используйте DTO

@dataclass
class UserResponseDTO:
    id: int
    email: str
    name: str

    @classmethod
    def from_entity(cls, user: User) -> "UserResponseDTO":
        return cls(id=user.id, email=user.email.value, name=user.name)
```

### Антипаттерны

#### 1. Sinkhole Anti-pattern
```python
# ПЛОХО: слой просто проксирует вызов без логики
class UserService:
    def get_user(self, user_id: int):
        return self._repository.find_by_id(user_id)  # Просто проброс
```

#### 2. Smart UI Anti-pattern
```python
# ПЛОХО: бизнес-логика в контроллере
@router.post("/users")
def create_user(request: UserCreateRequest, db: Session = Depends()):
    # Бизнес-логика не должна быть здесь!
    if User.query.filter_by(email=request.email).first():
        raise HTTPException(400, "Email exists")
    user = User(email=request.email)
    db.add(user)
    db.commit()
```

#### 3. Leaky Abstractions
```python
# ПЛОХО: детали реализации проникают в верхние слои
class UserService:
    def get_user(self, user_id: int):
        # SQLAlchemy модель возвращается из Application Layer
        return db.session.query(UserModel).filter_by(id=user_id).first()
```

## Связанные паттерны

| Паттерн | Связь |
|---------|-------|
| **Hexagonal Architecture** | Эволюция Layered, фокус на портах и адаптерах |
| **Clean Architecture** | Расширение с инверсией зависимостей |
| **Onion Architecture** | Кольцевая структура вместо слоёв |
| **Repository Pattern** | Используется в Data Access Layer |
| **Service Layer Pattern** | Реализует Application Layer |
| **DTO Pattern** | Передача данных между слоями |

## Ресурсы для изучения

### Книги
- **"Patterns of Enterprise Application Architecture"** — Martin Fowler
- **"Domain-Driven Design"** — Eric Evans
- **"Clean Architecture"** — Robert C. Martin

### Статьи
- [Layered Architecture — Microsoft Docs](https://docs.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures)
- [N-tier Architecture Style — Azure Architecture Center](https://docs.microsoft.com/en-us/azure/architecture/guide/architecture-styles/n-tier)

### Видео
- "Software Architecture: Layered Architecture" — Mark Richards
- "Clean Architecture" — Uncle Bob (Robert C. Martin)

---

## Резюме

Многослойная архитектура — это проверенный временем паттерн, который обеспечивает:
- Чёткое разделение ответственности
- Простоту понимания и поддержки
- Хорошую тестируемость

Но помните: это не серебряная пуля. Для высоконагруженных распределённых систем рассмотрите микросервисы или другие паттерны.
