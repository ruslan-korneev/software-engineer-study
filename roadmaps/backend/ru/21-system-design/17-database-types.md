# Database Types

[prev: 16-databases](./16-databases.md) | [next: 18-sharding-and-replication](./18-sharding-and-replication.md)

---

## Введение

Выбор типа базы данных — одно из ключевых архитектурных решений. Не существует универсальной БД, которая идеально подходит для всех задач. Каждый тип оптимизирован под определённые паттерны доступа к данным, модели консистентности и требования к масштабированию.

Современные системы часто используют **polyglot persistence** — комбинацию разных типов БД для разных задач в рамках одного приложения.

---

## 1. Реляционные базы данных (RDBMS)

### Описание

Реляционные базы данных хранят данные в **таблицах** со строгой схемой. Связи между таблицами устанавливаются через **внешние ключи**. Запросы выполняются на языке **SQL**.

### Ключевые характеристики

- **ACID-гарантии**: Atomicity, Consistency, Isolation, Durability
- **Строгая схема**: Структура данных определяется заранее
- **Нормализация**: Минимизация дублирования данных
- **Транзакции**: Поддержка сложных многошаговых операций
- **Вертикальное масштабирование**: Традиционно масштабируются вверх

### Популярные решения

#### PostgreSQL
```sql
-- Мощный open-source RDBMS
-- Поддержка JSON, массивов, полнотекстового поиска

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    profile JSONB,  -- Гибкое хранение JSON
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    total DECIMAL(10,2),
    status VARCHAR(50) DEFAULT 'pending'
);

-- Оконные функции
SELECT
    user_id,
    total,
    SUM(total) OVER (PARTITION BY user_id ORDER BY created_at) as running_total
FROM orders;
```

**Особенности PostgreSQL:**
- Расширяемость (PostGIS, TimescaleDB, pg_vector)
- Продвинутые типы данных (JSONB, arrays, hstore)
- Мощный планировщик запросов
- Партиционирование таблиц
- Logical replication

#### MySQL
```sql
-- Популярный выбор для веб-приложений
-- Разные движки хранения (InnoDB, MyISAM)

CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2),
    INDEX idx_name (name)
) ENGINE=InnoDB;

-- JSON поддержка (с версии 5.7)
SELECT JSON_EXTRACT(metadata, '$.color') as color
FROM products;
```

**Особенности MySQL:**
- Простота использования
- Широкая поддержка хостингов
- Разные storage engines
- MySQL Cluster для масштабирования

#### Oracle Database
- Корпоративный уровень
- RAC (Real Application Clusters)
- Advanced Security
- Очень дорогой, но мощный

#### SQL Server
- Интеграция с Microsoft stack
- Always On Availability Groups
- In-memory OLTP
- Хорош для .NET приложений

### Когда использовать RDBMS

**Идеально для:**
- Финансовых систем (ACID критичен)
- ERP/CRM систем
- E-commerce (транзакции, отчёты)
- Систем с чётко определённой схемой
- Complex queries с JOIN-ами
- Приложений с требованием консистентности

**Не лучший выбор для:**
- Неструктурированных данных
- Очень высокой нагрузки на запись
- Горизонтального масштабирования на тысячи нод
- Данных с часто меняющейся схемой

### Плюсы и минусы

| Плюсы | Минусы |
|-------|--------|
| ACID-гарантии | Сложность горизонтального масштабирования |
| Богатый SQL | Schema migrations могут быть болезненными |
| Зрелые инструменты | Не подходит для всех типов данных |
| Хорошо для сложных запросов | Вертикальное масштабирование имеет лимиты |
| Целостность данных | Overhead для простых use cases |

---

## 2. Document Databases

### Описание

Document databases хранят данные в виде **документов** (обычно JSON или BSON). Каждый документ — самодостаточная единица с произвольной структурой.

### Ключевые характеристики

- **Гибкая схема**: Каждый документ может иметь свою структуру
- **Вложенные данные**: Поддержка иерархических структур
- **Денормализация**: Данные часто дублируются для производительности
- **Горизонтальное масштабирование**: Нативная поддержка шардинга

### Популярные решения

#### MongoDB

```javascript
// Создание документа
db.users.insertOne({
    _id: ObjectId("507f1f77bcf86cd799439011"),
    name: "Иван Петров",
    email: "ivan@example.com",
    address: {
        city: "Москва",
        street: "Тверская",
        building: 15
    },
    orders: [
        { product: "iPhone", price: 999, date: ISODate("2024-01-15") },
        { product: "MacBook", price: 1999, date: ISODate("2024-02-20") }
    ],
    tags: ["premium", "active"],
    metadata: {
        lastLogin: ISODate("2024-03-01"),
        preferences: { theme: "dark", language: "ru" }
    }
});

// Запросы
// Поиск с фильтрацией
db.users.find({
    "address.city": "Москва",
    "orders.price": { $gt: 500 }
});

// Агрегация
db.orders.aggregate([
    { $match: { status: "completed" } },
    { $group: {
        _id: "$userId",
        totalSpent: { $sum: "$amount" },
        orderCount: { $count: {} }
    }},
    { $sort: { totalSpent: -1 } },
    { $limit: 10 }
]);

// Индексы
db.users.createIndex({ email: 1 }, { unique: true });
db.users.createIndex({ "address.city": 1, createdAt: -1 });
db.users.createIndex({ tags: 1 });  // Multikey index для массивов
```

**Особенности MongoDB:**
- Replica Sets для высокой доступности
- Sharding для горизонтального масштабирования
- Aggregation Pipeline для сложных запросов
- Change Streams для реактивных приложений
- Atlas — managed cloud solution

#### CouchDB

```javascript
// CouchDB использует HTTP API и MapReduce

// Документ
{
    "_id": "user:ivan",
    "_rev": "1-abc123",
    "type": "user",
    "name": "Иван",
    "email": "ivan@example.com"
}

// Map-Reduce view
// design/users/_view/by_email
function(doc) {
    if (doc.type === "user") {
        emit(doc.email, doc.name);
    }
}
```

**Особенности CouchDB:**
- Multi-master replication
- Offline-first подход
- HTTP REST API
- Хорош для мобильных приложений (PouchDB)

### Модель данных

