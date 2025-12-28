# Базы данных в системном дизайне

[prev: 15-api-gateway](./15-api-gateway.md) | [next: 17-database-types](./17-database-types.md)

---

## Введение

Базы данных — это фундаментальный компонент любой распределённой системы. Правильный выбор и архитектура хранилища данных определяют масштабируемость, производительность и надёжность всей системы. В этом разделе мы рассмотрим ключевые концепции и паттерны работы с данными в контексте системного дизайна.

---

## 1. Роль баз данных в системной архитектуре

### Центральное хранилище данных

База данных выступает единственным источником истины (Single Source of Truth) для приложения:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client    │────▶│   Server    │────▶│  Database   │
│  (Browser)  │     │   (API)     │     │  (Storage)  │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                    Business Logic
                    Validation
                    Transformation
```

### Основные функции баз данных в архитектуре

| Функция | Описание |
|---------|----------|
| **Персистентность** | Долговременное хранение данных |
| **Конкурентность** | Безопасный доступ множества клиентов |
| **Транзакции** | Атомарные операции над группами данных |
| **Индексация** | Быстрый поиск по различным критериям |
| **Репликация** | Копирование данных для отказоустойчивости |
| **Шардирование** | Распределение данных для масштабирования |

### Многоуровневая архитектура данных

В крупных системах данные часто проходят через несколько уровней:

```
┌────────────────────────────────────────────────────┐
│                    Application                      │
└────────────────────────────────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│    Cache    │   │   Search    │   │   Queue     │
│   (Redis)   │   │ (Elastic)   │   │   (Kafka)   │
└─────────────┘   └─────────────┘   └─────────────┘
         │                │                │
         └────────────────┼────────────────┘
                          ▼
              ┌─────────────────────┐
              │   Primary Database  │
              │    (PostgreSQL)     │
              └─────────────────────┘
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
    ┌─────────────┐         ┌─────────────┐
    │   Replica   │         │   Replica   │
    │   (Read)    │         │   (Read)    │
    └─────────────┘         └─────────────┘
```

---

## 2. ACID vs BASE

### ACID (Атомарность, Согласованность, Изоляция, Долговечность)

ACID — набор свойств, гарантирующих надёжную обработку транзакций.

#### Атомарность (Atomicity)

Транзакция выполняется полностью или не выполняется вообще:

```sql
-- Пример: перевод денег между счетами
BEGIN TRANSACTION;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- Если любой UPDATE падает — обе операции откатываются
COMMIT;
```

#### Согласованность (Consistency)

База данных переходит из одного валидного состояния в другое:

```sql
-- Constraints обеспечивают консистентность
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    balance DECIMAL(10, 2) CHECK (balance >= 0),  -- Баланс не может быть отрицательным
    user_id INTEGER REFERENCES users(id)          -- Внешний ключ
);
```

#### Изоляция (Isolation)

Параллельные транзакции не влияют друг на друга:

```sql
-- Транзакция A                    -- Транзакция B
BEGIN;                             BEGIN;
SELECT balance FROM accounts       SELECT balance FROM accounts
  WHERE id = 1;  -- 1000             WHERE id = 1;  -- 1000
UPDATE balance = 900...            UPDATE balance = 800...
COMMIT;                            COMMIT;
-- Результат зависит от уровня изоляции
```

**Уровни изоляции:**

| Уровень | Dirty Read | Non-repeatable Read | Phantom Read |
|---------|------------|---------------------|--------------|
| Read Uncommitted | Да | Да | Да |
| Read Committed | Нет | Да | Да |
| Repeatable Read | Нет | Нет | Да |
| Serializable | Нет | Нет | Нет |

#### Долговечность (Durability)

Закоммиченные данные не теряются даже при сбоях:

```
Write Request
     │
     ▼
┌─────────────┐
│   Buffer    │  ◀── Данные в памяти
│    Pool     │
└─────────────┘
     │
     ▼
┌─────────────┐
│  WAL (Log)  │  ◀── Write-Ahead Log на диске
└─────────────┘
     │
     ▼
┌─────────────┐
│  Data File  │  ◀── Основное хранилище
└─────────────┘
```

### BASE (Basically Available, Soft state, Eventually consistent)

BASE — альтернативная модель для распределённых систем, жертвующая строгой консистентностью ради доступности.

| Свойство | Описание |
|----------|----------|
| **Basically Available** | Система гарантирует доступность (может отдать устаревшие данные) |
| **Soft State** | Состояние может меняться со временем даже без входящих данных |
| **Eventually Consistent** | Данные станут консистентными через некоторое время |

#### Пример Eventually Consistent системы

```
┌─────────────────────────────────────────────────────────┐
│                   User updates profile                   │
└─────────────────────────────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
    ┌───────────┐   ┌───────────┐   ┌───────────┐
    │  Node A   │   │  Node B   │   │  Node C   │
    │ Updated   │   │   Stale   │   │   Stale   │
    │   ✓       │   │     ○     │   │     ○     │
    └───────────┘   └───────────┘   └───────────┘
            │               │               │
            └───────────────┼───────────────┘
                            │
                    Replication (async)
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
    ┌───────────┐   ┌───────────┐   ┌───────────┐
    │  Node A   │   │  Node B   │   │  Node C   │
    │ Updated   │   │ Updated   │   │ Updated   │
    │   ✓       │   │   ✓       │   │   ✓       │
    └───────────┘   └───────────┘   └───────────┘

        Eventually all nodes have consistent data
```

### ACID vs BASE: когда что использовать

| Сценарий | Рекомендация |
|----------|--------------|
| Финансовые транзакции | ACID (строгая консистентность критична) |
| Социальные сети (лайки, посты) | BASE (небольшая задержка допустима) |
| Инвентарь e-commerce | ACID (предотвращение overselling) |
| Аналитика и метрики | BASE (eventual consistency достаточна) |
| Банковские переводы | ACID (потеря денег недопустима) |
| Лента новостей | BASE (скорость важнее точности) |

---

## 3. CAP теорема

### Определение

CAP теорема утверждает, что распределённая система может одновременно обеспечить только два из трёх свойств:

```
                    Consistency
                        /\
                       /  \
                      /    \
                     /  CA  \
                    /________\
                   /\        /\
                  /  \      /  \
                 / CP \    / AP \
                /______\  /______\
        Partition        Availability
        Tolerance
```

### Три свойства

#### Consistency (Согласованность)

Все узлы видят одни и те же данные в один момент времени:

```python
# При записи на Node A
node_a.write("key", "value_v2")

