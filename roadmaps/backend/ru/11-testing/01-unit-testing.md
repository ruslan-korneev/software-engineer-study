# Модульное (Unit) тестирование

[prev: 08-hashing-algorithms](../10-web-security/08-hashing-algorithms.md) | [next: 02-integration-testing](./02-integration-testing.md)

---

## Что такое Unit-тестирование?

**Модульное тестирование** (Unit Testing) — это метод тестирования программного обеспечения, при котором отдельные компоненты (модули) программы тестируются изолированно от остальной системы. Цель — убедиться, что каждый модуль работает корректно сам по себе.

### Ключевые характеристики unit-тестов:

- **Изолированность** — тест проверяет только один модуль, без зависимостей
- **Быстрота** — выполняются за миллисекунды
- **Детерминированность** — всегда дают одинаковый результат
- **Независимость** — тесты не зависят друг от друга

## Пирамида тестирования

```
         /\
        /  \
       / E2E\        <- Мало (дорогие, медленные)
      /------\
     /Integration\   <- Средне
    /--------------\
   /   Unit Tests   \ <- Много (дешёвые, быстрые)
  /------------------\
```

Unit-тесты составляют основу пирамиды — их должно быть больше всего.

---

## Паттерны тестирования

### Паттерн AAA (Arrange-Act-Assert)

Самый популярный паттерн структурирования тестов:

```python
def test_calculate_discount():
    # Arrange (Подготовка)
    price = 100
    discount_percent = 20
    calculator = PriceCalculator()

    # Act (Действие)
    result = calculator.apply_discount(price, discount_percent)

    # Assert (Проверка)
    assert result == 80
```

### Паттерн Given-When-Then

Альтернативный паттерн, популярный в BDD:

```python
def test_user_registration():
    # Given (Дано) - начальное состояние
    user_data = {"email": "test@example.com", "password": "secure123"}
    user_service = UserService()

    # When (Когда) - выполняемое действие
    user = user_service.register(user_data)

    # Then (Тогда) - ожидаемый результат
    assert user.is_active == True
    assert user.email == "test@example.com"
```

---

## Практика с pytest

### Установка pytest

```bash
pip install pytest pytest-cov
```

### Базовый пример теста

```python
# calculator.py
class Calculator:
    def add(self, a: int, b: int) -> int:
        return a + b

    def divide(self, a: int, b: int) -> float:
        if b == 0:
            raise ValueError("Деление на ноль невозможно")
        return a / b

# test_calculator.py
import pytest
from calculator import Calculator

class TestCalculator:
    def setup_method(self):
        """Выполняется перед каждым тестом"""
        self.calc = Calculator()

    def test_add_positive_numbers(self):
        # Arrange & Act
        result = self.calc.add(2, 3)
        # Assert
        assert result == 5

    def test_add_negative_numbers(self):
        result = self.calc.add(-1, -1)
        assert result == -2

    def test_divide_success(self):
        result = self.calc.divide(10, 2)
        assert result == 5.0

    def test_divide_by_zero_raises_error(self):
        with pytest.raises(ValueError) as exc_info:
            self.calc.divide(10, 0)
        assert "Деление на ноль" in str(exc_info.value)
```

### Параметризованные тесты

```python
import pytest

@pytest.mark.parametrize("a, b, expected", [
    (1, 2, 3),
    (0, 0, 0),
    (-1, 1, 0),
    (100, 200, 300),
])
def test_add_parametrized(a, b, expected):
    calc = Calculator()
    assert calc.add(a, b) == expected
```

---

## Fixtures в pytest

Fixtures — это механизм подготовки данных и ресурсов для тестов.

```python
import pytest
from database import DatabaseConnection
from user_repository import UserRepository

@pytest.fixture
def db_connection():
    """Создаёт соединение с тестовой БД"""
    connection = DatabaseConnection(":memory:")
    connection.setup_schema()
    yield connection  # Передаём в тест
    connection.close()  # Очистка после теста

@pytest.fixture
def user_repository(db_connection):
    """Создаёт репозиторий с подготовленным соединением"""
    return UserRepository(db_connection)

def test_create_user(user_repository):
    # Arrange
    user_data = {"name": "Иван", "email": "ivan@test.ru"}

    # Act
    user = user_repository.create(user_data)

    # Assert
    assert user.id is not None
    assert user.name == "Иван"
```

