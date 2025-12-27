# Unit Testing (Модульное тестирование API)

## Введение

**Модульное тестирование (Unit Testing)** - это метод тестирования программного обеспечения, при котором отдельные компоненты (юниты) кода тестируются изолированно от остальной системы. В контексте API это означает тестирование отдельных функций, методов, классов или эндпоинтов без реальных сетевых запросов и зависимостей.

## Основные концепции

### Что такое Unit в контексте API?

В API-разработке "юнит" может означать:
- **Отдельную функцию** обработки данных
- **Метод сервиса** с бизнес-логикой
- **Валидатор** входных данных
- **Сериализатор/десериализатор** данных
- **Обработчик эндпоинта** (с замоканными зависимостями)

### Принципы FIRST

Качественные unit-тесты следуют принципам FIRST:

- **Fast (Быстрые)** - выполняются за миллисекунды
- **Isolated (Изолированные)** - не зависят друг от друга и внешних систем
- **Repeatable (Повторяемые)** - дают одинаковый результат при каждом запуске
- **Self-validating (Самопроверяющиеся)** - автоматически определяют успех/провал
- **Timely (Своевременные)** - пишутся до или вместе с кодом

### Паттерн AAA (Arrange-Act-Assert)

Стандартная структура unit-теста:

```python
def test_calculate_discount():
    # Arrange (Подготовка)
    price = 100
    discount_percent = 20

    # Act (Действие)
    result = calculate_discount(price, discount_percent)

    # Assert (Проверка)
    assert result == 80
```

## Практические примеры на Python (pytest)

### Установка и настройка

```bash
pip install pytest pytest-cov pytest-asyncio
```

### Тестирование функций валидации

```python
# validators.py
from typing import Optional
import re

def validate_email(email: str) -> bool:
    """Валидация email адреса."""
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return bool(re.match(pattern, email))

def validate_password(password: str) -> tuple[bool, Optional[str]]:
    """Валидация пароля с возвратом ошибки."""
    if len(password) < 8:
        return False, "Password must be at least 8 characters"
    if not re.search(r'[A-Z]', password):
        return False, "Password must contain uppercase letter"
    if not re.search(r'[0-9]', password):
        return False, "Password must contain a digit"
    return True, None
```

```python
# test_validators.py
import pytest
from validators import validate_email, validate_password

class TestEmailValidation:
    """Тесты для валидации email."""

    @pytest.mark.parametrize("email,expected", [
        ("user@example.com", True),
        ("user.name@domain.org", True),
        ("user+tag@example.co.uk", True),
        ("invalid-email", False),
        ("@nodomain.com", False),
        ("user@.com", False),
        ("", False),
    ])
    def test_validate_email(self, email: str, expected: bool):
        """Параметризованный тест валидации email."""
        assert validate_email(email) == expected


class TestPasswordValidation:
    """Тесты для валидации пароля."""

    def test_valid_password(self):
        """Тест корректного пароля."""
        is_valid, error = validate_password("SecurePass123")
        assert is_valid is True
        assert error is None

    def test_short_password(self):
        """Тест слишком короткого пароля."""
        is_valid, error = validate_password("Short1")
        assert is_valid is False
        assert "at least 8 characters" in error

    def test_password_without_uppercase(self):
        """Тест пароля без заглавных букв."""
        is_valid, error = validate_password("lowercase123")
        assert is_valid is False
        assert "uppercase" in error

    def test_password_without_digit(self):
        """Тест пароля без цифр."""
        is_valid, error = validate_password("NoDigitsHere")
        assert is_valid is False
        assert "digit" in error
```

### Тестирование сервисов с моками

```python
# services.py
from dataclasses import dataclass
from typing import Protocol

@dataclass
class User:
    id: int
    email: str
    name: str

class UserRepository(Protocol):
    """Протокол репозитория пользователей."""
    def get_by_id(self, user_id: int) -> User | None: ...
    def get_by_email(self, email: str) -> User | None: ...
    def save(self, user: User) -> User: ...

class UserService:
    """Сервис для работы с пользователями."""

    def __init__(self, repository: UserRepository):
        self.repository = repository

    def get_user(self, user_id: int) -> User:
        """Получение пользователя по ID."""
        user = self.repository.get_by_id(user_id)
        if not user:
            raise ValueError(f"User with id {user_id} not found")
        return user

    def create_user(self, email: str, name: str) -> User:
        """Создание нового пользователя."""
        existing = self.repository.get_by_email(email)
        if existing:
            raise ValueError(f"User with email {email} already exists")

        user = User(id=0, email=email, name=name)
        return self.repository.save(user)
```

