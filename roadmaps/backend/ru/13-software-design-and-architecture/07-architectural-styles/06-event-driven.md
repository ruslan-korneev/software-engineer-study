# Event-Driven Architecture

## Что такое событийно-ориентированная архитектура?

Событийно-ориентированная архитектура (Event-Driven Architecture, EDA) — это архитектурный паттерн, в котором компоненты системы взаимодействуют через асинхронную передачу событий. Вместо прямых вызовов между компонентами, производители генерируют события, а потребители реагируют на них.

> "В событийно-ориентированной архитектуре всё — это событие. Событие — это факт, который произошёл в системе."

## Основные концепции

```
┌─────────────────────────────────────────────────────────────┐
│                   Event-Driven System                        │
│                                                              │
│  ┌──────────┐    ┌─────────────────┐    ┌──────────────┐    │
│  │ Producer │───►│   Event Bus     │───►│   Consumer   │    │
│  │          │    │   / Broker      │    │              │    │
│  └──────────┘    └─────────────────┘    └──────────────┘    │
│                          │                                   │
│                          │                                   │
│                          ▼                                   │
│                  ┌──────────────┐                            │
│                  │   Consumer   │                            │
│                  │     #2       │                            │
│                  └──────────────┘                            │
└─────────────────────────────────────────────────────────────┘
```

### Ключевые компоненты

1. **Event (Событие)** — неизменяемый факт о том, что произошло
2. **Producer (Производитель)** — компонент, который генерирует события
3. **Consumer (Потребитель)** — компонент, который обрабатывает события
4. **Event Channel (Канал событий)** — механизм передачи событий

## Типы событий

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Any, Dict, Optional
from enum import Enum
import uuid

class EventType(Enum):
    """Типы событий."""
    # Domain Events
    USER_CREATED = "user.created"
    USER_UPDATED = "user.updated"
    ORDER_PLACED = "order.placed"
    ORDER_SHIPPED = "order.shipped"
    PAYMENT_RECEIVED = "payment.received"

    # Integration Events
    EMAIL_SENT = "notification.email.sent"
    SMS_SENT = "notification.sms.sent"

    # System Events
    SERVICE_STARTED = "system.service.started"
    ERROR_OCCURRED = "system.error.occurred"


@dataclass
class Event:
    """Базовый класс события."""
    event_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    event_type: str = ""
    timestamp: datetime = field(default_factory=datetime.utcnow)
    version: str = "1.0"
    source: str = ""  # Источник события
    correlation_id: Optional[str] = None  # Для отслеживания цепочки
    causation_id: Optional[str] = None  # ID события-причины
    metadata: Dict[str, Any] = field(default_factory=dict)
    data: Dict[str, Any] = field(default_factory=dict)

    def to_dict(self) -> Dict[str, Any]:
        return {
            "event_id": self.event_id,
            "event_type": self.event_type,
            "timestamp": self.timestamp.isoformat(),
            "version": self.version,
            "source": self.source,
            "correlation_id": self.correlation_id,
            "causation_id": self.causation_id,
            "metadata": self.metadata,
            "data": self.data
        }


# === Конкретные события ===

@dataclass
class UserCreatedEvent(Event):
    """Событие создания пользователя."""

    def __init__(self, user_id: str, email: str, name: str, **kwargs):
        super().__init__(
            event_type=EventType.USER_CREATED.value,
            data={
                "user_id": user_id,
                "email": email,
                "name": name
            },
            **kwargs
        )


@dataclass
class OrderPlacedEvent(Event):
    """Событие создания заказа."""

    def __init__(
        self,
        order_id: str,
        user_id: str,
        items: list,
        total_amount: float,
        **kwargs
    ):
        super().__init__(
            event_type=EventType.ORDER_PLACED.value,
            data={
                "order_id": order_id,
                "user_id": user_id,
                "items": items,
                "total_amount": total_amount
            },
            **kwargs
        )


