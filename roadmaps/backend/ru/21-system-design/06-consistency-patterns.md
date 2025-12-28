# Паттерны согласованности (Consistency Patterns)

[prev: 05-cap-theorem](./05-cap-theorem.md) | [next: 07-availability-patterns](./07-availability-patterns.md)

---

## Введение

Согласованность данных — одна из ключевых проблем в распределённых системах. Когда данные реплицируются между несколькими узлами, возникает вопрос: как гарантировать, что все узлы видят одинаковые данные? Разные паттерны согласованности предлагают разные компромиссы между согласованностью, доступностью и производительностью (CAP-теорема).

---

## 1. Сильная согласованность (Strong Consistency)

### Определение

**Сильная согласованность** гарантирует, что после успешной записи все последующие чтения вернут обновлённое значение. Система ведёт себя так, как будто существует только одна копия данных.

### Характеристики

- После подтверждения записи все читатели видят новое значение
- Все операции упорядочены глобально
- Высокая задержка из-за необходимости синхронизации
- Снижение доступности при сетевых разделениях

### Примеры

```python
# Пример: банковская транзакция
class BankAccount:
    def __init__(self):
        self.balance = 0
        self.lock = threading.Lock()

    def transfer(self, amount: int) -> bool:
        with self.lock:  # Гарантируем сильную согласованность
            if self.balance >= amount:
                self.balance -= amount
                # Запись синхронно реплицируется на все узлы
                self.sync_to_all_replicas()
                return True
            return False

    def get_balance(self) -> int:
        with self.lock:
            return self.balance  # Всегда актуальное значение
```

### Когда использовать

- Финансовые транзакции (банки, платежи)
- Системы бронирования (авиабилеты, отели)
- Инвентаризация с ограниченным количеством товаров
- Критически важные конфигурации
- Системы голосования

### Реализации

- **Google Spanner**: использует TrueTime для глобальной согласованности
- **CockroachDB**: распределённый SQL с сильной согласованностью
- **etcd, Consul, ZooKeeper**: координационные сервисы

---

## 2. Слабая согласованность (Weak Consistency)

### Определение

**Слабая согласованность** не гарантирует, что последующие чтения вернут записанное значение. Система может вернуть устаревшие данные или данные могут быть потеряны.

### Характеристики

- Нет гарантий времени синхронизации
- Минимальные задержки
- Максимальная доступность
- Подходит для данных, которые можно потерять

### Примеры

```python
# Пример: кэш с политикой "best effort"
class WeakCache:
    def __init__(self):
        self.local_cache = {}

    def write(self, key: str, value: any):
        self.local_cache[key] = value
        # Асинхронная отправка без подтверждения
        asyncio.create_task(self.async_replicate(key, value))

    async def async_replicate(self, key: str, value: any):
        try:
            await self.send_to_replicas(key, value)
        except Exception:
            pass  # Игнорируем ошибки

    def read(self, key: str) -> any:
        return self.local_cache.get(key)  # Может быть устаревшим
```

### Когда использовать

- VoIP и видеозвонки (потерянные пакеты не перезапрашиваются)
- Онлайн-игры (позиция игроков)
- Live-стриминг
- Метрики и телеметрия, где потеря части данных допустима
- Кэши без критичности актуальности

---

## 3. Согласованность в конечном счёте (Eventual Consistency)

### Определение

**Eventual Consistency** гарантирует, что если новых записей не происходит, все реплики в конечном счёте придут к одному состоянию. Это компромисс между сильной и слабой согласованностью.

### Характеристики

- Временно допускаются расхождения между репликами
- Система сходится к согласованному состоянию
- Высокая доступность и низкая задержка
- Необходимы механизмы разрешения конфликтов

### Временное окно несогласованности

```
Время: ─────────────────────────────────────────────>

Запись на     │
узел A        ▼
              ┌───────────────────┐
Узел A:       │ value = "new"     │
              └───────────────────┘

              │← Окно несогласованности →│

              ┌───────────────────────────┌───────────────────┐
Узел B:       │ value = "old"             │ value = "new"     │
              └───────────────────────────└───────────────────┘
                                          ▲
                                          │
                                    Репликация
                                    завершена
```

### Пример реализации