```python
# test_services.py
import pytest
from unittest.mock import Mock, MagicMock
from services import UserService, User

class TestUserService:
    """Тесты для UserService."""

    @pytest.fixture
    def mock_repository(self):
        """Фикстура для мок-репозитория."""
        return Mock()

    @pytest.fixture
    def user_service(self, mock_repository):
        """Фикстура для сервиса с замоканным репозиторием."""
        return UserService(repository=mock_repository)

    def test_get_user_success(self, user_service, mock_repository):
        """Тест успешного получения пользователя."""
        # Arrange
        expected_user = User(id=1, email="test@example.com", name="Test")
        mock_repository.get_by_id.return_value = expected_user

        # Act
        result = user_service.get_user(1)

        # Assert
        assert result == expected_user
        mock_repository.get_by_id.assert_called_once_with(1)

    def test_get_user_not_found(self, user_service, mock_repository):
        """Тест получения несуществующего пользователя."""
        # Arrange
        mock_repository.get_by_id.return_value = None

        # Act & Assert
        with pytest.raises(ValueError, match="not found"):
            user_service.get_user(999)

    def test_create_user_success(self, user_service, mock_repository):
        """Тест успешного создания пользователя."""
        # Arrange
        mock_repository.get_by_email.return_value = None
        mock_repository.save.return_value = User(
            id=1, email="new@example.com", name="New User"
        )

        # Act
        result = user_service.create_user("new@example.com", "New User")

        # Assert
        assert result.id == 1
        assert result.email == "new@example.com"
        mock_repository.save.assert_called_once()

    def test_create_user_duplicate_email(self, user_service, mock_repository):
        """Тест создания пользователя с существующим email."""
        # Arrange
        existing_user = User(id=1, email="exists@example.com", name="Existing")
        mock_repository.get_by_email.return_value = existing_user

        # Act & Assert
        with pytest.raises(ValueError, match="already exists"):
            user_service.create_user("exists@example.com", "New User")
```

### Тестирование FastAPI эндпоинтов

```python
# main.py
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel, EmailStr

app = FastAPI()

class UserCreate(BaseModel):
    email: EmailStr
    name: str

class UserResponse(BaseModel):
    id: int
    email: str
    name: str

# Простая in-memory база для примера
users_db: dict[int, dict] = {}
next_id = 1

@app.post("/users", response_model=UserResponse)
async def create_user(user: UserCreate):
    global next_id

    # Проверка на дубликат email
    for existing in users_db.values():
        if existing["email"] == user.email:
            raise HTTPException(status_code=400, detail="Email already registered")

    new_user = {"id": next_id, "email": user.email, "name": user.name}
    users_db[next_id] = new_user
    next_id += 1

    return new_user

@app.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: int):
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail="User not found")
    return users_db[user_id]
```

```python
# test_main.py
import pytest
from fastapi.testclient import TestClient
from main import app, users_db

@pytest.fixture(autouse=True)
def clear_db():
    """Очистка базы перед каждым тестом."""
    users_db.clear()
    yield
    users_db.clear()

@pytest.fixture
def client():
    """Тестовый клиент FastAPI."""
    return TestClient(app)

class TestCreateUser:
    """Тесты для создания пользователя."""

    def test_create_user_success(self, client):
        """Успешное создание пользователя."""
        response = client.post(
            "/users",
            json={"email": "test@example.com", "name": "Test User"}
        )

        assert response.status_code == 200
        data = response.json()
        assert data["email"] == "test@example.com"
        assert data["name"] == "Test User"
        assert "id" in data

    def test_create_user_invalid_email(self, client):
        """Создание пользователя с невалидным email."""
        response = client.post(
            "/users",
            json={"email": "invalid-email", "name": "Test User"}
        )

        assert response.status_code == 422  # Validation error

    def test_create_user_duplicate_email(self, client):
        """Создание пользователя с дублирующим email."""
        # Первый пользователь
        client.post("/users", json={"email": "test@example.com", "name": "First"})

        # Попытка создать с тем же email
        response = client.post(
            "/users",
            json={"email": "test@example.com", "name": "Second"}
        )

        assert response.status_code == 400
        assert "already registered" in response.json()["detail"]

class TestGetUser:
    """Тесты для получения пользователя."""

    def test_get_user_success(self, client):
        """Успешное получение пользователя."""
        # Создаем пользователя
        create_response = client.post(
            "/users",
            json={"email": "test@example.com", "name": "Test User"}
        )
        user_id = create_response.json()["id"]

        # Получаем пользователя
        response = client.get(f"/users/{user_id}")

        assert response.status_code == 200
        assert response.json()["email"] == "test@example.com"

    def test_get_user_not_found(self, client):
        """Получение несуществующего пользователя."""
        response = client.get("/users/999")

        assert response.status_code == 404
        assert "not found" in response.json()["detail"]
```

