# Mocking APIs (Мокирование API)

[prev: 04-load-testing](./04-load-testing.md) | [next: 06-contract-testing](./06-contract-testing.md)

---

## Введение

**Мокирование (Mocking)** - это техника создания имитации (мока) внешних зависимостей для изолированного тестирования. В контексте API мокирование позволяет:

- Тестировать код без реальных внешних сервисов
- Симулировать различные сценарии (ошибки, задержки)
- Ускорить выполнение тестов
- Обеспечить детерминированность тестов

## Что мокировать?

### Внешние API
- Платежные системы (Stripe, PayPal)
- Сервисы авторизации (OAuth, SSO)
- Третьи сервисы (email, SMS, push)
- API партнеров

### Внутренние зависимости
- Базы данных
- Очереди сообщений
- Кэш (Redis)
- Файловые системы

## Типы моков

### 1. Mock (Мок)
Объект с заранее запрограммированным поведением:

```python
from unittest.mock import Mock

payment_service = Mock()
payment_service.charge.return_value = {"status": "success", "id": "pay_123"}
```

### 2. Stub (Стаб)
Объект с фиксированными ответами:

```python
class PaymentServiceStub:
    def charge(self, amount):
        return {"status": "success", "id": "pay_123"}
```

### 3. Spy (Шпион)
Реальный объект, который записывает вызовы:

```python
from unittest.mock import patch

with patch.object(real_service, 'send', wraps=real_service.send) as spy:
    real_service.send("message")
    spy.assert_called_once_with("message")
```

### 4. Fake (Фейк)
Упрощенная рабочая реализация:

```python
class FakeEmailService:
    def __init__(self):
        self.sent_emails = []

    def send(self, to, subject, body):
        self.sent_emails.append({"to": to, "subject": subject, "body": body})
        return True
```

## Мокирование в Python

### unittest.mock

```python
from unittest.mock import Mock, MagicMock, patch, AsyncMock
import pytest

# Базовый мок
mock = Mock()
mock.method.return_value = "result"
assert mock.method() == "result"

# Мок с побочным эффектом
mock.method.side_effect = ValueError("Error!")
with pytest.raises(ValueError):
    mock.method()

# Мок с последовательными значениями
mock.method.side_effect = [1, 2, 3]
assert mock.method() == 1
assert mock.method() == 2
assert mock.method() == 3

# Проверка вызовов
mock.method("arg1", key="value")
mock.method.assert_called_once_with("arg1", key="value")
```

### Декоратор @patch

```python
# services.py
import requests

class WeatherService:
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.weather.com"

    def get_weather(self, city: str) -> dict:
        response = requests.get(
            f"{self.base_url}/weather",
            params={"city": city, "key": self.api_key}
        )
        response.raise_for_status()
        return response.json()

    def get_forecast(self, city: str, days: int = 7) -> dict:
        response = requests.get(
            f"{self.base_url}/forecast",
            params={"city": city, "days": days, "key": self.api_key}
        )
        response.raise_for_status()
        return response.json()
```

```python
# test_services.py
from unittest.mock import patch, Mock
import pytest
from services import WeatherService

class TestWeatherService:
    @pytest.fixture
    def service(self):
        return WeatherService(api_key="test_key")

    @patch("services.requests.get")
    def test_get_weather_success(self, mock_get, service):
        """Тест успешного получения погоды."""
        # Настраиваем мок ответа
        mock_response = Mock()
        mock_response.json.return_value = {
            "city": "Moscow",
            "temperature": 20,
            "conditions": "sunny"
        }
        mock_response.raise_for_status = Mock()
        mock_get.return_value = mock_response

        # Вызываем метод
        result = service.get_weather("Moscow")

        # Проверяем результат
        assert result["city"] == "Moscow"
        assert result["temperature"] == 20

        # Проверяем, что запрос был сделан правильно
        mock_get.assert_called_once_with(
            "https://api.weather.com/weather",
            params={"city": "Moscow", "key": "test_key"}
        )

    @patch("services.requests.get")
    def test_get_weather_api_error(self, mock_get, service):
        """Тест обработки ошибки API."""
        mock_response = Mock()
        mock_response.raise_for_status.side_effect = requests.HTTPError("404")
        mock_get.return_value = mock_response

        with pytest.raises(requests.HTTPError):
            service.get_weather("Unknown City")

    @patch("services.requests.get")
    def test_get_weather_network_error(self, mock_get, service):
        """Тест обработки сетевой ошибки."""
        mock_get.side_effect = requests.ConnectionError("Network error")

        with pytest.raises(requests.ConnectionError):
            service.get_weather("Moscow")
```