# Немедленно при чтении с Node B
result = node_b.read("key")
assert result == "value_v2"  # Всегда актуальные данные
```

#### Availability (Доступность)

Каждый запрос получает ответ (успех или ошибка), без гарантии актуальности:

```python
# Система всегда отвечает
try:
    result = database.query("SELECT * FROM users")
    # Получаем данные (возможно устаревшие)
except NetworkError:
    # Ошибка сети - это ожидаемое поведение при partition
    pass
```

#### Partition Tolerance (Устойчивость к разделению)

Система продолжает работать при потере связи между узлами:

```
┌─────────────────────────────────────────────┐
│             Network Partition               │
│                                             │
│   ┌─────────┐         ┌─────────┐          │
│   │ Node A  │  ✕✕✕✕✕  │ Node B  │          │
│   │         │         │         │          │
│   └─────────┘         └─────────┘          │
│       │                   │                 │
│       ▼                   ▼                 │
│   Clients A           Clients B             │
│   (continue           (continue             │
│    working)            working)             │
│                                             │
└─────────────────────────────────────────────┘
```

### Типы систем по CAP

#### CP (Consistency + Partition Tolerance)

Жертвуют доступностью ради консистентности:

```python
# CP система при network partition
def write_data(key, value):
    # Должны получить подтверждение от большинства узлов
    acks = replicate_to_quorum(key, value)

    if acks < QUORUM:
        # Если кворум недостижим - отказываем в записи
        raise UnavailableError("Cannot reach quorum")

    return Success()
```

**Примеры CP систем:**
- MongoDB (в режиме по умолчанию)
- HBase
- Redis Cluster (при определённых настройках)
- Zookeeper, etcd, Consul

#### AP (Availability + Partition Tolerance)

Жертвуют консистентностью ради доступности:

```python
# AP система при network partition
def write_data(key, value):
    # Записываем локально
    local_store.write(key, value)

    # Асинхронно реплицируем
    try:
        async_replicate(key, value)
    except NetworkError:
        # Реплицируем позже, когда связь восстановится
        queue_for_later_replication(key, value)

    return Success()  # Всегда успешно
```

**Примеры AP систем:**
- Cassandra
- CouchDB
- DynamoDB (при определённых настройках)
- Riak

#### CA (Consistency + Availability)

Возможна только при отсутствии network partition (не в распределённых системах):

```
┌─────────────────────────────────────────┐
│        Single Node Database             │
│                                         │
│   ┌─────────────────────────────┐      │
│   │       PostgreSQL            │      │
│   │                             │      │
│   │   Consistent ✓              │      │
│   │   Available ✓               │      │
│   │   No Partitions (single)    │      │
│   └─────────────────────────────┘      │
│                                         │
└─────────────────────────────────────────┘
```

**Примеры CA систем:**
- Традиционные RDBMS на одном сервере
- SQLite

### Практическое применение CAP

```
Выбор системы по требованиям:

┌─────────────────────────────────────────────────────────┐
│                    Ваши требования                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   Нужна 100% консистентность?                          │
│        │                                                │
│        ├── Да ──▶ CP система (MongoDB, HBase)          │
│        │         Готовы к временной недоступности       │
│        │                                                │
│        └── Нет ─▶ AP система (Cassandra, DynamoDB)     │
│                  Готовы к eventual consistency          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Классификация баз данных

### 4.1 Реляционные базы данных (SQL)

#### Характеристики

- Строгая схема данных
- ACID транзакции
- SQL как язык запросов
- Нормализованные данные
- Поддержка JOIN операций

#### Примеры

| СУБД | Особенности |
|------|-------------|
| **PostgreSQL** | Расширяемость, JSON поддержка, сложные запросы |
| **MySQL/MariaDB** | Простота, широкая поддержка, репликация |
| **Oracle** | Enterprise функции, высокая надёжность |
| **SQL Server** | Интеграция с Microsoft экосистемой |
| **SQLite** | Встраиваемая, файловая БД |

#### Пример схемы и запроса

```sql
-- Нормализованная схема
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    total DECIMAL(10, 2),
    status VARCHAR(50)
);

CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(id),
    product_id INTEGER REFERENCES products(id),
    quantity INTEGER,
    price DECIMAL(10, 2)
);

-- Сложный JOIN запрос
SELECT
    u.email,
    COUNT(o.id) as order_count,
    SUM(o.total) as total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.created_at > NOW() - INTERVAL '30 days'
GROUP BY u.id
HAVING SUM(o.total) > 1000
ORDER BY total_spent DESC;
```

### 4.2 NoSQL базы данных

#### Document Stores (Документные БД)

Хранят данные в виде JSON/BSON документов:

```javascript
// MongoDB - пример документа
{
    "_id": ObjectId("507f1f77bcf86cd799439011"),
    "username": "john_doe",
    "email": "john@example.com",
    "profile": {
        "firstName": "John",
        "lastName": "Doe",
        "age": 30
    },
    "orders": [
        {
            "orderId": "ORD-001",
            "items": [
                {"product": "Laptop", "price": 999.99},
                {"product": "Mouse", "price": 29.99}
            ],
            "total": 1029.98
        }
    ],
    "tags": ["premium", "verified"]
}
```

**Когда использовать:**
- Гибкая или эволюционирующая схема
- Иерархические данные
- Контент-менеджмент системы
- Каталоги продуктов

**Примеры:** MongoDB, CouchDB, Firebase Firestore

#### Key-Value Stores (Ключ-значение)

Простейшая модель: ключ → значение:

```python
# Redis примеры
import redis

r = redis.Redis()

# Простые значения
r.set("user:1001:name", "John Doe")
r.get("user:1001:name")  # "John Doe"

# С TTL (время жизни)
r.setex("session:abc123", 3600, "user_data")

# Структуры данных
r.hset("user:1001", mapping={
    "name": "John",
    "email": "john@example.com",
    "visits": 42
})

# Списки для очередей
r.lpush("queue:emails", "message1", "message2")
r.rpop("queue:emails")  # "message1"

# Sorted Sets для лидербордов
r.zadd("leaderboard:game1", {"player1": 100, "player2": 85})
r.zrange("leaderboard:game1", 0, 9, withscores=True)
```

**Когда использовать:**
- Кэширование
- Сессии пользователей
- Очереди сообщений
- Счётчики и метрики
- Лидерборды

**Примеры:** Redis, Memcached, Amazon DynamoDB, etcd

#### Column-Family Stores (Колоночные БД)

Оптимизированы для аналитических запросов по колонкам:

