# PostgreSQL vs NoSQL

## Что такое NoSQL?

**NoSQL** (Not Only SQL) — общий термин для баз данных, которые не используют традиционную реляционную модель. Они оптимизированы для специфических сценариев использования.

### Типы NoSQL баз данных

1. **Документные** — MongoDB, CouchDB
2. **Ключ-значение** — Redis, DynamoDB, etcd
3. **Колоночные** — Cassandra, HBase, ClickHouse
4. **Графовые** — Neo4j, ArangoDB, Amazon Neptune
5. **Временные ряды** — InfluxDB, TimescaleDB

## PostgreSQL vs MongoDB (документные БД)

### Модель данных

**MongoDB** — документно-ориентированная:

```javascript
// MongoDB: документ в коллекции users
{
    "_id": ObjectId("507f1f77bcf86cd799439011"),
    "name": "John Doe",
    "email": "john@example.com",
    "address": {
        "street": "123 Main St",
        "city": "Boston",
        "zip": "02101"
    },
    "orders": [
        { "product": "Laptop", "price": 1200 },
        { "product": "Mouse", "price": 25 }
    ]
}
```

**PostgreSQL** — реляционная + JSONB:

```sql
-- PostgreSQL: традиционный подход
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255)
);

CREATE TABLE addresses (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    street VARCHAR(255),
    city VARCHAR(100),
    zip VARCHAR(20)
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    product VARCHAR(100),
    price DECIMAL(10, 2)
);

-- PostgreSQL: с использованием JSONB (гибридный подход)
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255),
    profile JSONB  -- гибкие данные
);

INSERT INTO users (name, email, profile) VALUES (
    'John Doe',
    'john@example.com',
    '{
        "address": {"street": "123 Main St", "city": "Boston"},
        "preferences": {"theme": "dark", "notifications": true}
    }'
);

-- Запрос к JSONB
SELECT * FROM users
WHERE profile->'address'->>'city' = 'Boston';

-- Индекс на JSONB
CREATE INDEX idx_users_profile ON users USING GIN(profile);
```

### Сравнение возможностей

| Аспект | PostgreSQL | MongoDB |
|--------|------------|---------|
| **Схема** | Строгая + гибкая (JSONB) | Гибкая (schemaless) |
| **Транзакции** | Полный ACID | ACID с 4.0 (ограниченно) |
| **Joins** | Да, эффективные | $lookup (менее эффективно) |
| **Индексы** | B-tree, GIN, GiST, и др. | B-tree, hash, text |
| **Репликация** | Streaming, logical | Replica sets |
| **Шардирование** | Через расширения | Встроенное |
| **Агрегации** | SQL, мощные | Aggregation pipeline |

### Когда что выбрать

**PostgreSQL лучше для:**
- Сложных связей между данными
- Транзакционных систем
- Аналитики и отчётов
- Когда нужна гибкость JSONB + строгость SQL

**MongoDB лучше для:**
- Прототипирования с часто меняющейся схемой
- Хранения документов/контента
- Приложений с иерархическими данными
- Когда горизонтальное масштабирование критично

## PostgreSQL vs Redis (ключ-значение)

### Модель данных

**Redis** — in-memory хранилище ключ-значение:

```redis
# Простые ключи
SET user:1:name "John Doe"
GET user:1:name

# Хэши
HSET user:1 name "John" email "john@example.com"
HGETALL user:1

# Списки
LPUSH notifications:user:1 "New message"
LRANGE notifications:user:1 0 -1

# Множества
SADD user:1:tags "developer" "python" "postgresql"
SMEMBERS user:1:tags

# Отсортированные множества (для рейтингов, лидербордов)
ZADD leaderboard 1000 "player1" 2000 "player2"
ZREVRANGE leaderboard 0 10 WITHSCORES

# TTL (время жизни)
SET session:abc123 "user_data" EX 3600  # истекает через час
```

**PostgreSQL** для тех же задач:

```sql
-- Хранение сессий
CREATE TABLE sessions (
    id VARCHAR(64) PRIMARY KEY,
    data JSONB,
    expires_at TIMESTAMP
);

-- Автоматическое удаление истёкших сессий
CREATE INDEX idx_sessions_expires ON sessions(expires_at);
-- Периодический DELETE WHERE expires_at < NOW()

-- Для кэширования можно использовать UNLOGGED таблицы
CREATE UNLOGGED TABLE cache (
    key VARCHAR(255) PRIMARY KEY,
    value JSONB,
    expires_at TIMESTAMP
);
```

### Сравнение

| Аспект | PostgreSQL | Redis |
|--------|------------|-------|
| **Хранение** | На диске | В памяти (+ persistence) |
| **Скорость** | Миллисекунды | Микросекунды |
| **Структуры данных** | Таблицы, JSON | Strings, Lists, Sets, Hashes, Sorted Sets, Streams |
| **Persistence** | По умолчанию | Опционально (RDB, AOF) |
| **Запросы** | SQL | Команды |
| **Транзакции** | Полный ACID | Ограниченные (MULTI/EXEC) |
| **Масштабирование** | Репликация | Cluster, Sentinel |

