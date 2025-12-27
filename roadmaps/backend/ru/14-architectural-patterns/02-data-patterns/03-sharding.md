# Шардирование баз данных (Database Sharding)

## Определение

**Шардирование (Sharding)** — это техника горизонтального масштабирования базы данных, при которой данные разделяются на несколько независимых частей (шардов), каждая из которых хранится на отдельном сервере. Каждый шард содержит подмножество данных и может обрабатывать запросы независимо от других.

```
┌─────────────────────────────────────────────────────────────────┐
│                    До шардирования                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Все данные ──────► Один сервер БД (ограниченная мощность)     │
│   100M записей       - CPU bottleneck                           │
│                      - Memory bottleneck                         │
│                      - I/O bottleneck                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    После шардирования                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌──► Shard 1 (users A-H)    ~33M записей     │
│   Все данные ─────┼──► Shard 2 (users I-P)    ~33M записей     │
│   100M записей     └──► Shard 3 (users Q-Z)    ~33M записей     │
│                                                                  │
│   Каждый шард работает независимо и масштабируется отдельно    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Ключевые характеристики

### 1. Shard Key (Ключ шардирования)

Shard key — это поле (или комбинация полей), по которому определяется, на какой шард попадёт запись.

```python
from enum import Enum
from hashlib import md5
from typing import List, Optional

class ShardingStrategy(Enum):
    HASH = "hash"
    RANGE = "range"
    DIRECTORY = "directory"
    GEOGRAPHIC = "geographic"

class ShardRouter:
    """Маршрутизатор для определения шарда"""

    def __init__(
        self,
        shards: List[str],
        strategy: ShardingStrategy = ShardingStrategy.HASH
    ):
        self.shards = shards
        self.strategy = strategy
        self._directory: dict = {}  # Для directory-based sharding

    def get_shard(self, shard_key: str) -> str:
        """Определяет шард для данного ключа"""
        if self.strategy == ShardingStrategy.HASH:
            return self._hash_based(shard_key)
        elif self.strategy == ShardingStrategy.RANGE:
            return self._range_based(shard_key)
        elif self.strategy == ShardingStrategy.DIRECTORY:
            return self._directory_based(shard_key)
        elif self.strategy == ShardingStrategy.GEOGRAPHIC:
            return self._geo_based(shard_key)

    def _hash_based(self, shard_key: str) -> str:
        """Hash-based sharding: равномерное распределение"""
        hash_value = int(md5(shard_key.encode()).hexdigest(), 16)
        shard_index = hash_value % len(self.shards)
        return self.shards[shard_index]

    def _range_based(self, shard_key: str) -> str:
        """Range-based sharding: по диапазону значений"""
        # Пример: по первой букве
        first_char = shard_key[0].upper()
        if first_char <= 'H':
            return self.shards[0]
        elif first_char <= 'P':
            return self.shards[1]
        else:
            return self.shards[2]

    def _directory_based(self, shard_key: str) -> str:
        """Directory-based: явное указание в таблице маршрутизации"""
        if shard_key not in self._directory:
            # Назначаем шард по алгоритму балансировки
            shard = self._find_least_loaded_shard()
            self._directory[shard_key] = shard
        return self._directory[shard_key]
```

### 2. Consistent Hashing (Консистентное хеширование)

```python
import bisect
from typing import Dict, List, Optional

class ConsistentHashRing:
    """
    Consistent Hashing для шардирования.
    Минимизирует перемещение данных при добавлении/удалении шардов.
    """

    def __init__(self, virtual_nodes: int = 150):
        self.virtual_nodes = virtual_nodes
        self.ring: List[int] = []  # Отсортированный список хешей
        self.ring_to_shard: Dict[int, str] = {}  # Хеш -> шард

    def add_shard(self, shard: str) -> None:
        """Добавляет шард в кольцо"""
        for i in range(self.virtual_nodes):
            virtual_key = f"{shard}:{i}"
            hash_value = self._hash(virtual_key)

            if hash_value not in self.ring_to_shard:
                bisect.insort(self.ring, hash_value)
                self.ring_to_shard[hash_value] = shard

    def remove_shard(self, shard: str) -> None:
        """Удаляет шард из кольца"""
        for i in range(self.virtual_nodes):
            virtual_key = f"{shard}:{i}"
            hash_value = self._hash(virtual_key)

            if hash_value in self.ring_to_shard:
                self.ring.remove(hash_value)
                del self.ring_to_shard[hash_value]

    def get_shard(self, key: str) -> Optional[str]:
        """Находит шард для данного ключа"""
        if not self.ring:
            return None

        hash_value = self._hash(key)

        # Бинарный поиск первого хеша >= hash_value
        idx = bisect.bisect_right(self.ring, hash_value)

        # Если вышли за пределы, берём первый (кольцо)
        if idx == len(self.ring):
            idx = 0

        return self.ring_to_shard[self.ring[idx]]

    def _hash(self, key: str) -> int:
        """Вычисляет хеш ключа"""
        return int(md5(key.encode()).hexdigest(), 16)