```
Row-oriented (OLTP):
┌─────────┬───────┬─────┬────────┐
│ user_id │ name  │ age │ city   │
├─────────┼───────┼─────┼────────┤
│ 1       │ John  │ 30  │ NYC    │
│ 2       │ Jane  │ 25  │ LA     │
│ 3       │ Bob   │ 35  │ Chicago│
└─────────┴───────┴─────┴────────┘
Хранение: [1,John,30,NYC][2,Jane,25,LA][3,Bob,35,Chicago]

Column-oriented (OLAP):
user_id: [1, 2, 3]
name:    [John, Jane, Bob]
age:     [30, 25, 35]
city:    [NYC, LA, Chicago]

Преимущество: SELECT AVG(age) FROM users
Читает только колонку age, а не все данные
```

**Wide-Column Stores (Cassandra, HBase):**

```cql
-- Cassandra CQL
CREATE TABLE user_activity (
    user_id UUID,
    activity_date DATE,
    activity_time TIMESTAMP,
    activity_type TEXT,
    details TEXT,
    PRIMARY KEY ((user_id, activity_date), activity_time)
) WITH CLUSTERING ORDER BY (activity_time DESC);

-- Эффективный запрос по partition key
SELECT * FROM user_activity
WHERE user_id = ? AND activity_date = ?
LIMIT 100;
```

**Когда использовать:**
- Time-series данные
- Логирование событий
- IoT данные
- Высокая пропускная способность записи

**Примеры:**
- Wide-column: Cassandra, HBase, ScyllaDB
- Columnar (OLAP): ClickHouse, Apache Druid, Vertica

#### Graph Databases (Графовые БД)

Оптимизированы для связей между сущностями:

```
                    ┌─────────┐
                    │  Alice  │
                    │ (User)  │
                    └────┬────┘
              ┌─────────┴─────────┐
         FOLLOWS              FOLLOWS
              │                   │
        ┌─────▼─────┐       ┌─────▼─────┐
        │    Bob    │       │  Charlie  │
        │  (User)   │       │  (User)   │
        └─────┬─────┘       └───────────┘
              │
           LIKES
              │
        ┌─────▼─────┐
        │  Post #1  │
        │  (Post)   │
        └───────────┘
```

```cypher
// Neo4j Cypher Query Language

// Создание узлов и связей
CREATE (alice:User {name: 'Alice', age: 30})
CREATE (bob:User {name: 'Bob', age: 25})
CREATE (alice)-[:FOLLOWS]->(bob)
CREATE (alice)-[:FOLLOWS]->(charlie)

// Найти друзей друзей (2-й уровень)
MATCH (user:User {name: 'Alice'})-[:FOLLOWS*2]->(fof:User)
WHERE NOT (user)-[:FOLLOWS]->(fof) AND user <> fof
RETURN DISTINCT fof.name

// Найти кратчайший путь между пользователями
MATCH path = shortestPath(
    (a:User {name: 'Alice'})-[:FOLLOWS*]-(b:User {name: 'Dave'})
)
RETURN path
```

**Когда использовать:**
- Социальные сети
- Рекомендательные системы
- Fraud detection
- Knowledge graphs
- Сетевая топология

**Примеры:** Neo4j, Amazon Neptune, ArangoDB, JanusGraph

### 4.3 NewSQL

Сочетают ACID гарантии SQL с горизонтальной масштабируемостью NoSQL:

```
┌─────────────────────────────────────────────────────────┐
│                      NewSQL                              │
├─────────────────────────────────────────────────────────┤
│                                                          │
│   SQL + ACID   +   Horizontal Scaling                   │
│        ▲                    ▲                           │
│        │                    │                           │
│   From RDBMS           From NoSQL                       │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Примеры:**

| СУБД | Особенности |
|------|-------------|
| **CockroachDB** | PostgreSQL совместимость, geo-распределённость |
| **TiDB** | MySQL совместимость, HTAP (OLTP + OLAP) |
| **Google Spanner** | Глобальная консистентность, TrueTime |
| **YugabyteDB** | PostgreSQL совместимость, Cassandra API |
| **VoltDB** | In-memory, экстремально низкая латентность |

```sql
-- CockroachDB: стандартный PostgreSQL синтаксис
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email STRING UNIQUE,
    region STRING,
    created_at TIMESTAMP DEFAULT now()
);

-- Geo-partitioning для соответствия требованиям локализации данных
ALTER TABLE users PARTITION BY LIST (region) (
    PARTITION europe VALUES IN ('eu-west-1', 'eu-central-1'),
    PARTITION america VALUES IN ('us-east-1', 'us-west-2')
);
```

### 4.4 Time-Series Databases

Оптимизированы для данных с временными метками:

```
Time Series Data Pattern:

timestamp           | sensor_id | temperature | humidity
--------------------|-----------|-------------|----------
2024-01-15 10:00:00 | sensor_1  | 22.5        | 45.2
2024-01-15 10:00:01 | sensor_1  | 22.6        | 45.1
2024-01-15 10:00:02 | sensor_1  | 22.5        | 45.3
2024-01-15 10:00:00 | sensor_2  | 18.2        | 60.1
...
```

**Оптимизации Time-Series БД:**

1. **Compression** — эффективное сжатие временных данных
2. **Downsampling** — агрегация старых данных
3. **Retention policies** — автоматическое удаление старых данных
4. **Continuous queries** — предрассчитанные агрегаты

```sql
-- InfluxDB пример
-- Создание базы с политикой хранения
CREATE DATABASE metrics WITH DURATION 30d REPLICATION 1

-- Запись данных
INSERT cpu,host=server01,region=us-west value=0.64 1434055562000000000

-- Запрос с агрегацией
SELECT MEAN("value")
FROM "cpu"
WHERE time > now() - 1h
GROUP BY time(5m), "host"

-- Continuous Query для даунсемплинга
CREATE CONTINUOUS QUERY "cq_1h" ON "metrics"
BEGIN
  SELECT mean("value") AS "mean_value"
  INTO "metrics_1h"
  FROM "cpu"
  GROUP BY time(1h), *
END
```

**Примеры:** InfluxDB, TimescaleDB (на PostgreSQL), Prometheus, QuestDB, ClickHouse

---

## 5. Критерии выбора базы данных

### 5.1 Консистентность vs Доступность

```
                    Strong Consistency
                           │
    ┌──────────────────────┼──────────────────────┐
    │                      │                      │
    │     CP Systems       │     CA Systems       │
    │                      │                      │
    │   • MongoDB          │   • PostgreSQL       │
    │   • HBase            │   • MySQL            │
    │   • Zookeeper        │   (single node)      │
    │                      │                      │
