# Avoiding Nulls and Booleans

## Проблема с null

`null` (или `None` в Python, `nil` в Ruby) — одна из главных причин ошибок в программировании. Тони Хоар, создатель null-ссылки, назвал её "ошибкой на миллиард долларов".

### Почему null опасен?

```python
# TypeError: 'NoneType' object is not subscriptable
user = get_user_by_id(123)
print(user['name'])  # Упадёт, если user = None


# AttributeError: 'NoneType' object has no attribute 'email'
user = find_user(email="test@example.com")
send_email(user.email)  # Упадёт, если user не найден
```

### Проблемы null:

1. **Неявность** — сигнатура не показывает, что функция может вернуть null
2. **Вирусность** — null распространяется по коду
3. **Сложность отладки** — ошибка проявляется далеко от источника
4. **Множественные проверки** — код засоряется `if x is not None`

## Альтернативы null

### 1. Возврат пустых коллекций

```python
# Плохо — возвращает None
def get_user_orders(user_id: int) -> list | None:
    orders = db.query("SELECT * FROM orders WHERE user_id = ?", user_id)
    if not orders:
        return None
    return orders


# Вызывающий код вынужден проверять
orders = get_user_orders(123)
if orders is not None:  # Забыл проверить? NoneType has no len()
    for order in orders:
        process(order)


# Хорошо — возвращает пустой список
def get_user_orders(user_id: int) -> list:
    return db.query("SELECT * FROM orders WHERE user_id = ?", user_id)


# Вызывающий код простой
orders = get_user_orders(123)
for order in orders:  # Работает даже для пустого списка
    process(order)
```

### 2. Null Object Pattern

Создаём объект, представляющий "отсутствие значения":

```python
from abc import ABC, abstractmethod


class User(ABC):
    @abstractmethod
    def get_name(self) -> str:
        pass

    @abstractmethod
    def has_permission(self, permission: str) -> bool:
        pass


class RealUser(User):
    def __init__(self, name: str, permissions: list):
        self._name = name
        self._permissions = permissions

    def get_name(self) -> str:
        return self._name

    def has_permission(self, permission: str) -> bool:
        return permission in self._permissions


class NullUser(User):
    """Null Object — безопасное отсутствие пользователя."""

    def get_name(self) -> str:
        return "Guest"

    def has_permission(self, permission: str) -> bool:
        return False  # Гость не имеет прав


def get_user(user_id: int) -> User:
    user_data = db.find(user_id)
    if user_data:
        return RealUser(user_data['name'], user_data['permissions'])
    return NullUser()  # Вместо None


# Использование — никаких проверок на None!
user = get_user(123)
print(f"Welcome, {user.get_name()}")  # Работает всегда

if user.has_permission('admin'):
    show_admin_panel()
```

### 3. Optional/Maybe тип

```python
from typing import Optional


# Явно показываем, что значение может отсутствовать
def find_user(email: str) -> Optional[User]:
    """Возвращает пользователя или None."""
    return db.users.find_one(email=email)


# Вызывающий код видит Optional и обязан обработать
user: Optional[User] = find_user("test@example.com")

if user is not None:
    send_notification(user)
```

### 4. Result/Either тип

Возвращаем результат или ошибку:

```python
from dataclasses import dataclass
from typing import Generic, TypeVar, Union

T = TypeVar('T')
E = TypeVar('E')


@dataclass
class Success(Generic[T]):
    value: T


@dataclass
class Failure(Generic[E]):
    error: E


Result = Union[Success[T], Failure[E]]


def divide(a: float, b: float) -> Result[float, str]:
    if b == 0:
        return Failure("Division by zero")
    return Success(a / b)


# Использование
result = divide(10, 2)

match result:
    case Success(value):
        print(f"Result: {value}")
    case Failure(error):
        print(f"Error: {error}")


# Или проверка типа
if isinstance(result, Success):
    print(f"Result: {result.value}")
else:
    print(f"Error: {result.error}")
```

### 5. Исключения для исключительных случаев

