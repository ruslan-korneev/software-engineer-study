# Unit of Work

## Что такое Unit of Work?

**Unit of Work (Единица работы)** — это паттерн проектирования, который отслеживает все изменения объектов в рамках бизнес-транзакции и координирует запись этих изменений в базу данных. Он гарантирует атомарность операций: либо все изменения сохраняются, либо ни одно.

> Паттерн описан Мартином Фаулером в "Patterns of Enterprise Application Architecture" и тесно связан с паттернами Repository и Identity Map.

## Основная идея

Unit of Work накапливает изменения объектов и фиксирует их одной транзакцией:

```
Бизнес-операция
    │
    ├── Создать User
    ├── Обновить Order
    └── Удалить Product
    │
    v
Unit of Work (commit)
    │
    v
База данных (одна транзакция)
```

## Зачем нужен Unit of Work?

### Основные преимущества:

1. **Атомарность** — все изменения в рамках транзакции или ничего
2. **Консистентность** — гарантирует целостность данных
3. **Оптимизация** — минимизирует количество обращений к БД
4. **Отложенная запись** — изменения накапливаются и записываются одним batch
5. **Откат изменений** — легко отменить все изменения при ошибке

## Базовая реализация

### Абстрактный Unit of Work

```python
from abc import ABC, abstractmethod
from typing import TypeVar


class AbstractUnitOfWork(ABC):
    """Абстрактный Unit of Work."""

    @abstractmethod
    def __enter__(self):
        """Начало транзакции."""
        raise NotImplementedError

    @abstractmethod
    def __exit__(self, exc_type, exc_val, exc_tb):
        """Завершение транзакции с откатом при ошибке."""
        raise NotImplementedError

    @abstractmethod
    def commit(self):
        """Фиксация изменений."""
        raise NotImplementedError

    @abstractmethod
    def rollback(self):
        """Откат изменений."""
        raise NotImplementedError
```

### Реализация с SQLAlchemy

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from contextlib import contextmanager


class SQLAlchemyUnitOfWork(AbstractUnitOfWork):
    """Unit of Work с SQLAlchemy."""

    def __init__(self, session_factory: sessionmaker):
        self.session_factory = session_factory
        self.session: Session = None

    def __enter__(self):
        self.session = self.session_factory()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is not None:
            self.rollback()
        self.session.close()

    def commit(self):
        """Фиксация всех изменений."""
        try:
            self.session.commit()
        except Exception:
            self.rollback()
            raise

    def rollback(self):
        """Откат всех изменений."""
        self.session.rollback()


# Настройка
engine = create_engine("postgresql://localhost/mydb")
SessionFactory = sessionmaker(bind=engine)

# Использование
# with SQLAlchemyUnitOfWork(SessionFactory) as uow:
#     # работа с репозиториями
#     uow.commit()
```

## Unit of Work с репозиториями

### Полная реализация

```python
from abc import ABC, abstractmethod
from sqlalchemy.orm import Session, sessionmaker
from typing import Optional


# Абстрактные репозитории
class AbstractUserRepository(ABC):
    @abstractmethod
    def add(self, user) -> None:
        pass

    @abstractmethod
    def get(self, id: int):
        pass


class AbstractOrderRepository(ABC):
    @abstractmethod
    def add(self, order) -> None:
        pass

    @abstractmethod
    def get(self, id: int):
        pass


# Конкретные репозитории
class SQLAlchemyUserRepository(AbstractUserRepository):
    def __init__(self, session: Session):
        self.session = session

    def add(self, user) -> None:
        self.session.add(user)

    def get(self, id: int):
        return self.session.query(UserModel).filter_by(id=id).first()


class SQLAlchemyOrderRepository(AbstractOrderRepository):
    def __init__(self, session: Session):
        self.session = session

    def add(self, order) -> None:
        self.session.add(order)

    def get(self, id: int):
        return self.session.query(OrderModel).filter_by(id=id).first()


