# Distributed Systems

## Определение

**Distributed Systems** (распределённые системы) — это архитектурный стиль, в котором компоненты системы размещены на разных компьютерах (узлах), связанных сетью, и координируют свои действия путём обмена сообщениями. Для пользователя такая система выглядит как единое целое, несмотря на физическое распределение.

## Схема архитектуры

```
                          ┌─────────────────────────────────────┐
                          │         LOAD BALANCER               │
                          └─────────────────┬───────────────────┘
                                            │
              ┌─────────────────────────────┼─────────────────────────────┐
              │                             │                             │
              ▼                             ▼                             ▼
    ┌─────────────────┐           ┌─────────────────┐           ┌─────────────────┐
    │   Node 1        │           │   Node 2        │           │   Node 3        │
    │   (Region A)    │           │   (Region B)    │           │   (Region C)    │
    │ ┌─────────────┐ │           │ ┌─────────────┐ │           │ ┌─────────────┐ │
    │ │   Service   │ │◀─────────▶│ │   Service   │ │◀─────────▶│ │   Service   │ │
    │ └─────────────┘ │           │ └─────────────┘ │           │ └─────────────┘ │
    │ ┌─────────────┐ │           │ ┌─────────────┐ │           │ ┌─────────────┐ │
    │ │   Cache     │ │           │ │   Cache     │ │           │ │   Cache     │ │
    │ └─────────────┘ │           │ └─────────────┘ │           │ └─────────────┘ │
    │ ┌─────────────┐ │           │ ┌─────────────┐ │           │ ┌─────────────┐ │
    │ │  Database   │ │◀─────────▶│ │  Database   │ │◀─────────▶│ │  Database   │ │
    │ │  (Replica)  │ │  Sync     │ │  (Replica)  │ │  Sync     │ │  (Replica)  │ │
    │ └─────────────┘ │           │ └─────────────┘ │           │ └─────────────┘ │
    └─────────────────┘           └─────────────────┘           └─────────────────┘
```

## CAP-теорема

```
                        Consistency
                           /\
                          /  \
                         /    \
                        /      \
                       /   CA   \
                      /          \
                     /____________\
                    /\            /\
                   /  \    CP   /  \
                  / AP \      /     \
                 /______\    /  CA   \
                          \/          \
              Availability ──────────── Partition Tolerance
```

**Теорема CAP** утверждает, что распределённая система не может одновременно обеспечить все три свойства:

- **Consistency** (согласованность) — все узлы видят одинаковые данные
- **Availability** (доступность) — каждый запрос получает ответ
- **Partition Tolerance** (устойчивость к разделению) — система работает при потере связи между узлами

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional, Dict, Any, List, Set
from datetime import datetime
import asyncio
import uuid
import hashlib


class CAPStrategy(Enum):
    """Стратегии CAP."""
    CP = "consistency_partition"    # Жертвуем доступностью
    AP = "availability_partition"   # Жертвуем согласованностью
    CA = "consistency_availability" # Жертвуем устойчивостью (не для распределённых)
```

## Консенсус и координация

### 1. Алгоритм Raft для выбора лидера

```python
from enum import Enum
from typing import Optional, Callable, Awaitable
import random


class NodeState(Enum):
    FOLLOWER = "follower"
    CANDIDATE = "candidate"
    LEADER = "leader"


@dataclass
class LogEntry:
    """Запись в журнале репликации."""
    term: int
    index: int
    command: Dict[str, Any]
    timestamp: datetime = field(default_factory=datetime.utcnow)


@dataclass
class VoteRequest:
    """Запрос на голосование."""
    term: int
    candidate_id: str
    last_log_index: int
    last_log_term: int


@dataclass
class VoteResponse:
    """Ответ на запрос голосования."""
    term: int
    vote_granted: bool


@dataclass
class AppendEntriesRequest:
    """Запрос на репликацию записей."""
    term: int
    leader_id: str
    prev_log_index: int
    prev_log_term: int
    entries: List[LogEntry]
    leader_commit: int


@dataclass
class AppendEntriesResponse:
    """Ответ на запрос репликации."""
    term: int
    success: bool
    match_index: int