```
Реляционная модель:          Document модель:
┌─────────────┐              ┌─────────────────────────────┐
│   users     │              │ {                           │
├─────────────┤              │   "_id": "user:1",          │
│ id: 1       │              │   "name": "Иван",           │
│ name: Иван  │              │   "email": "ivan@ex.com",   │
└─────────────┘              │   "address": {              │
       │                     │     "city": "Москва",       │
       ▼                     │     "street": "Тверская"    │
┌─────────────┐              │   },                        │
│  addresses  │              │   "orders": [               │
├─────────────┤              │     { "product": "iPhone",  │
│ user_id: 1  │      VS      │       "price": 999 },       │
│ city: Москва│              │     { "product": "MacBook", │
│ street: ... │              │       "price": 1999 }       │
└─────────────┘              │   ]                         │
       │                     │ }                           │
       ▼                     └─────────────────────────────┘
┌─────────────┐
│   orders    │              Один документ содержит
├─────────────┤              всю связанную информацию
│ user_id: 1  │
│ product:... │
└─────────────┘
```

### Use Cases

**Идеально для:**
- Content Management Systems (CMS)
- Каталогов продуктов (разные атрибуты)
- Real-time analytics
- Mobile/IoT приложений
- Прототипирования (гибкая схема)
- User profiles с разной структурой

**Паттерны использования:**
```javascript
// Пример: Каталог продуктов с разными атрибутами
// Телефон
{
    "_id": "prod:iphone",
    "type": "phone",
    "name": "iPhone 15",
    "price": 999,
    "specs": {
        "screen": "6.1 inch",
        "camera": "48MP",
        "storage": ["128GB", "256GB", "512GB"]
    }
}

// Ноутбук - другая структура specs
{
    "_id": "prod:macbook",
    "type": "laptop",
    "name": "MacBook Pro",
    "price": 1999,
    "specs": {
        "screen": "14 inch",
        "cpu": "M3 Pro",
        "ram": "18GB",
        "ports": ["USB-C", "HDMI", "MagSafe"]
    }
}
```

### Плюсы и минусы

| Плюсы | Минусы |
|-------|--------|
| Гибкая схема | Нет JOIN-ов (нужна денормализация) |
| Естественное представление объектов | Дублирование данных |
| Горизонтальное масштабирование | Сложнее обеспечить консистентность |
| Быстрые операции чтения | Сложные транзакции между документами |
| Отлично для JSON API | Нужна дисциплина в моделировании |

---

## 3. Key-Value Stores

### Описание

Простейшая модель: каждое значение ассоциировано с уникальным ключом. Это как гигантская хеш-таблица. Значение — это просто blob (строка, число, бинарные данные).

### Ключевые характеристики

- **Простая модель**: GET/SET/DELETE
- **Сверхбыстрые операции**: O(1) доступ
- **In-memory**: Часто хранят данные в памяти
- **Горизонтальное масштабирование**: Легко партиционировать по ключу

### Популярные решения

#### Redis

```redis
# Базовые операции
SET user:1:name "Иван"
GET user:1:name
DEL user:1:name

# TTL (Time To Live)
SET session:abc123 "user_data" EX 3600  # Истекает через час
TTL session:abc123

# Структуры данных
# Строки
SET counter 0
INCR counter
INCRBY counter 10

# Хеши (объекты)
HSET user:1 name "Иван" email "ivan@ex.com" age 30
HGET user:1 name
HGETALL user:1

# Списки (очереди)
LPUSH queue:tasks "task1"
RPUSH queue:tasks "task2"
LPOP queue:tasks
LRANGE queue:tasks 0 -1

# Множества
SADD user:1:tags "premium" "active" "moscow"
SMEMBERS user:1:tags
SINTER user:1:tags user:2:tags  # Пересечение

# Sorted Sets (leaderboards)
ZADD leaderboard 1000 "player1" 1500 "player2" 800 "player3"
ZRANGE leaderboard 0 -1 WITHSCORES
ZRANK leaderboard "player1"

# Pub/Sub
SUBSCRIBE channel:notifications
PUBLISH channel:notifications "New message!"

# Streams (event log)
XADD events * type "order" user_id "123" amount "99.99"
XREAD STREAMS events 0
```

**Особенности Redis:**
- Богатые структуры данных
- Lua scripting
- Redis Cluster для масштабирования
- Persistence (RDB, AOF)
- Pub/Sub
- Redis Streams

**Паттерны использования Redis:**
```python
import redis

r = redis.Redis(host='localhost', port=6379)

# Кеширование
def get_user(user_id):
    cache_key = f"user:{user_id}"
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    r.setex(cache_key, 3600, json.dumps(user))  # Cache на 1 час
    return user

# Rate limiting
def is_rate_limited(user_id, limit=100, window=60):
    key = f"rate:{user_id}"
    current = r.incr(key)
    if current == 1:
        r.expire(key, window)
    return current > limit

# Distributed lock
def acquire_lock(lock_name, timeout=10):
    lock_key = f"lock:{lock_name}"
    token = str(uuid.uuid4())
    if r.set(lock_key, token, ex=timeout, nx=True):
        return token
    return None

# Session storage
def save_session(session_id, data, ttl=86400):
    r.hset(f"session:{session_id}", mapping=data)
    r.expire(f"session:{session_id}", ttl)
```

#### Amazon DynamoDB

```python
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

# Put item
table.put_item(Item={
    'user_id': 'user123',
    'sort_key': 'profile',
    'name': 'Иван',
    'email': 'ivan@example.com',
    'created_at': '2024-01-15'
})

# Get item
response = table.get_item(Key={
    'user_id': 'user123',
    'sort_key': 'profile'
})

# Query (по partition key)
response = table.query(
    KeyConditionExpression=Key('user_id').eq('user123')
)

# Scan с фильтрацией (дорогая операция)
response = table.scan(
    FilterExpression=Attr('age').gt(25)
)
```

**Особенности DynamoDB:**
- Fully managed
- Single-digit millisecond latency
- Auto-scaling
- Global tables
- DAX (in-memory cache)
- Partition key + Sort key модель

#### Memcached

```python
import memcache

mc = memcache.Client(['localhost:11211'])

# Простые операции
mc.set('key', 'value', time=3600)
value = mc.get('key')
mc.delete('key')

# Атомарные операции
mc.incr('counter')
mc.decr('counter')

# Multi-get
values = mc.get_multi(['key1', 'key2', 'key3'])
```

