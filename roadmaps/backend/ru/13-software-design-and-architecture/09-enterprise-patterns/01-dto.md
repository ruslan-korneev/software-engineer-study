# Data Transfer Objects (DTO)

## Что такое DTO?

**Data Transfer Object (DTO)** — это объект, который переносит данные между процессами или слоями приложения. DTO представляет собой простой контейнер для данных без какой-либо бизнес-логики. Основная цель DTO — уменьшить количество вызовов между компонентами системы, объединяя несколько параметров в один объект.

> Паттерн был впервые описан Мартином Фаулером в книге "Patterns of Enterprise Application Architecture" (2002).

## Зачем нужны DTO?

### Основные причины использования:

1. **Разделение слоёв** — DTO отделяет внутреннюю модель данных от внешнего представления
2. **Контроль данных** — позволяет явно определить, какие данные передаются
3. **Версионирование API** — упрощает поддержку разных версий API
4. **Безопасность** — скрывает внутренние поля сущностей (пароли, служебные поля)
5. **Оптимизация** — передаёт только необходимые данные, уменьшая размер ответа

## Простой пример DTO на Python

### Использование dataclasses

```python
from dataclasses import dataclass, asdict
from typing import Optional
from datetime import datetime


@dataclass
class UserDTO:
    """DTO для передачи данных пользователя."""
    id: int
    username: str
    email: str
    full_name: Optional[str] = None
    created_at: Optional[datetime] = None

    def to_dict(self) -> dict:
        """Преобразование в словарь для JSON-сериализации."""
        return asdict(self)


# Использование
user_dto = UserDTO(
    id=1,
    username="john_doe",
    email="john@example.com",
    full_name="John Doe"
)

print(user_dto.to_dict())
# {'id': 1, 'username': 'john_doe', 'email': 'john@example.com',
#  'full_name': 'John Doe', 'created_at': None}
```

### Использование Pydantic

```python
from pydantic import BaseModel, EmailStr, Field
from typing import Optional
from datetime import datetime


class UserDTO(BaseModel):
    """DTO пользователя с валидацией."""
    id: int
    username: str = Field(..., min_length=3, max_length=50)
    email: EmailStr
    full_name: Optional[str] = None
    created_at: Optional[datetime] = None

    class Config:
        # Позволяет создавать DTO из ORM-объектов
        from_attributes = True


class UserCreateDTO(BaseModel):
    """DTO для создания пользователя."""
    username: str = Field(..., min_length=3, max_length=50)
    email: EmailStr
    password: str = Field(..., min_length=8)
    full_name: Optional[str] = None


class UserUpdateDTO(BaseModel):
    """DTO для обновления пользователя."""
    username: Optional[str] = Field(None, min_length=3, max_length=50)
    email: Optional[EmailStr] = None
    full_name: Optional[str] = None


# Создание DTO из словаря
user_data = {
    "id": 1,
    "username": "john_doe",
    "email": "john@example.com"
}
user_dto = UserDTO(**user_data)
print(user_dto.model_dump())  # Pydantic v2
```

## DTO в реальном приложении

### Преобразование Entity в DTO

```python
from dataclasses import dataclass
from datetime import datetime
from typing import List, Optional


# Доменная сущность (Entity)
class User:
    def __init__(
        self,
        id: int,
        username: str,
        email: str,
        password_hash: str,  # Не должен попасть в DTO!
        full_name: Optional[str] = None,
        is_active: bool = True,
        created_at: datetime = None,
        last_login: datetime = None
    ):
        self.id = id
        self.username = username
        self.email = email
        self.password_hash = password_hash
        self.full_name = full_name
        self.is_active = is_active
        self.created_at = created_at or datetime.now()
        self.last_login = last_login


# DTO для ответа API
@dataclass
class UserResponseDTO:
    id: int
    username: str
    email: str
    full_name: Optional[str]
    is_active: bool
    member_since: str  # Форматированная дата

    @classmethod
    def from_entity(cls, user: User) -> "UserResponseDTO":
        """Фабричный метод для создания DTO из Entity."""
        return cls(
            id=user.id,
            username=user.username,
            email=user.email,
            full_name=user.full_name,
            is_active=user.is_active,
            member_since=user.created_at.strftime("%Y-%m-%d")
        )


# Использование
user = User(
    id=1,
    username="john_doe",
    email="john@example.com",
    password_hash="$2b$12$...",  # Хеш пароля
    full_name="John Doe"
)

# Преобразуем в безопасный DTO
user_dto = UserResponseDTO.from_entity(user)
# password_hash не попадёт в DTO!
```

### Маппер для преобразования

```python
from typing import List, Type, TypeVar
from dataclasses import dataclass


T = TypeVar("T")
D = TypeVar("D")


class DTOMapper:
    """Универсальный маппер Entity -> DTO."""

    @staticmethod
    def to_dto(entity, dto_class: Type[T], mapping: dict = None) -> T:
        """
        Преобразует entity в DTO.

        Args:
            entity: Исходный объект
            dto_class: Класс DTO
            mapping: Словарь маппинга полей {dto_field: entity_field}
        """
        mapping = mapping or {}
        dto_fields = dto_class.__dataclass_fields__.keys()

        kwargs = {}
        for field in dto_fields:
            source_field = mapping.get(field, field)
            if hasattr(entity, source_field):
                kwargs[field] = getattr(entity, source_field)

        return dto_class(**kwargs)

    @staticmethod
    def to_dto_list(entities: List, dto_class: Type[T], mapping: dict = None) -> List[T]:
        """Преобразует список entities в список DTO."""
        return [DTOMapper.to_dto(e, dto_class, mapping) for e in entities]


# Пример использования
@dataclass
class ProductDTO:
    id: int
    name: str
    price: float
    category_name: str


class Product:
    def __init__(self, id, name, price, category):
        self.id = id
        self.name = name
        self.price = price
        self.category = category  # Объект Category


class Category:
    def __init__(self, id, name):
        self.id = id
        self.name = name


# Для сложных преобразований лучше создать отдельный маппер
class ProductDTOMapper:
    @staticmethod
    def to_dto(product: Product) -> ProductDTO:
        return ProductDTO(
            id=product.id,
            name=product.name,
            price=product.price,
            category_name=product.category.name
        )
```