```python
# Плохо — null для ошибки
def parse_int(value: str) -> int | None:
    try:
        return int(value)
    except ValueError:
        return None


# Хорошо — исключение для невалидного ввода
def parse_int(value: str) -> int:
    """
    Raises:
        ValueError: если строка не является числом
    """
    return int(value)


# Или кастомное исключение
class ParseError(Exception):
    pass


def parse_config(path: str) -> dict:
    try:
        with open(path) as f:
            return json.load(f)
    except FileNotFoundError:
        raise ParseError(f"Config file not found: {path}")
    except json.JSONDecodeError as e:
        raise ParseError(f"Invalid JSON in config: {e}")
```

### 6. Значения по умолчанию

```python
# Плохо
def get_setting(key: str) -> str | None:
    return settings.get(key)


# Хорошо — со значением по умолчанию
def get_setting(key: str, default: str = "") -> str:
    return settings.get(key, default)


# Использование
timeout = get_setting("timeout", "30")
```

## Проблема с boolean-параметрами

Boolean-параметры делают код менее читаемым и сигнализируют о нарушении Single Responsibility.

### Почему boolean-параметры плохи?

```python
# Что делает этот вызов?
process_order(order, True, False, True)

# Нужно смотреть сигнатуру, чтобы понять
def process_order(
    order,
    send_notification: bool,
    apply_discount: bool,
    use_express_shipping: bool
):
    pass
```

**Проблемы:**
1. Непонятно без контекста
2. Функция делает разные вещи в зависимости от флагов
3. Комбинаторный взрыв поведений

## Альтернативы boolean-параметрам

### 1. Разделение на отдельные функции

```python
# Плохо — флаг меняет поведение
def get_users(include_inactive: bool = False):
    if include_inactive:
        return db.users.all()
    return db.users.filter(active=True)


# Хорошо — две понятные функции
def get_active_users():
    return db.users.filter(active=True)


def get_all_users():
    return db.users.all()
```

### 2. Enum вместо boolean

```python
from enum import Enum, auto


# Плохо
def sort_users(users, ascending: bool = True):
    pass


# Хорошо — enum
class SortOrder(Enum):
    ASCENDING = auto()
    DESCENDING = auto()


def sort_users(users, order: SortOrder = SortOrder.ASCENDING):
    if order == SortOrder.ASCENDING:
        return sorted(users, key=lambda u: u.name)
    return sorted(users, key=lambda u: u.name, reverse=True)


# Вызов читается понятно
sort_users(users, SortOrder.DESCENDING)
```

### 3. Конфигурационные объекты

```python
from dataclasses import dataclass


# Плохо — много boolean параметров
def create_report(
    data,
    include_header: bool = True,
    include_footer: bool = True,
    use_colors: bool = False,
    landscape: bool = False,
    export_pdf: bool = False
):
    pass


# Хорошо — конфигурационный объект
@dataclass
class ReportConfig:
    include_header: bool = True
    include_footer: bool = True
    use_colors: bool = False
    orientation: str = "portrait"
    export_format: str = "html"


def create_report(data, config: ReportConfig = None):
    config = config or ReportConfig()
    # ...


# Использование — понятно и расширяемо
config = ReportConfig(
    use_colors=True,
    orientation="landscape",
    export_format="pdf"
)
create_report(data, config)
```

### 4. Стратегии и политики

```python
from abc import ABC, abstractmethod


# Плохо — boolean для стратегии
def calculate_shipping(order, use_express: bool = False):
    if use_express:
        return order.weight * 10
    return order.weight * 5


# Хорошо — стратегия
class ShippingStrategy(ABC):
    @abstractmethod
    def calculate(self, order) -> float:
        pass


class StandardShipping(ShippingStrategy):
    def calculate(self, order) -> float:
        return order.weight * 5


class ExpressShipping(ShippingStrategy):
    def calculate(self, order) -> float:
        return order.weight * 10


class FreeShipping(ShippingStrategy):
    def calculate(self, order) -> float:
        return 0


def calculate_shipping(order, strategy: ShippingStrategy) -> float:
    return strategy.calculate(order)


# Использование
cost = calculate_shipping(order, ExpressShipping())
```

