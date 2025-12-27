# CQRS (Command Query Responsibility Segregation)

## Что такое CQRS?

CQRS (Command Query Responsibility Segregation) — это архитектурный паттерн, который разделяет операции чтения (Query) и записи (Command) данных на отдельные модели. Вместо единой модели для всех операций используются две: одна оптимизирована для записи, другая — для чтения.

```
┌─────────────────────────────────────────────────────────────────┐
│                         CQRS Pattern                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Традиционный CRUD:                                            │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                    Single Model                          │  │
│   │              ┌─────────────────────┐                    │  │
│   │   Create ───►│                     │                    │  │
│   │   Read ─────►│      Domain         │                    │  │
│   │   Update ───►│      Model          │                    │  │
│   │   Delete ───►│                     │                    │  │
│   │              └─────────────────────┘                    │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   CQRS:                                                         │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                                                          │  │
│   │   Commands ──►┌─────────────┐     ┌─────────────┐◄── Queries
│   │   (Create,    │   Write     │ ──► │    Read     │     (Get,  │
│   │    Update,    │   Model     │     │    Model    │     List)  │
│   │    Delete)    └─────────────┘     └─────────────┘           │
│   │                      │                   ▲                   │
│   │                      └───────────────────┘                   │
│   │                        Синхронизация                        │
│   └─────────────────────────────────────────────────────────────┘
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Зачем нужен CQRS?

### Проблемы традиционного CRUD

```python
# Традиционная модель — компромисс между чтением и записью

class Order:
    """
    Единая модель для всех операций.
    Проблемы:
    - Сложные запросы для отчётов требуют JOIN-ов
    - Денормализация для чтения усложняет запись
    - Валидация и бизнес-правила смешаны с отображением
    """

    def __init__(self, order_id: str):
        self.order_id = order_id
        self.customer_id = ""
        self.customer_name = ""  # Денормализация для отображения
        self.items = []
        self.status = "pending"
        self.total = 0.0
        # Множество полей для разных случаев использования...

    def to_list_view(self) -> dict:
        """Для списка заказов"""
        return {
            "order_id": self.order_id,
            "customer_name": self.customer_name,
            "total": self.total,
            "status": self.status
        }

    def to_detail_view(self) -> dict:
        """Для детальной страницы"""
        return {
            "order_id": self.order_id,
            "customer": self._get_full_customer_info(),
            "items": self._get_items_with_products(),
            "shipping": self._get_shipping_info(),
            # Много JOIN-ов для полной информации
        }

    def to_report_view(self) -> dict:
        """Для отчётов"""
        # Ещё больше агрегаций и JOIN-ов
        pass
```

### Решение с CQRS

```python
# Разделение на Commands и Queries

# ===== COMMAND SIDE (Write Model) =====

from dataclasses import dataclass
from typing import List
from abc import ABC, abstractmethod


@dataclass
class Command(ABC):
    """Базовый класс для команд"""
    pass


@dataclass
class CreateOrderCommand(Command):
    """Команда создания заказа"""
    customer_id: str
    items: List[dict]
    shipping_address: dict


@dataclass
class AddOrderItemCommand(Command):
    """Команда добавления товара"""
    order_id: str
    product_id: str
    quantity: int


@dataclass
class CancelOrderCommand(Command):
    """Команда отмены заказа"""
    order_id: str
    reason: str


class CommandHandler(ABC):
    """Базовый обработчик команд"""

    @abstractmethod
    def handle(self, command: Command) -> None:
        pass


class CreateOrderHandler(CommandHandler):
    """Обработчик создания заказа"""

    def __init__(self, order_repository, event_publisher):
        self._repository = order_repository
        self._publisher = event_publisher

    def handle(self, command: CreateOrderCommand) -> str:
        # Создаём агрегат (Write Model)
        order = Order.create(
            customer_id=command.customer_id,
            items=command.items,
            shipping_address=command.shipping_address
        )

        # Валидация бизнес-правил
        order.validate()

        # Сохраняем
        self._repository.save(order)

        # Публикуем события для обновления Read Model
        for event in order.get_events():
            self._publisher.publish(event)

        return order.order_id