```python
import time
import asyncio
from dataclasses import dataclass
from typing import Dict, Optional

@dataclass
class VersionedValue:
    value: any
    timestamp: float
    vector_clock: Dict[str, int]

class EventuallyConsistentStore:
    def __init__(self, node_id: str, replicas: list):
        self.node_id = node_id
        self.replicas = replicas
        self.data: Dict[str, VersionedValue] = {}
        self.vector_clock: Dict[str, int] = {node_id: 0}

    def write(self, key: str, value: any) -> VersionedValue:
        # Увеличиваем локальные часы
        self.vector_clock[self.node_id] = self.vector_clock.get(self.node_id, 0) + 1

        versioned = VersionedValue(
            value=value,
            timestamp=time.time(),
            vector_clock=self.vector_clock.copy()
        )
        self.data[key] = versioned

        # Асинхронная репликация
        asyncio.create_task(self._replicate_async(key, versioned))

        return versioned

    async def _replicate_async(self, key: str, versioned: VersionedValue):
        """Фоновая репликация на другие узлы"""
        for replica in self.replicas:
            try:
                await replica.receive_update(key, versioned)
            except Exception as e:
                # Retry logic, anti-entropy repair
                await self._schedule_retry(replica, key, versioned)

    def receive_update(self, key: str, incoming: VersionedValue):
        """Получение обновления от другого узла"""
        current = self.data.get(key)

        if current is None or self._should_accept(current, incoming):
            self.data[key] = incoming
            self._merge_vector_clock(incoming.vector_clock)

    def _should_accept(self, current: VersionedValue, incoming: VersionedValue) -> bool:
        """Last-Write-Wins стратегия разрешения конфликтов"""
        return incoming.timestamp > current.timestamp

    def read(self, key: str) -> Optional[any]:
        versioned = self.data.get(key)
        return versioned.value if versioned else None
```

### Стратегии разрешения конфликтов

```python
class ConflictResolution:

    @staticmethod
    def last_write_wins(values: list[VersionedValue]) -> VersionedValue:
        """Побеждает последняя запись по timestamp"""
        return max(values, key=lambda v: v.timestamp)

    @staticmethod
    def merge_sets(values: list[set]) -> set:
        """Объединение множеств (CRDT-подход)"""
        result = set()
        for v in values:
            result.update(v)
        return result

    @staticmethod
    def application_specific(values: list, resolver_func) -> any:
        """Пользовательская логика разрешения"""
        return resolver_func(values)

# CRDT: Grow-Only Counter
class GCounter:
    """Conflict-free Replicated Data Type для счётчиков"""

    def __init__(self, node_id: str):
        self.node_id = node_id
        self.counts: Dict[str, int] = {}

    def increment(self, amount: int = 1):
        self.counts[self.node_id] = self.counts.get(self.node_id, 0) + amount

    def value(self) -> int:
        return sum(self.counts.values())

    def merge(self, other: 'GCounter'):
        """Слияние всегда успешно — берём максимум по каждому узлу"""
        for node, count in other.counts.items():
            self.counts[node] = max(self.counts.get(node, 0), count)
```

### Когда использовать

- Социальные сети (лайки, комментарии)
- Системы рекомендаций
- Сессии пользователей
- Корзины в интернет-магазинах
- DNS
- CDN

---

## 4. Read-Your-Writes Consistency

### Определение

**Read-Your-Writes (RYW)** гарантирует, что пользователь всегда видит свои собственные записи. После записи этот же пользователь гарантированно увидит обновлённые данные, но другие пользователи могут видеть устаревшие.

### Характеристики

- Сессионная гарантия для конкретного пользователя
- Другие пользователи могут видеть устаревшие данные
- Улучшает пользовательский опыт

### Пример реализации

```python
from dataclasses import dataclass
from typing import Dict, Optional

@dataclass
class SessionContext:
    user_id: str
    last_write_timestamp: float = 0

class RYWStore:
    def __init__(self):
        self.data: Dict[str, tuple[any, float]] = {}  # key -> (value, timestamp)
        self.replicas: list = []

    def write(self, key: str, value: any, session: SessionContext) -> float:
        timestamp = time.time()
        self.data[key] = (value, timestamp)
        session.last_write_timestamp = timestamp

        # Асинхронная репликация
        self._replicate_async(key, value, timestamp)

        return timestamp

    def read(self, key: str, session: SessionContext) -> Optional[any]:
        # Читаем с реплики, но проверяем timestamp
        value, timestamp = self._read_from_replica(key)

        if timestamp < session.last_write_timestamp:
            # Данные устарели — читаем с primary
            value, timestamp = self._read_from_primary(key)

        return value

    def _read_from_replica(self, key: str) -> tuple:
        # Чтение с ближайшей реплики
        return self.replicas[0].get(key, (None, 0))

    def _read_from_primary(self, key: str) -> tuple:
        # Чтение с главного узла
        return self.data.get(key, (None, 0))

# Использование
session = SessionContext(user_id="user_123")
store = RYWStore()

# Пользователь пишет
store.write("profile", {"name": "Иван"}, session)

# Тот же пользователь читает — гарантированно видит своё изменение
profile = store.read("profile", session)  # {"name": "Иван"}
```

