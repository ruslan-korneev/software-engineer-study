# Repository Pattern

## Что такое Repository?

**Repository (Репозиторий)** — это паттерн проектирования, который абстрагирует слой доступа к данным, предоставляя коллекционно-подобный интерфейс для работы с доменными объектами. Репозиторий скрывает детали работы с базой данных, позволяя бизнес-логике работать с объектами так, как будто они находятся в памяти.

> Паттерн описан Мартином Фаулером в "Patterns of Enterprise Application Architecture" и является ключевым элементом Domain-Driven Design (DDD).

## Основная идея

Репозиторий выступает посредником между доменом и слоем данных, создавая иллюзию коллекции объектов в памяти:

```
Домен  <-->  Repository  <-->  База данных
              (Абстракция)      (Реализация)
```

## Зачем нужен Repository?

### Основные преимущества:

1. **Абстракция хранилища** — бизнес-логика не знает о БД
2. **Тестируемость** — легко подменять репозиторий mock-объектами
3. **Единая точка доступа** — все запросы к сущности в одном месте
4. **Разделение ответственности** — домен не зависит от инфраструктуры
5. **Переиспользование** — общие запросы выносятся в методы репозитория

## Базовая реализация

### Абстрактный репозиторий

```python
from abc import ABC, abstractmethod
from typing import Generic, TypeVar, List, Optional

T = TypeVar("T")


class AbstractRepository(ABC, Generic[T]):
    """Базовый абстрактный репозиторий."""

    @abstractmethod
    def add(self, entity: T) -> None:
        """Добавить сущность в репозиторий."""
        raise NotImplementedError

    @abstractmethod
    def get(self, id: int) -> Optional[T]:
        """Получить сущность по ID."""
        raise NotImplementedError

    @abstractmethod
    def get_all(self) -> List[T]:
        """Получить все сущности."""
        raise NotImplementedError

    @abstractmethod
    def update(self, entity: T) -> None:
        """Обновить сущность."""
        raise NotImplementedError

    @abstractmethod
    def delete(self, id: int) -> None:
        """Удалить сущность по ID."""
        raise NotImplementedError
```

### Доменная сущность

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Optional


@dataclass
class User:
    """Доменная сущность пользователя."""
    id: Optional[int]
    username: str
    email: str
    password_hash: str
    is_active: bool = True
    created_at: datetime = None

    def __post_init__(self):
        if self.created_at is None:
            self.created_at = datetime.now()

    def deactivate(self):
        """Бизнес-логика деактивации."""
        self.is_active = False

    def activate(self):
        """Бизнес-логика активации."""
        self.is_active = True
```

### Конкретная реализация с SQLAlchemy

```python
from sqlalchemy import Column, Integer, String, Boolean, DateTime, create_engine
from sqlalchemy.orm import declarative_base, Session, sessionmaker
from typing import List, Optional
from datetime import datetime

Base = declarative_base()


# ORM модель (не путать с доменной сущностью!)
class UserModel(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, autoincrement=True)
    username = Column(String(50), unique=True, nullable=False)
    email = Column(String(100), unique=True, nullable=False)
    password_hash = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.now)


class SQLAlchemyUserRepository(AbstractRepository[User]):
    """Реализация репозитория пользователей с SQLAlchemy."""

    def __init__(self, session: Session):
        self.session = session

    def _to_domain(self, model: UserModel) -> User:
        """Преобразование ORM модели в доменную сущность."""
        return User(
            id=model.id,
            username=model.username,
            email=model.email,
            password_hash=model.password_hash,
            is_active=model.is_active,
            created_at=model.created_at
        )

    def _to_model(self, entity: User) -> UserModel:
        """Преобразование доменной сущности в ORM модель."""
        return UserModel(
            id=entity.id,
            username=entity.username,
            email=entity.email,
            password_hash=entity.password_hash,
            is_active=entity.is_active,
            created_at=entity.created_at
        )

    def add(self, entity: User) -> None:
        model = self._to_model(entity)
        self.session.add(model)
        self.session.flush()  # Получаем ID
        entity.id = model.id

    def get(self, id: int) -> Optional[User]:
        model = self.session.query(UserModel).filter_by(id=id).first()
        return self._to_domain(model) if model else None

    def get_all(self) -> List[User]:
        models = self.session.query(UserModel).all()
        return [self._to_domain(m) for m in models]

    def update(self, entity: User) -> None:
        model = self.session.query(UserModel).filter_by(id=entity.id).first()
        if model:
            model.username = entity.username
            model.email = entity.email
            model.password_hash = entity.password_hash
            model.is_active = entity.is_active

    def delete(self, id: int) -> None:
        model = self.session.query(UserModel).filter_by(id=id).first()
        if model:
            self.session.delete(model)

    # Дополнительные методы, специфичные для User
    def get_by_username(self, username: str) -> Optional[User]:
        model = self.session.query(UserModel).filter_by(username=username).first()
        return self._to_domain(model) if model else None

    def get_by_email(self, email: str) -> Optional[User]:
        model = self.session.query(UserModel).filter_by(email=email).first()
        return self._to_domain(model) if model else None

    def get_active_users(self) -> List[User]:
        models = self.session.query(UserModel).filter_by(is_active=True).all()
        return [self._to_domain(m) for m in models]
