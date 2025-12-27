# Controller-Responder (Master-Slave)

## Определение

**Controller-Responder** (ранее известный как Master-Slave) — это архитектурный паттерн, в котором один компонент (Controller/Master) управляет работой одного или нескольких подчинённых компонентов (Responders/Slaves). Controller распределяет задачи, координирует работу и агрегирует результаты, в то время как Responders выполняют фактическую работу.

> **Примечание**: Термин "Master-Slave" постепенно заменяется на "Controller-Responder", "Primary-Replica", "Leader-Follower" в современной документации по соображениям инклюзивности.

```
┌─────────────────────────────────────────────────────────────────────┐
│                           CONTROLLER                                 │
│                          (Primary/Master)                            │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │  • Принимает клиентские запросы                                ││
│  │  • Распределяет задачи между Responders                        ││
│  │  • Координирует выполнение                                     ││
│  │  • Агрегирует результаты                                       ││
│  │  • Обрабатывает отказы Responders                             ││
│  └─────────────────────────────────────────────────────────────────┘│
└────────────────────────────────┬────────────────────────────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
              ▼                  ▼                  ▼
    ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
    │   RESPONDER 1   │ │   RESPONDER 2   │ │   RESPONDER 3   │
    │   (Replica)     │ │   (Replica)     │ │   (Replica)     │
    │                 │ │                 │ │                 │
    │ • Выполняет     │ │ • Выполняет     │ │ • Выполняет     │
    │   задачи        │ │   задачи        │ │   задачи        │
    │ • Возвращает    │ │ • Возвращает    │ │ • Возвращает    │
    │   результат     │ │   результат     │ │   результат     │
    └─────────────────┘ └─────────────────┘ └─────────────────┘
```

## Ключевые характеристики

### 1. Централизованное управление

Controller полностью контролирует поток работы:

```python
from abc import ABC, abstractmethod
from typing import List, Any
import threading
import queue

class Responder(ABC):
    """Абстрактный Responder (Worker)"""

    @abstractmethod
    def execute(self, task: Any) -> Any:
        pass

    @abstractmethod
    def is_healthy(self) -> bool:
        pass

class Controller:
    """Центральный координатор работы"""

    def __init__(self, responders: List[Responder]):
        self.responders = responders
        self.task_queue = queue.Queue()
        self.results = {}
        self._lock = threading.Lock()

    def submit_task(self, task_id: str, task: Any):
        """Принимает задачу от клиента"""
        self.task_queue.put((task_id, task))

    def distribute_tasks(self):
        """Распределяет задачи между доступными Responders"""
        while not self.task_queue.empty():
            task_id, task = self.task_queue.get()

            # Находим свободный и здоровый Responder
            responder = self._select_responder()
            if responder:
                result = responder.execute(task)
                with self._lock:
                    self.results[task_id] = result

    def _select_responder(self) -> Responder:
        """Выбирает подходящий Responder (Round-Robin, Least-Loaded, и т.д.)"""
        for responder in self.responders:
            if responder.is_healthy():
                return responder
        return None

    def get_result(self, task_id: str) -> Any:
        """Возвращает результат клиенту"""
        with self._lock:
            return self.results.get(task_id)
```

### 2. Репликация данных

В контексте баз данных — Controller (Primary) обрабатывает запись, Responders (Replicas) — чтение:

