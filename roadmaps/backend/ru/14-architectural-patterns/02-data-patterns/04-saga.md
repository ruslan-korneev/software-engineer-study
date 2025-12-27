# Saga Pattern - Паттерн для распределённых транзакций

## Определение

**Saga** — это архитектурный паттерн для управления распределёнными транзакциями в микросервисной архитектуре. Saga представляет собой последовательность локальных транзакций, где каждая транзакция обновляет данные в одном сервисе и публикует событие или сообщение для запуска следующей транзакции. Если какой-либо шаг завершается неудачей, выполняются компенсирующие транзакции для отмены предыдущих изменений.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Традиционная транзакция (ACID)                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   BEGIN TRANSACTION                                              │
│   ├── Операция 1 (Service A)                                    │
│   ├── Операция 2 (Service B)                                    │
│   ├── Операция 3 (Service C)                                    │
│   COMMIT / ROLLBACK                                              │
│                                                                  │
│   Проблема: невозможно в распределённой системе!                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                         Saga Pattern                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   T1 (Service A) ──► T2 (Service B) ──► T3 (Service C)          │
│        │                  │                  │                   │
│        ▼                  ▼                  ▼                   │
│   C1 (rollback)      C2 (rollback)      C3 (rollback)           │
│                                                                  │
│   При ошибке: C3 → C2 → C1 (компенсация в обратном порядке)     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Ключевые характеристики

### 1. Типы Saga

#### Choreography (Хореография)

Каждый сервис самостоятельно слушает события и решает, какое действие выполнить.

```python
# Choreography-based Saga
# Сервисы общаются через события без централизованного координатора

# === Order Service ===
class OrderService:
    def __init__(self, event_bus: EventBus):
        self.event_bus = event_bus

    async def create_order(self, order_data: dict) -> str:
        order = Order.create(order_data)
        await self.order_repo.save(order)

        # Публикуем событие — следующий сервис подхватит
        await self.event_bus.publish(
            OrderCreatedEvent(
                order_id=order.id,
                customer_id=order.customer_id,
                amount=order.total_amount,
                items=order.items
            )
        )
        return order.id

    @event_handler(PaymentFailedEvent)
    async def handle_payment_failed(self, event: PaymentFailedEvent):
        """Компенсация: отменяем заказ"""
        await self.order_repo.update_status(
            event.order_id,
            OrderStatus.CANCELLED
        )

# === Payment Service ===
class PaymentService:
    @event_handler(OrderCreatedEvent)
    async def handle_order_created(self, event: OrderCreatedEvent):
        """Реагируем на создание заказа"""
        try:
            payment = await self.process_payment(
                event.customer_id,
                event.amount
            )
            await self.event_bus.publish(
                PaymentCompletedEvent(
                    order_id=event.order_id,
                    payment_id=payment.id
                )
            )
        except PaymentException as e:
            await self.event_bus.publish(
                PaymentFailedEvent(
                    order_id=event.order_id,
                    reason=str(e)
                )
            )

    @event_handler(InventoryReservationFailedEvent)
    async def handle_inventory_failed(self, event: InventoryReservationFailedEvent):
        """Компенсация: возвращаем деньги"""
        await self.refund_payment(event.order_id)

# === Inventory Service ===
class InventoryService:
    @event_handler(PaymentCompletedEvent)
    async def handle_payment_completed(self, event: PaymentCompletedEvent):
        """Резервируем товары после оплаты"""
        try:
            await self.reserve_inventory(event.order_id)
            await self.event_bus.publish(
                InventoryReservedEvent(order_id=event.order_id)
            )
        except InsufficientInventoryException as e:
            await self.event_bus.publish(
                InventoryReservationFailedEvent(
                    order_id=event.order_id,
                    reason=str(e)
                )
            )
```

```
Choreography Flow:

┌──────────────┐    OrderCreated    ┌─────────────────┐
│    Order     │ ─────────────────► │    Payment      │
│   Service    │                    │    Service      │
└──────────────┘                    └────────┬────────┘
       ▲                                     │
       │                            PaymentCompleted
       │ PaymentFailed                       │
       │                                     ▼
       │                            ┌─────────────────┐
       └────────────────────────────│   Inventory     │
                                    │    Service      │
                                    └─────────────────┘
```

#### Orchestration (Оркестрация)

Централизованный координатор (оркестратор) управляет последовательностью шагов.

