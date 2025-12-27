# Functional Testing (Функциональное тестирование)

## Введение

**Функциональное тестирование** - это тип тестирования, при котором проверяется соответствие системы функциональным требованиям. В контексте API это означает проверку того, что API выполняет именно те функции, которые от него ожидаются, с точки зрения конечного пользователя.

## Отличие от других видов тестирования

| Тип тестирования | Фокус | Что проверяет |
|------------------|-------|---------------|
| **Unit Testing** | Код | Отдельные функции/методы |
| **Integration Testing** | Компоненты | Взаимодействие между модулями |
| **Functional Testing** | Требования | Бизнес-логика, пользовательские сценарии |
| **Load Testing** | Производительность | Поведение под нагрузкой |

## Основные концепции

### Что тестируется в функциональном тестировании API

1. **Корректность ответов** - правильные данные в ответе
2. **HTTP-статусы** - соответствующие коды ответов
3. **Бизнес-логика** - правила и ограничения домена
4. **Авторизация и аутентификация** - права доступа
5. **Валидация данных** - проверка входных параметров
6. **Форматы данных** - JSON Schema, структура ответа
7. **Пользовательские сценарии** - последовательности операций

### Подходы к функциональному тестированию

#### Black-box Testing

Тестирование без знания внутренней реализации:

```python
# Тестируем только входы и выходы API
def test_create_order():
    response = client.post("/orders", json={
        "items": [{"product_id": 1, "quantity": 2}]
    })
    assert response.status_code == 201
    assert "order_id" in response.json()
```

#### Specification-based Testing

Тестирование на основе спецификации (OpenAPI, Swagger):

```python
# Тест соответствия спецификации
def test_response_matches_schema():
    response = client.get("/users/1")
    validate(response.json(), user_schema)
```

## Практические примеры на Python

### Настройка

```bash
pip install pytest requests jsonschema pydantic
```

### Базовая структура тестов

```python
# conftest.py
import pytest
import requests

class APIClient:
    """Клиент для функционального тестирования API."""

    def __init__(self, base_url: str, token: str = None):
        self.base_url = base_url
        self.session = requests.Session()
        if token:
            self.session.headers["Authorization"] = f"Bearer {token}"

    def get(self, path: str, **kwargs):
        return self.session.get(f"{self.base_url}{path}", **kwargs)

    def post(self, path: str, **kwargs):
        return self.session.post(f"{self.base_url}{path}", **kwargs)

    def put(self, path: str, **kwargs):
        return self.session.put(f"{self.base_url}{path}", **kwargs)

    def patch(self, path: str, **kwargs):
        return self.session.patch(f"{self.base_url}{path}", **kwargs)

    def delete(self, path: str, **kwargs):
        return self.session.delete(f"{self.base_url}{path}", **kwargs)

@pytest.fixture(scope="session")
def api_client():
    """Клиент API для тестов."""
    return APIClient(base_url="http://localhost:8000/api")

@pytest.fixture
def auth_client(api_client):
    """Аутентифицированный клиент."""
    # Получаем токен
    response = api_client.post("/auth/login", json={
        "email": "test@example.com",
        "password": "password123"
    })
    token = response.json()["access_token"]
    return APIClient(base_url="http://localhost:8000/api", token=token)
```

### Тестирование CRUD операций

