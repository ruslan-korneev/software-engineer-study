# Sharding and Replication

Шардирование и репликация — две фундаментальные техники масштабирования баз данных и распределённых систем. Они решают разные задачи, но часто используются вместе для создания высокодоступных и масштабируемых систем.

---

## Репликация

### Что такое репликация и зачем она нужна

**Репликация** — это процесс создания и поддержания копий данных на нескольких серверах (узлах). Каждая копия называется **репликой**.

#### Зачем нужна репликация:

1. **Высокая доступность (High Availability)** — если один сервер выходит из строя, другие продолжают обслуживать запросы
2. **Отказоустойчивость (Fault Tolerance)** — данные не теряются при сбое оборудования
3. **Масштабирование чтения** — нагрузка на чтение распределяется между репликами
4. **Географическое распределение** — данные ближе к пользователям, меньше latency
5. **Резервное копирование** — реплики могут служить для backup без нагрузки на основной сервер

```
┌─────────────────────────────────────────────────────────────┐
│                     Клиенты                                 │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Load Balancer                             │
└─────────────────────────────────────────────────────────────┘
           │                  │                  │
           ▼                  ▼                  ▼
    ┌──────────┐       ┌──────────┐       ┌──────────┐
    │ Replica 1│       │ Replica 2│       │ Replica 3│
    │  (копия) │       │  (копия) │       │  (копия) │
    └──────────┘       └──────────┘       └──────────┘
```

---

### Типы репликации

#### 1. Master-Slave (Primary-Replica)

Самый распространённый тип репликации. Один узел является **master** (primary), остальные — **slave** (replica).

**Характеристики:**
- Все записи идут только на master
- Чтение может идти с любого узла
- Master реплицирует изменения на slave узлы

```
                    ┌──────────────┐
     Writes ───────▶│    Master    │
                    │   (Primary)  │
                    └──────────────┘
                           │
              Replication  │  (асинхронно или синхронно)
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   ┌──────────┐     ┌──────────┐     ┌──────────┐
   │  Slave 1 │     │  Slave 2 │     │  Slave 3 │
   │ (Replica)│     │ (Replica)│     │ (Replica)│
   └──────────┘     └──────────┘     └──────────┘
          ▲                ▲                ▲
          │                │                │
          └────────────────┴────────────────┘
                           │
                        Reads
```

**Преимущества:**
- Простота реализации и понимания
- Нет конфликтов записи (один источник правды)
- Хорошо масштабирует чтение

**Недостатки:**
- Master — единая точка отказа для записи
- При failover возможна потеря данных (если асинхронная репликация)
- Все записи ограничены производительностью одного узла

**Пример конфигурации PostgreSQL:**

```sql
-- На master (postgresql.conf)
wal_level = replica
max_wal_senders = 5
synchronous_commit = on

-- На replica
primary_conninfo = 'host=master_host port=5432 user=replicator'
```

---

#### 2. Master-Master (Multi-Master)

Все узлы могут принимать как чтение, так и запись.

```
                    ┌──────────────┐
     Writes/Reads ──│   Master 1   │
                    └──────────────┘
                           │
              Bi-directional replication
                           │
                    ┌──────────────┐
     Writes/Reads ──│   Master 2   │
                    └──────────────┘
```

**Преимущества:**
- Нет единой точки отказа для записи
- Лучше распределяет нагрузку на запись
- Отлично подходит для географически распределённых систем

**Недостатки:**
- **Конфликты записи** — два узла могут одновременно изменить одни данные
- Сложнее в настройке и поддержке
- Требует механизмов разрешения конфликтов

**Стратегии разрешения конфликтов:**

```python
# Last Write Wins (LWW) - побеждает последняя запись
def resolve_conflict_lww(record1, record2):
    if record1.timestamp > record2.timestamp:
        return record1
    return record2

# Merge - объединение изменений
def resolve_conflict_merge(record1, record2):
    merged = {}
    for field in set(record1.fields) | set(record2.fields):
        # Берём более свежее значение для каждого поля
        if record1.field_timestamp[field] > record2.field_timestamp[field]:
            merged[field] = record1[field]
        else:
            merged[field] = record2[field]
    return merged

# Custom resolution - пользовательская логика
def resolve_conflict_custom(record1, record2):
    # Например, для счётчиков можно суммировать
    return Record(count=record1.count + record2.count)
```

---

#### 3. Leaderless Replication

Нет выделенного leader/master. Все узлы равноправны, клиент пишет и читает с нескольких узлов одновременно.

```
                    Клиент
                       │
         ┌─────────────┼─────────────┐
         │             │             │
         ▼             ▼             ▼
    ┌─────────┐   ┌─────────┐   ┌─────────┐
    │  Node 1 │   │  Node 2 │   │  Node 3 │
    └─────────┘   └─────────┘   └─────────┘

    Write: клиент пишет на W узлов
    Read: клиент читает с R узлов
    Кворум: W + R > N (гарантирует консистентность)
```

**Quorum-based подход:**
- **N** — общее количество реплик
- **W** — количество узлов, которые должны подтвердить запись
- **R** — количество узлов, с которых нужно прочитать

```python
# Пример: N=3, W=2, R=2
# Гарантируется, что хотя бы один узел при чтении
# будет содержать самые свежие данные

class LeaderlessClient:
    def __init__(self, nodes, w=2, r=2):
        self.nodes = nodes  # Все узлы кластера
        self.w = w  # Write quorum
        self.r = r  # Read quorum

    async def write(self, key, value, version):
        """Запись с кворумом W"""
        responses = await asyncio.gather(*[
            node.write(key, value, version)
            for node in self.nodes
        ], return_exceptions=True)

        successful = sum(1 for r in responses if r is True)
        if successful >= self.w:
            return True
        raise WriteQuorumNotReached()

    async def read(self, key):
        """Чтение с кворумом R, возврат самой свежей версии"""
        responses = await asyncio.gather(*[
            node.read(key)
            for node in self.nodes
        ], return_exceptions=True)

        valid_responses = [r for r in responses if not isinstance(r, Exception)]
        if len(valid_responses) < self.r:
            raise ReadQuorumNotReached()

        # Возвращаем значение с максимальной версией
        return max(valid_responses, key=lambda x: x.version)
```