```python
# Orchestration-based Saga
# Центральный координатор управляет всем процессом

from enum import Enum
from dataclasses import dataclass, field
from typing import List, Optional, Callable, Awaitable

class SagaStepStatus(Enum):
    PENDING = "pending"
    COMPLETED = "completed"
    COMPENSATED = "compensated"
    FAILED = "failed"

@dataclass
class SagaStep:
    name: str
    action: Callable[..., Awaitable[None]]
    compensation: Callable[..., Awaitable[None]]
    status: SagaStepStatus = SagaStepStatus.PENDING

@dataclass
class SagaState:
    saga_id: str
    current_step: int = 0
    steps: List[SagaStep] = field(default_factory=list)
    context: dict = field(default_factory=dict)
    status: str = "pending"

class SagaOrchestrator:
    """Оркестратор для управления Saga"""

    def __init__(self, saga_store: SagaStore, event_bus: EventBus):
        self.saga_store = saga_store
        self.event_bus = event_bus

    async def execute(self, saga: SagaState) -> bool:
        """Выполняет все шаги Saga"""
        await self.saga_store.save(saga)

        try:
            for i, step in enumerate(saga.steps):
                saga.current_step = i

                try:
                    # Выполняем шаг
                    await step.action(saga.context)
                    step.status = SagaStepStatus.COMPLETED
                    await self.saga_store.save(saga)

                except Exception as e:
                    step.status = SagaStepStatus.FAILED
                    saga.status = "failed"
                    await self.saga_store.save(saga)

                    # Запускаем компенсацию
                    await self._compensate(saga, from_step=i - 1)
                    return False

            saga.status = "completed"
            await self.saga_store.save(saga)
            return True

        except Exception as e:
            saga.status = "error"
            await self.saga_store.save(saga)
            raise

    async def _compensate(self, saga: SagaState, from_step: int) -> None:
        """Выполняет компенсирующие действия в обратном порядке"""
        for i in range(from_step, -1, -1):
            step = saga.steps[i]

            if step.status == SagaStepStatus.COMPLETED:
                try:
                    await step.compensation(saga.context)
                    step.status = SagaStepStatus.COMPENSATED
                except Exception as e:
                    # Логируем ошибку, но продолжаем компенсацию
                    logger.error(f"Compensation failed for step {step.name}: {e}")

                await self.saga_store.save(saga)

# === Определение конкретной Saga ===
class OrderSagaDefinition:
    """Определение Saga для создания заказа"""

    def __init__(
        self,
        order_service: OrderService,
        payment_service: PaymentService,
        inventory_service: InventoryService,
        shipping_service: ShippingService
    ):
        self.order_service = order_service
        self.payment_service = payment_service
        self.inventory_service = inventory_service
        self.shipping_service = shipping_service

    def create(self, order_data: dict) -> SagaState:
        """Создаёт экземпляр Saga"""
        return SagaState(
            saga_id=str(uuid4()),
            context={"order_data": order_data},
            steps=[
                SagaStep(
                    name="create_order",
                    action=self._create_order,
                    compensation=self._cancel_order
                ),
                SagaStep(
                    name="reserve_inventory",
                    action=self._reserve_inventory,
                    compensation=self._release_inventory
                ),
                SagaStep(
                    name="process_payment",
                    action=self._process_payment,
                    compensation=self._refund_payment
                ),
                SagaStep(
                    name="schedule_shipping",
                    action=self._schedule_shipping,
                    compensation=self._cancel_shipping
                )
            ]
        )

    # === Actions ===

    async def _create_order(self, ctx: dict) -> None:
        order_id = await self.order_service.create_order(ctx["order_data"])
        ctx["order_id"] = order_id

    async def _reserve_inventory(self, ctx: dict) -> None:
        reservation_id = await self.inventory_service.reserve(
            ctx["order_id"],
            ctx["order_data"]["items"]
        )
        ctx["reservation_id"] = reservation_id

    async def _process_payment(self, ctx: dict) -> None:
        payment_id = await self.payment_service.process(
            ctx["order_id"],
            ctx["order_data"]["amount"]
        )
        ctx["payment_id"] = payment_id

    async def _schedule_shipping(self, ctx: dict) -> None:
        shipping_id = await self.shipping_service.schedule(
            ctx["order_id"],
            ctx["order_data"]["shipping_address"]
        )
        ctx["shipping_id"] = shipping_id

    # === Compensations ===

    async def _cancel_order(self, ctx: dict) -> None:
        if "order_id" in ctx:
            await self.order_service.cancel(ctx["order_id"])

    async def _release_inventory(self, ctx: dict) -> None:
        if "reservation_id" in ctx:
            await self.inventory_service.release(ctx["reservation_id"])

    async def _refund_payment(self, ctx: dict) -> None:
        if "payment_id" in ctx:
            await self.payment_service.refund(ctx["payment_id"])

    async def _cancel_shipping(self, ctx: dict) -> None:
        if "shipping_id" in ctx:
            await self.shipping_service.cancel(ctx["shipping_id"])
```

