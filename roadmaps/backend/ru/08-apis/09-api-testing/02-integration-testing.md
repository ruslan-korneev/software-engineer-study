# Integration Testing (Интеграционное тестирование)

## Введение

**Интеграционное тестирование** - это уровень тестирования, при котором проверяется взаимодействие между несколькими компонентами системы. В отличие от unit-тестов, которые проверяют изолированные части кода, интеграционные тесты проверяют, как эти части работают вместе.

В контексте API интеграционное тестирование проверяет:
- Взаимодействие с базой данных
- Работу с внешними сервисами
- Корректность HTTP-запросов и ответов
- Middleware и аутентификацию
- Сериализацию/десериализацию данных

## Виды интеграционного тестирования

### 1. Big Bang Integration

Все компоненты интегрируются одновременно и тестируются как единое целое.

**Преимущества:**
- Простота подхода
- Подходит для небольших систем

**Недостатки:**
- Сложно локализовать ошибки
- Требует готовности всех компонентов

### 2. Incremental Integration

Компоненты добавляются и тестируются постепенно.

#### Top-Down
Тестирование начинается с верхнего уровня (API endpoints) и постепенно спускается к нижним уровням.

#### Bottom-Up
Тестирование начинается с нижних уровней (репозитории, сервисы) и поднимается вверх.

#### Sandwich (Hybrid)
Комбинация top-down и bottom-up подходов.

## Практические примеры на Python

### Настройка тестовой инфраструктуры

```bash
pip install pytest pytest-asyncio httpx testcontainers sqlalchemy asyncpg
```

### Тестирование с реальной базой данных

```python
# conftest.py
import pytest
import asyncio
from typing import AsyncGenerator
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
from testcontainers.postgres import PostgresContainer

from app.database import Base
from app.main import app
from app.dependencies import get_db

# Фикстура для PostgreSQL контейнера
@pytest.fixture(scope="session")
def postgres_container():
    """Запуск PostgreSQL в Docker контейнере."""
    with PostgresContainer("postgres:15") as postgres:
        yield postgres

@pytest.fixture(scope="session")
def event_loop():
    """Создание event loop для асинхронных тестов."""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()

@pytest.fixture(scope="session")
async def async_engine(postgres_container):
    """Создание асинхронного engine для тестов."""
    connection_url = postgres_container.get_connection_url()
    # Преобразуем URL для asyncpg
    async_url = connection_url.replace("postgresql://", "postgresql+asyncpg://")

    engine = create_async_engine(async_url, echo=True)

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    yield engine

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)

    await engine.dispose()

@pytest.fixture
async def db_session(async_engine) -> AsyncGenerator[AsyncSession, None]:
    """Создание сессии базы данных для каждого теста."""
    async_session = sessionmaker(
        async_engine, class_=AsyncSession, expire_on_commit=False
    )

    async with async_session() as session:
        yield session
        await session.rollback()

@pytest.fixture
async def client(db_session):
    """Тестовый клиент с подменой зависимости базы данных."""
    from httpx import AsyncClient

    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db

    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac

    app.dependency_overrides.clear()
```

### Модели и репозиторий

```python
# models.py
from sqlalchemy import Column, Integer, String, DateTime, ForeignKey
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func
from app.database import Base

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True, nullable=False)
    name = Column(String, nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())

    posts = relationship("Post", back_populates="author")

class Post(Base):
    __tablename__ = "posts"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String, nullable=False)
    content = Column(String, nullable=False)
    author_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())

    author = relationship("User", back_populates="posts")
```