**Особенности Memcached:**
- Простота
- Multi-threaded
- Только in-memory
- Нет persistence
- LRU eviction

### Когда использовать

**Идеально для:**
- Кеширования
- Сессий пользователей
- Rate limiting
- Real-time leaderboards
- Pub/Sub messaging
- Distributed locks
- Счётчиков

**Сравнение:**

| Критерий | Redis | Memcached | DynamoDB |
|----------|-------|-----------|----------|
| Структуры данных | Богатые | Только strings | Documents |
| Persistence | Да | Нет | Да |
| Clustering | Redis Cluster | Client-side | Managed |
| Memory model | Single-threaded | Multi-threaded | Managed |
| Use case | Cache + DB | Pure cache | Primary DB |

---

## 4. Column-Family Databases

### Описание

Column-family (wide-column) databases организуют данные в **строки** и **column families**. Каждая строка может иметь разный набор колонок. Оптимизированы для записи и чтения больших объёмов данных.

### Ключевые характеристики

- **Широкие строки**: Миллионы колонок в одной строке
- **Column families**: Группировка связанных колонок
- **Sparse data**: Не хранит NULL значения
- **Write-optimized**: LSM-tree, append-only
- **Линейное масштабирование**: Добавление нод = рост производительности

### Модель данных

```
Традиционная таблица:          Wide-Column модель:
┌────────┬──────┬───────┐      Row Key: "user:1"
│ user_id│ name │ email │      ├── profile:name = "Иван"
├────────┼──────┼───────┤      ├── profile:email = "ivan@ex.com"
│ 1      │ Иван │ ivan@ │      ├── profile:age = 30
│ 2      │ Петр │ petr@ │      ├── orders:2024-01 = {...}
│ 3      │ NULL │ anna@ │      ├── orders:2024-02 = {...}
└────────┴──────┴───────┘      └── settings:theme = "dark"

                               Row Key: "user:2"
                               ├── profile:name = "Петр"
                               └── profile:email = "petr@ex.com"
                               (нет age, orders, settings)
```

### Популярные решения

#### Apache Cassandra

```sql
-- Keyspace (аналог database)
CREATE KEYSPACE ecommerce
WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'dc1': 3, 'dc2': 3
};

-- Таблица с partition key и clustering columns
CREATE TABLE orders (
    user_id UUID,
    order_date TIMESTAMP,
    order_id UUID,
    product_name TEXT,
    amount DECIMAL,
    PRIMARY KEY ((user_id), order_date, order_id)
) WITH CLUSTERING ORDER BY (order_date DESC);

-- Вставка
INSERT INTO orders (user_id, order_date, order_id, product_name, amount)
VALUES (uuid(), toTimestamp(now()), uuid(), 'iPhone', 999.00);

-- Запросы (должны использовать partition key)
SELECT * FROM orders WHERE user_id = ?;
SELECT * FROM orders WHERE user_id = ? AND order_date > '2024-01-01';

-- Time-series паттерн с bucketing
CREATE TABLE sensor_data (
    sensor_id TEXT,
    day TEXT,  -- partition key для bucketing
    timestamp TIMESTAMP,
    value DOUBLE,
    PRIMARY KEY ((sensor_id, day), timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

**Особенности Cassandra:**
- Masterless архитектура (все ноды равны)
- Tunable consistency (ONE, QUORUM, ALL)
- Linear scalability
- Multi-datacenter replication
- CQL (похож на SQL)

**Моделирование данных в Cassandra:**
```sql
-- Правило: Одна таблица = один query pattern
-- Query: Получить все посты пользователя
CREATE TABLE user_posts (
    user_id UUID,
    post_id TIMEUUID,
    content TEXT,
    PRIMARY KEY (user_id, post_id)
) WITH CLUSTERING ORDER BY (post_id DESC);

-- Query: Получить все посты по тегу
CREATE TABLE posts_by_tag (
    tag TEXT,
    post_id TIMEUUID,
    user_id UUID,
    content TEXT,
    PRIMARY KEY (tag, post_id)
) WITH CLUSTERING ORDER BY (post_id DESC);

-- Materialized View
CREATE MATERIALIZED VIEW posts_by_user AS
    SELECT * FROM posts
    WHERE user_id IS NOT NULL AND post_id IS NOT NULL
    PRIMARY KEY (user_id, post_id);
```

#### Apache HBase

```java
// HBase Java API
Configuration config = HBaseConfiguration.create();
Connection connection = ConnectionFactory.createConnection(config);
Table table = connection.getTable(TableName.valueOf("users"));

// Put
Put put = new Put(Bytes.toBytes("user:1"));
put.addColumn(
    Bytes.toBytes("profile"),      // Column family
    Bytes.toBytes("name"),         // Qualifier
    Bytes.toBytes("Иван")          // Value
);
table.put(put);

// Get
Get get = new Get(Bytes.toBytes("user:1"));
Result result = table.get(get);
byte[] name = result.getValue(
    Bytes.toBytes("profile"),
    Bytes.toBytes("name")
);

// Scan
Scan scan = new Scan();
scan.addColumn(Bytes.toBytes("profile"), Bytes.toBytes("name"));
scan.setStartRow(Bytes.toBytes("user:1"));
scan.setStopRow(Bytes.toBytes("user:9"));
ResultScanner scanner = table.getScanner(scan);
```

**Особенности HBase:**
- Интеграция с Hadoop ecosystem
- Strong consistency
- HDFS как storage layer
- Хорош для batch processing

#### ScyllaDB

```sql
-- API совместим с Cassandra
-- Написан на C++ (vs Java для Cassandra)

CREATE TABLE events (
    tenant_id UUID,
    event_date DATE,
    event_id TIMEUUID,
    event_type TEXT,
    payload TEXT,
    PRIMARY KEY ((tenant_id, event_date), event_id)
);

