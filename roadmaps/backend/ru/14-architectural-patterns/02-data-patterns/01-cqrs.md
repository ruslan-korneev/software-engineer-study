# CQRS - Command Query Responsibility Segregation

## Определение

**CQRS (Command Query Responsibility Segregation)** — архитектурный паттерн, который разделяет операции чтения данных (Query) и операции изменения данных (Command) на две отдельные модели. Вместо использования единой модели данных для всех операций, CQRS предлагает использовать разные модели, оптимизированные под конкретные задачи.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Традиционный подход                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Client ──────► Single Model ──────► Single Database           │
│            (Read + Write)                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                          CQRS подход                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌─► Query Model ──────► Read Database         │
│   Client ─────────┤                                              │
│                    └─► Command Model ────► Write Database        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Паттерн основан на принципе CQS (Command Query Separation), предложенном Бертраном Мейером, но применяет его на архитектурном уровне.

---

## Ключевые характеристики

### 1. Разделение моделей

```python
# Command Model - оптимизирована для записи
class OrderCommandModel:
    def __init__(self):
        self.id: str
        self.customer_id: str
        self.items: List[OrderItem]
        self.status: OrderStatus
        self.version: int  # Optimistic locking

    def add_item(self, item: OrderItem) -> None:
        self._validate_can_modify()
        self.items.append(item)
        self.version += 1

    def submit(self) -> None:
        self._validate_can_submit()
        self.status = OrderStatus.SUBMITTED

# Query Model - оптимизирована для чтения (денормализована)
@dataclass
class OrderReadModel:
    id: str
    customer_name: str  # Денормализовано
    customer_email: str  # Денормализовано
    items_count: int
    total_amount: Decimal
    status: str
    created_at: datetime
    # Нет методов изменения - только данные
```

### 2. Отдельные хранилища данных

```python
class WriteRepository:
    """Репозиторий для команд - работает с нормализованными данными"""

    def __init__(self, session: Session):
        self.session = session

    async def save(self, order: OrderCommandModel) -> None:
        await self.session.execute(
            """
            INSERT INTO orders (id, customer_id, status, version)
            VALUES (:id, :customer_id, :status, :version)
            ON CONFLICT (id) DO UPDATE SET
                status = :status,
                version = :version
            WHERE orders.version = :expected_version
            """,
            {
                "id": order.id,
                "customer_id": order.customer_id,
                "status": order.status.value,
                "version": order.version,
                "expected_version": order.version - 1
            }
        )

class ReadRepository:
    """Репозиторий для запросов - работает с денормализованными данными"""

    def __init__(self, read_db: AsyncConnection):
        self.db = read_db

    async def get_order_summary(self, order_id: str) -> OrderReadModel:
        # Простой запрос к денормализованной таблице
        result = await self.db.fetch_one(
            "SELECT * FROM order_summaries WHERE id = :id",
            {"id": order_id}
        )
        return OrderReadModel(**result)

    async def get_customer_orders(
        self,
        customer_id: str,
        page: int,
        size: int
    ) -> List[OrderReadModel]:
        # Эффективный запрос без JOIN
        return await self.db.fetch_all(
            """
            SELECT * FROM order_summaries
            WHERE customer_id = :customer_id
            ORDER BY created_at DESC
            LIMIT :size OFFSET :offset
            """,
            {"customer_id": customer_id, "size": size, "offset": page * size}
        )
```

### 3. Синхронизация моделей

```python
class OrderProjection:
    """Проекция для синхронизации Read Model"""

    def __init__(self, read_db: AsyncConnection):
        self.read_db = read_db

    async def handle_order_created(self, event: OrderCreatedEvent) -> None:
        await self.read_db.execute(
            """
            INSERT INTO order_summaries
            (id, customer_id, customer_name, customer_email,
             items_count, total_amount, status, created_at)
            VALUES (:id, :customer_id, :customer_name, :customer_email,
                    :items_count, :total_amount, :status, :created_at)
            """,
            {
                "id": event.order_id,
                "customer_id": event.customer_id,
                "customer_name": event.customer_name,
                "customer_email": event.customer_email,
                "items_count": 0,
                "total_amount": Decimal("0"),
                "status": "created",
                "created_at": event.timestamp
            }
        )

    async def handle_item_added(self, event: ItemAddedEvent) -> None:
        await self.read_db.execute(
            """
            UPDATE order_summaries SET
                items_count = items_count + 1,
                total_amount = total_amount + :item_price
            WHERE id = :order_id
            """,
            {"order_id": event.order_id, "item_price": event.item_price}
        )
```