**Примеры систем:**
- **Cassandra** — использует leaderless репликацию
- **DynamoDB** — вдохновлён этим подходом
- **Riak** — полностью leaderless

---

### Синхронная vs Асинхронная репликация

#### Синхронная репликация

Транзакция считается завершённой только после подтверждения от всех (или части) реплик.

```
Client        Master        Replica
   │             │             │
   │──Write─────▶│             │
   │             │──Replicate─▶│
   │             │◀──ACK───────│
   │◀──OK────────│             │
   │             │             │
```

**Преимущества:**
- Гарантированная консистентность данных
- Нет потери данных при failover

**Недостатки:**
- Увеличенная latency (ждём ответ от реплик)
- Доступность зависит от всех синхронных реплик

#### Асинхронная репликация

Транзакция подтверждается сразу после записи на master, репликация происходит в фоне.

```
Client        Master        Replica
   │             │             │
   │──Write─────▶│             │
   │◀──OK────────│             │
   │             │──Replicate─▶│  (в фоне)
   │             │             │
```

**Преимущества:**
- Низкая latency для клиента
- Master не зависит от доступности реплик

**Недостатки:**
- **Replication lag** — реплики отстают
- Возможна потеря данных при сбое master

#### Полусинхронная репликация (Semi-synchronous)

Компромисс: ждём подтверждения от **одной** реплики.

```sql
-- MySQL semi-sync replication
SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_master_wait_for_slave_count = 1;
```

---

### Replication Lag и его последствия

**Replication lag** — задержка между записью на master и появлением данных на репликах.

```
Время:     t=0       t=1       t=2       t=3
           │         │         │         │
Master:    [A]──────[B]──────[C]──────[D]
Replica:   [A]──────[A]──────[B]──────[C]
           │         │         │         │
           └─────────┴─────────┴─────────┘
                  Replication Lag
```

#### Проблемы из-за replication lag:

**1. Read-after-write inconsistency:**
```python
# Пользователь обновляет профиль
await master.update_profile(user_id, new_data)  # t=0

# Сразу читает с реплики - видит старые данные!
profile = await replica.get_profile(user_id)  # t=0.1
# profile содержит старые данные, т.к. репликация ещё не завершена
```

**Решение — Read-your-writes consistency:**
```python
class SmartClient:
    def __init__(self, master, replicas):
        self.master = master
        self.replicas = replicas
        self.last_write_timestamp = {}

    async def write(self, user_id, data):
        result = await self.master.write(data)
        self.last_write_timestamp[user_id] = time.time()
        return result

    async def read(self, user_id, query):
        # Если недавно писали - читаем с master
        if self.last_write_timestamp.get(user_id, 0) > time.time() - 5:
            return await self.master.read(query)

        # Иначе можно читать с реплики
        return await random.choice(self.replicas).read(query)
```

**2. Monotonic reads violation:**
```python
# Пользователь видит комментарий на Replica1
comment = await replica1.get_comment(id)  # версия: 5

# Следующий запрос попадает на отстающую Replica2
comment = await replica2.get_comment(id)  # версия: 3
# Комментарий "исчез" или вернулся к старой версии!
```

**Решение — Session stickiness:**
```python
# Привязываем пользователя к конкретной реплике
def get_replica_for_user(user_id, replicas):
    return replicas[hash(user_id) % len(replicas)]
```

**3. Causal consistency violation:**
```python
# User A пишет пост
await master.create_post(post_id, content)

# User B видит пост на Replica1 и комментирует
await master.create_comment(post_id, comment)

# User C читает с отстающей Replica2
comments = await replica2.get_comments(post_id)  # Видит комментарий
post = await replica2.get_post(post_id)  # НЕ видит пост!
# Комментарий есть, а поста нет - нарушение причинности
```

---

### Read Replicas для масштабирования чтения

Read replicas — реплики, предназначенные только для чтения.

```
                        ┌──────────────────────┐
        Writes ────────▶│       Master         │
                        └──────────────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
             ┌──────────┐   ┌──────────┐   ┌──────────┐
             │ Read     │   │ Read     │   │ Read     │
             │ Replica 1│   │ Replica 2│   │ Replica 3│
             └──────────┘   └──────────┘   └──────────┘
                    ▲              ▲              ▲
                    └──────────────┴──────────────┘
                                   │
                                Reads
```

**Когда использовать:**
- Нагрузка на чтение значительно превышает запись (OLTP системы)
- Тяжёлые аналитические запросы (выносим на отдельную реплику)
- Географическое распределение читателей

**Пример архитектуры с Cloud провайдером:**

```python
# AWS RDS с read replicas
import boto3

rds = boto3.client('rds')

# Создание read replica
rds.create_db_instance_read_replica(
    DBInstanceIdentifier='mydb-replica-1',
    SourceDBInstanceIdentifier='mydb-master',
    DBInstanceClass='db.r5.large',
    AvailabilityZone='us-east-1b'
)
```

**Паттерн Command Query Responsibility Segregation (CQRS):**

```python
class UserRepository:
    def __init__(self, master_db, replica_db):
        self.master = master_db
        self.replica = replica_db

    # Commands (записи) - идут на master
    async def create_user(self, user: User):
        return await self.master.execute(
            "INSERT INTO users (...) VALUES (...)", user
        )

    async def update_user(self, user_id: int, data: dict):
        return await self.master.execute(
            "UPDATE users SET ... WHERE id = ?", user_id, data
        )

    # Queries (чтения) - идут на replica
    async def get_user(self, user_id: int):
        return await self.replica.fetch_one(
            "SELECT * FROM users WHERE id = ?", user_id
        )

    async def search_users(self, query: str):
        return await self.replica.fetch_all(
            "SELECT * FROM users WHERE name LIKE ?", f"%{query}%"
        )
```