class RaftNode:
    """Узел с реализацией алгоритма Raft."""

    def __init__(
        self,
        node_id: str,
        peers: List[str],
        send_func: Callable[[str, Any], Awaitable[Any]]
    ):
        self.node_id = node_id
        self.peers = peers
        self.send = send_func

        # Персистентное состояние
        self.current_term = 0
        self.voted_for: Optional[str] = None
        self.log: List[LogEntry] = []

        # Волатильное состояние
        self.state = NodeState.FOLLOWER
        self.commit_index = 0
        self.last_applied = 0

        # Состояние лидера
        self.next_index: Dict[str, int] = {}
        self.match_index: Dict[str, int] = {}

        # Таймеры
        self.election_timeout = random.uniform(150, 300) / 1000  # 150-300ms
        self.heartbeat_interval = 50 / 1000  # 50ms
        self._election_timer: Optional[asyncio.Task] = None
        self._heartbeat_timer: Optional[asyncio.Task] = None

    async def start(self) -> None:
        """Запускает узел."""
        self._reset_election_timer()

    async def _reset_election_timer(self) -> None:
        """Сбрасывает таймер выборов."""
        if self._election_timer:
            self._election_timer.cancel()

        async def election_timeout():
            await asyncio.sleep(self.election_timeout)
            await self._start_election()

        self._election_timer = asyncio.create_task(election_timeout())

    async def _start_election(self) -> None:
        """Начинает выборы лидера."""
        self.state = NodeState.CANDIDATE
        self.current_term += 1
        self.voted_for = self.node_id

        votes_received = 1  # Голос за себя

        last_log_index = len(self.log) - 1 if self.log else -1
        last_log_term = self.log[-1].term if self.log else 0

        request = VoteRequest(
            term=self.current_term,
            candidate_id=self.node_id,
            last_log_index=last_log_index,
            last_log_term=last_log_term
        )

        # Запрашиваем голоса у всех узлов
        async def request_vote(peer: str) -> int:
            try:
                response: VoteResponse = await self.send(peer, request)
                if response.vote_granted:
                    return 1
            except Exception:
                pass
            return 0

        tasks = [request_vote(peer) for peer in self.peers]
        results = await asyncio.gather(*tasks)
        votes_received += sum(results)

        # Проверяем, набрали ли большинство
        majority = (len(self.peers) + 1) // 2 + 1
        if votes_received >= majority and self.state == NodeState.CANDIDATE:
            await self._become_leader()
        else:
            self.state = NodeState.FOLLOWER
            await self._reset_election_timer()

    async def _become_leader(self) -> None:
        """Становимся лидером."""
        self.state = NodeState.LEADER

        # Инициализируем индексы для репликации
        last_log_index = len(self.log)
        for peer in self.peers:
            self.next_index[peer] = last_log_index
            self.match_index[peer] = 0

        # Запускаем heartbeat
        await self._start_heartbeat()

    async def _start_heartbeat(self) -> None:
        """Запускает периодическую отправку heartbeat."""
        async def heartbeat_loop():
            while self.state == NodeState.LEADER:
                await self._send_heartbeats()
                await asyncio.sleep(self.heartbeat_interval)

        self._heartbeat_timer = asyncio.create_task(heartbeat_loop())

    async def _send_heartbeats(self) -> None:
        """Отправляет heartbeat всем узлам."""
        for peer in self.peers:
            await self._send_append_entries(peer)

    async def _send_append_entries(self, peer: str) -> None:
        """Отправляет AppendEntries конкретному узлу."""
        next_idx = self.next_index.get(peer, 0)
        prev_log_index = next_idx - 1
        prev_log_term = self.log[prev_log_index].term if prev_log_index >= 0 and self.log else 0

        entries = self.log[next_idx:] if next_idx < len(self.log) else []

        request = AppendEntriesRequest(
            term=self.current_term,
            leader_id=self.node_id,
            prev_log_index=prev_log_index,
            prev_log_term=prev_log_term,
            entries=entries,
            leader_commit=self.commit_index
        )

        try:
            response: AppendEntriesResponse = await self.send(peer, request)

            if response.success:
                self.next_index[peer] = response.match_index + 1
                self.match_index[peer] = response.match_index
                await self._update_commit_index()
            else:
                # Уменьшаем next_index и повторяем
                self.next_index[peer] = max(0, self.next_index[peer] - 1)
        except Exception:
            pass

    async def _update_commit_index(self) -> None:
        """Обновляет commit_index на основе match_index."""
        for n in range(len(self.log) - 1, self.commit_index, -1):
            # Считаем сколько узлов имеют эту запись
            count = 1  # Лидер
            for peer in self.peers:
                if self.match_index.get(peer, 0) >= n:
                    count += 1

            majority = (len(self.peers) + 1) // 2 + 1
            if count >= majority and self.log[n].term == self.current_term:
                self.commit_index = n
                break

    async def handle_vote_request(self, request: VoteRequest) -> VoteResponse:
        """Обрабатывает запрос на голосование."""
        if request.term < self.current_term:
            return VoteResponse(term=self.current_term, vote_granted=False)

        if request.term > self.current_term:
            self.current_term = request.term
            self.voted_for = None
            self.state = NodeState.FOLLOWER

        # Проверяем, можем ли голосовать
        vote_granted = False
        if self.voted_for is None or self.voted_for == request.candidate_id:
            # Проверяем актуальность лога кандидата
            last_log_index = len(self.log) - 1 if self.log else -1
            last_log_term = self.log[-1].term if self.log else 0

            if (request.last_log_term > last_log_term or
                (request.last_log_term == last_log_term and
                 request.last_log_index >= last_log_index)):
                vote_granted = True
                self.voted_for = request.candidate_id
                await self._reset_election_timer()

        return VoteResponse(term=self.current_term, vote_granted=vote_granted)

    async def handle_append_entries(
        self,
        request: AppendEntriesRequest
    ) -> AppendEntriesResponse:
        """Обрабатывает запрос на репликацию."""
        if request.term < self.current_term:
            return AppendEntriesResponse(
                term=self.current_term,
                success=False,
                match_index=0
            )

        await self._reset_election_timer()

        if request.term > self.current_term:
            self.current_term = request.term
            self.state = NodeState.FOLLOWER
            self.voted_for = None

        # Проверяем prev_log
        if request.prev_log_index >= 0:
            if len(self.log) <= request.prev_log_index:
                return AppendEntriesResponse(
                    term=self.current_term,
                    success=False,
                    match_index=len(self.log) - 1
                )
            if self.log[request.prev_log_index].term != request.prev_log_term:
                # Удаляем конфликтующие записи
                self.log = self.log[:request.prev_log_index]
                return AppendEntriesResponse(
                    term=self.current_term,
                    success=False,
                    match_index=len(self.log) - 1
                )

        # Добавляем новые записи
        for entry in request.entries:
            if entry.index < len(self.log):
                if self.log[entry.index].term != entry.term:
                    self.log = self.log[:entry.index]
                    self.log.append(entry)
            else:
                self.log.append(entry)

        # Обновляем commit_index
        if request.leader_commit > self.commit_index:
            self.commit_index = min(request.leader_commit, len(self.log) - 1)

        return AppendEntriesResponse(
            term=self.current_term,
            success=True,
            match_index=len(self.log) - 1
        )

    async def propose(self, command: Dict[str, Any]) -> bool:
        """Предлагает команду для репликации (только лидер)."""
        if self.state != NodeState.LEADER:
            return False

        entry = LogEntry(
            term=self.current_term,
            index=len(self.log),
            command=command
        )
        self.log.append(entry)

        # Реплицируем на все узлы
        for peer in self.peers:
            asyncio.create_task(self._send_append_entries(peer))

        return True