```
Orchestration Flow:

                        ┌─────────────────────┐
                        │   Saga Orchestrator  │
                        │    (Coordinator)     │
                        └──────────┬──────────┘
                                   │
           ┌───────────────────────┼───────────────────────┐
           │                       │                       │
           ▼                       ▼                       ▼
    ┌─────────────┐         ┌─────────────┐         ┌─────────────┐
    │   Order     │         │  Payment    │         │  Inventory  │
    │  Service    │         │  Service    │         │  Service    │
    └─────────────┘         └─────────────┘         └─────────────┘

    Orchestrator вызывает сервисы напрямую и управляет компенсацией
```

### 2. Хранение состояния Saga

```python
from datetime import datetime
from typing import Optional
import json

class SagaStore:
    """Хранилище состояния Saga для обеспечения надёжности"""

    def __init__(self, db: AsyncConnection):
        self.db = db

    async def save(self, saga: SagaState) -> None:
        """Сохраняет текущее состояние Saga"""
        steps_data = [
            {
                "name": step.name,
                "status": step.status.value
            }
            for step in saga.steps
        ]

        await self.db.execute(
            """
            INSERT INTO sagas (id, current_step, steps, context, status, updated_at)
            VALUES ($1, $2, $3, $4, $5, $6)
            ON CONFLICT (id) DO UPDATE SET
                current_step = $2,
                steps = $3,
                context = $4,
                status = $5,
                updated_at = $6
            """,
            saga.saga_id,
            saga.current_step,
            json.dumps(steps_data),
            json.dumps(saga.context),
            saga.status,
            datetime.utcnow()
        )

    async def load(self, saga_id: str) -> Optional[SagaState]:
        """Загружает Saga для восстановления"""
        row = await self.db.fetchrow(
            "SELECT * FROM sagas WHERE id = $1",
            saga_id
        )

        if not row:
            return None

        return self._row_to_saga(row)

    async def find_incomplete(self) -> List[SagaState]:
        """Находит незавершённые Saga для восстановления"""
        rows = await self.db.fetch(
            """
            SELECT * FROM sagas
            WHERE status NOT IN ('completed', 'compensated')
            AND updated_at < $1
            """,
            datetime.utcnow() - timedelta(minutes=5)  # Timeout
        )

        return [self._row_to_saga(row) for row in rows]

class SagaRecoveryService:
    """Сервис восстановления незавершённых Saga"""

    def __init__(
        self,
        saga_store: SagaStore,
        saga_orchestrator: SagaOrchestrator
    ):
        self.saga_store = saga_store
        self.orchestrator = saga_orchestrator

    async def recover_incomplete_sagas(self) -> None:
        """Запускается периодически для восстановления зависших Saga"""
        incomplete = await self.saga_store.find_incomplete()

        for saga in incomplete:
            try:
                if saga.status == "failed":
                    # Продолжаем компенсацию
                    await self.orchestrator._compensate(
                        saga,
                        from_step=saga.current_step - 1
                    )
                else:
                    # Пытаемся продолжить выполнение
                    await self.orchestrator.execute(saga)

            except Exception as e:
                logger.error(f"Failed to recover saga {saga.saga_id}: {e}")
```

---

## Когда использовать

### Идеальные сценарии

1. **Микросервисная архитектура** — когда транзакция затрагивает несколько сервисов

2. **Long-running процессы** — операции, которые занимают минуты/часы/дни

3. **Бизнес-процессы с несколькими участниками** — заказ, бронирование, регистрация

4. **Когда 2PC (Two-Phase Commit) неприемлем** — из-за производительности или доступности