```python
# test_users_functional.py
import pytest

class TestUsersCRUD:
    """Функциональные тесты CRUD операций для пользователей."""

    @pytest.fixture
    def created_user(self, auth_client):
        """Создание тестового пользователя."""
        response = auth_client.post("/users", json={
            "email": "newuser@example.com",
            "name": "New User",
            "role": "user"
        })
        assert response.status_code == 201
        user = response.json()
        yield user
        # Cleanup
        auth_client.delete(f"/users/{user['id']}")

    def test_create_user_with_valid_data(self, auth_client):
        """Создание пользователя с валидными данными."""
        response = auth_client.post("/users", json={
            "email": "valid@example.com",
            "name": "Valid User",
            "role": "user"
        })

        assert response.status_code == 201
        data = response.json()
        assert data["email"] == "valid@example.com"
        assert data["name"] == "Valid User"
        assert "id" in data
        assert "created_at" in data

        # Cleanup
        auth_client.delete(f"/users/{data['id']}")

    def test_create_user_without_required_fields(self, auth_client):
        """Создание пользователя без обязательных полей."""
        response = auth_client.post("/users", json={
            "name": "No Email User"
        })

        assert response.status_code == 422
        errors = response.json()["detail"]
        assert any("email" in str(e).lower() for e in errors)

    def test_get_existing_user(self, auth_client, created_user):
        """Получение существующего пользователя."""
        response = auth_client.get(f"/users/{created_user['id']}")

        assert response.status_code == 200
        data = response.json()
        assert data["id"] == created_user["id"]
        assert data["email"] == created_user["email"]

    def test_get_nonexistent_user(self, auth_client):
        """Получение несуществующего пользователя."""
        response = auth_client.get("/users/999999")

        assert response.status_code == 404
        assert "not found" in response.json()["detail"].lower()

    def test_update_user(self, auth_client, created_user):
        """Обновление данных пользователя."""
        response = auth_client.patch(
            f"/users/{created_user['id']}",
            json={"name": "Updated Name"}
        )

        assert response.status_code == 200
        assert response.json()["name"] == "Updated Name"
        assert response.json()["email"] == created_user["email"]

    def test_delete_user(self, auth_client):
        """Удаление пользователя."""
        # Создаем пользователя
        create_response = auth_client.post("/users", json={
            "email": "todelete@example.com",
            "name": "To Delete",
            "role": "user"
        })
        user_id = create_response.json()["id"]

        # Удаляем
        delete_response = auth_client.delete(f"/users/{user_id}")
        assert delete_response.status_code == 204

        # Проверяем, что удален
        get_response = auth_client.get(f"/users/{user_id}")
        assert get_response.status_code == 404

    def test_list_users_pagination(self, auth_client):
        """Тестирование пагинации списка пользователей."""
        # Первая страница
        response = auth_client.get("/users", params={"page": 1, "size": 10})

        assert response.status_code == 200
        data = response.json()
        assert "items" in data
        assert "total" in data
        assert "page" in data
        assert "size" in data
        assert len(data["items"]) <= 10
```

### Тестирование бизнес-логики

```python
# test_orders_business_logic.py
import pytest
from datetime import datetime, timedelta

class TestOrderBusinessLogic:
    """Функциональные тесты бизнес-логики заказов."""

    @pytest.fixture
    def product(self, auth_client):
        """Создание тестового продукта."""
        response = auth_client.post("/products", json={
            "name": "Test Product",
            "price": 100.00,
            "stock": 50
        })
        product = response.json()
        yield product
        auth_client.delete(f"/products/{product['id']}")

    def test_order_calculates_total_correctly(self, auth_client, product):
        """Заказ правильно рассчитывает итоговую сумму."""
        response = auth_client.post("/orders", json={
            "items": [
                {"product_id": product["id"], "quantity": 3}
            ]
        })

        assert response.status_code == 201
        order = response.json()
        # 3 * 100.00 = 300.00
        assert order["total"] == 300.00

    def test_order_with_discount(self, auth_client, product):
        """Применение скидки к заказу."""
        response = auth_client.post("/orders", json={
            "items": [
                {"product_id": product["id"], "quantity": 2}
            ],
            "discount_code": "SAVE10"  # 10% скидка
        })

        assert response.status_code == 201
        order = response.json()
        # 2 * 100.00 = 200.00, со скидкой 10% = 180.00
        assert order["total"] == 180.00
        assert order["discount_applied"] == 20.00

    def test_order_respects_stock_limit(self, auth_client, product):
        """Заказ не превышает количество на складе."""
        response = auth_client.post("/orders", json={
            "items": [
                {"product_id": product["id"], "quantity": 100}  # stock = 50
            ]
        })

        assert response.status_code == 400
        assert "stock" in response.json()["detail"].lower()

    def test_order_status_transitions(self, auth_client, product):
        """Правильные переходы статусов заказа."""
        # Создаем заказ
        order_response = auth_client.post("/orders", json={
            "items": [{"product_id": product["id"], "quantity": 1}]
        })
        order_id = order_response.json()["id"]
        assert order_response.json()["status"] == "pending"

        # Оплачиваем
        pay_response = auth_client.post(f"/orders/{order_id}/pay", json={
            "payment_method": "card"
        })
        assert pay_response.status_code == 200
        assert pay_response.json()["status"] == "paid"

        # Отправляем
        ship_response = auth_client.post(f"/orders/{order_id}/ship")
        assert ship_response.status_code == 200
        assert ship_response.json()["status"] == "shipped"

        # Доставляем
        deliver_response = auth_client.post(f"/orders/{order_id}/deliver")
        assert deliver_response.status_code == 200
        assert deliver_response.json()["status"] == "delivered"

    def test_cannot_cancel_shipped_order(self, auth_client, product):
        """Нельзя отменить отправленный заказ."""
        # Создаем и отправляем заказ
        order = auth_client.post("/orders", json={
            "items": [{"product_id": product["id"], "quantity": 1}]
        }).json()
        auth_client.post(f"/orders/{order['id']}/pay", json={"payment_method": "card"})
        auth_client.post(f"/orders/{order['id']}/ship")

        # Пытаемся отменить
        cancel_response = auth_client.post(f"/orders/{order['id']}/cancel")

        assert cancel_response.status_code == 400
        assert "cannot cancel" in cancel_response.json()["detail"].lower()

    def test_order_within_cancellation_window(self, auth_client, product):
        """Заказ можно отменить в течение окна отмены."""
        order = auth_client.post("/orders", json={
            "items": [{"product_id": product["id"], "quantity": 1}]
        }).json()

        # Отменяем сразу после создания
        cancel_response = auth_client.post(f"/orders/{order['id']}/cancel")

        assert cancel_response.status_code == 200
        assert cancel_response.json()["status"] == "cancelled"
```