### Мокирование асинхронных функций

```python
# async_services.py
import httpx

class AsyncPaymentService:
    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url
        self.api_key = api_key

    async def create_payment(self, amount: float, currency: str) -> dict:
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.base_url}/payments",
                json={"amount": amount, "currency": currency},
                headers={"Authorization": f"Bearer {self.api_key}"}
            )
            response.raise_for_status()
            return response.json()

    async def get_payment(self, payment_id: str) -> dict:
        async with httpx.AsyncClient() as client:
            response = await client.get(
                f"{self.base_url}/payments/{payment_id}",
                headers={"Authorization": f"Bearer {self.api_key}"}
            )
            response.raise_for_status()
            return response.json()
```

```python
# test_async_services.py
import pytest
from unittest.mock import AsyncMock, patch, MagicMock
from async_services import AsyncPaymentService

class TestAsyncPaymentService:
    @pytest.fixture
    def service(self):
        return AsyncPaymentService(
            base_url="https://api.payment.com",
            api_key="test_key"
        )

    @pytest.mark.asyncio
    async def test_create_payment_success(self, service):
        """Тест создания платежа."""
        mock_response = MagicMock()
        mock_response.json.return_value = {
            "id": "pay_123",
            "status": "pending",
            "amount": 100.0
        }
        mock_response.raise_for_status = MagicMock()

        with patch("async_services.httpx.AsyncClient") as mock_client_class:
            mock_client = AsyncMock()
            mock_client.post.return_value = mock_response
            mock_client_class.return_value.__aenter__.return_value = mock_client

            result = await service.create_payment(100.0, "USD")

            assert result["id"] == "pay_123"
            assert result["status"] == "pending"

            mock_client.post.assert_called_once_with(
                "https://api.payment.com/payments",
                json={"amount": 100.0, "currency": "USD"},
                headers={"Authorization": "Bearer test_key"}
            )

    @pytest.mark.asyncio
    async def test_get_payment_not_found(self, service):
        """Тест получения несуществующего платежа."""
        with patch("async_services.httpx.AsyncClient") as mock_client_class:
            mock_client = AsyncMock()
            mock_client.get.side_effect = httpx.HTTPStatusError(
                "Not Found",
                request=MagicMock(),
                response=MagicMock(status_code=404)
            )
            mock_client_class.return_value.__aenter__.return_value = mock_client

            with pytest.raises(httpx.HTTPStatusError):
                await service.get_payment("invalid_id")
```

### pytest-mock

```python
# Более удобный интерфейс для мокирования
import pytest

def test_with_mocker(mocker):
    """Использование pytest-mock."""
    mock_get = mocker.patch("requests.get")
    mock_get.return_value.json.return_value = {"data": "test"}

    # Ваш тест
    pass

def test_spy(mocker):
    """Использование шпиона."""
    spy = mocker.spy(some_module, "some_function")

    some_module.some_function("arg")

    spy.assert_called_once_with("arg")
```

## Мокирование HTTP-запросов

### responses (для requests)

```bash
pip install responses
```