```python
# Пример: бронирование путешествия
class TravelBookingSaga:
    """Saga для бронирования путешествия"""

    def create(self, booking_data: dict) -> SagaState:
        return SagaState(
            saga_id=str(uuid4()),
            context={"booking": booking_data},
            steps=[
                # Каждый шаг — вызов внешнего сервиса
                SagaStep("reserve_flight", self._reserve_flight, self._cancel_flight),
                SagaStep("reserve_hotel", self._reserve_hotel, self._cancel_hotel),
                SagaStep("reserve_car", self._reserve_car, self._cancel_car),
                SagaStep("charge_customer", self._charge, self._refund),
                SagaStep("confirm_all", self._confirm_all, self._noop),
            ]
        )
```

### Когда НЕ использовать

- В монолитных приложениях (используйте обычные транзакции)
- Для простых операций внутри одного сервиса
- Когда требуется строгая консистентность (ACID)
- При отсутствии инфраструктуры для обработки событий

---

## Преимущества и недостатки

### Преимущества

| Преимущество | Описание |
|-------------|----------|
| **Слабая связанность** | Сервисы не зависят друг от друга напрямую |
| **Масштабируемость** | Каждый сервис масштабируется независимо |
| **Отказоустойчивость** | Сбой одного сервиса не блокирует всю систему |
| **Гибкость** | Легко добавлять/изменять шаги процесса |
| **Аудит** | Полная история всех шагов |

### Недостатки

| Недостаток | Описание |
|-----------|----------|
| **Eventual Consistency** | Данные временно несогласованы |
| **Сложность** | Требует продуманной архитектуры |
| **Компенсации** | Не всегда возможно полностью откатить действие |
| **Дублирование** | Нужна идемпотентность всех операций |
| **Отладка** | Сложно отслеживать распределённый процесс |

---

## Примеры реализации

### Semantic Lock (Семантическая блокировка)

```python
class SemanticLock:
    """
    Семантическая блокировка для защиты от параллельных изменений.
    Вместо технической блокировки — бизнес-статус.
    """

    async def acquire(self, resource_id: str, saga_id: str) -> bool:
        """Устанавливает блокировку"""
        result = await self.db.execute(
            """
            UPDATE resources SET
                locked_by_saga = $1,
                locked_at = $2
            WHERE id = $3 AND locked_by_saga IS NULL
            RETURNING id
            """,
            saga_id,
            datetime.utcnow(),
            resource_id
        )
        return result is not None

    async def release(self, resource_id: str, saga_id: str) -> None:
        """Снимает блокировку"""
        await self.db.execute(
            """
            UPDATE resources SET
                locked_by_saga = NULL,
                locked_at = NULL
            WHERE id = $1 AND locked_by_saga = $2
            """,
            resource_id,
            saga_id
        )

class OrderWithLock:
    """Заказ с семантической блокировкой"""

    def __init__(self):
        self.status: OrderStatus
        self.pending_saga_id: Optional[str] = None

    def start_processing(self, saga_id: str) -> None:
        """Начинает обработку заказа"""
        if self.pending_saga_id is not None:
            raise OrderAlreadyProcessingError()

        self.pending_saga_id = saga_id
        self.status = OrderStatus.PENDING

    def complete_processing(self, saga_id: str) -> None:
        """Завершает обработку успешно"""
        if self.pending_saga_id != saga_id:
            raise InvalidSagaError()

        self.pending_saga_id = None
        self.status = OrderStatus.CONFIRMED

    def fail_processing(self, saga_id: str) -> None:
        """Отменяет обработку"""
        if self.pending_saga_id != saga_id:
            raise InvalidSagaError()

        self.pending_saga_id = None
        self.status = OrderStatus.CANCELLED
```

### Идемпотентность операций