```

## Репозиторий для тестирования

```python
from typing import Dict, List, Optional


class InMemoryUserRepository(AbstractRepository[User]):
    """In-memory репозиторий для тестов."""

    def __init__(self):
        self._storage: Dict[int, User] = {}
        self._next_id = 1

    def add(self, entity: User) -> None:
        if entity.id is None:
            entity.id = self._next_id
            self._next_id += 1
        self._storage[entity.id] = entity

    def get(self, id: int) -> Optional[User]:
        return self._storage.get(id)

    def get_all(self) -> List[User]:
        return list(self._storage.values())

    def update(self, entity: User) -> None:
        if entity.id in self._storage:
            self._storage[entity.id] = entity

    def delete(self, id: int) -> None:
        self._storage.pop(id, None)

    def get_by_username(self, username: str) -> Optional[User]:
        for user in self._storage.values():
            if user.username == username:
                return user
        return None

    def get_by_email(self, email: str) -> Optional[User]:
        for user in self._storage.values():
            if user.email == email:
                return user
        return None

    def get_active_users(self) -> List[User]:
        return [u for u in self._storage.values() if u.is_active]
```

## Использование репозитория в сервисе

```python
from typing import Optional
from dataclasses import dataclass


@dataclass
class CreateUserDTO:
    username: str
    email: str
    password: str


class UserService:
    """Сервис для работы с пользователями."""

    def __init__(self, user_repository: AbstractRepository[User]):
        self.repository = user_repository

    def create_user(self, dto: CreateUserDTO) -> User:
        # Проверка уникальности
        if self.repository.get_by_username(dto.username):
            raise ValueError(f"Username '{dto.username}' already exists")

        if self.repository.get_by_email(dto.email):
            raise ValueError(f"Email '{dto.email}' already exists")

        # Создание сущности
        user = User(
            id=None,
            username=dto.username,
            email=dto.email,
            password_hash=self._hash_password(dto.password)
        )

        # Сохранение через репозиторий
        self.repository.add(user)
        return user

    def deactivate_user(self, user_id: int) -> Optional[User]:
        user = self.repository.get(user_id)
        if user:
            user.deactivate()  # Бизнес-логика в доменной сущности
            self.repository.update(user)
        return user

    def get_user(self, user_id: int) -> Optional[User]:
        return self.repository.get(user_id)

    def _hash_password(self, password: str) -> str:
        import hashlib
        return hashlib.sha256(password.encode()).hexdigest()


# Использование
# В продакшене:
# repository = SQLAlchemyUserRepository(session)

# В тестах:
# repository = InMemoryUserRepository()

# service = UserService(repository)
```

## Спецификации (Specification Pattern)

Для сложных запросов используйте паттерн Specification:

```python
from abc import ABC, abstractmethod
from typing import TypeVar, Generic, List

T = TypeVar("T")


class Specification(ABC, Generic[T]):
    """Базовая спецификация."""

    @abstractmethod
    def is_satisfied_by(self, entity: T) -> bool:
        """Проверяет, удовлетворяет ли сущность условию."""
        raise NotImplementedError

    def __and__(self, other: "Specification[T]") -> "AndSpecification[T]":
        return AndSpecification(self, other)

    def __or__(self, other: "Specification[T]") -> "OrSpecification[T]":
        return OrSpecification(self, other)

    def __invert__(self) -> "NotSpecification[T]":
        return NotSpecification(self)


class AndSpecification(Specification[T]):
    def __init__(self, left: Specification[T], right: Specification[T]):
        self.left = left
        self.right = right

    def is_satisfied_by(self, entity: T) -> bool:
        return self.left.is_satisfied_by(entity) and self.right.is_satisfied_by(entity)


class OrSpecification(Specification[T]):
    def __init__(self, left: Specification[T], right: Specification[T]):
        self.left = left
        self.right = right

    def is_satisfied_by(self, entity: T) -> bool:
        return self.left.is_satisfied_by(entity) or self.right.is_satisfied_by(entity)


class NotSpecification(Specification[T]):
    def __init__(self, spec: Specification[T]):
        self.spec = spec

    def is_satisfied_by(self, entity: T) -> bool:
        return not self.spec.is_satisfied_by(entity)


# Конкретные спецификации
class IsActiveUserSpecification(Specification[User]):
    def is_satisfied_by(self, user: User) -> bool:
        return user.is_active


class EmailDomainSpecification(Specification[User]):
    def __init__(self, domain: str):
        self.domain = domain

    def is_satisfied_by(self, user: User) -> bool:
        return user.email.endswith(f"@{self.domain}")