```python
import responses
import requests

@responses.activate
def test_external_api():
    """Мокирование внешнего API с responses."""
    # Регистрируем мок ответа
    responses.add(
        responses.GET,
        "https://api.example.com/users",
        json={"users": [{"id": 1, "name": "John"}]},
        status=200
    )

    # Делаем запрос
    response = requests.get("https://api.example.com/users")

    # Проверяем
    assert response.status_code == 200
    assert response.json()["users"][0]["name"] == "John"

@responses.activate
def test_api_error():
    """Мокирование ошибки API."""
    responses.add(
        responses.GET,
        "https://api.example.com/users",
        json={"error": "Server Error"},
        status=500
    )

    response = requests.get("https://api.example.com/users")
    assert response.status_code == 500

@responses.activate
def test_multiple_calls():
    """Мокирование нескольких вызовов."""
    # Первый вызов
    responses.add(
        responses.POST,
        "https://api.example.com/token",
        json={"access_token": "token123"},
        status=200
    )

    # Второй вызов
    responses.add(
        responses.GET,
        "https://api.example.com/data",
        json={"data": "secret"},
        status=200
    )

    # Тест
    token_response = requests.post("https://api.example.com/token")
    data_response = requests.get("https://api.example.com/data")

    assert token_response.json()["access_token"] == "token123"
    assert data_response.json()["data"] == "secret"
```

### respx (для httpx)

```bash
pip install respx
```

```python
import httpx
import respx
import pytest

@respx.mock
@pytest.mark.asyncio
async def test_async_api():
    """Мокирование асинхронного API."""
    # Регистрируем мок
    respx.get("https://api.example.com/users").mock(
        return_value=httpx.Response(
            200,
            json={"users": [{"id": 1}]}
        )
    )

    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/users")

    assert response.status_code == 200
    assert response.json()["users"][0]["id"] == 1

@respx.mock
@pytest.mark.asyncio
async def test_with_pattern():
    """Мокирование с паттерном URL."""
    # Мок для любого ID пользователя
    respx.get(url__regex=r"https://api.example.com/users/\d+").mock(
        return_value=httpx.Response(200, json={"id": 1, "name": "User"})
    )

    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/users/123")

    assert response.status_code == 200

@respx.mock
@pytest.mark.asyncio
async def test_side_effect():
    """Мокирование с side effect."""
    call_count = 0

    def custom_handler(request):
        nonlocal call_count
        call_count += 1
        if call_count == 1:
            return httpx.Response(429)  # Rate limit
        return httpx.Response(200, json={"data": "success"})

    respx.get("https://api.example.com/data").mock(side_effect=custom_handler)

    async with httpx.AsyncClient() as client:
        # Первый вызов - rate limit
        r1 = await client.get("https://api.example.com/data")
        assert r1.status_code == 429

        # Второй вызов - успех
        r2 = await client.get("https://api.example.com/data")
        assert r2.status_code == 200
```

### VCR.py (запись и воспроизведение)

```bash
pip install vcrpy
```

```python
import vcr
import requests

# Записывает реальные HTTP-ответы в файл и воспроизводит их
@vcr.use_cassette('fixtures/vcr_cassettes/weather.yaml')
def test_weather_api():
    """Тест с записанными ответами."""
    response = requests.get(
        "https://api.weather.com/weather",
        params={"city": "Moscow"}
    )

    assert response.status_code == 200
    assert "temperature" in response.json()

# Настройка VCR
my_vcr = vcr.VCR(
    cassette_library_dir='fixtures/vcr_cassettes',
    record_mode='once',  # Записать один раз, потом воспроизводить
    match_on=['uri', 'method'],
    filter_headers=['authorization'],  # Не записывать секреты
    filter_query_parameters=['api_key']
)

@my_vcr.use_cassette('api_call.yaml')
def test_with_custom_vcr():
    pass
```

## Мокирование в JavaScript

### Jest mocks

```javascript
// services/paymentService.js
const axios = require('axios');

class PaymentService {
    constructor(apiKey) {
        this.apiKey = apiKey;
        this.baseUrl = 'https://api.payment.com';
    }

    async createPayment(amount, currency) {
        const response = await axios.post(
            `${this.baseUrl}/payments`,
            { amount, currency },
            { headers: { 'Authorization': `Bearer ${this.apiKey}` } }
        );
        return response.data;
    }
}

module.exports = { PaymentService };
```