# Использование
ring = ConsistentHashRing(virtual_nodes=100)
ring.add_shard("shard-1")
ring.add_shard("shard-2")
ring.add_shard("shard-3")

# Данные пользователя user_123 будут на одном шарде
shard = ring.get_shard("user_123")  # -> "shard-2"

# При добавлении нового шарда только часть данных перемещается
ring.add_shard("shard-4")  # ~25% данных переместится на новый шард
```

### 3. Архитектура шардированной системы

```
┌─────────────────────────────────────────────────────────────────┐
│                         Application                              │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Shard Router / Proxy                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Routing Logic (Consistent Hashing)          │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Connection Pool Management                  │    │
│  └─────────────────────────────────────────────────────────┘    │
└───────────────────────────────┬─────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
┌───────────────┐       ┌───────────────┐       ┌───────────────┐
│   Shard 1     │       │   Shard 2     │       │   Shard 3     │
│  ┌─────────┐  │       │  ┌─────────┐  │       │  ┌─────────┐  │
│  │ Primary │  │       │  │ Primary │  │       │  │ Primary │  │
│  └────┬────┘  │       │  └────┬────┘  │       │  └────┬────┘  │
│       │       │       │       │       │       │       │       │
│  ┌────▼────┐  │       │  ┌────▼────┐  │       │  ┌────▼────┐  │
│  │ Replica │  │       │  │ Replica │  │       │  │ Replica │  │
│  └─────────┘  │       │  └─────────┘  │       │  └─────────┘  │
└───────────────┘       └───────────────┘       └───────────────┘
```

---

## Когда использовать

### Идеальные сценарии

1. **Объём данных превышает возможности одного сервера**
   - Терабайты данных, не помещающиеся на один диск
   - Нагрузка на CPU/память превышает возможности сервера

2. **Высокая нагрузка на запись**
   - Тысячи транзакций в секунду
   - Репликация не решает проблему (master bottleneck)

3. **Географическое распределение**
   - Пользователи в разных регионах
   - Требования к latency (данные ближе к пользователю)

4. **Изоляция арендаторов (Multi-tenancy)**
   - Отдельные шарды для крупных клиентов
   - Соответствие требованиям безопасности

```python
# Пример: выбор стратегии шардирования

class ShardingAdvisor:
    """Помощник для выбора стратегии шардирования"""

    @staticmethod
    def recommend_strategy(
        data_size_tb: float,
        writes_per_second: int,
        query_patterns: List[str],
        geographic_distribution: bool
    ) -> ShardingStrategy:

        # Если есть географическое распределение — geo-sharding
        if geographic_distribution:
            return ShardingStrategy.GEOGRAPHIC

        # Если запросы часто по диапазонам — range-based
        range_queries = any("range" in p or "between" in p for p in query_patterns)
        if range_queries:
            return ShardingStrategy.RANGE

        # Для большинства случаев — hash-based (равномерное распределение)
        return ShardingStrategy.HASH
```

### Когда НЕ использовать

- Данные помещаются на один сервер
- Много запросов с JOIN между шардами
- Транзакции затрагивают данные на разных шардах
- Недостаточно экспертизы для поддержки

---

## Преимущества и недостатки

### Преимущества

| Преимущество | Описание |
|-------------|----------|
| **Горизонтальное масштабирование** | Практически неограниченный рост |
| **Улучшенная производительность** | Нагрузка распределяется между серверами |
| **Отказоустойчивость** | Сбой одного шарда не влияет на другие |
| **Географическая близость** | Данные ближе к пользователям |
| **Изоляция** | Проблемы на одном шарде не влияют на другие |

### Недостатки

| Недостаток | Описание |
|-----------|----------|
| **Сложность разработки** | JOIN и транзакции между шардами |
| **Сложность операций** | Бэкапы, миграции, мониторинг |
| **Resharding** | Перебалансировка данных — сложная операция |
| **Неравномерное распределение** | Hotspots — одни шарды нагружены больше других |
| **Уникальность** | Глобально уникальные ID требуют особого подхода |

---

## Примеры реализации

### Шардированный репозиторий

```python
from typing import TypeVar, Generic, Optional, List
import asyncpg