```

### 2. Consistent Hashing для распределения данных

```python
import bisect
from typing import Generic, TypeVar

K = TypeVar("K")
V = TypeVar("V")


class ConsistentHash(Generic[K]):
    """Consistent Hashing для распределения данных."""

    def __init__(self, virtual_nodes: int = 150):
        self.virtual_nodes = virtual_nodes
        self._ring: List[int] = []
        self._node_map: Dict[int, str] = {}  # hash -> node_id
        self._nodes: Set[str] = set()

    def _hash(self, key: str) -> int:
        """Вычисляет хеш ключа."""
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node_id: str) -> None:
        """Добавляет узел в кольцо."""
        if node_id in self._nodes:
            return

        self._nodes.add(node_id)

        # Добавляем виртуальные узлы
        for i in range(self.virtual_nodes):
            virtual_key = f"{node_id}:{i}"
            hash_val = self._hash(virtual_key)

            # Вставляем в отсортированное кольцо
            bisect.insort(self._ring, hash_val)
            self._node_map[hash_val] = node_id

    def remove_node(self, node_id: str) -> None:
        """Удаляет узел из кольца."""
        if node_id not in self._nodes:
            return

        self._nodes.remove(node_id)

        # Удаляем виртуальные узлы
        for i in range(self.virtual_nodes):
            virtual_key = f"{node_id}:{i}"
            hash_val = self._hash(virtual_key)

            self._ring.remove(hash_val)
            del self._node_map[hash_val]

    def get_node(self, key: str) -> Optional[str]:
        """Находит узел для ключа."""
        if not self._ring:
            return None

        hash_val = self._hash(key)

        # Находим ближайший узел по часовой стрелке
        idx = bisect.bisect_right(self._ring, hash_val)
        if idx == len(self._ring):
            idx = 0

        return self._node_map[self._ring[idx]]

    def get_nodes(self, key: str, count: int = 3) -> List[str]:
        """Находит несколько узлов для ключа (для репликации)."""
        if not self._ring:
            return []

        result: List[str] = []
        hash_val = self._hash(key)

        idx = bisect.bisect_right(self._ring, hash_val)

        while len(result) < count and len(result) < len(self._nodes):
            if idx >= len(self._ring):
                idx = 0

            node_id = self._node_map[self._ring[idx]]
            if node_id not in result:
                result.append(node_id)

            idx += 1

        return result