```python
from dataclasses import dataclass
from typing import Dict, List, Optional
import time

@dataclass
class WriteOperation:
    table: str
    data: dict
    timestamp: float

class PrimaryDatabase:
    """Primary (Controller) — обрабатывает все записи"""

    def __init__(self):
        self.data: Dict[str, dict] = {}
        self.replicas: List['ReplicaDatabase'] = []
        self.write_log: List[WriteOperation] = []

    def add_replica(self, replica: 'ReplicaDatabase'):
        self.replicas.append(replica)

    def write(self, key: str, value: dict):
        """Запись только через Primary"""
        operation = WriteOperation(
            table="main",
            data={key: value},
            timestamp=time.time()
        )

        # Записываем локально
        self.data[key] = value
        self.write_log.append(operation)

        # Реплицируем на все Replicas
        self._replicate(operation)

    def _replicate(self, operation: WriteOperation):
        """Синхронная репликация (можно сделать асинхронной)"""
        for replica in self.replicas:
            replica.apply_operation(operation)

    def read(self, key: str) -> Optional[dict]:
        """Primary тоже может читать"""
        return self.data.get(key)


class ReplicaDatabase:
    """Replica (Responder) — обрабатывает чтение"""

    def __init__(self, replica_id: str):
        self.replica_id = replica_id
        self.data: Dict[str, dict] = {}
        self.last_applied_timestamp: float = 0

    def apply_operation(self, operation: WriteOperation):
        """Применяет операцию записи от Primary"""
        if operation.timestamp > self.last_applied_timestamp:
            self.data.update(operation.data)
            self.last_applied_timestamp = operation.timestamp

    def read(self, key: str) -> Optional[dict]:
        """Чтение из реплики"""
        return self.data.get(key)


# Использование
primary = PrimaryDatabase()
replica1 = ReplicaDatabase("replica-1")
replica2 = ReplicaDatabase("replica-2")

primary.add_replica(replica1)
primary.add_replica(replica2)

# Записи идут через Primary
primary.write("user:1", {"name": "Alice", "email": "alice@example.com"})

# Чтения можно распределять по Replicas
print(replica1.read("user:1"))  # {"name": "Alice", ...}
print(replica2.read("user:1"))  # {"name": "Alice", ...}
```

### 3. Распределённые вычисления (MapReduce)

```python
from typing import List, Callable, TypeVar, Iterable
from concurrent.futures import ThreadPoolExecutor, as_completed
import itertools

T = TypeVar('T')
R = TypeVar('R')

class MapReduceController:
    """Controller для MapReduce вычислений"""

    def __init__(self, num_workers: int = 4):
        self.num_workers = num_workers

    def execute(
        self,
        data: List[T],
        mapper: Callable[[T], Iterable[R]],
        reducer: Callable[[R, R], R]
    ) -> R:
        """
        Выполняет MapReduce:
        1. Разбивает данные на части (split)
        2. Применяет mapper к каждой части параллельно (map)
        3. Объединяет результаты (reduce)
        """
        # Split: разбиваем данные на чанки
        chunks = self._split(data, self.num_workers)

        # Map: параллельное выполнение mapper на каждом чанке
        with ThreadPoolExecutor(max_workers=self.num_workers) as executor:
            futures = []
            for chunk in chunks:
                future = executor.submit(self._map_chunk, chunk, mapper)
                futures.append(future)

            # Собираем результаты
            mapped_results = []
            for future in as_completed(futures):
                mapped_results.extend(future.result())

        # Reduce: объединяем все результаты
        return self._reduce(mapped_results, reducer)

    def _split(self, data: List[T], num_chunks: int) -> List[List[T]]:
        """Разбивает данные на равные части"""
        chunk_size = max(1, len(data) // num_chunks)
        return [data[i:i + chunk_size] for i in range(0, len(data), chunk_size)]

    def _map_chunk(self, chunk: List[T], mapper: Callable) -> List[R]:
        """Применяет mapper к каждому элементу чанка"""
        results = []
        for item in chunk:
            results.extend(mapper(item))
        return results

    def _reduce(self, items: List[R], reducer: Callable) -> R:
        """Редуцирует список до одного значения"""
        if not items:
            return None

        result = items[0]
        for item in items[1:]:
            result = reducer(result, item)
        return result


# Пример: подсчёт слов
def word_mapper(text: str) -> List[tuple]:
    """Mapper: текст -> [(word, 1), ...]"""
    words = text.lower().split()
    return [(word, 1) for word in words]

def count_reducer(a: dict, b: dict) -> dict:
    """Reducer: объединяет подсчёты"""
    result = a.copy()
    for word, count in b.items():
        result[word] = result.get(word, 0) + count
    return result

# Использование
controller = MapReduceController(num_workers=4)

texts = [
    "Hello world hello",
    "World of programming",
    "Hello programming world"
]

# Модифицированный mapper для reducer'а с dict
def word_count_mapper(text: str) -> List[dict]:
    words = text.lower().split()
    result = {}
    for word in words:
        result[word] = result.get(word, 0) + 1
    return [result]

word_counts = controller.execute(texts, word_count_mapper, count_reducer)
# {'hello': 3, 'world': 3, 'of': 1, 'programming': 2}
```

