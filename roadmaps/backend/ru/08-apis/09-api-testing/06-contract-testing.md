# Contract Testing (Контрактное тестирование)

## Введение

**Контрактное тестирование** - это методология тестирования, при которой проверяется соответствие взаимодействия между сервисами заранее определенному контракту. Контракт описывает ожидаемые запросы и ответы между потребителем (consumer) и провайдером (provider) API.

## Зачем нужно контрактное тестирование?

### Проблемы традиционного подхода

1. **Интеграционные тесты медленные** - требуют запуска всех сервисов
2. **Сложность координации** - изменения в одном сервисе ломают другие
3. **Ненадежность** - тесты могут падать из-за сетевых проблем
4. **Позднее обнаружение проблем** - ошибки находятся на поздних этапах

### Преимущества контрактного тестирования

1. **Раннее обнаружение несовместимостей** - до развертывания
2. **Независимое тестирование** - каждый сервис тестируется отдельно
3. **Документация API** - контракты служат живой документацией
4. **Быстрые тесты** - не требуют реальных сервисов
5. **Уверенность в совместимости** - при изменениях

## Виды контрактного тестирования

### Consumer-Driven Contracts (CDC)

Потребитель определяет ожидания от провайдера.

```
Consumer -> записывает ожидания -> Contract -> проверяется на -> Provider
```

### Provider-Driven Contracts

Провайдер определяет контракт, потребители должны его соблюдать.

```
Provider -> публикует контракт -> Consumers -> адаптируются
```

### Bi-Directional Contracts

Обе стороны участвуют в определении контракта.

## Pact - инструмент для CDC

### Установка

```bash
# Python
pip install pact-python

# JavaScript
npm install --save-dev @pact-foundation/pact
```

### Consumer Side (Python)

```python
# test_consumer.py
import pytest
import requests
from pact import Consumer, Provider

# Создаем Pact между consumer и provider
pact = Consumer('OrderService').has_pact_with(
    Provider('UserService'),
    host_name='localhost',
    port=1234,
    pact_dir='./pacts'
)

@pytest.fixture(scope='session')
def pact_setup():
    pact.start_service()
    yield pact
    pact.stop_service()

class TestUserServiceConsumer:
    """Тесты контрактов для UserService."""

    def test_get_user_by_id(self, pact_setup):
        """Контракт: получение пользователя по ID."""
        # Определяем ожидаемое взаимодействие
        (pact_setup
            .given('user with id 1 exists')
            .upon_receiving('a request for user 1')
            .with_request('GET', '/api/users/1')
            .will_respond_with(200, body={
                'id': 1,
                'name': 'John Doe',
                'email': 'john@example.com'
            }))

        with pact_setup:
            # Выполняем запрос к mock-серверу
            response = requests.get('http://localhost:1234/api/users/1')

            # Проверяем, что наш код правильно обрабатывает ответ
            assert response.status_code == 200
            user = response.json()
            assert user['id'] == 1
            assert 'email' in user

    def test_get_user_not_found(self, pact_setup):
        """Контракт: пользователь не найден."""
        (pact_setup
            .given('user with id 999 does not exist')
            .upon_receiving('a request for non-existent user')
            .with_request('GET', '/api/users/999')
            .will_respond_with(404, body={
                'error': 'User not found'
            }))

        with pact_setup:
            response = requests.get('http://localhost:1234/api/users/999')

            assert response.status_code == 404
            assert 'error' in response.json()

    def test_create_user(self, pact_setup):
        """Контракт: создание пользователя."""
        expected_user = {
            'id': 1,
            'name': 'New User',
            'email': 'new@example.com'
        }

        (pact_setup
            .given('no user with email new@example.com exists')
            .upon_receiving('a request to create a user')
            .with_request(
                'POST',
                '/api/users',
                headers={'Content-Type': 'application/json'},
                body={
                    'name': 'New User',
                    'email': 'new@example.com'
                }
            )
            .will_respond_with(201, body=expected_user))

        with pact_setup:
            response = requests.post(
                'http://localhost:1234/api/users',
                json={'name': 'New User', 'email': 'new@example.com'},
                headers={'Content-Type': 'application/json'}
            )

            assert response.status_code == 201
            assert response.json()['id'] == 1
```

### Provider Side (Python)