Partition              No Partition           High
Tolerant               Tolerant           Availability
    │                                           │
    │                      │                    │
    │     AP Systems       │                    │
    │                      │                    │
    │   • Cassandra        │                    │
    │   • DynamoDB         │                    │
    │   • CouchDB          │                    │
    │                      │                    │
    └──────────────────────┼────────────────────┘
                           │
                  Eventual Consistency
```

### 5.2 Модель данных

| Модель данных | Подходящая БД | Пример сценария |
|---------------|---------------|-----------------|
| Табличная с отношениями | PostgreSQL, MySQL | ERP, финансы |
| Документы (JSON) | MongoDB, CouchDB | CMS, каталоги |
| Ключ-значение | Redis, DynamoDB | Кэш, сессии |
| Графы | Neo4j, Neptune | Соцсети, рекомендации |
| Временные ряды | InfluxDB, TimescaleDB | IoT, мониторинг |
| Широкие колонки | Cassandra, HBase | Логи, события |

### 5.3 Масштабирование

#### Вертикальное масштабирование

```
Before:                      After:
┌─────────────┐             ┌─────────────┐
│   4 CPU     │             │   32 CPU    │
│   16 GB RAM │   ────▶     │   256 GB RAM│
│   500 GB SSD│             │   4 TB NVMe │
└─────────────┘             └─────────────┘

Подходит для: PostgreSQL, MySQL, Oracle
Ограничения: есть предел мощности одного сервера
```

#### Горизонтальное масштабирование

```
Before:                      After:
┌─────────────┐             ┌─────────┐ ┌─────────┐ ┌─────────┐
│   Node 1    │             │ Node 1  │ │ Node 2  │ │ Node 3  │
│   100%      │   ────▶     │  33%    │ │  33%    │ │  33%    │
│   load      │             │  load   │ │  load   │ │  load   │
└─────────────┘             └─────────┘ └─────────┘ └─────────┘

Подходит для: Cassandra, MongoDB, CockroachDB
Преимущество: практически неограниченное масштабирование
```

### 5.4 Производительность

#### Характеристики нагрузки

```python
# Определение характера нагрузки
workload_characteristics = {
    "read_heavy": {
        "ratio": "90% reads, 10% writes",
        "solutions": ["Read replicas", "Caching", "CDN"],
        "databases": ["PostgreSQL + Redis", "MongoDB"]
    },
    "write_heavy": {
        "ratio": "10% reads, 90% writes",
        "solutions": ["Write-optimized storage", "Async writes", "Batching"],
        "databases": ["Cassandra", "InfluxDB", "Kafka + DB"]
    },
    "mixed": {
        "ratio": "50% reads, 50% writes",
        "solutions": ["CQRS", "Separate read/write paths"],
        "databases": ["PostgreSQL", "CockroachDB"]
    }
}
```

### 5.5 Матрица выбора

```
┌────────────────────────────────────────────────────────────────┐
│                    Матрица выбора БД                            │
├─────────────────┬──────────────────────────────────────────────┤
│ Требование      │ Рекомендуемые решения                        │
├─────────────────┼──────────────────────────────────────────────┤
│ ACID + SQL      │ PostgreSQL, MySQL, CockroachDB               │
├─────────────────┼──────────────────────────────────────────────┤
│ Гибкая схема    │ MongoDB, CouchDB                             │
├─────────────────┼──────────────────────────────────────────────┤
│ Высокая запись  │ Cassandra, ScyllaDB, Kafka                   │
├─────────────────┼──────────────────────────────────────────────┤
│ Кэширование     │ Redis, Memcached                             │
├─────────────────┼──────────────────────────────────────────────┤
│ Полнотекстовый  │ Elasticsearch, PostgreSQL FTS                │
│ поиск           │                                              │
├─────────────────┼──────────────────────────────────────────────┤
│ Графовые связи  │ Neo4j, Amazon Neptune                        │
├─────────────────┼──────────────────────────────────────────────┤
│ Time-series     │ InfluxDB, TimescaleDB, ClickHouse            │
├─────────────────┼──────────────────────────────────────────────┤
│ Глобальная      │ CockroachDB, Google Spanner                  │
│ распределённость│                                              │
└─────────────────┴──────────────────────────────────────────────┘
```

---

## 6. Паттерны работы с данными

### 6.1 CQRS (Command Query Responsibility Segregation)

Разделение операций чтения и записи на разные модели:

```
┌─────────────────────────────────────────────────────────────┐
│                         CQRS Pattern                         │
└─────────────────────────────────────────────────────────────┘

                    ┌─────────────┐
                    │   Client    │
                    └──────┬──────┘
                           │
           ┌───────────────┴───────────────┐
           │                               │
     Commands                          Queries
     (Write)                           (Read)
           │                               │
           ▼                               ▼
    ┌─────────────┐                ┌─────────────┐
    │  Command    │                │   Query     │
    │  Handler    │                │   Handler   │
    └──────┬──────┘                └──────┬──────┘
           │                               │
           ▼                               ▼
    ┌─────────────┐                ┌─────────────┐
    │   Write     │  ──────────▶   │    Read     │
    │   Model     │   Sync/Async   │    Model    │
    │ (PostgreSQL)│                │   (Redis/   │
    │             │                │   Elastic)  │
    └─────────────┘                └─────────────┘
```

**Реализация на Python:**

```python
# Command side
class CreateOrderCommand:
    user_id: str
    items: list[OrderItem]

class OrderCommandHandler:
    def __init__(self, write_db, event_bus):
        self.write_db = write_db
        self.event_bus = event_bus

    def handle(self, command: CreateOrderCommand):
        # Записываем в основную БД
        order = Order.create(command.user_id, command.items)
        self.write_db.save(order)

        # Публикуем событие для синхронизации read model
        self.event_bus.publish(OrderCreatedEvent(order))

        return order.id

# Query side
class OrderQueryHandler:
    def __init__(self, read_db):
        self.read_db = read_db  # Оптимизированное для чтения хранилище

    def get_user_orders(self, user_id: str) -> list[OrderDTO]:
        # Быстрое чтение из денормализованного хранилища
        return self.read_db.get_orders_by_user(user_id)

    def get_order_details(self, order_id: str) -> OrderDetailDTO:
        return self.read_db.get_order_with_items(order_id)

# Event handler для синхронизации
class OrderEventHandler:
    def __init__(self, read_db):
        self.read_db = read_db

    def handle_order_created(self, event: OrderCreatedEvent):
        # Обновляем read model
        self.read_db.upsert_order_view(event.order.to_read_model())
