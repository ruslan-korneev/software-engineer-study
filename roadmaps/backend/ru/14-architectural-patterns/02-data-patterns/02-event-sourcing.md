# Event Sourcing

## Определение

**Event Sourcing** — архитектурный паттерн, при котором состояние приложения определяется последовательностью событий (events), а не текущим снимком данных. Вместо хранения только актуального состояния объекта, мы сохраняем все события, которые привели к этому состоянию.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Традиционный подход (CRUD)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Account: { id: 1, balance: 150 }                              │
│                                                                  │
│   История изменений ПОТЕРЯНА                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Event Sourcing подход                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Event 1: AccountCreated { id: 1, initial_balance: 0 }         │
│   Event 2: MoneyDeposited { amount: 200 }                       │
│   Event 3: MoneyWithdrawn { amount: 50 }                        │
│                                                                  │
│   Текущий баланс = 0 + 200 - 50 = 150                           │
│                                                                  │
│   Полная история СОХРАНЕНА                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Ключевая идея: события — это факты, которые уже произошли. Они неизменяемы (immutable) и хранятся в append-only хранилище.

---

## Ключевые характеристики

### 1. Неизменяемость событий (Immutability)

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Any
import hashlib
import json

@dataclass(frozen=True)  # Делаем класс неизменяемым
class DomainEvent:
    """Базовый класс для всех доменных событий"""
    event_id: str
    aggregate_id: str
    aggregate_type: str
    version: int
    timestamp: datetime

    def to_dict(self) -> dict:
        return {
            "event_id": self.event_id,
            "event_type": self.__class__.__name__,
            "aggregate_id": self.aggregate_id,
            "aggregate_type": self.aggregate_type,
            "version": self.version,
            "timestamp": self.timestamp.isoformat(),
            "data": self._get_event_data()
        }

    def _get_event_data(self) -> dict:
        """Переопределяется в дочерних классах"""
        raise NotImplementedError

@dataclass(frozen=True)
class AccountCreated(DomainEvent):
    owner_id: str
    account_type: str

    def _get_event_data(self) -> dict:
        return {
            "owner_id": self.owner_id,
            "account_type": self.account_type
        }

@dataclass(frozen=True)
class MoneyDeposited(DomainEvent):
    amount: Decimal
    source: str

    def _get_event_data(self) -> dict:
        return {
            "amount": str(self.amount),
            "source": self.source
        }

@dataclass(frozen=True)
class MoneyWithdrawn(DomainEvent):
    amount: Decimal
    destination: str

    def _get_event_data(self) -> dict:
        return {
            "amount": str(self.amount),
            "destination": self.destination
        }
```

### 2. Event Store (Хранилище событий)

```python
from abc import ABC, abstractmethod
from typing import List, Optional

class EventStore(ABC):
    """Абстрактный интерфейс хранилища событий"""

    @abstractmethod
    async def append(
        self,
        aggregate_id: str,
        events: List[DomainEvent],
        expected_version: int
    ) -> None:
        """Добавляет события с оптимистичной блокировкой"""
        pass

    @abstractmethod
    async def get_events(
        self,
        aggregate_id: str,
        from_version: int = 0
    ) -> List[DomainEvent]:
        """Получает все события для агрегата"""
        pass

    @abstractmethod
    async def get_all_events(
        self,
        from_position: int = 0
    ) -> AsyncIterator[DomainEvent]:
        """Получает все события в хранилище (для проекций)"""
        pass