```python
# test_provider.py
import pytest
from pact import Verifier
from app import create_app
import subprocess

@pytest.fixture(scope='module')
def app():
    """Создание тестового приложения."""
    app = create_app(config='testing')
    return app

@pytest.fixture(scope='module')
def provider_server(app):
    """Запуск провайдера для верификации."""
    import threading
    from werkzeug.serving import make_server

    server = make_server('localhost', 8080, app)
    thread = threading.Thread(target=server.serve_forever)
    thread.start()

    yield 'http://localhost:8080'

    server.shutdown()

class TestProviderVerification:
    """Верификация контрактов провайдером."""

    def test_verify_pacts(self, provider_server):
        """Проверка всех контрактов."""
        verifier = Verifier(
            provider='UserService',
            provider_base_url=provider_server
        )

        # Настройка состояний провайдера
        verifier.set_state_handler(self.setup_provider_states)

        # Верификация контрактов
        success, logs = verifier.verify_pacts(
            './pacts/orderservice-userservice.json',
            enable_pending=False,
            publish_version='1.0.0',
            provider_states_setup_url=f'{provider_server}/_pact/provider_states'
        )

        assert success, f"Pact verification failed: {logs}"

    def setup_provider_states(self, state):
        """Настройка состояния провайдера."""
        if state == 'user with id 1 exists':
            # Создаем пользователя в тестовой БД
            create_test_user(id=1, name='John Doe', email='john@example.com')

        elif state == 'user with id 999 does not exist':
            # Убеждаемся, что пользователя нет
            delete_test_user(id=999)

        elif state == 'no user with email new@example.com exists':
            # Удаляем пользователя с таким email если есть
            delete_user_by_email('new@example.com')
```

### Consumer Side (JavaScript)

```javascript
// consumer.spec.js
const { Pact } = require('@pact-foundation/pact');
const { UserClient } = require('../src/userClient');
const path = require('path');

describe('User Service Contract', () => {
    const provider = new Pact({
        consumer: 'OrderService',
        provider: 'UserService',
        port: 1234,
        log: path.resolve(process.cwd(), 'logs', 'pact.log'),
        dir: path.resolve(process.cwd(), 'pacts'),
        logLevel: 'warn',
    });

    beforeAll(() => provider.setup());
    afterAll(() => provider.finalize());
    afterEach(() => provider.verify());

    describe('get user by id', () => {
        test('returns user when exists', async () => {
            // Arrange
            await provider.addInteraction({
                state: 'user with id 1 exists',
                uponReceiving: 'a request for user 1',
                withRequest: {
                    method: 'GET',
                    path: '/api/users/1',
                    headers: {
                        Accept: 'application/json'
                    }
                },
                willRespondWith: {
                    status: 200,
                    headers: {
                        'Content-Type': 'application/json'
                    },
                    body: {
                        id: 1,
                        name: 'John Doe',
                        email: 'john@example.com'
                    }
                }
            });

            // Act
            const client = new UserClient('http://localhost:1234');
            const user = await client.getUser(1);

            // Assert
            expect(user.id).toBe(1);
            expect(user.name).toBe('John Doe');
        });

        test('returns 404 when user not found', async () => {
            await provider.addInteraction({
                state: 'user with id 999 does not exist',
                uponReceiving: 'a request for non-existent user',
                withRequest: {
                    method: 'GET',
                    path: '/api/users/999',
                    headers: {
                        Accept: 'application/json'
                    }
                },
                willRespondWith: {
                    status: 404,
                    body: {
                        error: 'User not found'
                    }
                }
            });

            const client = new UserClient('http://localhost:1234');

            await expect(client.getUser(999))
                .rejects.toThrow('User not found');
        });
    });

    describe('create user', () => {
        test('creates user successfully', async () => {
            await provider.addInteraction({
                state: 'no user with email new@example.com exists',
                uponReceiving: 'a request to create a user',
                withRequest: {
                    method: 'POST',
                    path: '/api/users',
                    headers: {
                        'Content-Type': 'application/json'
                    },
                    body: {
                        name: 'New User',
                        email: 'new@example.com'
                    }
                },
                willRespondWith: {
                    status: 201,
                    body: {
                        id: 1,
                        name: 'New User',
                        email: 'new@example.com'
                    }
                }
            });

            const client = new UserClient('http://localhost:1234');
            const user = await client.createUser({
                name: 'New User',
                email: 'new@example.com'
            });

            expect(user.id).toBe(1);
        });
    });
});
```