### 4. Load Balancing

```python
from abc import ABC, abstractmethod
from typing import List, Optional
import random
import time

class Worker:
    """Worker (Responder) с метриками нагрузки"""

    def __init__(self, worker_id: str):
        self.worker_id = worker_id
        self.current_load = 0
        self.max_load = 100
        self.is_active = True
        self.response_times: List[float] = []

    def process(self, request: dict) -> dict:
        """Обрабатывает запрос"""
        start = time.time()
        self.current_load += 1

        try:
            # Симуляция обработки
            result = {"worker": self.worker_id, "processed": request}
            return result
        finally:
            self.current_load -= 1
            self.response_times.append(time.time() - start)

    @property
    def average_response_time(self) -> float:
        if not self.response_times:
            return 0
        return sum(self.response_times[-100:]) / len(self.response_times[-100:])


class LoadBalancingStrategy(ABC):
    """Стратегия балансировки нагрузки"""

    @abstractmethod
    def select(self, workers: List[Worker]) -> Optional[Worker]:
        pass


class RoundRobinStrategy(LoadBalancingStrategy):
    """Round-Robin балансировка"""

    def __init__(self):
        self.current_index = 0

    def select(self, workers: List[Worker]) -> Optional[Worker]:
        active_workers = [w for w in workers if w.is_active]
        if not active_workers:
            return None

        worker = active_workers[self.current_index % len(active_workers)]
        self.current_index += 1
        return worker


class LeastConnectionsStrategy(LoadBalancingStrategy):
    """Выбор наименее загруженного worker'а"""

    def select(self, workers: List[Worker]) -> Optional[Worker]:
        active_workers = [w for w in workers if w.is_active]
        if not active_workers:
            return None

        return min(active_workers, key=lambda w: w.current_load)


class WeightedResponseTimeStrategy(LoadBalancingStrategy):
    """Выбор worker'а с лучшим временем отклика"""

    def select(self, workers: List[Worker]) -> Optional[Worker]:
        active_workers = [w for w in workers if w.is_active]
        if not active_workers:
            return None

        return min(active_workers, key=lambda w: w.average_response_time)


class LoadBalancer:
    """Controller (Load Balancer)"""

    def __init__(self, workers: List[Worker], strategy: LoadBalancingStrategy):
        self.workers = workers
        self.strategy = strategy

    def handle_request(self, request: dict) -> dict:
        """Обрабатывает запрос, направляя его на подходящий worker"""
        worker = self.strategy.select(self.workers)

        if not worker:
            raise Exception("No available workers")

        return worker.process(request)

    def add_worker(self, worker: Worker):
        self.workers.append(worker)

    def remove_worker(self, worker_id: str):
        self.workers = [w for w in self.workers if w.worker_id != worker_id]

    def health_check(self):
        """Проверяет здоровье всех workers"""
        for worker in self.workers:
            # Здесь может быть реальная проверка (ping, HTTP check, etc.)
            if worker.current_load >= worker.max_load:
                worker.is_active = False
            else:
                worker.is_active = True


# Использование
workers = [Worker(f"worker-{i}") for i in range(3)]
lb = LoadBalancer(workers, LeastConnectionsStrategy())

for i in range(10):
    result = lb.handle_request({"request_id": i})
    print(f"Request {i} handled by {result['worker']}")
```