---

### Failover и высокая доступность

**Failover** — автоматическое переключение на резервный сервер при сбое основного.

#### Типы failover:

**1. Automatic Failover:**
```
        ┌─────────────┐
        │  Sentinel   │ ◀── Мониторинг
        │  / Arbiter  │
        └─────────────┘
               │
     Обнаружение сбоя
               │
               ▼
    ┌──────────────────┐
    │ Promote replica  │
    │   to master      │
    └──────────────────┘
               │
               ▼
    ┌──────────────────┐
    │ Redirect clients │
    │ to new master    │
    └──────────────────┘
```

**2. Manual Failover:**
- Администратор вручную инициирует переключение
- Используется для плановых работ

#### Компоненты системы failover:

```python
class FailoverManager:
    def __init__(self, master, replicas, check_interval=1):
        self.master = master
        self.replicas = replicas
        self.check_interval = check_interval

    async def monitor(self):
        """Мониторинг здоровья master"""
        consecutive_failures = 0

        while True:
            try:
                await self.master.ping()
                consecutive_failures = 0
            except ConnectionError:
                consecutive_failures += 1

                if consecutive_failures >= 3:  # Порог для failover
                    await self.perform_failover()
                    break

            await asyncio.sleep(self.check_interval)

    async def perform_failover(self):
        """Выполнение failover"""
        # 1. Выбираем лучшую реплику (с минимальным lag)
        best_replica = await self.select_best_replica()

        # 2. Продвигаем её до master
        await best_replica.promote_to_master()

        # 3. Переконфигурируем остальные реплики
        for replica in self.replicas:
            if replica != best_replica:
                await replica.follow_new_master(best_replica)

        # 4. Обновляем DNS/конфигурацию
        await self.update_master_endpoint(best_replica)

        self.master = best_replica

    async def select_best_replica(self):
        """Выбор реплики с минимальным отставанием"""
        replica_lags = []
        for replica in self.replicas:
            try:
                lag = await replica.get_replication_lag()
                replica_lags.append((replica, lag))
            except:
                pass

        return min(replica_lags, key=lambda x: x[1])[0]
```

#### Redis Sentinel — пример автоматического failover:

```
┌─────────────────────────────────────────────────────────────┐
│                    Sentinel Cluster                         │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │Sentinel 1│    │Sentinel 2│    │Sentinel 3│              │
│  └──────────┘    └──────────┘    └──────────┘              │
│        │              │              │                      │
│        └──────────────┴──────────────┘                      │
│                       │                                     │
│              Мониторинг и голосование                       │
└───────────────────────┼─────────────────────────────────────┘
                        │
         ┌──────────────┼──────────────┐
         ▼              ▼              ▼
   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │  Master  │   │  Slave 1 │   │  Slave 2 │
   │  (Redis) │   │  (Redis) │   │  (Redis) │
   └──────────┘   └──────────┘   └──────────┘
```

```bash
# Конфигурация Sentinel
sentinel monitor mymaster 192.168.1.10 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

#### Split-brain проблема:

Когда из-за сетевого разделения образуется два master:

```
     Network Partition
            ║
┌───────────║───────────┐
│  DC 1     ║    DC 2   │
│           ║           │
│ Master ───╳─── Slave  │
│   ↓       ║     ↓     │
│ Writes    ║  Promoted │
│           ║  to Master│
│           ║     ↓     │
│           ║  Writes   │
└───────────║───────────┘
            ║
   Два master = конфликты!
```

**Решения:**
1. **Fencing** — отключение старого master (STONITH — Shoot The Other Node In The Head)
2. **Кворум** — требуется большинство голосов для продвижения
3. **Witness node** — дополнительный узел для разрешения споров

---

## Шардирование (Partitioning)

### Что такое шардирование

**Шардирование** (также partitioning) — разделение данных между несколькими серверами, где каждый сервер (**шард**) содержит часть данных.

```
┌─────────────────────────────────────────────────────────────┐
│                      Все данные                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Users: 1-1M | Orders: 1-10M | Products: 1-100K        │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
                   Шардирование
                              │
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
   ┌──────────┐         ┌──────────┐         ┌──────────┐
   │  Shard 1 │         │  Shard 2 │         │  Shard 3 │
   │ Users    │         │ Users    │         │ Users    │
   │ 1-333K   │         │ 334K-666K│         │ 667K-1M  │
   └──────────┘         └──────────┘         └──────────┘
```

#### Зачем нужно шардирование:

1. **Горизонтальное масштабирование записи** — в отличие от репликации, шардирование увеличивает пропускную способность записи
2. **Увеличение объёма данных** — когда данные не помещаются на один сервер
3. **Распределение нагрузки** — каждый шард обрабатывает свою часть запросов
4. **Изоляция** — проблемы одного шарда не влияют на другие

---

### Горизонтальное vs Вертикальное партицирование

#### Вертикальное партицирование (Vertical Partitioning)

Разделение **по столбцам** — разные атрибуты хранятся на разных серверах.

```
Исходная таблица Users:
┌────┬──────┬───────┬─────────┬──────────────┬───────────────┐
│ id │ name │ email │ password│ profile_photo│ preferences   │
└────┴──────┴───────┴─────────┴──────────────┴───────────────┘

После вертикального партицирования:

Server 1 (Core Data):        Server 2 (Media):        Server 3 (Settings):
┌────┬──────┬───────┬───────┐ ┌────┬──────────────┐    ┌────┬───────────────┐
│ id │ name │ email │ pass  │ │ id │ profile_photo│    │ id │ preferences   │
└────┴──────┴───────┴───────┘ └────┴──────────────┘    └────┴───────────────┘
```

**Преимущества:**
- Часто используемые данные отделены от редко используемых
- Разные требования к хранению (BLOB vs обычные данные)
- Можно использовать разные типы хранилищ

**Недостатки:**
- JOIN между серверами дорогой
- Не решает проблему большого количества строк

#### Горизонтальное партицирование (Horizontal Partitioning / Sharding)

Разделение **по строкам** — разные записи хранятся на разных серверах.

```
Исходная таблица Users (1M записей):
┌────────────────────────────────────────────┐
│ id │ name    │ email           │ ...      │
├────┼─────────┼─────────────────┼──────────┤
│ 1  │ Alice   │ alice@mail.com  │ ...      │
│ 2  │ Bob     │ bob@mail.com    │ ...      │
│... │ ...     │ ...             │ ...      │
│ 1M │ Zara    │ zara@mail.com   │ ...      │
└────────────────────────────────────────────┘

После горизонтального партицирования:

Shard 1:                 Shard 2:                 Shard 3:
┌────┬─────────┬────┐    ┌────┬─────────┬────┐    ┌────┬─────────┬────┐
│ 1  │ Alice   │... │    │334K│ Mike    │... │    │667K│ Yuki    │... │
│ 2  │ Bob     │... │    │... │ ...     │... │    │... │ ...     │... │
│... │ ...     │... │    │666K│ Nora    │... │    │ 1M │ Zara    │... │
│333K│ Luna    │... │    └────┴─────────┴────┘    └────┴─────────┴────┘
└────┴─────────┴────┘
```

**Преимущества:**
- Масштабирует и чтение, и запись
- Решает проблему большого объёма данных
- Каждый шард независим

**Недостатки:**
- Сложнее в реализации
- Cross-shard запросы дорогие
- Нужно выбирать shard key

---

### Стратегии шардирования

#### 1. Range-based Sharding

Данные распределяются по диапазонам значений shard key.

```
Shard key: user_id

Shard 1: user_id 1 - 1,000,000
Shard 2: user_id 1,000,001 - 2,000,000
Shard 3: user_id 2,000,001 - 3,000,000
```

```python
class RangeBasedRouter:
    def __init__(self, ranges: list):
        # [(0, 1000000, 'shard1'), (1000001, 2000000, 'shard2'), ...]
        self.ranges = sorted(ranges, key=lambda x: x[0])

    def get_shard(self, user_id: int) -> str:
        for start, end, shard in self.ranges:
            if start <= user_id <= end:
                return shard
        raise ValueError(f"No shard found for user_id {user_id}")

    def get_shards_for_range(self, start_id: int, end_id: int) -> list:
        """Для range queries - какие шарды нужно опросить"""
        shards = []
        for range_start, range_end, shard in self.ranges:
            if range_start <= end_id and range_end >= start_id:
                shards.append(shard)
        return shards
```

**Преимущества:**
- Простота понимания и реализации
- Эффективные range queries (ORDER BY, BETWEEN)
- Легко добавлять новые шарды

**Недостатки:**
- **Hot spots** — неравномерное распределение (новые пользователи все на последнем шарде)
- Требует ручного ребалансирования

**Хороший use case:** Данные с временными метками (логи, события) — старые данные можно архивировать целыми шардами.

---

#### 2. Hash-based Sharding

Данные распределяются по хешу от shard key.

```
shard_number = hash(user_id) % number_of_shards

hash(123) % 3 = 0 → Shard 0
hash(456) % 3 = 1 → Shard 1
hash(789) % 3 = 2 → Shard 2
```

```python
import hashlib

class HashBasedRouter:
    def __init__(self, shards: list):
        self.shards = shards
        self.num_shards = len(shards)

    def _hash(self, key) -> int:
        """Консистентный хеш для любого типа ключа"""
        key_bytes = str(key).encode('utf-8')
        return int(hashlib.md5(key_bytes).hexdigest(), 16)

    def get_shard(self, key) -> str:
        shard_index = self._hash(key) % self.num_shards
        return self.shards[shard_index]

# Использование
router = HashBasedRouter(['shard1', 'shard2', 'shard3'])
print(router.get_shard(123))  # shard2
print(router.get_shard(456))  # shard1
```

**Преимущества:**
- Равномерное распределение данных
- Нет hot spots (при хорошей hash функции)
- Простая маршрутизация

**Недостатки:**
- Range queries требуют опроса всех шардов
- **Resharding — катастрофа**: при изменении количества шардов почти все данные нужно перемещать

```
До: 3 шарда
hash(key) % 3 = 1 → Shard 1

После: 4 шарда
hash(key) % 4 = 2 → Shard 2  ← Данные нужно переместить!
```

---

#### 3. Directory-based Sharding

Отдельный сервис (directory/lookup service) хранит mapping ключей к шардам.

```
┌─────────────────────────────────────────────────────────────┐
│                    Directory Service                        │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  user_id: 123    → shard_2                            │  │
│  │  user_id: 456    → shard_1                            │  │
│  │  user_id: 789    → shard_3                            │  │
│  │  ...                                                  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
   ┌──────────┐         ┌──────────┐         ┌──────────┐
   │  Shard 1 │         │  Shard 2 │         │  Shard 3 │
   └──────────┘         └──────────┘         └──────────┘
```

```python
class DirectoryBasedRouter:
    def __init__(self, directory_service):
        self.directory = directory_service
        self.cache = {}  # Локальный кеш для производительности

    async def get_shard(self, key) -> str:
        # Проверяем кеш
        if key in self.cache:
            return self.cache[key]

        # Запрос к directory service
        shard = await self.directory.lookup(key)
        self.cache[key] = shard
        return shard

    async def register_key(self, key, shard: str):
        """Регистрация нового ключа"""
        await self.directory.register(key, shard)
        self.cache[key] = shard

    async def move_key(self, key, from_shard: str, to_shard: str):
        """Перемещение ключа между шардами"""
        # 1. Копируем данные
        data = await self.shards[from_shard].get(key)
        await self.shards[to_shard].put(key, data)

        # 2. Обновляем directory
        await self.directory.update(key, to_shard)

        # 3. Удаляем старые данные
        await self.shards[from_shard].delete(key)

        # 4. Обновляем кеш
        self.cache[key] = to_shard