# Write Model — оптимизирована для бизнес-логики
class Order:
    """Write Model для заказа"""

    def __init__(self):
        self.order_id = ""
        self.customer_id = ""
        self.items = []
        self.status = "draft"
        self._events = []

    @classmethod
    def create(cls, customer_id: str, items: list,
               shipping_address: dict) -> 'Order':
        order = cls()
        order.order_id = str(uuid.uuid4())
        order.customer_id = customer_id
        order.items = items
        order.shipping_address = shipping_address
        order.status = "pending"

        order._events.append(OrderCreatedEvent(
            order_id=order.order_id,
            customer_id=customer_id,
            items=items,
            total=order._calculate_total()
        ))

        return order

    def add_item(self, product_id: str, quantity: int, price: float):
        """Бизнес-логика добавления товара"""
        if self.status != "draft":
            raise ValueError("Cannot modify non-draft order")

        self.items.append({
            "product_id": product_id,
            "quantity": quantity,
            "price": price
        })

        self._events.append(OrderItemAddedEvent(
            order_id=self.order_id,
            product_id=product_id,
            quantity=quantity
        ))

    def cancel(self, reason: str):
        """Бизнес-логика отмены"""
        if self.status in ["shipped", "delivered"]:
            raise ValueError("Cannot cancel shipped order")

        self.status = "cancelled"
        self._events.append(OrderCancelledEvent(
            order_id=self.order_id,
            reason=reason
        ))

    def validate(self):
        """Валидация бизнес-правил"""
        if not self.items:
            raise ValueError("Order must have at least one item")
        if not self.customer_id:
            raise ValueError("Customer is required")

    def get_events(self) -> list:
        return self._events
```

## Read Model (Query Side)

```python
# ===== QUERY SIDE (Read Model) =====

from dataclasses import dataclass
from typing import Optional, List


@dataclass
class Query(ABC):
    """Базовый класс для запросов"""
    pass


@dataclass
class GetOrderByIdQuery(Query):
    """Запрос заказа по ID"""
    order_id: str


@dataclass
class GetCustomerOrdersQuery(Query):
    """Запрос заказов клиента"""
    customer_id: str
    status: Optional[str] = None
    limit: int = 20
    offset: int = 0


@dataclass
class GetOrderStatisticsQuery(Query):
    """Запрос статистики заказов"""
    start_date: str
    end_date: str


# Read Models — оптимизированы для запросов
@dataclass
class OrderListItem:
    """Read Model для списка заказов"""
    order_id: str
    customer_name: str
    items_count: int
    total: float
    status: str
    created_at: str


@dataclass
class OrderDetails:
    """Read Model для деталей заказа"""
    order_id: str
    customer: dict
    items: List[dict]
    shipping_address: dict
    billing_address: dict
    total: float
    subtotal: float
    tax: float
    status: str
    status_history: List[dict]
    created_at: str
    updated_at: str


@dataclass
class OrderStatistics:
    """Read Model для статистики"""
    total_orders: int
    total_revenue: float
    average_order_value: float
    orders_by_status: dict
    top_products: List[dict]


class QueryHandler(ABC):
    """Базовый обработчик запросов"""

    @abstractmethod
    def handle(self, query: Query):
        pass


class GetOrderByIdHandler(QueryHandler):
    """Обработчик запроса заказа"""

    def __init__(self, read_repository):
        self._repository = read_repository

    def handle(self, query: GetOrderByIdQuery) -> Optional[OrderDetails]:
        # Читаем из Read Model (денормализованные данные)
        return self._repository.get_order_details(query.order_id)


class GetCustomerOrdersHandler(QueryHandler):
    """Обработчик запроса заказов клиента"""

    def __init__(self, read_repository):
        self._repository = read_repository

    def handle(self, query: GetCustomerOrdersQuery) -> List[OrderListItem]:
        return self._repository.get_customer_orders(
            customer_id=query.customer_id,
            status=query.status,
            limit=query.limit,
            offset=query.offset
        )


