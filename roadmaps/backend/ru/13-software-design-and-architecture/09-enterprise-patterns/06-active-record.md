# Active Record

## Что такое Active Record?

**Active Record** — это паттерн проектирования, в котором объект содержит как данные (соответствующие строке в таблице БД), так и поведение, включая логику для работы с базой данных (CRUD-операции). Объект "знает", как сохранить себя в БД.

> Паттерн описан Мартином Фаулером в "Patterns of Enterprise Application Architecture" и популяризирован фреймворком Ruby on Rails.

## Основная идея

В Active Record объект и запись в БД — это одно и то же:

```
       Active Record объект
┌─────────────────────────────────┐
│  Данные:                        │
│    - id = 1                     │
│    - name = "Alice"             │
│    - email = "alice@example.com"│
│                                 │
│  Методы персистентности:        │
│    - save()                     │
│    - delete()                   │
│    - find(id)                   │
└─────────────────────────────────┘
            │
            v
      Таблица users
┌────┬─────────┬───────────────────┐
│ id │  name   │      email        │
├────┼─────────┼───────────────────┤
│  1 │ Alice   │ alice@example.com │
└────┴─────────┴───────────────────┘
```

## Зачем нужен Active Record?

### Основные преимущества:

1. **Простота** — интуитивно понятный API
2. **Быстрая разработка** — минимум boilerplate кода
3. **Один объект** — данные и поведение вместе
4. **Discoverable** — легко понять, как работать с данными

## Базовая реализация

### Простой Active Record

```python
from abc import ABC
from typing import Optional, List, Dict, Any
import sqlite3


class ActiveRecord(ABC):
    """Базовый класс Active Record."""

    _table_name: str = ""
    _primary_key: str = "id"

    def __init__(self, **kwargs):
        for key, value in kwargs.items():
            setattr(self, key, value)

    @classmethod
    def get_connection(cls):
        """Получение соединения с БД."""
        # В реальном приложении используйте connection pool
        return sqlite3.connect("database.db")

    @classmethod
    def find(cls, id: int) -> Optional["ActiveRecord"]:
        """Найти запись по ID."""
        conn = cls.get_connection()
        cursor = conn.execute(
            f"SELECT * FROM {cls._table_name} WHERE {cls._primary_key} = ?",
            (id,)
        )
        row = cursor.fetchone()
        conn.close()

        if row is None:
            return None

        columns = [desc[0] for desc in cursor.description]
        data = dict(zip(columns, row))
        return cls(**data)

    @classmethod
    def find_all(cls) -> List["ActiveRecord"]:
        """Получить все записи."""
        conn = cls.get_connection()
        cursor = conn.execute(f"SELECT * FROM {cls._table_name}")
        rows = cursor.fetchall()
        columns = [desc[0] for desc in cursor.description]
        conn.close()

        return [cls(**dict(zip(columns, row))) for row in rows]

    @classmethod
    def where(cls, **conditions) -> List["ActiveRecord"]:
        """Поиск по условиям."""
        conn = cls.get_connection()
        where_clause = " AND ".join(f"{k} = ?" for k in conditions.keys())
        cursor = conn.execute(
            f"SELECT * FROM {cls._table_name} WHERE {where_clause}",
            tuple(conditions.values())
        )
        rows = cursor.fetchall()
        columns = [desc[0] for desc in cursor.description]
        conn.close()

        return [cls(**dict(zip(columns, row))) for row in rows]

    def save(self) -> None:
        """Сохранить запись (INSERT или UPDATE)."""
        if hasattr(self, self._primary_key) and getattr(self, self._primary_key):
            self._update()
        else:
            self._insert()

    def _insert(self) -> None:
        """Вставка новой записи."""
        data = self._get_data()
        data.pop(self._primary_key, None)  # Убираем ID для INSERT

        columns = ", ".join(data.keys())
        placeholders = ", ".join("?" * len(data))

        conn = self.get_connection()
        cursor = conn.execute(
            f"INSERT INTO {self._table_name} ({columns}) VALUES ({placeholders})",
            tuple(data.values())
        )
        setattr(self, self._primary_key, cursor.lastrowid)
        conn.commit()
        conn.close()

    def _update(self) -> None:
        """Обновление существующей записи."""
        data = self._get_data()
        id_value = data.pop(self._primary_key)

        set_clause = ", ".join(f"{k} = ?" for k in data.keys())

        conn = self.get_connection()
        conn.execute(
            f"UPDATE {self._table_name} SET {set_clause} WHERE {self._primary_key} = ?",
            (*data.values(), id_value)
        )
        conn.commit()
        conn.close()

    def delete(self) -> None:
        """Удаление записи."""
        conn = self.get_connection()
        conn.execute(
            f"DELETE FROM {self._table_name} WHERE {self._primary_key} = ?",
            (getattr(self, self._primary_key),)
        )
        conn.commit()
        conn.close()

    def _get_data(self) -> Dict[str, Any]:
        """Получение данных объекта для сохранения."""
        return {k: v for k, v in self.__dict__.items() if not k.startswith("_")}


# Конкретная модель
class User(ActiveRecord):
    _table_name = "users"

    def __init__(
        self,
        id: int = None,
        username: str = "",
        email: str = "",
        is_active: bool = True
    ):
        super().__init__(id=id, username=username, email=email, is_active=is_active)

    def deactivate(self):
        """Бизнес-метод."""
        self.is_active = False
        self.save()


# Использование
user = User(username="alice", email="alice@example.com")
user.save()  # INSERT

user.email = "newemail@example.com"
user.save()  # UPDATE

found_user = User.find(1)
all_users = User.find_all()
active_users = User.where(is_active=True)

user.delete()  # DELETE
```