-- Scylla специфичные оптимизации
-- Workload prioritization
ALTER TABLE events WITH workload_type = 'general';
```

**Особенности ScyllaDB:**
- Совместим с Cassandra
- Значительно быстрее (C++ vs Java)
- Меньше latency jitter
- Shard-per-core архитектура

### Use Cases

**Идеально для:**
- Time-series данных
- IoT и sensor data
- Event logging и audit trails
- Messaging (история сообщений)
- Recommendation engines
- Fraud detection
- Аналитики в реальном времени

**Паттерн: Time-series с Cassandra**
```sql
-- Данные телеметрии с автоматическим TTL
CREATE TABLE metrics (
    device_id TEXT,
    bucket TEXT,  -- '2024-03-15'
    timestamp TIMESTAMP,
    cpu DOUBLE,
    memory DOUBLE,
    disk DOUBLE,
    PRIMARY KEY ((device_id, bucket), timestamp)
) WITH default_time_to_live = 604800  -- 7 дней
  AND CLUSTERING ORDER BY (timestamp DESC);

-- Запрос последних метрик устройства
SELECT * FROM metrics
WHERE device_id = 'server-1' AND bucket = '2024-03-15'
LIMIT 100;
```

### Плюсы и минусы

| Плюсы | Минусы |
|-------|--------|
| Линейное масштабирование | Нет JOIN-ов |
| Высокая производительность записи | Query-driven моделирование |
| Отказоустойчивость | Ограниченные вторичные индексы |
| Multi-DC репликация | Eventual consistency по умолчанию |
| Хорош для time-series | Сложность операций |

---

## 5. Graph Databases

### Описание

Graph databases хранят данные как **узлы** (entities) и **рёбра** (relationships). Оптимизированы для запросов, которые traverse связи между сущностями.

### Ключевые характеристики

- **Узлы и рёбра**: Естественное представление связей
- **Index-free adjacency**: Связи хранятся непосредственно
- **Graph traversal**: Эффективные запросы по связям
- **Свойства**: Узлы и рёбра могут иметь атрибуты

### Модель данных

```
Реляционная модель:               Graph модель:
┌─────────┐    ┌───────────────┐
│ users   │    │ friendships   │      (Иван)──FOLLOWS──▶(Петр)
├─────────┤    ├───────────────┤         │                  │
│ id: 1   │◀───│ user_id: 1    │      LIKES             FOLLOWS
│ name:...|    │ friend_id: 2  │         ▼                  ▼
└─────────┘    │ since: 2024   │      (Post1)◀──WROTE──(Анна)
               └───────────────┘

Для запроса "друзья друзей"       Простой traversal
нужен self-JOIN                   без JOIN-ов
```

### Популярные решения

#### Neo4j

```cypher
// Cypher Query Language

// Создание узлов
CREATE (ivan:Person {name: 'Иван', age: 30, city: 'Москва'})
CREATE (petr:Person {name: 'Петр', age: 28, city: 'СПб'})
CREATE (anna:Person {name: 'Анна', age: 25, city: 'Москва'})
CREATE (company:Company {name: 'TechCorp', industry: 'IT'})

// Создание связей
MATCH (ivan:Person {name: 'Иван'}), (petr:Person {name: 'Петр'})
CREATE (ivan)-[:FOLLOWS {since: 2023}]->(petr)

MATCH (ivan:Person {name: 'Иван'}), (company:Company {name: 'TechCorp'})
CREATE (ivan)-[:WORKS_AT {position: 'Developer', since: 2020}]->(company)

// Запросы
// Найти друзей Ивана
MATCH (ivan:Person {name: 'Иван'})-[:FOLLOWS]->(friend)
RETURN friend.name

// Друзья друзей (2 уровня)
MATCH (ivan:Person {name: 'Иван'})-[:FOLLOWS*2]->(fof)
WHERE NOT (ivan)-[:FOLLOWS]->(fof) AND ivan <> fof
RETURN DISTINCT fof.name

// Кратчайший путь между людьми
MATCH path = shortestPath(
    (ivan:Person {name: 'Иван'})-[:FOLLOWS*]-(anna:Person {name: 'Анна'})
)
RETURN path

// Рекомендации (люди с общими интересами)
MATCH (ivan:Person {name: 'Иван'})-[:LIKES]->(interest)<-[:LIKES]-(other)
WHERE ivan <> other
WITH other, COUNT(interest) as commonInterests
ORDER BY commonInterests DESC
LIMIT 10
RETURN other.name, commonInterests

// PageRank-подобный запрос
MATCH (p:Person)
OPTIONAL MATCH (p)<-[:FOLLOWS]-(follower)
WITH p, COUNT(follower) as followers
ORDER BY followers DESC
RETURN p.name, followers
```

**Особенности Neo4j:**
- Native graph storage
- Cypher query language
- ACID transactions
- Causal clustering
- Graph Data Science library

**Пример: Fraud Detection**
```cypher
// Найти подозрительные паттерны
// Аккаунты, связанные через общий телефон/email/адрес
MATCH (a1:Account)-[:HAS_PHONE]->(phone)<-[:HAS_PHONE]-(a2:Account)
WHERE a1 <> a2
WITH a1, a2, phone
MATCH (a1)-[:HAS_EMAIL]->(email)<-[:HAS_EMAIL]-(a2)
RETURN a1, a2, phone, email

// Ring of fraud (циклы переводов)
MATCH path = (a:Account)-[:TRANSFERRED*3..6]->(a)
WHERE ALL(t IN relationships(path) WHERE t.amount > 10000)
RETURN path
```

#### Amazon Neptune

```sparql
# SPARQL для RDF графов
PREFIX ex: <http://example.org/>

# Запрос друзей
SELECT ?friend ?name WHERE {
    ex:Ivan ex:knows ?friend .
    ?friend ex:name ?name .
}

# Gremlin для property graphs
g.V().has('Person', 'name', 'Ivan')
  .out('FOLLOWS')
  .values('name')
```

**Особенности Neptune:**
- Managed service
- Поддержка SPARQL и Gremlin
- Multi-AZ
- Read replicas

### Use Cases

**Идеально для:**
- Социальных сетей
- Recommendation engines
- Fraud detection
- Knowledge graphs
- Network topology
- Identity и Access Management
- Supply chain

**Пример: Recommendation Engine**
```cypher
// Рекомендации товаров на основе покупок похожих пользователей
MATCH (user:User {id: $userId})-[:PURCHASED]->(product)<-[:PURCHASED]-(other)
WITH other, COUNT(product) as commonPurchases
ORDER BY commonPurchases DESC
LIMIT 10