class GetOrderStatisticsHandler(QueryHandler):
    """Обработчик запроса статистики"""

    def __init__(self, read_repository):
        self._repository = read_repository

    def handle(self, query: GetOrderStatisticsQuery) -> OrderStatistics:
        return self._repository.get_statistics(
            start_date=query.start_date,
            end_date=query.end_date
        )
```

## Синхронизация моделей

```python
# Синхронизация Write и Read моделей через события

class ReadModelUpdater:
    """
    Обновляет Read Model на основе событий.
    Слушает события от Write Model.
    """

    def __init__(self, read_db):
        self._db = read_db

    def handle_order_created(self, event: OrderCreatedEvent):
        """Обработка события создания заказа"""
        # Денормализуем данные для быстрого чтения
        customer = self._fetch_customer(event.customer_id)

        order_view = {
            "order_id": event.order_id,
            "customer_id": event.customer_id,
            "customer_name": customer["name"],
            "customer_email": customer["email"],
            "items": self._enrich_items(event.items),
            "items_count": len(event.items),
            "total": event.total,
            "status": "pending",
            "created_at": event.timestamp,
            "updated_at": event.timestamp
        }

        self._db.orders.insert_one(order_view)

        # Обновляем агрегаты для статистики
        self._update_statistics(event)

    def handle_order_item_added(self, event: OrderItemAddedEvent):
        """Обработка добавления товара"""
        product = self._fetch_product(event.product_id)

        self._db.orders.update_one(
            {"order_id": event.order_id},
            {
                "$push": {
                    "items": {
                        "product_id": event.product_id,
                        "product_name": product["name"],
                        "quantity": event.quantity,
                        "price": product["price"]
                    }
                },
                "$inc": {
                    "items_count": 1,
                    "total": product["price"] * event.quantity
                },
                "$set": {"updated_at": event.timestamp}
            }
        )

    def handle_order_cancelled(self, event: OrderCancelledEvent):
        """Обработка отмены заказа"""
        self._db.orders.update_one(
            {"order_id": event.order_id},
            {
                "$set": {
                    "status": "cancelled",
                    "cancellation_reason": event.reason,
                    "updated_at": event.timestamp
                },
                "$push": {
                    "status_history": {
                        "status": "cancelled",
                        "timestamp": event.timestamp,
                        "reason": event.reason
                    }
                }
            }
        )

        # Обновляем статистику
        self._update_statistics_on_cancel(event)

    def _enrich_items(self, items: list) -> list:
        """Обогащаем данные товаров"""
        enriched = []
        for item in items:
            product = self._fetch_product(item["product_id"])
            enriched.append({
                **item,
                "product_name": product["name"],
                "product_image": product["image_url"]
            })
        return enriched

    def _update_statistics(self, event: OrderCreatedEvent):
        """Обновление агрегированной статистики"""
        date = event.timestamp.strftime("%Y-%m-%d")

        self._db.statistics.update_one(
            {"date": date},
            {
                "$inc": {
                    "orders_count": 1,
                    "total_revenue": event.total
                }
            },
            upsert=True
        )
```

## Разные хранилища для Read и Write

```python
# Архитектура с разными базами данных

class WriteRepository:
    """Репозиторий для Write Model (PostgreSQL)"""

    def __init__(self, connection):
        self._conn = connection

    def save(self, order: Order) -> None:
        """Сохранение в нормализованную БД"""
        with self._conn.cursor() as cur:
            # Сохраняем заказ
            cur.execute(
                """INSERT INTO orders (order_id, customer_id, status, created_at)
                   VALUES (%s, %s, %s, %s)
                   ON CONFLICT (order_id) DO UPDATE
                   SET status = EXCLUDED.status""",
                (order.order_id, order.customer_id, order.status, datetime.now())
            )

            # Сохраняем позиции
            for item in order.items:
                cur.execute(
                    """INSERT INTO order_items (order_id, product_id, quantity, price)
                       VALUES (%s, %s, %s, %s)""",
                    (order.order_id, item["product_id"],
                     item["quantity"], item["price"])
                )

            self._conn.commit()

    def get(self, order_id: str) -> Optional[Order]:
        """Загрузка для обработки команд"""
        with self._conn.cursor() as cur:
            cur.execute(
                "SELECT * FROM orders WHERE order_id = %s",
                (order_id,)
            )
            row = cur.fetchone()

            if not row:
                return None

            order = Order()
            order.order_id = row["order_id"]
            order.customer_id = row["customer_id"]
            order.status = row["status"]

            cur.execute(
                "SELECT * FROM order_items WHERE order_id = %s",
                (order_id,)
            )
            order.items = cur.fetchall()

            return order