```

**Когда использовать CQRS:**
- Высокая нагрузка на чтение
- Сложные запросы агрегации
- Разные требования к read/write операциям
- Необходимость разных представлений данных

### 6.2 Event Sourcing

Хранение состояния как последовательности событий:

```
┌─────────────────────────────────────────────────────────────┐
│                    Event Sourcing                            │
└─────────────────────────────────────────────────────────────┘

Traditional (State):          Event Sourcing:
┌─────────────────┐          ┌─────────────────────────────┐
│ Account         │          │ Events Stream               │
│ ─────────────── │          │ ─────────────────────────── │
│ balance: 150    │          │ 1. AccountCreated(0)        │
│                 │    vs    │ 2. MoneyDeposited(100)      │
│                 │          │ 3. MoneyDeposited(100)      │
│                 │          │ 4. MoneyWithdrawn(50)       │
│                 │          │ Current: 0+100+100-50 = 150 │
└─────────────────┘          └─────────────────────────────┘
```

**Реализация:**

```python
from dataclasses import dataclass
from datetime import datetime
from abc import ABC, abstractmethod

# События
@dataclass
class Event(ABC):
    aggregate_id: str
    timestamp: datetime
    version: int

@dataclass
class AccountCreated(Event):
    initial_balance: float

@dataclass
class MoneyDeposited(Event):
    amount: float

@dataclass
class MoneyWithdrawn(Event):
    amount: float

# Агрегат
class BankAccount:
    def __init__(self, account_id: str):
        self.id = account_id
        self.balance = 0
        self.version = 0
        self._changes = []

    # Применение событий (replay)
    def apply(self, event: Event):
        if isinstance(event, AccountCreated):
            self.balance = event.initial_balance
        elif isinstance(event, MoneyDeposited):
            self.balance += event.amount
        elif isinstance(event, MoneyWithdrawn):
            self.balance -= event.amount
        self.version = event.version

    # Команды генерируют события
    def deposit(self, amount: float):
        if amount <= 0:
            raise ValueError("Amount must be positive")
        event = MoneyDeposited(
            aggregate_id=self.id,
            timestamp=datetime.now(),
            version=self.version + 1,
            amount=amount
        )
        self.apply(event)
        self._changes.append(event)

    def withdraw(self, amount: float):
        if amount > self.balance:
            raise ValueError("Insufficient funds")
        event = MoneyWithdrawn(
            aggregate_id=self.id,
            timestamp=datetime.now(),
            version=self.version + 1,
            amount=amount
        )
        self.apply(event)
        self._changes.append(event)

    def get_uncommitted_changes(self):
        return self._changes

    def mark_changes_as_committed(self):
        self._changes = []

# Event Store
class EventStore:
    def __init__(self, db):
        self.db = db

    def save_events(self, aggregate_id: str, events: list[Event], expected_version: int):
        # Оптимистичная блокировка
        current_version = self.db.get_version(aggregate_id)
        if current_version != expected_version:
            raise ConcurrencyError("Aggregate was modified")

        for event in events:
            self.db.append_event(aggregate_id, event)

    def get_events(self, aggregate_id: str) -> list[Event]:
        return self.db.get_events_for_aggregate(aggregate_id)

    def rebuild_aggregate(self, aggregate_id: str) -> BankAccount:
        account = BankAccount(aggregate_id)
        events = self.get_events(aggregate_id)
        for event in events:
            account.apply(event)
        return account
```

**Преимущества Event Sourcing:**
- Полная история изменений (audit log)
- Возможность "отмотать" состояние на любой момент
- Debug и анализ (что привело к этому состоянию?)
- Легко добавлять новые projections

**Недостатки:**
- Сложность реализации
- Eventual consistency
- Необходимость snapshots для производительности

### 6.3 Database per Service

В микросервисной архитектуре каждый сервис имеет свою БД:

```
┌─────────────────────────────────────────────────────────────┐
│                  Database per Service                        │
└─────────────────────────────────────────────────────────────┘

      ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
      │    User     │     │   Order     │     │  Inventory  │
      │   Service   │     │   Service   │     │   Service   │
      └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
             │                   │                   │
             ▼                   ▼                   ▼
      ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
      │  PostgreSQL │     │   MongoDB   │     │    Redis    │
      │   (Users)   │     │  (Orders)   │     │ (Inventory) │
      └─────────────┘     └─────────────┘     └─────────────┘

                              ▲
                              │
                    ┌─────────┴─────────┐
                    │   Shared Nothing   │
                    │   Architecture     │
                    └───────────────────┘
```

**Преимущества:**
- Независимое масштабирование
- Выбор подходящей БД для каждого сервиса
- Изоляция сбоев
- Независимые релизы

**Вызовы:**

1. **Распределённые транзакции:**

```python
# Saga Pattern для распределённых транзакций
class OrderSaga:
    def create_order(self, order_data):
        try:
            # Шаг 1: Создать заказ
            order_id = order_service.create(order_data)

            # Шаг 2: Зарезервировать товары
            inventory_service.reserve(order_data.items)

            # Шаг 3: Списать деньги
            payment_service.charge(order_data.user_id, order_data.total)

            # Шаг 4: Подтвердить заказ
            order_service.confirm(order_id)

        except InventoryError:
            # Компенсация: удалить заказ
            order_service.cancel(order_id)
            raise

        except PaymentError:
            # Компенсация: вернуть резерв и удалить заказ
            inventory_service.release(order_data.items)
            order_service.cancel(order_id)
            raise
```

2. **Запросы между сервисами:**

```python
# API Composition для запросов между сервисами
class OrderDetailsComposer:
    def get_order_with_details(self, order_id):
        # Получаем данные из разных сервисов
        order = order_service.get(order_id)
        user = user_service.get(order.user_id)
        items = inventory_service.get_items(order.item_ids)

        # Композируем ответ
        return {
            "order": order,
            "user": user,
            "items": items
        }
```

---

## 7. Индексы и их роль в производительности

### Типы индексов

#### B-Tree индексы (наиболее распространённые)

```
B-Tree Structure:

                    ┌───────────────────────┐
                    │    [30]    [60]       │  Root
                    └───────────────────────┘
                   /          |          \
         ┌────────┐    ┌────────┐    ┌────────┐
         │[10][20]│    │[40][50]│    │[70][80]│  Internal
         └────────┘    └────────┘    └────────┘
         /   |   \      /   |   \     /   |   \
       ┌──┐┌──┐┌──┐  ┌──┐┌──┐┌──┐ ┌──┐┌──┐┌──┐
       │5 ││15││25│  │35││45││55│ │65││75││85│  Leaves
       └──┘└──┘└──┘  └──┘└──┘└──┘ └──┘└──┘└──┘

