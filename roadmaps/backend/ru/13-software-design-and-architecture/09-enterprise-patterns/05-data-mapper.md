# Data Mapper

## Что такое Data Mapper?

**Data Mapper** — это паттерн проектирования, который отделяет доменные объекты от логики работы с базой данных. Маппер выступает посредником между объектами в памяти и записями в БД, перенося данные между ними и изолируя их друг от друга.

> Паттерн описан Мартином Фаулером в "Patterns of Enterprise Application Architecture" и является альтернативой паттерну Active Record.

## Основная идея

Data Mapper полностью разделяет домен и персистентность:

```
Доменный объект          Data Mapper          База данных
     User        <-->     UserMapper     <-->    users
(чистая логика)       (маппинг/запросы)       (хранение)
```

Доменный объект ничего не знает о том, как он сохраняется — это ответственность маппера.

## Зачем нужен Data Mapper?

### Преимущества разделения:

1. **Чистый домен** — бизнес-логика не загрязнена SQL и ORM
2. **Тестируемость** — домен можно тестировать без БД
3. **Гибкость** — легко менять схему БД или ORM
4. **Независимость** — домен и хранилище развиваются отдельно
5. **Сложные маппинги** — поддержка разных схем маппинга

## Сравнение с Active Record

```python
# Active Record — объект сам себя сохраняет
class User(ActiveRecord):
    def save(self):
        # SQL внутри доменного объекта
        db.execute("INSERT INTO users ...")

user = User(name="Alice")
user.save()  # Объект знает о БД


# Data Mapper — отдельный маппер
class User:
    def __init__(self, name: str):
        self.name = name
    # Никакого SQL!

class UserMapper:
    def save(self, user: User):
        db.execute("INSERT INTO users ...", user.name)

user = User(name="Alice")
mapper = UserMapper()
mapper.save(user)  # Маппер знает о БД, объект — нет
```

## Базовая реализация

### Доменная сущность

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional, List


@dataclass
class User:
    """Чистая доменная сущность — никаких зависимостей от БД."""
    id: Optional[int] = None
    username: str = ""
    email: str = ""
    is_active: bool = True
    created_at: datetime = field(default_factory=datetime.now)

    def deactivate(self):
        """Бизнес-логика."""
        self.is_active = False

    def can_login(self) -> bool:
        """Бизнес-правило."""
        return self.is_active

    def change_email(self, new_email: str):
        """Изменение email с валидацией."""
        if "@" not in new_email:
            raise ValueError("Invalid email format")
        self.email = new_email
```

### Data Mapper

```python
from typing import Optional, List


class UserMapper:
    """Маппер для преобразования User <-> БД."""

    def __init__(self, connection):
        self.connection = connection

    def find_by_id(self, id: int) -> Optional[User]:
        """Загрузка пользователя по ID."""
        cursor = self.connection.execute(
            "SELECT id, username, email, is_active, created_at "
            "FROM users WHERE id = ?",
            (id,)
        )
        row = cursor.fetchone()

        if row is None:
            return None

        return self._map_row_to_user(row)

    def find_by_username(self, username: str) -> Optional[User]:
        """Поиск по имени пользователя."""
        cursor = self.connection.execute(
            "SELECT id, username, email, is_active, created_at "
            "FROM users WHERE username = ?",
            (username,)
        )
        row = cursor.fetchone()
        return self._map_row_to_user(row) if row else None

    def find_all(self) -> List[User]:
        """Получение всех пользователей."""
        cursor = self.connection.execute(
            "SELECT id, username, email, is_active, created_at FROM users"
        )
        return [self._map_row_to_user(row) for row in cursor.fetchall()]

    def find_active(self) -> List[User]:
        """Получение активных пользователей."""
        cursor = self.connection.execute(
            "SELECT id, username, email, is_active, created_at "
            "FROM users WHERE is_active = 1"
        )
        return [self._map_row_to_user(row) for row in cursor.fetchall()]

    def insert(self, user: User) -> None:
        """Вставка нового пользователя."""
        cursor = self.connection.execute(
            "INSERT INTO users (username, email, is_active, created_at) "
            "VALUES (?, ?, ?, ?)",
            (user.username, user.email, user.is_active, user.created_at)
        )
        user.id = cursor.lastrowid

    def update(self, user: User) -> None:
        """Обновление существующего пользователя."""
        self.connection.execute(
            "UPDATE users SET username = ?, email = ?, is_active = ? "
            "WHERE id = ?",
            (user.username, user.email, user.is_active, user.id)
        )

    def delete(self, user: User) -> None:
        """Удаление пользователя."""
        self.connection.execute(
            "DELETE FROM users WHERE id = ?",
            (user.id,)
        )

    def _map_row_to_user(self, row) -> User:
        """Преобразование строки БД в доменный объект."""
        return User(
            id=row[0],
            username=row[1],
            email=row[2],
            is_active=bool(row[3]),
            created_at=datetime.fromisoformat(row[4]) if row[4] else datetime.now()
        )