@dataclass
class PaymentReceivedEvent(Event):
    """Событие получения платежа."""

    def __init__(
        self,
        payment_id: str,
        order_id: str,
        amount: float,
        method: str,
        **kwargs
    ):
        super().__init__(
            event_type=EventType.PAYMENT_RECEIVED.value,
            data={
                "payment_id": payment_id,
                "order_id": order_id,
                "amount": amount,
                "method": method
            },
            **kwargs
        )
```

## Event Bus (Шина событий)

```python
from typing import Callable, Dict, List, Set
from abc import ABC, abstractmethod
import asyncio
import logging

# Тип обработчика событий
EventHandler = Callable[[Event], None]
AsyncEventHandler = Callable[[Event], asyncio.coroutine]


class EventBus(ABC):
    """Абстрактная шина событий."""

    @abstractmethod
    def subscribe(self, event_type: str, handler: EventHandler) -> None:
        """Подписка на событие."""
        pass

    @abstractmethod
    def unsubscribe(self, event_type: str, handler: EventHandler) -> None:
        """Отписка от события."""
        pass

    @abstractmethod
    def publish(self, event: Event) -> None:
        """Публикация события."""
        pass


class InMemoryEventBus(EventBus):
    """Простая in-memory реализация шины событий."""

    def __init__(self):
        self._handlers: Dict[str, List[EventHandler]] = {}
        self._logger = logging.getLogger("event_bus")

    def subscribe(self, event_type: str, handler: EventHandler) -> None:
        if event_type not in self._handlers:
            self._handlers[event_type] = []

        if handler not in self._handlers[event_type]:
            self._handlers[event_type].append(handler)
            self._logger.info(f"Subscribed to {event_type}")

    def unsubscribe(self, event_type: str, handler: EventHandler) -> None:
        if event_type in self._handlers:
            self._handlers[event_type].remove(handler)

    def publish(self, event: Event) -> None:
        self._logger.info(f"Publishing event: {event.event_type}")

        handlers = self._handlers.get(event.event_type, [])
        handlers.extend(self._handlers.get("*", []))  # Wildcard handlers

        for handler in handlers:
            try:
                handler(event)
            except Exception as e:
                self._logger.error(f"Error in handler: {e}")


class AsyncEventBus(EventBus):
    """Асинхронная шина событий."""

    def __init__(self):
        self._handlers: Dict[str, List[AsyncEventHandler]] = {}
        self._queue: asyncio.Queue = asyncio.Queue()
        self._running = False
        self._logger = logging.getLogger("async_event_bus")

    def subscribe(self, event_type: str, handler: AsyncEventHandler) -> None:
        if event_type not in self._handlers:
            self._handlers[event_type] = []
        self._handlers[event_type].append(handler)

    def unsubscribe(self, event_type: str, handler: AsyncEventHandler) -> None:
        if event_type in self._handlers:
            self._handlers[event_type].remove(handler)

    async def publish(self, event: Event) -> None:
        await self._queue.put(event)

    async def start(self) -> None:
        """Запуск обработки событий."""
        self._running = True
        while self._running:
            try:
                event = await asyncio.wait_for(
                    self._queue.get(),
                    timeout=1.0
                )
                await self._dispatch(event)
            except asyncio.TimeoutError:
                continue

    async def stop(self) -> None:
        """Остановка обработки."""
        self._running = False

    async def _dispatch(self, event: Event) -> None:
        """Диспетчеризация события обработчикам."""
        handlers = self._handlers.get(event.event_type, [])

        tasks = [
            asyncio.create_task(handler(event))
            for handler in handlers
        ]

        if tasks:
            results = await asyncio.gather(*tasks, return_exceptions=True)
            for result in results:
                if isinstance(result, Exception):
                    self._logger.error(f"Handler error: {result}")
```

## Event Store (Хранилище событий)

```python
from typing import List, Optional, Iterator
from abc import ABC, abstractmethod
import json