```python
# repositories.py
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload
from typing import Optional, List

from app.models import User, Post

class UserRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def create(self, email: str, name: str) -> User:
        user = User(email=email, name=name)
        self.session.add(user)
        await self.session.commit()
        await self.session.refresh(user)
        return user

    async def get_by_id(self, user_id: int) -> Optional[User]:
        result = await self.session.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()

    async def get_by_email(self, email: str) -> Optional[User]:
        result = await self.session.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()

    async def get_with_posts(self, user_id: int) -> Optional[User]:
        result = await self.session.execute(
            select(User)
            .options(selectinload(User.posts))
            .where(User.id == user_id)
        )
        return result.scalar_one_or_none()

class PostRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def create(self, title: str, content: str, author_id: int) -> Post:
        post = Post(title=title, content=content, author_id=author_id)
        self.session.add(post)
        await self.session.commit()
        await self.session.refresh(post)
        return post

    async def get_by_author(self, author_id: int) -> List[Post]:
        result = await self.session.execute(
            select(Post).where(Post.author_id == author_id)
        )
        return list(result.scalars().all())
```

### Интеграционные тесты репозиториев

```python
# test_repositories.py
import pytest
from app.repositories import UserRepository, PostRepository

class TestUserRepository:
    """Интеграционные тесты для UserRepository."""

    @pytest.fixture
    def user_repo(self, db_session):
        return UserRepository(db_session)

    @pytest.mark.asyncio
    async def test_create_user(self, user_repo):
        """Тест создания пользователя в базе данных."""
        user = await user_repo.create(
            email="test@example.com",
            name="Test User"
        )

        assert user.id is not None
        assert user.email == "test@example.com"
        assert user.name == "Test User"
        assert user.created_at is not None

    @pytest.mark.asyncio
    async def test_get_user_by_id(self, user_repo):
        """Тест получения пользователя по ID."""
        created = await user_repo.create("test@example.com", "Test User")

        fetched = await user_repo.get_by_id(created.id)

        assert fetched is not None
        assert fetched.id == created.id
        assert fetched.email == created.email

    @pytest.mark.asyncio
    async def test_get_user_by_email(self, user_repo):
        """Тест получения пользователя по email."""
        await user_repo.create("unique@example.com", "Unique User")

        fetched = await user_repo.get_by_email("unique@example.com")

        assert fetched is not None
        assert fetched.email == "unique@example.com"

    @pytest.mark.asyncio
    async def test_get_nonexistent_user(self, user_repo):
        """Тест получения несуществующего пользователя."""
        fetched = await user_repo.get_by_id(99999)

        assert fetched is None

class TestUserWithPosts:
    """Интеграционные тесты связей между User и Post."""

    @pytest.mark.asyncio
    async def test_create_user_with_posts(self, db_session):
        """Тест создания пользователя с постами."""
        user_repo = UserRepository(db_session)
        post_repo = PostRepository(db_session)

        # Создаем пользователя
        user = await user_repo.create("author@example.com", "Author")

        # Создаем посты
        await post_repo.create("First Post", "Content 1", user.id)
        await post_repo.create("Second Post", "Content 2", user.id)

        # Получаем пользователя с постами
        user_with_posts = await user_repo.get_with_posts(user.id)

        assert user_with_posts is not None
        assert len(user_with_posts.posts) == 2
        assert user_with_posts.posts[0].author_id == user.id

    @pytest.mark.asyncio
    async def test_get_posts_by_author(self, db_session):
        """Тест получения всех постов автора."""
        user_repo = UserRepository(db_session)
        post_repo = PostRepository(db_session)

        # Создаем двух авторов
        author1 = await user_repo.create("author1@example.com", "Author 1")
        author2 = await user_repo.create("author2@example.com", "Author 2")

        # Создаем посты для каждого
        await post_repo.create("Post by Author 1", "Content", author1.id)
        await post_repo.create("Another by Author 1", "Content", author1.id)
        await post_repo.create("Post by Author 2", "Content", author2.id)

        # Получаем посты первого автора
        author1_posts = await post_repo.get_by_author(author1.id)

        assert len(author1_posts) == 2
        assert all(p.author_id == author1.id for p in author1_posts)
```

### Интеграционные тесты API

