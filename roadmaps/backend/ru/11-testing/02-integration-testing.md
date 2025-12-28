# Интеграционное тестирование

[prev: 01-unit-testing](./01-unit-testing.md) | [next: 03-functional-testing](./03-functional-testing.md)

---

## Что такое интеграционное тестирование?

**Интеграционное тестирование** (Integration Testing) — это уровень тестирования, на котором проверяется взаимодействие между различными модулями или компонентами системы. В отличие от unit-тестов, интеграционные тесты проверяют, как модули работают вместе.

### Ключевые характеристики:

- **Проверка взаимодействия** — тестируется связь между компонентами
- **Реальные зависимости** — используются настоящие (или близкие к ним) сервисы
- **Медленнее unit-тестов** — требуют настройки окружения
- **Обнаружение ошибок интеграции** — находят проблемы на стыках модулей

---

## Отличия от Unit-тестирования

| Аспект | Unit-тесты | Интеграционные тесты |
|--------|------------|----------------------|
| Область | Один модуль/функция | Несколько модулей |
| Зависимости | Замокированы | Реальные или эмулированные |
| Скорость | Очень быстрые (мс) | Медленнее (сек) |
| Изоляция | Полная | Частичная |
| База данных | Mock | Тестовая БД |
| Сеть | Mock | Реальные запросы |

---

## Подходы к интеграционному тестированию

### Big Bang подход

Все модули интегрируются одновременно, затем тестируется вся система.

```
   +------+    +------+    +------+
   |  A   |--->|  B   |--->|  C   |
   +------+    +------+    +------+
        \         |         /
         \        v        /
          +------+------+
          |   Тестируем |
          |   всё сразу |
          +-------------+
```

**Плюсы:** Простота подготовки
**Минусы:** Сложно локализовать ошибки

### Инкрементальный подход

Модули интегрируются и тестируются постепенно.

#### Top-Down (Сверху вниз)

```python
# Сначала тестируем верхний уровень с заглушками
def test_api_layer():
    # Service layer - заглушка
    mock_service = Mock()
    api = APIController(service=mock_service)
    response = api.get_users()
    assert response.status_code == 200

# Затем добавляем реальный сервис
def test_api_with_service():
    mock_repo = Mock()  # Repository - ещё заглушка
    service = UserService(repository=mock_repo)
    api = APIController(service=service)
    # ...

# Наконец, полная интеграция
def test_full_integration():
    db = TestDatabase()
    repo = UserRepository(db)
    service = UserService(repository=repo)
    api = APIController(service=service)
    # ...
```

#### Bottom-Up (Снизу вверх)

```python
# Сначала тестируем нижний уровень (БД)
def test_repository_with_database():
    db = TestDatabase()
    repo = UserRepository(db)
    user = repo.create({"name": "Test"})
    assert user.id is not None

# Затем добавляем сервис
def test_service_with_repository():
    db = TestDatabase()
    repo = UserRepository(db)
    service = UserService(repository=repo)
    # ...
```

---

## Тестирование с базой данных

### Использование тестовой БД (pytest + SQLAlchemy)

```python
# conftest.py
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from models import Base

@pytest.fixture(scope="function")
def db_session():
    """Создаёт чистую БД для каждого теста"""
    # Используем SQLite in-memory для тестов
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)

    Session = sessionmaker(bind=engine)
    session = Session()

    yield session

    session.close()
    Base.metadata.drop_all(engine)

@pytest.fixture
def user_repository(db_session):
    return UserRepository(db_session)

# test_user_repository.py
def test_create_and_find_user(user_repository, db_session):
    # Arrange
    user_data = {"name": "Иван", "email": "ivan@test.ru"}

    # Act
    created_user = user_repository.create(user_data)
    found_user = user_repository.find_by_id(created_user.id)

    # Assert
    assert found_user is not None
    assert found_user.name == "Иван"
    assert found_user.email == "ivan@test.ru"

def test_update_user(user_repository):
    # Arrange
    user = user_repository.create({"name": "Иван", "email": "ivan@test.ru"})

    # Act
    user_repository.update(user.id, {"name": "Пётр"})
    updated_user = user_repository.find_by_id(user.id)

    # Assert
    assert updated_user.name == "Пётр"

def test_delete_user(user_repository):
    # Arrange
    user = user_repository.create({"name": "Иван", "email": "ivan@test.ru"})

    # Act
    user_repository.delete(user.id)

    # Assert
    assert user_repository.find_by_id(user.id) is None
```