---

## Когда использовать

### Идеальные сценарии

1. **Высокая нагрузка на чтение** — когда операций чтения значительно больше, чем записи (соотношение 100:1 и выше)

2. **Сложные запросы** — когда для отображения данных требуются сложные агрегации и JOIN

3. **Разные модели представления** — когда одни и те же данные нужно показывать по-разному

4. **Масштабирование** — когда нужно независимо масштабировать чтение и запись

5. **Совместно с Event Sourcing** — CQRS естественно дополняет Event Sourcing

```python
# Пример: аналитическая система интернет-магазина
class AnalyticsDashboardQuery:
    """Сложные аналитические запросы - отдельная Read Model"""

    async def get_sales_summary(
        self,
        start_date: date,
        end_date: date
    ) -> SalesSummary:
        # Запрос к предварительно агрегированной таблице
        return await self.read_db.fetch_one(
            """
            SELECT
                date,
                total_orders,
                total_revenue,
                average_order_value,
                top_products
            FROM daily_sales_summary
            WHERE date BETWEEN :start AND :end
            """,
            {"start": start_date, "end": end_date}
        )
```

### Когда НЕ использовать

- Простые CRUD-приложения
- Маленькие проекты с низкой нагрузкой
- Когда eventual consistency неприемлема
- При отсутствии опыта работы с распределёнными системами

---

## Преимущества и недостатки

### Преимущества

| Преимущество | Описание |
|-------------|----------|
| **Оптимизация производительности** | Каждая модель оптимизирована под свои задачи |
| **Независимое масштабирование** | Read и Write модели масштабируются отдельно |
| **Упрощение моделей** | Каждая модель проще, чем универсальная |
| **Гибкость хранения** | Разные БД для разных задач (SQL для записи, NoSQL для чтения) |
| **Безопасность** | Легче контролировать доступ к командам и запросам |

### Недостатки

| Недостаток | Описание |
|-----------|----------|
| **Сложность** | Увеличивает общую сложность системы |
| **Eventual Consistency** | Данные в Read Model могут отставать |
| **Дублирование данных** | Те же данные хранятся в разных формах |
| **Синхронизация** | Требуется механизм синхронизации моделей |
| **Отладка** | Сложнее отслеживать проблемы |

---

## Примеры реализации

### Полная реализация CQRS

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Generic, TypeVar
from uuid import uuid4

# === Commands ===

@dataclass
class Command(ABC):
    """Базовый класс для команд"""
    correlation_id: str = None

    def __post_init__(self):
        if self.correlation_id is None:
            self.correlation_id = str(uuid4())

@dataclass
class CreateOrderCommand(Command):
    customer_id: str
    items: List[dict]

@dataclass
class AddItemCommand(Command):
    order_id: str
    product_id: str
    quantity: int

# === Command Handlers ===

class CommandHandler(ABC, Generic[TypeVar("C", bound=Command)]):
    @abstractmethod
    async def handle(self, command: C) -> None:
        pass

class CreateOrderHandler(CommandHandler[CreateOrderCommand]):
    def __init__(
        self,
        write_repo: WriteRepository,
        event_bus: EventBus
    ):
        self.write_repo = write_repo
        self.event_bus = event_bus

    async def handle(self, command: CreateOrderCommand) -> str:
        # Создаём доменную модель
        order = Order.create(
            customer_id=command.customer_id,
            items=command.items
        )

        # Сохраняем в Write Database
        await self.write_repo.save(order)

        # Публикуем события для обновления Read Model
        await self.event_bus.publish(
            OrderCreatedEvent(
                order_id=order.id,
                customer_id=order.customer_id,
                items=order.items,
                timestamp=datetime.utcnow()
            )
        )

        return order.id

# === Queries ===

@dataclass
class Query(ABC):
    """Базовый класс для запросов"""
    pass

@dataclass
class GetOrderQuery(Query):
    order_id: str