class ReadRepository:
    """Репозиторий для Read Model (MongoDB)"""

    def __init__(self, database):
        self._db = database

    def get_order_details(self, order_id: str) -> Optional[OrderDetails]:
        """Получение денормализованных данных"""
        doc = self._db.orders.find_one({"order_id": order_id})

        if not doc:
            return None

        return OrderDetails(
            order_id=doc["order_id"],
            customer=doc["customer"],
            items=doc["items"],
            shipping_address=doc["shipping_address"],
            billing_address=doc.get("billing_address"),
            total=doc["total"],
            subtotal=doc["subtotal"],
            tax=doc["tax"],
            status=doc["status"],
            status_history=doc.get("status_history", []),
            created_at=doc["created_at"],
            updated_at=doc["updated_at"]
        )

    def get_customer_orders(self, customer_id: str,
                            status: str = None,
                            limit: int = 20,
                            offset: int = 0) -> List[OrderListItem]:
        """Быстрый запрос списка заказов"""
        query = {"customer_id": customer_id}
        if status:
            query["status"] = status

        cursor = self._db.orders.find(query)\
            .sort("created_at", -1)\
            .skip(offset)\
            .limit(limit)

        return [
            OrderListItem(
                order_id=doc["order_id"],
                customer_name=doc["customer_name"],
                items_count=doc["items_count"],
                total=doc["total"],
                status=doc["status"],
                created_at=doc["created_at"]
            )
            for doc in cursor
        ]

    def get_statistics(self, start_date: str, end_date: str) -> OrderStatistics:
        """Агрегированная статистика"""
        pipeline = [
            {
                "$match": {
                    "created_at": {"$gte": start_date, "$lte": end_date}
                }
            },
            {
                "$group": {
                    "_id": None,
                    "total_orders": {"$sum": 1},
                    "total_revenue": {"$sum": "$total"},
                    "avg_order_value": {"$avg": "$total"}
                }
            }
        ]

        result = list(self._db.orders.aggregate(pipeline))

        if not result:
            return OrderStatistics(0, 0.0, 0.0, {}, [])

        stats = result[0]
        return OrderStatistics(
            total_orders=stats["total_orders"],
            total_revenue=stats["total_revenue"],
            average_order_value=stats["avg_order_value"],
            orders_by_status=self._get_orders_by_status(start_date, end_date),
            top_products=self._get_top_products(start_date, end_date)
        )
```

## CQRS с Event Sourcing

```python
# CQRS часто используется вместе с Event Sourcing

class EventSourcedOrder:
    """Write Model с Event Sourcing"""

    def __init__(self):
        self.order_id = ""
        self.items = []
        self.status = "draft"
        self._version = 0
        self._events = []

    def apply(self, event: Event):
        """Применить событие к состоянию"""
        handler = getattr(self, f"_apply_{event.event_type}", None)
        if handler:
            handler(event)
        self._version += 1

    def _apply_order_created(self, event):
        self.order_id = event.order_id
        self.customer_id = event.customer_id
        self.status = "pending"

    def _apply_item_added(self, event):
        self.items.append({
            "product_id": event.product_id,
            "quantity": event.quantity
        })

    def _apply_order_cancelled(self, event):
        self.status = "cancelled"