class DistributedKeyValueStore:
    """Распределённое Key-Value хранилище."""

    def __init__(self, replication_factor: int = 3):
        self.replication_factor = replication_factor
        self.hash_ring = ConsistentHash[str]()
        self.local_data: Dict[str, Dict[str, Any]] = {}  # node_id -> data
        self.node_id: Optional[str] = None

    def join_cluster(self, node_id: str, existing_nodes: List[str]) -> None:
        """Присоединяется к кластеру."""
        self.node_id = node_id
        self.local_data[node_id] = {}

        # Добавляем все узлы в кольцо
        for existing in existing_nodes:
            self.hash_ring.add_node(existing)
        self.hash_ring.add_node(node_id)

    async def put(self, key: str, value: Any) -> bool:
        """Сохраняет значение с репликацией."""
        target_nodes = self.hash_ring.get_nodes(key, self.replication_factor)

        success_count = 0
        for node_id in target_nodes:
            try:
                if node_id == self.node_id:
                    self.local_data[node_id][key] = value
                    success_count += 1
                else:
                    # Отправляем на удалённый узел
                    # await self._remote_put(node_id, key, value)
                    success_count += 1
            except Exception:
                pass

        # Кворум записи: W > N/2
        write_quorum = self.replication_factor // 2 + 1
        return success_count >= write_quorum

    async def get(self, key: str) -> Optional[Any]:
        """Получает значение с кворумом чтения."""
        target_nodes = self.hash_ring.get_nodes(key, self.replication_factor)

        responses: List[tuple] = []  # (value, version)

        for node_id in target_nodes:
            try:
                if node_id == self.node_id:
                    if key in self.local_data.get(node_id, {}):
                        responses.append((self.local_data[node_id][key], 0))
                else:
                    # Получаем с удалённого узла
                    # response = await self._remote_get(node_id, key)
                    pass
            except Exception:
                pass

        if not responses:
            return None

        # Кворум чтения: R > N/2
        read_quorum = self.replication_factor // 2 + 1
        if len(responses) >= read_quorum:
            # Возвращаем значение с максимальной версией
            return max(responses, key=lambda x: x[1])[0]

        return None
```

## Распределённые транзакции

### Two-Phase Commit (2PC)

```
┌─────────────┐          ┌─────────────┐          ┌─────────────┐
│ Coordinator │          │ Participant │          │ Participant │
│             │          │      A      │          │      B      │
└──────┬──────┘          └──────┬──────┘          └──────┬──────┘
       │                        │                        │
       │  Phase 1: Prepare      │                        │
       │───────────────────────▶│                        │
       │                        │                        │
       │◀───── VOTE_YES ────────│                        │
       │                        │                        │
       │  Prepare               │                        │
       │────────────────────────────────────────────────▶│
       │                        │                        │
       │◀─────────────────────────────── VOTE_YES ───────│
       │                        │                        │
       │  Phase 2: Commit       │                        │
       │───────────────────────▶│                        │
       │                        │                        │
       │◀────── ACK ────────────│                        │
       │                        │                        │
       │  Commit                │                        │
       │────────────────────────────────────────────────▶│
       │                        │                        │
       │◀──────────────────────────────── ACK ──────────│
```

```python
from enum import Enum
from dataclasses import dataclass


class TransactionState(Enum):
    INITIATED = "initiated"
    PREPARING = "preparing"
    PREPARED = "prepared"
    COMMITTING = "committing"
    COMMITTED = "committed"
    ABORTING = "aborting"
    ABORTED = "aborted"


class VoteResult(Enum):
    YES = "yes"
    NO = "no"
    TIMEOUT = "timeout"


@dataclass
class TransactionLog:
    """Журнал транзакции для восстановления."""
    transaction_id: str
    state: TransactionState
    participants: List[str]
    votes: Dict[str, VoteResult]
    timestamp: datetime = field(default_factory=datetime.utcnow)