```python
class IdempotentOperationHandler:
    """Обеспечивает идемпотентность операций в Saga"""

    def __init__(self, db: AsyncConnection):
        self.db = db

    async def execute_once(
        self,
        operation_id: str,
        operation: Callable[[], Awaitable[T]]
    ) -> T:
        """
        Выполняет операцию только один раз.
        При повторном вызове возвращает сохранённый результат.
        """
        # Проверяем, выполнялась ли операция
        existing = await self.db.fetchrow(
            "SELECT result FROM idempotent_operations WHERE id = $1",
            operation_id
        )

        if existing:
            return json.loads(existing["result"])

        # Выполняем операцию
        try:
            result = await operation()

            # Сохраняем результат
            await self.db.execute(
                """
                INSERT INTO idempotent_operations (id, result, created_at)
                VALUES ($1, $2, $3)
                ON CONFLICT (id) DO NOTHING
                """,
                operation_id,
                json.dumps(result),
                datetime.utcnow()
            )

            return result

        except Exception as e:
            # Сохраняем ошибку
            await self.db.execute(
                """
                INSERT INTO idempotent_operations (id, error, created_at)
                VALUES ($1, $2, $3)
                ON CONFLICT (id) DO NOTHING
                """,
                operation_id,
                str(e),
                datetime.utcnow()
            )
            raise

# Использование в Saga
class PaymentStep:
    async def process_payment(self, ctx: dict) -> None:
        operation_id = f"{ctx['saga_id']}_payment"

        payment_id = await self.idempotent_handler.execute_once(
            operation_id,
            lambda: self.payment_service.charge(
                ctx["customer_id"],
                ctx["amount"]
            )
        )

        ctx["payment_id"] = payment_id
```

### Timeout и Deadline

```python
from datetime import datetime, timedelta
import asyncio

class SagaWithTimeout:
    """Saga с поддержкой таймаутов"""

    def __init__(
        self,
        orchestrator: SagaOrchestrator,
        default_timeout: timedelta = timedelta(minutes=30)
    ):
        self.orchestrator = orchestrator
        self.default_timeout = default_timeout

    async def execute_with_timeout(
        self,
        saga: SagaState,
        timeout: Optional[timedelta] = None
    ) -> bool:
        """Выполняет Saga с ограничением по времени"""
        timeout = timeout or self.default_timeout
        deadline = datetime.utcnow() + timeout
        saga.context["deadline"] = deadline.isoformat()

        try:
            return await asyncio.wait_for(
                self.orchestrator.execute(saga),
                timeout=timeout.total_seconds()
            )
        except asyncio.TimeoutError:
            # Таймаут — запускаем компенсацию
            saga.status = "timeout"
            await self.orchestrator._compensate(saga, saga.current_step)
            return False

class StepWithTimeout:
    """Шаг Saga с индивидуальным таймаутом"""

    def __init__(
        self,
        action: Callable,
        compensation: Callable,
        timeout: timedelta
    ):
        self.action = action
        self.compensation = compensation
        self.timeout = timeout

    async def execute(self, ctx: dict) -> None:
        try:
            await asyncio.wait_for(
                self.action(ctx),
                timeout=self.timeout.total_seconds()
            )
        except asyncio.TimeoutError:
            raise StepTimeoutError(f"Step timed out after {self.timeout}")
```

### Parallel Saga Steps

```python
class ParallelSagaStep:
    """Параллельное выполнение нескольких шагов"""

    def __init__(self, steps: List[SagaStep]):
        self.steps = steps

    async def execute(self, ctx: dict) -> None:
        """Выполняет все шаги параллельно"""
        tasks = [step.action(ctx) for step in self.steps]

        try:
            await asyncio.gather(*tasks)

            for step in self.steps:
                step.status = SagaStepStatus.COMPLETED

        except Exception as e:
            # При ошибке компенсируем уже выполненные
            completed = [s for s in self.steps if s.status == SagaStepStatus.COMPLETED]
            await self._compensate_parallel(completed, ctx)
            raise

    async def _compensate_parallel(
        self,
        steps: List[SagaStep],
        ctx: dict
    ) -> None:
        """Параллельная компенсация"""
        tasks = [step.compensation(ctx) for step in steps]
        await asyncio.gather(*tasks, return_exceptions=True)

# Использование
class TravelBookingSagaWithParallel:
    def create(self, booking: dict) -> SagaState:
        return SagaState(
            saga_id=str(uuid4()),
            context={"booking": booking},
            steps=[
                # Последовательные шаги
                SagaStep("validate", self._validate, self._noop),

                # Параллельные бронирования
                ParallelSagaStep([
                    SagaStep("flight", self._book_flight, self._cancel_flight),
                    SagaStep("hotel", self._book_hotel, self._cancel_hotel),
                    SagaStep("car", self._book_car, self._cancel_car),
                ]),

                # После успешного бронирования — оплата
                SagaStep("payment", self._charge, self._refund),
            ]
        )
```