## Когда использовать

### Идеальные случаи применения

| Use Case | Пример |
|----------|--------|
| **Репликация БД** | PostgreSQL Primary-Replica, MySQL Master-Slave |
| **Распределённые вычисления** | Hadoop MapReduce, Apache Spark |
| **Load Balancing** | Nginx, HAProxy, AWS ELB |
| **Очереди задач** | Celery с workers |
| **Кэширование** | Redis Cluster, Memcached |
| **Резервное копирование** | Primary-Secondary failover |

### Примеры из реального мира

```python
# 1. Celery: Controller (Beat) + Workers
from celery import Celery

app = Celery('tasks', broker='redis://localhost:6379')

# Worker (Responder) задачи
@app.task
def process_image(image_id: str):
    # Обработка изображения
    return {"status": "processed", "image_id": image_id}

# Controller отправляет задачи
def submit_image_processing(image_ids: list):
    for image_id in image_ids:
        process_image.delay(image_id)  # Отправляется workers


# 2. Nginx как Load Balancer (Controller)
"""
# nginx.conf
upstream backend {
    least_conn;  # Стратегия балансировки
    server backend1:8000;  # Responder 1
    server backend2:8000;  # Responder 2
    server backend3:8000;  # Responder 3
}

server {
    location / {
        proxy_pass http://backend;
    }
}
"""


# 3. PostgreSQL Streaming Replication
"""
# Primary (Controller) - postgresql.conf
wal_level = replica
max_wal_senders = 3

# Replica (Responder) - recovery.conf
primary_conninfo = 'host=primary port=5432 user=replicator'
"""
```

## Преимущества и недостатки

### Преимущества

| Преимущество | Описание |
|-------------|----------|
| **Масштабируемость чтения** | Добавление Replicas увеличивает пропускную способность |
| **Отказоустойчивость** | Replicas могут заменить Primary при сбое |
| **Разделение нагрузки** | Read/Write разделение снижает нагрузку на Primary |
| **Географическое распределение** | Replicas ближе к пользователям |
| **Простота модели** | Понятная иерархическая структура |
| **Консистентность** | Все записи через один узел |

### Недостатки

| Недостаток | Описание |
|-----------|----------|
| **Single Point of Failure** | Primary — единая точка отказа |
| **Задержка репликации** | Replicas могут отставать от Primary |
| **Сложность failover** | Переключение на новый Primary нетривиально |
| **Бутылочное горлышко записи** | Все записи через один узел |
| **Eventually Consistent** | Данные на Replicas могут быть устаревшими |
| **Сложность масштабирования записи** | Добавление Replicas не помогает записи |

## Примеры реализации

### Система репликации с автоматическим failover