class TwoPhaseCommitCoordinator:
    """Координатор протокола 2PC."""

    def __init__(
        self,
        coordinator_id: str,
        send_func: Callable[[str, Any], Awaitable[Any]]
    ):
        self.coordinator_id = coordinator_id
        self.send = send_func
        self.transaction_log: Dict[str, TransactionLog] = {}

    async def execute_transaction(
        self,
        transaction_id: str,
        participants: List[str],
        operations: Dict[str, Dict[str, Any]]
    ) -> bool:
        """Выполняет распределённую транзакцию."""
        # Создаём запись в журнале
        log = TransactionLog(
            transaction_id=transaction_id,
            state=TransactionState.INITIATED,
            participants=participants,
            votes={}
        )
        self.transaction_log[transaction_id] = log

        # Phase 1: Prepare
        log.state = TransactionState.PREPARING
        all_voted_yes = await self._prepare_phase(transaction_id, participants, operations)

        if all_voted_yes:
            # Phase 2: Commit
            log.state = TransactionState.COMMITTING
            await self._commit_phase(transaction_id, participants)
            log.state = TransactionState.COMMITTED
            return True
        else:
            # Phase 2: Abort
            log.state = TransactionState.ABORTING
            await self._abort_phase(transaction_id, participants)
            log.state = TransactionState.ABORTED
            return False

    async def _prepare_phase(
        self,
        transaction_id: str,
        participants: List[str],
        operations: Dict[str, Dict[str, Any]]
    ) -> bool:
        """Фаза подготовки: запрашиваем голоса."""
        log = self.transaction_log[transaction_id]

        async def prepare_participant(participant: str) -> VoteResult:
            try:
                request = {
                    "type": "prepare",
                    "transaction_id": transaction_id,
                    "operation": operations.get(participant, {})
                }
                response = await asyncio.wait_for(
                    self.send(participant, request),
                    timeout=5.0
                )
                return VoteResult.YES if response.get("vote") == "yes" else VoteResult.NO
            except asyncio.TimeoutError:
                return VoteResult.TIMEOUT
            except Exception:
                return VoteResult.NO

        tasks = [prepare_participant(p) for p in participants]
        results = await asyncio.gather(*tasks)

        for participant, vote in zip(participants, results):
            log.votes[participant] = vote

        log.state = TransactionState.PREPARED
        return all(v == VoteResult.YES for v in results)

    async def _commit_phase(
        self,
        transaction_id: str,
        participants: List[str]
    ) -> None:
        """Фаза фиксации: отправляем commit всем."""
        async def commit_participant(participant: str) -> bool:
            try:
                request = {
                    "type": "commit",
                    "transaction_id": transaction_id
                }
                await self.send(participant, request)
                return True
            except Exception:
                # Будем повторять до успеха
                return False

        # Повторяем пока все не подтвердят
        pending = set(participants)
        while pending:
            tasks = [commit_participant(p) for p in pending]
            results = await asyncio.gather(*tasks)

            for participant, success in zip(list(pending), results):
                if success:
                    pending.remove(participant)

            if pending:
                await asyncio.sleep(1.0)  # Ждём перед повтором

    async def _abort_phase(
        self,
        transaction_id: str,
        participants: List[str]
    ) -> None:
        """Фаза отката: отправляем abort всем."""
        async def abort_participant(participant: str) -> None:
            try:
                request = {
                    "type": "abort",
                    "transaction_id": transaction_id
                }
                await self.send(participant, request)
            except Exception:
                pass

        tasks = [abort_participant(p) for p in participants]
        await asyncio.gather(*tasks)


class TwoPhaseCommitParticipant:
    """Участник протокола 2PC."""

    def __init__(self, participant_id: str):
        self.participant_id = participant_id
        self.pending_transactions: Dict[str, Dict[str, Any]] = {}
        self.prepared_data: Dict[str, Any] = {}

    async def handle_prepare(
        self,
        transaction_id: str,
        operation: Dict[str, Any]
    ) -> Dict[str, Any]:
        """Обрабатывает запрос prepare."""
        try:
            # Проверяем, можем ли выполнить операцию
            # Блокируем ресурсы
            # Записываем в журнал
            self.pending_transactions[transaction_id] = operation
            self.prepared_data[transaction_id] = self._prepare_operation(operation)

            return {"vote": "yes"}
        except Exception as e:
            return {"vote": "no", "reason": str(e)}

    def _prepare_operation(self, operation: Dict[str, Any]) -> Any:
        """Подготавливает операцию (проверяет и блокирует ресурсы)."""
        # Реализация зависит от типа операции
        return operation

    async def handle_commit(self, transaction_id: str) -> Dict[str, Any]:
        """Обрабатывает запрос commit."""
        if transaction_id not in self.pending_transactions:
            return {"status": "unknown"}

        try:
            # Применяем подготовленные изменения
            prepared = self.prepared_data.pop(transaction_id)
            self._apply_operation(prepared)

            del self.pending_transactions[transaction_id]
            return {"status": "committed"}
        except Exception as e:
            return {"status": "error", "reason": str(e)}

    def _apply_operation(self, prepared: Any) -> None:
        """Применяет операцию."""
        pass

    async def handle_abort(self, transaction_id: str) -> Dict[str, Any]:
        """Обрабатывает запрос abort."""
        if transaction_id in self.pending_transactions:
            # Откатываем подготовленные изменения
            self.prepared_data.pop(transaction_id, None)
            del self.pending_transactions[transaction_id]

        return {"status": "aborted"}
