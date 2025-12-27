# ORM (Object-Relational Mapping)

## Введение

**ORM (Object-Relational Mapping)** — это технология программирования, которая связывает реляционные базы данных с концепциями объектно-ориентированного программирования. ORM создает "виртуальную объектную базу данных", позволяя работать с данными как с обычными объектами языка программирования.

Простыми словами: ORM — это "переводчик" между миром объектов (классы, экземпляры) и миром реляционных баз данных (таблицы, строки, столбцы).

---

## Концепция ORM

### Основная идея

В реляционных базах данных информация хранится в таблицах:
- Таблица = набор записей одного типа
- Строка = одна запись (entity)
- Столбец = атрибут записи

В объектно-ориентированном программировании:
- Класс = описание типа данных
- Объект = экземпляр класса с данными
- Атрибут = свойство объекта

**ORM выполняет маппинг:**
```
Таблица    ←→  Класс (Model)
Строка     ←→  Объект (Instance)
Столбец    ←→  Атрибут класса (Field)
```

### Как это работает

```
┌─────────────────────────────────────────────────────────┐
│                    Приложение                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │           Python Objects / Classes               │   │
│  │   user = User(name="Ivan", email="i@mail.ru")   │   │
│  └─────────────────────────────────────────────────┘   │
│                         ↕                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │                    ORM                           │   │
│  │    • Преобразование объектов в SQL              │   │
│  │    • Преобразование результатов в объекты       │   │
│  │    • Управление соединениями                     │   │
│  │    • Отслеживание изменений                      │   │
│  └─────────────────────────────────────────────────┘   │
│                         ↕                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              SQL Queries                         │   │
│  │  INSERT INTO users (name, email) VALUES (...)   │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                          ↕
┌─────────────────────────────────────────────────────────┐
│                   База данных                           │
│         PostgreSQL / MySQL / SQLite / etc.              │
└─────────────────────────────────────────────────────────┘
```

### Паттерны ORM

#### 1. Active Record
Объект сам знает, как сохранить и загрузить себя из БД:

```python
# Пример паттерна Active Record
user = User(name="Ivan")
user.save()  # объект сам себя сохраняет
user.delete()  # объект сам себя удаляет
```

**Используется в:** Django ORM, Ruby on Rails ActiveRecord

#### 2. Data Mapper
Отдельный слой (mapper) отвечает за преобразование:

```python
# Пример паттерна Data Mapper
user = User(name="Ivan")
session.add(user)  # mapper сохраняет объект
session.commit()
```

**Используется в:** SQLAlchemy, Hibernate

---

## Популярные ORM библиотеки

### Python

#### SQLAlchemy
Самая мощная и гибкая ORM для Python. Поддерживает оба паттерна.

```python
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.orm import declarative_base, sessionmaker

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    name = Column(String(50))
    email = Column(String(100))
```

#### Django ORM
Встроенная ORM в Django фреймворк. Простая и удобная.

```python
from django.db import models

class User(models.Model):
    name = models.CharField(max_length=50)
    email = models.EmailField()

    class Meta:
        db_table = 'users'
```

#### Tortoise ORM
Асинхронная ORM, вдохновленная Django ORM.

```python
from tortoise import fields, models

class User(models.Model):
    id = fields.IntField(pk=True)
    name = fields.CharField(max_length=50)
    email = fields.CharField(max_length=100)
```

#### SQLModel
Современная ORM от создателя FastAPI, объединяет SQLAlchemy и Pydantic.

```python
from sqlmodel import SQLModel, Field

class User(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str
    email: str
```

### JavaScript/TypeScript

#### Prisma
Современная ORM с генерацией типов из схемы.

```prisma
// schema.prisma
model User {
  id    Int     @id @default(autoincrement())
  name  String
  email String  @unique
}
```

#### TypeORM
Популярная ORM для TypeScript с поддержкой декораторов.

```typescript
@Entity()
class User {
    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    name: string;
}
```

#### Sequelize
Классическая ORM для Node.js.

```javascript
const User = sequelize.define('User', {
    name: DataTypes.STRING,
    email: DataTypes.STRING
});
```