T = TypeVar("T")

class ShardedRepository(Generic[T]):
    """Репозиторий с поддержкой шардирования"""

    def __init__(
        self,
        shard_router: ConsistentHashRing,
        connection_pools: Dict[str, asyncpg.Pool]
    ):
        self.router = shard_router
        self.pools = connection_pools

    def _get_shard_key(self, entity: T) -> str:
        """Извлекает ключ шардирования из сущности"""
        raise NotImplementedError

    def _get_pool(self, shard_key: str) -> asyncpg.Pool:
        """Получает пул соединений для шарда"""
        shard = self.router.get_shard(shard_key)
        return self.pools[shard]

    async def save(self, entity: T) -> None:
        """Сохраняет сущность в нужный шард"""
        shard_key = self._get_shard_key(entity)
        pool = self._get_pool(shard_key)

        async with pool.acquire() as conn:
            await conn.execute(
                self._get_insert_query(),
                *self._entity_to_params(entity)
            )

    async def get(self, shard_key: str, entity_id: str) -> Optional[T]:
        """Получает сущность из нужного шарда"""
        pool = self._get_pool(shard_key)

        async with pool.acquire() as conn:
            row = await conn.fetchrow(
                self._get_select_query(),
                entity_id
            )
            return self._row_to_entity(row) if row else None

    async def query_all_shards(self, query: str, *args) -> List[T]:
        """Выполняет запрос на всех шардах (scatter-gather)"""
        results = []

        # Параллельный запрос ко всем шардам
        tasks = []
        for pool in self.pools.values():
            tasks.append(self._query_shard(pool, query, *args))

        shard_results = await asyncio.gather(*tasks)

        for shard_result in shard_results:
            results.extend(shard_result)

        return results

    async def _query_shard(
        self,
        pool: asyncpg.Pool,
        query: str,
        *args
    ) -> List[T]:
        async with pool.acquire() as conn:
            rows = await conn.fetch(query, *args)
            return [self._row_to_entity(row) for row in rows]

class UserRepository(ShardedRepository["User"]):
    """Репозиторий пользователей с шардированием по user_id"""

    def _get_shard_key(self, user: User) -> str:
        return user.id

    def _get_insert_query(self) -> str:
        return """
            INSERT INTO users (id, name, email, created_at)
            VALUES ($1, $2, $3, $4)
        """

    def _entity_to_params(self, user: User) -> tuple:
        return (user.id, user.name, user.email, user.created_at)

    def _get_select_query(self) -> str:
        return "SELECT * FROM users WHERE id = $1"

    def _row_to_entity(self, row) -> User:
        return User(
            id=row["id"],
            name=row["name"],
            email=row["email"],
            created_at=row["created_at"]
        )
```

### Генерация глобально уникальных ID

```python
import time
from dataclasses import dataclass

class SnowflakeIDGenerator:
    """
    Генератор уникальных ID по алгоритму Snowflake (Twitter).

    Структура ID (64 бита):
    - 1 бит: знак (всегда 0)
    - 41 бит: timestamp в миллисекундах
    - 10 бит: machine ID (до 1024 машин)
    - 12 бит: sequence number (до 4096 ID/мс на машину)
    """

    EPOCH = 1609459200000  # 2021-01-01 00:00:00 UTC в миллисекундах

    MACHINE_ID_BITS = 10
    SEQUENCE_BITS = 12

    MAX_MACHINE_ID = (1 << MACHINE_ID_BITS) - 1
    MAX_SEQUENCE = (1 << SEQUENCE_BITS) - 1

    def __init__(self, machine_id: int):
        if machine_id < 0 or machine_id > self.MAX_MACHINE_ID:
            raise ValueError(f"Machine ID must be between 0 and {self.MAX_MACHINE_ID}")

        self.machine_id = machine_id
        self.sequence = 0
        self.last_timestamp = -1

    def generate(self) -> int:
        """Генерирует уникальный ID"""
        timestamp = self._current_time_ms()

        if timestamp < self.last_timestamp:
            raise RuntimeError("Clock moved backwards!")

        if timestamp == self.last_timestamp:
            self.sequence = (self.sequence + 1) & self.MAX_SEQUENCE
            if self.sequence == 0:
                # Sequence overflow, wait for next millisecond
                timestamp = self._wait_next_ms(timestamp)
        else:
            self.sequence = 0

        self.last_timestamp = timestamp

        # Формируем ID
        id_value = (
            ((timestamp - self.EPOCH) << (self.MACHINE_ID_BITS + self.SEQUENCE_BITS)) |
            (self.machine_id << self.SEQUENCE_BITS) |
            self.sequence
        )

        return id_value

    def _current_time_ms(self) -> int:
        return int(time.time() * 1000)

    def _wait_next_ms(self, current_timestamp: int) -> int:
        timestamp = self._current_time_ms()
        while timestamp <= current_timestamp:
            timestamp = self._current_time_ms()
        return timestamp

    @staticmethod
    def parse(id_value: int) -> dict:
        """Разбирает ID на компоненты"""
        sequence = id_value & SnowflakeIDGenerator.MAX_SEQUENCE
        machine_id = (id_value >> SnowflakeIDGenerator.SEQUENCE_BITS) & SnowflakeIDGenerator.MAX_MACHINE_ID
        timestamp = (id_value >> (SnowflakeIDGenerator.MACHINE_ID_BITS + SnowflakeIDGenerator.SEQUENCE_BITS)) + SnowflakeIDGenerator.EPOCH

        return {
            "timestamp": timestamp,
            "machine_id": machine_id,
            "sequence": sequence,
            "datetime": datetime.fromtimestamp(timestamp / 1000)
        }