class PostgresEventStore(EventStore):
    """Реализация Event Store на PostgreSQL"""

    def __init__(self, connection_pool):
        self.pool = connection_pool

    async def append(
        self,
        aggregate_id: str,
        events: List[DomainEvent],
        expected_version: int
    ) -> None:
        async with self.pool.acquire() as conn:
            async with conn.transaction():
                # Проверяем текущую версию (optimistic locking)
                current = await conn.fetchval(
                    """
                    SELECT MAX(version) FROM events
                    WHERE aggregate_id = $1
                    """,
                    aggregate_id
                )
                current_version = current or 0

                if current_version != expected_version:
                    raise ConcurrencyError(
                        f"Expected version {expected_version}, "
                        f"but found {current_version}"
                    )

                # Записываем события
                for event in events:
                    await conn.execute(
                        """
                        INSERT INTO events
                        (event_id, aggregate_id, aggregate_type,
                         event_type, version, timestamp, data)
                        VALUES ($1, $2, $3, $4, $5, $6, $7)
                        """,
                        event.event_id,
                        event.aggregate_id,
                        event.aggregate_type,
                        event.__class__.__name__,
                        event.version,
                        event.timestamp,
                        json.dumps(event._get_event_data())
                    )

    async def get_events(
        self,
        aggregate_id: str,
        from_version: int = 0
    ) -> List[DomainEvent]:
        async with self.pool.acquire() as conn:
            rows = await conn.fetch(
                """
                SELECT * FROM events
                WHERE aggregate_id = $1 AND version > $2
                ORDER BY version ASC
                """,
                aggregate_id, from_version
            )
            return [self._deserialize_event(row) for row in rows]
```

### 3. Агрегат с Event Sourcing

```python
class EventSourcedAggregate(ABC):
    """Базовый класс для агрегатов с Event Sourcing"""

    def __init__(self):
        self._uncommitted_events: List[DomainEvent] = []
        self._version: int = 0
        self._id: Optional[str] = None

    @property
    def id(self) -> str:
        return self._id

    @property
    def version(self) -> int:
        return self._version

    @property
    def uncommitted_events(self) -> List[DomainEvent]:
        return self._uncommitted_events.copy()

    def clear_uncommitted_events(self) -> None:
        self._uncommitted_events.clear()

    def _apply_event(self, event: DomainEvent) -> None:
        """Применяет событие к агрегату"""
        handler_name = f"_on_{self._to_snake_case(event.__class__.__name__)}"
        handler = getattr(self, handler_name, None)
        if handler:
            handler(event)
        self._version = event.version

    def _raise_event(self, event: DomainEvent) -> None:
        """Создаёт новое событие"""
        self._apply_event(event)
        self._uncommitted_events.append(event)

    @classmethod
    def _from_events(cls, events: List[DomainEvent]) -> "EventSourcedAggregate":
        """Восстанавливает агрегат из событий"""
        if not events:
            raise ValueError("Cannot create aggregate from empty events")

        instance = cls.__new__(cls)
        instance._uncommitted_events = []
        instance._version = 0
        instance._id = None

        for event in events:
            instance._apply_event(event)

        return instance

    @staticmethod
    def _to_snake_case(name: str) -> str:
        import re
        return re.sub(r'(?<!^)(?=[A-Z])', '_', name).lower()