### Другие языки

| Язык | ORM |
|------|-----|
| Java | Hibernate, JPA, MyBatis |
| C# | Entity Framework, Dapper |
| Ruby | ActiveRecord (Rails), Sequel |
| Go | GORM, Ent |
| PHP | Doctrine, Eloquent (Laravel) |
| Rust | Diesel, SeaORM |

---

## Примеры кода с SQLAlchemy

### Настройка и подключение

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base

# Создание движка (connection pool)
engine = create_engine(
    "postgresql://user:password@localhost:5432/mydb",
    echo=True,  # Логирование SQL запросов
    pool_size=5,  # Размер пула соединений
    max_overflow=10  # Дополнительные соединения
)

# Базовый класс для моделей
Base = declarative_base()

# Фабрика сессий
SessionLocal = sessionmaker(
    autocommit=False,
    autoflush=False,
    bind=engine
)

# Получение сессии
def get_session():
    session = SessionLocal()
    try:
        yield session
    finally:
        session.close()
```

### Определение моделей

```python
from sqlalchemy import Column, Integer, String, ForeignKey, DateTime, Boolean, Text
from sqlalchemy.orm import relationship
from datetime import datetime

class User(Base):
    __tablename__ = 'users'

    # Первичный ключ
    id = Column(Integer, primary_key=True, index=True)

    # Обычные поля
    username = Column(String(50), unique=True, nullable=False)
    email = Column(String(100), unique=True, nullable=False)
    hashed_password = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)

    # Отношение один-ко-многим (у пользователя много постов)
    posts = relationship("Post", back_populates="author", lazy="selectin")

    def __repr__(self):
        return f"<User(id={self.id}, username='{self.username}')>"


class Post(Base):
    __tablename__ = 'posts'

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(200), nullable=False)
    content = Column(Text)
    published = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow)

    # Внешний ключ
    author_id = Column(Integer, ForeignKey('users.id'), nullable=False)

    # Обратное отношение
    author = relationship("User", back_populates="posts")

    # Отношение многие-ко-многим через промежуточную таблицу
    tags = relationship("Tag", secondary="post_tags", back_populates="posts")


class Tag(Base):
    __tablename__ = 'tags'

    id = Column(Integer, primary_key=True)
    name = Column(String(50), unique=True)

    posts = relationship("Post", secondary="post_tags", back_populates="tags")


# Промежуточная таблица для many-to-many
from sqlalchemy import Table

post_tags = Table(
    'post_tags',
    Base.metadata,
    Column('post_id', Integer, ForeignKey('posts.id'), primary_key=True),
    Column('tag_id', Integer, ForeignKey('tags.id'), primary_key=True)
)
```

### Создание таблиц

```python
# Создать все таблицы
Base.metadata.create_all(bind=engine)

# Удалить все таблицы (осторожно!)
Base.metadata.drop_all(bind=engine)
```

### CRUD операции

#### Create (Создание)

```python
from sqlalchemy.orm import Session

def create_user(session: Session, username: str, email: str, password: str) -> User:
    """Создание нового пользователя"""
    user = User(
        username=username,
        email=email,
        hashed_password=hash_password(password)  # функция хеширования
    )
    session.add(user)
    session.commit()
    session.refresh(user)  # Обновить объект данными из БД (включая id)
    return user


def create_post_with_tags(session: Session, user_id: int, title: str, content: str, tag_names: list[str]) -> Post:
    """Создание поста с тегами"""
    # Получить или создать теги
    tags = []
    for name in tag_names:
        tag = session.query(Tag).filter(Tag.name == name).first()
        if not tag:
            tag = Tag(name=name)
            session.add(tag)
        tags.append(tag)

    # Создать пост
    post = Post(
        title=title,
        content=content,
        author_id=user_id,
        tags=tags
    )
    session.add(post)
    session.commit()
    session.refresh(post)
    return post
```

#### Read (Чтение)

```python
def get_user_by_id(session: Session, user_id: int) -> User | None:
    """Получить пользователя по ID"""
    return session.query(User).filter(User.id == user_id).first()
    # Или современный синтаксис SQLAlchemy 2.0:
    # return session.get(User, user_id)