```python
# test_api_integration.py
import pytest

class TestUserAPIIntegration:
    """Интеграционные тесты для User API."""

    @pytest.mark.asyncio
    async def test_full_user_lifecycle(self, client):
        """Тест полного цикла: создание, получение, обновление."""
        # 1. Создание пользователя
        create_response = await client.post(
            "/api/users",
            json={"email": "lifecycle@example.com", "name": "Lifecycle Test"}
        )
        assert create_response.status_code == 201
        user_id = create_response.json()["id"]

        # 2. Получение пользователя
        get_response = await client.get(f"/api/users/{user_id}")
        assert get_response.status_code == 200
        assert get_response.json()["email"] == "lifecycle@example.com"

        # 3. Обновление пользователя
        update_response = await client.patch(
            f"/api/users/{user_id}",
            json={"name": "Updated Name"}
        )
        assert update_response.status_code == 200
        assert update_response.json()["name"] == "Updated Name"

        # 4. Проверка обновления
        verify_response = await client.get(f"/api/users/{user_id}")
        assert verify_response.json()["name"] == "Updated Name"

    @pytest.mark.asyncio
    async def test_create_user_with_posts(self, client):
        """Тест создания пользователя и его постов."""
        # Создаем пользователя
        user_response = await client.post(
            "/api/users",
            json={"email": "blogger@example.com", "name": "Blogger"}
        )
        user_id = user_response.json()["id"]

        # Создаем пост
        post_response = await client.post(
            "/api/posts",
            json={
                "title": "My First Post",
                "content": "Hello, World!",
                "author_id": user_id
            }
        )
        assert post_response.status_code == 201

        # Получаем пользователя с постами
        user_with_posts = await client.get(f"/api/users/{user_id}/posts")
        assert user_with_posts.status_code == 200
        assert len(user_with_posts.json()) == 1
        assert user_with_posts.json()[0]["title"] == "My First Post"

    @pytest.mark.asyncio
    async def test_database_constraints(self, client):
        """Тест соблюдения ограничений базы данных."""
        # Создаем пользователя
        await client.post(
            "/api/users",
            json={"email": "unique@example.com", "name": "First"}
        )

        # Пытаемся создать с тем же email
        duplicate_response = await client.post(
            "/api/users",
            json={"email": "unique@example.com", "name": "Second"}
        )

        assert duplicate_response.status_code == 400
        assert "already exists" in duplicate_response.json()["detail"].lower()
```

### Тестирование с внешними сервисами

```python
# test_external_services.py
import pytest
from unittest.mock import AsyncMock, patch
import httpx

class TestPaymentIntegration:
    """Тесты интеграции с платежным сервисом."""

    @pytest.fixture
    def mock_payment_service(self):
        """Мок для платежного сервиса."""
        with patch("app.services.payment.PaymentClient") as mock:
            client = AsyncMock()
            mock.return_value = client
            yield client

    @pytest.mark.asyncio
    async def test_successful_payment_flow(self, client, mock_payment_service):
        """Тест успешного процесса оплаты."""
        # Настраиваем мок ответа платежного сервиса
        mock_payment_service.create_payment.return_value = {
            "payment_id": "pay_123",
            "status": "pending",
            "confirmation_url": "https://payment.example.com/confirm/123"
        }
        mock_payment_service.confirm_payment.return_value = {
            "payment_id": "pay_123",
            "status": "completed"
        }

        # Создаем заказ
        order_response = await client.post(
            "/api/orders",
            json={"items": [{"product_id": 1, "quantity": 2}]}
        )
        order_id = order_response.json()["id"]

        # Инициируем оплату
        payment_response = await client.post(
            f"/api/orders/{order_id}/pay",
            json={"payment_method": "card"}
        )

        assert payment_response.status_code == 200
        assert "confirmation_url" in payment_response.json()

        # Подтверждаем оплату (webhook от платежной системы)
        confirm_response = await client.post(
            "/api/webhooks/payment",
            json={"payment_id": "pay_123", "status": "completed"}
        )

        assert confirm_response.status_code == 200

        # Проверяем статус заказа
        order_status = await client.get(f"/api/orders/{order_id}")
        assert order_status.json()["status"] == "paid"

class TestEmailServiceIntegration:
    """Тесты интеграции с email-сервисом."""

    @pytest.mark.asyncio
    async def test_registration_sends_welcome_email(self, client, mocker):
        """Тест отправки welcome-email при регистрации."""
        # Мокаем email-сервис
        send_email_mock = mocker.patch(
            "app.services.email.send_email",
            new_callable=AsyncMock
        )

        # Регистрируем пользователя
        response = await client.post(
            "/api/auth/register",
            json={
                "email": "newuser@example.com",
                "password": "SecurePass123",
                "name": "New User"
            }
        )

        assert response.status_code == 201

        # Проверяем, что email был отправлен
        send_email_mock.assert_called_once()
        call_args = send_email_mock.call_args
        assert call_args[1]["to"] == "newuser@example.com"
        assert "welcome" in call_args[1]["template"].lower()
```