## Active Record с SQLAlchemy

SQLAlchemy ORM в декларативном режиме похож на Active Record:

```python
from sqlalchemy import Column, Integer, String, Boolean, DateTime, create_engine, ForeignKey
from sqlalchemy.orm import declarative_base, sessionmaker, relationship
from datetime import datetime

Base = declarative_base()


class User(Base):
    """Active Record модель пользователя."""
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    username = Column(String(50), unique=True, nullable=False)
    email = Column(String(100), nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.now)

    # Связи
    posts = relationship("Post", back_populates="author")

    def __repr__(self):
        return f"<User(id={self.id}, username='{self.username}')>"

    # Бизнес-методы
    def deactivate(self):
        """Деактивация пользователя."""
        self.is_active = False

    def can_post(self) -> bool:
        """Проверка возможности постинга."""
        return self.is_active

    # Статические методы для поиска
    @classmethod
    def find_by_username(cls, session, username: str):
        return session.query(cls).filter_by(username=username).first()

    @classmethod
    def find_active(cls, session):
        return session.query(cls).filter_by(is_active=True).all()


class Post(Base):
    """Active Record модель поста."""
    __tablename__ = "posts"

    id = Column(Integer, primary_key=True)
    title = Column(String(200), nullable=False)
    content = Column(String, nullable=False)
    author_id = Column(Integer, ForeignKey("users.id"))
    published_at = Column(DateTime, nullable=True)

    author = relationship("User", back_populates="posts")

    def publish(self):
        """Публикация поста."""
        if not self.author.can_post():
            raise ValueError("Author cannot post")
        self.published_at = datetime.now()


# Настройка
engine = create_engine("sqlite:///blog.db")
Base.metadata.create_all(engine)
Session = sessionmaker(bind=engine)


# Использование
def example_usage():
    session = Session()

    # Создание
    user = User(username="alice", email="alice@example.com")
    session.add(user)
    session.commit()

    # Чтение
    alice = User.find_by_username(session, "alice")
    print(alice)

    # Обновление
    alice.email = "alice.new@example.com"
    session.commit()

    # Создание связанного объекта
    post = Post(title="Hello World", content="My first post", author=alice)
    session.add(post)
    post.publish()
    session.commit()

    # Навигация по связям
    for p in alice.posts:
        print(f"Post: {p.title}")

    # Удаление
    session.delete(post)
    session.commit()

    session.close()
```