def get_user_by_email(session: Session, email: str) -> User | None:
    """Получить пользователя по email"""
    return session.query(User).filter(User.email == email).first()


def get_active_users(session: Session, skip: int = 0, limit: int = 100) -> list[User]:
    """Получить список активных пользователей с пагинацией"""
    return (
        session.query(User)
        .filter(User.is_active == True)
        .order_by(User.created_at.desc())
        .offset(skip)
        .limit(limit)
        .all()
    )


def get_posts_with_authors(session: Session) -> list[Post]:
    """Получить посты с загрузкой авторов (избегаем N+1)"""
    from sqlalchemy.orm import joinedload

    return (
        session.query(Post)
        .options(joinedload(Post.author))  # Eager loading
        .filter(Post.published == True)
        .all()
    )


def search_posts(session: Session, query: str) -> list[Post]:
    """Поиск постов по названию"""
    return (
        session.query(Post)
        .filter(Post.title.ilike(f"%{query}%"))  # Case-insensitive LIKE
        .all()
    )
```

#### Update (Обновление)

```python
def update_user(session: Session, user_id: int, **kwargs) -> User | None:
    """Обновить данные пользователя"""
    user = session.query(User).filter(User.id == user_id).first()
    if not user:
        return None

    for key, value in kwargs.items():
        if hasattr(user, key):
            setattr(user, key, value)

    session.commit()
    session.refresh(user)
    return user


def bulk_update_posts(session: Session, author_id: int, published: bool):
    """Массовое обновление постов"""
    session.query(Post).filter(Post.author_id == author_id).update(
        {"published": published},
        synchronize_session="fetch"
    )
    session.commit()
```

#### Delete (Удаление)

```python
def delete_user(session: Session, user_id: int) -> bool:
    """Удалить пользователя"""
    user = session.query(User).filter(User.id == user_id).first()
    if not user:
        return False

    session.delete(user)
    session.commit()
    return True


def soft_delete_user(session: Session, user_id: int) -> bool:
    """Мягкое удаление (деактивация)"""
    result = (
        session.query(User)
        .filter(User.id == user_id)
        .update({"is_active": False})
    )
    session.commit()
    return result > 0
```

### Сложные запросы

```python
from sqlalchemy import func, and_, or_, desc

def get_users_with_post_count(session: Session) -> list[tuple]:
    """Получить пользователей с количеством постов"""
    return (
        session.query(
            User.username,
            func.count(Post.id).label('post_count')
        )
        .outerjoin(Post)
        .group_by(User.id)
        .having(func.count(Post.id) > 0)
        .order_by(desc('post_count'))
        .all()
    )


def get_posts_by_date_range(
    session: Session,
    start_date: datetime,
    end_date: datetime
) -> list[Post]:
    """Получить посты за период"""
    return (
        session.query(Post)
        .filter(
            and_(
                Post.created_at >= start_date,
                Post.created_at <= end_date,
                Post.published == True
            )
        )
        .all()
    )


def get_popular_tags(session: Session, limit: int = 10) -> list[tuple]:
    """Получить популярные теги"""
    return (
        session.query(
            Tag.name,
            func.count(post_tags.c.post_id).label('usage_count')
        )
        .join(post_tags)
        .group_by(Tag.id)
        .order_by(desc('usage_count'))
        .limit(limit)
        .all()
    )
```

### Асинхронный SQLAlchemy

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

# Асинхронный движок
async_engine = create_async_engine(
    "postgresql+asyncpg://user:password@localhost/mydb",
    echo=True
)

# Асинхронная фабрика сессий
AsyncSessionLocal = sessionmaker(
    async_engine,
    class_=AsyncSession,
    expire_on_commit=False
)


async def get_async_session():
    async with AsyncSessionLocal() as session:
        yield session


async def get_user_async(session: AsyncSession, user_id: int) -> User | None:
    """Асинхронное получение пользователя"""
    from sqlalchemy import select

    result = await session.execute(
        select(User).where(User.id == user_id)
    )
    return result.scalar_one_or_none()


async def create_user_async(session: AsyncSession, username: str, email: str) -> User:
    """Асинхронное создание пользователя"""
    user = User(username=username, email=email, hashed_password="...")
    session.add(user)
    await session.commit()
    await session.refresh(user)
    return user
```

