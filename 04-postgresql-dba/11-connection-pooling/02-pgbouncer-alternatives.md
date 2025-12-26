# Альтернативы PgBouncer

## Введение

Хотя PgBouncer является наиболее популярным пулером соединений для PostgreSQL, существует ряд альтернативных решений, каждое со своими особенностями и преимуществами. В этом разделе рассмотрим основные альтернативы:

1. **Pgpool-II** — многофункциональный middleware
2. **Odyssey** — современный пулер от Яндекса
3. **PgCat** — новый пулер на Rust
4. **Встроенный пулинг приложений** — connection pools в ORM/драйверах

## Pgpool-II

### Обзор

**Pgpool-II** — это не просто пулер соединений, а полноценный middleware для PostgreSQL, предоставляющий:

- Connection pooling
- Репликация (native replication)
- Load balancing
- Query caching
- Watchdog для высокой доступности
- Online recovery

```
                    ┌─────────────────┐
                    │   Pgpool-II     │
Clients ──────────► │  - Pooling      │ ──────► PostgreSQL Primary
                    │  - Load Balance │ ──────► PostgreSQL Replica 1
                    │  - Query Cache  │ ──────► PostgreSQL Replica 2
                    │  - Watchdog     │
                    └─────────────────┘
```

### Установка

```bash
# Ubuntu/Debian
sudo apt install pgpool2

# CentOS/RHEL
sudo yum install pgpool-II

# Из исходников
wget https://www.pgpool.net/mediawiki/images/pgpool-II-4.4.4.tar.gz
tar xzf pgpool-II-4.4.4.tar.gz
cd pgpool-II-4.4.4
./configure
make
make install
```

### Базовая конфигурация

```ini
# /etc/pgpool2/pgpool.conf

# Прослушивание
listen_addresses = '*'
port = 9999
socket_dir = '/var/run/pgpool'

# Backend серверы
backend_hostname0 = 'primary.db.local'
backend_port0 = 5432
backend_weight0 = 1
backend_flag0 = 'ALLOW_TO_FAILOVER'

backend_hostname1 = 'replica1.db.local'
backend_port1 = 5432
backend_weight1 = 1
backend_flag1 = 'ALLOW_TO_FAILOVER'

# Connection pooling
connection_cache = on
num_init_children = 32
max_pool = 4
child_life_time = 300
client_idle_limit = 0
connection_life_time = 0

# Load balancing
load_balance_mode = on
ignore_leading_white_space = on
database_redirect_preference_list = ''
app_name_redirect_preference_list = ''

# Репликация
replication_mode = off
master_slave_mode = on
master_slave_sub_mode = 'stream'

# Аутентификация
enable_pool_hba = on
pool_passwd = 'pool_passwd'
authentication_timeout = 60

# Health check
health_check_period = 10
health_check_timeout = 20
health_check_user = 'pgpool'
health_check_password = 'pgpool_password'
health_check_max_retries = 3
health_check_retry_delay = 1

# Failover
failover_command = '/etc/pgpool2/failover.sh %d %h %p %D %m %H %M %P %r %R'
failback_command = ''

# Query cache (опционально)
memory_cache_enabled = off
memqcache_method = 'memcached'
memqcache_memcached_host = 'localhost'
memqcache_memcached_port = 11211
```

### Настройка аутентификации

```ini
# /etc/pgpool2/pool_hba.conf
# TYPE  DATABASE    USER        ADDRESS          METHOD
local   all         all                          trust
host    all         all         127.0.0.1/32     md5
host    all         all         192.168.0.0/16   md5
host    all         all         ::1/128          md5
```

### Скрипт failover