### Тестирование авторизации

```python
# test_authorization.py
import pytest

class TestAuthorization:
    """Функциональные тесты авторизации."""

    @pytest.fixture
    def admin_client(self, api_client):
        """Клиент с правами администратора."""
        response = api_client.post("/auth/login", json={
            "email": "admin@example.com",
            "password": "adminpass"
        })
        token = response.json()["access_token"]
        return APIClient(base_url="http://localhost:8000/api", token=token)

    @pytest.fixture
    def user_client(self, api_client):
        """Клиент с правами обычного пользователя."""
        response = api_client.post("/auth/login", json={
            "email": "user@example.com",
            "password": "userpass"
        })
        token = response.json()["access_token"]
        return APIClient(base_url="http://localhost:8000/api", token=token)

    def test_unauthenticated_access_denied(self, api_client):
        """Неаутентифицированный доступ запрещен."""
        response = api_client.get("/users")

        assert response.status_code == 401
        assert "unauthorized" in response.json()["detail"].lower()

    def test_invalid_token_rejected(self, api_client):
        """Невалидный токен отклоняется."""
        client = APIClient(
            base_url="http://localhost:8000/api",
            token="invalid.token.here"
        )
        response = client.get("/users")

        assert response.status_code == 401

    def test_expired_token_rejected(self, api_client):
        """Истекший токен отклоняется."""
        # Используем заранее сгенерированный истекший токен
        expired_token = "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9..."
        client = APIClient(
            base_url="http://localhost:8000/api",
            token=expired_token
        )
        response = client.get("/users")

        assert response.status_code == 401
        assert "expired" in response.json()["detail"].lower()

    def test_admin_can_delete_users(self, admin_client, user_client):
        """Администратор может удалять пользователей."""
        # Создаем пользователя
        user = admin_client.post("/users", json={
            "email": "todelete@example.com",
            "name": "To Delete",
            "role": "user"
        }).json()

        # Удаляем как администратор
        response = admin_client.delete(f"/users/{user['id']}")
        assert response.status_code == 204

    def test_user_cannot_delete_other_users(self, admin_client, user_client):
        """Обычный пользователь не может удалять других."""
        # Создаем пользователя как админ
        user = admin_client.post("/users", json={
            "email": "protected@example.com",
            "name": "Protected",
            "role": "user"
        }).json()

        # Пытаемся удалить как обычный пользователь
        response = user_client.delete(f"/users/{user['id']}")

        assert response.status_code == 403
        assert "forbidden" in response.json()["detail"].lower()

        # Cleanup
        admin_client.delete(f"/users/{user['id']}")

    def test_user_can_access_own_profile(self, user_client):
        """Пользователь может видеть свой профиль."""
        response = user_client.get("/users/me")

        assert response.status_code == 200
        assert "email" in response.json()

    def test_user_cannot_access_admin_endpoints(self, user_client):
        """Пользователь не может использовать административные эндпоинты."""
        response = user_client.get("/admin/statistics")

        assert response.status_code == 403
```

### Тестирование валидации