### Реализация через sticky sessions

```python
class StickySessionRouter:
    """Маршрутизация пользователя на один и тот же узел"""

    def __init__(self, nodes: list):
        self.nodes = nodes

    def get_node_for_user(self, user_id: str) -> str:
        # Consistent hashing для привязки к узлу
        hash_value = hash(user_id)
        node_index = hash_value % len(self.nodes)
        return self.nodes[node_index]

    def route_request(self, user_id: str, operation: callable):
        node = self.get_node_for_user(user_id)
        return node.execute(operation)
```

---

## 5. Monotonic Reads Consistency

### Определение

**Monotonic Reads** гарантирует, что если пользователь прочитал значение, все последующие чтения вернут то же или более новое значение. Время не "откатывается назад".

### Проблема без Monotonic Reads

```
Время: ──────────────────────────────────────────>

Запись value=100:    ────┐
                         ▼
Primary:    ─────────────[100]─────────────────────

Replica A:  ─────────────────[100]─────────────────
                                  ▲
                                  │ Sync

Replica B:  ───────────────────────────────[100]───
                                            ▲
                                            │ Slow sync

Чтения пользователя:
1) Read from A → 100 ✓
2) Read from B → 50  ✗ (старое значение!)
3) Read from A → 100 ✓

Пользователь видит: 100 → 50 → 100 (нарушение!)
```

### Пример реализации

```python
from dataclasses import dataclass

@dataclass
class ReadToken:
    """Токен для отслеживания версии данных"""
    key: str
    version: int
    replica_id: str

class MonotonicReadsStore:
    def __init__(self, replicas: list):
        self.replicas = replicas
        self.user_tokens: Dict[str, Dict[str, ReadToken]] = {}  # user_id -> key -> token

    def read(self, key: str, user_id: str) -> tuple[any, ReadToken]:
        last_token = self.user_tokens.get(user_id, {}).get(key)

        if last_token:
            # Читаем с реплики, которая имеет версию >= последней прочитанной
            value, token = self._read_with_min_version(key, last_token.version)
        else:
            # Первое чтение — любая реплика
            value, token = self._read_any_replica(key)

        # Обновляем токен пользователя
        if user_id not in self.user_tokens:
            self.user_tokens[user_id] = {}
        self.user_tokens[user_id][key] = token

        return value, token

    def _read_with_min_version(self, key: str, min_version: int) -> tuple:
        for replica in self.replicas:
            value, version = replica.read(key)
            if version >= min_version:
                return value, ReadToken(key, version, replica.id)

        # Если нет подходящей реплики — читаем с primary
        return self._read_from_primary(key)

    def _read_any_replica(self, key: str) -> tuple:
        replica = self.replicas[0]
        value, version = replica.read(key)
        return value, ReadToken(key, version, replica.id)
```

---

## 6. Causal Consistency

### Определение

**Causal Consistency** гарантирует, что причинно связанные операции видны всем узлам в правильном порядке. Если операция B зависит от A, то все увидят A перед B.

### Причинно-следственные связи

```
Причинно связанные операции:
1. A читает X
2. A пишет Y (на основе прочитанного X)
→ Операции 1 и 2 причинно связаны

Независимые операции:
1. A пишет X
2. B пишет Y (не зная о X)
→ Могут быть видны в любом порядке
```

### Пример с Vector Clocks