# Использование
generator = SnowflakeIDGenerator(machine_id=1)  # shard_id как machine_id
user_id = generator.generate()  # 1234567890123456789
```

### Cross-Shard операции

```python
class CrossShardQueryExecutor:
    """Выполнение запросов через несколько шардов"""

    def __init__(self, shard_pools: Dict[str, asyncpg.Pool]):
        self.pools = shard_pools

    async def aggregate(
        self,
        query: str,
        aggregation_func: Callable[[List[Any]], Any],
        *args
    ) -> Any:
        """
        Scatter-Gather паттерн для агрегаций.

        Пример: подсчёт общего количества пользователей
        """
        tasks = []
        for shard_name, pool in self.pools.items():
            tasks.append(self._execute_on_shard(pool, query, *args))

        results = await asyncio.gather(*tasks)
        return aggregation_func(results)

    async def _execute_on_shard(
        self,
        pool: asyncpg.Pool,
        query: str,
        *args
    ) -> Any:
        async with pool.acquire() as conn:
            return await conn.fetchval(query, *args)

# Использование
executor = CrossShardQueryExecutor(shard_pools)

# Общее количество пользователей
total_users = await executor.aggregate(
    "SELECT COUNT(*) FROM users",
    aggregation_func=sum
)

# Максимальный ID
max_id = await executor.aggregate(
    "SELECT MAX(id) FROM users",
    aggregation_func=max
)
```

### Resharding (перебалансировка)

```python
class ReshardingManager:
    """Управление перебалансировкой шардов"""

    def __init__(
        self,
        old_router: ConsistentHashRing,
        new_router: ConsistentHashRing,
        shard_pools: Dict[str, asyncpg.Pool]
    ):
        self.old_router = old_router
        self.new_router = new_router
        self.pools = shard_pools
        self._in_progress = False

    async def start_resharding(self) -> None:
        """Начинает процесс перебалансировки"""
        self._in_progress = True

        # 1. Включаем dual-write (запись в оба шарда)
        # 2. Мигрируем существующие данные
        await self._migrate_data()

        # 3. Переключаемся на новый роутер
        # 4. Очищаем старые данные

        self._in_progress = False

    async def _migrate_data(self) -> None:
        """Мигрирует данные на новые шарды"""
        for old_shard, pool in self.pools.items():
            async with pool.acquire() as conn:
                # Читаем данные порциями
                async for batch in self._read_in_batches(conn, batch_size=1000):
                    for record in batch:
                        shard_key = record["id"]
                        new_shard = self.new_router.get_shard(shard_key)

                        if new_shard != old_shard:
                            # Данные должны переехать на другой шард
                            await self._move_record(record, new_shard)

    async def get_shard_for_read(self, shard_key: str) -> str:
        """Определяет шард для чтения (учитывает resharding)"""
        if self._in_progress:
            # Во время resharding сначала пробуем новый шард
            new_shard = self.new_router.get_shard(shard_key)
            if await self._exists_in_shard(shard_key, new_shard):
                return new_shard
            # Fallback на старый шард
            return self.old_router.get_shard(shard_key)

        return self.new_router.get_shard(shard_key)
