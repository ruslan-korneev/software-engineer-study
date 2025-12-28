# Зачем нужны микросервисы?

[prev: readme.md](./readme.md) | [next: 02-backend-for-frontend.md](./02-backend-for-frontend.md)

---

## Введение

Микросервисная архитектура — это подход к разработке программного обеспечения, при котором приложение строится как набор небольших, независимых сервисов. Каждый сервис выполняет определённую бизнес-функцию, имеет собственную базу данных и может быть развёрнут независимо от других.

## Монолит vs Микросервисы

### Монолитная архитектура

```
┌─────────────────────────────────────────┐
│              МОНОЛИТ                     │
│  ┌─────────┬─────────┬─────────┐        │
│  │  Users  │ Orders  │ Products│        │
│  ├─────────┴─────────┴─────────┤        │
│  │       Общая база данных      │        │
│  └─────────────────────────────┘        │
└─────────────────────────────────────────┘
```

### Микросервисная архитектура

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│  Users   │    │  Orders  │    │ Products │
│  Service │    │  Service │    │  Service │
├──────────┤    ├──────────┤    ├──────────┤
│  Users   │    │  Orders  │    │ Products │
│    DB    │    │    DB    │    │    DB    │
└──────────┘    └──────────┘    └──────────┘
      │              │               │
      └──────────────┼───────────────┘
                     │
              ┌──────────┐
              │   API    │
              │ Gateway  │
              └──────────┘
```

## Преимущества микросервисов

### 1. Независимое развёртывание

Каждый сервис можно обновлять независимо, без необходимости перезапуска всего приложения.

```python
# Пример: независимый сервис пользователей
# user_service/main.py
import asyncio
from aiohttp import web
import aiohttp

class UserService:
    def __init__(self):
        self.users = {}  # В реальности - база данных
        self.next_id = 1

    async def create_user(self, request: web.Request) -> web.Response:
        data = await request.json()
        user_id = self.next_id
        self.next_id += 1

        self.users[user_id] = {
            'id': user_id,
            'name': data['name'],
            'email': data['email']
        }

        return web.json_response(self.users[user_id], status=201)

    async def get_user(self, request: web.Request) -> web.Response:
        user_id = int(request.match_info['id'])

        if user_id not in self.users:
            return web.json_response(
                {'error': 'User not found'},
                status=404
            )

        return web.json_response(self.users[user_id])

    async def health_check(self, request: web.Request) -> web.Response:
        """Health endpoint для оркестраторов (Kubernetes, Docker Swarm)"""
        return web.json_response({'status': 'healthy', 'service': 'users'})


def create_app() -> web.Application:
    service = UserService()
    app = web.Application()

    app.router.add_post('/users', service.create_user)
    app.router.add_get('/users/{id}', service.get_user)
    app.router.add_get('/health', service.health_check)

    return app


if __name__ == '__main__':
    web.run_app(create_app(), port=8001)
```

### 2. Технологическая гибкость

Разные сервисы могут использовать разные технологии:

```python
# order_service/main.py - использует PostgreSQL
import asyncio
import asyncpg
from aiohttp import web


class OrderService:
    def __init__(self, db_pool: asyncpg.Pool):
        self.db = db_pool

    async def create_order(self, request: web.Request) -> web.Response:
        data = await request.json()

        async with self.db.acquire() as conn:
            order_id = await conn.fetchval('''
                INSERT INTO orders (user_id, product_id, quantity, status)
                VALUES ($1, $2, $3, 'pending')
                RETURNING id
            ''', data['user_id'], data['product_id'], data['quantity'])

        return web.json_response({'order_id': order_id}, status=201)

    async def get_order(self, request: web.Request) -> web.Response:
        order_id = int(request.match_info['id'])

        async with self.db.acquire() as conn:
            order = await conn.fetchrow(
                'SELECT * FROM orders WHERE id = $1',
                order_id
            )

        if not order:
            return web.json_response({'error': 'Order not found'}, status=404)

        return web.json_response(dict(order))


async def create_app() -> web.Application:
    db_pool = await asyncpg.create_pool(
        'postgresql://user:pass@localhost/orders_db',
        min_size=5,
        max_size=20
    )

    service = OrderService(db_pool)
    app = web.Application()

    app.router.add_post('/orders', service.create_order)
    app.router.add_get('/orders/{id}', service.get_order)

    # Cleanup при завершении
    async def cleanup(app):
        await db_pool.close()

    app.on_cleanup.append(cleanup)

    return app


if __name__ == '__main__':
    web.run_app(create_app(), port=8002)
```

### 3. Масштабирование по необходимости

```python
# Пример конфигурации для масштабирования
# docker-compose.yml (в виде Python-словаря для понимания)

