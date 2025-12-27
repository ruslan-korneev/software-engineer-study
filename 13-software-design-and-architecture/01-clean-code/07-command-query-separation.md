# Command Query Separation

## Что такое CQS?

Command Query Separation (CQS) — это принцип проектирования, сформулированный Бертраном Мейером. Суть принципа:

> **Каждый метод должен быть либо командой, либо запросом, но не тем и другим одновременно.**

- **Команда (Command)**: изменяет состояние системы, ничего не возвращает (void)
- **Запрос (Query)**: возвращает данные, не изменяет состояние системы

## Почему это важно?

### 1. Предсказуемость

```python
# С CQS — понятно, что происходит
user = get_user(123)        # Запрос: получаем данные
update_user(user, data)     # Команда: изменяем состояние

# Без CQS — что произошло?
user = update_and_return_user(123, data)  # Изменил? Вернул новый? Старый?
```

### 2. Безопасность вызовов

```python
# Запросы безопасно вызывать много раз
count = get_user_count()
count = get_user_count()  # Ничего не изменилось

# Команды нужно вызывать осторожно
delete_user(123)
delete_user(123)  # Ошибка! Пользователь уже удалён
```

### 3. Кэширование

```python
# Запросы можно кэшировать
@lru_cache
def get_user(user_id: int) -> User:
    return db.users.find(user_id)

# Команды кэшировать нельзя
def update_user(user_id: int, data: dict) -> None:
    db.users.update(user_id, data)
```

## Команды (Commands)

Команды изменяют состояние и ничего не возвращают:

```python
class UserService:
    def create_user(self, name: str, email: str) -> None:
        """Команда: создаёт пользователя."""
        user = User(name=name, email=email)
        self.repository.save(user)

    def update_email(self, user_id: int, new_email: str) -> None:
        """Команда: обновляет email пользователя."""
        user = self.repository.find(user_id)
        user.email = new_email
        self.repository.save(user)

    def delete_user(self, user_id: int) -> None:
        """Команда: удаляет пользователя."""
        self.repository.delete(user_id)

    def activate_user(self, user_id: int) -> None:
        """Команда: активирует пользователя."""
        user = self.repository.find(user_id)
        user.is_active = True
        self.repository.save(user)
```

## Запросы (Queries)

Запросы возвращают данные и не изменяют состояние:

```python
class UserService:
    def get_user(self, user_id: int) -> User:
        """Запрос: возвращает пользователя по ID."""
        return self.repository.find(user_id)

    def get_all_users(self) -> List[User]:
        """Запрос: возвращает всех пользователей."""
        return self.repository.find_all()

    def get_active_users(self) -> List[User]:
        """Запрос: возвращает активных пользователей."""
        return self.repository.find_by(is_active=True)

    def count_users(self) -> int:
        """Запрос: возвращает количество пользователей."""
        return self.repository.count()

    def exists(self, user_id: int) -> bool:
        """Запрос: проверяет существование пользователя."""
        return self.repository.exists(user_id)
```

## Типичные нарушения CQS

### 1. Команда, возвращающая значение

```python
# Плохо — команда возвращает созданный объект
class UserService:
    def create_user(self, name: str, email: str) -> User:  # Нарушение!
        user = User(name=name, email=email)
        self.repository.save(user)
        return user


# Хорошо — разделение команды и запроса
class UserService:
    def create_user(self, name: str, email: str) -> None:
        """Команда: создаёт пользователя."""
        user_id = self._generate_id()
        user = User(id=user_id, name=name, email=email)
        self.repository.save(user)

    def get_user(self, user_id: int) -> User:
        """Запрос: возвращает пользователя."""
        return self.repository.find(user_id)


# Использование
service.create_user("John", "john@example.com")
user = service.get_user(expected_id)  # Или get_by_email
```

### 2. Pop операция

```python
# Плохо — изменяет и возвращает одновременно
class Stack:
    def pop(self) -> T:  # Нарушение CQS!
        if not self.items:
            raise EmptyStackError()
        return self.items.pop()


# Хорошо — разделение
class Stack:
    def top(self) -> T:
        """Запрос: возвращает верхний элемент."""
        if not self.items:
            raise EmptyStackError()
        return self.items[-1]

    def remove_top(self) -> None:
        """Команда: удаляет верхний элемент."""
        if not self.items:
            raise EmptyStackError()
        self.items.pop()


# Использование
element = stack.top()
stack.remove_top()
```

### 3. Счётчики и логирование