```

### Saga Pattern для распределённых транзакций

```python
from abc import ABC, abstractmethod


@dataclass
class SagaStep:
    """Шаг саги."""
    name: str
    action: Callable[..., Awaitable[Any]]
    compensation: Callable[..., Awaitable[Any]]


class SagaOrchestrator:
    """Оркестратор саги."""

    def __init__(self, saga_id: str):
        self.saga_id = saga_id
        self.steps: List[SagaStep] = []
        self.completed_steps: List[str] = []
        self.results: Dict[str, Any] = {}

    def add_step(
        self,
        name: str,
        action: Callable[..., Awaitable[Any]],
        compensation: Callable[..., Awaitable[Any]]
    ) -> "SagaOrchestrator":
        """Добавляет шаг в сагу."""
        self.steps.append(SagaStep(name=name, action=action, compensation=compensation))
        return self

    async def execute(self, context: Dict[str, Any]) -> tuple[bool, Dict[str, Any]]:
        """Выполняет сагу."""
        for step in self.steps:
            try:
                result = await step.action(context, self.results)
                self.results[step.name] = result
                self.completed_steps.append(step.name)
            except Exception as e:
                # Откатываем выполненные шаги
                await self._compensate()
                return False, {"error": str(e), "failed_step": step.name}

        return True, self.results

    async def _compensate(self) -> None:
        """Выполняет компенсирующие действия в обратном порядке."""
        for step_name in reversed(self.completed_steps):
            step = next(s for s in self.steps if s.name == step_name)
            try:
                await step.compensation(self.results.get(step_name))
            except Exception as e:
                # Логируем ошибку компенсации
                print(f"Compensation failed for {step_name}: {e}")


# Пример использования саги для заказа
async def order_saga_example():
    saga = SagaOrchestrator("order-123")

    # Шаг 1: Резервирование товара
    async def reserve_inventory(ctx, results):
        # Резервируем товар
        return {"reservation_id": "res-123", "items": ctx["items"]}

    async def release_inventory(result):
        # Отменяем резервирование
        print(f"Releasing reservation: {result['reservation_id']}")

    # Шаг 2: Списание со счёта
    async def charge_customer(ctx, results):
        # Списываем деньги
        return {"payment_id": "pay-456", "amount": ctx["amount"]}

    async def refund_customer(result):
        # Возвращаем деньги
        print(f"Refunding payment: {result['payment_id']}")

    # Шаг 3: Создание доставки
    async def create_shipment(ctx, results):
        # Создаём заказ на доставку
        return {"shipment_id": "ship-789"}

    async def cancel_shipment(result):
        # Отменяем доставку
        print(f"Canceling shipment: {result['shipment_id']}")

    saga.add_step("reserve_inventory", reserve_inventory, release_inventory)
    saga.add_step("charge_customer", charge_customer, refund_customer)
    saga.add_step("create_shipment", create_shipment, cancel_shipment)

    context = {
        "order_id": "order-123",
        "items": [{"sku": "LAPTOP-1", "quantity": 1}],
        "amount": 1500.00
    }

    success, result = await saga.execute(context)
    print(f"Saga completed: {success}, Result: {result}")
```

## Service Discovery

```python
from dataclasses import dataclass, field
from typing import Callable, Optional
from datetime import datetime, timedelta


@dataclass
class ServiceInstance:
    """Экземпляр сервиса."""
    service_id: str
    service_name: str
    host: str
    port: int
    health_check_url: str
    metadata: Dict[str, str] = field(default_factory=dict)
    status: str = "UP"
    last_heartbeat: datetime = field(default_factory=datetime.utcnow)


