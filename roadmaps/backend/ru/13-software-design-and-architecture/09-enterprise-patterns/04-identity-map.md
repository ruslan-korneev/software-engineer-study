# Identity Map

## Что такое Identity Map?

**Identity Map (Карта идентичности)** — это паттерн проектирования, который гарантирует, что каждый объект из базы данных загружается только один раз в рамках одной сессии работы. Он хранит карту соответствия между идентификаторами записей в БД и объектами в памяти.

> Паттерн описан Мартином Фаулером в "Patterns of Enterprise Application Architecture" и является фундаментальным для работы ORM.

## Основная идея

Identity Map предотвращает дублирование объектов в памяти:

```
Запрос 1: SELECT * FROM users WHERE id = 1
    │
    v
Identity Map: {User(1): <объект в памяти>}
    │
    v
Запрос 2: SELECT * FROM users WHERE id = 1
    │
    └── Возвращает тот же объект из кэша!
```

## Зачем нужен Identity Map?

### Проблемы без Identity Map:

```python
# Без Identity Map
user1 = db.query("SELECT * FROM users WHERE id = 1")
user2 = db.query("SELECT * FROM users WHERE id = 1")

# Два разных объекта в памяти!
print(user1 is user2)  # False

# Изменение одного не влияет на другой
user1.name = "Alice"
print(user2.name)  # Всё ещё "Bob"

# Проблема: какой объект сохранять?
```

### Решение с Identity Map:

```python
# С Identity Map
user1 = identity_map.get(User, 1)  # Загружает из БД
user2 = identity_map.get(User, 1)  # Возвращает тот же объект

# Один объект в памяти!
print(user1 is user2)  # True

# Изменения видны везде
user1.name = "Alice"
print(user2.name)  # "Alice"
```

### Основные преимущества:

1. **Консистентность** — один объект = одна запись в БД
2. **Производительность** — избегает повторных запросов
3. **Целостность** — нет конфликтов при изменениях
4. **Экономия памяти** — нет дублирования объектов

## Базовая реализация

### Простой Identity Map

```python
from typing import Dict, Type, TypeVar, Optional, Tuple

T = TypeVar("T")


class IdentityMap:
    """Простая реализация Identity Map."""

    def __init__(self):
        # Ключ: (тип сущности, идентификатор)
        # Значение: объект сущности
        self._map: Dict[Tuple[Type, int], object] = {}

    def get(self, entity_type: Type[T], id: int) -> Optional[T]:
        """Получить объект из карты."""
        key = (entity_type, id)
        return self._map.get(key)

    def add(self, entity: object) -> None:
        """Добавить объект в карту."""
        entity_type = type(entity)
        entity_id = getattr(entity, 'id', None)
        if entity_id is not None:
            key = (entity_type, entity_id)
            self._map[key] = entity

    def remove(self, entity: object) -> None:
        """Удалить объект из карты."""
        entity_type = type(entity)
        entity_id = getattr(entity, 'id', None)
        if entity_id is not None:
            key = (entity_type, entity_id)
            self._map.pop(key, None)

    def contains(self, entity_type: Type, id: int) -> bool:
        """Проверить наличие объекта в карте."""
        return (entity_type, id) in self._map

    def clear(self) -> None:
        """Очистить карту."""
        self._map.clear()

    def __len__(self) -> int:
        return len(self._map)


# Пример использования
class User:
    def __init__(self, id: int, name: str):
        self.id = id
        self.name = name


identity_map = IdentityMap()

# Первая загрузка
user = User(1, "Alice")
identity_map.add(user)

# Повторный доступ — тот же объект
cached_user = identity_map.get(User, 1)
print(user is cached_user)  # True
```

### Identity Map с типизацией