# Unit of Work с репозиториями
class AbstractUnitOfWork(ABC):
    users: AbstractUserRepository
    orders: AbstractOrderRepository

    @abstractmethod
    def __enter__(self):
        raise NotImplementedError

    @abstractmethod
    def __exit__(self, exc_type, exc_val, exc_tb):
        raise NotImplementedError

    @abstractmethod
    def commit(self):
        raise NotImplementedError

    @abstractmethod
    def rollback(self):
        raise NotImplementedError


class SQLAlchemyUnitOfWork(AbstractUnitOfWork):
    """Полная реализация Unit of Work."""

    def __init__(self, session_factory: sessionmaker):
        self.session_factory = session_factory

    def __enter__(self):
        self.session = self.session_factory()
        # Инициализируем репозитории с текущей сессией
        self.users = SQLAlchemyUserRepository(self.session)
        self.orders = SQLAlchemyOrderRepository(self.session)
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is not None:
            self.rollback()
        self.session.close()

    def commit(self):
        self.session.commit()

    def rollback(self):
        self.session.rollback()
```

## Использование в сервисном слое

```python
from dataclasses import dataclass
from typing import Optional


@dataclass
class CreateOrderDTO:
    user_id: int
    product_ids: list[int]
    quantities: list[int]


class OrderService:
    """Сервис заказов с Unit of Work."""

    def __init__(self, uow_factory):
        self.uow_factory = uow_factory

    def create_order(self, dto: CreateOrderDTO) -> Order:
        """Создание заказа в транзакции."""
        with self.uow_factory() as uow:
            # Получаем пользователя
            user = uow.users.get(dto.user_id)
            if not user:
                raise ValueError(f"User {dto.user_id} not found")

            # Создаём заказ
            order = Order(
                user_id=dto.user_id,
                status="pending"
            )
            uow.orders.add(order)

            # Добавляем позиции заказа
            for product_id, qty in zip(dto.product_ids, dto.quantities):
                product = uow.products.get(product_id)
                if not product:
                    raise ValueError(f"Product {product_id} not found")

                # Проверяем наличие
                if product.stock < qty:
                    raise ValueError(f"Insufficient stock for {product.name}")

                # Уменьшаем запас
                product.stock -= qty

                # Добавляем позицию
                order_item = OrderItem(
                    order=order,
                    product_id=product_id,
                    quantity=qty,
                    price=product.price
                )
                uow.order_items.add(order_item)

            # Фиксируем все изменения одной транзакцией
            uow.commit()
            return order


# Использование
# uow_factory = lambda: SQLAlchemyUnitOfWork(SessionFactory)
# service = OrderService(uow_factory)
# order = service.create_order(dto)
```

## Отслеживание изменений

### Unit of Work с Identity Map

```python
from typing import Dict, Set, Optional
from dataclasses import dataclass, field
from enum import Enum


class ObjectState(Enum):
    NEW = "new"
    DIRTY = "dirty"
    DELETED = "deleted"
    CLEAN = "clean"


@dataclass
class TrackedObject:
    """Отслеживаемый объект."""
    obj: object
    state: ObjectState
    original_data: dict = field(default_factory=dict)