MATCH (other)-[:PURCHASED]->(recommendation)
WHERE NOT (user)-[:PURCHASED]->(recommendation)
RETURN recommendation, COUNT(*) as score
ORDER BY score DESC
LIMIT 5
```

### Плюсы и минусы

| Плюсы | Минусы |
|-------|--------|
| Естественное моделирование связей | Не для всех типов данных |
| Быстрые graph traversals | Сложнее масштабировать |
| Гибкая схема | Меньше инструментов и специалистов |
| Интуитивные запросы | Не подходит для простых CRUD |
| Хорош для сложных связей | Стоимость может быть высокой |

---

## 6. Time-Series Databases

### Описание

Time-series databases оптимизированы для данных с **временной меткой**: метрики, события, измерения сенсоров. Особенности: высокая скорость записи, эффективное сжатие, автоматическое удаление старых данных.

### Ключевые характеристики

- **Append-only**: Данные только добавляются
- **Time-based partitioning**: Данные разбиты по времени
- **Compression**: Специализированные алгоритмы сжатия
- **Downsampling**: Агрегация старых данных
- **Retention policies**: Автоматическое удаление

### Популярные решения

#### InfluxDB

```sql
-- InfluxQL / Flux

-- Создание базы с retention policy
CREATE DATABASE metrics
CREATE RETENTION POLICY "one_week" ON "metrics" DURATION 7d REPLICATION 1 DEFAULT

-- Запись данных (Line Protocol)
-- measurement,tag_set field_set timestamp
cpu,host=server01,region=us-east usage=80.5,temperature=65 1609459200000000000

-- InfluxQL запросы
SELECT mean("usage")
FROM "cpu"
WHERE time > now() - 1h
GROUP BY time(5m), "host"

-- Flux (современный язык)
from(bucket: "metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "cpu")
  |> filter(fn: (r) => r.host == "server01")
  |> aggregateWindow(every: 5m, fn: mean)
  |> yield()

-- Continuous Query (автоматическая агрегация)
CREATE CONTINUOUS QUERY "downsample_cpu" ON "metrics"
BEGIN
  SELECT mean("usage") AS "usage"
  INTO "metrics"."long_term"."cpu"
  FROM "cpu"
  GROUP BY time(1h), *
END
```

**Особенности InfluxDB:**
- Line Protocol для записи
- InfluxQL и Flux
- Telegraf для сбора метрик
- Built-in dashboards
- Kapacitor для alerting

#### TimescaleDB

```sql
-- Расширение PostgreSQL

-- Создание hypertable
CREATE TABLE metrics (
    time TIMESTAMPTZ NOT NULL,
    device_id TEXT NOT NULL,
    cpu DOUBLE PRECISION,
    memory DOUBLE PRECISION,
    disk DOUBLE PRECISION
);

SELECT create_hypertable('metrics', 'time');

-- Автоматическое партиционирование по времени
SELECT add_dimension('metrics', 'device_id', number_partitions => 4);

-- Запросы (обычный SQL!)
SELECT
    time_bucket('5 minutes', time) AS bucket,
    device_id,
    AVG(cpu) as avg_cpu,
    MAX(cpu) as max_cpu
FROM metrics
WHERE time > NOW() - INTERVAL '1 hour'
GROUP BY bucket, device_id
ORDER BY bucket DESC;

-- Continuous Aggregates
CREATE MATERIALIZED VIEW cpu_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    device_id,
    AVG(cpu) as avg_cpu,
    MAX(cpu) as max_cpu,
    MIN(cpu) as min_cpu
FROM metrics
GROUP BY bucket, device_id;

-- Retention Policy
SELECT add_retention_policy('metrics', INTERVAL '30 days');

-- Compression (старые данные)
ALTER TABLE metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'device_id'
);
SELECT add_compression_policy('metrics', INTERVAL '7 days');
```

**Особенности TimescaleDB:**
- Полная совместимость с PostgreSQL
- Автоматическое партиционирование
- Continuous aggregates
- Compression
- Retention policies
- Можно использовать любые PostgreSQL расширения

#### Prometheus

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'app'
    static_configs:
      - targets: ['app:8080']
```

```promql
# PromQL запросы

# Текущее значение
node_cpu_seconds_total{mode="idle"}

# Rate (изменение за период)
rate(http_requests_total[5m])

# Агрегация
sum(rate(http_requests_total[5m])) by (method)

# Histograms
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)

# Alerts
# В prometheus.yml или rules файле
groups:
  - name: example
    rules:
      - alert: HighCPU
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
```

**Особенности Prometheus:**
- Pull-based модель сбора
- Мощный PromQL
- AlertManager интеграция
- Service discovery
- Часть CNCF/Kubernetes ecosystem

### Use Cases

**Идеально для:**
- Мониторинга инфраструктуры
- APM (Application Performance Monitoring)
- IoT и sensor data
- Финансовых данных (котировки)
- Логов и событий
- Business metrics

**Пример: Мониторинг приложения**
```python
# Python приложение с Prometheus
from prometheus_client import Counter, Histogram, start_http_server
import time

# Метрики
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint'],
    buckets=[.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10]
)

def process_request(method, endpoint):
    start = time.time()
    try:
        # ... обработка запроса
        status = '200'
    except Exception:
        status = '500'
    finally:
        REQUEST_COUNT.labels(method, endpoint, status).inc()
        REQUEST_LATENCY.labels(method, endpoint).observe(time.time() - start)

# Экспорт метрик
start_http_server(8000)
```

### Плюсы и минусы

| Плюсы | Минусы |
|-------|--------|
| Оптимизированы для временных рядов | Ограничены одним типом данных |
| Высокая скорость записи | Запросы только по времени |
| Эффективное сжатие | Сложнее делать произвольные запросы |
| Автоматический retention | Каждый инструмент — свой язык |
| Downsampling | Нужна интеграция с визуализацией |

---

## 7. Search Engines

### Описание

Search engines оптимизированы для **полнотекстового поиска** и **аналитики**. Используют **inverted index** для быстрого поиска по тексту.

### Ключевые характеристики

- **Inverted index**: Маппинг слов к документам
- **Relevance scoring**: Ранжирование результатов
- **Analyzers**: Токенизация, стемминг, нормализация
- **Faceted search**: Фильтрация и агрегация
- **Near real-time**: Данные доступны для поиска быстро

### Как работает Inverted Index