@dataclass
class GetCustomerOrdersQuery(Query):
    customer_id: str
    page: int = 0
    size: int = 20

# === Query Handlers ===

R = TypeVar("R")

class QueryHandler(ABC, Generic[TypeVar("Q", bound=Query), R]):
    @abstractmethod
    async def handle(self, query: Q) -> R:
        pass

class GetOrderHandler(QueryHandler[GetOrderQuery, OrderReadModel]):
    def __init__(self, read_repo: ReadRepository):
        self.read_repo = read_repo

    async def handle(self, query: GetOrderQuery) -> OrderReadModel:
        return await self.read_repo.get_order_summary(query.order_id)

class GetCustomerOrdersHandler(
    QueryHandler[GetCustomerOrdersQuery, List[OrderReadModel]]
):
    def __init__(self, read_repo: ReadRepository):
        self.read_repo = read_repo

    async def handle(
        self,
        query: GetCustomerOrdersQuery
    ) -> List[OrderReadModel]:
        return await self.read_repo.get_customer_orders(
            query.customer_id,
            query.page,
            query.size
        )

# === Mediator (Command/Query Bus) ===

class Mediator:
    def __init__(self):
        self._command_handlers: Dict[Type[Command], CommandHandler] = {}
        self._query_handlers: Dict[Type[Query], QueryHandler] = {}

    def register_command_handler(
        self,
        command_type: Type[Command],
        handler: CommandHandler
    ) -> None:
        self._command_handlers[command_type] = handler

    def register_query_handler(
        self,
        query_type: Type[Query],
        handler: QueryHandler
    ) -> None:
        self._query_handlers[query_type] = handler

    async def send(self, command: Command) -> Any:
        handler = self._command_handlers.get(type(command))
        if not handler:
            raise ValueError(f"No handler for {type(command)}")
        return await handler.handle(command)

    async def query(self, query: Query) -> Any:
        handler = self._query_handlers.get(type(query))
        if not handler:
            raise ValueError(f"No handler for {type(query)}")
        return await handler.handle(query)

# === Использование ===

async def main():
    mediator = Mediator()

    # Регистрируем обработчики
    mediator.register_command_handler(
        CreateOrderCommand,
        CreateOrderHandler(write_repo, event_bus)
    )
    mediator.register_query_handler(
        GetOrderQuery,
        GetOrderHandler(read_repo)
    )

    # Создаём заказ (Command)
    order_id = await mediator.send(
        CreateOrderCommand(
            customer_id="cust-123",
            items=[{"product_id": "prod-1", "quantity": 2}]
        )
    )

    # Получаем заказ (Query)
    order = await mediator.query(GetOrderQuery(order_id=order_id))
```

### Архитектура с разными базами данных

```
┌─────────────────────────────────────────────────────────────────────┐
│                           API Gateway                                │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                ┌───────────────┴───────────────┐
                │                               │
                ▼                               ▼
┌───────────────────────────┐   ┌───────────────────────────┐
│      Command Service      │   │       Query Service       │
│  ┌─────────────────────┐  │   │  ┌─────────────────────┐  │
│  │  Command Handlers   │  │   │  │   Query Handlers    │  │
│  └──────────┬──────────┘  │   │  └──────────┬──────────┘  │
│             │             │   │             │             │
│  ┌──────────▼──────────┐  │   │  ┌──────────▼──────────┐  │
│  │   Domain Model      │  │   │  │    Read Models      │  │
│  └──────────┬──────────┘  │   │  └──────────┬──────────┘  │
└─────────────┼─────────────┘   └─────────────┼─────────────┘
              │                               │
              ▼                               ▼
┌───────────────────────┐       ┌───────────────────────────┐
│   PostgreSQL (Write)  │       │   Elasticsearch (Read)    │
│   - Normalized data   │       │   - Denormalized views    │
│   - ACID transactions │       │   - Full-text search      │
│   - Referential       │       │   - Fast aggregations     │
│     integrity         │       │                           │
└───────────┬───────────┘       └───────────────────────────┘
            │                               ▲
            │     ┌─────────────────┐       │
            └────►│  Event Bus      │───────┘
                  │  (Kafka/RabbitMQ)│
                  │  ┌───────────┐  │
                  │  │ Projector │  │
                  │  └───────────┘  │
                  └─────────────────┘