---

## Преимущества ORM

### 1. Абстракция от базы данных
```python
# Один и тот же код работает с разными СУБД
# Просто меняем connection string

engine_postgres = create_engine("postgresql://...")
engine_mysql = create_engine("mysql://...")
engine_sqlite = create_engine("sqlite:///./db.sqlite")
```

### 2. Безопасность (защита от SQL-инъекций)
```python
# Плохо - уязвимо для SQL-инъекции
cursor.execute(f"SELECT * FROM users WHERE name = '{user_input}'")

# Хорошо - ORM автоматически экранирует параметры
session.query(User).filter(User.name == user_input).all()
```

### 3. Продуктивность разработки
```python
# Вместо написания SQL вручную
"""
INSERT INTO users (username, email, created_at)
VALUES ('john', 'john@mail.com', NOW())
RETURNING id;
"""

# Простой Python-код
user = User(username='john', email='john@mail.com')
session.add(user)
session.commit()
```

### 4. Поддерживаемость кода
- Модели как единый источник истины
- Автодополнение и проверка типов в IDE
- Рефакторинг безопаснее

### 5. Отслеживание изменений
```python
user = session.query(User).first()
user.email = "new@mail.com"  # ORM отслеживает изменение
session.commit()  # Генерируется UPDATE только для измененных полей
```

### 6. Управление отношениями
```python
# ORM автоматически управляет JOIN и связями
user = session.query(User).first()
for post in user.posts:  # Загружаются связанные записи
    print(post.title)
```

---

## Недостатки ORM

### 1. Накладные расходы на производительность
```python
# ORM генерирует подобный SQL для простого запроса
session.query(User).filter(User.id == 1).first()

# SQL:
# SELECT users.id, users.username, users.email, users.hashed_password,
#        users.is_active, users.created_at
# FROM users
# WHERE users.id = 1
# LIMIT 1

# Если нужно только имя - все равно выбираются все колонки
```

### 2. Сложность оптимизации
```python
# Иногда ORM генерирует неоптимальные запросы
# Особенно для сложной бизнес-логики

# Нужно использовать explain для анализа
from sqlalchemy import text
result = session.execute(text("EXPLAIN ANALYZE SELECT ..."))
```

### 3. Проблема N+1 запросов
```python
# Плохо - N+1 проблема
users = session.query(User).all()
for user in users:  # 1 запрос
    print(user.posts)  # N дополнительных запросов

# Хорошо - eager loading
from sqlalchemy.orm import joinedload
users = session.query(User).options(joinedload(User.posts)).all()
# 1 запрос с JOIN
```

### 4. Утечка абстракции
```python
# Иногда нужно знать детали конкретной СУБД
# Например, специфичные для PostgreSQL фичи
from sqlalchemy.dialects.postgresql import JSONB, ARRAY

class Product(Base):
    metadata_ = Column(JSONB)  # PostgreSQL-специфичный тип
```

### 5. Кривая обучения
- Нужно понимать и SQL, и ORM
- У каждой ORM свой синтаксис и особенности
- Сложная документация

---

## Best Practices

### 1. Используйте сессии правильно

```python
# Плохо - одна глобальная сессия
global_session = SessionLocal()  # Проблемы с многопоточностью

# Хорошо - сессия на запрос
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Хорошо - контекстный менеджер
with SessionLocal() as session:
    user = session.query(User).first()
```

### 2. Всегда используйте индексы

```python
class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    email = Column(String, unique=True, index=True)  # Индекс!
    username = Column(String, index=True)  # Индекс!

    # Составной индекс
    __table_args__ = (
        Index('ix_user_email_active', 'email', 'is_active'),
    )
```

### 3. Избегайте N+1 с помощью eager loading