```
Документы:
Doc1: "Redis is a key-value database"
Doc2: "MongoDB is a document database"
Doc3: "Redis supports pub/sub"

Inverted Index:
┌──────────┬────────────┐
│ Term     │ Documents  │
├──────────┼────────────┤
│ redis    │ [1, 3]     │
│ mongodb  │ [2]        │
│ database │ [1, 2]     │
│ key      │ [1]        │
│ value    │ [1]        │
│ document │ [2]        │
│ pub      │ [3]        │
│ sub      │ [3]        │
└──────────┴────────────┘

Поиск "redis database" → Docs: 1 (оба термина)
```

### Популярные решения

#### Elasticsearch

```json
// Создание индекса с маппингом
PUT /products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "russian_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "russian_stemmer"]
        }
      },
      "filter": {
        "russian_stemmer": {
          "type": "stemmer",
          "language": "russian"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "russian_analyzer",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "description": { "type": "text", "analyzer": "russian_analyzer" },
      "price": { "type": "float" },
      "category": { "type": "keyword" },
      "tags": { "type": "keyword" },
      "created_at": { "type": "date" },
      "location": { "type": "geo_point" }
    }
  }
}

// Индексация документа
POST /products/_doc
{
  "name": "iPhone 15 Pro",
  "description": "Новый смартфон Apple с процессором A17 Pro",
  "price": 999.99,
  "category": "electronics",
  "tags": ["phone", "apple", "premium"],
  "created_at": "2024-03-15",
  "location": { "lat": 55.75, "lon": 37.62 }
}

// Полнотекстовый поиск
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "description": "смартфон apple" } }
      ],
      "filter": [
        { "term": { "category": "electronics" } },
        { "range": { "price": { "lte": 1500 } } }
      ]
    }
  },
  "highlight": {
    "fields": { "description": {} }
  }
}

// Aggregations (аналитика)
GET /products/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category" },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } },
        "price_ranges": {
          "range": {
            "field": "price",
            "ranges": [
              { "to": 100 },
              { "from": 100, "to": 500 },
              { "from": 500 }
            ]
          }
        }
      }
    }
  }
}

// Geo-поиск
GET /products/_search
{
  "query": {
    "geo_distance": {
      "distance": "10km",
      "location": { "lat": 55.75, "lon": 37.62 }
    }
  },
  "sort": [
    {
      "_geo_distance": {
        "location": { "lat": 55.75, "lon": 37.62 },
        "order": "asc"
      }
    }
  ]
}

// Autocomplete (suggest)
GET /products/_search
{
  "suggest": {
    "product-suggest": {
      "prefix": "iph",
      "completion": {
        "field": "name.suggest"
      }
    }
  }
}
```

**Особенности Elasticsearch:**
- Distributed и scalable
- REST API
- Aggregations для аналитики
- Kibana для визуализации
- ELK stack (Elasticsearch, Logstash, Kibana)
- Vector search (с версии 8.x)

**Python пример:**
```python
from elasticsearch import Elasticsearch

es = Elasticsearch(['http://localhost:9200'])

# Индексация
es.index(index='products', document={
    'name': 'MacBook Pro',
    'description': 'Мощный ноутбук для профессионалов',
    'price': 1999.99,
    'category': 'electronics'
})

# Поиск
result = es.search(index='products', query={
    'multi_match': {
        'query': 'ноутбук профессионал',
        'fields': ['name^2', 'description']  # boost для name
    }
})

for hit in result['hits']['hits']:
    print(f"{hit['_source']['name']}: {hit['_score']}")
```

#### Apache Solr

```xml
<!-- schema.xml -->
<field name="id" type="string" indexed="true" stored="true" required="true"/>
<field name="name" type="text_ru" indexed="true" stored="true"/>
<field name="price" type="pfloat" indexed="true" stored="true"/>
```

```
# Solr запрос
http://localhost:8983/solr/products/select?
  q=смартфон&
  fq=category:electronics&
  fq=price:[0 TO 1000]&
  sort=score desc&
  fl=id,name,price&
  facet=true&
  facet.field=category
```

**Особенности Solr:**
- Часть Apache ecosystem
- XML/JSON конфигурация
- Faceted search
- SolrCloud для distributed
- Старше Elasticsearch

### Use Cases

**Идеально для:**
- E-commerce поиска
- Поиска по документам
- Логов и мониторинга (ELK)
- Аналитики в реальном времени
- Geo-поиска
- Autocomplete
- Recommendation engines

**Паттерн: Поиск + Primary DB**
```
┌─────────┐     ┌──────────────┐     ┌─────────────┐
│ Client  │────▶│   API        │────▶│ PostgreSQL  │
└─────────┘     └──────────────┘     │ (primary)   │
     │                │               └──────┬──────┘
     │                │                      │
     ▼                ▼                      ▼
┌─────────────────────────────────────────────────────┐
│              Elasticsearch                          │
│         (search & analytics)                        │
└─────────────────────────────────────────────────────┘

- Write идёт в PostgreSQL
- PostgreSQL синхронизирует в Elasticsearch (CDC или app-level)
- Search запросы идут в Elasticsearch
- Detail запросы идут в PostgreSQL
```

### Плюсы и минусы

| Плюсы | Минусы |
|-------|--------|
| Быстрый полнотекстовый поиск | Не primary database |
| Relevance scoring | Eventual consistency |
| Мощные aggregations | Ресурсоёмкий (память) |
| Geo и vector search | Сложность операций |
| Масштабируемость | Нужна синхронизация с primary DB |

---

## 8. NewSQL Databases

### Описание

NewSQL = SQL + горизонтальное масштабирование + ACID. Цель — получить преимущества реляционных БД без их ограничений в масштабировании.

### Ключевые характеристики

- **SQL интерфейс**: Совместимость с SQL
- **ACID транзакции**: Распределённые транзакции
- **Горизонтальное масштабирование**: Sharding "из коробки"
- **High availability**: Автоматический failover
- **Strong consistency**: В отличие от NoSQL

### Популярные решения

#### CockroachDB