class BankAccount(EventSourcedAggregate):
    """Банковский счёт с Event Sourcing"""

    def __init__(self):
        super().__init__()
        self._owner_id: Optional[str] = None
        self._balance: Decimal = Decimal("0")
        self._is_active: bool = False

    @classmethod
    def create(
        cls,
        account_id: str,
        owner_id: str,
        account_type: str = "checking"
    ) -> "BankAccount":
        """Фабричный метод для создания нового счёта"""
        account = cls()
        account._raise_event(
            AccountCreated(
                event_id=str(uuid4()),
                aggregate_id=account_id,
                aggregate_type="BankAccount",
                version=1,
                timestamp=datetime.utcnow(),
                owner_id=owner_id,
                account_type=account_type
            )
        )
        return account

    def deposit(self, amount: Decimal, source: str) -> None:
        """Внести деньги на счёт"""
        if not self._is_active:
            raise AccountInactiveError("Cannot deposit to inactive account")
        if amount <= 0:
            raise InvalidAmountError("Amount must be positive")

        self._raise_event(
            MoneyDeposited(
                event_id=str(uuid4()),
                aggregate_id=self._id,
                aggregate_type="BankAccount",
                version=self._version + 1,
                timestamp=datetime.utcnow(),
                amount=amount,
                source=source
            )
        )

    def withdraw(self, amount: Decimal, destination: str) -> None:
        """Снять деньги со счёта"""
        if not self._is_active:
            raise AccountInactiveError("Cannot withdraw from inactive account")
        if amount <= 0:
            raise InvalidAmountError("Amount must be positive")
        if amount > self._balance:
            raise InsufficientFundsError(
                f"Cannot withdraw {amount}, balance is {self._balance}"
            )

        self._raise_event(
            MoneyWithdrawn(
                event_id=str(uuid4()),
                aggregate_id=self._id,
                aggregate_type="BankAccount",
                version=self._version + 1,
                timestamp=datetime.utcnow(),
                amount=amount,
                destination=destination
            )
        )

    # Event handlers (применяют события к состоянию)

    def _on_account_created(self, event: AccountCreated) -> None:
        self._id = event.aggregate_id
        self._owner_id = event.owner_id
        self._balance = Decimal("0")
        self._is_active = True

    def _on_money_deposited(self, event: MoneyDeposited) -> None:
        self._balance += event.amount

    def _on_money_withdrawn(self, event: MoneyWithdrawn) -> None:
        self._balance -= event.amount
```

### 4. Repository для Event-Sourced агрегатов

```python
class EventSourcedRepository:
    """Репозиторий для работы с Event-Sourced агрегатами"""

    def __init__(
        self,
        event_store: EventStore,
        event_bus: EventBus,
        aggregate_class: Type[EventSourcedAggregate]
    ):
        self.event_store = event_store
        self.event_bus = event_bus
        self.aggregate_class = aggregate_class

    async def get(self, aggregate_id: str) -> EventSourcedAggregate:
        """Загружает агрегат, восстанавливая его из событий"""
        events = await self.event_store.get_events(aggregate_id)
        if not events:
            raise AggregateNotFoundError(
                f"{self.aggregate_class.__name__} with id {aggregate_id} not found"
            )
        return self.aggregate_class._from_events(events)

    async def save(self, aggregate: EventSourcedAggregate) -> None:
        """Сохраняет новые события агрегата"""
        events = aggregate.uncommitted_events
        if not events:
            return

        expected_version = aggregate.version - len(events)

        await self.event_store.append(
            aggregate.id,
            events,
            expected_version
        )

        # Публикуем события для подписчиков (проекции, саги и т.д.)
        for event in events:
            await self.event_bus.publish(event)

        aggregate.clear_uncommitted_events()
```

---

## Когда использовать

### Идеальные сценарии

1. **Аудит и compliance** — когда нужна полная история всех изменений

2. **Финансовые системы** — банкинг, платежи, биллинг

3. **Отмена операций** — возможность "отмотать" состояние назад

4. **Аналитика** — анализ поведения пользователей, A/B тестирование

5. **Распределённые системы** — когда нужна надёжная синхронизация

```python
# Пример: восстановление состояния на определённый момент времени
class BankAccountRepository:
    async def get_at_time(
        self,
        account_id: str,
        at_time: datetime
    ) -> BankAccount:
        """Получить состояние счёта на определённый момент"""
        events = await self.event_store.get_events(account_id)

        # Фильтруем события до указанного времени
        filtered_events = [
            e for e in events
            if e.timestamp <= at_time
        ]

        return BankAccount._from_events(filtered_events)

# Пример: полная история операций
class TransactionHistoryService:
    async def get_account_history(
        self,
        account_id: str
    ) -> List[TransactionRecord]:
        """Получить полную историю операций"""
        events = await self.event_store.get_events(account_id)

        history = []
        balance = Decimal("0")

        for event in events:
            if isinstance(event, MoneyDeposited):
                balance += event.amount
                history.append(TransactionRecord(
                    type="deposit",
                    amount=event.amount,
                    balance_after=balance,
                    timestamp=event.timestamp,
                    details={"source": event.source}
                ))
            elif isinstance(event, MoneyWithdrawn):
                balance -= event.amount
                history.append(TransactionRecord(
                    type="withdrawal",
                    amount=event.amount,
                    balance_after=balance,
                    timestamp=event.timestamp,
                    details={"destination": event.destination}
                ))

        return history