```python
from sqlalchemy.orm import selectinload, joinedload, subqueryload

# selectinload - отдельный SELECT IN запрос (хорошо для больших коллекций)
session.query(User).options(selectinload(User.posts)).all()

# joinedload - LEFT JOIN (хорошо для небольших коллекций)
session.query(Post).options(joinedload(Post.author)).all()

# subqueryload - подзапрос
session.query(User).options(subqueryload(User.posts)).all()
```

### 4. Используйте транзакции явно

```python
def transfer_money(session: Session, from_id: int, to_id: int, amount: float):
    try:
        # Начало транзакции (неявно при первом запросе)
        from_account = session.query(Account).filter(Account.id == from_id).with_for_update().first()
        to_account = session.query(Account).filter(Account.id == to_id).with_for_update().first()

        if from_account.balance < amount:
            raise ValueError("Insufficient funds")

        from_account.balance -= amount
        to_account.balance += amount

        session.commit()  # Коммит транзакции
    except Exception:
        session.rollback()  # Откат при ошибке
        raise
```

### 5. Используйте миграции

```bash
# Alembic - инструмент миграций для SQLAlchemy
pip install alembic
alembic init migrations

# Создание миграции
alembic revision --autogenerate -m "Add users table"

# Применение миграций
alembic upgrade head
```

### 6. Используйте Pydantic схемы для валидации

```python
from pydantic import BaseModel, EmailStr

class UserCreate(BaseModel):
    username: str
    email: EmailStr
    password: str

class UserResponse(BaseModel):
    id: int
    username: str
    email: str

    class Config:
        from_attributes = True  # Поддержка ORM-моделей

# Использование
def create_user(session: Session, user_data: UserCreate) -> UserResponse:
    user = User(**user_data.model_dump(exclude={'password'}))
    user.hashed_password = hash_password(user_data.password)
    session.add(user)
    session.commit()
    session.refresh(user)
    return UserResponse.model_validate(user)
```

### 7. Логируйте SQL-запросы в development

```python
import logging

logging.basicConfig()
logging.getLogger('sqlalchemy.engine').setLevel(logging.INFO)

# Или через параметр движка
engine = create_engine("...", echo=True)
```

---

## Типичные ошибки и как их избежать

### 1. Использование сессии после закрытия

```python
# Ошибка
def get_user():
    session = SessionLocal()
    user = session.query(User).first()
    session.close()
    return user

user = get_user()
print(user.posts)  # DetachedInstanceError!

# Решение - eager loading или expire_on_commit=False
SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)
```

### 2. Забытый commit

```python
# Ошибка - данные не сохранятся
user = User(username="john")
session.add(user)
# Забыли session.commit()

# Решение - всегда коммитить или использовать autocommit
session.add(user)
session.commit()
```

### 3. Не обновленные объекты

```python
# Ошибка - user.id будет None до refresh
user = User(username="john")
session.add(user)
session.commit()
print(user.id)  # Может быть None!

# Решение
session.refresh(user)
print(user.id)  # Теперь корректный ID
```

### 4. Lazy loading в async коде

```python
# Ошибка - lazy loading не работает в async
async def get_user_posts(session: AsyncSession, user_id: int):
    user = await session.get(User, user_id)
    return user.posts  # MissingGreenlet error!

# Решение - использовать eager loading
from sqlalchemy.orm import selectinload
from sqlalchemy import select

async def get_user_posts(session: AsyncSession, user_id: int):
    result = await session.execute(
        select(User)
        .options(selectinload(User.posts))
        .where(User.id == user_id)
    )
    user = result.scalar_one()
    return user.posts
```

### 5. Изменение первичного ключа

```python
# Ошибка - может сломать связи
user = session.query(User).first()
user.id = 999  # Очень плохая идея!

# Решение - никогда не меняйте PK
# Если нужно - создайте новую запись и удалите старую
```

### 6. Кэширование сессии

```python
# Ошибка - устаревшие данные
user = session.query(User).first()
# Кто-то другой изменил user в БД
print(user.email)  # Старые данные!

# Решение - expire или refresh
session.expire(user)
# или
session.refresh(user)
# или
session.expire_all()
```

---

## Когда использовать ORM vs чистый SQL

### Используйте ORM когда:

✅ **CRUD операции** - стандартные операции создания/чтения/обновления/удаления