```

**Преимущества:**
- Максимальная гибкость — любая стратегия распределения
- Легко перемещать данные между шардами
- Можно балансировать нагрузку динамически

**Недостатки:**
- Directory service — единая точка отказа (нужна репликация)
- Дополнительный hop для каждого запроса
- Нужно хранить mapping для всех ключей

---

#### 4. Consistent Hashing

Решает главную проблему hash-based подхода — минимизирует перемещение данных при изменении количества узлов.

```
                    Hash Ring (0 - 2^32)

                           0°
                           │
                     ╭─────┼─────╮
                   ╱       │       ╲
                 ╱    Node A       ╲
                │         │         │
           270°─┼─────────┼─────────┼─ 90°
                │         │         │
                 ╲    Node C       ╱
                   ╲       │       ╱
                     ╰─────┼─────╯
                       Node B
                         180°

Ключи распределяются по кольцу и "принадлежат"
ближайшему узлу по часовой стрелке.
```

```python
import hashlib
from bisect import bisect_right

class ConsistentHash:
    def __init__(self, nodes: list = None, virtual_nodes: int = 100):
        self.virtual_nodes = virtual_nodes  # Количество виртуальных узлов
        self.ring = {}  # hash -> node
        self.sorted_keys = []  # Отсортированные хеши для бинарного поиска

        if nodes:
            for node in nodes:
                self.add_node(node)

    def _hash(self, key: str) -> int:
        """Хеширование ключа в позицию на кольце"""
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node: str):
        """Добавление узла с виртуальными репликами"""
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)
            self.ring[hash_value] = node
            self.sorted_keys.append(hash_value)

        self.sorted_keys.sort()

    def remove_node(self, node: str):
        """Удаление узла"""
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)
            del self.ring[hash_value]
            self.sorted_keys.remove(hash_value)

    def get_node(self, key: str) -> str:
        """Получение узла для ключа"""
        if not self.ring:
            return None

        hash_value = self._hash(key)

        # Находим первый узел по часовой стрелке
        idx = bisect_right(self.sorted_keys, hash_value)

        # Если дошли до конца, переходим в начало кольца
        if idx == len(self.sorted_keys):
            idx = 0

        return self.ring[self.sorted_keys[idx]]

    def get_nodes(self, key: str, count: int = 3) -> list:
        """Получение нескольких узлов (для репликации)"""
        if not self.ring:
            return []

        hash_value = self._hash(key)
        idx = bisect_right(self.sorted_keys, hash_value)

        nodes = []
        seen = set()

        while len(nodes) < count and len(seen) < len(set(self.ring.values())):
            if idx >= len(self.sorted_keys):
                idx = 0

            node = self.ring[self.sorted_keys[idx]]
            if node not in seen:
                nodes.append(node)
                seen.add(node)

            idx += 1

        return nodes


# Демонстрация
ch = ConsistentHash(['node1', 'node2', 'node3'], virtual_nodes=100)

# Распределение ключей
keys = ['user:1', 'user:2', 'user:3', 'order:100', 'order:200']
print("Начальное распределение:")
for key in keys:
    print(f"  {key} → {ch.get_node(key)}")

# Добавляем новый узел
print("\nДобавляем node4:")
ch.add_node('node4')
for key in keys:
    print(f"  {key} → {ch.get_node(key)}")
# Только ~25% ключей переместятся (1/4 от нового количества узлов)
```

**Преимущества:**
- При добавлении узла перемещается только ~K/N ключей (K — всего ключей, N — узлов)
- При удалении узла его ключи равномерно распределяются между соседями
- Виртуальные узлы обеспечивают равномерность

**Недостатки:**
- Сложнее в реализации
- Range queries всё ещё проблема
- Требует больше памяти для хранения кольца

**Используется в:** Cassandra, DynamoDB, Memcached, Riak

---

### Shard Key — выбор ключа шардирования

Выбор shard key — **критически важное решение**, которое влияет на:
- Распределение данных
- Производительность запросов
- Возможность масштабирования

#### Критерии хорошего shard key:

**1. Высокая кардинальность (много уникальных значений)**
```python
# Плохо: статус заказа (только 5 значений)
shard_key = order.status  # 'pending', 'processing', 'shipped', 'delivered', 'cancelled'
# Все заказы попадут в 5 шардов максимум

# Хорошо: ID заказа
shard_key = order.id  # Миллионы уникальных значений
```

**2. Равномерное распределение**
```python
# Плохо: страна пользователя
shard_key = user.country  # 50% данных в USA, 30% в China...
# Несколько шардов будут перегружены

# Хорошо: хеш от user_id
shard_key = hash(user.id)  # Равномерное распределение
```

**3. Соответствие паттернам доступа**
```python
# Если большинство запросов — по user_id:
SELECT * FROM orders WHERE user_id = ?

# То shard_key = user_id — хороший выбор
# Все заказы пользователя на одном шарде — эффективный JOIN
```

**4. Неизменяемость**
```python
# Плохо: email пользователя (может измениться)
shard_key = user.email
# При изменении email нужно перемещать все данные

# Хорошо: user_id (никогда не меняется)
shard_key = user.id
```

#### Примеры выбора shard key:

```python
# E-commerce: заказы
# Если часто запрашиваем заказы пользователя:
shard_key = user_id
# Все заказы пользователя на одном шарде

# Если часто ищем по дате:
shard_key = (year, month)
# Range queries по датам эффективны

# Social network: посты
# Если часто читаем ленту пользователя:
shard_key = author_id