```python
from typing import Dict, Type, TypeVar, Optional, Generic
from dataclasses import dataclass

T = TypeVar("T")


class TypedIdentityMap(Generic[T]):
    """Identity Map для конкретного типа сущности."""

    def __init__(self, entity_type: Type[T]):
        self.entity_type = entity_type
        self._map: Dict[int, T] = {}

    def get(self, id: int) -> Optional[T]:
        return self._map.get(id)

    def add(self, entity: T) -> None:
        entity_id = getattr(entity, 'id', None)
        if entity_id is not None:
            self._map[entity_id] = entity

    def remove(self, id: int) -> None:
        self._map.pop(id, None)

    def all(self) -> list[T]:
        return list(self._map.values())

    def clear(self) -> None:
        self._map.clear()


# Использование
@dataclass
class Product:
    id: int
    name: str
    price: float


product_map = TypedIdentityMap(Product)
product_map.add(Product(1, "Laptop", 999.99))
product_map.add(Product(2, "Mouse", 29.99))

laptop = product_map.get(1)
print(laptop.name)  # "Laptop"
```

## Identity Map с репозиторием

```python
from abc import ABC, abstractmethod
from typing import Optional, List, Dict


class User:
    def __init__(self, id: int, username: str, email: str):
        self.id = id
        self.username = username
        self.email = email


class UserRepository:
    """Репозиторий с встроенным Identity Map."""

    def __init__(self, db_session):
        self.session = db_session
        self._identity_map: Dict[int, User] = {}

    def get(self, id: int) -> Optional[User]:
        """Получить пользователя по ID с кэшированием."""
        # Сначала проверяем Identity Map
        if id in self._identity_map:
            return self._identity_map[id]

        # Загружаем из БД
        row = self.session.execute(
            "SELECT id, username, email FROM users WHERE id = ?",
            (id,)
        ).fetchone()

        if row is None:
            return None

        # Создаём объект и добавляем в карту
        user = User(id=row[0], username=row[1], email=row[2])
        self._identity_map[id] = user
        return user

    def get_by_ids(self, ids: List[int]) -> List[User]:
        """Пакетная загрузка с использованием Identity Map."""
        result = []
        missing_ids = []

        # Сначала берём из кэша
        for id in ids:
            if id in self._identity_map:
                result.append(self._identity_map[id])
            else:
                missing_ids.append(id)

        # Загружаем недостающие из БД
        if missing_ids:
            placeholders = ','.join('?' * len(missing_ids))
            rows = self.session.execute(
                f"SELECT id, username, email FROM users WHERE id IN ({placeholders})",
                missing_ids
            ).fetchall()

            for row in rows:
                user = User(id=row[0], username=row[1], email=row[2])
                self._identity_map[user.id] = user
                result.append(user)

        return result

    def add(self, user: User) -> None:
        """Добавить пользователя."""
        self.session.execute(
            "INSERT INTO users (username, email) VALUES (?, ?)",
            (user.username, user.email)
        )
        user.id = self.session.lastrowid
        self._identity_map[user.id] = user

    def clear_cache(self) -> None:
        """Очистить Identity Map."""
        self._identity_map.clear()
```

## Identity Map в SQLAlchemy

SQLAlchemy Session уже реализует Identity Map:

```python
from sqlalchemy import Column, Integer, String, create_engine
from sqlalchemy.orm import declarative_base, Session, sessionmaker

Base = declarative_base()


class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    username = Column(String(50))
    email = Column(String(100))


# Настройка
engine = create_engine("sqlite:///example.db")
SessionFactory = sessionmaker(bind=engine)


def demonstrate_identity_map():
    session = SessionFactory()

    # Первый запрос — загрузка из БД
    user1 = session.query(User).get(1)
    print(f"Загружен: {user1}")

    # Второй запрос — из Identity Map (без SQL!)
    user2 = session.query(User).get(1)
    print(f"Из кэша: {user2}")

    # Это один и тот же объект
    print(f"user1 is user2: {user1 is user2}")  # True

    # Изменение одного меняет "оба"
    user1.username = "new_name"
    print(f"user2.username: {user2.username}")  # "new_name"

    session.close()


# Проверка через identity map напрямую
def check_identity_map():
    session = SessionFactory()

    user = session.query(User).get(1)

    # Проверяем, есть ли объект в identity map
    from sqlalchemy import inspect
    mapper = inspect(User)

    # Получаем identity key
    identity_key = mapper.identity_key_from_instance(user)
    print(f"Identity key: {identity_key}")

    # Проверяем в session
    cached = session.identity_map.get(identity_key)
    print(f"В кэше: {cached is user}")  # True

    session.close()
```

