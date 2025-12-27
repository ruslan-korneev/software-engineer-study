# Event Sourcing

## Что такое Event Sourcing?

Event Sourcing — это архитектурный паттерн, при котором состояние приложения определяется последовательностью событий (events), а не текущим снимком данных. Вместо хранения текущего состояния объекта мы храним все события, которые привели к этому состоянию.

```
┌─────────────────────────────────────────────────────────────────┐
│                      Event Sourcing                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Традиционный подход:                                          │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Account: { id: 123, balance: 150 }                      │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   Event Sourcing:                                               │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Event 1: AccountCreated { id: 123 }                     │  │
│   │  Event 2: MoneyDeposited { amount: 100 }                 │  │
│   │  Event 3: MoneyDeposited { amount: 100 }                 │  │
│   │  Event 4: MoneyWithdrawn { amount: 50 }                  │  │
│   │                                                          │  │
│   │  Текущий баланс = 0 + 100 + 100 - 50 = 150              │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Основные концепции

### Event (Событие)

Событие — это неизменяемый факт, который произошёл в системе.

```python
from dataclasses import dataclass, field
from datetime import datetime
from uuid import uuid4
from typing import Any
import json


@dataclass(frozen=True)
class Event:
    """Базовый класс для событий"""
    event_id: str = field(default_factory=lambda: str(uuid4()))
    timestamp: datetime = field(default_factory=datetime.now)
    version: int = 1

    @property
    def event_type(self) -> str:
        return self.__class__.__name__

    def to_dict(self) -> dict:
        return {
            'event_id': self.event_id,
            'event_type': self.event_type,
            'timestamp': self.timestamp.isoformat(),
            'version': self.version,
            'data': self._get_data()
        }

    def _get_data(self) -> dict:
        """Переопределяется в подклассах"""
        return {}


# События для банковского счёта
@dataclass(frozen=True)
class AccountCreated(Event):
    """Событие: счёт создан"""
    account_id: str = ""
    owner_name: str = ""
    initial_balance: float = 0.0

    def _get_data(self) -> dict:
        return {
            'account_id': self.account_id,
            'owner_name': self.owner_name,
            'initial_balance': self.initial_balance
        }


@dataclass(frozen=True)
class MoneyDeposited(Event):
    """Событие: деньги внесены"""
    account_id: str = ""
    amount: float = 0.0
    description: str = ""

    def _get_data(self) -> dict:
        return {
            'account_id': self.account_id,
            'amount': self.amount,
            'description': self.description
        }


@dataclass(frozen=True)
class MoneyWithdrawn(Event):
    """Событие: деньги сняты"""
    account_id: str = ""
    amount: float = 0.0
    description: str = ""

    def _get_data(self) -> dict:
        return {
            'account_id': self.account_id,
            'amount': self.amount,
            'description': self.description
        }


@dataclass(frozen=True)
class AccountClosed(Event):
    """Событие: счёт закрыт"""
    account_id: str = ""
    reason: str = ""

    def _get_data(self) -> dict:
        return {
            'account_id': self.account_id,
            'reason': self.reason
        }
```

### Event Store (Хранилище событий)

Event Store — это база данных, оптимизированная для хранения и чтения событий.

```python
from abc import ABC, abstractmethod
from typing import List, Optional
from datetime import datetime
import json


class EventStore(ABC):
    """Абстрактный интерфейс для хранилища событий"""

    @abstractmethod
    def append(self, stream_id: str, events: List[Event],
               expected_version: Optional[int] = None) -> None:
        """Добавить события в поток"""
        pass

    @abstractmethod
    def get_events(self, stream_id: str,
                   from_version: int = 0) -> List[Event]:
        """Получить события из потока"""
        pass

    @abstractmethod
    def get_all_events(self, from_position: int = 0) -> List[Event]:
        """Получить все события (для проекций)"""
        pass