```python
from typing import Dict
from dataclasses import dataclass, field

@dataclass
class VectorClock:
    """Векторные часы для отслеживания причинности"""
    clock: Dict[str, int] = field(default_factory=dict)

    def increment(self, node_id: str):
        self.clock[node_id] = self.clock.get(node_id, 0) + 1

    def merge(self, other: 'VectorClock'):
        for node, time in other.clock.items():
            self.clock[node] = max(self.clock.get(node, 0), time)

    def happens_before(self, other: 'VectorClock') -> bool:
        """Проверка: self → other (self случилось раньше other)"""
        at_least_one_less = False
        for node in set(self.clock.keys()) | set(other.clock.keys()):
            self_time = self.clock.get(node, 0)
            other_time = other.clock.get(node, 0)
            if self_time > other_time:
                return False
            if self_time < other_time:
                at_least_one_less = True
        return at_least_one_less

    def concurrent_with(self, other: 'VectorClock') -> bool:
        """Проверка: операции независимы (конкурентны)"""
        return not self.happens_before(other) and not other.happens_before(self)

class CausalStore:
    def __init__(self, node_id: str):
        self.node_id = node_id
        self.clock = VectorClock()
        self.data: Dict[str, tuple[any, VectorClock]] = {}
        self.pending: list = []  # Операции, ожидающие причинных зависимостей

    def write(self, key: str, value: any, dependencies: VectorClock = None):
        # Учитываем зависимости от прочитанных данных
        if dependencies:
            self.clock.merge(dependencies)

        self.clock.increment(self.node_id)
        self.data[key] = (value, self.clock.copy())

        return self.clock.copy()

    def receive_remote_write(self, key: str, value: any, remote_clock: VectorClock):
        """Получение записи от другого узла"""
        # Проверяем, выполнены ли причинные зависимости
        if self._dependencies_satisfied(remote_clock):
            self.data[key] = (value, remote_clock)
            self.clock.merge(remote_clock)
            self._process_pending()
        else:
            # Откладываем до выполнения зависимостей
            self.pending.append((key, value, remote_clock))

    def _dependencies_satisfied(self, remote_clock: VectorClock) -> bool:
        """Все ли причинные зависимости уже применены?"""
        for node, time in remote_clock.clock.items():
            if node == self.node_id:
                continue
            local_time = self.clock.clock.get(node, 0)
            # Для каждого другого узла мы должны видеть time-1 операций
            if local_time < time - 1:
                return False
        return True

    def _process_pending(self):
        """Обработка отложенных операций"""
        still_pending = []
        for key, value, clock in self.pending:
            if self._dependencies_satisfied(clock):
                self.data[key] = (value, clock)
                self.clock.merge(clock)
            else:
                still_pending.append((key, value, clock))
        self.pending = still_pending
```

### Пример использования

```python
# Сценарий: социальная сеть
store_alice = CausalStore("alice")
store_bob = CausalStore("bob")

# Alice создаёт пост
post_clock = store_alice.write("post_1", "Привет мир!")

# Bob читает пост и комментирует
# Комментарий причинно зависит от поста
comment_clock = store_bob.write(
    "comment_1",
    "Отличный пост!",
    dependencies=post_clock
)

# Causal consistency гарантирует:
# Любой узел, увидевший комментарий, сначала увидит пост
```

---

## 7. Linearizability vs Serializability

### Linearizability (Линеаризуемость)

**Linearizability** — самая строгая гарантия согласованности для одного объекта:
- Каждая операция выполняется атомарно в какой-то момент времени
- Этот момент находится между началом и концом операции
- Операции упорядочены по реальному времени

```
Real-time ordering требование:
Если операция A завершилась до начала операции B,
то A должна быть упорядочена перед B в глобальном порядке.

Время: ─────────────────────────────────────────────>

Client 1: ──[Write X=1]──

Client 2:         ──────[Read X]──
                                  │
                                  ▼
                        Должен вернуть 1

Client 3:                    ──[Write X=2]──
                                            │
Client 4:                             ──────[Read X]──
                                                      │
                                                      ▼
                                            Должен вернуть 2
```

### Serializability (Сериализуемость)

**Serializability** — гарантия для транзакций:
- Результат параллельного выполнения транзакций эквивалентен какому-то последовательному выполнению
- Не требует соблюдения реального времени

```python
# Пример: банковские транзакции
# Начальное состояние: A=100, B=100

# T1: Transfer $50 from A to B
def t1():
    a = read("A")  # 100
    b = read("B")  # 100
    write("A", a - 50)  # 50
    write("B", b + 50)  # 150

# T2: Calculate total
def t2():
    a = read("A")
    b = read("B")
    return a + b

# Serializable: T2 видит либо (100, 100) либо (50, 150)
# Всегда total = 200

# Non-serializable (плохо): T2 видит (50, 100)
# total = 150 — потеряли $50!
```

### Strict Serializability

**Strict Serializability = Linearizability + Serializability**:
- Транзакции сериализуемы
- Порядок соответствует реальному времени