# Расширенный репозиторий
class SpecificationRepository(AbstractRepository[T]):
    @abstractmethod
    def find_by_specification(self, spec: Specification[T]) -> List[T]:
        """Найти сущности по спецификации."""
        raise NotImplementedError


class InMemoryUserRepositoryWithSpec(InMemoryUserRepository, SpecificationRepository[User]):
    def find_by_specification(self, spec: Specification[User]) -> List[User]:
        return [user for user in self._storage.values() if spec.is_satisfied_by(user)]


# Использование
# repo = InMemoryUserRepositoryWithSpec()
# active_spec = IsActiveUserSpecification()
# gmail_spec = EmailDomainSpecification("gmail.com")
#
# # Активные пользователи с Gmail
# active_gmail_users = repo.find_by_specification(active_spec & gmail_spec)
```

## Репозиторий с пагинацией и сортировкой

```python
from dataclasses import dataclass
from typing import List, Optional, Generic, TypeVar
from enum import Enum

T = TypeVar("T")


class SortOrder(Enum):
    ASC = "asc"
    DESC = "desc"


@dataclass
class SortCriteria:
    field: str
    order: SortOrder = SortOrder.ASC


@dataclass
class PaginationParams:
    page: int = 1
    per_page: int = 10


@dataclass
class PaginatedResult(Generic[T]):
    items: List[T]
    total: int
    page: int
    per_page: int
    total_pages: int

    @property
    def has_next(self) -> bool:
        return self.page < self.total_pages

    @property
    def has_prev(self) -> bool:
        return self.page > 1


class PaginatedRepository(AbstractRepository[T]):
    """Репозиторий с пагинацией."""

    @abstractmethod
    def find_paginated(
        self,
        pagination: PaginationParams,
        sort: Optional[SortCriteria] = None,
        **filters
    ) -> PaginatedResult[T]:
        raise NotImplementedError


class SQLAlchemyPaginatedUserRepository(SQLAlchemyUserRepository, PaginatedRepository[User]):
    """Реализация с пагинацией."""

    def find_paginated(
        self,
        pagination: PaginationParams,
        sort: Optional[SortCriteria] = None,
        **filters
    ) -> PaginatedResult[User]:
        query = self.session.query(UserModel)

        # Применяем фильтры
        if "is_active" in filters:
            query = query.filter_by(is_active=filters["is_active"])

        # Считаем общее количество
        total = query.count()

        # Сортировка
        if sort:
            column = getattr(UserModel, sort.field, None)
            if column:
                if sort.order == SortOrder.DESC:
                    column = column.desc()
                query = query.order_by(column)

        # Пагинация
        offset = (pagination.page - 1) * pagination.per_page
        query = query.offset(offset).limit(pagination.per_page)

        models = query.all()
        items = [self._to_domain(m) for m in models]

        total_pages = (total + pagination.per_page - 1) // pagination.per_page

        return PaginatedResult(
            items=items,
            total=total,
            page=pagination.page,
            per_page=pagination.per_page,
            total_pages=total_pages
        )
```

## Сравнение с альтернативными подходами

| Подход | Описание | Когда использовать |
|--------|----------|-------------------|
| **Repository** | Абстракция коллекции | DDD, сложная бизнес-логика |
| **Active Record** | Сущность сама себя сохраняет | Простые CRUD |
| **DAO** | Низкоуровневый доступ к данным | Прямая работа с БД |
| **Query Object** | Объектное представление запроса | Сложные динамические запросы |

## Плюсы и минусы

### Преимущества

1. **Инкапсуляция** — логика доступа к данным скрыта
2. **Тестируемость** — легко создавать mock-объекты
3. **Гибкость** — легко менять хранилище данных
4. **Чистый домен** — доменные объекты не знают о персистентности
5. **Переиспользование** — типичные запросы в методах репозитория

### Недостатки

1. **Накладные расходы** — дополнительный слой абстракции
2. **Сложность** — много boilerplate кода
3. **Риск утечки** — ORM-специфичные конструкции могут просочиться
4. **Дублирование** — похожие методы в разных репозиториях

## Когда использовать Repository?

### Рекомендуется:
- Domain-Driven Design (DDD)
- Сложная бизнес-логика
- Необходимость тестирования без БД
- Возможность смены хранилища
- Множество сложных запросов

### Не рекомендуется:
- Простые CRUD-приложения
- Прототипы и MVP
- Когда Active Record достаточен

## Лучшие практики

1. **Один репозиторий — один агрегат** в терминах DDD
2. **Не возвращайте IQueryable** — это утечка абстракции
3. **Используйте спецификации** для сложных условий
4. **Избегайте generic-репозиториев** — они часто слишком абстрактны
5. **Репозиторий работает с доменными объектами**, а не с ORM-моделями

## Заключение

Repository — один из ключевых паттернов для построения чистой архитектуры. Он обеспечивает разделение домена и инфраструктуры, упрощает тестирование и делает код более гибким к изменениям. Однако для простых приложений паттерн может быть избыточным — важно оценивать сложность проекта перед его применением.