### Использование PostgreSQL с Docker

```python
# conftest.py
import pytest
import docker
from sqlalchemy import create_engine

@pytest.fixture(scope="session")
def postgres_container():
    """Запускает PostgreSQL в Docker для тестов"""
    client = docker.from_env()

    container = client.containers.run(
        "postgres:15",
        environment={
            "POSTGRES_USER": "test",
            "POSTGRES_PASSWORD": "test",
            "POSTGRES_DB": "test_db"
        },
        ports={"5432/tcp": 5433},
        detach=True,
    )

    # Ждём готовности БД
    import time
    time.sleep(3)

    yield container

    container.stop()
    container.remove()

@pytest.fixture(scope="function")
def db_engine(postgres_container):
    engine = create_engine(
        "postgresql://test:test@localhost:5433/test_db"
    )
    Base.metadata.create_all(engine)

    yield engine

    Base.metadata.drop_all(engine)
```

### Использование pytest-docker-compose

```yaml
# docker-compose.test.yml
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: test_db
    ports:
      - "5433:5432"

  redis:
    image: redis:7
    ports:
      - "6380:6379"
```

```python
# conftest.py
import pytest

@pytest.fixture(scope="session")
def docker_compose_file():
    return "docker-compose.test.yml"

def test_with_real_services(docker_services):
    # docker_services автоматически поднимает контейнеры
    pass
```

---

## Тестирование API (FastAPI)

### Настройка тестового клиента

```python
# conftest.py
import pytest
from fastapi.testclient import TestClient
from httpx import AsyncClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from main import app
from database import get_db, Base

# Тестовая БД
SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"
engine = create_engine(SQLALCHEMY_DATABASE_URL)
TestingSessionLocal = sessionmaker(bind=engine)

def override_get_db():
    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()

@pytest.fixture(scope="function")
def client():
    # Создаём таблицы
    Base.metadata.create_all(bind=engine)

    # Подменяем зависимость БД
    app.dependency_overrides[get_db] = override_get_db

    with TestClient(app) as test_client:
        yield test_client

    # Очищаем
    Base.metadata.drop_all(bind=engine)
    app.dependency_overrides.clear()

# Для async тестов
@pytest.fixture
async def async_client():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac
```

### Тесты API endpoints

```python
# test_api.py
import pytest

class TestUserAPI:

    def test_create_user(self, client):
        # Arrange
        user_data = {
            "name": "Иван Петров",
            "email": "ivan@example.com",
            "password": "securepassword123"
        }

        # Act
        response = client.post("/api/users", json=user_data)

        # Assert
        assert response.status_code == 201
        data = response.json()
        assert data["name"] == "Иван Петров"
        assert data["email"] == "ivan@example.com"
        assert "id" in data
        assert "password" not in data  # Пароль не должен возвращаться

    def test_create_user_duplicate_email(self, client):
        # Arrange
        user_data = {"name": "User", "email": "test@test.com", "password": "pass"}
        client.post("/api/users", json=user_data)  # Первый пользователь

        # Act
        response = client.post("/api/users", json=user_data)

        # Assert
        assert response.status_code == 400
        assert "already exists" in response.json()["detail"].lower()

    def test_get_user(self, client):
        # Arrange - создаём пользователя
        create_response = client.post("/api/users", json={
            "name": "Test User",
            "email": "test@test.com",
            "password": "password"
        })
        user_id = create_response.json()["id"]

        # Act
        response = client.get(f"/api/users/{user_id}")

        # Assert
        assert response.status_code == 200
        assert response.json()["id"] == user_id

    def test_get_user_not_found(self, client):
        # Act
        response = client.get("/api/users/99999")

        # Assert
        assert response.status_code == 404

    def test_list_users_with_pagination(self, client):
        # Arrange - создаём несколько пользователей
        for i in range(15):
            client.post("/api/users", json={
                "name": f"User {i}",
                "email": f"user{i}@test.com",
                "password": "password"
            })

        # Act
        response = client.get("/api/users?page=1&per_page=10")

        # Assert
        assert response.status_code == 200
        data = response.json()
        assert len(data["items"]) == 10
        assert data["total"] == 15
        assert data["page"] == 1
```