```python
import threading
import time
from enum import Enum
from typing import Optional, List, Dict
from dataclasses import dataclass, field
import random

class NodeState(Enum):
    PRIMARY = "primary"
    REPLICA = "replica"
    CANDIDATE = "candidate"
    OFFLINE = "offline"

@dataclass
class ReplicationLog:
    entries: List[dict] = field(default_factory=list)
    last_index: int = 0

    def append(self, entry: dict) -> int:
        self.last_index += 1
        entry['index'] = self.last_index
        entry['timestamp'] = time.time()
        self.entries.append(entry)
        return self.last_index

    def get_entries_after(self, index: int) -> List[dict]:
        return [e for e in self.entries if e['index'] > index]


class DatabaseNode:
    """Узел базы данных с поддержкой репликации"""

    def __init__(self, node_id: str):
        self.node_id = node_id
        self.state = NodeState.REPLICA
        self.data: Dict[str, any] = {}
        self.replication_log = ReplicationLog()
        self.last_applied_index = 0
        self.primary: Optional['DatabaseNode'] = None
        self.replicas: List['DatabaseNode'] = []
        self.is_running = True
        self._lock = threading.Lock()

    def become_primary(self):
        """Переход в режим Primary"""
        with self._lock:
            self.state = NodeState.PRIMARY
            self.primary = None
            print(f"[{self.node_id}] Became PRIMARY")

    def become_replica(self, primary: 'DatabaseNode'):
        """Переход в режим Replica"""
        with self._lock:
            self.state = NodeState.REPLICA
            self.primary = primary
            print(f"[{self.node_id}] Became REPLICA of {primary.node_id}")

    def write(self, key: str, value: any) -> bool:
        """Запись данных (только для Primary)"""
        if self.state != NodeState.PRIMARY:
            if self.primary:
                return self.primary.write(key, value)
            raise Exception("Not a primary and no primary available")

        with self._lock:
            # Записываем локально
            self.data[key] = value

            # Добавляем в лог репликации
            entry = {"operation": "write", "key": key, "value": value}
            index = self.replication_log.append(entry)

            # Реплицируем
            self._replicate_to_all(index)

            return True

    def read(self, key: str) -> Optional[any]:
        """Чтение данных (можно с любого узла)"""
        with self._lock:
            return self.data.get(key)

    def _replicate_to_all(self, up_to_index: int):
        """Репликация на все Replicas"""
        for replica in self.replicas:
            try:
                self._replicate_to(replica, up_to_index)
            except Exception as e:
                print(f"[{self.node_id}] Failed to replicate to {replica.node_id}: {e}")

    def _replicate_to(self, replica: 'DatabaseNode', up_to_index: int):
        """Репликация на конкретную Replica"""
        entries = self.replication_log.get_entries_after(replica.last_applied_index)
        replica.apply_replication_entries(entries)

    def apply_replication_entries(self, entries: List[dict]):
        """Применение записей репликации (для Replica)"""
        with self._lock:
            for entry in entries:
                if entry['index'] <= self.last_applied_index:
                    continue

                if entry['operation'] == 'write':
                    self.data[entry['key']] = entry['value']

                self.last_applied_index = entry['index']

    def add_replica(self, replica: 'DatabaseNode'):
        """Добавление новой Replica"""
        self.replicas.append(replica)
        replica.become_replica(self)

        # Синхронизируем данные
        replica.data = self.data.copy()
        replica.last_applied_index = self.replication_log.last_index


class ClusterManager:
    """Менеджер кластера с автоматическим failover"""

    def __init__(self, nodes: List[DatabaseNode]):
        self.nodes = nodes
        self.primary: Optional[DatabaseNode] = None
        self.is_running = True

    def initialize(self):
        """Инициализация кластера"""
        if not self.nodes:
            raise Exception("No nodes in cluster")

        # Первый узел становится Primary
        self.primary = self.nodes[0]
        self.primary.become_primary()

        # Остальные — Replicas
        for node in self.nodes[1:]:
            self.primary.add_replica(node)

        # Запускаем мониторинг
        threading.Thread(target=self._health_monitor, daemon=True).start()

    def _health_monitor(self):
        """Мониторинг здоровья Primary"""
        while self.is_running:
            time.sleep(1)

            if self.primary and self.primary.state == NodeState.OFFLINE:
                self._perform_failover()

    def _perform_failover(self):
        """Автоматическое переключение на новый Primary"""
        print(f"[Cluster] Primary {self.primary.node_id} is OFFLINE, performing failover...")

        # Находим Replica с наибольшим last_applied_index
        candidates = [n for n in self.nodes if n.state == NodeState.REPLICA]
        if not candidates:
            print("[Cluster] No candidates for failover!")
            return

        new_primary = max(candidates, key=lambda n: n.last_applied_index)

        # Переключаем
        self.primary = new_primary
        new_primary.become_primary()

        # Переназначаем Replicas
        for node in candidates:
            if node != new_primary:
                new_primary.add_replica(node)

        print(f"[Cluster] Failover complete. New primary: {new_primary.node_id}")

    def get_read_node(self) -> DatabaseNode:
        """Получить узел для чтения (балансировка)"""
        replicas = [n for n in self.nodes if n.state == NodeState.REPLICA]
        if replicas:
            return random.choice(replicas)
        return self.primary

    def get_write_node(self) -> DatabaseNode:
        """Получить узел для записи (всегда Primary)"""
        return self.primary


# Использование
nodes = [DatabaseNode(f"node-{i}") for i in range(3)]
cluster = ClusterManager(nodes)
cluster.initialize()

# Запись через Primary
cluster.get_write_node().write("user:1", {"name": "Alice"})

# Чтение с любого узла
print(cluster.get_read_node().read("user:1"))

# Симуляция отказа Primary
nodes[0].state = NodeState.OFFLINE
time.sleep(2)  # Ждём failover
```