```sql
-- Совместим с PostgreSQL!

-- Создание базы
CREATE DATABASE ecommerce;

-- Таблица с автоматическим шардингом
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    product_id UUID NOT NULL,
    amount DECIMAL(10,2),
    status VARCHAR(50) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT now(),

    INDEX idx_user (user_id),
    INDEX idx_status_created (status, created_at DESC)
);

-- Распределённая транзакция (работает автоматически)
BEGIN;
    INSERT INTO orders (user_id, product_id, amount)
    VALUES ('user-1', 'product-1', 99.99);

    UPDATE inventory SET quantity = quantity - 1
    WHERE product_id = 'product-1';

    UPDATE user_balance SET balance = balance - 99.99
    WHERE user_id = 'user-1';
COMMIT;

-- Geo-partitioning (данные ближе к пользователям)
ALTER TABLE users CONFIGURE ZONE USING
    constraints = '{"+region=us-east": 1}';

-- Multi-region setup
ALTER DATABASE ecommerce SET PRIMARY REGION "us-east1";
ALTER DATABASE ecommerce ADD REGION "eu-west1";
ALTER DATABASE ecommerce ADD REGION "ap-northeast1";

-- Таблица с regional-by-row
CREATE TABLE user_data (
    id UUID PRIMARY KEY,
    user_id UUID,
    region crdb_internal_region NOT NULL,
    data JSONB
) LOCALITY REGIONAL BY ROW AS region;
```

**Особенности CockroachDB:**
- PostgreSQL wire protocol
- Automatic sharding и rebalancing
- Serializable isolation (strictest)
- Multi-region с geo-partitioning
- Kubernetes-native

#### TiDB

```sql
-- Совместим с MySQL!

-- Создание таблицы
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_RANDOM,
    user_id BIGINT NOT NULL,
    amount DECIMAL(10,2),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    KEY idx_user (user_id)
);

-- Партиционирование (Range, Hash, List)
CREATE TABLE logs (
    id BIGINT,
    log_time DATETIME,
    message TEXT,
    PRIMARY KEY (id, log_time)
) PARTITION BY RANGE (YEAR(log_time)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);

-- TiFlash для OLAP (columnar storage)
ALTER TABLE orders SET TIFLASH REPLICA 1;

-- HTAP запрос (OLTP + OLAP)
SELECT
    DATE(created_at) as date,
    SUM(amount) as daily_total
FROM orders
WHERE created_at > '2024-01-01'
GROUP BY DATE(created_at);
```

**Особенности TiDB:**
- MySQL совместимость
- HTAP (OLTP + OLAP)
- TiKV (распределённый key-value)
- TiFlash (columnar для аналитики)
- Horizontal scaling

#### Google Spanner

```sql
-- Google Cloud Spanner

-- Создание таблицы с interleaved data
CREATE TABLE Users (
    UserId INT64 NOT NULL,
    Name STRING(100),
    Email STRING(255)
) PRIMARY KEY (UserId);

CREATE TABLE Orders (
    UserId INT64 NOT NULL,
    OrderId INT64 NOT NULL,
    Amount NUMERIC,
    CreatedAt TIMESTAMP
) PRIMARY KEY (UserId, OrderId),
  INTERLEAVE IN PARENT Users ON DELETE CASCADE;

-- Stale reads (для производительности)
SELECT * FROM Users
WHERE UserId = 123
STALE AS OF TIMESTAMP_BOUND MAX_STALENESS 10s;

-- Multi-region configuration через UI/API
-- Данные автоматически реплицируются между регионами
```

**Особенности Spanner:**
- Global distribution
- External consistency (TrueTime)
- Fully managed
- Очень дорогой
- Multi-region by design

### Сравнение архитектур

```
Traditional RDBMS:          NewSQL:
┌─────────────────┐        ┌─────┬─────┬─────┐
│   Single Node   │        │Node1│Node2│Node3│
│   (scale up)    │        │     │     │     │
└─────────────────┘        └──┬──┴──┬──┴──┬──┘
         │                    │     │     │
         ▼                    ▼     ▼     ▼
    ┌─────────┐           ┌─────────────────┐
    │  Disk   │           │ Distributed     │
    └─────────┘           │ Storage Layer   │
                          └─────────────────┘

Single point of           Distributed, fault-
failure, limited          tolerant, scales
scalability               horizontally
```

### Use Cases

**Идеально для:**
- Финансовых систем требующих масштабирование
- Global приложений (multi-region)
- Миграции с RDBMS без потери функций
- Систем требующих ACID + scale
- E-commerce с высокой нагрузкой

### Плюсы и минусы

| Плюсы | Минусы |
|-------|--------|
| SQL совместимость | Сложнее в эксплуатации |
| ACID + масштабирование | Latency выше чем у локального RDBMS |
| Automatic sharding | Меньше экосистема |
| High availability | Стоимость (особенно Spanner) |
| Strong consistency | Нужна экспертиза |

---

## 9. Сравнительная таблица типов БД

### По модели данных

| Тип | Модель | Примеры | Лучше всего для |
|-----|--------|---------|-----------------|
| RDBMS | Таблицы, связи | PostgreSQL, MySQL | Транзакции, отчёты |
| Document | JSON документы | MongoDB, CouchDB | Гибкие схемы, CMS |
| Key-Value | Ключ → Значение | Redis, DynamoDB | Кеш, сессии |
| Column-Family | Широкие строки | Cassandra, HBase | Time-series, IoT |
| Graph | Узлы и связи | Neo4j, Neptune | Социальные сети |
| Time-Series | Временные ряды | InfluxDB, TimescaleDB | Метрики, мониторинг |
| Search | Inverted index | Elasticsearch | Полнотекстовый поиск |
| NewSQL | Таблицы (distributed) | CockroachDB, TiDB | Scale + ACID |

### По характеристикам

| Тип | Consistency | Scalability | Schema | Transactions |
|-----|-------------|-------------|--------|--------------|
| RDBMS | Strong | Vertical | Fixed | ACID |
| Document | Configurable | Horizontal | Flexible | Limited |
| Key-Value | Configurable | Horizontal | None | Limited |
| Column-Family | Eventual | Horizontal | Flexible | Limited |
| Graph | Strong | Vertical | Flexible | ACID |
| Time-Series | Strong | Horizontal | Fixed | Limited |
| Search | Eventual | Horizontal | Flexible | None |
| NewSQL | Strong | Horizontal | Fixed | ACID |

### По сценариям использования

```
                    Consistency
                         ▲
                         │
              RDBMS  ●   │   ● NewSQL
                         │
          Graph ●        │        ● Spanner
                         │
        ──────────────────┼────────────────▶ Scalability
                         │
    Time-Series ●        │        ● Cassandra
                         │
         Search ●        │   ● DynamoDB
                         │
          Document ●     │        ● MongoDB
                         │
```