# Multi-tenant SaaS:
shard_key = tenant_id
# Данные каждого клиента изолированы на своём шарде
```

#### Compound shard key:

```python
# MongoDB compound shard key
db.orders.createIndex({ "user_id": 1, "created_at": 1 })
sh.shardCollection("mydb.orders", { "user_id": 1, "created_at": 1 })

# Позволяет эффективно выполнять:
# 1. Запросы по user_id (попадает на один шард)
# 2. Range queries по user_id + created_at
```

---

### Проблемы шардирования

#### 1. Cross-shard Queries

Запросы, требующие данных с нескольких шардов.

```sql
-- Если users шардированы по user_id, а orders по order_id:

SELECT u.name, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.total > 1000;

-- Нужно опросить ВСЕ шарды обеих таблиц!
```

**Решения:**

```python
# 1. Scatter-Gather паттерн
async def cross_shard_query(query):
    # Отправляем запрос на все шарды параллельно
    results = await asyncio.gather(*[
        shard.execute(query) for shard in all_shards
    ])

    # Объединяем результаты
    return merge_results(results)

# 2. Денормализация — дублируем данные
# Вместо JOIN храним нужные данные вместе
{
    "order_id": 123,
    "user_id": 456,
    "user_name": "Alice",  # Денормализовано из users
    "total": 1500
}

# 3. Правильный выбор shard key
# Шардируем связанные таблицы по одному ключу
shard_key = user_id  # И для users, и для orders
```

---

#### 2. Rebalancing

Перераспределение данных между шардами при:
- Добавлении новых шардов
- Неравномерной нагрузке
- Удалении шардов

```
До ребалансирования:              После ребалансирования:
┌──────────┐                      ┌──────────┐
│ Shard 1  │ 40%                  │ Shard 1  │ 25%
│ 4TB data │                      │ 2.5TB    │
└──────────┘                      └──────────┘

┌──────────┐                      ┌──────────┐
│ Shard 2  │ 40%                  │ Shard 2  │ 25%
│ 4TB data │                      │ 2.5TB    │
└──────────┘                      └──────────┘

┌──────────┐                      ┌──────────┐
│ Shard 3  │ 20%                  │ Shard 3  │ 25%
│ 2TB data │                      │ 2.5TB    │
└──────────┘                      └──────────┘

                                  ┌──────────┐
                                  │ Shard 4  │ 25%  ← NEW
                                  │ 2.5TB    │
                                  └──────────┘
```

**Проблемы ребалансирования:**
- Перемещение данных создаёт нагрузку
- Нужно обеспечить доступность во время миграции
- Сложная координация

```python
class ShardRebalancer:
    async def rebalance(self, source_shard, target_shard, key_range):
        """Миграция данных с минимальным downtime"""

        # 1. Начинаем двойную запись (пишем на оба шарда)
        await self.enable_dual_write(key_range, source_shard, target_shard)

        # 2. Копируем исторические данные порциями
        cursor = None
        while True:
            batch, cursor = await source_shard.scan(key_range, cursor, limit=1000)
            if not batch:
                break

            await target_shard.bulk_insert(batch)
            await asyncio.sleep(0.1)  # Throttling

        # 3. Переключаем роутинг на новый шард
        await self.update_routing(key_range, target_shard)

        # 4. Отключаем двойную запись
        await self.disable_dual_write(key_range)

        # 5. Удаляем данные со старого шарда (опционально)
        # await source_shard.delete_range(key_range)
```

---

#### 3. Hot Spots

Неравномерная нагрузка — некоторые шарды получают больше запросов.

```
Celebrities Problem:
User @famous (id: 12345) имеет 100M подписчиков

Shard, где хранятся данные @famous, получает в 1000 раз больше запросов!
```

**Причины:**
- Популярные записи (celebrities, viral content)
- Неудачный shard key
- Временные паттерны (все запросы за "сегодня" на одном шарде)

**Решения:**

```python
# 1. Добавление случайного суффикса для hot keys
def get_shard_key(user_id, is_hot=False):
    if is_hot:
        # Распределяем hot key между несколькими шардами
        suffix = random.randint(0, 9)
        return f"{user_id}:{suffix}"
    return str(user_id)

# При чтении hot key — запрашиваем все варианты
async def read_hot_user(user_id):
    keys = [f"{user_id}:{i}" for i in range(10)]
    shards = [router.get_shard(key) for key in keys]
    results = await asyncio.gather(*[
        shard.get(key) for shard, key in zip(shards, keys)
    ])
    return merge_results(results)

# 2. Кеширование hot data
# Выносим популярные данные в Redis/Memcached
if is_hot_key(key):
    result = await cache.get(key)
    if result:
        return result

# 3. Rate limiting для hot shards
```

---

### Resharding

Изменение количества шардов или стратегии шардирования.

#### Когда нужен resharding:
- Шарды переполнены
- Изменились паттерны доступа
- Нужно добавить/удалить серверы

#### Подходы к resharding:

**1. Double-write migration:**

```python
class ReshardingMigrator:
    def __init__(self, old_router, new_router):
        self.old_router = old_router
        self.new_router = new_router
        self.migration_mode = False

    async def write(self, key, value):
        # Пишем в старую схему
        old_shard = self.old_router.get_shard(key)
        await old_shard.write(key, value)

        # Если миграция активна — пишем и в новую
        if self.migration_mode:
            new_shard = self.new_router.get_shard(key)
            if new_shard != old_shard:
                await new_shard.write(key, value)

    async def read(self, key):
        if self.migration_mode:
            # Сначала пробуем новую схему
            new_shard = self.new_router.get_shard(key)
            result = await new_shard.read(key)
            if result:
                return result

        # Fallback на старую схему
        old_shard = self.old_router.get_shard(key)
        return await old_shard.read(key)
```

**2. Shadow migration (постепенная):**

```
Phase 1: Dual-write active
┌────────────┐
│   Write    │──────┬──────▶ Old Shards (primary)
│  Request   │      │
└────────────┘      └──────▶ New Shards (shadow)

