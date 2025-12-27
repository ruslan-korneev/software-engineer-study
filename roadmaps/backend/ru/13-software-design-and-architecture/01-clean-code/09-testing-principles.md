# Testing Principles

## Зачем нужны тесты?

Тесты — неотъемлемая часть чистого кода. Они:

- **Предотвращают регрессии** — изменения не ломают существующий функционал
- **Документируют поведение** — тесты показывают, как использовать код
- **Улучшают дизайн** — тестируемый код = хорошо спроектированный код
- **Дают уверенность** — можно смело рефакторить
- **Ускоряют разработку** — быстрая обратная связь

## Пирамида тестирования

```
        /\
       /  \
      / E2E \        Медленные, дорогие
     /--------\
    /Integration\    Средние
   /--------------\
  /      Unit      \  Быстрые, дешёвые
 /------------------\
```

### Unit-тесты

Тестируют отдельные функции и классы в изоляции:

```python
def add(a: int, b: int) -> int:
    return a + b


def test_add():
    assert add(2, 3) == 5
    assert add(-1, 1) == 0
    assert add(0, 0) == 0
```

### Интеграционные тесты

Тестируют взаимодействие нескольких компонентов:

```python
def test_user_service_creates_user_in_database():
    db = TestDatabase()
    service = UserService(db)

    service.create_user("John", "john@example.com")

    user = db.users.find_by_email("john@example.com")
    assert user is not None
    assert user.name == "John"
```

### E2E тесты

Тестируют систему целиком:

```python
def test_user_can_register_and_login():
    # Регистрация
    response = client.post("/register", json={
        "name": "John",
        "email": "john@example.com",
        "password": "secret123"
    })
    assert response.status_code == 201

    # Логин
    response = client.post("/login", json={
        "email": "john@example.com",
        "password": "secret123"
    })
    assert response.status_code == 200
    assert "token" in response.json
```

## Принципы хороших тестов

### 1. FIRST

- **F**ast — быстрые (миллисекунды)
- **I**ndependent — независимые друг от друга
- **R**epeatable — повторяемые (одинаковый результат)
- **S**elf-validating — с автоматической проверкой (pass/fail)
- **T**imely — написаны вовремя (перед или вместе с кодом)

### 2. Arrange-Act-Assert (AAA)

```python
def test_user_can_change_email():
    # Arrange (подготовка)
    user = User(name="John", email="old@example.com")
    repository = InMemoryUserRepository()
    repository.save(user)
    service = UserService(repository)

    # Act (действие)
    service.change_email(user.id, "new@example.com")

    # Assert (проверка)
    updated_user = repository.find(user.id)
    assert updated_user.email == "new@example.com"
```

### 3. Given-When-Then (BDD стиль)

```python
def test_order_total_includes_tax():
    # Given (дано)
    order = Order()
    order.add_item(Item(price=100, quantity=2))  # 200
    tax_rate = Decimal("0.1")  # 10%

    # When (когда)
    total = order.calculate_total(tax_rate=tax_rate)

    # Then (тогда)
    assert total == Decimal("220")  # 200 + 20 tax
```

### 4. Один тест — одна концепция

```python
# Плохо — тестирует слишком много
def test_user():
    user = User("John", "john@example.com")

    # Тестирует создание
    assert user.name == "John"
    assert user.email == "john@example.com"

    # Тестирует изменение
    user.name = "Jane"
    assert user.name == "Jane"

    # Тестирует валидацию
    with pytest.raises(ValueError):
        user.email = "invalid"


# Хорошо — отдельные тесты
def test_user_creation():
    user = User("John", "john@example.com")
    assert user.name == "John"
    assert user.email == "john@example.com"


def test_user_name_can_be_changed():
    user = User("John", "john@example.com")
    user.name = "Jane"
    assert user.name == "Jane"


def test_user_email_validation():
    user = User("John", "john@example.com")
    with pytest.raises(ValueError):
        user.email = "invalid"
```

## Тестирование чистого кода

### Тестирование чистых функций

```python
# Чистые функции легко тестировать
def calculate_discount(price: Decimal, percent: int) -> Decimal:
    return price * percent / 100


@pytest.mark.parametrize("price,percent,expected", [
    (Decimal("100"), 10, Decimal("10")),
    (Decimal("50"), 20, Decimal("10")),
    (Decimal("0"), 50, Decimal("0")),
    (Decimal("100"), 0, Decimal("0")),
])
def test_calculate_discount(price, percent, expected):
    assert calculate_discount(price, percent) == expected
```

### Изоляция побочных эффектов