class TrackingUnitOfWork:
    """Unit of Work с отслеживанием изменений."""

    def __init__(self):
        self._identity_map: Dict[tuple, TrackedObject] = {}
        self._new: Set[object] = set()
        self._dirty: Set[object] = set()
        self._deleted: Set[object] = set()

    def register_new(self, obj: object) -> None:
        """Регистрация нового объекта."""
        self._new.add(obj)

    def register_dirty(self, obj: object) -> None:
        """Регистрация изменённого объекта."""
        if obj not in self._new:
            self._dirty.add(obj)

    def register_deleted(self, obj: object) -> None:
        """Регистрация удалённого объекта."""
        if obj in self._new:
            self._new.remove(obj)
        else:
            self._dirty.discard(obj)
            self._deleted.add(obj)

    def register_clean(self, obj: object) -> None:
        """Регистрация загруженного из БД объекта."""
        key = self._get_key(obj)
        self._identity_map[key] = TrackedObject(
            obj=obj,
            state=ObjectState.CLEAN,
            original_data=self._snapshot(obj)
        )

    def get_identity(self, entity_type: type, id: int) -> Optional[object]:
        """Получение объекта из Identity Map."""
        key = (entity_type, id)
        tracked = self._identity_map.get(key)
        return tracked.obj if tracked else None

    def commit(self, db_session):
        """Фиксация всех изменений."""
        try:
            # INSERT новых объектов
            for obj in self._new:
                db_session.add(obj)

            # UPDATE изменённых объектов
            for obj in self._dirty:
                db_session.merge(obj)

            # DELETE удалённых объектов
            for obj in self._deleted:
                db_session.delete(obj)

            db_session.commit()
            self._clear()
        except Exception:
            db_session.rollback()
            raise

    def rollback(self):
        """Откат всех изменений."""
        # Восстанавливаем оригинальные данные
        for key, tracked in self._identity_map.items():
            if tracked.state == ObjectState.DIRTY:
                self._restore(tracked.obj, tracked.original_data)
        self._clear()

    def _clear(self):
        """Очистка состояния."""
        self._new.clear()
        self._dirty.clear()
        self._deleted.clear()

    def _get_key(self, obj: object) -> tuple:
        """Получение ключа для Identity Map."""
        return (type(obj), getattr(obj, 'id', None))

    def _snapshot(self, obj: object) -> dict:
        """Создание снимка объекта."""
        return {k: v for k, v in obj.__dict__.items() if not k.startswith('_')}

    def _restore(self, obj: object, data: dict) -> None:
        """Восстановление объекта из снимка."""
        for k, v in data.items():
            setattr(obj, k, v)
```

## Асинхронный Unit of Work

```python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker
from abc import ABC, abstractmethod


class AbstractAsyncUnitOfWork(ABC):
    """Абстрактный асинхронный Unit of Work."""

    @abstractmethod
    async def __aenter__(self):
        raise NotImplementedError

    @abstractmethod
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        raise NotImplementedError

    @abstractmethod
    async def commit(self):
        raise NotImplementedError

    @abstractmethod
    async def rollback(self):
        raise NotImplementedError


class AsyncSQLAlchemyUnitOfWork(AbstractAsyncUnitOfWork):
    """Асинхронный Unit of Work с SQLAlchemy."""

    def __init__(self, session_factory: async_sessionmaker[AsyncSession]):
        self.session_factory = session_factory

    async def __aenter__(self):
        self.session = self.session_factory()
        self.users = AsyncUserRepository(self.session)
        self.orders = AsyncOrderRepository(self.session)
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if exc_type is not None:
            await self.rollback()
        await self.session.close()

    async def commit(self):
        await self.session.commit()

    async def rollback(self):
        await self.session.rollback()


# Использование
async def create_order_async(dto: CreateOrderDTO):
    async with AsyncSQLAlchemyUnitOfWork(async_session_factory) as uow:
        user = await uow.users.get(dto.user_id)
        order = Order(user_id=user.id, status="pending")
        await uow.orders.add(order)
        await uow.commit()
        return order
```

## Unit of Work для тестирования

```python
from typing import Dict, List


class FakeUnitOfWork(AbstractUnitOfWork):
    """Fake Unit of Work для тестов."""

    def __init__(self):
        self.committed = False
        self.rolled_back = False

    def __enter__(self):
        self.users = FakeUserRepository()
        self.orders = FakeOrderRepository()
        self.committed = False
        self.rolled_back = False
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is not None:
            self.rollback()

    def commit(self):
        self.committed = True

    def rollback(self):
        self.rolled_back = True


class FakeUserRepository:
    def __init__(self):
        self._users: Dict[int, User] = {}
        self._next_id = 1

    def add(self, user: User) -> None:
        if user.id is None:
            user.id = self._next_id
            self._next_id += 1
        self._users[user.id] = user

    def get(self, id: int) -> Optional[User]:
        return self._users.get(id)