### Scope фикстур

```python
@pytest.fixture(scope="function")  # По умолчанию, для каждого теста
@pytest.fixture(scope="class")     # Для всех тестов в классе
@pytest.fixture(scope="module")    # Для всех тестов в файле
@pytest.fixture(scope="session")   # Для всей тестовой сессии
```

---

## Mocking (Подмена зависимостей)

Mock-объекты позволяют изолировать тестируемый код от внешних зависимостей.

```python
from unittest.mock import Mock, patch, MagicMock
import pytest

# email_service.py
class EmailService:
    def send(self, to: str, subject: str, body: str) -> bool:
        # Реальная отправка email
        pass

# user_service.py
class UserService:
    def __init__(self, email_service: EmailService):
        self.email_service = email_service

    def register_user(self, email: str) -> dict:
        user = {"email": email, "status": "active"}
        self.email_service.send(
            to=email,
            subject="Добро пожаловать!",
            body="Вы успешно зарегистрированы"
        )
        return user

# test_user_service.py
def test_register_user_sends_welcome_email():
    # Arrange
    mock_email_service = Mock(spec=EmailService)
    mock_email_service.send.return_value = True

    user_service = UserService(mock_email_service)

    # Act
    result = user_service.register_user("test@example.com")

    # Assert
    assert result["status"] == "active"
    mock_email_service.send.assert_called_once_with(
        to="test@example.com",
        subject="Добро пожаловать!",
        body="Вы успешно зарегистрированы"
    )
```

### Использование patch

```python
from unittest.mock import patch

def test_with_patch():
    with patch('requests.get') as mock_get:
        mock_get.return_value.status_code = 200
        mock_get.return_value.json.return_value = {"data": "test"}

        # Теперь requests.get будет возвращать мок
        result = fetch_data_from_api()

        assert result == {"data": "test"}
```

---

## Практика с unittest

Стандартная библиотека Python для тестирования.

```python
import unittest
from calculator import Calculator

class TestCalculator(unittest.TestCase):

    def setUp(self):
        """Подготовка перед каждым тестом"""
        self.calc = Calculator()

    def tearDown(self):
        """Очистка после каждого теста"""
        pass

    def test_add(self):
        self.assertEqual(self.calc.add(2, 3), 5)

    def test_divide_by_zero(self):
        with self.assertRaises(ValueError):
            self.calc.divide(10, 0)

    def test_divide_result_type(self):
        result = self.calc.divide(10, 3)
        self.assertIsInstance(result, float)
        self.assertAlmostEqual(result, 3.333, places=2)

if __name__ == '__main__':
    unittest.main()
```

### Методы assert в unittest

| Метод | Проверяет |
|-------|-----------|
| `assertEqual(a, b)` | a == b |
| `assertNotEqual(a, b)` | a != b |
| `assertTrue(x)` | bool(x) is True |
| `assertFalse(x)` | bool(x) is False |
| `assertIs(a, b)` | a is b |
| `assertIsNone(x)` | x is None |
| `assertIn(a, b)` | a in b |
| `assertRaises(exc)` | Исключение exc |
| `assertAlmostEqual(a, b)` | round(a-b, 7) == 0 |

---

## Лучшие практики

### 1. Называйте тесты понятно

```python
# Плохо
def test_1():
    pass

# Хорошо
def test_user_registration_with_invalid_email_raises_validation_error():
    pass
```

### 2. Один тест — одна проверка (концептуально)

```python
# Плохо - слишком много проверок разных аспектов
def test_user():
    user = create_user()
    assert user.is_valid()
    assert user.can_login()
    assert user.has_permissions()
    assert user.profile.is_complete()

# Хорошо - разделите на отдельные тесты
def test_new_user_is_valid():
    user = create_user()
    assert user.is_valid()

def test_new_user_can_login():
    user = create_user()
    assert user.can_login()
```