---

## Best Practices и антипаттерны

### Best Practices

```python
# 1. Идемпотентность ВСЕХ операций
class PaymentService:
    async def charge(self, idempotency_key: str, amount: Decimal) -> str:
        # Используем idempotency_key для защиты от дублирования
        existing = await self.find_by_idempotency_key(idempotency_key)
        if existing:
            return existing.payment_id

        return await self._create_payment(idempotency_key, amount)

# 2. Компенсация должна быть безопасной для повторного вызова
class InventoryService:
    async def release_reservation(self, reservation_id: str) -> None:
        reservation = await self.find_reservation(reservation_id)

        # Проверяем, не отменено ли уже
        if reservation is None or reservation.status == "released":
            return  # Идемпотентно

        await self.update_status(reservation_id, "released")
        await self.restore_inventory(reservation.items)

# 3. Логирование и трейсинг
class TracedSagaOrchestrator(SagaOrchestrator):
    async def execute(self, saga: SagaState) -> bool:
        with tracer.start_span("saga_execution") as span:
            span.set_attribute("saga_id", saga.saga_id)

            for step in saga.steps:
                with tracer.start_span(f"step_{step.name}") as step_span:
                    try:
                        await step.action(saga.context)
                        step_span.set_attribute("status", "completed")
                    except Exception as e:
                        step_span.set_attribute("status", "failed")
                        step_span.record_exception(e)
                        raise

# 4. Graceful degradation
class ResilientSagaStep:
    async def execute_with_fallback(self, ctx: dict) -> None:
        try:
            await self.primary_action(ctx)
        except ServiceUnavailableError:
            # Fallback стратегия
            await self.fallback_action(ctx)
```

### Антипаттерны

```python
# ❌ Антипаттерн: Неатомарные операции
async def bad_create_order(ctx: dict) -> None:
    order = await create_order(ctx)
    # Если здесь упадёт, заказ создан, но ctx не обновлён
    ctx["order_id"] = order.id

# ✅ Правильно: Атомарно обновляем контекст
async def good_create_order(ctx: dict) -> None:
    order = await create_order(ctx)
    ctx["order_id"] = order.id
    # Персистим контекст сразу
    await saga_store.save_context(ctx)

# ❌ Антипаттерн: Невозможная компенсация
async def send_email(ctx: dict) -> None:
    await email_service.send(ctx["customer_email"], "Order confirmed!")
    # Как "откатить" отправленное письмо?

# ✅ Правильно: Продумать последовательность
# Отправлять email только после подтверждения всех шагов
# Или: использовать "pending" статус в письме

# ❌ Антипаттерн: Синхронные вызовы между сервисами в Choreography
class BadChoreography:
    @event_handler(PaymentCompleted)
    async def handle_payment(self, event):
        # Плохо: синхронный вызов другого сервиса
        inventory = await inventory_client.reserve(event.order_id)

# ✅ Правильно: Асинхронные события
class GoodChoreography:
    @event_handler(PaymentCompleted)
    async def handle_payment(self, event):
        # Хорошо: публикуем событие
        await self.event_bus.publish(ReserveInventoryCommand(event.order_id))
```

---

## Связанные паттерны

| Паттерн | Связь с Saga |
|---------|-------------|
| **Event Sourcing** | События Saga можно хранить как Event Store |
| **CQRS** | Saga координирует команды между сервисами |
| **Outbox Pattern** | Гарантирует доставку событий Saga |
| **Choreography/Orchestration** | Два подхода к реализации Saga |
| **Compensating Transaction** | Основа механизма отката в Saga |

---

## Ресурсы для изучения

### Книги
- "Microservices Patterns" — Chris Richardson (глава о Saga)
- "Enterprise Integration Patterns" — Hohpe, Woolf

### Frameworks
- **Temporal** — платформа для durable workflows (рекомендуется!)
- **Camunda** — BPMN движок с поддержкой Saga
- **Eventuate Tram** — Saga framework для Java

### Статьи
- [Saga Pattern by Chris Richardson](https://microservices.io/patterns/data/saga.html)
- [Saga Pattern in Azure](https://docs.microsoft.com/en-us/azure/architecture/patterns/saga)

### Видео
- "Managing Data Consistency in Microservices" — Chris Richardson
- "Long Running Processes with Saga" — Bernd Ruecker