class ServiceRegistry:
    """Реестр сервисов."""

    def __init__(self, heartbeat_timeout: int = 30):
        self.services: Dict[str, Dict[str, ServiceInstance]] = {}  # name -> id -> instance
        self.heartbeat_timeout = timedelta(seconds=heartbeat_timeout)

    def register(self, instance: ServiceInstance) -> None:
        """Регистрирует экземпляр сервиса."""
        if instance.service_name not in self.services:
            self.services[instance.service_name] = {}

        self.services[instance.service_name][instance.service_id] = instance

    def deregister(self, service_name: str, service_id: str) -> None:
        """Отменяет регистрацию экземпляра."""
        if service_name in self.services:
            self.services[service_name].pop(service_id, None)

    def heartbeat(self, service_name: str, service_id: str) -> bool:
        """Обновляет heartbeat для экземпляра."""
        if service_name in self.services and service_id in self.services[service_name]:
            instance = self.services[service_name][service_id]
            instance.last_heartbeat = datetime.utcnow()
            return True
        return False

    def get_instances(self, service_name: str) -> List[ServiceInstance]:
        """Возвращает все активные экземпляры сервиса."""
        self._cleanup_expired()

        if service_name not in self.services:
            return []

        return [
            instance for instance in self.services[service_name].values()
            if instance.status == "UP"
        ]

    def _cleanup_expired(self) -> None:
        """Удаляет экземпляры с истекшим heartbeat."""
        cutoff = datetime.utcnow() - self.heartbeat_timeout

        for service_name in list(self.services.keys()):
            expired = [
                service_id for service_id, instance
                in self.services[service_name].items()
                if instance.last_heartbeat < cutoff
            ]
            for service_id in expired:
                del self.services[service_name][service_id]


class LoadBalancer:
    """Балансировщик нагрузки."""

    def __init__(self, registry: ServiceRegistry):
        self.registry = registry
        self._round_robin_index: Dict[str, int] = {}

    def get_instance(
        self,
        service_name: str,
        strategy: str = "round_robin"
    ) -> Optional[ServiceInstance]:
        """Выбирает экземпляр сервиса."""
        instances = self.registry.get_instances(service_name)
        if not instances:
            return None

        if strategy == "round_robin":
            return self._round_robin(service_name, instances)
        elif strategy == "random":
            return random.choice(instances)
        elif strategy == "least_connections":
            return self._least_connections(instances)
        else:
            return instances[0]

    def _round_robin(
        self,
        service_name: str,
        instances: List[ServiceInstance]
    ) -> ServiceInstance:
        """Round-robin выбор."""
        if service_name not in self._round_robin_index:
            self._round_robin_index[service_name] = 0

        idx = self._round_robin_index[service_name] % len(instances)
        self._round_robin_index[service_name] = idx + 1

        return instances[idx]

    def _least_connections(
        self,
        instances: List[ServiceInstance]
    ) -> ServiceInstance:
        """Выбор с наименьшим числом соединений."""
        # В реальности нужно отслеживать активные соединения
        return random.choice(instances)
```

## Circuit Breaker Pattern

```python
from enum import Enum
from dataclasses import dataclass


class CircuitState(Enum):
    CLOSED = "closed"      # Нормальная работа
    OPEN = "open"          # Отказ, запросы блокируются
    HALF_OPEN = "half_open"  # Пробуем восстановиться


@dataclass
class CircuitBreakerConfig:
    """Конфигурация Circuit Breaker."""
    failure_threshold: int = 5      # Порог ошибок для открытия
    success_threshold: int = 3      # Порог успехов для закрытия
    timeout: float = 30.0           # Время до перехода в half-open
    half_open_max_calls: int = 3    # Максимум вызовов в half-open


class CircuitBreaker:
    """Реализация паттерна Circuit Breaker."""

    def __init__(self, config: CircuitBreakerConfig):
        self.config = config
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time: Optional[datetime] = None
        self.half_open_calls = 0

    def can_execute(self) -> bool:
        """Проверяет, можно ли выполнить запрос."""
        if self.state == CircuitState.CLOSED:
            return True

        if self.state == CircuitState.OPEN:
            # Проверяем, прошёл ли timeout
            if self.last_failure_time:
                elapsed = (datetime.utcnow() - self.last_failure_time).total_seconds()
                if elapsed >= self.config.timeout:
                    self._transition_to_half_open()
                    return True
            return False

        if self.state == CircuitState.HALF_OPEN:
            return self.half_open_calls < self.config.half_open_max_calls

        return False

    def record_success(self) -> None:
        """Записывает успешный вызов."""
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.config.success_threshold:
                self._transition_to_closed()
        elif self.state == CircuitState.CLOSED:
            self.failure_count = 0

    def record_failure(self) -> None:
        """Записывает неудачный вызов."""
        self.failure_count += 1
        self.last_failure_time = datetime.utcnow()

        if self.state == CircuitState.HALF_OPEN:
            self._transition_to_open()
        elif self.state == CircuitState.CLOSED:
            if self.failure_count >= self.config.failure_threshold:
                self._transition_to_open()

    def _transition_to_open(self) -> None:
        """Переход в состояние OPEN."""
        self.state = CircuitState.OPEN
        self.success_count = 0

    def _transition_to_half_open(self) -> None:
        """Переход в состояние HALF_OPEN."""
        self.state = CircuitState.HALF_OPEN
        self.half_open_calls = 0
        self.success_count = 0

    def _transition_to_closed(self) -> None:
        """Переход в состояние CLOSED."""
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.half_open_calls = 0

    async def execute(
        self,
        func: Callable[..., Awaitable[Any]],
        *args,
        **kwargs
    ) -> Any:
        """Выполняет функцию через Circuit Breaker."""
        if not self.can_execute():
            raise CircuitBreakerOpenError(
                f"Circuit breaker is {self.state.value}"
            )

        if self.state == CircuitState.HALF_OPEN:
            self.half_open_calls += 1

        try:
            result = await func(*args, **kwargs)
            self.record_success()
            return result
        except Exception as e:
            self.record_failure()
            raise