### Тестирование с аутентификацией

```python
@pytest.fixture
def auth_headers(client):
    """Создаёт пользователя и возвращает заголовки с токеном"""
    # Регистрация
    client.post("/api/auth/register", json={
        "email": "auth@test.com",
        "password": "password123"
    })

    # Логин
    response = client.post("/api/auth/login", data={
        "username": "auth@test.com",
        "password": "password123"
    })
    token = response.json()["access_token"]

    return {"Authorization": f"Bearer {token}"}

def test_protected_endpoint(client, auth_headers):
    # Act
    response = client.get("/api/profile", headers=auth_headers)

    # Assert
    assert response.status_code == 200
    assert response.json()["email"] == "auth@test.com"

def test_protected_endpoint_without_auth(client):
    # Act
    response = client.get("/api/profile")

    # Assert
    assert response.status_code == 401
```

---

## Тестирование внешних сервисов

### Использование responses для HTTP-запросов

```python
import responses
import requests

@responses.activate
def test_external_api_call():
    # Arrange - мокаем внешний API
    responses.add(
        responses.GET,
        "https://api.external-service.com/data",
        json={"status": "success", "data": [1, 2, 3]},
        status=200
    )

    # Act
    result = fetch_external_data()

    # Assert
    assert result["status"] == "success"
    assert len(responses.calls) == 1

@responses.activate
def test_external_api_error():
    # Arrange - эмулируем ошибку
    responses.add(
        responses.GET,
        "https://api.external-service.com/data",
        json={"error": "Service unavailable"},
        status=503
    )

    # Act & Assert
    with pytest.raises(ExternalServiceError):
        fetch_external_data()
```

### Тестирование с Redis

```python
import pytest
import fakeredis

@pytest.fixture
def redis_client():
    """Использует fake Redis для тестов"""
    return fakeredis.FakeRedis(decode_responses=True)

@pytest.fixture
def cache_service(redis_client):
    return CacheService(redis_client)

def test_cache_set_and_get(cache_service):
    # Act
    cache_service.set("key", "value", ttl=60)
    result = cache_service.get("key")

    # Assert
    assert result == "value"

def test_cache_expiration(cache_service, redis_client):
    # Arrange
    cache_service.set("key", "value", ttl=1)

    # Act - эмулируем истечение времени
    redis_client.expire("key", 0)

    # Assert
    assert cache_service.get("key") is None
```

---

## Паттерн Given-When-Then в интеграционных тестах

```python
class TestOrderProcessing:
    """
    Тестирование полного цикла обработки заказа:
    API -> Service -> Repository -> Database
    """

    def test_complete_order_flow(self, client, db_session):
        # Given - пользователь и товары существуют
        user = create_user(db_session, {"name": "Покупатель"})
        product = create_product(db_session, {
            "name": "Ноутбук",
            "price": 50000,
            "stock": 10
        })

        # When - пользователь создаёт заказ
        response = client.post("/api/orders", json={
            "user_id": user.id,
            "items": [{"product_id": product.id, "quantity": 2}]
        })

        # Then - заказ создан успешно
        assert response.status_code == 201
        order_data = response.json()
        assert order_data["total_amount"] == 100000
        assert order_data["status"] == "pending"

        # And - остаток товара уменьшился
        db_session.refresh(product)
        assert product.stock == 8

    def test_order_insufficient_stock(self, client, db_session):
        # Given - товара мало на складе
        user = create_user(db_session, {"name": "Покупатель"})
        product = create_product(db_session, {
            "name": "Редкий товар",
            "price": 1000,
            "stock": 1
        })

        # When - пытаемся заказать больше, чем есть
        response = client.post("/api/orders", json={
            "user_id": user.id,
            "items": [{"product_id": product.id, "quantity": 5}]
        })

        # Then - получаем ошибку
        assert response.status_code == 400
        assert "insufficient stock" in response.json()["detail"].lower()

        # And - остаток не изменился
        db_session.refresh(product)
        assert product.stock == 1
```