```python
class StrictSerializableDB:
    """Пример с использованием 2PL (Two-Phase Locking)"""

    def __init__(self):
        self.data = {}
        self.locks = {}
        self.lock_manager = threading.Lock()

    def begin_transaction(self) -> int:
        return self._generate_txn_id()

    def read(self, txn_id: int, key: str) -> any:
        self._acquire_shared_lock(txn_id, key)
        return self.data.get(key)

    def write(self, txn_id: int, key: str, value: any):
        self._acquire_exclusive_lock(txn_id, key)
        self.data[key] = value

    def commit(self, txn_id: int):
        self._release_all_locks(txn_id)

    def _acquire_shared_lock(self, txn_id: int, key: str):
        # Shared lock — множество читателей или один писатель
        with self.lock_manager:
            if key not in self.locks:
                self.locks[key] = {"readers": set(), "writer": None}

            while self.locks[key]["writer"] is not None:
                self.lock_manager.wait()

            self.locks[key]["readers"].add(txn_id)

    def _acquire_exclusive_lock(self, txn_id: int, key: str):
        with self.lock_manager:
            if key not in self.locks:
                self.locks[key] = {"readers": set(), "writer": None}

            while (self.locks[key]["readers"] - {txn_id} or
                   self.locks[key]["writer"] not in (None, txn_id)):
                self.lock_manager.wait()

            self.locks[key]["writer"] = txn_id
```

### Сравнительная таблица

| Свойство | Linearizability | Serializability | Strict Serializability |
|----------|-----------------|-----------------|------------------------|
| Область | Один объект | Транзакции | Транзакции |
| Real-time ordering | Да | Нет | Да |
| Атомарность | Операция | Транзакция | Транзакция |
| Примеры | etcd, ZooKeeper | PostgreSQL (SSI) | Spanner, CockroachDB |

---

## 8. Quorum-based Consistency (W + R > N)

### Определение

**Quorum-based подход** использует кворумы (минимальное количество узлов) для чтения и записи. Правило `W + R > N` гарантирует пересечение множеств читающих и пишущих узлов.

```
N = общее количество реплик
W = количество подтверждений для записи (Write Quorum)
R = количество узлов для чтения (Read Quorum)

W + R > N гарантирует:
Хотя бы один узел участвует и в записи, и в чтении
→ чтение вернёт актуальные данные
```

### Визуализация

```
N = 5 реплик

Запись (W = 3):
   [1] ✓  [2] ✓  [3] ✓  [4] ✗  [5] ✗
   Запись успешна (3 подтверждения)

Чтение (R = 3):
   [1] ✓  [2] ✗  [3] ✓  [4] ✓  [5] ✗

Пересечение: узлы 1 и 3 имеют актуальные данные
→ Читатель получит актуальное значение
```

### Конфигурации кворумов

```python
class QuorumConfig:
    """Разные конфигурации для разных требований"""

    def __init__(self, n: int):
        self.n = n

    def strong_consistency(self) -> tuple[int, int]:
        """W + R > N, гарантирует согласованность"""
        w = self.n // 2 + 1
        r = self.n // 2 + 1
        return w, r  # Например: N=5 → W=3, R=3

    def read_optimized(self) -> tuple[int, int]:
        """Быстрое чтение, медленная запись"""
        w = self.n  # Запись на все узлы
        r = 1       # Чтение с любого узла
        return w, r

    def write_optimized(self) -> tuple[int, int]:
        """Быстрая запись, медленное чтение"""
        w = 1       # Запись на один узел
        r = self.n  # Чтение со всех узлов
        return w, r

    def available_consistency(self) -> tuple[int, int]:
        """Баланс между доступностью и согласованностью"""
        w = self.n // 2 + 1
        r = 1
        return w, r  # W + R может быть <= N!

# Пример использования
config = QuorumConfig(n=5)
print(config.strong_consistency())  # (3, 3)
print(config.read_optimized())      # (5, 1)
print(config.write_optimized())     # (1, 5)
```

### Реализация с версионированием