class CircuitBreakerOpenError(Exception):
    """Ошибка: Circuit Breaker открыт."""
    pass
```

## Полный пример распределённой системы

```python
import asyncio


async def distributed_system_example():
    """Пример распределённой системы."""

    # Создаём реестр сервисов
    registry = ServiceRegistry(heartbeat_timeout=30)

    # Регистрируем экземпляры
    for i in range(3):
        instance = ServiceInstance(
            service_id=f"order-service-{i}",
            service_name="order-service",
            host=f"192.168.1.{10 + i}",
            port=8080,
            health_check_url="/health"
        )
        registry.register(instance)

    # Создаём балансировщик
    load_balancer = LoadBalancer(registry)

    # Создаём Circuit Breaker
    circuit_breaker = CircuitBreaker(CircuitBreakerConfig(
        failure_threshold=5,
        success_threshold=3,
        timeout=30.0
    ))

    # Функция вызова сервиса
    async def call_order_service(order_data: Dict[str, Any]) -> Dict[str, Any]:
        instance = load_balancer.get_instance("order-service", "round_robin")
        if not instance:
            raise Exception("No available instances")

        # Имитация вызова
        print(f"Calling {instance.host}:{instance.port}")
        return {"status": "success", "order_id": "ORD-123"}

    # Выполняем запрос через Circuit Breaker
    try:
        result = await circuit_breaker.execute(
            call_order_service,
            {"items": [{"sku": "LAPTOP-1"}]}
        )
        print(f"Result: {result}")
    except CircuitBreakerOpenError as e:
        print(f"Circuit breaker open: {e}")
    except Exception as e:
        print(f"Error: {e}")


if __name__ == "__main__":
    asyncio.run(distributed_system_example())
```

## Преимущества распределённых систем

1. **Масштабируемость** — горизонтальное масштабирование добавлением узлов
2. **Отказоустойчивость** — система продолжает работать при сбое узлов
3. **Географическое распределение** — размещение ближе к пользователям
4. **Изоляция сбоев** — проблема в одном компоненте не затрагивает другие
5. **Гибкость технологий** — разные компоненты могут использовать разные стеки

## Недостатки

1. **Сложность** — распределённые алгоритмы сложнее понимать и отлаживать
2. **Сетевые проблемы** — latency, partition, потери пакетов
3. **Консистентность** — сложно обеспечить согласованность данных
4. **Операционная сложность** — мониторинг, деплой, логирование
5. **Стоимость** — дополнительные ресурсы на координацию

## Когда использовать

**Подходит для:**
- Высоконагруженных систем
- Систем с требованиями высокой доступности
- Географически распределённых приложений
- Систем с большими объёмами данных

**Не подходит для:**
- Простых приложений без требований к масштабированию
- Систем с жёсткими требованиями к консистентности
- Приложений с ограниченным бюджетом на инфраструктуру

## Ключевые паттерны

| Паттерн | Назначение |
|---------|------------|
| **Raft/Paxos** | Консенсус и выбор лидера |
| **Consistent Hashing** | Распределение данных |
| **2PC/3PC** | Распределённые транзакции |
| **Saga** | Долгие бизнес-транзакции |
| **Circuit Breaker** | Защита от каскадных сбоев |
| **Service Discovery** | Обнаружение сервисов |
| **Bulkhead** | Изоляция сбоев |
| **Retry with Backoff** | Восстановление после ошибок |

## Лучшие практики

1. **Design for Failure** — предполагайте, что всё может сломаться
2. **Idempotent Operations** — операции должны быть идемпотентными
3. **Timeouts Everywhere** — устанавливайте таймауты на всех уровнях
4. **Graceful Degradation** — деградируйте красиво при проблемах
5. **Observability** — логи, метрики, трассировка во всех компонентах
6. **Eventual Consistency** — примите eventual consistency где возможно
7. **Test Chaos** — тестируйте сбои (Chaos Engineering)