# Тестирование
def test_create_order_commits_transaction():
    uow = FakeUnitOfWork()
    service = OrderService(lambda: uow)

    # Подготовка данных
    with uow:
        uow.users.add(User(id=1, username="test"))
        uow.commit()

    # Тест
    dto = CreateOrderDTO(user_id=1, product_ids=[1], quantities=[1])
    with uow:
        service.create_order(dto)

    assert uow.committed is True
    assert uow.rolled_back is False
```

## Паттерн с событиями домена

```python
from dataclasses import dataclass, field
from typing import List, Callable
from datetime import datetime


@dataclass
class DomainEvent:
    """Базовое доменное событие."""
    occurred_at: datetime = field(default_factory=datetime.now)


@dataclass
class OrderCreatedEvent(DomainEvent):
    order_id: int
    user_id: int


@dataclass
class StockDepletedEvent(DomainEvent):
    product_id: int


class UnitOfWorkWithEvents(SQLAlchemyUnitOfWork):
    """Unit of Work с поддержкой доменных событий."""

    def __init__(self, session_factory, event_handlers: List[Callable] = None):
        super().__init__(session_factory)
        self._events: List[DomainEvent] = []
        self._event_handlers = event_handlers or []

    def add_event(self, event: DomainEvent) -> None:
        """Добавление события."""
        self._events.append(event)

    def commit(self):
        """Фиксация с публикацией событий."""
        super().commit()
        # Публикуем события после успешного коммита
        self._publish_events()

    def _publish_events(self):
        """Публикация накопленных событий."""
        for event in self._events:
            for handler in self._event_handlers:
                try:
                    handler(event)
                except Exception as e:
                    # Логируем ошибку, но не откатываем транзакцию
                    print(f"Event handler error: {e}")
        self._events.clear()


# Использование
def on_order_created(event: OrderCreatedEvent):
    print(f"Order {event.order_id} created for user {event.user_id}")
    # Отправить email, уведомление и т.д.


def on_stock_depleted(event: StockDepletedEvent):
    print(f"Product {event.product_id} out of stock!")
    # Создать задачу на пополнение склада


# uow = UnitOfWorkWithEvents(
#     SessionFactory,
#     event_handlers=[on_order_created, on_stock_depleted]
# )
```

## Сравнение подходов

| Подход | Описание | Транзакции |
|--------|----------|------------|
| **Unit of Work** | Накопление и batch-запись | Явные, контролируемые |
| **Active Record** | Немедленное сохранение | Неявные, по одной операции |
| **Auto-commit** | Каждая операция = транзакция | Автоматические |

## Плюсы и минусы

### Преимущества

1. **Атомарность** — все изменения или ничего
2. **Производительность** — batch-операции вместо множества запросов
3. **Консистентность** — целостность данных гарантирована
4. **Изоляция** — бизнес-логика не знает о транзакциях
5. **Отслеживание** — можно знать, что изменилось

### Недостатки

1. **Сложность** — дополнительный уровень абстракции
2. **Память** — хранение изменений до коммита
3. **Длинные транзакции** — могут вызвать блокировки
4. **Отладка** — сложнее понять, когда происходит запись

## Когда использовать?

### Рекомендуется:
- Сложные бизнес-операции с несколькими сущностями
- Требуется гарантия атомарности
- Оптимизация количества запросов к БД
- Domain-Driven Design

### Не рекомендуется:
- Простые CRUD-операции
- Одиночные изменения
- Когда ORM уже управляет транзакциями

## Лучшие практики

1. **Короткие транзакции** — не держите UoW открытым долго
2. **Один UoW на запрос** — типичный паттерн для веб-приложений
3. **Явный commit** — не полагайтесь на автоматический
4. **Обработка ошибок** — всегда обрабатывайте исключения
5. **Не смешивайте** — один UoW для одной бизнес-операции

## Заключение

Unit of Work — это мощный паттерн для управления транзакциями и изменениями в приложении. Он особенно ценен в сложных бизнес-сценариях, где нужна атомарность и оптимизация работы с базой данных. SQLAlchemy Session уже реализует этот паттерн, поэтому обёртка над ней добавляет удобный интерфейс и интеграцию с репозиториями.