Phase 2: Background copy historical data
Old Shards ──────────────▶ New Shards

Phase 3: Switch read to new
┌────────────┐
│   Read     │──────────────▶ New Shards (primary)
│  Request   │
└────────────┘

Phase 4: Disable old shards
```

**3. Использование consistent hashing:**

```python
# При consistent hashing добавление узла
# перемещает минимум данных

ch = ConsistentHash(['shard1', 'shard2', 'shard3'])

# Добавляем 4-й шард
ch.add_node('shard4')

# Только ~25% ключей изменят свой шард
# (те, что теперь ближе к shard4 на кольце)
```

---

## Комбинирование: Репликация + Шардирование

В production системах репликация и шардирование используются **вместе**.

### Архитектура "Sharded + Replicated"

```
                              Load Balancer
                                   │
            ┌──────────────────────┼──────────────────────┐
            ▼                      ▼                      ▼
    ┌───────────────┐      ┌───────────────┐      ┌───────────────┐
    │   Shard 1     │      │   Shard 2     │      │   Shard 3     │
    │ ┌───────────┐ │      │ ┌───────────┐ │      │ ┌───────────┐ │
    │ │  Master   │ │      │ │  Master   │ │      │ │  Master   │ │
    │ └───────────┘ │      │ └───────────┘ │      │ └───────────┘ │
    │      │        │      │      │        │      │      │        │
    │ ┌────┴────┐   │      │ ┌────┴────┐   │      │ ┌────┴────┐   │
    │ ▼         ▼   │      │ ▼         ▼   │      │ ▼         ▼   │
    │┌─────┐ ┌─────┐│      │┌─────┐ ┌─────┐│      │┌─────┐ ┌─────┐│
    ││Rep 1│ │Rep 2││      ││Rep 1│ │Rep 2││      ││Rep 1│ │Rep 2││
    │└─────┘ └─────┘│      │└─────┘ └─────┘│      │└─────┘ └─────┘│
    └───────────────┘      └───────────────┘      └───────────────┘

    Users 1-1M             Users 1M-2M            Users 2M-3M
```

**Что это даёт:**
- **Шардирование**: масштабирование записи и объёма данных
- **Репликация**: высокая доступность и масштабирование чтения для каждого шарда

### Пример: MongoDB Sharded Cluster

```
┌─────────────────────────────────────────────────────────────────┐
│                         Config Servers                          │
│              (Replica Set: хранят метаданные)                   │
│                                                                 │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐               │
│  │ Config 1  │    │ Config 2  │    │ Config 3  │               │
│  └───────────┘    └───────────┘    └───────────┘               │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────┼─────────────────────────────────┐
│                         mongos                                 │
│                    (Query Routers)                             │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐              │
│  │  mongos 1 │    │  mongos 2 │    │  mongos 3 │              │
│  └───────────┘    └───────────┘    └───────────┘              │
└─────────────────────────────┼─────────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│    Shard 1      │  │    Shard 2      │  │    Shard 3      │
│  (Replica Set)  │  │  (Replica Set)  │  │  (Replica Set)  │
│                 │  │                 │  │                 │
│ Primary         │  │ Primary         │  │ Primary         │
│ Secondary       │  │ Secondary       │  │ Secondary       │
│ Secondary       │  │ Secondary       │  │ Secondary       │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

```javascript
// Включение шардирования
sh.enableSharding("mydb")

// Шардирование коллекции
sh.shardCollection("mydb.users", { "user_id": "hashed" })

// Проверка статуса
sh.status()
```

### Пример: Redis Cluster

```
┌─────────────────────────────────────────────────────────────────┐
│                      Redis Cluster                              │
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │ Slot 0-5460     │  │ Slot 5461-10922 │  │ Slot 10923-16383│  │
│  │                 │  │                 │  │                 │  │
│  │ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │  │
│  │ │   Master 1  │ │  │ │   Master 2  │ │  │ │   Master 3  │ │  │
│  │ └──────┬──────┘ │  │ └──────┬──────┘ │  │ └──────┬──────┘ │  │
│  │        │        │  │        │        │  │        │        │  │
│  │ ┌──────▼──────┐ │  │ ┌──────▼──────┐ │  │ ┌──────▼──────┐ │  │
│  │ │   Slave 1   │ │  │ │   Slave 2   │ │  │ │   Slave 3   │ │  │
│  │ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│                                                                 │
│  Hash Slots: 16384 слотов распределены между шардами           │
│  slot = CRC16(key) mod 16384                                    │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Создание кластера
redis-cli --cluster create \
    127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
    127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
    --cluster-replicas 1

# Добавление нового узла
redis-cli --cluster add-node 127.0.0.1:7006 127.0.0.1:7000

# Ребалансирование
redis-cli --cluster rebalance 127.0.0.1:7000
```

---

### Best Practices

#### 1. Планируйте шардирование заранее

```python
# Даже если сейчас один сервер — проектируйте код
# с учётом будущего шардирования

class UserRepository:
    def __init__(self, shard_router):
        self.router = shard_router

    async def get_user(self, user_id: int):
        shard = self.router.get_shard(user_id)
        return await shard.get_user(user_id)
```

#### 2. Выбирайте shard key осознанно

```python
# Анализируйте паттерны запросов ПЕРЕД выбором shard key

# Плохо: шардировать orders по order_id, если 90% запросов по user_id
# Хорошо: шардировать orders по user_id

# Compound shard key для сложных случаев:
shard_key = (tenant_id, user_id)
```

#### 3. Минимизируйте cross-shard операции

```python
# Храните связанные данные вместе
# User и его Orders на одном шарде

# Денормализуйте где возможно
order = {
    "order_id": 123,
    "user_id": 456,
    "user_name": "Alice",  # Копия из users
    "user_email": "alice@example.com"  # Копия из users
}
```

#### 4. Мониторьте распределение данных