class CQRSWithEventSourcing:
    """
    Полная архитектура CQRS + Event Sourcing.
    """

    def __init__(self, event_store, read_db):
        self.event_store = event_store
        self.read_db = read_db
        self.projections = []

    def execute_command(self, command: Command):
        """Выполнение команды"""
        # Загружаем агрегат из Event Store
        events = self.event_store.get_events(command.aggregate_id)
        aggregate = self._rebuild_aggregate(events)

        # Выполняем команду
        new_events = aggregate.execute(command)

        # Сохраняем новые события
        self.event_store.append(command.aggregate_id, new_events)

        # Обновляем Read Model
        for event in new_events:
            self._update_projections(event)

    def execute_query(self, query: Query):
        """Выполнение запроса"""
        # Читаем напрямую из Read Model
        handler = self._get_query_handler(query)
        return handler.handle(query)

    def _rebuild_aggregate(self, events: list):
        """Восстановление агрегата из событий"""
        aggregate = EventSourcedOrder()
        for event in events:
            aggregate.apply(event)
        return aggregate

    def _update_projections(self, event: Event):
        """Обновление всех проекций"""
        for projection in self.projections:
            if event.event_type in projection.handled_events:
                projection.handle(event)
```

## Архитектурная диаграмма

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CQRS Architecture                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│                              UI / API                                    │
│                     ┌───────────┴───────────┐                           │
│                     │                       │                           │
│                     ▼                       ▼                           │
│              ┌─────────────┐         ┌─────────────┐                   │
│              │  Commands   │         │   Queries   │                   │
│              │   (Write)   │         │   (Read)    │                   │
│              └──────┬──────┘         └──────┬──────┘                   │
│                     │                       │                           │
│                     ▼                       ▼                           │
│              ┌─────────────┐         ┌─────────────┐                   │
│              │  Command    │         │   Query     │                   │
│              │  Handlers   │         │  Handlers   │                   │
│              └──────┬──────┘         └──────┬──────┘                   │
│                     │                       │                           │
│                     ▼                       ▼                           │
│              ┌─────────────┐         ┌─────────────┐                   │
│              │   Write     │         │    Read     │                   │
│              │   Model     │         │    Model    │                   │
│              │ (Aggregates)│         │ (Projections)                   │
│              └──────┬──────┘         └──────┬──────┘                   │
│                     │                       ▲                           │
│                     ▼                       │                           │
│              ┌─────────────┐         ┌──────┴──────┐                   │
│              │   Write     │────────►│   Event     │                   │
│              │   Store     │  Events │   Handler   │                   │
│              │ (PostgreSQL)│         │             │                   │
│              └─────────────┘         └──────┬──────┘                   │
│                                             │                           │
│                                             ▼                           │
│                                      ┌─────────────┐                   │
│                                      │    Read     │                   │
│                                      │    Store    │                   │
│                                      │  (MongoDB)  │                   │
│                                      └─────────────┘                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## Плюсы и минусы CQRS

### Плюсы

1. **Оптимизация** — каждая модель оптимизирована для своей задачи
2. **Масштабируемость** — Read и Write можно масштабировать независимо
3. **Простота** — модели проще, так как решают одну задачу
4. **Гибкость** — разные хранилища для разных моделей
5. **Производительность** — быстрые чтения из денормализованных данных

### Минусы

1. **Сложность** — больше кода и инфраструктуры
2. **Eventual Consistency** — Read Model может отставать
3. **Дублирование** — данные дублируются между моделями
4. **Синхронизация** — нужно поддерживать согласованность моделей
5. **Overhead** — избыточен для простых CRUD приложений

## Когда использовать CQRS

### Используйте CQRS когда:

- Чтение и запись имеют разные требования к производительности
- Сложные запросы для отчётов и аналитики
- Высокая нагрузка на чтение (read-heavy системы)
- Нужны разные представления одних данных
- Используется Event Sourcing

### Не используйте CQRS когда:

- Простое CRUD приложение
- Низкая нагрузка
- Строгие требования к консистентности
- Маленькая команда без опыта
- Нет явного разделения требований read/write

## Заключение

CQRS — мощный паттерн для систем с разными требованиями к чтению и записи. Он особенно эффективен в сочетании с Event Sourcing и микросервисами. Однако CQRS добавляет сложность, поэтому его стоит применять только когда преимущества перевешивают затраты на реализацию.