### Комбинированное использование

Часто PostgreSQL и Redis используются вместе:

```python
# Типичный паттерн: PostgreSQL + Redis cache
import redis
import psycopg2

redis_client = redis.Redis()
pg_conn = psycopg2.connect("postgresql://...")

def get_user(user_id):
    # Сначала проверяем кэш
    cached = redis_client.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)

    # Если нет в кэше — идём в PostgreSQL
    cursor = pg_conn.cursor()
    cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    user = cursor.fetchone()

    # Сохраняем в кэш на 5 минут
    redis_client.setex(f"user:{user_id}", 300, json.dumps(user))

    return user
```

## PostgreSQL vs Cassandra (колоночные БД)

### Модель данных

**Cassandra** — распределённая колоночная БД:

```cql
-- Cassandra: денормализованная модель
CREATE TABLE user_orders_by_date (
    user_id UUID,
    order_date TIMESTAMP,
    order_id UUID,
    product_name TEXT,
    price DECIMAL,
    PRIMARY KEY ((user_id), order_date, order_id)
) WITH CLUSTERING ORDER BY (order_date DESC);

-- Запрос только по partition key эффективен
SELECT * FROM user_orders_by_date WHERE user_id = ?;

-- Запрос с диапазоном дат
SELECT * FROM user_orders_by_date
WHERE user_id = ?
AND order_date >= '2024-01-01' AND order_date < '2024-02-01';
```

**PostgreSQL** с партиционированием:

```sql
-- PostgreSQL: партиционированная таблица
CREATE TABLE orders (
    id UUID DEFAULT gen_random_uuid(),
    user_id UUID,
    order_date TIMESTAMP,
    product_name VARCHAR(255),
    price DECIMAL(10, 2)
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Индекс на user_id
CREATE INDEX idx_orders_user ON orders(user_id);

-- Запрос
SELECT * FROM orders
WHERE user_id = '...'
AND order_date >= '2024-01-01' AND order_date < '2024-02-01';
```

### Сравнение

| Аспект | PostgreSQL | Cassandra |
|--------|------------|-----------|
| **CAP теорема** | CP (Consistency + Partition tolerance) | AP (Availability + Partition tolerance) |
| **Согласованность** | Строгая (ACID) | Eventual consistency |
| **Масштабирование** | Вертикальное + read replicas | Горизонтальное (линейное) |
| **Модель данных** | Нормализованная | Денормализованная |
| **Запросы** | Гибкий SQL | Ограниченный CQL |
| **Write throughput** | Высокий | Очень высокий |
| **Joins** | Да | Нет |

### Когда что выбрать

**Cassandra лучше для:**
- Очень высокой нагрузки на запись (миллионы ops/sec)
- Географически распределённых данных
- Данных временных рядов
- Когда eventual consistency допустима

**PostgreSQL лучше для:**
- Транзакционных систем
- Сложных запросов
- Когда нужна строгая согласованность
- Умеренных объёмов данных (до нескольких TB)

## PostgreSQL vs Neo4j (графовые БД)

### Модель данных

**Neo4j** — графовая база данных:

```cypher
// Neo4j: создание узлов и связей
CREATE (john:Person {name: 'John', age: 30})
CREATE (alice:Person {name: 'Alice', age: 28})
CREATE (acme:Company {name: 'ACME Inc'})

CREATE (john)-[:KNOWS {since: 2020}]->(alice)
CREATE (john)-[:WORKS_AT {role: 'Developer'}]->(acme)
CREATE (alice)-[:WORKS_AT {role: 'Manager'}]->(acme)

// Запрос: друзья друзей
MATCH (john:Person {name: 'John'})-[:KNOWS*2..3]-(friend)
RETURN DISTINCT friend.name

// Кратчайший путь
MATCH path = shortestPath(
    (john:Person {name: 'John'})-[*]-(target:Person {name: 'Bob'})
)
RETURN path
```

**PostgreSQL** для графов:

```sql
-- PostgreSQL: таблицы для графа
CREATE TABLE nodes (
    id SERIAL PRIMARY KEY,
    type VARCHAR(50),
    properties JSONB
);

CREATE TABLE edges (
    id SERIAL PRIMARY KEY,
    source_id INTEGER REFERENCES nodes(id),
    target_id INTEGER REFERENCES nodes(id),
    type VARCHAR(50),
    properties JSONB
);

-- Рекурсивный запрос для поиска путей
WITH RECURSIVE friends AS (
    -- Базовый случай: прямые друзья
    SELECT
        e.target_id,
        1 AS depth,
        ARRAY[e.source_id, e.target_id] AS path
    FROM edges e
    WHERE e.source_id = 1 AND e.type = 'KNOWS'

    UNION ALL

    -- Рекурсия: друзья друзей
    SELECT
        e.target_id,
        f.depth + 1,
        f.path || e.target_id
    FROM friends f
    JOIN edges e ON e.source_id = f.target_id
    WHERE f.depth < 3
      AND e.type = 'KNOWS'
      AND NOT (e.target_id = ANY(f.path))  -- избегаем циклов
)
SELECT DISTINCT n.properties->>'name' AS friend_name, f.depth
FROM friends f
JOIN nodes n ON n.id = f.target_id;
```