## Слабые ссылки в Identity Map

Для предотвращения утечек памяти используйте слабые ссылки:

```python
import weakref
from typing import Dict, Type, TypeVar, Optional

T = TypeVar("T")


class WeakIdentityMap:
    """Identity Map со слабыми ссылками."""

    def __init__(self):
        self._map: Dict[tuple, weakref.ref] = {}

    def get(self, entity_type: Type[T], id: int) -> Optional[T]:
        key = (entity_type, id)
        ref = self._map.get(key)

        if ref is None:
            return None

        # Получаем объект из слабой ссылки
        obj = ref()

        # Если объект был удалён сборщиком мусора
        if obj is None:
            del self._map[key]
            return None

        return obj

    def add(self, entity: object) -> None:
        entity_type = type(entity)
        entity_id = getattr(entity, 'id', None)

        if entity_id is not None:
            key = (entity_type, entity_id)
            # Создаём слабую ссылку с callback для очистки
            self._map[key] = weakref.ref(
                entity,
                lambda ref, k=key: self._cleanup(k)
            )

    def _cleanup(self, key: tuple) -> None:
        """Удаление записи при сборке мусора."""
        self._map.pop(key, None)

    def __len__(self) -> int:
        # Подсчёт только живых объектов
        return sum(1 for ref in self._map.values() if ref() is not None)


# Демонстрация
class Entity:
    def __init__(self, id: int, name: str):
        self.id = id
        self.name = name


identity_map = WeakIdentityMap()

# Добавляем объект
entity = Entity(1, "Test")
identity_map.add(entity)
print(f"В карте: {len(identity_map)}")  # 1

# Удаляем ссылку
del entity

# После сборки мусора объект удаляется из карты
import gc
gc.collect()
print(f"После GC: {len(identity_map)}")  # 0
```

## Identity Map с версионированием

```python
from dataclasses import dataclass, field
from typing import Dict, Optional, Tuple
from datetime import datetime


@dataclass
class VersionedEntity:
    """Сущность с версионированием."""
    id: int
    name: str
    version: int = 1
    updated_at: datetime = field(default_factory=datetime.now)


class VersionedIdentityMap:
    """Identity Map с отслеживанием версий."""

    def __init__(self):
        self._map: Dict[Tuple[type, int], VersionedEntity] = {}

    def get(self, entity_type: type, id: int) -> Optional[VersionedEntity]:
        key = (entity_type, id)
        return self._map.get(key)

    def add(self, entity: VersionedEntity) -> None:
        key = (type(entity), entity.id)

        # Проверяем конфликт версий
        existing = self._map.get(key)
        if existing and existing.version > entity.version:
            raise OptimisticLockError(
                f"Stale entity: expected version > {existing.version}, "
                f"got {entity.version}"
            )

        self._map[key] = entity

    def update(self, entity: VersionedEntity) -> None:
        """Обновление с инкрементом версии."""
        key = (type(entity), entity.id)
        existing = self._map.get(key)

        if existing and existing.version != entity.version:
            raise OptimisticLockError(
                f"Concurrent modification detected for {key}"
            )

        entity.version += 1
        entity.updated_at = datetime.now()
        self._map[key] = entity


class OptimisticLockError(Exception):
    """Ошибка оптимистичной блокировки."""
    pass
```

## Identity Map + Unit of Work