class EventStore(ABC):
    """Абстрактное хранилище событий."""

    @abstractmethod
    def append(self, stream_id: str, event: Event) -> None:
        """Добавление события в поток."""
        pass

    @abstractmethod
    def get_events(
        self,
        stream_id: str,
        from_version: int = 0
    ) -> List[Event]:
        """Получение событий потока."""
        pass

    @abstractmethod
    def get_all_events(self, from_position: int = 0) -> Iterator[Event]:
        """Получение всех событий."""
        pass


class InMemoryEventStore(EventStore):
    """In-memory хранилище событий."""

    def __init__(self):
        self._streams: Dict[str, List[Event]] = {}
        self._all_events: List[Event] = []
        self._position = 0

    def append(self, stream_id: str, event: Event) -> None:
        if stream_id not in self._streams:
            self._streams[stream_id] = []

        self._streams[stream_id].append(event)
        self._all_events.append(event)
        self._position += 1

    def get_events(
        self,
        stream_id: str,
        from_version: int = 0
    ) -> List[Event]:
        events = self._streams.get(stream_id, [])
        return events[from_version:]

    def get_all_events(self, from_position: int = 0) -> Iterator[Event]:
        for event in self._all_events[from_position:]:
            yield event


class PostgresEventStore(EventStore):
    """PostgreSQL хранилище событий."""

    def __init__(self, connection_string: str):
        self.connection_string = connection_string
        # self.pool = create_pool(connection_string)

    async def append(self, stream_id: str, event: Event) -> None:
        query = """
            INSERT INTO events (
                event_id, stream_id, event_type, event_data,
                metadata, version, created_at
            ) VALUES ($1, $2, $3, $4, $5, $6, $7)
        """
        # async with self.pool.acquire() as conn:
        #     await conn.execute(query, ...)

    async def get_events(
        self,
        stream_id: str,
        from_version: int = 0
    ) -> List[Event]:
        query = """
            SELECT * FROM events
            WHERE stream_id = $1 AND version >= $2
            ORDER BY version ASC
        """
        # async with self.pool.acquire() as conn:
        #     rows = await conn.fetch(query, stream_id, from_version)
        return []

    async def get_all_events(self, from_position: int = 0) -> Iterator[Event]:
        query = """
            SELECT * FROM events
            WHERE global_position >= $1
            ORDER BY global_position ASC
        """
        # Streaming implementation
        pass
```

## Event Sourcing

```python
from typing import List, Type, TypeVar
from abc import ABC, abstractmethod

T = TypeVar('T', bound='AggregateRoot')


class AggregateRoot(ABC):
    """Базовый класс агрегата с Event Sourcing."""

    def __init__(self, aggregate_id: str):
        self.aggregate_id = aggregate_id
        self.version = 0
        self._uncommitted_events: List[Event] = []

    def apply_event(self, event: Event) -> None:
        """Применение события к агрегату."""
        self._apply(event)
        self.version += 1

    @abstractmethod
    def _apply(self, event: Event) -> None:
        """Внутренний метод применения события."""
        pass

    def raise_event(self, event: Event) -> None:
        """Генерация нового события."""
        self._uncommitted_events.append(event)
        self.apply_event(event)

    def get_uncommitted_events(self) -> List[Event]:
        """Получение незафиксированных событий."""
        return list(self._uncommitted_events)

    def clear_uncommitted_events(self) -> None:
        """Очистка незафиксированных событий."""
        self._uncommitted_events.clear()

    @classmethod
    def load_from_history(cls: Type[T], aggregate_id: str, events: List[Event]) -> T:
        """Восстановление агрегата из истории событий."""
        instance = cls(aggregate_id)
        for event in events:
            instance.apply_event(event)
        return instance


# === Пример: Order Aggregate ===