class InMemoryEventStore(EventStore):
    """In-memory реализация Event Store"""

    def __init__(self):
        self.streams: dict[str, list[dict]] = {}
        self.all_events: list[dict] = []
        self.stream_versions: dict[str, int] = {}

    def append(self, stream_id: str, events: List[Event],
               expected_version: Optional[int] = None) -> None:
        """Добавить события с оптимистичной блокировкой"""

        current_version = self.stream_versions.get(stream_id, 0)

        # Проверка версии для оптимистичной блокировки
        if expected_version is not None:
            if current_version != expected_version:
                raise ConcurrencyError(
                    f"Expected version {expected_version}, "
                    f"but current is {current_version}"
                )

        if stream_id not in self.streams:
            self.streams[stream_id] = []

        for event in events:
            current_version += 1
            event_data = {
                **event.to_dict(),
                'stream_id': stream_id,
                'stream_version': current_version,
                'global_position': len(self.all_events)
            }
            self.streams[stream_id].append(event_data)
            self.all_events.append(event_data)

        self.stream_versions[stream_id] = current_version

    def get_events(self, stream_id: str,
                   from_version: int = 0) -> List[dict]:
        """Получить события из потока"""
        if stream_id not in self.streams:
            return []

        return [
            e for e in self.streams[stream_id]
            if e['stream_version'] > from_version
        ]

    def get_all_events(self, from_position: int = 0) -> List[dict]:
        """Получить все события"""
        return self.all_events[from_position:]


class ConcurrencyError(Exception):
    """Ошибка конкурентного доступа"""
    pass


# PostgreSQL реализация Event Store
class PostgresEventStore(EventStore):
    """Event Store на PostgreSQL"""

    def __init__(self, connection_string: str):
        import psycopg2
        self.conn = psycopg2.connect(connection_string)
        self._create_tables()

    def _create_tables(self):
        """Создание таблиц для хранения событий"""
        with self.conn.cursor() as cur:
            cur.execute("""
                CREATE TABLE IF NOT EXISTS events (
                    global_position SERIAL PRIMARY KEY,
                    stream_id VARCHAR(255) NOT NULL,
                    stream_version INT NOT NULL,
                    event_id VARCHAR(36) NOT NULL UNIQUE,
                    event_type VARCHAR(255) NOT NULL,
                    event_data JSONB NOT NULL,
                    metadata JSONB,
                    timestamp TIMESTAMP NOT NULL,
                    UNIQUE(stream_id, stream_version)
                );

                CREATE INDEX IF NOT EXISTS idx_events_stream_id
                ON events(stream_id, stream_version);
            """)
            self.conn.commit()

    def append(self, stream_id: str, events: List[Event],
               expected_version: Optional[int] = None) -> None:
        """Добавить события с оптимистичной блокировкой"""
        with self.conn.cursor() as cur:
            # Получаем текущую версию
            cur.execute(
                "SELECT COALESCE(MAX(stream_version), 0) FROM events WHERE stream_id = %s",
                (stream_id,)
            )
            current_version = cur.fetchone()[0]

            if expected_version is not None and current_version != expected_version:
                raise ConcurrencyError(
                    f"Expected version {expected_version}, but current is {current_version}"
                )

            # Добавляем события
            for event in events:
                current_version += 1
                cur.execute(
                    """INSERT INTO events
                       (stream_id, stream_version, event_id, event_type,
                        event_data, timestamp)
                       VALUES (%s, %s, %s, %s, %s, %s)""",
                    (
                        stream_id,
                        current_version,
                        event.event_id,
                        event.event_type,
                        json.dumps(event._get_data()),
                        event.timestamp
                    )
                )

            self.conn.commit()

    def get_events(self, stream_id: str, from_version: int = 0) -> List[dict]:
        """Получить события из потока"""
        with self.conn.cursor() as cur:
            cur.execute(
                """SELECT event_id, event_type, event_data, timestamp,
                          stream_version, global_position
                   FROM events
                   WHERE stream_id = %s AND stream_version > %s
                   ORDER BY stream_version""",
                (stream_id, from_version)
            )
            rows = cur.fetchall()

            return [
                {
                    'event_id': row[0],
                    'event_type': row[1],
                    'data': row[2],
                    'timestamp': row[3].isoformat(),
                    'stream_version': row[4],
                    'global_position': row[5]
                }
                for row in rows
            ]
```

### Aggregate (Агрегат)

Агрегат восстанавливает своё состояние из событий и генерирует новые события.

```python
from typing import List
from abc import ABC, abstractmethod


class Aggregate(ABC):
    """Базовый класс для агрегатов с Event Sourcing"""

    def __init__(self):
        self._uncommitted_events: List[Event] = []
        self._version = 0

    @property
    def version(self) -> int:
        return self._version

    def get_uncommitted_events(self) -> List[Event]:
        """Получить события, ещё не сохранённые в store"""
        return self._uncommitted_events.copy()

    def clear_uncommitted_events(self) -> None:
        """Очистить несохранённые события"""
        self._uncommitted_events.clear()

    def load_from_history(self, events: List[dict]) -> None:
        """Восстановить состояние из истории событий"""
        for event_data in events:
            self._apply_event(event_data, is_new=False)
            self._version = event_data.get('stream_version', self._version + 1)

    def _apply_event(self, event_data: dict, is_new: bool = True) -> None:
        """Применить событие к агрегату"""
        event_type = event_data.get('event_type')
        handler_name = f'_apply_{self._to_snake_case(event_type)}'

        handler = getattr(self, handler_name, None)
        if handler:
            handler(event_data.get('data', event_data))

        if is_new:
            self._uncommitted_events.append(event_data)

    def _raise_event(self, event: Event) -> None:
        """Сгенерировать новое событие"""
        self._apply_event(event.to_dict(), is_new=True)

    @staticmethod
    def _to_snake_case(name: str) -> str:
        """CamelCase -> snake_case"""
        import re
        return re.sub(r'(?<!^)(?=[A-Z])', '_', name).lower()