```python
from enum import Enum
from typing import Dict, Set, Type, Optional
from dataclasses import dataclass


class EntityState(Enum):
    NEW = "new"
    CLEAN = "clean"
    DIRTY = "dirty"
    DELETED = "deleted"


@dataclass
class TrackedEntity:
    entity: object
    state: EntityState
    original_hash: int


class IdentityMapWithTracking:
    """Identity Map с отслеживанием изменений для Unit of Work."""

    def __init__(self):
        self._map: Dict[tuple, TrackedEntity] = {}
        self._new: Set[object] = set()

    def register_new(self, entity: object) -> None:
        """Регистрация новой сущности."""
        self._new.add(entity)

    def register_clean(self, entity: object) -> None:
        """Регистрация загруженной сущности."""
        key = self._get_key(entity)
        self._map[key] = TrackedEntity(
            entity=entity,
            state=EntityState.CLEAN,
            original_hash=self._compute_hash(entity)
        )

    def get(self, entity_type: Type, id: int) -> Optional[object]:
        key = (entity_type, id)
        tracked = self._map.get(key)
        return tracked.entity if tracked else None

    def get_dirty_entities(self) -> list:
        """Получить изменённые сущности."""
        dirty = []
        for tracked in self._map.values():
            if tracked.state == EntityState.DIRTY:
                dirty.append(tracked.entity)
            elif tracked.state == EntityState.CLEAN:
                # Проверяем, изменился ли объект
                if self._compute_hash(tracked.entity) != tracked.original_hash:
                    tracked.state = EntityState.DIRTY
                    dirty.append(tracked.entity)
        return dirty

    def get_new_entities(self) -> list:
        """Получить новые сущности."""
        return list(self._new)

    def mark_deleted(self, entity: object) -> None:
        """Пометить сущность как удалённую."""
        key = self._get_key(entity)
        if key in self._map:
            self._map[key].state = EntityState.DELETED

    def get_deleted_entities(self) -> list:
        """Получить удалённые сущности."""
        return [
            tracked.entity
            for tracked in self._map.values()
            if tracked.state == EntityState.DELETED
        ]

    def clear(self) -> None:
        """Очистить после коммита."""
        self._map.clear()
        self._new.clear()

    def _get_key(self, entity: object) -> tuple:
        return (type(entity), getattr(entity, 'id', None))

    def _compute_hash(self, entity: object) -> int:
        """Вычисление хэша состояния объекта."""
        return hash(tuple(
            (k, v) for k, v in sorted(entity.__dict__.items())
            if not k.startswith('_')
        ))
```

## Сравнение подходов

| Подход | Гарантия уникальности | Кэширование | Память |
|--------|----------------------|-------------|--------|
| **Identity Map** | Да | В рамках сессии | Контролируемая |
| **Простой кэш** | Нет | Да | Может расти |
| **Без кэша** | Нет | Нет | Минимальная |

## Плюсы и минусы

### Преимущества

1. **Уникальность объектов** — нет дублирования
2. **Производительность** — меньше запросов к БД
3. **Консистентность** — единое состояние объекта
4. **Отслеживание изменений** — базис для Unit of Work

### Недостатки

1. **Память** — объекты хранятся в памяти
2. **Staleness** — данные могут устареть
3. **Сложность** — нужно правильно управлять жизненным циклом
4. **Область видимости** — обычно ограничена сессией

## Когда использовать?

### Рекомендуется:
- ORM и Data Mapper
- Unit of Work паттерн
- Сложные бизнес-транзакции
- Когда нужна консистентность объектов

### Не рекомендуется:
- Read-only операции с большими объёмами
- Stateless API без сессий
- Когда данные часто меняются извне

## Лучшие практики

1. **Ограничивайте scope** — один Identity Map на запрос/транзакцию
2. **Очищайте своевременно** — после commit/rollback
3. **Используйте слабые ссылки** — для предотвращения утечек памяти
4. **Интегрируйте с Unit of Work** — для отслеживания изменений
5. **Не используйте глобально** — это приведёт к проблемам

## Заключение

Identity Map — это фундаментальный паттерн для работы с персистентными объектами. Он гарантирует, что в рамках одной сессии каждая запись из базы данных представлена ровно одним объектом в памяти. SQLAlchemy и другие современные ORM реализуют этот паттерн автоматически через механизм сессий.