```

### Когда НЕ использовать

- Простые CRUD-приложения без требований к аудиту
- Системы с частым обновлением и удалением данных
- Когда данные "забываются" по требованию (GDPR right to be forgotten)
- При отсутствии опыта работы с eventual consistency

---

## Преимущества и недостатки

### Преимущества

| Преимущество | Описание |
|-------------|----------|
| **Полный аудит** | Каждое изменение записано и может быть проанализировано |
| **Time Travel** | Можно восстановить состояние на любой момент времени |
| **Отладка** | Легко понять, что и когда произошло |
| **Устойчивость** | Append-only хранилище сложнее повредить |
| **Масштабирование** | События можно распределять между сервисами |
| **Гибкость проекций** | Можно создавать любые представления данных |

### Недостатки

| Недостаток | Описание |
|-----------|----------|
| **Сложность** | Требует изменения мышления и архитектуры |
| **Eventual Consistency** | Проекции отстают от текущего состояния |
| **Схема событий** | Эволюция схемы событий требует особого подхода |
| **Хранение** | Объём данных растёт со временем |
| **Производительность** | Восстановление состояния из большого числа событий |

---

## Примеры реализации

### Снапшоты (Snapshots)

```python
@dataclass
class Snapshot:
    """Снимок состояния агрегата"""
    aggregate_id: str
    aggregate_type: str
    version: int
    state: dict
    created_at: datetime

class SnapshotStore:
    """Хранилище снапшотов"""

    async def save_snapshot(self, snapshot: Snapshot) -> None:
        await self.db.execute(
            """
            INSERT INTO snapshots
            (aggregate_id, aggregate_type, version, state, created_at)
            VALUES ($1, $2, $3, $4, $5)
            ON CONFLICT (aggregate_id) DO UPDATE SET
                version = $3,
                state = $4,
                created_at = $5
            WHERE snapshots.version < $3
            """,
            snapshot.aggregate_id,
            snapshot.aggregate_type,
            snapshot.version,
            json.dumps(snapshot.state),
            snapshot.created_at
        )

    async def get_snapshot(
        self,
        aggregate_id: str
    ) -> Optional[Snapshot]:
        row = await self.db.fetchrow(
            "SELECT * FROM snapshots WHERE aggregate_id = $1",
            aggregate_id
        )
        if row:
            return Snapshot(
                aggregate_id=row["aggregate_id"],
                aggregate_type=row["aggregate_type"],
                version=row["version"],
                state=json.loads(row["state"]),
                created_at=row["created_at"]
            )
        return None

class EventSourcedRepositoryWithSnapshots:
    """Репозиторий с поддержкой снапшотов"""

    SNAPSHOT_FREQUENCY = 100  # Создавать снапшот каждые 100 событий

    def __init__(
        self,
        event_store: EventStore,
        snapshot_store: SnapshotStore,
        aggregate_class: Type[EventSourcedAggregate]
    ):
        self.event_store = event_store
        self.snapshot_store = snapshot_store
        self.aggregate_class = aggregate_class

    async def get(self, aggregate_id: str) -> EventSourcedAggregate:
        """Загружает агрегат, используя снапшот если доступен"""

        # Пробуем загрузить снапшот
        snapshot = await self.snapshot_store.get_snapshot(aggregate_id)

        if snapshot:
            # Восстанавливаем из снапшота
            aggregate = self.aggregate_class._from_snapshot(snapshot)

            # Загружаем только события после снапшота
            events = await self.event_store.get_events(
                aggregate_id,
                from_version=snapshot.version
            )
        else:
            # Загружаем все события
            aggregate = self.aggregate_class()
            events = await self.event_store.get_events(aggregate_id)

        # Применяем события
        for event in events:
            aggregate._apply_event(event)

        return aggregate

    async def save(self, aggregate: EventSourcedAggregate) -> None:
        """Сохраняет агрегат и создаёт снапшот при необходимости"""
        events = aggregate.uncommitted_events

        await self.event_store.append(
            aggregate.id,
            events,
            aggregate.version - len(events)
        )

        # Создаём снапшот если достигли порога
        if aggregate.version % self.SNAPSHOT_FREQUENCY == 0:
            snapshot = Snapshot(
                aggregate_id=aggregate.id,
                aggregate_type=aggregate.__class__.__name__,
                version=aggregate.version,
                state=aggregate._to_snapshot_state(),
                created_at=datetime.utcnow()
            )
            await self.snapshot_store.save_snapshot(snapshot)

        aggregate.clear_uncommitted_events()