```javascript
// __tests__/paymentService.test.js
const axios = require('axios');
const { PaymentService } = require('../services/paymentService');

// Мокаем весь модуль axios
jest.mock('axios');

describe('PaymentService', () => {
    let service;

    beforeEach(() => {
        service = new PaymentService('test_key');
        jest.clearAllMocks();
    });

    test('creates payment successfully', async () => {
        // Настраиваем мок
        axios.post.mockResolvedValue({
            data: {
                id: 'pay_123',
                status: 'pending',
                amount: 100
            }
        });

        const result = await service.createPayment(100, 'USD');

        expect(result.id).toBe('pay_123');
        expect(result.status).toBe('pending');
        expect(axios.post).toHaveBeenCalledWith(
            'https://api.payment.com/payments',
            { amount: 100, currency: 'USD' },
            { headers: { 'Authorization': 'Bearer test_key' } }
        );
    });

    test('handles API error', async () => {
        axios.post.mockRejectedValue(new Error('Payment failed'));

        await expect(service.createPayment(100, 'USD'))
            .rejects.toThrow('Payment failed');
    });

    test('handles multiple calls', async () => {
        axios.post
            .mockResolvedValueOnce({ data: { id: 'pay_1' } })
            .mockResolvedValueOnce({ data: { id: 'pay_2' } });

        const result1 = await service.createPayment(100, 'USD');
        const result2 = await service.createPayment(200, 'EUR');

        expect(result1.id).toBe('pay_1');
        expect(result2.id).toBe('pay_2');
        expect(axios.post).toHaveBeenCalledTimes(2);
    });
});
```

### nock (HTTP мокирование)

```bash
npm install --save-dev nock
```

```javascript
const nock = require('nock');
const axios = require('axios');

describe('External API', () => {
    afterEach(() => {
        nock.cleanAll();
    });

    test('mocks GET request', async () => {
        nock('https://api.example.com')
            .get('/users')
            .reply(200, {
                users: [{ id: 1, name: 'John' }]
            });

        const response = await axios.get('https://api.example.com/users');

        expect(response.data.users[0].name).toBe('John');
    });

    test('mocks POST request with body matching', async () => {
        nock('https://api.example.com')
            .post('/users', { name: 'John', email: 'john@example.com' })
            .reply(201, { id: 1 });

        const response = await axios.post('https://api.example.com/users', {
            name: 'John',
            email: 'john@example.com'
        });

        expect(response.status).toBe(201);
        expect(response.data.id).toBe(1);
    });

    test('mocks with delay', async () => {
        nock('https://api.example.com')
            .get('/slow')
            .delay(1000)  // 1 секунда задержки
            .reply(200, { data: 'slow response' });

        const start = Date.now();
        await axios.get('https://api.example.com/slow');
        const duration = Date.now() - start;

        expect(duration).toBeGreaterThan(1000);
    });

    test('mocks error response', async () => {
        nock('https://api.example.com')
            .get('/error')
            .reply(500, { error: 'Internal Server Error' });

        try {
            await axios.get('https://api.example.com/error');
            fail('Should have thrown');
        } catch (error) {
            expect(error.response.status).toBe(500);
        }
    });

    test('mocks network error', async () => {
        nock('https://api.example.com')
            .get('/network-error')
            .replyWithError('Connection refused');

        await expect(axios.get('https://api.example.com/network-error'))
            .rejects.toThrow('Connection refused');
    });
});
```

## Mock Server (WireMock)

### Запуск WireMock

```bash
# Docker
docker run -d -p 8080:8080 wiremock/wiremock

# Или как standalone
java -jar wiremock-standalone.jar --port 8080
```

### Конфигурация моков