```python
# Плохо — запрос с побочным эффектом
class Logger:
    def get_last_entries(self, count: int) -> List[LogEntry]:
        self.access_count += 1  # Побочный эффект!
        return self.entries[-count:]


# Хорошо — разделение
class Logger:
    def get_last_entries(self, count: int) -> List[LogEntry]:
        """Запрос: возвращает последние записи."""
        return self.entries[-count:]

    def record_access(self) -> None:
        """Команда: записывает факт доступа."""
        self.access_count += 1
```

## Исключения из правила

Иногда нарушение CQS оправдано:

### 1. Атомарные операции

```python
# Допустимо — атомарная операция
def increment_and_get(counter_id: int) -> int:
    """Атомарно увеличивает счётчик и возвращает новое значение."""
    return db.execute(
        "UPDATE counters SET value = value + 1 RETURNING value"
    )

# Альтернатива с CQS может быть race condition:
def increment(counter_id: int) -> None:
    current = get_counter(counter_id)
    set_counter(counter_id, current + 1)  # Race condition!

current = get_counter(counter_id)  # Может вернуть устаревшее значение
```

### 2. Создание с возвратом ID

```python
# Допустимо — генерация ID на стороне БД
def create_user(name: str) -> int:
    """Создаёт пользователя и возвращает ID."""
    return db.execute(
        "INSERT INTO users (name) VALUES (?) RETURNING id",
        name
    )

# Альтернатива: генерировать ID заранее
import uuid

def create_user(name: str) -> None:
    user_id = uuid.uuid4()
    db.execute("INSERT INTO users (id, name) VALUES (?, ?)", user_id, name)
```

### 3. Fluent API

```python
# Допустимо — Builder pattern
class QueryBuilder:
    def where(self, condition: str) -> 'QueryBuilder':
        self.conditions.append(condition)
        return self

    def order_by(self, field: str) -> 'QueryBuilder':
        self.order = field
        return self


# Использование
results = (
    QueryBuilder()
    .where("status = 'active'")
    .order_by("created_at")
    .execute()
)
```

## Применение CQS

### В репозиториях

```python
from abc import ABC, abstractmethod
from typing import List, Optional


class UserRepository(ABC):
    # Команды
    @abstractmethod
    def save(self, user: User) -> None:
        """Сохраняет пользователя."""
        pass

    @abstractmethod
    def delete(self, user_id: int) -> None:
        """Удаляет пользователя."""
        pass

    # Запросы
    @abstractmethod
    def find(self, user_id: int) -> Optional[User]:
        """Возвращает пользователя по ID."""
        pass

    @abstractmethod
    def find_all(self) -> List[User]:
        """Возвращает всех пользователей."""
        pass

    @abstractmethod
    def find_by_email(self, email: str) -> Optional[User]:
        """Возвращает пользователя по email."""
        pass

    @abstractmethod
    def count(self) -> int:
        """Возвращает количество пользователей."""
        pass
```

### В сервисах

```python
class OrderService:
    def __init__(
        self,
        order_repo: OrderRepository,
        payment_service: PaymentService,
        notification_service: NotificationService
    ):
        self.order_repo = order_repo
        self.payment_service = payment_service
        self.notification_service = notification_service

    # Команды
    def place_order(self, order: Order) -> None:
        """Команда: размещает заказ."""
        self._validate_order(order)
        self.order_repo.save(order)
        self.notification_service.notify_order_placed(order)

    def cancel_order(self, order_id: int) -> None:
        """Команда: отменяет заказ."""
        order = self.order_repo.find(order_id)
        if order.status != OrderStatus.PENDING:
            raise InvalidOperationError("Cannot cancel processed order")

        order.status = OrderStatus.CANCELLED
        self.order_repo.save(order)
        self.notification_service.notify_order_cancelled(order)

    def process_payment(self, order_id: int) -> None:
        """Команда: обрабатывает платёж."""
        order = self.order_repo.find(order_id)
        self.payment_service.charge(order.customer_id, order.total)
        order.status = OrderStatus.PAID
        self.order_repo.save(order)

    # Запросы
    def get_order(self, order_id: int) -> Order:
        """Запрос: возвращает заказ."""
        return self.order_repo.find(order_id)

    def get_pending_orders(self) -> List[Order]:
        """Запрос: возвращает ожидающие заказы."""
        return self.order_repo.find_by_status(OrderStatus.PENDING)

    def get_order_total(self, order_id: int) -> Decimal:
        """Запрос: возвращает сумму заказа."""
        order = self.order_repo.find(order_id)
        return order.calculate_total()

    def can_cancel(self, order_id: int) -> bool:
        """Запрос: проверяет возможность отмены."""
        order = self.order_repo.find(order_id)
        return order.status == OrderStatus.PENDING
```