## Примеры на JavaScript (Jest + Supertest)

### Настройка

```javascript
// jest.config.js
module.exports = {
    testEnvironment: 'node',
    setupFilesAfterEnv: ['./tests/setup.js'],
    testMatch: ['**/tests/integration/**/*.test.js'],
    globalSetup: './tests/globalSetup.js',
    globalTeardown: './tests/globalTeardown.js',
};
```

```javascript
// tests/globalSetup.js
const { PostgreSqlContainer } = require('@testcontainers/postgresql');

module.exports = async () => {
    // Запуск PostgreSQL контейнера
    const container = await new PostgreSqlContainer().start();

    // Сохраняем URL подключения в глобальные переменные
    process.env.DATABASE_URL = container.getConnectionUri();
    global.__POSTGRES_CONTAINER__ = container;
};
```

```javascript
// tests/globalTeardown.js
module.exports = async () => {
    if (global.__POSTGRES_CONTAINER__) {
        await global.__POSTGRES_CONTAINER__.stop();
    }
};
```

### Интеграционные тесты

```javascript
// tests/integration/users.test.js
const request = require('supertest');
const { app } = require('../../src/app');
const { sequelize } = require('../../src/database');
const { User, Post } = require('../../src/models');

describe('Users API Integration', () => {
    beforeAll(async () => {
        await sequelize.sync({ force: true });
    });

    afterEach(async () => {
        await Post.destroy({ where: {} });
        await User.destroy({ where: {} });
    });

    afterAll(async () => {
        await sequelize.close();
    });

    describe('User lifecycle', () => {
        test('complete user CRUD operations', async () => {
            // Create
            const createResponse = await request(app)
                .post('/api/users')
                .send({ email: 'test@example.com', name: 'Test User' });

            expect(createResponse.status).toBe(201);
            const userId = createResponse.body.id;

            // Read
            const getResponse = await request(app)
                .get(`/api/users/${userId}`);

            expect(getResponse.status).toBe(200);
            expect(getResponse.body.email).toBe('test@example.com');

            // Update
            const updateResponse = await request(app)
                .patch(`/api/users/${userId}`)
                .send({ name: 'Updated Name' });

            expect(updateResponse.status).toBe(200);
            expect(updateResponse.body.name).toBe('Updated Name');

            // Delete
            const deleteResponse = await request(app)
                .delete(`/api/users/${userId}`);

            expect(deleteResponse.status).toBe(204);

            // Verify deletion
            const verifyResponse = await request(app)
                .get(`/api/users/${userId}`);

            expect(verifyResponse.status).toBe(404);
        });
    });

    describe('User with Posts', () => {
        test('creates user with related posts', async () => {
            // Create user
            const userResponse = await request(app)
                .post('/api/users')
                .send({ email: 'author@example.com', name: 'Author' });

            const userId = userResponse.body.id;

            // Create posts
            await request(app)
                .post('/api/posts')
                .send({
                    title: 'First Post',
                    content: 'Content here',
                    authorId: userId
                });

            await request(app)
                .post('/api/posts')
                .send({
                    title: 'Second Post',
                    content: 'More content',
                    authorId: userId
                });

            // Get user with posts
            const userWithPosts = await request(app)
                .get(`/api/users/${userId}/posts`);

            expect(userWithPosts.status).toBe(200);
            expect(userWithPosts.body).toHaveLength(2);
        });

        test('enforces foreign key constraints', async () => {
            // Try to create post with non-existent author
            const response = await request(app)
                .post('/api/posts')
                .send({
                    title: 'Orphan Post',
                    content: 'No author',
                    authorId: 99999
                });

            expect(response.status).toBe(400);
        });
    });

    describe('Database constraints', () => {
        test('enforces unique email constraint', async () => {
            await request(app)
                .post('/api/users')
                .send({ email: 'unique@example.com', name: 'First' });

            const response = await request(app)
                .post('/api/users')
                .send({ email: 'unique@example.com', name: 'Second' });

            expect(response.status).toBe(400);
            expect(response.body.error).toMatch(/already exists/i);
        });
    });
});
```