```json
// mappings/get-users.json
{
    "request": {
        "method": "GET",
        "urlPattern": "/api/users.*"
    },
    "response": {
        "status": 200,
        "headers": {
            "Content-Type": "application/json"
        },
        "jsonBody": {
            "users": [
                {"id": 1, "name": "John"},
                {"id": 2, "name": "Jane"}
            ]
        }
    }
}
```

```json
// mappings/create-user.json
{
    "request": {
        "method": "POST",
        "url": "/api/users",
        "bodyPatterns": [
            {"matchesJsonPath": "$.name"},
            {"matchesJsonPath": "$.email"}
        ]
    },
    "response": {
        "status": 201,
        "jsonBody": {
            "id": "{{randomValue type='UUID'}}",
            "name": "{{jsonPath request.body '$.name'}}",
            "email": "{{jsonPath request.body '$.email'}}"
        },
        "transformers": ["response-template"]
    }
}
```

### Использование в тестах

```python
import pytest
import requests

@pytest.fixture(scope="session")
def wiremock_url():
    return "http://localhost:8080"

def test_get_users(wiremock_url):
    response = requests.get(f"{wiremock_url}/api/users")

    assert response.status_code == 200
    assert len(response.json()["users"]) == 2
```

## Best Practices

### 1. Мокайте на правильном уровне

```python
# Плохо: мокаем слишком низко
with patch("socket.socket"):
    pass

# Хорошо: мокаем на уровне HTTP-клиента
with patch("requests.get"):
    pass

# Еще лучше: мокаем сервис
with patch.object(payment_service, "charge"):
    pass
```

### 2. Проверяйте, что моки вызваны

```python
def test_email_sent_on_registration(mocker):
    mock_send = mocker.patch("services.email.send")

    register_user("test@example.com", "password")

    # Обязательно проверяем вызов
    mock_send.assert_called_once_with(
        to="test@example.com",
        subject="Welcome!",
        template="welcome"
    )
```

### 3. Используйте фикстуры для общих моков

```python
@pytest.fixture
def mock_payment_gateway(mocker):
    """Общий мок платежного шлюза."""
    mock = mocker.patch("services.PaymentGateway")
    mock.return_value.charge.return_value = {
        "status": "success",
        "transaction_id": "txn_123"
    }
    return mock
```

### 4. Документируйте сложные моки

```python
@pytest.fixture
def mock_oauth_flow(mocker):
    """
    Мокирует полный OAuth flow:
    1. Получение authorization code
    2. Обмен на access token
    3. Получение user info
    """
    mock_oauth = mocker.patch("services.oauth")
    mock_oauth.get_auth_url.return_value = "https://oauth.example.com/auth?..."
    mock_oauth.exchange_code.return_value = {"access_token": "token123"}
    mock_oauth.get_user_info.return_value = {"id": "user_1", "email": "user@example.com"}
    return mock_oauth
```

## Инструменты

| Инструмент | Язык | Описание |
|------------|------|----------|
| **unittest.mock** | Python | Встроенная библиотека |
| **pytest-mock** | Python | Удобная обертка для pytest |
| **responses** | Python | Мокирование requests |
| **respx** | Python | Мокирование httpx |
| **VCR.py** | Python | Запись/воспроизведение HTTP |
| **Jest** | JavaScript | Встроенные моки |
| **nock** | JavaScript | HTTP мокирование |
| **WireMock** | Java/Any | Standalone mock server |
| **Prism** | Any | Mock server из OpenAPI |

## Заключение

Мокирование - неотъемлемая часть эффективного тестирования API:

- **Изоляция** - тесты не зависят от внешних сервисов
- **Скорость** - моки работают мгновенно
- **Контроль** - можно симулировать любые сценарии
- **Надежность** - тесты всегда детерминированы

Ключевые правила:
- Мокайте только то, что необходимо
- Проверяйте вызовы моков
- Используйте реалистичные данные
- Не забывайте тестировать реальную интеграцию

---

[prev: 04-load-testing](./04-load-testing.md) | [next: 06-contract-testing](./06-contract-testing.md)