## Active Record со связями

```python
from sqlalchemy import Column, Integer, String, ForeignKey, Table
from sqlalchemy.orm import relationship, declarative_base

Base = declarative_base()


# Many-to-Many связь
user_roles = Table(
    "user_roles",
    Base.metadata,
    Column("user_id", Integer, ForeignKey("users.id")),
    Column("role_id", Integer, ForeignKey("roles.id"))
)


class Role(Base):
    __tablename__ = "roles"

    id = Column(Integer, primary_key=True)
    name = Column(String(50), unique=True)

    users = relationship("User", secondary=user_roles, back_populates="roles")


class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    username = Column(String(50))

    # One-to-Many
    posts = relationship("Post", back_populates="author", cascade="all, delete-orphan")

    # One-to-One
    profile = relationship("Profile", back_populates="user", uselist=False)

    # Many-to-Many
    roles = relationship("Role", secondary=user_roles, back_populates="users")

    def add_role(self, role: Role):
        """Добавление роли пользователю."""
        if role not in self.roles:
            self.roles.append(role)

    def has_role(self, role_name: str) -> bool:
        """Проверка наличия роли."""
        return any(r.name == role_name for r in self.roles)


class Post(Base):
    __tablename__ = "posts"

    id = Column(Integer, primary_key=True)
    title = Column(String(200))
    author_id = Column(Integer, ForeignKey("users.id"))

    author = relationship("User", back_populates="posts")


class Profile(Base):
    __tablename__ = "profiles"

    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey("users.id"), unique=True)
    bio = Column(String(500))

    user = relationship("User", back_populates="profile")
```

## Валидация в Active Record

```python
from sqlalchemy import Column, Integer, String, event
from sqlalchemy.orm import declarative_base, validates
import re

Base = declarative_base()


class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    username = Column(String(50), nullable=False)
    email = Column(String(100), nullable=False)
    age = Column(Integer)

    @validates("email")
    def validate_email(self, key, value):
        """Валидация email при присваивании."""
        if not re.match(r"[^@]+@[^@]+\.[^@]+", value):
            raise ValueError(f"Invalid email: {value}")
        return value

    @validates("username")
    def validate_username(self, key, value):
        """Валидация username."""
        if len(value) < 3:
            raise ValueError("Username must be at least 3 characters")
        if not value.isalnum():
            raise ValueError("Username must be alphanumeric")
        return value.lower()

    @validates("age")
    def validate_age(self, key, value):
        """Валидация возраста."""
        if value is not None and (value < 0 or value > 150):
            raise ValueError(f"Invalid age: {value}")
        return value


# Валидация через события
@event.listens_for(User, "before_insert")
def before_insert_user(mapper, connection, target):
    """Проверка перед вставкой."""
    if not target.email:
        raise ValueError("Email is required")


@event.listens_for(User, "before_update")
def before_update_user(mapper, connection, target):
    """Проверка перед обновлением."""
    # Дополнительная логика валидации
    pass
```

## Callbacks (Hooks) в Active Record