```bash
#!/bin/bash
# /etc/pgpool2/failover.sh

FAILED_NODE=$1
FAILED_HOST=$2
FAILED_PORT=$3
FAILED_DB_CLUSTER=$4
NEW_MASTER=$5
NEW_MASTER_HOST=$6
NEW_MASTER_PORT=$7
OLD_MASTER=$8
OLD_MASTER_HOST=$9
OLD_MASTER_PORT=${10}

echo "Failover triggered: node $FAILED_NODE ($FAILED_HOST) failed"
echo "New master: $NEW_MASTER_HOST:$NEW_MASTER_PORT"

# Promote replica to primary
ssh -T postgres@$NEW_MASTER_HOST "pg_ctl promote -D /var/lib/postgresql/data"

# Notify administrators
echo "PostgreSQL failover occurred" | mail -s "DB Failover Alert" admin@example.com
```

### Load Balancing

```ini
# Распределение нагрузки по весам
backend_weight0 = 1   # Primary получает 25% запросов
backend_weight1 = 2   # Replica 1 получает 50% запросов
backend_weight2 = 1   # Replica 2 получает 25% запросов

# Направление запросов на чтение на реплики
load_balance_mode = on

# Запросы, которые всегда идут на primary
black_function_list = 'nextval,setval,lastval'

# Запросы, которые могут идти на реплику
white_function_list = ''
```

### Мониторинг Pgpool-II

```sql
-- Подключение к виртуальной БД pgpool
psql -h localhost -p 9999 -U postgres pgpool

-- Показать статус нод
SHOW pool_nodes;

-- Показать статус пулов
SHOW pool_pools;

-- Показать процессы
SHOW pool_processes;

-- Показать кеш
SHOW pool_cache;

-- Показать версию
SHOW pool_version;
```

### Сравнение PgBouncer vs Pgpool-II

| Характеристика | PgBouncer | Pgpool-II |
|----------------|-----------|-----------|
| Потребление памяти | ~2 МБ | ~100+ МБ |
| Connection pooling | ✅ Отличный | ✅ Хороший |
| Load balancing | ❌ | ✅ |
| Репликация | ❌ | ✅ |
| Query caching | ❌ | ✅ |
| Failover | ❌ | ✅ |
| Простота настройки | ✅ Простой | ⚠️ Сложный |
| Производительность | ✅ Высокая | ⚠️ Средняя |
| Поддержка SSL | ✅ | ✅ |

## Odyssey

### Обзор

**Odyssey** — это современный многопоточный пулер соединений, разработанный в Яндексе. Он спроектирован для работы с очень большими нагрузками и предоставляет:

- Многопоточная архитектура (в отличие от однопоточного PgBouncer)
- Расширенный мониторинг
- Гибкая конфигурация на уровне пользователей/БД
- Поддержка SCRAM-SHA-256
- Graceful restart

```
                    ┌─────────────────────────┐
                    │        Odyssey          │
                    │  ┌─────┐ ┌─────┐ ┌─────┐│
Clients ──────────► │  │ W1  │ │ W2  │ │ W3  ││ ──────► PostgreSQL
                    │  └─────┘ └─────┘ └─────┘│
                    │     (worker threads)    │
                    └─────────────────────────┘
```

### Установка

```bash
# Из репозитория Яндекса (Ubuntu)
curl -fsSL https://repo.yandex.ru/odyssey/gpg.key | sudo apt-key add -
echo "deb https://repo.yandex.ru/odyssey stable main" | sudo tee /etc/apt/sources.list.d/odyssey.list
sudo apt update
sudo apt install odyssey

# Сборка из исходников
git clone https://github.com/yandex/odyssey.git
cd odyssey
mkdir build && cd build
cmake ..
make
make install
```

### Конфигурация