## Best Practices и антипаттерны

### Best Practices

```python
# 1. Read-your-writes consistency
class SmartClient:
    """Клиент с гарантией чтения своих записей"""

    def __init__(self, cluster: ClusterManager):
        self.cluster = cluster
        self.last_write_index: int = 0

    def write(self, key: str, value: any):
        node = self.cluster.get_write_node()
        node.write(key, value)
        self.last_write_index = node.replication_log.last_index

    def read(self, key: str):
        # Ищем Replica, которая уже применила нашу последнюю запись
        for node in self.cluster.nodes:
            if node.last_applied_index >= self.last_write_index:
                return node.read(key)

        # Fallback на Primary
        return self.cluster.get_write_node().read(key)


# 2. Replication lag monitoring
class ReplicationMonitor:
    """Мониторинг отставания репликации"""

    def __init__(self, primary: DatabaseNode, replicas: List[DatabaseNode]):
        self.primary = primary
        self.replicas = replicas

    def get_lag_seconds(self, replica: DatabaseNode) -> float:
        """Вычисляет отставание реплики в секундах"""
        primary_index = self.primary.replication_log.last_index
        replica_index = replica.last_applied_index

        if primary_index == replica_index:
            return 0.0

        # Находим timestamp последней применённой записи
        primary_entries = self.primary.replication_log.entries
        replica_last = next(
            (e for e in primary_entries if e['index'] == replica_index),
            None
        )
        primary_last = primary_entries[-1] if primary_entries else None

        if replica_last and primary_last:
            return primary_last['timestamp'] - replica_last['timestamp']

        return float('inf')

    def get_healthy_replicas(self, max_lag_seconds: float = 5.0):
        """Возвращает реплики с допустимым отставанием"""
        return [
            r for r in self.replicas
            if self.get_lag_seconds(r) <= max_lag_seconds
        ]


# 3. Graceful degradation при потере Primary
class ResilientClient:
    """Клиент с graceful degradation"""

    def __init__(self, cluster: ClusterManager):
        self.cluster = cluster
        self.pending_writes: List[tuple] = []

    def write(self, key: str, value: any):
        try:
            write_node = self.cluster.get_write_node()
            if write_node:
                write_node.write(key, value)
                self._flush_pending_writes(write_node)
            else:
                raise Exception("No primary available")
        except Exception:
            # Сохраняем для повторной попытки
            self.pending_writes.append((key, value))
            print(f"Write queued for retry: {key}")

    def _flush_pending_writes(self, node: DatabaseNode):
        """Применяет накопленные записи"""
        while self.pending_writes:
            key, value = self.pending_writes.pop(0)
            node.write(key, value)
```

### Антипаттерны