✅ **Простые запросы** - фильтрация, сортировка, пагинация

✅ **Управление связями** - работа с отношениями между таблицами

✅ **Быстрая разработка** - прототипирование и MVP

✅ **Кроссплатформенность** - поддержка разных СУБД

✅ **Типобезопасность** - проверка типов и автодополнение

### Используйте чистый SQL когда:

❌ **Сложные аналитические запросы** - агрегации, оконные функции

```python
# Сложный аналитический запрос - лучше чистый SQL
from sqlalchemy import text

query = text("""
    WITH monthly_stats AS (
        SELECT
            DATE_TRUNC('month', created_at) as month,
            COUNT(*) as post_count,
            AVG(views) as avg_views
        FROM posts
        GROUP BY 1
    )
    SELECT
        month,
        post_count,
        avg_views,
        SUM(post_count) OVER (ORDER BY month) as running_total
    FROM monthly_stats
""")
result = session.execute(query)
```

❌ **Массовые операции** - bulk insert/update/delete

```python
# Массовая вставка - быстрее через чистый SQL или core
from sqlalchemy import insert

# Bulk insert через SQLAlchemy Core (быстрее ORM)
session.execute(
    insert(User),
    [
        {"username": "user1", "email": "user1@mail.com"},
        {"username": "user2", "email": "user2@mail.com"},
        # ... тысячи записей
    ]
)
```

❌ **Специфичные функции СУБД** - JSON операторы, полнотекстовый поиск

```python
# PostgreSQL полнотекстовый поиск
query = text("""
    SELECT * FROM posts
    WHERE to_tsvector('russian', content) @@ plainto_tsquery('russian', :query)
""")
result = session.execute(query, {"query": "поисковый запрос"})
```

❌ **Критичная производительность** - highload системы

❌ **Сложные отчеты** - отчеты с множеством JOIN и подзапросов

### Гибридный подход (рекомендуется)

```python
class PostRepository:
    def __init__(self, session: Session):
        self.session = session

    # Простые операции - ORM
    def get_by_id(self, post_id: int) -> Post | None:
        return self.session.query(Post).filter(Post.id == post_id).first()

    def create(self, title: str, content: str, author_id: int) -> Post:
        post = Post(title=title, content=content, author_id=author_id)
        self.session.add(post)
        self.session.commit()
        return post

    # Сложная аналитика - чистый SQL
    def get_monthly_statistics(self) -> list[dict]:
        query = text("""
            SELECT
                DATE_TRUNC('month', created_at) as month,
                COUNT(*) as count,
                AVG(views) as avg_views
            FROM posts
            WHERE published = true
            GROUP BY 1
            ORDER BY 1 DESC
        """)
        result = self.session.execute(query)
        return [dict(row) for row in result]
```

---

## Резюме

**ORM (Object-Relational Mapping)** — мощный инструмент, который:

1. **Что это**: Технология маппинга между объектами Python и таблицами БД

2. **Главные преимущества**:
   - Абстракция от конкретной СУБД
   - Защита от SQL-инъекций
   - Ускорение разработки
   - Лучшая поддерживаемость кода

3. **Главные недостатки**:
   - Накладные расходы на производительность
   - Проблема N+1
   - Сложность оптимизации
   - Утечка абстракции

4. **Популярные ORM для Python**:
   - **SQLAlchemy** — самая мощная и гибкая
   - **Django ORM** — простая и встроенная в Django
   - **SQLModel** — современная, совместимая с FastAPI
   - **Tortoise ORM** — асинхронная

5. **Ключевые практики**:
   - Используйте eager loading для избежания N+1
   - Правильно управляйте сессиями
   - Используйте миграции (Alembic)
   - Логируйте SQL в development
   - Комбинируйте ORM и raw SQL по необходимости

6. **Когда использовать**:
   - ORM — для CRUD и простых запросов
   - Raw SQL — для сложной аналитики и высокой нагрузки
   - Гибридный подход — оптимальное решение

ORM не заменяет знание SQL, а дополняет его. Хороший разработчик должен уметь работать и с ORM, и с чистым SQL, выбирая подходящий инструмент для каждой задачи.