```

## Data Mapper с SQLAlchemy

### Классический подход (Declarative)

```python
from sqlalchemy import Column, Integer, String, Boolean, DateTime, create_engine
from sqlalchemy.orm import declarative_base, Session, sessionmaker
from datetime import datetime

Base = declarative_base()


# ORM модель (Data Mapper Layer)
class UserModel(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    username = Column(String(50), unique=True)
    email = Column(String(100))
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.now)


# Доменная сущность (чистая)
@dataclass
class User:
    id: Optional[int] = None
    username: str = ""
    email: str = ""
    is_active: bool = True
    created_at: datetime = field(default_factory=datetime.now)

    def deactivate(self):
        self.is_active = False


# Data Mapper
class SQLAlchemyUserMapper:
    """Маппер между User и UserModel."""

    def __init__(self, session: Session):
        self.session = session

    def find_by_id(self, id: int) -> Optional[User]:
        model = self.session.query(UserModel).filter_by(id=id).first()
        return self._to_domain(model) if model else None

    def find_all(self) -> List[User]:
        models = self.session.query(UserModel).all()
        return [self._to_domain(m) for m in models]

    def save(self, user: User) -> None:
        if user.id is None:
            model = self._to_model(user)
            self.session.add(model)
            self.session.flush()
            user.id = model.id
        else:
            model = self.session.query(UserModel).filter_by(id=user.id).first()
            if model:
                self._update_model(model, user)

    def delete(self, user: User) -> None:
        model = self.session.query(UserModel).filter_by(id=user.id).first()
        if model:
            self.session.delete(model)

    def _to_domain(self, model: UserModel) -> User:
        """ORM модель -> Доменная сущность."""
        return User(
            id=model.id,
            username=model.username,
            email=model.email,
            is_active=model.is_active,
            created_at=model.created_at
        )

    def _to_model(self, user: User) -> UserModel:
        """Доменная сущность -> ORM модель."""
        return UserModel(
            id=user.id,
            username=user.username,
            email=user.email,
            is_active=user.is_active,
            created_at=user.created_at
        )

    def _update_model(self, model: UserModel, user: User) -> None:
        """Обновление ORM модели из доменной сущности."""
        model.username = user.username
        model.email = user.email
        model.is_active = user.is_active
```

### Императивный маппинг (SQLAlchemy)

```python
from sqlalchemy import Table, Column, Integer, String, Boolean, DateTime, MetaData
from sqlalchemy.orm import registry

mapper_registry = registry()
metadata = MetaData()

# Определяем таблицу отдельно
users_table = Table(
    "users",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("username", String(50)),
    Column("email", String(100)),
    Column("is_active", Boolean, default=True),
    Column("created_at", DateTime),
)


# Чистая доменная сущность
class User:
    def __init__(
        self,
        id: int = None,
        username: str = "",
        email: str = "",
        is_active: bool = True,
        created_at: datetime = None
    ):
        self.id = id
        self.username = username
        self.email = email
        self.is_active = is_active
        self.created_at = created_at or datetime.now()

    def deactivate(self):
        self.is_active = False


# Императивный маппинг — связываем класс с таблицей
mapper_registry.map_imperatively(User, users_table)


# Теперь SQLAlchemy автоматически маппит User <-> users
def example_usage():
    engine = create_engine("sqlite:///example.db")
    Session = sessionmaker(bind=engine)
    session = Session()

    # Работаем с доменными объектами напрямую
    user = User(username="alice", email="alice@example.com")
    session.add(user)
    session.commit()

    # Загрузка
    loaded_user = session.query(User).filter_by(username="alice").first()
    loaded_user.deactivate()
    session.commit()
```

## Сложный маппинг

### Маппинг с агрегатами

```python
from dataclasses import dataclass, field
from typing import List, Optional


# Доменные сущности
@dataclass
class OrderItem:
    id: Optional[int] = None
    product_name: str = ""
    quantity: int = 0
    price: float = 0.0

    @property
    def total(self) -> float:
        return self.quantity * self.price


@dataclass
class Order:
    id: Optional[int] = None
    customer_id: int = 0
    status: str = "pending"
    items: List[OrderItem] = field(default_factory=list)

    @property
    def total_amount(self) -> float:
        return sum(item.total for item in self.items)

    def add_item(self, item: OrderItem):
        self.items.append(item)

    def complete(self):
        if not self.items:
            raise ValueError("Cannot complete empty order")
        self.status = "completed"