Сложность поиска: O(log n)
```

```sql
-- Создание B-Tree индекса
CREATE INDEX idx_users_email ON users(email);

-- Составной индекс (порядок важен!)
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);

-- Частичный индекс
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;
```

#### Hash индексы

```
Hash Index:

email "alice@example.com" ──▶ hash() ──▶ bucket_42 ──▶ row_id: 1001
email "bob@example.com"   ──▶ hash() ──▶ bucket_17 ──▶ row_id: 1002

Сложность: O(1) для equality
Ограничение: не поддерживает range queries
```

```sql
-- PostgreSQL: создание hash индекса
CREATE INDEX idx_users_email_hash ON users USING hash(email);

-- Хорошо для: WHERE email = 'alice@example.com'
-- Плохо для: WHERE email LIKE 'alice%'
```

#### GiST и GIN индексы

```sql
-- GIN для полнотекстового поиска
CREATE INDEX idx_articles_search ON articles
USING gin(to_tsvector('russian', title || ' ' || content));

-- Поиск
SELECT * FROM articles
WHERE to_tsvector('russian', title || ' ' || content) @@
      plainto_tsquery('russian', 'системный дизайн');

-- GiST для геоданных
CREATE INDEX idx_locations_geom ON locations USING gist(geom);

-- Поиск ближайших точек
SELECT * FROM locations
ORDER BY geom <-> ST_MakePoint(37.6173, 55.7558)
LIMIT 10;
```

### Влияние индексов на производительность

```
Без индекса (Full Table Scan):
┌────────────────────────────────────────────┐
│ SELECT * FROM users WHERE email = 'x'      │
│                                            │
│ ┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐  │
│ │✗ ││✗ ││✗ ││✗ ││✓ ││✗ ││✗ ││✗ ││✗ ││✗ │  │
│ └──┘└──┘└──┘└──┘└──┘└──┘└──┘└──┘└──┘└──┘  │
│                                            │
│ Проверено: 10 строк (100%)                 │
│ Сложность: O(n)                            │
└────────────────────────────────────────────┘

С индексом (Index Scan):
┌────────────────────────────────────────────┐
│ SELECT * FROM users WHERE email = 'x'      │
│                                            │
│ Index lookup: email = 'x' ──▶ row_id: 5    │
│                                            │
│ Проверено: 1 строка (index) + 1 строка     │
│ Сложность: O(log n)                        │
└────────────────────────────────────────────┘
```

### Анализ производительности запросов

```sql
-- EXPLAIN ANALYZE показывает план выполнения
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 123
AND created_at > '2024-01-01';

-- Результат:
-- Index Scan using idx_orders_user_date on orders
--   Index Cond: (user_id = 123) AND (created_at > '2024-01-01')
--   Actual time: 0.025..0.031 ms
--   Rows: 42
```

### Правила индексирования

```
✓ Индексируйте:
  • Колонки в WHERE условиях
  • Колонки в JOIN условиях
  • Колонки в ORDER BY
  • Колонки с высокой selectivity

✗ Не индексируйте:
  • Таблицы с малым количеством строк
  • Колонки с низкой selectivity (пол, boolean)
  • Таблицы с частыми INSERT/UPDATE
  • Колонки, которые редко используются в запросах
```

---

## 8. Нормализация vs Денормализация

### Нормализация

Процесс организации данных для минимизации избыточности:

```
Ненормализованные данные:
┌────────────────────────────────────────────────────────────┐
│ orders                                                      │
├──────┬───────────┬─────────────────┬──────────┬────────────┤
│ id   │ user_name │ user_email      │ product  │ price      │
├──────┼───────────┼─────────────────┼──────────┼────────────┤
│ 1    │ John Doe  │ john@mail.com   │ Laptop   │ 999.99     │
│ 2    │ John Doe  │ john@mail.com   │ Mouse    │ 29.99      │
│ 3    │ Jane Doe  │ jane@mail.com   │ Laptop   │ 999.99     │
└──────┴───────────┴─────────────────┴──────────┴────────────┘

Проблема: данные дублируются, update anomalies

Нормализованные данные (3NF):
┌─────────────────────┐     ┌─────────────────────┐
│ users               │     │ products            │
├──────┬──────────────┤     ├──────┬──────────────┤
│ id   │ email        │     │ id   │ name   │price│
├──────┼──────────────┤     ├──────┼──────────────┤
│ 1    │ john@mail.com│     │ 1    │ Laptop │999  │
│ 2    │ jane@mail.com│     │ 2    │ Mouse  │29   │
└──────┴──────────────┘     └──────┴──────────────┘
           │                         │
           └──────────┬──────────────┘
                      │
              ┌───────▼───────┐
              │ orders        │
              ├───────────────┤
              │id│user│product│
              ├───────────────┤
              │1 │ 1  │  1    │
              │2 │ 1  │  2    │
              │3 │ 2  │  1    │
              └───────────────┘
```

### Денормализация

Намеренное добавление избыточности для производительности:

```sql
-- Нормализованный запрос (медленный)
SELECT
    o.id,
    u.name,
    u.email,
    p.name as product_name,
    p.price
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN products p ON o.product_id = p.id
WHERE u.id = 123;

-- Денормализованная таблица (быстрый)
CREATE TABLE order_details_view (
    order_id INTEGER,
    user_id INTEGER,
    user_name VARCHAR(100),     -- Дублирование
    user_email VARCHAR(255),    -- Дублирование
    product_name VARCHAR(100),  -- Дублирование
    product_price DECIMAL,      -- Дублирование
    created_at TIMESTAMP
);

-- Простой запрос без JOIN
SELECT * FROM order_details_view WHERE user_id = 123;
```

### Когда применять

| Нормализация | Денормализация |
|--------------|----------------|
| OLTP системы | OLAP / аналитика |
| Частые UPDATE | Частые SELECT |
| Небольшие объёмы | Большие объёмы данных |
| Важна консистентность | Важна скорость чтения |
| Мало JOIN запросов | Много сложных JOIN |

### Materialized Views как компромисс

```sql
-- Создание материализованного представления
CREATE MATERIALIZED VIEW order_summary AS
SELECT
    u.id as user_id,
    u.email,
    COUNT(o.id) as order_count,
    SUM(o.total) as total_spent,
    MAX(o.created_at) as last_order_date
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.email;