## Примеры на JavaScript (Jest)

### Установка

```bash
npm install --save-dev jest @types/jest supertest
```

### Тестирование функций

```javascript
// validators.js
function validateEmail(email) {
    const pattern = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/;
    return pattern.test(email);
}

function validatePassword(password) {
    const errors = [];

    if (password.length < 8) {
        errors.push('Password must be at least 8 characters');
    }
    if (!/[A-Z]/.test(password)) {
        errors.push('Password must contain uppercase letter');
    }
    if (!/[0-9]/.test(password)) {
        errors.push('Password must contain a digit');
    }

    return {
        isValid: errors.length === 0,
        errors
    };
}

module.exports = { validateEmail, validatePassword };
```

```javascript
// validators.test.js
const { validateEmail, validatePassword } = require('./validators');

describe('validateEmail', () => {
    test.each([
        ['user@example.com', true],
        ['user.name@domain.org', true],
        ['invalid-email', false],
        ['@nodomain.com', false],
        ['', false],
    ])('validates %s as %s', (email, expected) => {
        expect(validateEmail(email)).toBe(expected);
    });
});

describe('validatePassword', () => {
    test('accepts valid password', () => {
        const result = validatePassword('SecurePass123');
        expect(result.isValid).toBe(true);
        expect(result.errors).toHaveLength(0);
    });

    test('rejects short password', () => {
        const result = validatePassword('Short1');
        expect(result.isValid).toBe(false);
        expect(result.errors).toContain('Password must be at least 8 characters');
    });

    test('rejects password without uppercase', () => {
        const result = validatePassword('lowercase123');
        expect(result.isValid).toBe(false);
        expect(result.errors).toContain('Password must contain uppercase letter');
    });
});
```

### Тестирование Express эндпоинтов

```javascript
// app.js
const express = require('express');
const app = express();

app.use(express.json());

const users = new Map();
let nextId = 1;

app.post('/users', (req, res) => {
    const { email, name } = req.body;

    // Валидация
    if (!email || !name) {
        return res.status(400).json({ error: 'Email and name required' });
    }

    // Проверка дубликата
    for (const user of users.values()) {
        if (user.email === email) {
            return res.status(400).json({ error: 'Email already registered' });
        }
    }

    const user = { id: nextId++, email, name };
    users.set(user.id, user);

    res.status(201).json(user);
});

app.get('/users/:id', (req, res) => {
    const id = parseInt(req.params.id);
    const user = users.get(id);

    if (!user) {
        return res.status(404).json({ error: 'User not found' });
    }

    res.json(user);
});

module.exports = { app, users };
```