---

## Транзакции и откат данных

```python
import pytest
from sqlalchemy.orm import Session

@pytest.fixture
def db_session(db_engine) -> Session:
    """
    Каждый тест выполняется в транзакции,
    которая откатывается после теста
    """
    connection = db_engine.connect()
    transaction = connection.begin()

    session = Session(bind=connection)

    yield session

    session.close()
    transaction.rollback()  # Откат всех изменений
    connection.close()

def test_with_rollback(db_session):
    # Любые изменения будут откачены
    user = User(name="Test")
    db_session.add(user)
    db_session.commit()

    # После теста БД будет чистой
```

---

## Лучшие практики

### 1. Изолируйте тестовое окружение

```python
# Используйте отдельную БД для тестов
TEST_DATABASE_URL = os.getenv(
    "TEST_DATABASE_URL",
    "postgresql://test:test@localhost:5433/test_db"
)

# Не используйте продакшн данные
assert "production" not in TEST_DATABASE_URL
```

### 2. Очищайте данные между тестами

```python
@pytest.fixture(autouse=True)
def cleanup(db_session):
    yield
    # Очистка после каждого теста
    db_session.query(Order).delete()
    db_session.query(User).delete()
    db_session.commit()
```

### 3. Используйте фабрики для тестовых данных

```python
# factories.py
class UserFactory:
    @staticmethod
    def create(db_session, **overrides) -> User:
        defaults = {
            "name": fake.name(),
            "email": fake.email(),
            "is_active": True
        }
        data = {**defaults, **overrides}
        user = User(**data)
        db_session.add(user)
        db_session.commit()
        return user
```

### 4. Тестируйте граничные случаи

```python
def test_concurrent_orders(client, db_session):
    """Тест на конкурентный доступ"""
    import threading

    product = create_product(db_session, {"stock": 1})

    results = []

    def make_order():
        response = client.post("/api/orders", json={...})
        results.append(response.status_code)

    # Параллельные запросы
    threads = [threading.Thread(target=make_order) for _ in range(5)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()

    # Только один заказ должен быть успешным
    assert results.count(201) == 1
    assert results.count(400) == 4
```

---

## Типичные ошибки

### 1. Зависимость от порядка тестов

```python
# Плохо - тесты зависят друг от друга
def test_create_order():
    global order_id
    order_id = create_order()

def test_get_order():
    order = get_order(order_id)  # Ошибка если test_create_order не запустился!

# Хорошо - каждый тест независим
def test_get_order(db_session):
    order = OrderFactory.create(db_session)
    result = get_order(order.id)
    assert result is not None
```

### 2. Использование продакшн сервисов

```python
# Плохо - тест отправляет реальные email!
def test_send_notification():
    send_email("user@real-domain.com", "Test")  # Опасно!

# Хорошо - используем тестовый сервис
def test_send_notification(mock_email_service):
    send_notification("test@test.com", "Test")
    mock_email_service.send.assert_called_once()
```

### 3. Игнорирование очистки ресурсов

```python
# Плохо - соединения утекают
def test_database():
    conn = create_connection()
    # Тест завершается без закрытия соединения

# Хорошо - используем контекстный менеджер
def test_database():
    with create_connection() as conn:
        # Соединение закроется автоматически
        pass
```

---

## Запуск интеграционных тестов

```bash
# Запуск только интеграционных тестов
pytest tests/integration/ -v

# С маркером
pytest -m integration

# С реальной БД
DATABASE_URL=postgresql://test:test@localhost/test pytest tests/integration/

# Параллельный запуск (требует pytest-xdist)
pytest tests/integration/ -n 4
```

```python
# pytest.ini
[pytest]
markers =
    integration: marks tests as integration tests
    slow: marks tests as slow
```

---

## Полезные ресурсы

- [Testing FastAPI](https://fastapi.tiangolo.com/tutorial/testing/)
- [pytest-docker](https://github.com/avast/pytest-docker)
- [Testcontainers Python](https://testcontainers-python.readthedocs.io/)
- [Factory Boy](https://factoryboy.readthedocs.io/)

---

[prev: 01-unit-testing](./01-unit-testing.md) | [next: 03-functional-testing](./03-functional-testing.md)