```python
import time
from dataclasses import dataclass
from typing import List, Tuple, Optional
import asyncio

@dataclass
class VersionedData:
    value: any
    version: int
    timestamp: float

class QuorumStore:
    def __init__(self, replicas: List['Replica'], w: int, r: int):
        self.replicas = replicas
        self.n = len(replicas)
        self.w = w
        self.r = r

        assert w + r > self.n, "W + R must be > N for strong consistency"

    async def write(self, key: str, value: any) -> bool:
        version = int(time.time() * 1000000)  # Microseconds as version
        data = VersionedData(value=value, version=version, timestamp=time.time())

        # Отправляем запись на все реплики параллельно
        tasks = [
            replica.write(key, data)
            for replica in self.replicas
        ]
        results = await asyncio.gather(*tasks, return_exceptions=True)

        # Считаем успешные записи
        successes = sum(1 for r in results if r is True)

        if successes >= self.w:
            return True
        else:
            raise Exception(f"Write quorum not reached: {successes}/{self.w}")

    async def read(self, key: str) -> Optional[any]:
        # Читаем с R реплик
        tasks = [
            replica.read(key)
            for replica in self.replicas[:self.r]
        ]
        results = await asyncio.gather(*tasks, return_exceptions=True)

        # Фильтруем успешные ответы
        valid_results: List[VersionedData] = [
            r for r in results
            if isinstance(r, VersionedData)
        ]

        if not valid_results:
            return None

        # Возвращаем значение с максимальной версией
        latest = max(valid_results, key=lambda x: x.version)

        # Read repair: обновляем устаревшие реплики
        await self._read_repair(key, latest, valid_results)

        return latest.value

    async def _read_repair(self, key: str, latest: VersionedData,
                           results: List[VersionedData]):
        """Фоновое исправление устаревших реплик"""
        for i, result in enumerate(results):
            if result.version < latest.version:
                await self.replicas[i].write(key, latest)

class Replica:
    def __init__(self, replica_id: str):
        self.id = replica_id
        self.data: Dict[str, VersionedData] = {}

    async def write(self, key: str, data: VersionedData) -> bool:
        current = self.data.get(key)
        if current is None or data.version > current.version:
            self.data[key] = data
        return True

    async def read(self, key: str) -> Optional[VersionedData]:
        return self.data.get(key)
```

### Sloppy Quorum и Hinted Handoff

```python
class SloppyQuorumStore:
    """
    Sloppy Quorum: если целевой узел недоступен,
    записываем на другой узел с "подсказкой" (hint)
    """

    def __init__(self, replicas: List['Replica'], w: int):
        self.replicas = replicas
        self.w = w
        self.hints: Dict[str, List[Tuple[str, VersionedData]]] = {}

    async def write_with_hints(self, key: str, value: any) -> bool:
        target_replicas = self._get_preference_list(key)
        data = VersionedData(value=value, version=time.time(), timestamp=time.time())

        successes = 0
        for replica in target_replicas:
            try:
                await replica.write(key, data)
                successes += 1
            except ReplicaUnavailable:
                # Записываем на следующий доступный узел с hint
                hint_replica = self._find_available_replica(target_replicas)
                if hint_replica:
                    await hint_replica.write_hint(key, data, target=replica.id)
                    successes += 1

            if successes >= self.w:
                return True

        return successes >= self.w

    async def handoff_hints(self, replica: 'Replica'):
        """Когда узел восстанавливается, отправляем ему накопленные hints"""
        hints = self.hints.get(replica.id, [])
        for key, data in hints:
            await replica.write(key, data)
        self.hints[replica.id] = []
```

---

## 9. Примеры реализации в популярных базах данных

### PostgreSQL

```sql
-- Уровни изоляции транзакций
-- Read Uncommitted (PostgreSQL трактует как Read Committed)
-- Read Committed (default)
-- Repeatable Read
-- Serializable

-- Serializable Snapshot Isolation
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- 100
UPDATE accounts SET balance = balance - 50 WHERE id = 1;
UPDATE accounts SET balance = balance + 50 WHERE id = 2;
COMMIT;  -- Может откатиться при конфликте

-- Синхронная репликация для сильной согласованности
-- postgresql.conf:
-- synchronous_commit = on
-- synchronous_standby_names = 'replica1, replica2'
```

### MongoDB

```javascript
// Write Concern — сколько узлов должны подтвердить запись
db.collection.insertOne(
  { item: "example" },
  {
    writeConcern: {
      w: "majority",      // Большинство узлов
      j: true,            // Запись в журнал
      wtimeout: 5000      // Таймаут в мс
    }
  }
)

// Read Concern — какие данные можно читать
db.collection.find().readConcern("majority")  // Только подтверждённые большинством
db.collection.find().readConcern("linearizable")  // Строгая согласованность

// Read Preference — откуда читать
db.collection.find().readPref("primaryPreferred")  // Предпочтительно с primary
db.collection.find().readPref("nearest")  // С ближайшего узла

// Causal Consistency с сессиями
const session = db.getMongo().startSession({ causalConsistency: true });
const coll = session.getDatabase("test").getCollection("items");
coll.insertOne({ name: "item1" });
coll.find({ name: "item1" });  // Гарантированно увидит вставленную запись
session.endSession();
```

### Cassandra

```cql
-- Consistency Levels
-- ANY, ONE, TWO, THREE, QUORUM, LOCAL_QUORUM, EACH_QUORUM, ALL

-- Для записи
INSERT INTO users (id, name) VALUES (1, 'Alice')
USING CONSISTENCY QUORUM;  -- W = N/2 + 1

-- Для чтения
SELECT * FROM users WHERE id = 1
CONSISTENCY LOCAL_QUORUM;  -- R = N/2 + 1 в локальном дата-центре

-- Lightweight Transactions (Linearizable)
INSERT INTO users (id, name)
VALUES (1, 'Alice')
IF NOT EXISTS;  -- Использует Paxos, строгая согласованность
```