```javascript
// app.test.js
const request = require('supertest');
const { app, users } = require('./app');

describe('User API', () => {
    beforeEach(() => {
        users.clear();
    });

    describe('POST /users', () => {
        test('creates user successfully', async () => {
            const response = await request(app)
                .post('/users')
                .send({ email: 'test@example.com', name: 'Test User' });

            expect(response.status).toBe(201);
            expect(response.body.email).toBe('test@example.com');
            expect(response.body.name).toBe('Test User');
            expect(response.body.id).toBeDefined();
        });

        test('returns 400 for missing fields', async () => {
            const response = await request(app)
                .post('/users')
                .send({ email: 'test@example.com' });

            expect(response.status).toBe(400);
            expect(response.body.error).toContain('required');
        });

        test('returns 400 for duplicate email', async () => {
            await request(app)
                .post('/users')
                .send({ email: 'test@example.com', name: 'First' });

            const response = await request(app)
                .post('/users')
                .send({ email: 'test@example.com', name: 'Second' });

            expect(response.status).toBe(400);
            expect(response.body.error).toContain('already registered');
        });
    });

    describe('GET /users/:id', () => {
        test('returns user by id', async () => {
            const createResponse = await request(app)
                .post('/users')
                .send({ email: 'test@example.com', name: 'Test User' });

            const userId = createResponse.body.id;

            const response = await request(app).get(`/users/${userId}`);

            expect(response.status).toBe(200);
            expect(response.body.email).toBe('test@example.com');
        });

        test('returns 404 for non-existent user', async () => {
            const response = await request(app).get('/users/999');

            expect(response.status).toBe(404);
            expect(response.body.error).toContain('not found');
        });
    });
});
```

## Best Practices

### 1. Именование тестов

```python
# Плохо
def test_1():
    pass

# Хорошо - описывает что тестируется и ожидаемый результат
def test_create_user_with_valid_data_returns_user_with_id():
    pass

def test_create_user_with_duplicate_email_raises_value_error():
    pass
```

### 2. Один assert на тест (когда возможно)

```python
# Лучше иметь несколько фокусированных тестов
def test_user_has_correct_email():
    user = create_user("test@example.com", "Test")
    assert user.email == "test@example.com"

def test_user_has_correct_name():
    user = create_user("test@example.com", "Test")
    assert user.name == "Test"
```

### 3. Используйте фикстуры для повторяющегося кода

```python
@pytest.fixture
def sample_user():
    """Фикстура для тестового пользователя."""
    return User(id=1, email="test@example.com", name="Test User")

@pytest.fixture
def authenticated_client(client, sample_user):
    """Фикстура для аутентифицированного клиента."""
    client.force_login(sample_user)
    return client
```

### 4. Тестируйте граничные случаи

```python
class TestPagination:
    def test_first_page(self): ...
    def test_last_page(self): ...
    def test_empty_result(self): ...
    def test_negative_page_number(self): ...
    def test_page_size_zero(self): ...
    def test_page_size_exceeds_max(self): ...
```

### 5. Изолируйте тесты друг от друга

```python
@pytest.fixture(autouse=True)
def reset_state():
    """Сброс состояния перед каждым тестом."""
    global_cache.clear()
    yield
    global_cache.clear()
```

## Инструменты и библиотеки

### Python

| Инструмент | Описание |
|------------|----------|
| **pytest** | Самый популярный фреймворк для тестирования |
| **pytest-cov** | Измерение покрытия кода |
| **pytest-asyncio** | Тестирование асинхронного кода |
| **pytest-mock** | Интеграция с unittest.mock |
| **hypothesis** | Property-based тестирование |
| **factory_boy** | Фабрики для создания тестовых данных |

### JavaScript/TypeScript

| Инструмент | Описание |
|------------|----------|
| **Jest** | Полноценный фреймворк для тестирования |
| **Mocha** | Гибкий тестовый фреймворк |
| **Chai** | Библиотека assertions |
| **Sinon** | Моки, шпионы, стабы |
| **supertest** | Тестирование HTTP-запросов |

### Запуск тестов

```bash
# Python
pytest                          # Запуск всех тестов
pytest tests/test_api.py       # Конкретный файл
pytest -v                       # Verbose вывод
pytest --cov=src               # С покрытием
pytest -k "test_create"        # По имени

# JavaScript
npm test                        # Запуск тестов
npm test -- --coverage         # С покрытием
npm test -- --watch            # Watch mode
```

## Метрики и покрытие кода

```bash
# Генерация отчета о покрытии
pytest --cov=src --cov-report=html

# Минимальный порог покрытия
pytest --cov=src --cov-fail-under=80
```

## Заключение

Unit-тестирование API - это фундамент качественного кода. Оно позволяет:
- Быстро находить регрессии при изменениях
- Документировать ожидаемое поведение кода
- Уверенно проводить рефакторинг
- Ускорить разработку в долгосрочной перспективе

Начинайте с простых тестов и постепенно увеличивайте покрытие, фокусируясь на критичной бизнес-логике.