```

### Эволюция схемы событий (Event Upcasting)

```python
class EventUpcaster:
    """Обновляет старые версии событий до актуальной схемы"""

    def upcast(self, event_data: dict) -> dict:
        """Приводит событие к актуальной версии"""
        event_type = event_data["event_type"]
        version = event_data.get("schema_version", 1)

        while version < self._get_current_version(event_type):
            upcaster_name = f"_upcast_{event_type}_v{version}_to_v{version + 1}"
            upcaster = getattr(self, upcaster_name, None)

            if upcaster:
                event_data = upcaster(event_data)

            version += 1
            event_data["schema_version"] = version

        return event_data

    def _upcast_MoneyDeposited_v1_to_v2(self, event_data: dict) -> dict:
        """Добавляем поле currency, которого не было в v1"""
        data = event_data.copy()
        data["data"]["currency"] = "USD"  # Default для старых событий
        return data

    def _upcast_MoneyDeposited_v2_to_v3(self, event_data: dict) -> dict:
        """Переименовываем source в origin"""
        data = event_data.copy()
        data["data"]["origin"] = data["data"].pop("source")
        return data
```

### Проекции (Projections)

```python
class AccountBalanceProjection:
    """Проекция для быстрого получения баланса"""

    def __init__(self, db: AsyncConnection):
        self.db = db

    async def handle(self, event: DomainEvent) -> None:
        """Обрабатывает событие и обновляет проекцию"""
        handler_name = f"_handle_{event.__class__.__name__}"
        handler = getattr(self, handler_name, None)
        if handler:
            await handler(event)

    async def _handle_AccountCreated(self, event: AccountCreated) -> None:
        await self.db.execute(
            """
            INSERT INTO account_balances
            (account_id, owner_id, balance, last_updated, version)
            VALUES ($1, $2, 0, $3, $4)
            """,
            event.aggregate_id,
            event.owner_id,
            event.timestamp,
            event.version
        )

    async def _handle_MoneyDeposited(self, event: MoneyDeposited) -> None:
        await self.db.execute(
            """
            UPDATE account_balances SET
                balance = balance + $1,
                last_updated = $2,
                version = $3
            WHERE account_id = $4 AND version < $3
            """,
            event.amount,
            event.timestamp,
            event.version,
            event.aggregate_id
        )

    async def _handle_MoneyWithdrawn(self, event: MoneyWithdrawn) -> None:
        await self.db.execute(
            """
            UPDATE account_balances SET
                balance = balance - $1,
                last_updated = $2,
                version = $3
            WHERE account_id = $4 AND version < $3
            """,
            event.amount,
            event.timestamp,
            event.version,
            event.aggregate_id
        )

class ProjectionManager:
    """Управляет всеми проекциями"""

    def __init__(self, event_store: EventStore):
        self.event_store = event_store
        self.projections: List[Projection] = []
        self._position: int = 0

    def register(self, projection: Projection) -> None:
        self.projections.append(projection)

    async def rebuild_all(self) -> None:
        """Полная перестройка всех проекций"""
        # Очищаем проекции
        for projection in self.projections:
            await projection.reset()

        # Проходим по всем событиям
        async for event in self.event_store.get_all_events(from_position=0):
            for projection in self.projections:
                await projection.handle(event)

    async def catch_up(self) -> None:
        """Догоняем новые события"""
        async for event in self.event_store.get_all_events(
            from_position=self._position
        ):
            for projection in self.projections:
                await projection.handle(event)
            self._position += 1