# Маппер для агрегата
class OrderMapper:
    """Маппер для агрегата Order с вложенными OrderItem."""

    def __init__(self, connection):
        self.connection = connection

    def find_by_id(self, id: int) -> Optional[Order]:
        # Загружаем заказ
        order_row = self.connection.execute(
            "SELECT id, customer_id, status FROM orders WHERE id = ?",
            (id,)
        ).fetchone()

        if not order_row:
            return None

        order = Order(
            id=order_row[0],
            customer_id=order_row[1],
            status=order_row[2]
        )

        # Загружаем позиции
        item_rows = self.connection.execute(
            "SELECT id, product_name, quantity, price "
            "FROM order_items WHERE order_id = ?",
            (id,)
        ).fetchall()

        for row in item_rows:
            order.items.append(OrderItem(
                id=row[0],
                product_name=row[1],
                quantity=row[2],
                price=row[3]
            ))

        return order

    def save(self, order: Order) -> None:
        if order.id is None:
            self._insert(order)
        else:
            self._update(order)

    def _insert(self, order: Order) -> None:
        cursor = self.connection.execute(
            "INSERT INTO orders (customer_id, status) VALUES (?, ?)",
            (order.customer_id, order.status)
        )
        order.id = cursor.lastrowid

        for item in order.items:
            cursor = self.connection.execute(
                "INSERT INTO order_items (order_id, product_name, quantity, price) "
                "VALUES (?, ?, ?, ?)",
                (order.id, item.product_name, item.quantity, item.price)
            )
            item.id = cursor.lastrowid

    def _update(self, order: Order) -> None:
        self.connection.execute(
            "UPDATE orders SET customer_id = ?, status = ? WHERE id = ?",
            (order.customer_id, order.status, order.id)
        )

        # Удаляем старые позиции и вставляем новые
        self.connection.execute(
            "DELETE FROM order_items WHERE order_id = ?",
            (order.id,)
        )

        for item in order.items:
            cursor = self.connection.execute(
                "INSERT INTO order_items (order_id, product_name, quantity, price) "
                "VALUES (?, ?, ?, ?)",
                (order.id, item.product_name, item.quantity, item.price)
            )
            item.id = cursor.lastrowid
```

## Абстрактный Data Mapper

```python
from abc import ABC, abstractmethod
from typing import TypeVar, Generic, Optional, List

T = TypeVar("T")


class AbstractMapper(ABC, Generic[T]):
    """Базовый абстрактный маппер."""

    @abstractmethod
    def find_by_id(self, id: int) -> Optional[T]:
        """Найти сущность по ID."""
        pass

    @abstractmethod
    def find_all(self) -> List[T]:
        """Получить все сущности."""
        pass

    @abstractmethod
    def save(self, entity: T) -> None:
        """Сохранить сущность (insert или update)."""
        pass

    @abstractmethod
    def delete(self, entity: T) -> None:
        """Удалить сущность."""
        pass


class UserMapper(AbstractMapper[User]):
    """Конкретный маппер для User."""

    def __init__(self, session):
        self.session = session

    def find_by_id(self, id: int) -> Optional[User]:
        # реализация
        pass

    def find_all(self) -> List[User]:
        # реализация
        pass

    def save(self, user: User) -> None:
        # реализация
        pass

    def delete(self, user: User) -> None:
        # реализация
        pass
```

## Сравнение Data Mapper и Active Record

| Аспект | Data Mapper | Active Record |
|--------|-------------|---------------|
| **Разделение ответственности** | Полное | Смешанное |
| **Сложность** | Выше | Ниже |
| **Тестируемость домена** | Отличная | Требует моков БД |
| **Гибкость маппинга** | Высокая | Ограниченная |
| **Boilerplate код** | Больше | Меньше |
| **Подходит для** | Сложный домен | Простой CRUD |

## Плюсы и минусы

### Преимущества

1. **Чистый домен** — бизнес-логика без зависимостей от БД
2. **Тестируемость** — домен тестируется изолированно
3. **Гибкость** — сложные маппинги, разные схемы БД
4. **Независимость** — изменения схемы не затрагивают домен
5. **Переиспользование** — один домен, разные хранилища

### Недостатки

1. **Сложность** — больше кода, классов, абстракций
2. **Boilerplate** — много однотипного кода маппинга
3. **Производительность** — overhead на преобразования
4. **Кривая обучения** — сложнее понять для новичков

## Когда использовать?

### Рекомендуется:
- Сложная бизнес-логика (DDD)
- Разные схемы хранения
- Высокие требования к тестируемости
- Долгоживущие проекты
- Микросервисы с чистой архитектурой

### Не рекомендуется:
- Простые CRUD приложения
- Прототипы и MVP
- Маленькие проекты
- Когда Active Record достаточен

## Лучшие практики

1. **Доменные объекты — чистые** — никаких импортов ORM
2. **Маппер — отдельный класс** — не смешивайте с репозиторием
3. **Тестируйте отдельно** — домен и маппер тестируются независимо
4. **Используйте Identity Map** — для консистентности объектов
5. **Интегрируйте с Unit of Work** — для управления транзакциями

## Заключение

Data Mapper — это мощный паттерн для проектов со сложной бизнес-логикой. Он обеспечивает полное разделение между доменом и хранилищем данных, что упрощает тестирование и поддержку кода. SQLAlchemy поддерживает оба подхода: декларативный (похожий на Active Record) и императивный (чистый Data Mapper), позволяя выбрать оптимальный вариант для вашего проекта.