### 5. Именованные аргументы

Если boolean неизбежен, используйте именованные аргументы:

```python
# Плохо
process_order(order, True, False)


# Лучше — именованные аргументы
process_order(
    order,
    send_notification=True,
    apply_discount=False
)
```

## Проблема с boolean-возвратами

### Потеря информации

```python
# Плохо — теряем информацию о причине неудачи
def validate_email(email: str) -> bool:
    if not email:
        return False
    if '@' not in email:
        return False
    if len(email) > 254:
        return False
    return True


# Какая проблема? Непонятно!
if not validate_email(user_input):
    print("Invalid email")  # Почему?
```

### Решение — возврат детальной информации

```python
from dataclasses import dataclass
from typing import List


@dataclass
class ValidationResult:
    is_valid: bool
    errors: List[str]


def validate_email(email: str) -> ValidationResult:
    errors = []

    if not email:
        errors.append("Email is required")
    elif '@' not in email:
        errors.append("Email must contain @")
    elif len(email) > 254:
        errors.append("Email is too long")

    return ValidationResult(
        is_valid=len(errors) == 0,
        errors=errors
    )


# Использование
result = validate_email(user_input)
if not result.is_valid:
    for error in result.errors:
        print(f"Error: {error}")
```

## Примеры рефакторинга

### До: null и boolean

```python
def find_and_process_user(
    user_id: int,
    send_email: bool = False,
    include_deleted: bool = False
) -> dict | None:
    if include_deleted:
        user = db.users.get(user_id)
    else:
        user = db.users.get_active(user_id)

    if user is None:
        return None

    result = {'id': user.id, 'name': user.name}

    if send_email:
        send_notification(user.email)

    return result
```

### После: без null и boolean

```python
from dataclasses import dataclass
from enum import Enum
from typing import List


class UserStatus(Enum):
    ACTIVE = "active"
    ALL = "all"


@dataclass
class ProcessingOptions:
    send_notification: bool = False


class UserNotFoundError(Exception):
    pass


@dataclass
class ProcessedUser:
    id: int
    name: str


def find_user(user_id: int, status: UserStatus = UserStatus.ACTIVE) -> User:
    """
    Raises:
        UserNotFoundError: если пользователь не найден
    """
    if status == UserStatus.ALL:
        user = db.users.get(user_id)
    else:
        user = db.users.get_active(user_id)

    if user is None:
        raise UserNotFoundError(f"User {user_id} not found")

    return user


def process_user(user: User, options: ProcessingOptions = None) -> ProcessedUser:
    options = options or ProcessingOptions()

    if options.send_notification:
        send_notification(user.email)

    return ProcessedUser(id=user.id, name=user.name)


# Использование
try:
    user = find_user(123, UserStatus.ACTIVE)
    result = process_user(user, ProcessingOptions(send_notification=True))
except UserNotFoundError as e:
    handle_not_found(e)
```

## Best Practices

### Для null:

1. **Возвращайте пустые коллекции** вместо null для списков
2. **Используйте Null Object** для объектов
3. **Используйте Optional** для явного обозначения
4. **Бросайте исключения** для исключительных случаев
5. **Предоставляйте значения по умолчанию**

### Для boolean:

1. **Разделяйте функции** с разным поведением
2. **Используйте enum** для ограниченных наборов опций
3. **Создавайте конфигурационные объекты** для множества параметров
4. **Применяйте паттерн Strategy** для сменных алгоритмов
5. **Используйте именованные аргументы** если boolean неизбежен

### Общие правила:

1. **Fail fast** — обнаруживайте проблемы как можно раньше
2. **Explicit is better than implicit** — явное лучше неявного
3. **Типизируйте код** — используйте аннотации типов

## Заключение

Избегание null и boolean-параметров делает код:

- **Безопаснее** — меньше NullPointerException
- **Читабельнее** — понятно без контекста
- **Гибче** — легче расширять
- **Тестируемее** — меньше комбинаций для проверки

Помните: каждый null в коде — потенциальный баг, каждый boolean-параметр — потенциальное нарушение SRP.