### Provider Side (JavaScript)

```javascript
// provider.spec.js
const { Verifier } = require('@pact-foundation/pact');
const app = require('../src/app');
const { setupTestData, clearTestData } = require('./helpers');

describe('Provider Verification', () => {
    let server;

    beforeAll((done) => {
        server = app.listen(8080, done);
    });

    afterAll((done) => {
        server.close(done);
    });

    test('validates the expectations of OrderService', async () => {
        const opts = {
            provider: 'UserService',
            providerBaseUrl: 'http://localhost:8080',
            pactUrls: ['./pacts/orderservice-userservice.json'],
            stateHandlers: {
                'user with id 1 exists': async () => {
                    await setupTestData({
                        users: [{ id: 1, name: 'John Doe', email: 'john@example.com' }]
                    });
                },
                'user with id 999 does not exist': async () => {
                    await clearTestData();
                },
                'no user with email new@example.com exists': async () => {
                    await clearTestData();
                }
            }
        };

        const result = await new Verifier(opts).verifyProvider();
        expect(result).toBeTruthy();
    });
});
```

## Pact Broker

### Развертывание

```yaml
# docker-compose.yml
version: '3'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: pact
      POSTGRES_PASSWORD: pact
      POSTGRES_DB: pact

  pact-broker:
    image: pactfoundation/pact-broker
    ports:
      - "9292:9292"
    environment:
      PACT_BROKER_DATABASE_URL: postgres://pact:pact@postgres/pact
      PACT_BROKER_BASIC_AUTH_USERNAME: admin
      PACT_BROKER_BASIC_AUTH_PASSWORD: admin
    depends_on:
      - postgres
```

### Публикация контрактов

```bash
# Python
pact-broker publish ./pacts \
    --broker-base-url=http://localhost:9292 \
    --consumer-app-version=1.0.0 \
    --tag=main

# JavaScript
npx pact-broker publish ./pacts \
    --broker-base-url=http://localhost:9292 \
    --consumer-app-version=1.0.0 \
    --tag=main
```

### Верификация из Broker

```python
# test_provider_broker.py
def test_verify_pacts_from_broker():
    verifier = Verifier(
        provider='UserService',
        provider_base_url='http://localhost:8080'
    )

    success, logs = verifier.verify_with_broker(
        broker_url='http://localhost:9292',
        broker_username='admin',
        broker_password='admin',
        publish_version='1.0.0',
        provider_tags=['main']
    )

    assert success
```

## OpenAPI Contract Testing

### Schemathesis

```bash
pip install schemathesis
```

```python
# test_openapi_contract.py
import schemathesis

# Загрузка схемы
schema = schemathesis.from_uri("http://localhost:8000/openapi.json")

@schema.parametrize()
def test_api_contract(case):
    """Автоматическое тестирование всех эндпоинтов."""
    response = case.call()
    case.validate_response(response)

# Или загрузка из файла
schema = schemathesis.from_path("./openapi.yaml")

# С аутентификацией
@schema.parametrize()
def test_api_with_auth(case):
    response = case.call(
        headers={"Authorization": "Bearer token123"}
    )
    case.validate_response(response)
```

### Dredd

```bash
npm install -g dredd

# Запуск тестов
dredd openapi.yaml http://localhost:8000

# С конфигурацией
dredd
```

```yaml
# dredd.yml
dry-run: false
hookfiles: ./hooks.js
language: nodejs
endpoint: http://localhost:8000
path:
  - ./openapi.yaml
reporter:
  - apiary
  - cli
```

```javascript
// hooks.js
const hooks = require('hooks');

hooks.beforeAll((transactions, done) => {
    // Подготовка перед всеми тестами
    done();
});

hooks.beforeEach((transaction, done) => {
    // Добавление заголовков
    transaction.request.headers['Authorization'] = 'Bearer token123';
    done();
});

hooks.before('Users > Get User > 200', (transaction, done) => {
    // Подготовка данных для конкретного теста
    createTestUser({ id: 1, name: 'John' }).then(done);
});
```

## Prism - Mock Server из OpenAPI