```yaml
# /etc/odyssey/odyssey.conf

# Глобальные настройки
daemonize yes
pid_file "/var/run/odyssey/odyssey.pid"
log_file "/var/log/odyssey/odyssey.log"
log_format "%p %t %l [%i %s] (%c) %m\n"
log_debug no
log_config yes
log_session yes
log_query no
log_stats yes
stats_interval 60

# Рабочие потоки
workers 4
resolvers 1

# Прослушивание
listen {
    host "*"
    port 6432
    backlog 128
    tls "disable"
}

# Правила маршрутизации
storage "postgres_server" {
    type "remote"
    host "localhost"
    port 5432
    tls "disable"
}

# Настройки базы данных
database "mydb" {
    user "myuser" {
        authentication "md5"
        password "mypassword"

        storage "postgres_server"
        storage_db "mydb"
        storage_user "myuser"
        storage_password "mypassword"

        pool "transaction"
        pool_size 20
        pool_timeout 0
        pool_ttl 60
        pool_discard yes
        pool_cancel yes
        pool_rollback yes

        client_max 1000

        # Форвардинг параметров
        forward_params ["application_name", "client_encoding"]
    }
}

# Административный доступ
database "console" {
    user default {
        authentication "none"
        pool "session"
        storage "local"
    }
}

# Default правило
database default {
    user default {
        authentication "md5"
        auth_common_name default
        auth_pam off
        auth_pam_service "odyssey"

        storage "postgres_server"

        pool "transaction"
        pool_size 10
        pool_timeout 0
        pool_ttl 60
        pool_discard yes
        pool_cancel yes
        pool_rollback yes

        client_max 100
    }
}
```

### Расширенная конфигурация

```yaml
# Разные пулы для разных пользователей
database "app_db" {
    # Обычный пользователь приложения
    user "app_user" {
        authentication "scram-sha-256"
        password "app_password"

        storage "postgres_server"
        storage_db "app_db"
        storage_user "app_user"
        storage_password "app_password"

        pool "transaction"
        pool_size 50
        pool_timeout 10000      # 10 секунд ожидания
        pool_ttl 3600           # 1 час жизни соединения
        pool_discard yes
        pool_rollback yes

        client_max 5000

        # Мониторинг
        log_debug no
        log_query no
        quantiles "0.99,0.95,0.5"
    }

    # Миграции/административные задачи
    user "admin_user" {
        authentication "scram-sha-256"
        password "admin_password"

        storage "postgres_server"
        storage_db "app_db"
        storage_user "admin_user"
        storage_password "admin_password"

        pool "session"          # Session mode для DDL
        pool_size 5
        client_max 10
    }

    # Read-only пользователь для аналитики
    user "analytics" {
        authentication "md5"
        password "analytics_password"

        storage "postgres_replica"  # Отдельный storage для реплики
        storage_db "app_db"
        storage_user "analytics"
        storage_password "analytics_password"

        pool "transaction"
        pool_size 20
        client_max 100
    }
}

# Реплика для чтения
storage "postgres_replica" {
    type "remote"
    host "replica.db.local"
    port 5432
    tls "require"
    tls_ca_file "/etc/odyssey/ca.crt"
}
```

### Мониторинг Odyssey

```bash
# Подключение к консоли
psql -h localhost -p 6432 -d console

# Команды консоли
SHOW stats;
SHOW pools;
SHOW databases;
SHOW servers;
SHOW clients;
SHOW lists;
SHOW errors;
```

### Prometheus метрики

Odyssey поддерживает экспорт метрик в формате Prometheus:

```yaml
# odyssey.conf
prometheus {
    port 9090
}
```

Метрики доступны по адресу `http://localhost:9090/metrics`.

## PgCat

### Обзор

**PgCat** — это новый пулер соединений, написанный на Rust. Он разработан для современных cloud-native окружений:

- Написан на Rust (безопасность памяти)
- Sharding (разделение данных между серверами)
- Query load balancing
- Automatic failover
- Health checks
- Prometheus метрики
- Graceful restarts

### Установка

```bash
# Из cargo
cargo install pgcat

# Docker
docker run -d \
  --name pgcat \
  -v /path/to/pgcat.toml:/etc/pgcat/pgcat.toml \
  -p 6432:6432 \
  ghcr.io/postgresml/pgcat:latest

# Из исходников
git clone https://github.com/postgresml/pgcat.git
cd pgcat
cargo build --release
```

