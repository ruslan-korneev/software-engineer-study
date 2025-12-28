# Шардирование данных (Sharding Patterns)

[prev: 02-data-partitioning](./02-data-partitioning.md) | [next: 04-normalization](./04-normalization.md)

---

## Введение

Шардирование (Sharding) - это горизонтальное масштабирование базы данных путем распределения данных между несколькими серверами (шардами). В отличие от партиционирования, которое работает в рамках одного сервера, шардирование распределяет данные между физически разными базами данных.

## Зачем нужно шардирование

### Преимущества

1. **Горизонтальное масштабирование** - увеличение мощности путем добавления серверов
2. **Распределение нагрузки** - запросы распределяются между шардами
3. **Увеличение объема хранения** - каждый шард хранит часть данных
4. **Географическое распределение** - данные ближе к пользователям
5. **Изоляция отказов** - сбой одного шарда не влияет на другие

### Когда использовать

- Данные не помещаются на один сервер
- Вертикальное масштабирование достигло предела
- Нужна высокая доступность и отказоустойчивость
- Географическое распределение пользователей

## Стратегии шардирования

### 1. Range-based Sharding (по диапазону)

Данные распределяются по диапазонам ключей.

```
Шард 1: user_id 1-1000000
Шард 2: user_id 1000001-2000000
Шард 3: user_id 2000001-3000000
```

```sql
-- Логика определения шарда (псевдокод)
CREATE OR REPLACE FUNCTION get_shard_by_range(user_id BIGINT)
RETURNS TEXT AS $$
BEGIN
    IF user_id <= 1000000 THEN
        RETURN 'shard_1';
    ELSIF user_id <= 2000000 THEN
        RETURN 'shard_2';
    ELSE
        RETURN 'shard_3';
    END IF;
END;
$$ LANGUAGE plpgsql;
```

**Преимущества:**
- Простая реализация
- Эффективные range-запросы

**Недостатки:**
- Неравномерное распределение (hot spots)
- Сложное добавление новых шардов

### 2. Hash-based Sharding (по хешу)

Данные распределяются на основе хеш-функции.

```sql
-- Функция определения шарда по хешу
CREATE OR REPLACE FUNCTION get_shard_by_hash(
    key_value TEXT,
    num_shards INTEGER DEFAULT 4
)
RETURNS INTEGER AS $$
BEGIN
    RETURN abs(hashtext(key_value)) % num_shards;
END;
$$ LANGUAGE plpgsql;

-- Использование
SELECT get_shard_by_hash('user_12345', 4);  -- Возвращает 0, 1, 2 или 3
```

**Преимущества:**
- Равномерное распределение данных
- Нет hot spots

**Недостатки:**
- Range-запросы требуют обращения ко всем шардам
- Сложное добавление шардов (resharding)

### 3. Directory-based Sharding (по справочнику)

Отдельный сервис хранит маппинг ключей на шарды.

```sql
-- Таблица маппинга (в отдельной БД)
CREATE TABLE shard_mapping (
    entity_type VARCHAR(50),
    entity_id BIGINT,
    shard_id INTEGER,
    PRIMARY KEY (entity_type, entity_id)
);

-- Индекс для быстрого поиска
CREATE INDEX idx_shard_mapping ON shard_mapping(shard_id);

-- Получение шарда
SELECT shard_id FROM shard_mapping
WHERE entity_type = 'user' AND entity_id = 12345;
```

**Преимущества:**
- Гибкость в распределении
- Легкое добавление/удаление шардов

**Недостатки:**
- Дополнительная точка отказа
- Накладные расходы на lookup

### 4. Geographic Sharding (по географии)

Данные распределяются по географическому признаку.

```sql
-- Маршрутизация по региону
CREATE OR REPLACE FUNCTION get_shard_by_region(region VARCHAR)
RETURNS TEXT AS $$
BEGIN
    CASE region
        WHEN 'EU' THEN RETURN 'shard_europe';
        WHEN 'US' THEN RETURN 'shard_usa';
        WHEN 'ASIA' THEN RETURN 'shard_asia';
        ELSE RETURN 'shard_default';
    END CASE;
END;
$$ LANGUAGE plpgsql;
```

## Реализация шардирования в PostgreSQL

### Использование Foreign Data Wrapper (FDW)

```sql
-- На координаторе (центральном сервере)
CREATE EXTENSION postgres_fdw;

-- Создание серверов для каждого шарда
CREATE SERVER shard_1
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host 'shard1.example.com', dbname 'mydb', port '5432');

CREATE SERVER shard_2
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host 'shard2.example.com', dbname 'mydb', port '5432');

-- Создание маппинга пользователей
CREATE USER MAPPING FOR current_user
SERVER shard_1 OPTIONS (user 'postgres', password 'secret');

CREATE USER MAPPING FOR current_user
SERVER shard_2 OPTIONS (user 'postgres', password 'secret');

-- Создание внешних таблиц
CREATE FOREIGN TABLE users_shard_1 (
    id BIGINT,
    name VARCHAR(100),
    email VARCHAR(255)
) SERVER shard_1 OPTIONS (table_name 'users');

CREATE FOREIGN TABLE users_shard_2 (
    id BIGINT,
    name VARCHAR(100),
    email VARCHAR(255)
) SERVER shard_2 OPTIONS (table_name 'users');

-- Создание view для объединения данных
CREATE VIEW users AS
    SELECT * FROM users_shard_1
    UNION ALL
    SELECT * FROM users_shard_2;
```