```python
# test_validation.py
import pytest
from jsonschema import validate, ValidationError

# JSON Schema для ответа пользователя
USER_SCHEMA = {
    "type": "object",
    "required": ["id", "email", "name", "created_at"],
    "properties": {
        "id": {"type": "integer"},
        "email": {"type": "string", "format": "email"},
        "name": {"type": "string", "minLength": 1},
        "created_at": {"type": "string", "format": "date-time"},
        "role": {"type": "string", "enum": ["user", "admin", "moderator"]}
    }
}

class TestValidation:
    """Тесты валидации входных данных и схем ответов."""

    def test_response_matches_schema(self, auth_client):
        """Ответ соответствует JSON Schema."""
        response = auth_client.get("/users/1")

        assert response.status_code == 200
        validate(instance=response.json(), schema=USER_SCHEMA)

    @pytest.mark.parametrize("invalid_email", [
        "not-an-email",
        "@nodomain.com",
        "user@",
        "",
        "user name@domain.com",
    ])
    def test_invalid_email_rejected(self, auth_client, invalid_email):
        """Невалидные email адреса отклоняются."""
        response = auth_client.post("/users", json={
            "email": invalid_email,
            "name": "Test User"
        })

        assert response.status_code == 422

    @pytest.mark.parametrize("field,value,expected_error", [
        ("name", "", "must not be empty"),
        ("name", "a" * 256, "too long"),
        ("role", "superadmin", "not a valid choice"),
    ])
    def test_field_validation_errors(
        self, auth_client, field, value, expected_error
    ):
        """Тестирование ошибок валидации полей."""
        payload = {
            "email": "test@example.com",
            "name": "Test User",
            field: value
        }
        response = auth_client.post("/users", json=payload)

        assert response.status_code == 422
        error_details = str(response.json()["detail"]).lower()
        assert expected_error in error_details or field in error_details

    def test_request_body_too_large(self, auth_client):
        """Слишком большой запрос отклоняется."""
        large_content = "x" * (10 * 1024 * 1024)  # 10MB
        response = auth_client.post("/posts", json={
            "title": "Test",
            "content": large_content
        })

        assert response.status_code in [413, 422]

    def test_content_type_validation(self, auth_client):
        """Проверка Content-Type."""
        response = auth_client.session.post(
            f"{auth_client.base_url}/users",
            data="not json",
            headers={"Content-Type": "text/plain"}
        )

        assert response.status_code in [415, 422]
```

## Примеры на JavaScript

### Настройка

```javascript
// test/functional/setup.js
const axios = require('axios');

class APIClient {
    constructor(baseURL, token = null) {
        this.client = axios.create({
            baseURL,
            headers: token ? { Authorization: `Bearer ${token}` } : {}
        });
    }

    async login(email, password) {
        const response = await this.client.post('/auth/login', {
            email, password
        });
        this.client.defaults.headers.Authorization =
            `Bearer ${response.data.access_token}`;
        return response.data;
    }

    get(path, config) { return this.client.get(path, config); }
    post(path, data, config) { return this.client.post(path, data, config); }
    patch(path, data, config) { return this.client.patch(path, data, config); }
    delete(path, config) { return this.client.delete(path, config); }
}

module.exports = { APIClient };
```

### Функциональные тесты

```javascript
// test/functional/orders.test.js
const { APIClient } = require('./setup');

describe('Order Business Logic', () => {
    let client;
    let product;

    beforeAll(async () => {
        client = new APIClient('http://localhost:8000/api');
        await client.login('test@example.com', 'password123');

        // Создаем тестовый продукт
        const response = await client.post('/products', {
            name: 'Test Product',
            price: 100.00,
            stock: 50
        });
        product = response.data;
    });

    afterAll(async () => {
        await client.delete(`/products/${product.id}`);
    });

    describe('Order Creation', () => {
        test('calculates total correctly', async () => {
            const response = await client.post('/orders', {
                items: [{ product_id: product.id, quantity: 3 }]
            });

            expect(response.status).toBe(201);
            expect(response.data.total).toBe(300.00);
        });

        test('applies discount code', async () => {
            const response = await client.post('/orders', {
                items: [{ product_id: product.id, quantity: 2 }],
                discount_code: 'SAVE10'
            });

            expect(response.status).toBe(201);
            expect(response.data.total).toBe(180.00);
            expect(response.data.discount_applied).toBe(20.00);
        });

        test('rejects order exceeding stock', async () => {
            try {
                await client.post('/orders', {
                    items: [{ product_id: product.id, quantity: 100 }]
                });
                fail('Should have thrown error');
            } catch (error) {
                expect(error.response.status).toBe(400);
                expect(error.response.data.detail).toMatch(/stock/i);
            }
        });
    });

    describe('Order Status Workflow', () => {
        let orderId;

        beforeEach(async () => {
            const response = await client.post('/orders', {
                items: [{ product_id: product.id, quantity: 1 }]
            });
            orderId = response.data.id;
        });

        test('follows correct status transitions', async () => {
            // Initial status
            let response = await client.get(`/orders/${orderId}`);
            expect(response.data.status).toBe('pending');

            // Pay
            response = await client.post(`/orders/${orderId}/pay`, {
                payment_method: 'card'
            });
            expect(response.data.status).toBe('paid');

            // Ship
            response = await client.post(`/orders/${orderId}/ship`);
            expect(response.data.status).toBe('shipped');

            // Deliver
            response = await client.post(`/orders/${orderId}/deliver`);
            expect(response.data.status).toBe('delivered');
        });

        test('cannot cancel shipped order', async () => {
            await client.post(`/orders/${orderId}/pay`, {
                payment_method: 'card'
            });
            await client.post(`/orders/${orderId}/ship`);

            try {
                await client.post(`/orders/${orderId}/cancel`);
                fail('Should have thrown error');
            } catch (error) {
                expect(error.response.status).toBe(400);
                expect(error.response.data.detail).toMatch(/cannot cancel/i);
            }
        });
    });
});
```