## DTO в FastAPI

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, EmailStr, Field
from typing import List, Optional
from datetime import datetime

app = FastAPI()


# DTO для запросов
class UserCreateDTO(BaseModel):
    username: str = Field(..., min_length=3, max_length=50)
    email: EmailStr
    password: str = Field(..., min_length=8)


class UserUpdateDTO(BaseModel):
    username: Optional[str] = Field(None, min_length=3, max_length=50)
    email: Optional[EmailStr] = None


# DTO для ответов
class UserResponseDTO(BaseModel):
    id: int
    username: str
    email: str
    created_at: datetime

    class Config:
        from_attributes = True


class UserListResponseDTO(BaseModel):
    users: List[UserResponseDTO]
    total: int
    page: int
    per_page: int


# Эндпоинты
@app.post("/users", response_model=UserResponseDTO)
async def create_user(user_dto: UserCreateDTO):
    """Создание пользователя."""
    # user_dto автоматически валидируется Pydantic
    # Создаём пользователя в БД...
    user = create_user_in_db(user_dto)
    return UserResponseDTO.model_validate(user)


@app.get("/users/{user_id}", response_model=UserResponseDTO)
async def get_user(user_id: int):
    """Получение пользователя по ID."""
    user = get_user_from_db(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return UserResponseDTO.model_validate(user)


@app.get("/users", response_model=UserListResponseDTO)
async def list_users(page: int = 1, per_page: int = 10):
    """Список пользователей с пагинацией."""
    users, total = get_users_from_db(page, per_page)
    return UserListResponseDTO(
        users=[UserResponseDTO.model_validate(u) for u in users],
        total=total,
        page=page,
        per_page=per_page
    )
```

## Вложенные DTO

```python
from pydantic import BaseModel
from typing import List, Optional
from datetime import datetime


class AddressDTO(BaseModel):
    street: str
    city: str
    country: str
    postal_code: str


class OrderItemDTO(BaseModel):
    product_id: int
    product_name: str
    quantity: int
    price: float

    @property
    def total(self) -> float:
        return self.quantity * self.price


class OrderDTO(BaseModel):
    id: int
    order_number: str
    status: str
    items: List[OrderItemDTO]
    shipping_address: AddressDTO
    created_at: datetime

    @property
    def total_amount(self) -> float:
        return sum(item.total for item in self.items)


# Пример данных
order_data = {
    "id": 1,
    "order_number": "ORD-2024-001",
    "status": "pending",
    "items": [
        {"product_id": 1, "product_name": "Laptop", "quantity": 1, "price": 999.99},
        {"product_id": 2, "product_name": "Mouse", "quantity": 2, "price": 29.99}
    ],
    "shipping_address": {
        "street": "123 Main St",
        "city": "New York",
        "country": "USA",
        "postal_code": "10001"
    },
    "created_at": "2024-01-15T10:30:00"
}

order_dto = OrderDTO(**order_data)
print(f"Order total: ${order_dto.total_amount:.2f}")
```

## Сравнение подходов

| Подход | Плюсы | Минусы |
|--------|-------|--------|
| **dataclasses** | Простота, встроен в Python | Нет валидации из коробки |
| **Pydantic** | Валидация, JSON-сериализация | Зависимость, overhead |
| **NamedTuple** | Иммутабельность, быстрый | Неудобный синтаксис |
| **attrs** | Гибкость, производительность | Дополнительная зависимость |

## Плюсы и минусы паттерна DTO

### Преимущества

1. **Чёткое разделение** — API и внутренняя модель независимы
2. **Безопасность** — контроль над тем, какие данные передаются
3. **Гибкость** — разные DTO для разных клиентов/версий API
4. **Документация** — DTO служат контрактом API
5. **Валидация** — входные данные проверяются на границе системы

### Недостатки

1. **Дублирование кода** — много похожих классов
2. **Накладные расходы** — преобразования Entity <-> DTO
3. **Синхронизация** — изменения модели требуют изменений DTO
4. **Сложность** — для простых CRUD избыточно

## Когда использовать DTO?

### Рекомендуется:
- REST API и GraphQL
- Микросервисная архитектура
- Когда нужно скрыть внутреннюю структуру
- Для сложных преобразований данных
- При версионировании API

### Не рекомендуется:
- Простые CRUD приложения
- Внутренние сервисы без внешних клиентов
- Прототипы и MVP

## Лучшие практики

1. **Именование** — используйте суффиксы: `UserDTO`, `UserCreateDTO`, `UserResponseDTO`
2. **Иммутабельность** — DTO должны быть неизменяемыми по возможности
3. **Валидация** — проверяйте данные на входе в систему
4. **Документация** — описывайте поля DTO для генерации API-документации
5. **Маппинг** — используйте автоматический маппинг где возможно

## Заключение

DTO — это фундаментальный паттерн для построения чистых и поддерживаемых API. Он обеспечивает контроль над данными, которые передаются между слоями приложения, и защищает внутреннюю модель от изменений во внешнем интерфейсе. В Python экосистеме Pydantic стал де-факто стандартом для создания DTO благодаря встроенной валидации и отличной интеграции с FastAPI.