```

---

## Best Practices и антипаттерны

### Best Practices

```python
# 1. Выбирайте shard key с высокой кардинальностью
# ✅ Хорошо: user_id (миллионы уникальных значений)
# ❌ Плохо: country (только ~200 значений, неравномерное распределение)

# 2. Избегайте hotspots
class DateBasedSharding:
    """Антипаттерн: шардирование по дате создаёт hotspots"""
    def get_shard(self, created_at: datetime) -> str:
        # Плохо: все новые записи идут на один шард
        return f"shard_{created_at.year}_{created_at.month}"

class CompositeKeySharding:
    """Правильно: комбинированный ключ"""
    def get_shard(self, user_id: str, created_at: datetime) -> str:
        # Хорошо: данные распределяются равномерно
        return self.hash_router.get_shard(user_id)

# 3. Храните связанные данные вместе
class OrderSharding:
    """Заказы шардируются по тому же ключу, что и пользователи"""
    def get_shard(self, order: Order) -> str:
        # Используем user_id, не order_id
        # Это позволяет делать JOIN внутри шарда
        return self.router.get_shard(order.user_id)

# 4. Предусматривайте рост
class ScalableShardConfig:
    """Конфигурация с запасом для роста"""

    # Начинаем с виртуальных шардов
    VIRTUAL_SHARDS = 1024  # Много виртуальных шардов
    PHYSICAL_SHARDS = 4     # Мало физических изначально

    def __init__(self):
        # Маппинг: виртуальный шард -> физический
        self.mapping = {
            v: f"shard_{v % self.PHYSICAL_SHARDS}"
            for v in range(self.VIRTUAL_SHARDS)
        }

    def add_physical_shard(self, new_shard: str) -> None:
        """Добавление нового физического шарда"""
        # Перераспределяем часть виртуальных шардов
        # без изменения алгоритма хеширования
        pass
```

### Антипаттерны

```python
# ❌ Антипаттерн: Cross-shard транзакции
async def transfer_money_bad(from_user: str, to_user: str, amount: Decimal):
    """Плохо: транзакция между шардами"""
    from_shard = router.get_shard(from_user)
    to_shard = router.get_shard(to_user)

    # Невозможно гарантировать ACID между шардами!
    await debit(from_shard, from_user, amount)
    await credit(to_shard, to_user, amount)  # Что если здесь ошибка?

# ✅ Правильно: используйте Saga паттерн
async def transfer_money_saga(from_user: str, to_user: str, amount: Decimal):
    """Правильно: Saga для распределённой транзакции"""
    saga = TransferSaga(from_user, to_user, amount)
    await saga.execute()

# ❌ Антипаттерн: частые cross-shard запросы
async def get_friends_activity_bad(user_id: str) -> List[Activity]:
    """Плохо: много запросов к разным шардам"""
    friends = await get_friends(user_id)
    activities = []
    for friend in friends:  # N запросов к N шардам!
        activities.extend(await get_user_activity(friend.id))
    return activities

# ✅ Правильно: денормализация или async event-driven
class ActivityFeed:
    """Правильно: храним ленту активности для каждого пользователя"""
    async def on_friend_activity(self, event: ActivityEvent):
        # При активности друга — обновляем ленту пользователя
        friends = await get_user_friends(event.user_id)
        for friend_id in friends:
            shard = router.get_shard(friend_id)
            await add_to_feed(shard, friend_id, event)
```

---

## Связанные паттерны

| Паттерн | Связь с Sharding |
|---------|-----------------|
| **Consistent Hashing** | Алгоритм распределения данных по шардам |
| **Replication** | Каждый шард может иметь реплики для отказоустойчивости |
| **CQRS** | Разные модели чтения/записи могут быть на разных шардах |
| **Saga** | Координация транзакций между шардами |
| **Event Sourcing** | События можно шардировать по aggregate_id |

---

## Ресурсы для изучения

### Книги
- "Designing Data-Intensive Applications" — Martin Kleppmann (глава о партиционировании)
- "Database Internals" — Alex Petrov

### Решения для шардирования
- **Vitess** — шардирование для MySQL (используется YouTube)
- **Citus** — расширение PostgreSQL для шардирования
- **CockroachDB** — distributed SQL с автоматическим шардированием
- **MongoDB** — встроенное шардирование

### Статьи
- [Sharding at Pinterest](https://medium.com/pinterest-engineering)
- [How Discord Stores Billions of Messages](https://discord.com/blog)
- [Scaling Memcache at Facebook](https://www.usenix.org/conference/nsdi13/technical-sessions)