```python
from sqlalchemy import Column, Integer, String, DateTime, event
from sqlalchemy.orm import declarative_base
from datetime import datetime
import hashlib

Base = declarative_base()


class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    username = Column(String(50))
    email = Column(String(100))
    password_hash = Column(String(255))
    _password = None  # Временное поле

    created_at = Column(DateTime)
    updated_at = Column(DateTime)

    @property
    def password(self):
        raise AttributeError("Password is write-only")

    @password.setter
    def password(self, value):
        self._password = value


# Before Insert
@event.listens_for(User, "before_insert")
def set_created_at(mapper, connection, target):
    target.created_at = datetime.now()
    target.updated_at = datetime.now()

    # Хеширование пароля
    if target._password:
        target.password_hash = hashlib.sha256(
            target._password.encode()
        ).hexdigest()


# Before Update
@event.listens_for(User, "before_update")
def set_updated_at(mapper, connection, target):
    target.updated_at = datetime.now()

    if target._password:
        target.password_hash = hashlib.sha256(
            target._password.encode()
        ).hexdigest()


# After Insert
@event.listens_for(User, "after_insert")
def after_insert_user(mapper, connection, target):
    print(f"User {target.username} created with ID {target.id}")
    # Отправка welcome email, логирование и т.д.


# After Delete
@event.listens_for(User, "after_delete")
def after_delete_user(mapper, connection, target):
    print(f"User {target.username} deleted")
    # Cleanup, аудит и т.д.
```

## Scopes (именованные запросы)

```python
from sqlalchemy import Column, Integer, String, Boolean, DateTime
from sqlalchemy.orm import declarative_base, Query
from datetime import datetime, timedelta

Base = declarative_base()


class UserQuery(Query):
    """Расширенный Query с именованными методами."""

    def active(self):
        return self.filter(User.is_active == True)

    def admins(self):
        return self.filter(User.is_admin == True)

    def recent(self, days=7):
        since = datetime.now() - timedelta(days=days)
        return self.filter(User.created_at >= since)

    def by_email_domain(self, domain):
        return self.filter(User.email.like(f"%@{domain}"))


class User(Base):
    __tablename__ = "users"
    query_class = UserQuery

    id = Column(Integer, primary_key=True)
    username = Column(String(50))
    email = Column(String(100))
    is_active = Column(Boolean, default=True)
    is_admin = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.now)

    # Альтернатива: методы класса
    @classmethod
    def active(cls, session):
        return session.query(cls).filter(cls.is_active == True)

    @classmethod
    def find_by_email(cls, session, email):
        return session.query(cls).filter(cls.email == email).first()


# Использование
# session.query(User).active().admins().recent(30).all()
# User.active(session).all()
```

## Сравнение с Data Mapper

| Аспект | Active Record | Data Mapper |
|--------|---------------|-------------|
| **Ответственность** | Объект знает о БД | Объект не знает о БД |
| **Простота** | Высокая | Ниже |
| **Тестируемость** | Требует БД | Изолированное |
| **Гибкость** | Ограниченная | Высокая |
| **Подходит для** | CRUD, простые модели | Сложный домен |

## Плюсы и минусы

### Преимущества

1. **Простота** — интуитивный API
2. **Быстрота разработки** — минимум кода
3. **Единство** — данные и поведение вместе
4. **Хорош для CRUD** — типичные операции просты

### Недостатки

1. **Нарушение SRP** — объект делает слишком много
2. **Сложное тестирование** — зависимость от БД
3. **Утечка абстракций** — домен знает о персистентности
4. **Ограниченная гибкость** — сложный маппинг затруднён

## Когда использовать?

### Рекомендуется:
- CRUD-приложения
- Простые бизнес-модели
- Быстрая разработка / MVP
- Админ-панели
- Прототипирование

### Не рекомендуется:
- Сложная бизнес-логика
- DDD-проекты
- Когда нужна высокая тестируемость
- Сложные маппинги данных

## Лучшие практики

1. **Держите модели тонкими** — выносите логику в сервисы
2. **Используйте валидацию** — проверяйте данные перед сохранением
3. **Применяйте callbacks умеренно** — не злоупотребляйте
4. **Тестируйте с тестовой БД** — используйте SQLite in-memory
5. **Разделяйте concerns** — для сложной логики используйте сервисы

## Заключение

Active Record — это простой и эффективный паттерн для типичных CRUD-операций. Он отлично подходит для быстрой разработки и проектов с простой бизнес-логикой. Однако для сложных доменов с богатой бизнес-логикой лучше рассмотреть паттерн Data Mapper, который обеспечивает лучшее разделение ответственности.