```python
# Python driver
from cassandra.cluster import Cluster
from cassandra import ConsistencyLevel
from cassandra.query import SimpleStatement

cluster = Cluster(['node1', 'node2', 'node3'])
session = cluster.connect('keyspace')

# Настройка consistency level
statement = SimpleStatement(
    "SELECT * FROM users WHERE id = %s",
    consistency_level=ConsistencyLevel.QUORUM
)
result = session.execute(statement, [user_id])
```

### Redis

```python
import redis

# Redis Cluster с репликацией
r = redis.RedisCluster(
    host='localhost',
    port=6379,
    read_from_replicas=True  # Чтение с реплик (eventual consistency)
)

# WAIT команда для синхронной репликации
r.set('key', 'value')
replicas_acked = r.wait(numreplicas=2, timeout=1000)
# Ждём подтверждения от 2 реплик

# Redis Streams с XREAD для упорядоченных сообщений
r.xadd('mystream', {'field': 'value'})
messages = r.xread({'mystream': '0'}, block=0)
```

### etcd

```python
import etcd3

# etcd обеспечивает linearizability
client = etcd3.client()

# Запись с подтверждением от большинства (Raft consensus)
client.put('/config/key', 'value')

# Чтение linearizable (по умолчанию)
value, metadata = client.get('/config/key')

# Serializable read (быстрее, но может быть stale)
value, metadata = client.get('/config/key', serializable=True)

# Watch для получения изменений в реальном времени
events_iterator, cancel = client.watch('/config/')
for event in events_iterator:
    print(event)

# Транзакции (compare-and-swap)
client.transaction(
    compare=[
        client.transactions.value('/config/key') == 'old_value'
    ],
    success=[
        client.transactions.put('/config/key', 'new_value')
    ],
    failure=[]
)
```

### Amazon DynamoDB

```python
import boto3
from boto3.dynamodb.conditions import Key

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

# Eventually consistent read (default, дешевле)
response = table.get_item(
    Key={'user_id': '123'},
    ConsistentRead=False
)

# Strongly consistent read (дороже, с primary)
response = table.get_item(
    Key={'user_id': '123'},
    ConsistentRead=True
)

# Conditional write (optimistic locking)
table.update_item(
    Key={'user_id': '123'},
    UpdateExpression='SET balance = :new_balance',
    ConditionExpression='balance = :expected_balance',
    ExpressionAttributeValues={
        ':new_balance': 150,
        ':expected_balance': 100
    }
)

# Transactions (ACID across items)
client = boto3.client('dynamodb')
client.transact_write_items(
    TransactItems=[
        {
            'Update': {
                'TableName': 'Accounts',
                'Key': {'account_id': {'S': 'A'}},
                'UpdateExpression': 'SET balance = balance - :amount',
                'ExpressionAttributeValues': {':amount': {'N': '50'}}
            }
        },
        {
            'Update': {
                'TableName': 'Accounts',
                'Key': {'account_id': {'S': 'B'}},
                'UpdateExpression': 'SET balance = balance + :amount',
                'ExpressionAttributeValues': {':amount': {'N': '50'}}
            }
        }
    ]
)
```

---

## 10. Как выбрать подходящий паттерн

### Дерево принятия решений

```
Нужна ли согласованность в реальном времени?
│
├─ ДА → Критичны ли финансовые данные / безопасность?
│       │
│       ├─ ДА → Strong Consistency / Linearizability
│       │       Примеры: банки, бронирование, inventory
│       │
│       └─ НЕТ → Важна ли сессионная согласованность?
│               │
│               ├─ ДА → Read-Your-Writes + Monotonic Reads
│               │       Примеры: социальные сети, профили
│               │
│               └─ НЕТ → Causal Consistency
│                       Примеры: мессенджеры, комментарии
│
└─ НЕТ → Допустима ли временная несогласованность?
        │
        ├─ ДА (секунды/минуты) → Eventual Consistency
        │       Примеры: лайки, просмотры, рекомендации
        │
        └─ НЕТ → Weak Consistency
                Примеры: метрики, логи, кэши
```

### Матрица выбора по сценариям