```python
# Бизнес-логика
def calculate_order_total(items: List[dict], tax_rate: Decimal) -> Decimal:
    """Чистая функция — легко тестировать."""
    subtotal = sum(
        Decimal(str(item['price'])) * item['quantity']
        for item in items
    )
    tax = subtotal * tax_rate
    return subtotal + tax


# Оркестратор с побочными эффектами
class OrderService:
    def __init__(self, repository, notifier):
        self.repository = repository
        self.notifier = notifier

    def process_order(self, order_id: int) -> None:
        order = self.repository.find(order_id)
        total = calculate_order_total(order.items, Decimal("0.1"))
        order.total = total
        self.repository.save(order)
        self.notifier.notify(order)


# Тест чистой функции — просто
def test_calculate_order_total():
    items = [
        {'price': 10, 'quantity': 2},
        {'price': 5, 'quantity': 3}
    ]
    assert calculate_order_total(items, Decimal("0.1")) == Decimal("38.5")


# Тест оркестратора — с моками
def test_process_order():
    mock_repo = Mock()
    mock_repo.find.return_value = Order(items=[{'price': 100, 'quantity': 1}])
    mock_notifier = Mock()

    service = OrderService(mock_repo, mock_notifier)
    service.process_order(1)

    mock_repo.save.assert_called_once()
    mock_notifier.notify.assert_called_once()
```

### Dependency Injection для тестируемости

```python
# Плохо — зависимость жёстко закодирована
class UserService:
    def __init__(self):
        self.db = PostgresDatabase()  # Нельзя подменить в тестах

    def get_user(self, user_id):
        return self.db.query(f"SELECT * FROM users WHERE id = {user_id}")


# Хорошо — зависимость внедряется
class UserService:
    def __init__(self, database: Database):
        self.db = database

    def get_user(self, user_id: int) -> User:
        return self.db.users.find(user_id)


# В тестах можно подменить
def test_get_user():
    mock_db = Mock()
    mock_db.users.find.return_value = User(id=1, name="John")

    service = UserService(mock_db)
    user = service.get_user(1)

    assert user.name == "John"
```

## Паттерны тестирования

### 1. Test Fixtures

```python
import pytest


@pytest.fixture
def user():
    """Создаёт тестового пользователя."""
    return User(name="Test User", email="test@example.com")


@pytest.fixture
def order(user):
    """Создаёт тестовый заказ."""
    order = Order(user_id=user.id)
    order.add_item(Item(name="Product", price=Decimal("10"), quantity=2))
    return order


def test_order_belongs_to_user(order, user):
    assert order.user_id == user.id


def test_order_has_items(order):
    assert len(order.items) == 1
```

### 2. Test Builders

```python
class UserBuilder:
    def __init__(self):
        self._name = "Default Name"
        self._email = "default@example.com"
        self._is_active = True

    def with_name(self, name: str) -> 'UserBuilder':
        self._name = name
        return self

    def with_email(self, email: str) -> 'UserBuilder':
        self._email = email
        return self

    def inactive(self) -> 'UserBuilder':
        self._is_active = False
        return self

    def build(self) -> User:
        return User(
            name=self._name,
            email=self._email,
            is_active=self._is_active
        )


def test_inactive_user_cannot_login():
    user = UserBuilder().inactive().build()
    assert not user.can_login()


def test_user_with_specific_email():
    user = UserBuilder().with_email("specific@test.com").build()
    assert user.email == "specific@test.com"
```

### 3. Object Mother

```python
class TestUsers:
    @staticmethod
    def john() -> User:
        return User(name="John Doe", email="john@example.com")

    @staticmethod
    def admin() -> User:
        return User(name="Admin", email="admin@example.com", role="admin")

    @staticmethod
    def inactive() -> User:
        user = User(name="Inactive", email="inactive@example.com")
        user.is_active = False
        return user


def test_admin_has_special_permissions():
    admin = TestUsers.admin()
    assert admin.has_permission("manage_users")
```

### 4. Моки и стабы

```python
from unittest.mock import Mock, patch


# Stub — возвращает предопределённые данные
def test_with_stub():
    stub_repo = Mock()
    stub_repo.find.return_value = User(id=1, name="John")

    service = UserService(stub_repo)
    user = service.get_user(1)

    assert user.name == "John"


# Mock — проверяет вызовы
def test_with_mock():
    mock_notifier = Mock()

    service = OrderService(notifier=mock_notifier)
    service.complete_order(order_id=1)

    mock_notifier.send.assert_called_once_with(
        "order_completed",
        order_id=1
    )


# Patch — подмена на уровне модуля
@patch('myapp.services.send_email')
def test_with_patch(mock_send_email):
    service = NotificationService()
    service.notify_user(user_id=1, message="Hello")

    mock_send_email.assert_called_once()
```

## Антипаттерны тестирования

### 1. Тестирование реализации, а не поведения

```python
# Плохо — тест завязан на реализацию
def test_user_service_calls_repository():
    mock_repo = Mock()
    service = UserService(mock_repo)

    service.create_user("John", "john@example.com")

    # Тестируем детали реализации
    mock_repo.save.assert_called_once()
    mock_repo.save.assert_called_with(
        User(name="John", email="john@example.com")
    )


# Хорошо — тест проверяет поведение
def test_created_user_can_be_found():
    repo = InMemoryUserRepository()
    service = UserService(repo)

    service.create_user("John", "john@example.com")

    user = service.get_user_by_email("john@example.com")
    assert user.name == "John"
```

### 2. Хрупкие тесты