### Конфигурация

```toml
# /etc/pgcat/pgcat.toml

[general]
host = "0.0.0.0"
port = 6432
admin_username = "admin"
admin_password = "admin_password"
connect_timeout = 5000           # ms
idle_timeout = 30000             # ms
shutdown_timeout = 60000         # ms
healthcheck_timeout = 1000       # ms
healthcheck_delay = 5000         # ms
ban_time = 60                    # seconds
autoreload = 15000               # ms, 0 to disable
worker_threads = 4
tcp_keepalives_idle = 5
tcp_keepalives_count = 5
tcp_keepalives_interval = 5
log_client_connections = false
log_client_disconnections = false
prometheus_exporter_port = 9930

# TLS настройки
[general.tls]
server_cert = "/etc/pgcat/server.crt"
server_key = "/etc/pgcat/server.key"

[pools.mydb]
pool_mode = "transaction"
default_role = "primary"
query_parser_enabled = true
query_parser_read_write_splitting = true
query_parser_max_length = 100
primary_reads_enabled = false
sharding_function = "pg_bigint_hash"
automatic_sharding_key = "id"
healthcheck_database = "postgres"
healthcheck_user = "postgres"
healthcheck_password = "postgres"
auth_query = "SELECT usename, passwd FROM pg_shadow WHERE usename=$1"
auth_query_user = "postgres"
auth_query_password = "postgres"
idle_client_in_transaction_timeout = 0
server_lifetime = 86400000       # ms
# ban_time = 60                  # override global
# connect_timeout = 5000         # override global

[pools.mydb.users.0]
username = "app_user"
password = "app_password"
pool_size = 20
min_pool_size = 5
pool_mode = "transaction"
statement_timeout = 30000

[pools.mydb.users.1]
username = "admin"
password = "admin_password"
pool_size = 5
pool_mode = "session"

[pools.mydb.shards.0]
database = "mydb"
servers = [
    ["primary.db.local", 5432, "primary"],
    ["replica1.db.local", 5432, "replica"],
    ["replica2.db.local", 5432, "replica"],
]

[pools.mydb.shards.1]
database = "mydb"
servers = [
    ["primary2.db.local", 5432, "primary"],
    ["replica3.db.local", 5432, "replica"],
]
```

### Sharding

PgCat поддерживает автоматический шардинг:

```toml
[pools.sharded_db]
sharding_function = "pg_bigint_hash"
automatic_sharding_key = "user_id"
shards = 4

[pools.sharded_db.shards.0]
database = "shard_0"
servers = [["shard0.db.local", 5432, "primary"]]

[pools.sharded_db.shards.1]
database = "shard_1"
servers = [["shard1.db.local", 5432, "primary"]]

[pools.sharded_db.shards.2]
database = "shard_2"
servers = [["shard2.db.local", 5432, "primary"]]

[pools.sharded_db.shards.3]
database = "shard_3"
servers = [["shard3.db.local", 5432, "primary"]]
```

### Read/Write Splitting

```toml
[pools.mydb]
query_parser_enabled = true
query_parser_read_write_splitting = true
primary_reads_enabled = false   # SELECT всегда на реплики

[pools.mydb.shards.0]
servers = [
    ["primary.db.local", 5432, "primary"],
    ["replica1.db.local", 5432, "replica"],
    ["replica2.db.local", 5432, "replica"],
]
```

### Мониторинг PgCat

```bash
# Административная консоль
psql -h localhost -p 6432 -U admin -d pgcat

# Команды
SHOW STATS;
SHOW POOLS;
SHOW DATABASES;
SHOW CONFIG;
SHOW VERSION;
RELOAD;
```

Prometheus метрики: `http://localhost:9930/metrics`

## Сравнение всех решений