class OrderStatus(Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"


class Order(AggregateRoot):
    """Агрегат заказа с Event Sourcing."""

    def __init__(self, order_id: str):
        super().__init__(order_id)
        self.user_id: Optional[str] = None
        self.items: List[dict] = []
        self.total_amount: float = 0.0
        self.status: Optional[OrderStatus] = None

    def _apply(self, event: Event) -> None:
        """Применение события к состоянию заказа."""
        if event.event_type == EventType.ORDER_PLACED.value:
            self._apply_order_placed(event)
        elif event.event_type == "order.confirmed":
            self._apply_order_confirmed(event)
        elif event.event_type == "order.shipped":
            self._apply_order_shipped(event)
        elif event.event_type == "order.cancelled":
            self._apply_order_cancelled(event)

    def _apply_order_placed(self, event: Event) -> None:
        self.user_id = event.data["user_id"]
        self.items = event.data["items"]
        self.total_amount = event.data["total_amount"]
        self.status = OrderStatus.PENDING

    def _apply_order_confirmed(self, event: Event) -> None:
        self.status = OrderStatus.CONFIRMED

    def _apply_order_shipped(self, event: Event) -> None:
        self.status = OrderStatus.SHIPPED

    def _apply_order_cancelled(self, event: Event) -> None:
        self.status = OrderStatus.CANCELLED

    # === Команды (изменяют состояние через события) ===

    @classmethod
    def place(
        cls,
        order_id: str,
        user_id: str,
        items: List[dict],
        total_amount: float
    ) -> 'Order':
        """Создание нового заказа."""
        order = cls(order_id)

        event = OrderPlacedEvent(
            order_id=order_id,
            user_id=user_id,
            items=items,
            total_amount=total_amount
        )
        order.raise_event(event)

        return order

    def confirm(self) -> None:
        """Подтверждение заказа."""
        if self.status != OrderStatus.PENDING:
            raise ValueError("Order cannot be confirmed")

        event = Event(
            event_type="order.confirmed",
            data={"order_id": self.aggregate_id}
        )
        self.raise_event(event)

    def ship(self, tracking_number: str) -> None:
        """Отправка заказа."""
        if self.status != OrderStatus.CONFIRMED:
            raise ValueError("Order cannot be shipped")

        event = Event(
            event_type="order.shipped",
            data={
                "order_id": self.aggregate_id,
                "tracking_number": tracking_number
            }
        )
        self.raise_event(event)

    def cancel(self, reason: str) -> None:
        """Отмена заказа."""
        if self.status in [OrderStatus.SHIPPED, OrderStatus.DELIVERED]:
            raise ValueError("Order cannot be cancelled")

        event = Event(
            event_type="order.cancelled",
            data={
                "order_id": self.aggregate_id,
                "reason": reason
            }
        )
        self.raise_event(event)


# === Repository для агрегатов ===

class OrderRepository:
    """Репозиторий для работы с заказами через Event Store."""

    def __init__(self, event_store: EventStore, event_bus: EventBus):
        self.event_store = event_store
        self.event_bus = event_bus

    def save(self, order: Order) -> None:
        """Сохранение заказа."""
        stream_id = f"order-{order.aggregate_id}"

        for event in order.get_uncommitted_events():
            self.event_store.append(stream_id, event)
            self.event_bus.publish(event)

        order.clear_uncommitted_events()

    def get(self, order_id: str) -> Optional[Order]:
        """Получение заказа по ID."""
        stream_id = f"order-{order_id}"
        events = self.event_store.get_events(stream_id)

        if not events:
            return None

        return Order.load_from_history(order_id, events)
```

## Проекции (Read Models)

```python
from typing import Dict, List, Optional
from abc import ABC, abstractmethod


class Projection(ABC):
    """Базовый класс проекции."""

    @abstractmethod
    def handle(self, event: Event) -> None:
        """Обработка события."""
        pass


class OrderListProjection(Projection):
    """Проекция для списка заказов."""

    def __init__(self):
        self.orders: Dict[str, dict] = {}

    def handle(self, event: Event) -> None:
        if event.event_type == EventType.ORDER_PLACED.value:
            self._handle_order_placed(event)
        elif event.event_type == "order.shipped":
            self._handle_order_shipped(event)
        elif event.event_type == "order.cancelled":
            self._handle_order_cancelled(event)

    def _handle_order_placed(self, event: Event) -> None:
        order_id = event.data["order_id"]
        self.orders[order_id] = {
            "id": order_id,
            "user_id": event.data["user_id"],
            "total_amount": event.data["total_amount"],
            "items_count": len(event.data["items"]),
            "status": "pending",
            "created_at": event.timestamp
        }

    def _handle_order_shipped(self, event: Event) -> None:
        order_id = event.data["order_id"]
        if order_id in self.orders:
            self.orders[order_id]["status"] = "shipped"
            self.orders[order_id]["tracking_number"] = event.data.get("tracking_number")

    def _handle_order_cancelled(self, event: Event) -> None:
        order_id = event.data["order_id"]
        if order_id in self.orders:
            self.orders[order_id]["status"] = "cancelled"

    # === Query методы ===

    def get_order(self, order_id: str) -> Optional[dict]:
        return self.orders.get(order_id)

    def get_user_orders(self, user_id: str) -> List[dict]:
        return [
            order for order in self.orders.values()
            if order["user_id"] == user_id
        ]

    def get_pending_orders(self) -> List[dict]:
        return [
            order for order in self.orders.values()
            if order["status"] == "pending"
        ]


class OrderStatisticsProjection(Projection):
    """Проекция для статистики заказов."""

    def __init__(self):
        self.total_orders = 0
        self.total_revenue = 0.0
        self.orders_by_status: Dict[str, int] = {}
        self.daily_orders: Dict[str, int] = {}

    def handle(self, event: Event) -> None:
        if event.event_type == EventType.ORDER_PLACED.value:
            self.total_orders += 1
            self.total_revenue += event.data["total_amount"]

            date = event.timestamp.strftime("%Y-%m-%d")
            self.daily_orders[date] = self.daily_orders.get(date, 0) + 1

            self.orders_by_status["pending"] = \
                self.orders_by_status.get("pending", 0) + 1

        elif event.event_type in ["order.shipped", "order.cancelled"]:
            status = event.event_type.split(".")[-1]
            self.orders_by_status["pending"] = \
                max(0, self.orders_by_status.get("pending", 0) - 1)
            self.orders_by_status[status] = \
                self.orders_by_status.get(status, 0) + 1

    def get_statistics(self) -> dict:
        return {
            "total_orders": self.total_orders,
            "total_revenue": self.total_revenue,
            "orders_by_status": self.orders_by_status,
            "daily_orders": self.daily_orders
        }


class ProjectionManager:
    """Менеджер проекций."""

    def __init__(self, event_store: EventStore):
        self.event_store = event_store
        self.projections: List[Projection] = []
        self.last_position = 0

    def register(self, projection: Projection) -> None:
        """Регистрация проекции."""
        self.projections.append(projection)

    def rebuild_all(self) -> None:
        """Перестроение всех проекций с нуля."""
        self.last_position = 0
        for event in self.event_store.get_all_events():
            for projection in self.projections:
                projection.handle(event)
            self.last_position += 1

    def catch_up(self) -> None:
        """Обновление проекций новыми событиями."""
        for event in self.event_store.get_all_events(self.last_position):
            for projection in self.projections:
                projection.handle(event)
            self.last_position += 1
```

## Saga Pattern

```python
from enum import Enum
from typing import Dict, List, Optional, Callable
from abc import ABC, abstractmethod


class SagaState(Enum):
    STARTED = "started"
    COMPENSATING = "compensating"
    COMPLETED = "completed"
    FAILED = "failed"


class SagaStep:
    """Шаг саги."""

    def __init__(
        self,
        name: str,
        action: Callable,
        compensation: Callable
    ):
        self.name = name
        self.action = action
        self.compensation = compensation
        self.completed = False


class Saga(ABC):
    """Базовый класс саги."""

    def __init__(self, saga_id: str):
        self.saga_id = saga_id
        self.state = SagaState.STARTED
        self.steps: List[SagaStep] = []
        self.current_step = 0
        self.context: Dict = {}

    @abstractmethod
    def configure_steps(self) -> None:
        """Конфигурация шагов саги."""
        pass

    async def execute(self) -> bool:
        """Выполнение саги."""
        self.configure_steps()

        try:
            for step in self.steps:
                await step.action(self.context)
                step.completed = True
                self.current_step += 1

            self.state = SagaState.COMPLETED
            return True

        except Exception as e:
            self.state = SagaState.COMPENSATING
            await self._compensate()
            self.state = SagaState.FAILED
            return False

    async def _compensate(self) -> None:
        """Компенсация выполненных шагов."""
        for step in reversed(self.steps[:self.current_step]):
            try:
                await step.compensation(self.context)
            except Exception as e:
                # Логирование ошибки компенсации
                pass


class OrderSaga(Saga):
    """Сага для создания заказа."""

    def __init__(
        self,
        saga_id: str,
        order_service,
        payment_service,
        inventory_service,
        notification_service
    ):
        super().__init__(saga_id)
        self.order_service = order_service
        self.payment_service = payment_service
        self.inventory_service = inventory_service
        self.notification_service = notification_service

    def configure_steps(self) -> None:
        self.steps = [
            SagaStep(
                "reserve_inventory",
                self._reserve_inventory,
                self._release_inventory
            ),
            SagaStep(
                "process_payment",
                self._process_payment,
                self._refund_payment
            ),
            SagaStep(
                "confirm_order",
                self._confirm_order,
                self._cancel_order
            ),
            SagaStep(
                "send_notification",
                self._send_notification,
                lambda ctx: None  # Уведомление не требует компенсации
            )
        ]

    async def _reserve_inventory(self, ctx: Dict) -> None:
        reservation_id = await self.inventory_service.reserve(
            ctx["items"]
        )
        ctx["reservation_id"] = reservation_id

    async def _release_inventory(self, ctx: Dict) -> None:
        await self.inventory_service.release(
            ctx.get("reservation_id")
        )

    async def _process_payment(self, ctx: Dict) -> None:
        payment_id = await self.payment_service.charge(
            ctx["user_id"],
            ctx["amount"]
        )
        ctx["payment_id"] = payment_id

    async def _refund_payment(self, ctx: Dict) -> None:
        await self.payment_service.refund(
            ctx.get("payment_id")
        )

    async def _confirm_order(self, ctx: Dict) -> None:
        await self.order_service.confirm(
            ctx["order_id"]
        )

    async def _cancel_order(self, ctx: Dict) -> None:
        await self.order_service.cancel(
            ctx["order_id"],
            "Saga compensation"
        )

    async def _send_notification(self, ctx: Dict) -> None:
        await self.notification_service.send_order_confirmation(
            ctx["user_id"],
            ctx["order_id"]
        )
```

## Паттерны EDA

### Event Notification

```python
# Простое уведомление о событии

class EventNotificationService:
    """Сервис уведомлений о событиях."""

    def __init__(self, event_bus: EventBus):
        self.event_bus = event_bus

    def notify_user_created(self, user_id: str, email: str) -> None:
        """Уведомление о создании пользователя."""
        event = UserCreatedEvent(
            user_id=user_id,
            email=email,
            name=""
        )
        self.event_bus.publish(event)
        # Минимум данных - только уведомление


class UserEventHandler:
    """Обработчик событий пользователей."""

    def __init__(self, user_service):
        self.user_service = user_service

    def handle_user_created(self, event: Event) -> None:
        # Получаем полные данные через API
        user = self.user_service.get_user(event.data["user_id"])
        # Обработка...
```

### Event-Carried State Transfer

```python
# Событие содержит все необходимые данные

class RichUserCreatedEvent(Event):
    """Событие с полными данными пользователя."""

    def __init__(
        self,
        user_id: str,
        email: str,
        name: str,
        address: dict,
        preferences: dict,
        **kwargs
    ):
        super().__init__(
            event_type="user.created.rich",
            data={
                "user_id": user_id,
                "email": email,
                "name": name,
                "address": address,
                "preferences": preferences
            },
            **kwargs
        )


class UserCacheHandler:
    """Обработчик для кэширования данных пользователя."""

    def __init__(self, cache):
        self.cache = cache

    def handle(self, event: Event) -> None:
        # Все данные уже в событии - не нужен запрос к API
        self.cache.set(
            f"user:{event.data['user_id']}",
            event.data
        )
```

### CQRS (Command Query Responsibility Segregation)

```python
from abc import ABC, abstractmethod

# === Commands ===

class Command(ABC):
    """Базовый класс команды."""
    pass


@dataclass
class CreateOrderCommand(Command):
    user_id: str
    items: List[dict]
    shipping_address: str


@dataclass
class ShipOrderCommand(Command):
    order_id: str
    tracking_number: str


class CommandHandler(ABC):
    """Базовый обработчик команд."""

    @abstractmethod
    def handle(self, command: Command) -> None:
        pass


class CreateOrderHandler(CommandHandler):
    """Обработчик создания заказа."""

    def __init__(self, repository: OrderRepository):
        self.repository = repository

    def handle(self, command: CreateOrderCommand) -> str:
        order = Order.place(
            order_id=str(uuid.uuid4()),
            user_id=command.user_id,
            items=command.items,
            total_amount=sum(item["price"] * item["quantity"] for item in command.items)
        )

        self.repository.save(order)
        return order.aggregate_id


# === Queries ===

class Query(ABC):
    """Базовый класс запроса."""
    pass


@dataclass
class GetOrderQuery(Query):
    order_id: str


@dataclass
class GetUserOrdersQuery(Query):
    user_id: str


class QueryHandler(ABC):
    """Базовый обработчик запросов."""

    @abstractmethod
    def handle(self, query: Query) -> Any:
        pass


class GetOrderHandler(QueryHandler):
    """Обработчик получения заказа."""

    def __init__(self, projection: OrderListProjection):
        self.projection = projection

    def handle(self, query: GetOrderQuery) -> Optional[dict]:
        return self.projection.get_order(query.order_id)


class GetUserOrdersHandler(QueryHandler):
    """Обработчик получения заказов пользователя."""

    def __init__(self, projection: OrderListProjection):
        self.projection = projection

    def handle(self, query: GetUserOrdersQuery) -> List[dict]:
        return self.projection.get_user_orders(query.user_id)
```

## Преимущества EDA

### 1. Слабая связанность

```python
# Компоненты не знают друг о друге

class OrderService:
    def __init__(self, event_bus: EventBus):
        self.event_bus = event_bus

    def create_order(self, data: dict) -> str:
        order_id = str(uuid.uuid4())
        # Создаём заказ и публикуем событие
        event = OrderPlacedEvent(order_id=order_id, **data)
        self.event_bus.publish(event)
        return order_id
        # OrderService не знает о NotificationService, InventoryService и т.д.


class NotificationService:
    def __init__(self, event_bus: EventBus):
        event_bus.subscribe(
            EventType.ORDER_PLACED.value,
            self.handle_order_placed
        )

    def handle_order_placed(self, event: Event) -> None:
        # Отправляем уведомление
        pass
```

### 2. Масштабируемость

```python
# Легко масштабировать обработку событий

class ScalableEventProcessor:
    def __init__(self, partition_count: int = 4):
        self.partitions = [[] for _ in range(partition_count)]

    def assign_partition(self, event: Event) -> int:
        # Распределение по партициям
        key = event.data.get("user_id", event.event_id)
        return hash(key) % len(self.partitions)

    async def process(self, event: Event) -> None:
        partition = self.assign_partition(event)
        self.partitions[partition].append(event)
        # Каждая партиция может обрабатываться отдельным воркером
```

### 3. Аудит и отладка

```python
# Полная история событий

class AuditService:
    def __init__(self, event_store: EventStore):
        self.event_store = event_store

    def get_order_history(self, order_id: str) -> List[dict]:
        stream_id = f"order-{order_id}"
        events = self.event_store.get_events(stream_id)

        return [
            {
                "event_type": e.event_type,
                "timestamp": e.timestamp,
                "data": e.data
            }
            for e in events
        ]

    def replay_events(self, from_time: datetime) -> None:
        # Воспроизведение событий для анализа
        for event in self.event_store.get_all_events():
            if event.timestamp >= from_time:
                print(f"{event.timestamp}: {event.event_type}")
```

## Недостатки EDA

### 1. Сложность отладки

```python
# Событие проходит через несколько сервисов

# Решение: Correlation ID
class CorrelatedEvent(Event):
    def __init__(self, correlation_id: str = None, **kwargs):
        super().__init__(**kwargs)
        self.correlation_id = correlation_id or str(uuid.uuid4())


class TracingEventBus(EventBus):
    def __init__(self, tracer):
        self.tracer = tracer
        self._handlers = {}

    def publish(self, event: Event) -> None:
        with self.tracer.start_span(
            f"event.{event.event_type}",
            correlation_id=event.correlation_id
        ) as span:
            for handler in self._handlers.get(event.event_type, []):
                handler(event)
```

### 2. Eventual Consistency

```python
# Данные могут быть временно несогласованными

class EventuallyConsistentRead:
    """Чтение с учётом eventual consistency."""

    def __init__(self, projection, event_store):
        self.projection = projection
        self.event_store = event_store

    def read_with_version(self, entity_id: str, min_version: int) -> Optional[dict]:
        """Чтение с проверкой версии."""
        current_version = self.projection.get_version(entity_id)

        if current_version < min_version:
            # Ждём или перестраиваем проекцию
            self._catch_up_projection(entity_id, min_version)

        return self.projection.get(entity_id)
```

## Best Practices

### 1. Идемпотентность обработчиков

```python
class IdempotentEventHandler:
    """Идемпотентный обработчик событий."""

    def __init__(self, cache):
        self.cache = cache
        self.processed_events: Set[str] = set()

    def handle(self, event: Event) -> None:
        if event.event_id in self.processed_events:
            return  # Уже обработано

        if self.cache.get(f"processed:{event.event_id}"):
            return  # Уже обработано (постоянное хранилище)

        # Обработка события
        self._process(event)

        # Отмечаем как обработанное
        self.processed_events.add(event.event_id)
        self.cache.set(f"processed:{event.event_id}", True, ttl=86400)
```

### 2. Dead Letter Queue

```python
class DeadLetterQueue:
    """Очередь для необработанных событий."""

    def __init__(self):
        self.failed_events: List[tuple] = []

    def add(self, event: Event, error: Exception, attempts: int) -> None:
        self.failed_events.append((event, str(error), attempts))

    def retry_all(self, handler: EventHandler, max_attempts: int = 3) -> None:
        for event, error, attempts in list(self.failed_events):
            if attempts < max_attempts:
                try:
                    handler(event)
                    self.failed_events.remove((event, error, attempts))
                except Exception as e:
                    # Увеличиваем счётчик попыток
                    pass
```

## Когда использовать?

### Подходит для:

1. **Микросервисных архитектур** с асинхронным взаимодействием
2. **Систем с высокой нагрузкой** требующих масштабирования
3. **Аудита и compliance** требований
4. **Real-time обработки** данных

### Не подходит для:

1. **Простых CRUD операций** без сложной логики
2. **Синхронных транзакций** требующих немедленной консистентности
3. **Систем без требований к масштабированию**

## Заключение

Событийно-ориентированная архитектура — мощный инструмент для построения масштабируемых, слабосвязанных систем. Event Sourcing и CQRS дополняют EDA, обеспечивая полную историю изменений и оптимизацию чтения/записи. При правильном применении EDA значительно упрощает развитие и масштабирование сложных распределённых систем.