-- Создание индекса на view
CREATE INDEX idx_order_summary_user ON order_summary(user_id);

-- Обновление данных
REFRESH MATERIALIZED VIEW CONCURRENTLY order_summary;

-- Быстрый запрос
SELECT * FROM order_summary WHERE user_id = 123;
```

---

## 9. Best Practices выбора БД для разных сценариев

### Сценарий 1: E-commerce платформа

```
Требования:
• Транзакционная консистентность (заказы, платежи)
• Полнотекстовый поиск по продуктам
• Рекомендации (связи между продуктами)
• Кэширование популярных товаров

Решение:
┌────────────────────────────────────────────────────┐
│                 E-commerce Stack                    │
├────────────────────────────────────────────────────┤
│                                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐  │
│  │ PostgreSQL  │  │ Elasticsearch│  │  Redis   │  │
│  │ (Orders,    │  │ (Product    │  │ (Cache,  │  │
│  │  Users,     │  │  Search)    │  │  Sessions│  │
│  │  Payments)  │  │             │  │  Cart)   │  │
│  └─────────────┘  └─────────────┘  └──────────┘  │
│        │                │               │         │
│        └────────────────┼───────────────┘         │
│                         │                         │
│                  ┌──────▼──────┐                  │
│                  │    Neo4j    │                  │
│                  │(Recommend-  │                  │
│                  │  ations)    │                  │
│                  └─────────────┘                  │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Сценарий 2: Социальная сеть

```
Требования:
• Миллионы пользователей
• Лента новостей в реальном времени
• Графовые связи (друзья, подписки)
• Высокая нагрузка на чтение

Решение:
┌────────────────────────────────────────────────────┐
│               Social Network Stack                  │
├────────────────────────────────────────────────────┤
│                                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐  │
│  │ Cassandra   │  │   Neo4j     │  │  Redis   │  │
│  │ (Posts,     │  │ (Social     │  │ (Feed    │  │
│  │  Activity)  │  │  Graph)     │  │  Cache)  │  │
│  └─────────────┘  └─────────────┘  └──────────┘  │
│        │                │               │         │
│        └────────────────┼───────────────┘         │
│                         │                         │
│  ┌─────────────────────────────────────────────┐ │
│  │                   Kafka                      │ │
│  │         (Real-time event streaming)          │ │
│  └─────────────────────────────────────────────┘ │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Сценарий 3: IoT платформа

```
Требования:
• Миллионы устройств
• Потоковые данные с временными метками
• Аналитика в реальном времени
• Длительное хранение с агрегацией

Решение:
┌────────────────────────────────────────────────────┐
│                   IoT Stack                         │
├────────────────────────────────────────────────────┤
│                                                    │
│  Devices ──▶ Kafka ──▶ Stream Processing          │
│                              │                     │
│              ┌───────────────┼───────────────┐    │
│              ▼               ▼               ▼    │
│       ┌──────────┐    ┌──────────┐    ┌────────┐ │
│       │ InfluxDB │    │ClickHouse│    │  Redis │ │
│       │ (Raw     │    │ (Analytics│   │ (Real- │ │
│       │  Metrics)│    │  OLAP)   │    │  time) │ │
│       └──────────┘    └──────────┘    └────────┘ │
│              │               │               │    │
│              └───────────────┴───────────────┘    │
│                              │                     │
│                       ┌──────▼──────┐             │
│                       │   Grafana   │             │
│                       │(Dashboards) │             │
│                       └─────────────┘             │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Сценарий 4: Финтех / банкинг

```
Требования:
• Строгая консистентность (ACID)
• Audit trail (история всех операций)
• Высокая доступность
• Соответствие регуляторным требованиям

Решение:
┌────────────────────────────────────────────────────┐
│                  Fintech Stack                      │
├────────────────────────────────────────────────────┤
│                                                    │
│  ┌─────────────────────────────────────────────┐  │
│  │              PostgreSQL (Primary)            │  │
│  │    ACID transactions, Event Sourcing        │  │
│  └─────────────────────────────────────────────┘  │
│                        │                           │
│         ┌──────────────┴──────────────┐           │
│         ▼                             ▼           │
│  ┌─────────────┐              ┌─────────────┐    │
│  │  Replica 1  │              │  Replica 2  │    │
│  │  (Sync)     │              │  (Sync)     │    │
│  └─────────────┘              └─────────────┘    │
│                                                    │
│  ┌─────────────────────────────────────────────┐  │
│  │              Event Store (Kafka)             │  │
│  │         Complete audit trail                 │  │
│  └─────────────────────────────────────────────┘  │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## 10. Примеры архитектурных решений

### Пример 1: Система бронирования (избежание double booking)

```python
# Проблема: два пользователя бронируют одно место одновременно

# Решение 1: Pessimistic Locking
def book_seat_pessimistic(seat_id, user_id):
    with db.transaction():
        # Блокируем строку на время транзакции
        seat = db.query(
            "SELECT * FROM seats WHERE id = %s FOR UPDATE",
            seat_id
        )

        if seat.is_booked:
            raise SeatAlreadyBookedError()

        db.execute(
            "UPDATE seats SET is_booked = true, user_id = %s WHERE id = %s",
            user_id, seat_id
        )

# Решение 2: Optimistic Locking с версионированием
def book_seat_optimistic(seat_id, user_id, expected_version):
    result = db.execute("""
        UPDATE seats
        SET is_booked = true, user_id = %s, version = version + 1
        WHERE id = %s AND version = %s AND is_booked = false
    """, user_id, seat_id, expected_version)

    if result.rowcount == 0:
        raise ConcurrencyError("Seat was modified, please retry")

# Решение 3: Unique constraint
# CREATE UNIQUE INDEX idx_booking ON bookings(seat_id, event_date)
# WHERE cancelled = false;

def book_seat_constraint(seat_id, user_id, event_date):
    try:
        db.execute("""
            INSERT INTO bookings (seat_id, user_id, event_date)
            VALUES (%s, %s, %s)
        """, seat_id, user_id, event_date)
    except UniqueViolation:
        raise SeatAlreadyBookedError()
```

### Пример 2: Реализация Rate Limiting с Redis

```python
import redis
import time

class RateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client

    # Token Bucket алгоритм
    def check_rate_limit(self, user_id, max_requests=100, window_seconds=60):
        key = f"rate_limit:{user_id}"
        current_time = int(time.time())
        window_start = current_time - window_seconds

        pipe = self.redis.pipeline()

        # Удаляем старые записи
        pipe.zremrangebyscore(key, 0, window_start)

        # Считаем текущие запросы
        pipe.zcard(key)

        # Добавляем новый запрос
        pipe.zadd(key, {f"{current_time}:{uuid4()}": current_time})

        # Устанавливаем TTL
        pipe.expire(key, window_seconds)

        results = pipe.execute()
        request_count = results[1]

        if request_count >= max_requests:
            return False, 0  # Превышен лимит

        return True, max_requests - request_count - 1  # Осталось запросов

    # Sliding Window Counter для более точного подсчёта
    def sliding_window_counter(self, user_id, max_requests=100, window_seconds=60):
        current_window = int(time.time() // window_seconds)
        previous_window = current_window - 1

        current_key = f"rate:{user_id}:{current_window}"
        previous_key = f"rate:{user_id}:{previous_window}"

        pipe = self.redis.pipeline()
        pipe.get(current_key)
        pipe.get(previous_key)
        results = pipe.execute()

        current_count = int(results[0] or 0)
        previous_count = int(results[1] or 0)

        # Взвешенный подсчёт
        elapsed = time.time() % window_seconds
        weighted_count = previous_count * (1 - elapsed / window_seconds) + current_count

        if weighted_count >= max_requests:
            return False

        # Инкремент текущего окна
        pipe = self.redis.pipeline()
        pipe.incr(current_key)
        pipe.expire(current_key, window_seconds * 2)
        pipe.execute()

        return True
```

### Пример 3: Распределённый кэш с cache-aside pattern

```python
import json
import hashlib
from functools import wraps

class CacheAside:
    def __init__(self, redis_client, db):
        self.cache = redis_client
        self.db = db

    def get_user(self, user_id):
        cache_key = f"user:{user_id}"

        # 1. Попытка получить из кэша
        cached = self.cache.get(cache_key)
        if cached:
            return json.loads(cached)

        # 2. Cache miss - идём в БД
        user = self.db.query("SELECT * FROM users WHERE id = %s", user_id)

        if user:
            # 3. Сохраняем в кэш
            self.cache.setex(
                cache_key,
                3600,  # TTL 1 час
                json.dumps(user)
            )

        return user

    def update_user(self, user_id, data):
        # 1. Обновляем в БД
        self.db.execute(
            "UPDATE users SET name = %s WHERE id = %s",
            data['name'], user_id
        )

        # 2. Инвалидируем кэш
        self.cache.delete(f"user:{user_id}")

        # Альтернатива: Write-through (обновляем кэш сразу)
        # self.cache.setex(f"user:{user_id}", 3600, json.dumps(data))

# Декоратор для автоматического кэширования
def cached(ttl_seconds=3600, key_prefix="cache"):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Генерируем ключ из аргументов
            cache_key = f"{key_prefix}:{func.__name__}:" + hashlib.md5(
                json.dumps({"args": args, "kwargs": kwargs}).encode()
            ).hexdigest()

            cached_result = redis_client.get(cache_key)
            if cached_result:
                return json.loads(cached_result)

            result = func(*args, **kwargs)
            redis_client.setex(cache_key, ttl_seconds, json.dumps(result))

            return result
        return wrapper
    return decorator

@cached(ttl_seconds=300)
def get_product_recommendations(user_id, limit=10):
    # Дорогой запрос к ML модели или сложный SQL
    return expensive_recommendation_query(user_id, limit)
```

### Пример 4: Шардирование данных

```python
import hashlib

class ShardedDatabase:
    def __init__(self, shard_connections):
        self.shards = shard_connections  # {0: conn0, 1: conn1, 2: conn2}
        self.num_shards = len(shard_connections)

    def _get_shard(self, shard_key):
        """Consistent hashing для определения шарда"""
        hash_value = int(hashlib.md5(str(shard_key).encode()).hexdigest(), 16)
        return hash_value % self.num_shards

    def get_user(self, user_id):
        shard_num = self._get_shard(user_id)
        conn = self.shards[shard_num]
        return conn.query("SELECT * FROM users WHERE id = %s", user_id)

    def create_user(self, user_data):
        # Используем UUID или snowflake ID для шардирования
        user_id = generate_distributed_id()
        shard_num = self._get_shard(user_id)
        conn = self.shards[shard_num]
        conn.execute(
            "INSERT INTO users (id, name, email) VALUES (%s, %s, %s)",
            user_id, user_data['name'], user_data['email']
        )
        return user_id

    def get_all_users(self, filter_criteria):
        """Fan-out запрос ко всем шардам"""
        results = []
        for shard_num, conn in self.shards.items():
            shard_results = conn.query(
                "SELECT * FROM users WHERE %s",
                filter_criteria
            )
            results.extend(shard_results)
        return results

# Виртуальные шарды для упрощения миграции
class VirtualShards:
    def __init__(self, physical_shards, virtual_shards_per_physical=100):
        self.physical_shards = physical_shards
        self.virtual_shards = {}

        # Распределяем виртуальные шарды по физическим
        for i in range(len(physical_shards) * virtual_shards_per_physical):
            physical_idx = i % len(physical_shards)
            self.virtual_shards[i] = physical_shards[physical_idx]

    def _get_virtual_shard(self, key):
        hash_value = int(hashlib.md5(str(key).encode()).hexdigest(), 16)
        return hash_value % len(self.virtual_shards)

    def get_shard(self, key):
        virtual_shard = self._get_virtual_shard(key)
        return self.virtual_shards[virtual_shard]
```

---

## Заключение

Выбор базы данных в системном дизайне — это всегда компромисс между различными требованиями:

| Аспект | Компромисс |
|--------|------------|
| Консистентность vs Доступность | CAP теорема |
| Гибкость схемы vs Целостность данных | SQL vs NoSQL |
| Простота vs Масштабируемость | Монолит vs распределённая система |
| Производительность записи vs чтения | Нормализация vs денормализация |
| Стоимость vs Надёжность | Single node vs кластер |

### Ключевые принципы

1. **Начинайте просто** — PostgreSQL покрывает 90% задач
2. **Добавляйте специализированные БД по мере необходимости**
3. **Используйте кэширование для оптимизации чтения**
4. **Планируйте шардирование заранее, но не реализуйте преждевременно**
5. **Мониторьте и профилируйте** — решения принимайте на основе данных

---

## Дополнительные ресурсы

- [Designing Data-Intensive Applications (Martin Kleppmann)](https://dataintensive.net/)
- [Database Internals (Alex Petrov)](https://www.databass.dev/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Redis University](https://university.redis.com/)
- [MongoDB University](https://university.mongodb.com/)