class BankAccount(Aggregate):
    """Банковский счёт с Event Sourcing"""

    def __init__(self):
        super().__init__()
        self.account_id: str = ""
        self.owner_name: str = ""
        self.balance: float = 0.0
        self.is_closed: bool = False

    # Команды (изменяют состояние через события)

    @classmethod
    def create(cls, account_id: str, owner_name: str,
               initial_balance: float = 0.0) -> 'BankAccount':
        """Создать новый счёт"""
        account = cls()
        account._raise_event(AccountCreated(
            account_id=account_id,
            owner_name=owner_name,
            initial_balance=initial_balance
        ))
        return account

    def deposit(self, amount: float, description: str = "") -> None:
        """Внести деньги"""
        if self.is_closed:
            raise ValueError("Cannot deposit to closed account")
        if amount <= 0:
            raise ValueError("Amount must be positive")

        self._raise_event(MoneyDeposited(
            account_id=self.account_id,
            amount=amount,
            description=description
        ))

    def withdraw(self, amount: float, description: str = "") -> None:
        """Снять деньги"""
        if self.is_closed:
            raise ValueError("Cannot withdraw from closed account")
        if amount <= 0:
            raise ValueError("Amount must be positive")
        if amount > self.balance:
            raise ValueError("Insufficient funds")

        self._raise_event(MoneyWithdrawn(
            account_id=self.account_id,
            amount=amount,
            description=description
        ))

    def close(self, reason: str = "") -> None:
        """Закрыть счёт"""
        if self.is_closed:
            raise ValueError("Account already closed")
        if self.balance != 0:
            raise ValueError("Cannot close account with non-zero balance")

        self._raise_event(AccountClosed(
            account_id=self.account_id,
            reason=reason
        ))

    # Обработчики событий (изменяют внутреннее состояние)

    def _apply_account_created(self, data: dict) -> None:
        self.account_id = data['account_id']
        self.owner_name = data['owner_name']
        self.balance = data['initial_balance']

    def _apply_money_deposited(self, data: dict) -> None:
        self.balance += data['amount']

    def _apply_money_withdrawn(self, data: dict) -> None:
        self.balance -= data['amount']

    def _apply_account_closed(self, data: dict) -> None:
        self.is_closed = True
```

### Repository (Репозиторий)

Репозиторий загружает и сохраняет агрегаты через Event Store.

```python
from typing import Optional, Type


class EventSourcedRepository:
    """Репозиторий для агрегатов с Event Sourcing"""

    def __init__(self, event_store: EventStore, aggregate_class: Type[Aggregate]):
        self.event_store = event_store
        self.aggregate_class = aggregate_class

    def get(self, aggregate_id: str) -> Optional[Aggregate]:
        """Загрузить агрегат по ID"""
        events = self.event_store.get_events(aggregate_id)

        if not events:
            return None

        aggregate = self.aggregate_class()
        aggregate.load_from_history(events)
        return aggregate

    def save(self, aggregate: Aggregate) -> None:
        """Сохранить агрегат"""
        events = aggregate.get_uncommitted_events()

        if not events:
            return

        # Получаем ID из первого события или агрегата
        stream_id = events[0].get('data', {}).get('account_id') or \
                    getattr(aggregate, 'account_id', str(id(aggregate)))

        # Сохраняем с оптимистичной блокировкой
        expected_version = aggregate.version - len(events)
        self.event_store.append(stream_id, events, expected_version)

        aggregate.clear_uncommitted_events()


# Пример использования
event_store = InMemoryEventStore()
repo = EventSourcedRepository(event_store, BankAccount)

# Создание счёта
account = BankAccount.create("ACC-001", "John Doe", 100.0)
account.deposit(50.0, "Salary")
account.withdraw(30.0, "Groceries")
repo.save(account)