### Партиционирование + FDW

```sql
-- Создание партиционированной таблицы
CREATE TABLE orders (
    id BIGSERIAL,
    customer_id INTEGER NOT NULL,
    order_date DATE NOT NULL,
    total DECIMAL(10,2)
) PARTITION BY HASH (customer_id);

-- Локальная партиция
CREATE TABLE orders_local PARTITION OF orders
    FOR VALUES WITH (MODULUS 2, REMAINDER 0);

-- Удаленная партиция (на другом сервере)
CREATE FOREIGN TABLE orders_remote PARTITION OF orders
    FOR VALUES WITH (MODULUS 2, REMAINDER 1)
    SERVER shard_2;
```

## Citus - расширение для шардирования PostgreSQL

Citus превращает PostgreSQL в распределенную базу данных.

### Установка и настройка Citus

```sql
-- Установка расширения
CREATE EXTENSION citus;

-- Добавление worker-нод
SELECT citus_add_node('worker1.example.com', 5432);
SELECT citus_add_node('worker2.example.com', 5432);

-- Проверка кластера
SELECT * FROM citus_get_active_worker_nodes();
```

### Distributed Tables (распределенные таблицы)

```sql
-- Создание таблицы
CREATE TABLE events (
    tenant_id INTEGER,
    event_id BIGSERIAL,
    event_type VARCHAR(50),
    payload JSONB,
    created_at TIMESTAMP
);

-- Распределение таблицы по tenant_id
SELECT create_distributed_table('events', 'tenant_id');

-- Вставка данных (Citus автоматически направляет на нужный шард)
INSERT INTO events (tenant_id, event_type, payload)
VALUES (1, 'click', '{"page": "/home"}');
```

### Reference Tables (референсные таблицы)

```sql
-- Таблицы-справочники реплицируются на все ноды
CREATE TABLE countries (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    code CHAR(2)
);

SELECT create_reference_table('countries');
```

### Co-location в Citus

```sql
-- Создание связанных таблиц с одинаковым распределением
CREATE TABLE customers (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE orders (
    id BIGSERIAL,
    customer_id INTEGER REFERENCES customers(id),
    total DECIMAL(10,2)
);

-- Распределение с co-location
SELECT create_distributed_table('customers', 'id');
SELECT create_distributed_table('orders', 'customer_id', colocate_with => 'customers');

-- JOIN будет выполняться локально на шарде
SELECT c.name, SUM(o.total)
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.name;
```

## Шардирование на уровне приложения

### Python пример с маршрутизацией

```python
import hashlib
from sqlalchemy import create_engine

class ShardRouter:
    def __init__(self, num_shards=4):
        self.num_shards = num_shards
        self.shards = {
            0: create_engine('postgresql://user:pass@shard0/db'),
            1: create_engine('postgresql://user:pass@shard1/db'),
            2: create_engine('postgresql://user:pass@shard2/db'),
            3: create_engine('postgresql://user:pass@shard3/db'),
        }

    def get_shard(self, shard_key):
        """Определение шарда по ключу"""
        hash_value = int(hashlib.md5(str(shard_key).encode()).hexdigest(), 16)
        shard_id = hash_value % self.num_shards
        return self.shards[shard_id]

    def execute_on_shard(self, shard_key, query, params=None):
        """Выполнение запроса на нужном шарде"""
        engine = self.get_shard(shard_key)
        with engine.connect() as conn:
            return conn.execute(query, params)

    def execute_on_all_shards(self, query, params=None):
        """Выполнение запроса на всех шардах"""
        results = []
        for shard_id, engine in self.shards.items():
            with engine.connect() as conn:
                results.extend(conn.execute(query, params).fetchall())
        return results

# Использование
router = ShardRouter()

# Вставка данных
user_id = 12345
router.execute_on_shard(
    user_id,
    "INSERT INTO users (id, name) VALUES (%s, %s)",
    (user_id, "John Doe")
)

# Выборка данных
user = router.execute_on_shard(
    user_id,
    "SELECT * FROM users WHERE id = %s",
    (user_id,)
)
```

### Scatter-Gather Pattern

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

class ScatterGather:
    def __init__(self, router):
        self.router = router
        self.executor = ThreadPoolExecutor(max_workers=10)

    async def query_all_shards(self, query, params=None):
        """Параллельный запрос ко всем шардам"""
        loop = asyncio.get_event_loop()

        tasks = []
        for shard_id, engine in self.router.shards.items():
            task = loop.run_in_executor(
                self.executor,
                self._execute_query,
                engine, query, params
            )
            tasks.append(task)

        results = await asyncio.gather(*tasks)
        return self._merge_results(results)

    def _execute_query(self, engine, query, params):
        with engine.connect() as conn:
            return conn.execute(query, params).fetchall()

    def _merge_results(self, results):
        """Объединение результатов со всех шардов"""
        merged = []
        for result in results:
            merged.extend(result)
        return merged