```python
async def check_shard_balance():
    """Мониторинг распределения данных"""
    stats = {}
    for shard_name, shard in shards.items():
        stats[shard_name] = {
            'row_count': await shard.count_rows(),
            'disk_usage': await shard.get_disk_usage(),
            'qps': await shard.get_qps(),
            'latency_p99': await shard.get_latency_p99()
        }

    # Alert если разброс > 20%
    counts = [s['row_count'] for s in stats.values()]
    if max(counts) / min(counts) > 1.2:
        alert("Shard imbalance detected!")

    return stats
```

#### 5. Автоматизируйте failover

```python
# Используйте готовые решения:
# - Redis Sentinel / Redis Cluster
# - MongoDB Replica Sets
# - PostgreSQL Patroni / Stolon
# - MySQL Group Replication / Orchestrator

# Тестируйте failover регулярно (Chaos Engineering)
```

#### 6. Планируйте capacity заранее

```python
# Правило: шард должен использовать не более 70% ресурсов
# Оставляйте запас для:
# - Пиков нагрузки
# - Ребалансирования
# - Failover (реплика становится master)

MAX_SHARD_UTILIZATION = 0.7

def should_add_shard(current_shards):
    for shard in current_shards:
        if shard.cpu_usage > MAX_SHARD_UTILIZATION:
            return True
        if shard.disk_usage > MAX_SHARD_UTILIZATION:
            return True
    return False
```

---

### Примеры архитектур

#### 1. E-commerce платформа

```
┌─────────────────────────────────────────────────────────────────┐
│                        API Gateway                              │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│ User Service  │     │ Order Service │     │Product Service│
└───────────────┘     └───────────────┘     └───────────────┘
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│    Users DB   │     │   Orders DB   │     │  Products DB  │
│   Sharded by  │     │   Sharded by  │     │   Replicated  │
│    user_id    │     │    user_id    │     │   (not sharded)│
│               │     │               │     │               │
│ Each shard:   │     │ Each shard:   │     │ Master + 3    │
│ 1 Master +    │     │ 1 Master +    │     │ Read Replicas │
│ 2 Replicas    │     │ 2 Replicas    │     │               │
└───────────────┘     └───────────────┘     └───────────────┘

Почему так:
- Users и Orders шардированы по user_id (связанные данные вместе)
- Products не шардированы (относительно небольшой объём, много чтений)
```

#### 2. Social Network

```
┌─────────────────────────────────────────────────────────────────┐
│                     Timeline Service                            │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                      Redis Cluster                       │   │
│  │              (Timeline Cache, Sharded)                   │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────┼─────────────────────────────────┐
│                        Posts DB                                │
│                                                                │
│   ┌──────────────────────────────────────────────────────┐    │
│   │              Cassandra Cluster                        │    │
│   │         Sharded by (user_id, post_id)                │    │
│   │                                                       │    │
│   │    Replication Factor: 3 (leaderless)                │    │
│   │    Consistency: QUORUM for writes, ONE for reads     │    │
│   └──────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────┼─────────────────────────────────┐
│                     Graph DB (Followers)                       │
│                                                                │
│   ┌──────────────────────────────────────────────────────┐    │
│   │              Neo4j Cluster                            │    │
│   │         (Replicated, not sharded)                    │    │
│   │    Causal Clustering: 1 Leader + 2 Followers         │    │
│   └──────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────┘
```

#### 3. Multi-tenant SaaS

```
┌─────────────────────────────────────────────────────────────────┐
│                   Tenant Router Service                         │
│              (Определяет шард по tenant_id)                     │
└─────────────────────────────────────────────────────────────────┘
                              │
     ┌────────────────────────┼────────────────────────────┐
     ▼                        ▼                            ▼
┌──────────────┐       ┌──────────────┐       ┌───────────────────┐
│   Shard 1    │       │   Shard 2    │       │   Dedicated Shard │
│              │       │              │       │   (Enterprise)     │
│ Tenants:     │       │ Tenants:     │       │                   │
│ - Small Co 1 │       │ - Small Co 3 │       │ Tenant:           │
│ - Small Co 2 │       │ - Small Co 4 │       │ - Big Enterprise  │
│ - Medium Co  │       │ - Medium Co 2│       │                   │
│              │       │              │       │ Dedicated         │
│ Shared       │       │ Shared       │       │ resources         │
│ resources    │       │ resources    │       │                   │
└──────────────┘       └──────────────┘       └───────────────────┘

Особенности:
- Мелкие клиенты — на shared шардах
- Крупные клиенты — dedicated шард
- Изоляция данных между tenant'ами
```

---

## Резюме

| Аспект | Репликация | Шардирование |
|--------|------------|--------------|
| **Цель** | Высокая доступность, масштабирование чтения | Масштабирование записи и объёма данных |
| **Данные** | Полная копия на каждом узле | Часть данных на каждом узле |
| **Записи** | Все на master (обычно) | Распределены между шардами |
| **Чтения** | С любой реплики | С шарда, содержащего данные |
| **Failover** | Автоматический (при настройке) | Нужен для каждого шарда |
| **Сложность** | Средняя | Высокая |

**Когда использовать:**
- **Только репликация**: высокая доступность, read-heavy нагрузка, данные помещаются на один сервер
- **Только шардирование**: большой объём данных, write-heavy нагрузка (редко используется без репликации)
- **Репликация + Шардирование**: production системы с высокими требованиями к доступности и масштабируемости

---

## Дополнительные ресурсы

1. **Designing Data-Intensive Applications** by Martin Kleppmann — главы о репликации и партицировании
2. **Database Internals** by Alex Petrov — глубокое погружение в distributed storage
3. Документация баз данных:
   - [MongoDB Sharding](https://docs.mongodb.com/manual/sharding/)
   - [PostgreSQL Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)
   - [Cassandra Architecture](https://cassandra.apache.org/doc/latest/architecture/)
   - [Redis Cluster](https://redis.io/topics/cluster-tutorial)