services_config = {
    'user_service': {
        'image': 'user-service:latest',
        'replicas': 2,  # 2 экземпляра
        'ports': ['8001:8001']
    },
    'order_service': {
        'image': 'order-service:latest',
        'replicas': 5,  # 5 экземпляров (больше нагрузки)
        'ports': ['8002:8002']
    },
    'product_service': {
        'image': 'product-service:latest',
        'replicas': 3,  # 3 экземпляра
        'ports': ['8003:8003']
    }
}
```

## Коммуникация между сервисами

### Синхронная коммуникация (HTTP/REST)

```python
import aiohttp
import asyncio
from typing import Optional


class ServiceClient:
    """Базовый клиент для взаимодействия с другими сервисами"""

    def __init__(self, base_url: str, timeout: int = 30):
        self.base_url = base_url
        self.timeout = aiohttp.ClientTimeout(total=timeout)
        self._session: Optional[aiohttp.ClientSession] = None

    async def _get_session(self) -> aiohttp.ClientSession:
        if self._session is None or self._session.closed:
            self._session = aiohttp.ClientSession(timeout=self.timeout)
        return self._session

    async def get(self, path: str) -> dict:
        session = await self._get_session()
        async with session.get(f'{self.base_url}{path}') as response:
            response.raise_for_status()
            return await response.json()

    async def post(self, path: str, data: dict) -> dict:
        session = await self._get_session()
        async with session.post(f'{self.base_url}{path}', json=data) as response:
            response.raise_for_status()
            return await response.json()

    async def close(self):
        if self._session and not self._session.closed:
            await self._session.close()


# Использование
class OrderServiceWithUserValidation:
    def __init__(self):
        self.user_client = ServiceClient('http://user-service:8001')
        self.product_client = ServiceClient('http://product-service:8003')

    async def create_order(self, user_id: int, product_id: int, quantity: int):
        # Проверяем существование пользователя
        try:
            user = await self.user_client.get(f'/users/{user_id}')
        except aiohttp.ClientError:
            raise ValueError(f"User {user_id} not found")

        # Проверяем доступность товара
        try:
            product = await self.product_client.get(f'/products/{product_id}')
            if product['stock'] < quantity:
                raise ValueError("Insufficient stock")
        except aiohttp.ClientError:
            raise ValueError(f"Product {product_id} not found")

        # Создаём заказ...
        return {'status': 'created', 'user': user, 'product': product}
```

### Асинхронная коммуникация (Message Queue)

```python
import asyncio
import aio_pika
import json


class MessageBroker:
    """Брокер сообщений для асинхронной коммуникации"""

    def __init__(self, rabbitmq_url: str):
        self.url = rabbitmq_url
        self.connection = None
        self.channel = None

    async def connect(self):
        self.connection = await aio_pika.connect_robust(self.url)
        self.channel = await self.connection.channel()

    async def publish(self, queue_name: str, message: dict):
        """Публикация сообщения в очередь"""
        queue = await self.channel.declare_queue(queue_name, durable=True)

        await self.channel.default_exchange.publish(
            aio_pika.Message(
                body=json.dumps(message).encode(),
                delivery_mode=aio_pika.DeliveryMode.PERSISTENT
            ),
            routing_key=queue_name
        )

    async def consume(self, queue_name: str, callback):
        """Подписка на сообщения из очереди"""
        queue = await self.channel.declare_queue(queue_name, durable=True)

        async def process_message(message: aio_pika.IncomingMessage):
            async with message.process():
                data = json.loads(message.body.decode())
                await callback(data)

        await queue.consume(process_message)

    async def close(self):
        if self.connection:
            await self.connection.close()


# Пример: сервис уведомлений
class NotificationService:
    def __init__(self, broker: MessageBroker):
        self.broker = broker

    async def start(self):
        await self.broker.connect()
        await self.broker.consume('order_events', self.handle_order_event)

    async def handle_order_event(self, event: dict):
        """Обработка событий заказов"""
        if event['type'] == 'order_created':
            print(f"Отправляем email пользователю {event['user_id']}")
            # await send_email(...)
        elif event['type'] == 'order_shipped':
            print(f"Отправляем SMS о доставке")
            # await send_sms(...)


# Пример: публикация события из сервиса заказов
async def publish_order_created(broker: MessageBroker, order: dict):
    await broker.publish('order_events', {
        'type': 'order_created',
        'order_id': order['id'],
        'user_id': order['user_id'],
        'timestamp': '2024-01-15T10:30:00Z'
    })
```

## Паттерны отказоустойчивости

### Circuit Breaker (Предохранитель)

```python
import asyncio
from enum import Enum
from datetime import datetime, timedelta
from typing import Callable, Any


class CircuitState(Enum):
    CLOSED = 'closed'      # Нормальная работа
    OPEN = 'open'          # Запросы блокируются
    HALF_OPEN = 'half_open'  # Тестовый режим