```

---

## Best Practices и антипаттерны

### Best Practices

```python
# 1. Идемпотентные проекции
class OrderProjection:
    async def handle_order_created(self, event: OrderCreatedEvent) -> None:
        # Используем upsert вместо insert
        await self.db.execute(
            """
            INSERT INTO order_read_models (id, ...)
            VALUES (:id, ...)
            ON CONFLICT (id) DO UPDATE SET ...
            WHERE event_version < :event_version
            """,
            {"id": event.order_id, "event_version": event.version}
        )

# 2. Версионирование для защиты от race conditions
@dataclass
class OrderReadModel:
    id: str
    event_version: int  # Версия последнего применённого события

# 3. Отслеживание задержки синхронизации
class ProjectionLagMonitor:
    async def check_lag(self) -> timedelta:
        write_latest = await self.write_db.get_latest_event_time()
        read_latest = await self.read_db.get_latest_projection_time()
        return write_latest - read_latest

# 4. Read-your-writes consistency при необходимости
class OrderService:
    async def create_and_get_order(
        self,
        command: CreateOrderCommand
    ) -> OrderReadModel:
        order_id = await self.mediator.send(command)

        # Ждём синхронизации или читаем из Write Model
        for _ in range(10):
            try:
                return await self.mediator.query(GetOrderQuery(order_id))
            except NotFoundError:
                await asyncio.sleep(0.1)

        # Fallback: читаем напрямую из Write Model
        return await self.read_from_write_model(order_id)
```

### Антипаттерны

```python
# ❌ Антипаттерн: Команды возвращают данные
class CreateOrderHandler:
    async def handle(self, command: CreateOrderCommand) -> OrderReadModel:
        order = Order.create(...)
        await self.write_repo.save(order)
        # Плохо: команда не должна возвращать Read Model
        return await self.read_repo.get_order(order.id)

# ✅ Правильно: Команда возвращает только ID
class CreateOrderHandler:
    async def handle(self, command: CreateOrderCommand) -> str:
        order = Order.create(...)
        await self.write_repo.save(order)
        return order.id  # Только идентификатор

# ❌ Антипаттерн: Запросы изменяют данные
class GetOrderHandler:
    async def handle(self, query: GetOrderQuery) -> OrderReadModel:
        order = await self.read_repo.get_order(query.order_id)
        # Плохо: запрос изменяет данные
        await self.read_repo.increment_view_count(query.order_id)
        return order

# ❌ Антипаттерн: Прямая связь между Write и Read
class OrderProjection:
    async def project(self) -> None:
        # Плохо: синхронный вызов Write репозитория
        order = await self.write_repo.get_order(order_id)
        await self.read_repo.save(self._to_read_model(order))

# ✅ Правильно: Проекция через события
class OrderProjection:
    async def handle_event(self, event: OrderCreatedEvent) -> None:
        # Хорошо: проекция из события
        read_model = self._create_from_event(event)
        await self.read_repo.save(read_model)
```

---

## Связанные паттерны

| Паттерн | Связь с CQRS |
|---------|-------------|
| **Event Sourcing** | Часто используются вместе; события ES используются для построения Read Model |
| **Domain-Driven Design** | CQRS хорошо сочетается с агрегатами и доменными событиями |
| **Saga** | Координация распределённых транзакций в CQRS-системах |
| **Mediator** | Используется для маршрутизации команд и запросов |
| **Repository** | Разделяется на Write Repository и Read Repository |

---

## Ресурсы для изучения

### Книги
- "Implementing Domain-Driven Design" — Vaughn Vernon
- "Patterns, Principles, and Practices of Domain-Driven Design" — Scott Millett

### Статьи
- [CQRS by Martin Fowler](https://martinfowler.com/bliki/CQRS.html)
- [Microsoft CQRS Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs)

### Библиотеки Python
- `eventsourcing` — библиотека для Event Sourcing и CQRS
- `mediatR` (порты для Python) — реализация паттерна Mediator

### Видео
- "CQRS and Event Sourcing" — Greg Young (автор термина CQRS)
- "A Decade of DDD, CQRS, Event Sourcing" — Greg Young