```bash
npm install -g @stoplight/prism-cli

# Запуск mock server
prism mock openapi.yaml

# С валидацией запросов
prism mock openapi.yaml --errors
```

## Spring Cloud Contract (Java)

```groovy
// build.gradle
plugins {
    id 'org.springframework.cloud.contract' version '3.1.0'
}

contracts {
    baseClassForTests = 'com.example.BaseContractTest'
    contractsDslDir = file("src/test/resources/contracts")
}
```

```groovy
// contracts/userService/getUser.groovy
Contract.make {
    description "should return user by id"

    request {
        method GET()
        url "/api/users/1"
        headers {
            accept(applicationJson())
        }
    }

    response {
        status OK()
        headers {
            contentType(applicationJson())
        }
        body([
            id: 1,
            name: "John Doe",
            email: "john@example.com"
        ])
    }
}
```

## CI/CD интеграция

### GitHub Actions

```yaml
# .github/workflows/contract-test.yml
name: Contract Tests

on: [push, pull_request]

jobs:
  consumer-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run consumer tests
        run: pytest tests/contract/consumer/

      - name: Publish pacts
        if: github.ref == 'refs/heads/main'
        run: |
          pact-broker publish ./pacts \
            --broker-base-url=${{ secrets.PACT_BROKER_URL }} \
            --consumer-app-version=${{ github.sha }} \
            --tag=main

  provider-tests:
    runs-on: ubuntu-latest
    needs: consumer-tests
    steps:
      - uses: actions/checkout@v3

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Verify pacts
        run: |
          python -m pytest tests/contract/provider/ \
            --broker-url=${{ secrets.PACT_BROKER_URL }}

      - name: Can I Deploy
        run: |
          pact-broker can-i-deploy \
            --pacticipant=UserService \
            --version=${{ github.sha }} \
            --to-environment=production
```

## Best Practices

### 1. Минимальные контракты

```python
# Плохо: слишком детальный контракт
.will_respond_with(200, body={
    'id': 1,
    'name': 'John Doe',
    'email': 'john@example.com',
    'created_at': '2024-01-01T00:00:00Z',
    'updated_at': '2024-01-01T00:00:00Z',
    'role': 'user',
    'settings': {...}
})

# Хорошо: только то, что нужно consumer
.will_respond_with(200, body={
    'id': Like(1),
    'email': Like('john@example.com')
})
```

### 2. Используйте matchers

```python
from pact import Like, EachLike, Term

# Соответствие типу
.will_respond_with(200, body={
    'id': Like(1),  # любое число
    'name': Like('string'),  # любая строка
})

# Массив элементов
.will_respond_with(200, body={
    'users': EachLike({
        'id': Like(1),
        'name': Like('User')
    })
})

# Регулярные выражения
.will_respond_with(200, body={
    'email': Term(r'.+@.+\..+', 'user@example.com')
})
```

### 3. Независимые состояния

```python
# Хорошо: каждый тест настраивает свое состояние
.given('user with id 1 exists and has 3 orders')
.given('user with id 2 exists and has no orders')
```

### 4. Версионирование контрактов

```bash
# Публикация с версией и тегом
pact-broker publish ./pacts \
    --consumer-app-version=$(git rev-parse HEAD) \
    --tag=$(git rev-parse --abbrev-ref HEAD)
```

## Инструменты

| Инструмент | Описание |
|------------|----------|
| **Pact** | Consumer-driven contract testing |
| **Pact Broker** | Хранение и управление контрактами |
| **Schemathesis** | Тестирование по OpenAPI |
| **Dredd** | Тестирование по API Blueprint/OpenAPI |
| **Prism** | Mock server из OpenAPI |
| **Spring Cloud Contract** | Контракты для Java/Spring |

## Заключение

Контрактное тестирование решает ключевые проблемы микросервисной архитектуры:

- **Раннее обнаружение** - проблемы совместимости находятся до деплоя
- **Независимая разработка** - команды могут работать параллельно
- **Живая документация** - контракты всегда актуальны
- **Уверенность в изменениях** - можно безопасно менять API

Ключевые принципы:
- Consumer определяет минимальные требования
- Provider гарантирует выполнение контрактов
- Контракты версионируются и хранятся централизованно
- Интеграция с CI/CD обеспечивает автоматическую проверку