# Использование
sg = ScatterGather(router)
all_users = await sg.query_all_shards("SELECT * FROM users WHERE created_at > %s", ('2024-01-01',))
```

## Resharding (перераспределение данных)

### Consistent Hashing

```python
import hashlib
from bisect import bisect_left

class ConsistentHash:
    def __init__(self, nodes=None, replicas=100):
        self.replicas = replicas
        self.ring = []
        self.nodes = {}

        if nodes:
            for node in nodes:
                self.add_node(node)

    def _hash(self, key):
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node):
        """Добавление ноды в кольцо"""
        for i in range(self.replicas):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)
            self.ring.append(hash_value)
            self.nodes[hash_value] = node
        self.ring.sort()

    def remove_node(self, node):
        """Удаление ноды из кольца"""
        for i in range(self.replicas):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)
            self.ring.remove(hash_value)
            del self.nodes[hash_value]

    def get_node(self, key):
        """Получение ноды для ключа"""
        if not self.ring:
            return None

        hash_value = self._hash(str(key))
        idx = bisect_left(self.ring, hash_value)

        if idx == len(self.ring):
            idx = 0

        return self.nodes[self.ring[idx]]

# Использование
ch = ConsistentHash(['shard1', 'shard2', 'shard3', 'shard4'])

# Добавление нового шарда перераспределяет только часть данных
ch.add_node('shard5')
```

### Двухфазный resharding

```sql
-- Фаза 1: Копирование данных
-- На новом шарде
INSERT INTO users
SELECT * FROM dblink('old_shard', 'SELECT * FROM users WHERE shard_key_condition')
AS t(id BIGINT, name VARCHAR, email VARCHAR);

-- Фаза 2: Переключение и удаление
-- Обновить маршрутизацию
UPDATE shard_mapping
SET shard_id = new_shard_id
WHERE entity_type = 'user' AND condition;

-- Удалить данные со старого шарда
DELETE FROM users WHERE shard_key_condition;
```

## Cross-shard Queries (запросы между шардами)

### Проблема JOIN между шардами

```sql
-- Этот запрос требует данных с разных шардов
SELECT u.name, COUNT(o.id) as order_count
FROM users u
JOIN orders o ON u.id = o.user_id
GROUP BY u.name;

-- Решение 1: Денормализация
-- Хранить имя пользователя в таблице orders

-- Решение 2: Co-location
-- Распределять связанные таблицы по одному ключу

-- Решение 3: Scatter-gather на уровне приложения
```

### Distributed Transactions

```python
# Использование 2PC (Two-Phase Commit)
import psycopg2

def transfer_between_shards(from_shard, to_shard, amount, from_account, to_account):
    conn1 = psycopg2.connect(from_shard)
    conn2 = psycopg2.connect(to_shard)

    try:
        # Фаза 1: Prepare
        cursor1 = conn1.cursor()
        cursor2 = conn2.cursor()

        cursor1.execute("BEGIN")
        cursor2.execute("BEGIN")

        cursor1.execute(
            "UPDATE accounts SET balance = balance - %s WHERE id = %s",
            (amount, from_account)
        )
        cursor2.execute(
            "UPDATE accounts SET balance = balance + %s WHERE id = %s",
            (amount, to_account)
        )

        # Фаза 2: Commit
        conn1.commit()
        conn2.commit()

    except Exception as e:
        conn1.rollback()
        conn2.rollback()
        raise e
    finally:
        conn1.close()
        conn2.close()
```

## Мониторинг шардированной системы

```sql
-- Проверка распределения данных по шардам
SELECT
    shard_id,
    COUNT(*) as row_count,
    pg_size_pretty(SUM(pg_column_size(t.*))) as data_size
FROM my_table t
GROUP BY shard_id;

-- В Citus
SELECT * FROM citus_shards;
SELECT * FROM citus_stat_statements;
```

## Best Practices

### 1. Выбор ключа шардирования

```sql
-- Хорошо: высокая кардинальность, часто используется в запросах
-- tenant_id, user_id, account_id

-- Плохо: низкая кардинальность, skew
-- status, country_code, boolean fields
```

### 2. Планирование шардов заранее

- Начните с запаса (больше шардов, чем нужно сейчас)
- Используйте consistent hashing для упрощения resharding
- Документируйте стратегию шардирования

### 3. Избегайте cross-shard транзакций

- Проектируйте схему для локальных операций
- Используйте eventual consistency где возможно

## Типичные ошибки

1. **Неправильный выбор shard key** - приводит к hot spots
2. **Слишком много cross-shard запросов** - снижает производительность
3. **Отсутствие плана resharding** - сложно масштабировать
4. **Игнорирование data locality** - JOIN между шардами очень дороги
5. **Недостаточный мониторинг** - не видно проблем с распределением

---

[prev: 02-data-partitioning](./02-data-partitioning.md) | [next: 04-normalization](./04-normalization.md)