### Сравнение

| Аспект | PostgreSQL | Neo4j |
|--------|------------|-------|
| **Модель** | Реляционная | Графовая |
| **Обход графа** | Рекурсивные CTE | Нативный |
| **Производительность графов** | Хорошая для простых | Отличная для сложных |
| **Язык запросов** | SQL | Cypher |
| **ACID** | Да | Да |
| **Гибкость** | Универсальная | Специализированная |

## PostgreSQL как "универсальный солдат"

PostgreSQL может частично заменить многие NoSQL решения:

```sql
-- Документное хранилище (как MongoDB)
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection VARCHAR(50),
    data JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_docs_collection ON documents(collection);
CREATE INDEX idx_docs_data ON documents USING GIN(data);

-- Ключ-значение (как Redis, но медленнее)
CREATE UNLOGGED TABLE kv_store (
    key VARCHAR(255) PRIMARY KEY,
    value BYTEA,
    expires_at TIMESTAMP
);

-- Временные ряды (как InfluxDB)
CREATE TABLE metrics (
    time TIMESTAMPTZ NOT NULL,
    device_id INTEGER,
    metric_name VARCHAR(50),
    value DOUBLE PRECISION
) PARTITION BY RANGE (time);

-- Полнотекстовый поиск (как Elasticsearch, базовый)
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    content TEXT,
    search_vector TSVECTOR GENERATED ALWAYS AS (
        to_tsvector('russian', title || ' ' || content)
    ) STORED
);
CREATE INDEX idx_articles_search ON articles USING GIN(search_vector);
```

## Сводная таблица

| Критерий | PostgreSQL | MongoDB | Redis | Cassandra | Neo4j |
|----------|------------|---------|-------|-----------|-------|
| **Модель** | Реляционная | Документная | Ключ-значение | Колоночная | Графовая |
| **ACID** | Да | Частично | Нет | Нет | Да |
| **Схема** | Строгая | Гибкая | Нет | Строгая | Гибкая |
| **Joins** | Да | Ограниченно | Нет | Нет | Да |
| **Масштабирование** | Вертикальное | Горизонтальное | Горизонтальное | Горизонтальное | Вертикальное |
| **Скорость чтения** | Высокая | Высокая | Очень высокая | Высокая | Высокая (графы) |
| **Скорость записи** | Высокая | Высокая | Очень высокая | Очень высокая | Средняя |
| **Use case** | Универсальный | Контент, каталоги | Кэш, сессии | Логи, IoT | Соц. сети, рекомендации |

## Best Practices выбора

### Когда PostgreSQL — лучший выбор

1. **Транзакционные системы** — финансы, e-commerce
2. **Сложные запросы** — аналитика, отчёты
3. **Структурированные данные** с связями
4. **Небольшие и средние проекты** — один инструмент на всё
5. **Когда нужна надёжность** и ACID-гарантии

### Когда стоит рассмотреть NoSQL

1. **Redis** — кэширование, сессии, real-time данные
2. **MongoDB** — прототипы, CMS, каталоги товаров
3. **Cassandra** — логи, IoT, временные ряды с огромными объёмами
4. **Neo4j** — социальные графы, рекомендательные системы
5. **Elasticsearch** — полнотекстовый поиск, логи

### Полиглот Persistence

В реальных проектах часто используют несколько баз данных:

```
┌─────────────────────────────────────────────────────────┐
│                     Application                          │
└─────────────────────────────────────────────────────────┘
         │              │              │              │
         ▼              ▼              ▼              ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ PostgreSQL  │ │   Redis     │ │Elasticsearch│ │   S3/MinIO  │
│ (основные   │ │   (кэш,     │ │   (поиск,   │ │  (файлы,    │
│  данные)    │ │  сессии)    │ │   логи)     │ │  бэкапы)    │
└─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘
```

## Заключение

PostgreSQL — универсальный инструмент, который подходит для большинства задач. Благодаря JSONB, расширениям и постоянному развитию, он может эффективно работать со многими типами данных.

NoSQL базы данных не заменяют PostgreSQL, а дополняют его для специфических сценариев:
- Экстремальное масштабирование
- Специализированные модели данных
- Ультранизкая задержка

**Начинайте с PostgreSQL**, добавляйте специализированные решения по мере необходимости.