| Характеристика | PgBouncer | Pgpool-II | Odyssey | PgCat |
|----------------|-----------|-----------|---------|-------|
| **Язык** | C | C | C | Rust |
| **Архитектура** | Single-threaded | Multi-process | Multi-threaded | Multi-threaded |
| **Память** | ~2 МБ | 100+ МБ | ~10-50 МБ | ~20-50 МБ |
| **Pooling** | ✅ Отличный | ✅ Хороший | ✅ Отличный | ✅ Отличный |
| **Load balancing** | ❌ | ✅ | ⚠️ Ограниченный | ✅ |
| **R/W splitting** | ❌ | ✅ | ⚠️ | ✅ |
| **Sharding** | ❌ | ❌ | ❌ | ✅ |
| **Failover** | ❌ | ✅ | ❌ | ✅ |
| **Prometheus** | Через exporter | Через exporter | ✅ Встроенный | ✅ Встроенный |
| **Простота** | ✅ Простой | ❌ Сложный | ✅ Средний | ✅ Средний |
| **Зрелость** | ✅ Очень зрелый | ✅ Зрелый | ✅ Зрелый | ⚠️ Новый |
| **SCRAM-SHA-256** | ✅ | ✅ | ✅ | ✅ |

## Встроенный пулинг в приложениях

### SQLAlchemy (Python)

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    "postgresql://user:pass@localhost:5432/mydb",
    poolclass=QueuePool,
    pool_size=20,
    max_overflow=10,
    pool_timeout=30,
    pool_pre_ping=True,
    pool_recycle=3600,
)
```

### HikariCP (Java)

```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
config.setUsername("user");
config.setPassword("password");
config.setMaximumPoolSize(20);
config.setMinimumIdle(5);
config.setIdleTimeout(300000);
config.setMaxLifetime(1800000);
config.setConnectionTimeout(30000);

HikariDataSource ds = new HikariDataSource(config);
```

### Node.js pg-pool

```javascript
const { Pool } = require('pg');

const pool = new Pool({
    host: 'localhost',
    port: 5432,
    database: 'mydb',
    user: 'user',
    password: 'password',
    max: 20,
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 5000,
});
```

### Когда использовать встроенный пулинг?

**Подходит для:**
- Один инстанс приложения
- Простые развёртывания
- Разработка и тестирование

**Не подходит для:**
- Множество инстансов приложения (каждый создаёт свой пул)
- Микросервисная архитектура
- Необходимость централизованного управления соединениями

## Рекомендации по выбору

### Выбирайте PgBouncer если:
- Нужен простой и надёжный пулер
- Важна минимальная overhead и максимальная производительность
- Не нужен load balancing или failover (используете отдельные инструменты)

### Выбирайте Pgpool-II если:
- Нужен комплексное решение "всё в одном"
- Требуется встроенный load balancing
- Нужен автоматический failover
- Готовы к сложной настройке

### Выбирайте Odyssey если:
- Очень высокая нагрузка (миллионы соединений)
- Нужна многопоточность
- Требуется гибкая настройка на уровне пользователей
- Работаете в Яндекс-инфраструктуре или похожем окружении

### Выбирайте PgCat если:
- Нужен sharding
- Cloud-native окружение
- Read/Write splitting из коробки
- Готовы использовать относительно новый инструмент

### Используйте встроенный пулинг если:
- Простое приложение с одним инстансом
- Разработка и тестирование
- Нет специфических требований к управлению соединениями

## Заключение

Выбор пулера соединений зависит от конкретных требований проекта:

1. **PgBouncer** остаётся лучшим выбором для большинства случаев благодаря простоте и надёжности
2. **Pgpool-II** подходит когда нужно комплексное решение с репликацией и failover
3. **Odyssey** оптимален для очень высоких нагрузок
4. **PgCat** — перспективное решение для cloud-native и sharding

Часто используют комбинацию инструментов: например, PgBouncer для пулинга + Patroni для failover + HAProxy для load balancing.