```

---

## Best Practices и антипаттерны

### Best Practices

```python
# 1. Маленькие, сфокусированные события
@dataclass(frozen=True)
class OrderItemAdded(DomainEvent):
    product_id: str
    quantity: int
    price: Decimal
    # НЕ включаем весь заказ, только данные об изменении

# 2. Идемпотентная обработка событий
class OrderProjection:
    async def handle(self, event: DomainEvent) -> None:
        # Проверяем, не обработали ли уже это событие
        if await self._is_processed(event.event_id):
            return

        await self._process(event)
        await self._mark_processed(event.event_id)

# 3. Версионирование схемы событий
@dataclass(frozen=True)
class MoneyDepositedV2(DomainEvent):
    """Версия 2 события с дополнительными полями"""
    amount: Decimal
    currency: str  # Новое поле
    origin: str    # Переименовано из source

    SCHEMA_VERSION = 2

# 4. Корреляция событий
@dataclass(frozen=True)
class TransferInitiated(DomainEvent):
    correlation_id: str  # Связывает все события одной операции
    from_account: str
    to_account: str
    amount: Decimal
```

### Антипаттерны

```python
# ❌ Антипаттерн: Изменяемые события
@dataclass
class MutableEvent:  # Плохо: должен быть frozen=True
    data: dict

    def update_data(self, key, value):  # Плохо: события неизменяемы!
        self.data[key] = value

# ❌ Антипаттерн: Слишком большие события
@dataclass(frozen=True)
class OrderUpdated(DomainEvent):
    # Плохо: включает ВСЁ состояние заказа
    full_order_state: dict  # Это не событие, а снапшот!

# ❌ Антипаттерн: Зависимость от внешних данных в событии
@dataclass(frozen=True)
class ProductAddedToCart(DomainEvent):
    product_id: str
    # Плохо: цена может измениться, а событие — нет
    # Нужно хранить цену НА МОМЕНТ добавления

# ✅ Правильно:
@dataclass(frozen=True)
class ProductAddedToCart(DomainEvent):
    product_id: str
    price_at_time: Decimal  # Цена зафиксирована в событии

# ❌ Антипаттерн: Удаление событий
async def delete_old_events():  # НИКОГДА не делайте это!
    await event_store.delete_before(date)

# ✅ Правильно: Архивация с сохранением
async def archive_old_events():
    events = await event_store.get_before(date)
    await archive_store.save(events)
    # События остаются в основном хранилище
```

---

## Связанные паттерны

| Паттерн | Связь с Event Sourcing |
|---------|----------------------|
| **CQRS** | Естественный компаньон; события используются для построения Read Model |
| **Domain Events** | Основа Event Sourcing — доменные события |
| **Saga** | Координация распределённых транзакций через события |
| **Aggregate** | Event Sourcing применяется к агрегатам DDD |
| **Snapshot** | Оптимизация восстановления состояния |

---

## Ресурсы для изучения

### Книги
- "Event Sourcing & CQRS" — Greg Young
- "Implementing Domain-Driven Design" — Vaughn Vernon (глава о Event Sourcing)
- "Building Event-Driven Microservices" — Adam Bellemare

### Библиотеки Python
- `eventsourcing` — полнофункциональная библиотека для Event Sourcing
- `esdbclient` — клиент для EventStoreDB

### Базы данных для Event Sourcing
- **EventStoreDB** — специализированная БД для Event Sourcing
- **PostgreSQL** — с правильной схемой можно использовать как Event Store
- **Apache Kafka** — для распределённых систем

### Видео
- "Event Sourcing" — Greg Young (автор паттерна)
- "The Many Meanings of Event-Driven Architecture" — Martin Fowler