| Сценарий | Рекомендуемый паттерн | Причина |
|----------|----------------------|---------|
| Банковские переводы | Strong / Linearizable | Нельзя терять деньги |
| Бронирование билетов | Strong + Distributed Lock | Нельзя продать одно место дважды |
| Корзина e-commerce | Eventual + CRDT | Допустима задержка, merge корзин |
| Лента социальной сети | Eventual | Порядок постов не критичен |
| Комментарии | Causal | Ответы должны идти после постов |
| Редактирование профиля | Read-Your-Writes | Пользователь должен видеть свои изменения |
| Чтение постов | Monotonic Reads | Нельзя "откатывать" ленту назад |
| Метрики/аналитика | Weak/Eventual | Потеря части данных допустима |
| Конфигурация сервисов | Strong (etcd, Consul) | Все сервисы должны видеть одинаковую конфигурацию |
| DNS | Eventual с TTL | Классический пример eventual consistency |
| Распределённый кэш | Eventual + TTL | Инвалидация через истечение |

### Пример архитектурного решения

```python
class ECommerceConsistencyStrategy:
    """Разные паттерны согласованности для разных данных"""

    def __init__(self):
        # Сильная согласованность для критичных данных
        self.inventory_store = StrongConsistencyStore()  # PostgreSQL with sync replication

        # Eventual consistency для корзины
        self.cart_store = EventualConsistencyStore()  # Redis Cluster

        # Read-your-writes для профилей
        self.profile_store = SessionConsistencyStore()  # MongoDB with read concern

        # Weak consistency для рекомендаций
        self.recommendations_store = WeakConsistencyStore()  # Cached ML results

    async def purchase_item(self, user_id: str, item_id: str, quantity: int):
        """Покупка требует разных уровней согласованности"""

        # 1. Проверка и резервирование товара — STRONG
        reserved = await self.inventory_store.reserve(
            item_id,
            quantity,
            consistency="linearizable"
        )
        if not reserved:
            raise OutOfStockError()

        try:
            # 2. Получение корзины — EVENTUAL (с RYW для пользователя)
            cart = await self.cart_store.get(
                f"cart:{user_id}",
                consistency="read_your_writes",
                session_id=user_id
            )

            # 3. Создание заказа — STRONG
            order = await self.orders_store.create(
                user_id=user_id,
                items=cart.items,
                consistency="serializable"
            )

            # 4. Очистка корзины — EVENTUAL (не критично)
            await self.cart_store.delete(
                f"cart:{user_id}",
                consistency="eventual"
            )

            # 5. Обновление рекомендаций — ASYNC/WEAK
            asyncio.create_task(
                self.recommendations_store.update_user_preferences(user_id, cart.items)
            )

            return order

        except Exception as e:
            # Откат резервирования
            await self.inventory_store.release(item_id, quantity)
            raise
```

### Компромиссы и trade-offs

```
                    Строгая                        Слабая
                 согласованность               согласованность
                      │                              │
Задержка:           Высокая ◄───────────────────► Низкая
Доступность:        Низкая  ◄───────────────────► Высокая
Сложность:          Высокая ◄───────────────────► Низкая
Корректность:       Гарантирована ◄─────────────► Вероятностная
Масштабируемость:   Ограниченная ◄──────────────► Отличная

CAP теорема:
┌─────────────────────────────────────────────────┐
│                                                 │
│        Consistency                              │
│            ▲                                    │
│           / \                                   │
│          /   \                                  │
│         /     \                                 │
│        /  CP   \  CA (только без разделений)    │
│       / systems \ systems                       │
│      /           \                              │
│     /─────────────\                             │
│    Availability ──── Partition tolerance        │
│         AP systems                              │
│                                                 │
└─────────────────────────────────────────────────┘

CP: etcd, ZooKeeper, Spanner
AP: Cassandra, DynamoDB, Riak
CA: Traditional RDBMS (single node)
```

---

## Заключение

### Ключевые выводы

1. **Нет универсального решения** — выбор паттерна зависит от требований бизнеса
2. **Разные данные — разные паттерны** — в одной системе можно комбинировать
3. **Понимай trade-offs** — CAP/PACELC теоремы определяют ограничения
4. **Думай о пользователе** — Read-Your-Writes часто достаточно для UX
5. **Мониторь задержку репликации** — eventual consistency требует отслеживания lag

### Чек-лист при проектировании

- [ ] Определить критичность данных (финансы, безопасность)
- [ ] Оценить допустимую задержку синхронизации
- [ ] Определить паттерны доступа (read-heavy, write-heavy)
- [ ] Рассмотреть географию пользователей
- [ ] Выбрать стратегию разрешения конфликтов
- [ ] Спланировать мониторинг согласованности
- [ ] Документировать гарантии для API consumers