# Загрузка счёта
loaded_account = repo.get("ACC-001")
print(f"Balance: {loaded_account.balance}")  # 120.0

# Продолжение работы
loaded_account.deposit(200.0, "Bonus")
repo.save(loaded_account)
```

## Projections (Проекции)

Проекции — это read-модели, построенные из событий.

```python
from abc import ABC, abstractmethod
from typing import Dict, List


class Projection(ABC):
    """Базовый класс для проекций"""

    @abstractmethod
    def handle(self, event: dict) -> None:
        """Обработать событие"""
        pass

    @abstractmethod
    def get_handled_events(self) -> List[str]:
        """Список обрабатываемых типов событий"""
        pass


class AccountBalanceProjection(Projection):
    """Проекция: текущие балансы счетов"""

    def __init__(self):
        self.balances: Dict[str, dict] = {}

    def get_handled_events(self) -> List[str]:
        return ['AccountCreated', 'MoneyDeposited', 'MoneyWithdrawn', 'AccountClosed']

    def handle(self, event: dict) -> None:
        event_type = event['event_type']
        data = event['data']

        if event_type == 'AccountCreated':
            self.balances[data['account_id']] = {
                'account_id': data['account_id'],
                'owner_name': data['owner_name'],
                'balance': data['initial_balance'],
                'is_closed': False
            }

        elif event_type == 'MoneyDeposited':
            account_id = data['account_id']
            if account_id in self.balances:
                self.balances[account_id]['balance'] += data['amount']

        elif event_type == 'MoneyWithdrawn':
            account_id = data['account_id']
            if account_id in self.balances:
                self.balances[account_id]['balance'] -= data['amount']

        elif event_type == 'AccountClosed':
            account_id = data['account_id']
            if account_id in self.balances:
                self.balances[account_id]['is_closed'] = True

    def get_balance(self, account_id: str) -> Optional[dict]:
        return self.balances.get(account_id)

    def get_all_balances(self) -> List[dict]:
        return list(self.balances.values())


class TransactionHistoryProjection(Projection):
    """Проекция: история транзакций"""

    def __init__(self):
        self.transactions: Dict[str, List[dict]] = {}

    def get_handled_events(self) -> List[str]:
        return ['MoneyDeposited', 'MoneyWithdrawn']

    def handle(self, event: dict) -> None:
        event_type = event['event_type']
        data = event['data']
        account_id = data['account_id']

        if account_id not in self.transactions:
            self.transactions[account_id] = []

        transaction = {
            'timestamp': event['timestamp'],
            'type': 'deposit' if event_type == 'MoneyDeposited' else 'withdrawal',
            'amount': data['amount'],
            'description': data.get('description', '')
        }
        self.transactions[account_id].append(transaction)

    def get_history(self, account_id: str) -> List[dict]:
        return self.transactions.get(account_id, [])


class ProjectionManager:
    """Менеджер проекций"""

    def __init__(self, event_store: EventStore):
        self.event_store = event_store
        self.projections: List[Projection] = []
        self.last_position = 0

    def register(self, projection: Projection) -> None:
        """Зарегистрировать проекцию"""
        self.projections.append(projection)

    def rebuild_all(self) -> None:
        """Перестроить все проекции с начала"""
        self.last_position = 0
        events = self.event_store.get_all_events(0)

        for event in events:
            self._dispatch_event(event)
            self.last_position = event.get('global_position', 0) + 1

    def update(self) -> None:
        """Обновить проекции новыми событиями"""
        events = self.event_store.get_all_events(self.last_position)

        for event in events:
            self._dispatch_event(event)
            self.last_position = event.get('global_position', 0) + 1

    def _dispatch_event(self, event: dict) -> None:
        """Отправить событие в подходящие проекции"""
        event_type = event.get('event_type')

        for projection in self.projections:
            if event_type in projection.get_handled_events():
                projection.handle(event)


# Использование проекций
balance_projection = AccountBalanceProjection()
history_projection = TransactionHistoryProjection()

manager = ProjectionManager(event_store)
manager.register(balance_projection)
manager.register(history_projection)

# Перестроить проекции из истории событий
manager.rebuild_all()

# Запросы к проекциям (быстрые, без replay событий)
balance = balance_projection.get_balance("ACC-001")
history = history_projection.get_history("ACC-001")
```

## Snapshots (Снимки)

Снимки ускоряют загрузку агрегатов с длинной историей.

```python
from dataclasses import dataclass
from datetime import datetime
import json


@dataclass
class Snapshot:
    """Снимок состояния агрегата"""
    aggregate_id: str
    aggregate_type: str
    version: int
    state: dict
    created_at: datetime