```python
# ❌ АНТИПАТТЕРН: Чтение после записи без учёта репликации
class BadClient:
    def process_order(self, order_id: str, order_data: dict):
        # Пишем в Primary
        self.primary.write(f"order:{order_id}", order_data)

        # СРАЗУ читаем из Replica — данные могут ещё не дойти!
        order = self.replica.read(f"order:{order_id}")
        self.send_confirmation(order)  # order может быть None!


# ❌ АНТИПАТТЕРН: Игнорирование replication lag
class BadLoadBalancer:
    def get_read_node(self):
        # Возвращаем любую Replica без проверки отставания
        return random.choice(self.replicas)
        # Может вернуть устаревшие данные!


# ❌ АНТИПАТТЕРН: Синхронная репликация без таймаутов
class BadPrimary:
    def write(self, key: str, value: any):
        self.data[key] = value

        # Ждём подтверждения от ВСЕХ реплик
        for replica in self.replicas:
            replica.apply(key, value)  # Если replica упадёт, всё зависнет!


# ✅ ПРАВИЛЬНО: Асинхронная репликация с подтверждением
class GoodPrimary:
    def write(self, key: str, value: any, wait_for: int = 1):
        """
        wait_for: сколько реплик должны подтвердить запись
        (0 = асинхронно, len(replicas) = полностью синхронно)
        """
        self.data[key] = value

        if wait_for == 0:
            # Полностью асинхронная репликация
            threading.Thread(target=self._replicate_async, args=(key, value)).start()
            return

        # Ждём подтверждения от N реплик с таймаутом
        confirmed = self._replicate_with_quorum(key, value, wait_for, timeout=5.0)
        if confirmed < wait_for:
            raise Exception(f"Only {confirmed}/{wait_for} replicas confirmed")
```

## Связанные паттерны

### Primary-Primary (Multi-Master)

В отличие от Controller-Responder, все узлы могут принимать записи:

```
┌──────────────┐     sync      ┌──────────────┐
│   Primary 1  │◄─────────────►│   Primary 2  │
│  (читает +   │               │  (читает +   │
│   пишет)     │               │   пишет)     │
└──────────────┘               └──────────────┘

Проблема: конфликты при одновременной записи
Решение: Conflict Resolution (Last-Write-Wins, Merge, etc.)
```

### Quorum-based Replication

Компромисс между консистентностью и доступностью:

```
W (write quorum) + R (read quorum) > N (total nodes)

Пример: N=5, W=3, R=3
- Запись успешна при подтверждении 3 из 5 узлов
- Чтение с 3 узлов гарантирует актуальные данные

┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐
│ Node1 │ │ Node2 │ │ Node3 │ │ Node4 │ │ Node5 │
│  ✓    │ │  ✓    │ │  ✓    │ │       │ │       │
└───────┘ └───────┘ └───────┘ └───────┘ └───────┘
   Write: 3 подтверждения — успех
```

### Сравнение подходов

| Аспект | Controller-Responder | Multi-Master | Quorum |
|--------|---------------------|--------------|--------|
| Консистентность | Строгая | Eventual | Tunable |
| Write throughput | Ограничен 1 узлом | Высокий | Средний |
| Сложность | Низкая | Высокая | Средняя |
| Конфликты | Нет | Да | Возможны |
| Failover | Нужен | Автоматический | Автоматический |

## Ресурсы для изучения

### Технологии
- **PostgreSQL Streaming Replication** — синхронная/асинхронная репликация
- **MySQL Replication** — binlog-based репликация
- **Redis Sentinel** — автоматический failover для Redis
- **Apache Kafka** — распределённый лог с репликацией

### Книги
- **"Designing Data-Intensive Applications"** — Martin Kleppmann (главы о репликации)
- **"Database Internals"** — Alex Petrov
- **"Distributed Systems"** — Maarten van Steen, Andrew Tanenbaum

### Онлайн-ресурсы
- [PostgreSQL Replication Documentation](https://www.postgresql.org/docs/current/high-availability.html)
- [MySQL Replication](https://dev.mysql.com/doc/refman/8.0/en/replication.html)
- [Martin Kleppmann's Blog](https://martin.kleppmann.com/)

---

**Предыдущая тема**: [Publish-Subscribe](./02-pub-sub.md)