---

## 10. Как выбрать тип БД

### Decision Tree

```
Какой тип данных?
├── Структурированные, чёткие связи
│   ├── Нужно масштабирование?
│   │   ├── Да → NewSQL (CockroachDB, TiDB)
│   │   └── Нет → RDBMS (PostgreSQL, MySQL)
│   └──
├── Полуструктурированные (JSON-like)
│   ├── Нужен полнотекстовый поиск?
│   │   ├── Да → Elasticsearch + Primary DB
│   │   └── Нет → Document DB (MongoDB)
│   └──
├── Временные ряды (метрики, события)
│   └── Time-Series DB (TimescaleDB, InfluxDB)
├── Сложные связи между сущностями
│   └── Graph DB (Neo4j)
├── Простые key-value операции
│   ├── Нужна персистентность?
│   │   ├── Да → Redis с AOF / DynamoDB
│   │   └── Нет → Memcached / Redis
│   └──
└── Большие объёмы с высокой записью
    └── Column-Family (Cassandra, ScyllaDB)
```

### Вопросы для выбора

1. **Какая модель данных?**
   - Таблицы с отношениями → RDBMS/NewSQL
   - Иерархические документы → Document DB
   - Простые пары ключ-значение → Key-Value
   - Связи важнее сущностей → Graph DB

2. **Какие требования к консистентности?**
   - Строгая (банк, финансы) → RDBMS, NewSQL
   - Eventual OK → NoSQL варианты

3. **Какая нагрузка?**
   - Read-heavy → Кеширование, реплики
   - Write-heavy → Column-family, append-only
   - Mixed → Зависит от паттернов

4. **Нужны ли транзакции?**
   - Сложные multi-step → RDBMS
   - Простые single-document → Document DB
   - Распределённые ACID → NewSQL

5. **Какое масштабирование?**
   - Вертикальное (до определённого предела) → RDBMS
   - Горизонтальное → NoSQL, NewSQL

6. **Какой бюджет и экспертиза?**
   - Managed services проще
   - Self-hosted дешевле но сложнее

### Примеры выбора для реальных систем

#### E-commerce Platform
```
┌─────────────────────────────────────────────────────────┐
│                    E-commerce System                     │
├─────────────────────────────────────────────────────────┤
│ Данные         │ Хранилище     │ Причина               │
├────────────────┼───────────────┼───────────────────────┤
│ Users, Orders  │ PostgreSQL    │ ACID, транзакции      │
│ Product Search │ Elasticsearch │ Полнотекстовый поиск  │
│ Shopping Cart  │ Redis         │ Быстрый доступ        │
│ Sessions       │ Redis         │ TTL, скорость         │
│ Analytics      │ ClickHouse    │ OLAP запросы          │
│ Recommendations│ Neo4j         │ Граф покупок          │
└─────────────────────────────────────────────────────────┘
```

#### Social Network
```
┌─────────────────────────────────────────────────────────┐
│                    Social Network                        │
├─────────────────────────────────────────────────────────┤
│ Данные         │ Хранилище     │ Причина               │
├────────────────┼───────────────┼───────────────────────┤
│ User Profiles  │ PostgreSQL    │ Core data, ACID       │
│ Posts/Feed     │ Cassandra     │ Write-heavy, scale    │
│ Friendships    │ Neo4j         │ Graph queries         │
│ Messages       │ Cassandra     │ Time-series pattern   │
│ Search         │ Elasticsearch │ User/post search      │
│ Cache          │ Redis         │ Feed caching          │
│ Media metadata │ MongoDB       │ Flexible schema       │
└─────────────────────────────────────────────────────────┘
```

#### IoT Platform
```
┌─────────────────────────────────────────────────────────┐
│                    IoT Platform                          │
├─────────────────────────────────────────────────────────┤
│ Данные         │ Хранилище     │ Причина               │
├────────────────┼───────────────┼───────────────────────┤
│ Device Registry│ PostgreSQL    │ ACID, relationships   │
│ Telemetry      │ TimescaleDB   │ Time-series optimize  │
│ Real-time      │ Redis Streams │ Pub/sub, low latency  │
│ Alerts         │ InfluxDB      │ Time-based queries    │
│ Device State   │ Redis         │ Fast read/write       │
│ Long-term      │ S3 + Parquet  │ Cold storage          │
└─────────────────────────────────────────────────────────┘
```

### Anti-patterns

**Не делай так:**

1. **Один размер для всех**
   - Использовать только PostgreSQL для всего (включая кеш)
   - Использовать только MongoDB для всего (включая финансовые транзакции)

2. **Преждевременная оптимизация**
   - Начинать с Cassandra когда достаточно PostgreSQL
   - Добавлять 5 разных БД в MVP

3. **Игнорирование операционной сложности**
   - Каждая БД = экспертиза, мониторинг, бекапы
   - Polyglot persistence имеет цену

4. **Неправильная модель данных**
   - Хранить графы в реляционной БД с рекурсивными JOIN
   - Хранить табличные данные в document DB и эмулировать JOIN

### Best Practices

1. **Начинай просто**
   - PostgreSQL покрывает 80% use cases
   - Добавляй специализированные БД когда есть конкретная проблема

2. **Polyglot Persistence**
   - Разные данные → разные хранилища
   - Но каждая БД — это operational cost

3. **Кеширование**
   - Redis/Memcached перед любой БД
   - Уменьшает нагрузку, улучшает latency

4. **Учитывай consistency requirements**
   - Финансы → Strong consistency
   - Social feed → Eventual OK

5. **Планируй рост**
   - Выбирай БД которая масштабируется с твоим ростом
   - Migration — болезненный процесс

---

## Заключение

Выбор базы данных — это trade-off между:
- **Consistency vs Availability**
- **Scalability vs Simplicity**
- **Performance vs Flexibility**
- **Features vs Operational Cost**

Не существует универсального решения. Современные системы часто комбинируют несколько типов БД для разных задач (polyglot persistence). Ключ — понимать характеристики каждого типа и матчить их с требованиями конкретной задачи.

**Помни:**
- PostgreSQL — отличная отправная точка для большинства задач
- Добавляй специализированные БД для конкретных проблем
- Каждая дополнительная БД — это operational overhead
- CAP theorem никуда не делась: выбирай свой trade-off осознанно