class SnapshotStore:
    """Хранилище снимков"""

    def __init__(self):
        self.snapshots: Dict[str, Snapshot] = {}

    def save(self, snapshot: Snapshot) -> None:
        self.snapshots[snapshot.aggregate_id] = snapshot

    def get(self, aggregate_id: str) -> Optional[Snapshot]:
        return self.snapshots.get(aggregate_id)


class SnapshotRepository:
    """Репозиторий с поддержкой снимков"""

    def __init__(self, event_store: EventStore,
                 snapshot_store: SnapshotStore,
                 aggregate_class: Type[Aggregate],
                 snapshot_frequency: int = 100):
        self.event_store = event_store
        self.snapshot_store = snapshot_store
        self.aggregate_class = aggregate_class
        self.snapshot_frequency = snapshot_frequency

    def get(self, aggregate_id: str) -> Optional[Aggregate]:
        """Загрузить агрегат (со снимком если есть)"""
        snapshot = self.snapshot_store.get(aggregate_id)

        aggregate = self.aggregate_class()

        if snapshot:
            # Восстанавливаем из снимка
            aggregate._restore_from_snapshot(snapshot.state)
            aggregate._version = snapshot.version

            # Загружаем только события после снимка
            events = self.event_store.get_events(
                aggregate_id,
                from_version=snapshot.version
            )
        else:
            # Загружаем все события
            events = self.event_store.get_events(aggregate_id)

        if not events and not snapshot:
            return None

        aggregate.load_from_history(events)
        return aggregate

    def save(self, aggregate: Aggregate) -> None:
        """Сохранить агрегат (со снимком если нужно)"""
        events = aggregate.get_uncommitted_events()

        if not events:
            return

        stream_id = getattr(aggregate, 'account_id', str(id(aggregate)))
        expected_version = aggregate.version - len(events)

        self.event_store.append(stream_id, events, expected_version)
        aggregate.clear_uncommitted_events()

        # Создаём снимок каждые N событий
        if aggregate.version % self.snapshot_frequency == 0:
            self._create_snapshot(aggregate)

    def _create_snapshot(self, aggregate: Aggregate) -> None:
        """Создать снимок состояния"""
        snapshot = Snapshot(
            aggregate_id=getattr(aggregate, 'account_id', str(id(aggregate))),
            aggregate_type=aggregate.__class__.__name__,
            version=aggregate.version,
            state=aggregate._get_snapshot_state(),
            created_at=datetime.now()
        )
        self.snapshot_store.save(snapshot)


class BankAccountWithSnapshot(BankAccount):
    """Банковский счёт с поддержкой снимков"""

    def _get_snapshot_state(self) -> dict:
        """Получить состояние для снимка"""
        return {
            'account_id': self.account_id,
            'owner_name': self.owner_name,
            'balance': self.balance,
            'is_closed': self.is_closed
        }

    def _restore_from_snapshot(self, state: dict) -> None:
        """Восстановить из снимка"""
        self.account_id = state['account_id']
        self.owner_name = state['owner_name']
        self.balance = state['balance']
        self.is_closed = state['is_closed']
```

## Плюсы и минусы Event Sourcing

### Плюсы

1. **Полная история** — можно восстановить любое состояние в любой момент времени
2. **Аудит** — встроенный аудит всех изменений
3. **Отладка** — легко понять, как система пришла к текущему состоянию
4. **Гибкость проекций** — можно создавать любые read-модели
5. **Temporal queries** — запросы к данным на определённый момент времени
6. **Event replay** — возможность перестроить состояние с исправлениями

### Минусы

1. **Сложность** — кривая обучения и сложность реализации
2. **Eventual consistency** — проекции могут отставать от записи
3. **Schema evolution** — сложность версионирования событий
4. **Размер хранилища** — события накапливаются со временем
5. **Запросы** — нельзя напрямую запросить текущее состояние без проекций

## Когда использовать Event Sourcing

### Используйте Event Sourcing когда:

- Важна полная история изменений (финансы, аудит)
- Нужен temporal querying
- Сложная бизнес-логика с множеством правил
- Event-driven архитектура
- Нужна возможность "отмотать" время назад

### Не используйте Event Sourcing когда:

- Простое CRUD приложение
- Нет требований к аудиту
- Команда не имеет опыта с ES
- Нужна строгая консистентность везде
- Прототип или MVP

## Заключение

Event Sourcing — мощный паттерн для систем, где важна история изменений и аудит. Он требует значительных инвестиций в обучение и инфраструктуру, но даёт уникальные возможности для анализа и отладки системы.