### 3. Тесты должны быть независимыми

```python
# Плохо - тесты зависят от порядка выполнения
class TestBad:
    user_id = None

    def test_create_user(self):
        self.user_id = create_user().id

    def test_get_user(self):
        user = get_user(self.user_id)  # Зависит от test_create_user!

# Хорошо - каждый тест создаёт свои данные
class TestGood:
    def test_create_user(self):
        user = create_user()
        assert user.id is not None

    def test_get_user(self):
        created_user = create_user()  # Своя подготовка
        user = get_user(created_user.id)
        assert user is not None
```

### 4. Используйте фабрики для создания тестовых данных

```python
# factories.py
from dataclasses import dataclass
from faker import Faker

fake = Faker('ru_RU')

@dataclass
class UserFactory:
    @staticmethod
    def create(**kwargs) -> dict:
        defaults = {
            "name": fake.name(),
            "email": fake.email(),
            "age": fake.random_int(18, 80)
        }
        return {**defaults, **kwargs}

# test_user.py
def test_user_creation():
    user_data = UserFactory.create(age=25)
    user = User(**user_data)
    assert user.age == 25
```

---

## Типичные ошибки

### 1. Тестирование реализации, а не поведения

```python
# Плохо - тест знает о внутренней реализации
def test_bad():
    calc = Calculator()
    calc._internal_cache = {}  # Доступ к приватным атрибутам!
    result = calc.add(2, 2)
    assert calc._internal_cache[(2, 2)] == 4

# Хорошо - тест проверяет только публичный API
def test_good():
    calc = Calculator()
    result = calc.add(2, 2)
    assert result == 4
```

### 2. Слишком много моков

```python
# Плохо - мокаем всё подряд
def test_over_mocked():
    mock_db = Mock()
    mock_cache = Mock()
    mock_logger = Mock()
    mock_validator = Mock()
    # ... тест ничего не проверяет

# Хорошо - мокаем только внешние зависимости
def test_properly_mocked():
    mock_email = Mock()  # Только email - внешний сервис
    service = UserService(email_service=mock_email)
    # Остальное работает реально
```

### 3. Игнорирование граничных случаев

```python
# Всегда тестируйте:
def test_empty_input():
    assert process([]) == []

def test_none_input():
    with pytest.raises(TypeError):
        process(None)

def test_large_input():
    large_list = list(range(10000))
    result = process(large_list)
    assert len(result) == 10000
```

---

## Запуск тестов и покрытие

```bash
# Запуск всех тестов
pytest

# Запуск с подробным выводом
pytest -v

# Запуск конкретного файла
pytest tests/test_calculator.py

# Запуск конкретного теста
pytest tests/test_calculator.py::test_add

# С измерением покрытия
pytest --cov=src --cov-report=html

# Остановка при первой ошибке
pytest -x

# Показать print() выводы
pytest -s
```

---

## Структура тестового проекта

```
project/
├── src/
│   ├── __init__.py
│   ├── calculator.py
│   └── user_service.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py          # Общие фикстуры
│   ├── test_calculator.py
│   ├── test_user_service.py
│   └── fixtures/
│       └── test_data.json
├── pytest.ini               # Конфигурация pytest
└── pyproject.toml
```

```ini
# pytest.ini
[pytest]
testpaths = tests
python_files = test_*.py
python_functions = test_*
addopts = -v --tb=short
```

---

## Полезные ресурсы

- [Официальная документация pytest](https://docs.pytest.org/)
- [Python Testing with pytest (книга)](https://pragprog.com/titles/bopytest2/python-testing-with-pytest-second-edition/)
- [unittest документация](https://docs.python.org/3/library/unittest.html)
- [Mock Object Library](https://docs.python.org/3/library/unittest.mock.html)

---

[prev: 08-hashing-algorithms](../10-web-security/08-hashing-algorithms.md) | [next: 02-integration-testing](./02-integration-testing.md)