## Best Practices

### 1. Изоляция тестов

```python
@pytest.fixture(autouse=True)
async def clean_database(db_session):
    """Очистка базы данных перед каждым тестом."""
    yield
    # Откат транзакции после теста
    await db_session.rollback()
```

### 2. Используйте транзакции

```python
@pytest.fixture
async def db_session(async_engine):
    """Каждый тест выполняется в отдельной транзакции."""
    async with async_engine.connect() as connection:
        transaction = await connection.begin()

        session = AsyncSession(bind=connection)
        yield session

        await transaction.rollback()
```

### 3. Тестовые контейнеры

```python
# Используйте testcontainers для реальных баз данных
from testcontainers.postgres import PostgresContainer
from testcontainers.redis import RedisContainer

@pytest.fixture(scope="session")
def postgres():
    with PostgresContainer("postgres:15") as pg:
        yield pg

@pytest.fixture(scope="session")
def redis():
    with RedisContainer("redis:7") as redis:
        yield redis
```

### 4. Фабрики для тестовых данных

```python
# factories.py
import factory
from factory.alchemy import SQLAlchemyModelFactory
from app.models import User, Post

class UserFactory(SQLAlchemyModelFactory):
    class Meta:
        model = User
        sqlalchemy_session_persistence = "commit"

    email = factory.Sequence(lambda n: f"user{n}@example.com")
    name = factory.Faker("name")

class PostFactory(SQLAlchemyModelFactory):
    class Meta:
        model = Post
        sqlalchemy_session_persistence = "commit"

    title = factory.Faker("sentence")
    content = factory.Faker("paragraph")
    author = factory.SubFactory(UserFactory)
```

### 5. Параллельное выполнение

```bash
# pytest с параллельным выполнением
pip install pytest-xdist
pytest -n auto tests/integration/
```

## Инструменты

| Инструмент | Описание |
|------------|----------|
| **testcontainers** | Docker-контейнеры для тестов |
| **pytest-asyncio** | Асинхронные тесты в pytest |
| **httpx** | Асинхронный HTTP-клиент |
| **factory_boy** | Фабрики для создания данных |
| **Faker** | Генерация фейковых данных |
| **pytest-xdist** | Параллельное выполнение тестов |

## Заключение

Интеграционное тестирование - это критически важный уровень тестирования API, который позволяет:
- Выявить проблемы взаимодействия между компонентами
- Проверить корректность работы с базой данных
- Убедиться в правильной обработке ошибок
- Валидировать бизнес-процессы end-to-end

Ключ к успешному интеграционному тестированию - правильная изоляция тестов и использование реалистичного тестового окружения.