### В обработчиках событий

```python
from dataclasses import dataclass


# Команды
@dataclass
class CreateUserCommand:
    name: str
    email: str


@dataclass
class UpdateUserEmailCommand:
    user_id: int
    new_email: str


@dataclass
class DeleteUserCommand:
    user_id: int


# Запросы
@dataclass
class GetUserQuery:
    user_id: int


@dataclass
class GetUsersByStatusQuery:
    status: str


# Обработчики команд
class CreateUserHandler:
    def __init__(self, repository: UserRepository):
        self.repository = repository

    def handle(self, command: CreateUserCommand) -> None:
        user = User(name=command.name, email=command.email)
        self.repository.save(user)


# Обработчики запросов
class GetUserHandler:
    def __init__(self, repository: UserRepository):
        self.repository = repository

    def handle(self, query: GetUserQuery) -> User:
        return self.repository.find(query.user_id)
```

## CQRS — развитие CQS

CQRS (Command Query Responsibility Segregation) — архитектурный паттерн, который разделяет модели для чтения и записи:

```python
# Write Model — для команд
@dataclass
class UserWriteModel:
    id: int
    name: str
    email: str
    password_hash: str
    created_at: datetime
    updated_at: datetime

    def update_email(self, email: str) -> None:
        self.email = email
        self.updated_at = datetime.now()


# Read Model — для запросов (оптимизирована для чтения)
@dataclass
class UserReadModel:
    id: int
    name: str
    email: str
    orders_count: int
    total_spent: Decimal
    last_order_date: datetime


# Разные репозитории
class UserWriteRepository:
    def save(self, user: UserWriteModel) -> None:
        pass

    def find(self, user_id: int) -> UserWriteModel:
        pass


class UserReadRepository:
    def find(self, user_id: int) -> UserReadModel:
        # Может читать из другой БД, кэша или денормализованной таблицы
        pass

    def find_top_customers(self, limit: int) -> List[UserReadModel]:
        pass
```

## Best Practices

### 1. Именование

```python
# Команды — глаголы действия
def create_user() -> None: pass
def update_email() -> None: pass
def delete_order() -> None: pass
def activate_account() -> None: pass
def send_notification() -> None: pass

# Запросы — get, find, is, has, can
def get_user() -> User: pass
def find_orders() -> List[Order]: pass
def is_active() -> bool: pass
def has_permission() -> bool: pass
def can_edit() -> bool: pass
```

### 2. Документация

```python
class UserService:
    def update_profile(self, user_id: int, data: dict) -> None:
        """
        Обновляет профиль пользователя.

        Команда: изменяет состояние, ничего не возвращает.

        Args:
            user_id: ID пользователя
            data: Данные для обновления

        Raises:
            UserNotFoundError: Если пользователь не найден
        """
        pass

    def get_profile(self, user_id: int) -> UserProfile:
        """
        Возвращает профиль пользователя.

        Запрос: не изменяет состояние, безопасен для повторных вызовов.

        Args:
            user_id: ID пользователя

        Returns:
            Профиль пользователя

        Raises:
            UserNotFoundError: Если пользователь не найден
        """
        pass
```

### 3. Тестирование

```python
# Тесты для команд — проверяем побочные эффекты
def test_create_user_saves_to_database():
    service = UserService(mock_repository)

    service.create_user("John", "john@example.com")

    mock_repository.save.assert_called_once()
    saved_user = mock_repository.save.call_args[0][0]
    assert saved_user.name == "John"


# Тесты для запросов — проверяем возвращаемые данные
def test_get_user_returns_user():
    mock_repository.find.return_value = User(id=1, name="John")
    service = UserService(mock_repository)

    result = service.get_user(1)

    assert result.name == "John"
```

## Заключение

CQS — простой, но мощный принцип:

1. **Команды** изменяют состояние, ничего не возвращают
2. **Запросы** возвращают данные, не изменяют состояние
3. Исключения допустимы для атомарных операций и fluent API
4. CQRS — архитектурное развитие CQS для сложных систем

Следование CQS делает код:
- Предсказуемым
- Легко тестируемым
- Безопасным для кэширования и параллелизма
- Понятным без чтения реализации