```python
# Плохо — тест падает при любом изменении структуры
def test_user_to_dict():
    user = User(name="John", email="john@example.com")

    result = user.to_dict()

    assert result == {
        'id': None,
        'name': 'John',
        'email': 'john@example.com',
        'is_active': True,
        'created_at': None,
        'updated_at': None,
        'last_login': None,
        # ... 20 других полей
    }


# Хорошо — проверяем только важное
def test_user_to_dict_includes_name_and_email():
    user = User(name="John", email="john@example.com")

    result = user.to_dict()

    assert result['name'] == 'John'
    assert result['email'] == 'john@example.com'
```

### 3. Медленные тесты

```python
# Плохо — реальная база данных
def test_user_creation_slow():
    db = PostgresDatabase()  # Медленное подключение
    service = UserService(db)

    service.create_user("John", "john@example.com")  # Медленная запись

    user = db.query("SELECT * FROM users WHERE email = 'john@example.com'")
    assert user is not None


# Хорошо — in-memory хранилище
def test_user_creation_fast():
    db = InMemoryDatabase()  # Мгновенно
    service = UserService(db)

    service.create_user("John", "john@example.com")

    user = db.users.find_by_email("john@example.com")
    assert user is not None
```

### 4. Тесты с зависимостями

```python
# Плохо — тесты зависят от порядка выполнения
class TestUserService:
    user_id = None

    def test_create_user(self):
        user = create_user("John", "john@example.com")
        TestUserService.user_id = user.id  # Сохраняем для следующего теста

    def test_get_user(self):
        user = get_user(TestUserService.user_id)  # Зависит от предыдущего
        assert user.name == "John"


# Хорошо — независимые тесты
class TestUserService:
    def test_create_user(self, db):
        service = UserService(db)
        service.create_user("John", "john@example.com")

        user = db.users.find_by_email("john@example.com")
        assert user is not None

    def test_get_user(self, db):
        # Создаём свои данные
        user = User(id=1, name="John", email="john@example.com")
        db.users.save(user)

        service = UserService(db)
        result = service.get_user(1)

        assert result.name == "John"
```

## TDD (Test-Driven Development)

### Цикл Red-Green-Refactor

```python
# 1. RED — пишем падающий тест
def test_password_must_have_uppercase():
    validator = PasswordValidator()
    result = validator.validate("lowercase123")
    assert not result.is_valid
    assert "uppercase" in result.errors[0].lower()


# 2. GREEN — минимальный код для прохождения
class PasswordValidator:
    def validate(self, password: str) -> ValidationResult:
        if not any(c.isupper() for c in password):
            return ValidationResult(
                is_valid=False,
                errors=["Password must contain uppercase letter"]
            )
        return ValidationResult(is_valid=True, errors=[])


# 3. REFACTOR — улучшаем код
class PasswordValidator:
    def validate(self, password: str) -> ValidationResult:
        errors = []
        errors.extend(self._check_uppercase(password))
        # Добавляем другие проверки...
        return ValidationResult(
            is_valid=len(errors) == 0,
            errors=errors
        )

    def _check_uppercase(self, password: str) -> List[str]:
        if not any(c.isupper() for c in password):
            return ["Password must contain uppercase letter"]
        return []
```

## Best Practices

### 1. Называйте тесты понятно

```python
# Плохо
def test_1():
    pass

def test_user():
    pass

# Хорошо
def test_new_user_has_default_role():
    pass

def test_admin_can_delete_any_user():
    pass

def test_inactive_user_cannot_login():
    pass
```

### 2. Используйте параметризацию

```python
@pytest.mark.parametrize("email,is_valid", [
    ("valid@example.com", True),
    ("also.valid@test.org", True),
    ("invalid", False),
    ("@no-local.com", False),
    ("no-domain@", False),
    ("", False),
])
def test_email_validation(email, is_valid):
    result = validate_email(email)
    assert result == is_valid
```

### 3. Группируйте тесты логически

```python
class TestUserCreation:
    def test_user_gets_default_role(self):
        pass

    def test_user_email_must_be_unique(self):
        pass

    def test_user_password_is_hashed(self):
        pass


class TestUserAuthentication:
    def test_valid_credentials_return_token(self):
        pass

    def test_invalid_password_returns_error(self):
        pass

    def test_inactive_user_cannot_login(self):
        pass
```

### 4. Покрывайте граничные случаи

```python
def test_divide_normal_case():
    assert divide(10, 2) == 5

def test_divide_by_zero():
    with pytest.raises(ZeroDivisionError):
        divide(10, 0)

def test_divide_negative_numbers():
    assert divide(-10, 2) == -5

def test_divide_result_is_float():
    assert divide(5, 2) == 2.5

def test_divide_very_large_numbers():
    assert divide(10**100, 10**50) == 10**50
```

## Заключение

Хорошие тесты — основа качественного кода:

1. **Следуйте пирамиде** — больше unit, меньше E2E
2. **Используйте AAA/GWT** — структурируйте тесты
3. **Тестируйте поведение** — не реализацию
4. **Изолируйте зависимости** — используйте DI
5. **Пишите чистые тесты** — они тоже код!

Помните: **код без тестов — технический долг с первого дня**.