## Best Practices

### 1. Независимые тесты

```python
# Каждый тест должен создавать свои данные
class TestOrders:
    @pytest.fixture
    def order(self, auth_client, product):
        """Создает заказ для теста."""
        order = auth_client.post("/orders", json={...}).json()
        yield order
        auth_client.delete(f"/orders/{order['id']}")

    def test_something(self, order):
        # Тест использует свой заказ
        pass
```

### 2. Проверяйте полный ответ

```python
def test_user_response_structure(auth_client):
    response = auth_client.get("/users/1")

    data = response.json()
    # Проверяем все ожидаемые поля
    assert "id" in data
    assert "email" in data
    assert "name" in data
    assert "created_at" in data
    # Проверяем отсутствие секретных данных
    assert "password" not in data
    assert "password_hash" not in data
```

### 3. Тестируйте граничные случаи

```python
class TestPagination:
    def test_first_page(self, auth_client):
        response = auth_client.get("/users?page=1&size=10")
        assert response.status_code == 200

    def test_empty_page(self, auth_client):
        response = auth_client.get("/users?page=9999&size=10")
        assert response.status_code == 200
        assert response.json()["items"] == []

    def test_invalid_page(self, auth_client):
        response = auth_client.get("/users?page=-1")
        assert response.status_code == 422

    def test_page_size_limit(self, auth_client):
        response = auth_client.get("/users?size=1000")
        # Размер страницы ограничен максимумом
        assert len(response.json()["items"]) <= 100
```

### 4. Используйте фикстуры для setup/teardown

```python
@pytest.fixture(scope="class")
def test_data(auth_client):
    """Подготовка тестовых данных."""
    users = []
    for i in range(5):
        user = auth_client.post("/users", json={
            "email": f"user{i}@example.com",
            "name": f"User {i}"
        }).json()
        users.append(user)

    yield {"users": users}

    # Cleanup
    for user in users:
        auth_client.delete(f"/users/{user['id']}")
```

## Инструменты

| Инструмент | Описание |
|------------|----------|
| **pytest** | Фреймворк для тестирования Python |
| **requests** | HTTP-клиент для Python |
| **jsonschema** | Валидация JSON Schema |
| **Pydantic** | Валидация данных с типизацией |
| **Jest** | Фреймворк для JavaScript |
| **axios** | HTTP-клиент для JavaScript |
| **Postman/Newman** | GUI и CLI для тестирования API |
| **Schemathesis** | Автоматическое тестирование по OpenAPI |

## Автоматизация с Schemathesis

```bash
pip install schemathesis

# Автоматическое тестирование API по OpenAPI спецификации
schemathesis run http://localhost:8000/openapi.json
```

```python
# Интеграция с pytest
import schemathesis

schema = schemathesis.from_uri("http://localhost:8000/openapi.json")

@schema.parametrize()
def test_api(case):
    """Автоматически тестирует все эндпоинты."""
    response = case.call()
    case.validate_response(response)
```

## Заключение

Функциональное тестирование API обеспечивает:
- Соответствие бизнес-требованиям
- Правильную работу пользовательских сценариев
- Корректную обработку ошибок
- Безопасность и авторизацию

Ключевые принципы:
- Тестируйте поведение, а не реализацию
- Покрывайте все критические сценарии
- Автоматизируйте тесты для CI/CD
- Используйте спецификации (OpenAPI) для валидации