class CircuitBreaker:
    """
    Паттерн Circuit Breaker для защиты от каскадных отказов.

    CLOSED -> OPEN: после N последовательных ошибок
    OPEN -> HALF_OPEN: после timeout
    HALF_OPEN -> CLOSED: успешный запрос
    HALF_OPEN -> OPEN: ошибка
    """

    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: int = 30,
        half_open_max_calls: int = 3
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.half_open_max_calls = half_open_max_calls

        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time = None
        self.half_open_calls = 0

    async def call(self, func: Callable, *args, **kwargs) -> Any:
        """Выполняет функцию через circuit breaker"""

        if self.state == CircuitState.OPEN:
            if self._should_try_reset():
                self.state = CircuitState.HALF_OPEN
                self.half_open_calls = 0
            else:
                raise CircuitBreakerOpenError("Circuit breaker is open")

        if self.state == CircuitState.HALF_OPEN:
            if self.half_open_calls >= self.half_open_max_calls:
                raise CircuitBreakerOpenError("Half-open limit reached")
            self.half_open_calls += 1

        try:
            result = await func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _should_try_reset(self) -> bool:
        if self.last_failure_time is None:
            return True
        return datetime.now() - self.last_failure_time > timedelta(
            seconds=self.recovery_timeout
        )

    def _on_success(self):
        self.failure_count = 0
        self.state = CircuitState.CLOSED

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = datetime.now()

        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN


class CircuitBreakerOpenError(Exception):
    pass


# Использование
class ResilientServiceClient:
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.circuit_breaker = CircuitBreaker(
            failure_threshold=3,
            recovery_timeout=60
        )

    async def get_user(self, user_id: int) -> dict:
        async def _fetch():
            async with aiohttp.ClientSession() as session:
                async with session.get(
                    f'{self.base_url}/users/{user_id}'
                ) as resp:
                    resp.raise_for_status()
                    return await resp.json()

        try:
            return await self.circuit_breaker.call(_fetch)
        except CircuitBreakerOpenError:
            # Возвращаем fallback или cached данные
            return {'id': user_id, 'name': 'Unknown', 'cached': True}
```

## Распространённые ошибки

### 1. Distributed Monolith (Распределённый монолит)

```python
# ПЛОХО: Сервисы сильно связаны
class BadOrderService:
    async def create_order(self, data):
        # Синхронно вызываем множество сервисов
        user = await self.user_service.get_user(data['user_id'])
        product = await self.product_service.get_product(data['product_id'])
        inventory = await self.inventory_service.check_stock(data['product_id'])
        pricing = await self.pricing_service.get_price(data['product_id'])
        discount = await self.discount_service.apply_discount(user, product)
        # ... ещё 10 вызовов
        # Это распределённый монолит!


# ХОРОШО: Минимизация зависимостей
class GoodOrderService:
    async def create_order(self, data):
        # Только необходимые данные
        order = await self.db.create_order(data)

        # Асинхронные события для остальных операций
        await self.event_bus.publish('order.created', order)

        return order
```

### 2. Отсутствие idempotency (идемпотентности)

```python
# ПЛОХО: Повторный запрос создаст дубликат
class BadPaymentService:
    async def process_payment(self, order_id: int, amount: float):
        # При retry создастся новый платёж!
        payment = await self.db.insert_payment(order_id, amount)
        return payment


# ХОРОШО: Идемпотентная операция
class GoodPaymentService:
    async def process_payment(
        self,
        order_id: int,
        amount: float,
        idempotency_key: str  # Уникальный ключ от клиента
    ):
        # Проверяем, был ли уже обработан этот запрос
        existing = await self.db.get_payment_by_key(idempotency_key)
        if existing:
            return existing

        payment = await self.db.insert_payment(
            order_id,
            amount,
            idempotency_key
        )
        return payment
```

## Когда НЕ использовать микросервисы

1. **Маленькая команда** (< 5 разработчиков)
2. **Простой домен** — нет сложной бизнес-логики
3. **Стартап на ранней стадии** — требования постоянно меняются
4. **Отсутствие DevOps-экспертизы** — нужны навыки работы с контейнерами, оркестрацией
5. **Нет необходимости в независимом масштабировании**

## Резюме

| Аспект | Монолит | Микросервисы |
|--------|---------|--------------|
| Развёртывание | Всё вместе | Независимо |
| Масштабирование | Всё приложение | По сервисам |
| Технологии | Единый стек | Разные стеки |
| Сложность | Проще в начале | Выше |
| Отладка | Проще | Сложнее |
| Команда | Любой размер | Большие команды |

---

[prev: readme.md](./readme.md) | [next: 02-backend-for-frontend.md](./02-backend-for-frontend.md)
