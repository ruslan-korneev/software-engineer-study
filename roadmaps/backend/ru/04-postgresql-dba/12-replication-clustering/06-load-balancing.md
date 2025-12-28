# Балансировка нагрузки в PostgreSQL кластерах

[prev: 05-monitoring](./05-monitoring.md) | [next: 01-pg_dump](../13-backup-recovery/01-pg_dump.md)

---

## Введение

Балансировка нагрузки в PostgreSQL кластерах позволяет распределять запросы между несколькими серверами для повышения производительности, отказоустойчивости и масштабируемости. Основные задачи — маршрутизация записи на primary сервер и распределение чтения между репликами.

## Архитектура балансировки нагрузки

### Общая схема

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            Applications                                  │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐            │
│  │   App 1   │  │   App 2   │  │   App 3   │  │   App N   │            │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘            │
└────────┼──────────────┼──────────────┼──────────────┼───────────────────┘
         │              │              │              │
         └──────────────┴──────────────┴──────────────┘
                                │
                    ┌───────────▼───────────┐
                    │    Load Balancer      │
                    │  (HAProxy/PgBouncer)  │
                    └───────────┬───────────┘
                                │
         ┌──────────────────────┼──────────────────────┐
         │                      │                      │
         ▼                      ▼                      ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   PostgreSQL    │  │   PostgreSQL    │  │   PostgreSQL    │
│    Primary      │  │    Replica 1    │  │    Replica 2    │
│  (Read/Write)   │  │   (Read-only)   │  │   (Read-only)   │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

### Типы балансировки

1. **Connection-level** — балансировка на уровне TCP соединений (HAProxy)
2. **Query-level** — анализ и маршрутизация запросов (Pgpool-II)
3. **Application-level** — разделение read/write в приложении

## HAProxy

### Установка HAProxy

```bash
# Ubuntu/Debian
apt-get update
apt-get install -y haproxy

# CentOS/RHEL
yum install -y haproxy

# Проверка версии
haproxy -v
```

### Базовая конфигурация HAProxy

```haproxy
# /etc/haproxy/haproxy.cfg

global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

    # Настройки SSL/TLS
    ssl-default-bind-ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11
    tune.ssl.default-dh-param 2048

defaults
    log     global
    mode    tcp
    option  tcplog
    option  dontlognull
    timeout connect 5000ms
    timeout client  30m
    timeout server  30m
    timeout check   5000ms
    retries 3

# Статистика HAProxy
listen stats
    bind *:7000
    mode http
    stats enable
    stats uri /stats
    stats refresh 10s
    stats admin if TRUE
    stats auth admin:admin_password

# PostgreSQL Primary (Read/Write)
listen postgresql_primary
    bind *:5000
    mode tcp
    option httpchk OPTIONS /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions

    server pg-node1 192.168.1.1:5432 check port 8008
    server pg-node2 192.168.1.2:5432 check port 8008
    server pg-node3 192.168.1.3:5432 check port 8008

# PostgreSQL Replicas (Read-only)
listen postgresql_replicas
    bind *:5001
    mode tcp
    balance roundrobin
    option httpchk OPTIONS /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions

    server pg-node1 192.168.1.1:5432 check port 8008
    server pg-node2 192.168.1.2:5432 check port 8008
    server pg-node3 192.168.1.3:5432 check port 8008

# PostgreSQL All Nodes (Read from any)
listen postgresql_all
    bind *:5002
    mode tcp
    balance roundrobin
    option httpchk OPTIONS /read-only
    http-check expect status 200
    default-server inter 3s fall 3 rise 2

    server pg-node1 192.168.1.1:5432 check port 8008
    server pg-node2 192.168.1.2:5432 check port 8008
    server pg-node3 192.168.1.3:5432 check port 8008
```

### HAProxy с Patroni

```haproxy
# Конфигурация для Patroni кластера

# Primary (Leader) - для записи
listen postgresql_primary
    bind *:5000
    mode tcp
    option httpchk OPTIONS /leader
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions

    server patroni1 192.168.1.1:5432 check port 8008
    server patroni2 192.168.1.2:5432 check port 8008
    server patroni3 192.168.1.3:5432 check port 8008

# Synchronous Replica - для критичных read запросов
listen postgresql_sync_replica
    bind *:5001
    mode tcp
    balance first
    option httpchk OPTIONS /synchronous
    http-check expect status 200
    default-server inter 3s fall 3 rise 2

    server patroni1 192.168.1.1:5432 check port 8008
    server patroni2 192.168.1.2:5432 check port 8008
    server patroni3 192.168.1.3:5432 check port 8008

# Async Replicas - для обычных read запросов
listen postgresql_async_replica
    bind *:5002
    mode tcp
    balance roundrobin
    option httpchk OPTIONS /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2

    server patroni1 192.168.1.1:5432 check port 8008 weight 10
    server patroni2 192.168.1.2:5432 check port 8008 weight 10
    server patroni3 192.168.1.3:5432 check port 8008 weight 10
```

### Алгоритмы балансировки в HAProxy

```haproxy
# Round Robin (по умолчанию)
balance roundrobin

# Least Connections - на сервер с минимальным количеством соединений
balance leastconn

# Source - привязка клиента к серверу по IP
balance source

# First - первый доступный сервер (для failover)
balance first

# URI - на основе URI (для кэширования)
balance uri

# Веса серверов
server pg-node1 192.168.1.1:5432 weight 100  # Более мощный сервер
server pg-node2 192.168.1.2:5432 weight 50   # Менее мощный сервер
```

## PgBouncer

### Установка PgBouncer

```bash
# Ubuntu/Debian
apt-get install -y pgbouncer

# CentOS/RHEL
yum install -y pgbouncer

# Из исходников
git clone https://github.com/pgbouncer/pgbouncer.git
cd pgbouncer
./configure --prefix=/usr/local
make
make install
```

### Конфигурация PgBouncer

```ini
# /etc/pgbouncer/pgbouncer.ini

[databases]
# Подключение к primary для записи
myapp_rw = host=192.168.1.1 port=5432 dbname=myapp
myapp_rw = host=haproxy_host port=5000 dbname=myapp

# Подключение к репликам для чтения
myapp_ro = host=192.168.1.2,192.168.1.3 port=5432 dbname=myapp
myapp_ro = host=haproxy_host port=5001 dbname=myapp

# Wildcard для всех баз данных
* = host=192.168.1.1 port=5432

[pgbouncer]
# Адрес и порт для прослушивания
listen_addr = 0.0.0.0
listen_port = 6432

# Режим пулинга
# session - держать соединение на время сессии клиента
# transaction - возвращать соединение после каждой транзакции
# statement - возвращать соединение после каждого запроса
pool_mode = transaction

# Размеры пулов
max_client_conn = 1000
default_pool_size = 20
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 5

# Аутентификация
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
auth_user = pgbouncer
auth_query = SELECT usename, passwd FROM pgbouncer.user_search($1)

# Таймауты
server_connect_timeout = 15
server_idle_timeout = 600
server_lifetime = 3600
client_idle_timeout = 0
client_login_timeout = 60

# Логирование
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
stats_period = 60

# Администрирование
admin_users = postgres, admin
stats_users = stats

# DNS
dns_max_ttl = 15

# TLS (опционально)
# server_tls_sslmode = require
# server_tls_ca_file = /etc/ssl/certs/ca.crt
```

### Файл пользователей PgBouncer

```bash
# /etc/pgbouncer/userlist.txt
# Формат: "username" "password_hash"

# Для md5 (устаревший)
"myapp" "md5d41d8cd98f00b204e9800998ecf8427e"

# Для scram-sha-256
"myapp" "SCRAM-SHA-256$4096:salt$StoredKey:ServerKey"

# Получить хэш пароля из PostgreSQL
# SELECT rolname, rolpassword FROM pg_authid WHERE rolname = 'myapp';
```

### Auth Query для PgBouncer

```sql
-- Создать схему и функцию для аутентификации
CREATE SCHEMA pgbouncer;

CREATE OR REPLACE FUNCTION pgbouncer.user_search(p_user TEXT)
RETURNS TABLE(usename name, passwd text) AS $$
BEGIN
    RETURN QUERY
    SELECT
        r.rolname::name,
        r.rolpassword::text
    FROM pg_authid r
    WHERE r.rolname = p_user;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Права
GRANT USAGE ON SCHEMA pgbouncer TO pgbouncer;
GRANT EXECUTE ON FUNCTION pgbouncer.user_search(text) TO pgbouncer;
```

### Команды администрирования PgBouncer

```bash
# Подключение к административной консоли
psql -h localhost -p 6432 -U admin pgbouncer

# Команды в консоли
SHOW POOLS;          # Информация о пулах
SHOW CLIENTS;        # Активные клиенты
SHOW SERVERS;        # Соединения с серверами
SHOW DATABASES;      # Конфигурация баз данных
SHOW STATS;          # Статистика
SHOW CONFIG;         # Конфигурация

PAUSE myapp;         # Приостановить пул
RESUME myapp;        # Возобновить пул
KILL myapp;          # Убить все соединения
RECONNECT myapp;     # Переподключиться

RELOAD;              # Перезагрузить конфигурацию
SHUTDOWN;            # Остановить PgBouncer
```

## Pgpool-II

### Установка Pgpool-II

```bash
# Ubuntu/Debian
apt-get install -y pgpool2

# CentOS/RHEL (из репозитория PostgreSQL)
yum install -y pgpool-II-16

# Из исходников
wget https://www.pgpool.net/mediawiki/images/pgpool-II-4.5.0.tar.gz
tar xzf pgpool-II-4.5.0.tar.gz
cd pgpool-II-4.5.0
./configure --prefix=/usr/local/pgpool
make
make install
```

### Конфигурация Pgpool-II

```bash
# /etc/pgpool-II/pgpool.conf

# Режим работы
backend_clustering_mode = 'streaming_replication'

# Прослушивание
listen_addresses = '*'
port = 9999

# Backend серверы
backend_hostname0 = '192.168.1.1'
backend_port0 = 5432
backend_weight0 = 1
backend_data_directory0 = '/var/lib/postgresql/16/main'
backend_flag0 = 'ALLOW_TO_FAILOVER'
backend_application_name0 = 'server0'

backend_hostname1 = '192.168.1.2'
backend_port1 = 5432
backend_weight1 = 1
backend_data_directory1 = '/var/lib/postgresql/16/main'
backend_flag1 = 'ALLOW_TO_FAILOVER'
backend_application_name1 = 'server1'

backend_hostname2 = '192.168.1.3'
backend_port2 = 5432
backend_weight2 = 1
backend_data_directory2 = '/var/lib/postgresql/16/main'
backend_flag2 = 'ALLOW_TO_FAILOVER'
backend_application_name2 = 'server2'

# Connection pooling
num_init_children = 32
max_pool = 4
child_life_time = 300
child_max_connections = 0
connection_life_time = 0
client_idle_limit = 0

# Аутентификация
enable_pool_hba = on
pool_passwd = 'pool_passwd'
authentication_timeout = 60

# Балансировка нагрузки
load_balance_mode = on
ignore_leading_white_space = on
read_only_function_list = ''
write_function_list = ''
primary_routing_query_pattern_list = ''

# Streaming replication check
sr_check_period = 10
sr_check_user = 'replicator'
sr_check_password = 'replicator_password'
sr_check_database = 'postgres'
delay_threshold = 10000000  # 10MB

# Health check
health_check_period = 10
health_check_timeout = 20
health_check_user = 'pgpool'
health_check_password = 'pgpool_password'
health_check_database = 'postgres'
health_check_max_retries = 3
health_check_retry_delay = 1

# Failover
failover_command = '/etc/pgpool-II/failover.sh %d %h %p %D %m %H %M %P %r %R'
failback_command = ''
follow_primary_command = '/etc/pgpool-II/follow_primary.sh %d %h %p %D %m %H %M %P %r %R'

# Watchdog (для HA самого Pgpool)
use_watchdog = on
wd_hostname = '192.168.1.10'
wd_port = 9000

# Другие watchdog ноды
other_pgpool_hostname0 = '192.168.1.11'
other_pgpool_port0 = 9999
other_wd_port0 = 9000

# Virtual IP
delegate_ip = '192.168.1.100'
if_cmd_path = '/sbin'
if_up_cmd = '/usr/bin/sudo /sbin/ip addr add $_IP_$/24 dev eth0 label eth0:0'
if_down_cmd = '/usr/bin/sudo /sbin/ip addr del $_IP_$/24 dev eth0'
arping_path = '/usr/sbin'
arping_cmd = '/usr/bin/sudo /usr/sbin/arping -U $_IP_$ -w 1 -I eth0'
```

### Скрипт failover для Pgpool-II

```bash
#!/bin/bash
# /etc/pgpool-II/failover.sh

FAILED_NODE_ID=$1
FAILED_HOST=$2
FAILED_PORT=$3
FAILED_DATA_DIR=$4
NEW_PRIMARY_ID=$5
NEW_PRIMARY_HOST=$6
NEW_PRIMARY_PORT=$7
OLD_PRIMARY_NODE_ID=$8
NEW_PRIMARY_DATA_DIR=$9
OLD_PRIMARY_HOST=${10}

PGDATA=/var/lib/postgresql/16/main

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S'): $1" >> /var/log/pgpool/failover.log
}

log "Starting failover"
log "Failed node: $FAILED_NODE_ID ($FAILED_HOST:$FAILED_PORT)"
log "New primary: $NEW_PRIMARY_ID ($NEW_PRIMARY_HOST:$NEW_PRIMARY_PORT)"

if [ "$FAILED_NODE_ID" == "$OLD_PRIMARY_NODE_ID" ]; then
    log "Primary node failed, promoting new primary..."

    # Promote new primary (если используется не Patroni)
    ssh -T postgres@$NEW_PRIMARY_HOST "pg_ctl -D $PGDATA promote"

    if [ $? -eq 0 ]; then
        log "Promotion successful"
    else
        log "Promotion failed"
        exit 1
    fi
fi

exit 0
```

### PCP команды (Pgpool Control)

```bash
# Статус пулов
pcp_pool_status -h localhost -p 9898 -U pgpool

# Информация о нодах
pcp_node_info -h localhost -p 9898 -U pgpool 0

# Подключить ноду
pcp_attach_node -h localhost -p 9898 -U pgpool 1

# Отключить ноду
pcp_detach_node -h localhost -p 9898 -U pgpool 1

# Статус watchdog
pcp_watchdog_info -h localhost -p 9898 -U pgpool
```

## Сравнение решений

| Функция | HAProxy | PgBouncer | Pgpool-II |
|---------|---------|-----------|-----------|
| Connection pooling | Нет | Да | Да |
| Load balancing | Да | Ограниченно | Да |
| Query routing | Нет | Нет | Да |
| Automatic failover | Нет* | Нет | Да |
| Query caching | Нет | Нет | Да |
| SSL termination | Да | Да | Да |
| Health checks | HTTP/TCP | SQL | SQL |
| Resource usage | Низкое | Низкое | Высокое |

*HAProxy определяет недоступность, но не инициирует failover PostgreSQL

## Комбинированные архитектуры

### HAProxy + PgBouncer + Patroni

```
Applications
     │
     ▼
┌─────────────────┐
│    HAProxy      │  ◄── Layer 4 load balancing
│  (VIP: 5432)    │      Health checks via Patroni API
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌────────┐ ┌────────┐
│PgBouncer│ │PgBouncer│  ◄── Connection pooling
│  Node1 │ │  Node2 │      Per-node deployment
└────┬───┘ └────┬───┘
     │         │
     └────┬────┘
          │
    ┌─────┼─────┐
    │     │     │
    ▼     ▼     ▼
┌─────┐ ┌─────┐ ┌─────┐
│ PG1 │ │ PG2 │ │ PG3 │  ◄── PostgreSQL + Patroni
│Patroni│Patroni│Patroni
└─────┘ └─────┘ └─────┘
```

### Конфигурация HAProxy для этой архитектуры

```haproxy
# HAProxy для PgBouncer
listen pgbouncer_primary
    bind *:5432
    mode tcp
    option httpchk OPTIONS /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2

    server pgbouncer1 192.168.1.1:6432 check port 8008
    server pgbouncer2 192.168.1.2:6432 check port 8008
    server pgbouncer3 192.168.1.3:6432 check port 8008

listen pgbouncer_replica
    bind *:5433
    mode tcp
    balance roundrobin
    option httpchk OPTIONS /replica
    http-check expect status 200

    server pgbouncer1 192.168.1.1:6432 check port 8008
    server pgbouncer2 192.168.1.2:6432 check port 8008
    server pgbouncer3 192.168.1.3:6432 check port 8008
```

### PgBouncer на каждом узле

```ini
# /etc/pgbouncer/pgbouncer.ini на каждом узле

[databases]
# Локальный PostgreSQL
* = host=127.0.0.1 port=5432

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
pool_mode = transaction
max_client_conn = 500
default_pool_size = 50
```

## Read/Write Split на уровне приложения

### Python с SQLAlchemy

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

class DatabaseRouter:
    def __init__(self):
        # Подключение для записи (primary)
        self.write_engine = create_engine(
            'postgresql://user:pass@haproxy:5000/myapp',
            pool_size=10,
            max_overflow=20
        )

        # Подключение для чтения (replicas)
        self.read_engine = create_engine(
            'postgresql://user:pass@haproxy:5001/myapp',
            pool_size=20,
            max_overflow=40
        )

        self.WriteSession = sessionmaker(bind=self.write_engine)
        self.ReadSession = sessionmaker(bind=self.read_engine)

    def get_write_session(self):
        return self.WriteSession()

    def get_read_session(self):
        return self.ReadSession()


# Использование
db = DatabaseRouter()

# Для записи
with db.get_write_session() as session:
    user = User(name='John')
    session.add(user)
    session.commit()

# Для чтения
with db.get_read_session() as session:
    users = session.query(User).all()
```

### Django с django-read-replica

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'myapp',
        'HOST': 'haproxy_host',
        'PORT': '5000',  # Primary
        'USER': 'myapp',
        'PASSWORD': 'password',
    },
    'replica': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'myapp',
        'HOST': 'haproxy_host',
        'PORT': '5001',  # Replicas
        'USER': 'myapp',
        'PASSWORD': 'password',
    }
}

DATABASE_ROUTERS = ['myapp.routers.ReadReplicaRouter']

# routers.py
class ReadReplicaRouter:
    def db_for_read(self, model, **hints):
        return 'replica'

    def db_for_write(self, model, **hints):
        return 'default'

    def allow_relation(self, obj1, obj2, **hints):
        return True

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        return db == 'default'
```

### Go с database/sql

```go
package main

import (
    "database/sql"
    "sync"

    _ "github.com/lib/pq"
)

type DBCluster struct {
    Primary  *sql.DB
    Replicas []*sql.DB
    mu       sync.Mutex
    next     int
}

func NewDBCluster(primaryDSN string, replicaDSNs []string) (*DBCluster, error) {
    primary, err := sql.Open("postgres", primaryDSN)
    if err != nil {
        return nil, err
    }
    primary.SetMaxOpenConns(25)
    primary.SetMaxIdleConns(5)

    var replicas []*sql.DB
    for _, dsn := range replicaDSNs {
        replica, err := sql.Open("postgres", dsn)
        if err != nil {
            return nil, err
        }
        replica.SetMaxOpenConns(50)
        replica.SetMaxIdleConns(10)
        replicas = append(replicas, replica)
    }

    return &DBCluster{
        Primary:  primary,
        Replicas: replicas,
    }, nil
}

func (c *DBCluster) Write() *sql.DB {
    return c.Primary
}

func (c *DBCluster) Read() *sql.DB {
    c.mu.Lock()
    defer c.mu.Unlock()

    if len(c.Replicas) == 0 {
        return c.Primary
    }

    replica := c.Replicas[c.next]
    c.next = (c.next + 1) % len(c.Replicas)
    return replica
}

// Использование
func main() {
    cluster, _ := NewDBCluster(
        "postgres://user:pass@haproxy:5000/myapp",
        []string{
            "postgres://user:pass@haproxy:5001/myapp",
        },
    )

    // Для записи
    cluster.Write().Exec("INSERT INTO users (name) VALUES ($1)", "John")

    // Для чтения
    rows, _ := cluster.Read().Query("SELECT * FROM users")
    defer rows.Close()
}
```

## Мониторинг балансировки нагрузки

### HAProxy статистика

```bash
# Получить статистику через сокет
echo "show stat" | socat unix-connect:/run/haproxy/admin.sock stdio

# Prometheus exporter для HAProxy
docker run -d \
    --name haproxy_exporter \
    -p 9101:9101 \
    quay.io/prometheus/haproxy-exporter \
    --haproxy.scrape-uri="http://admin:admin@haproxy:7000/stats;csv"
```

### PgBouncer статистика

```sql
-- В консоли PgBouncer
SHOW POOLS;
SHOW STATS;
SHOW CLIENTS;
SHOW SERVERS;

-- Prometheus exporter для PgBouncer
-- pgbouncer_exporter
docker run -d \
    --name pgbouncer_exporter \
    -p 9127:9127 \
    -e DATABASE_URL="postgres://pgbouncer@localhost:6432/pgbouncer" \
    spreaker/prometheus-pgbouncer-exporter
```

## Best Practices

### 1. Используйте отдельные порты для read/write

```
Port 5000 - Primary (read/write)
Port 5001 - Replicas (read-only)
Port 5002 - All nodes (read from any)
```

### 2. Настройте правильные таймауты

```haproxy
defaults
    timeout connect 5s     # Время на установку соединения
    timeout client 30m     # Таймаут клиента (для долгих запросов)
    timeout server 30m     # Таймаут сервера
    timeout check 5s       # Таймаут health check
```

### 3. Используйте connection pooling

```ini
# PgBouncer
pool_mode = transaction  # Рекомендуется для большинства случаев
default_pool_size = 20   # Размер пула на базу/пользователя
```

### 4. Мониторьте распределение нагрузки

```bash
# Проверка распределения соединений
SELECT
    client_addr,
    count(*) as connections
FROM pg_stat_activity
GROUP BY client_addr;
```

## Заключение

Балансировка нагрузки в PostgreSQL кластерах обеспечивает:
- Высокую доступность приложений
- Масштабирование чтения
- Эффективное использование ресурсов
- Изоляцию отказов

Рекомендуемые архитектуры:
- **Простая**: HAProxy + PostgreSQL с Patroni
- **С пулингом**: HAProxy + PgBouncer + Patroni
- **С маршрутизацией запросов**: Pgpool-II (для специфических случаев)

Выбор решения зависит от требований к производительности, сложности инфраструктуры и необходимости connection pooling.

---

[prev: 05-monitoring](./05-monitoring.md) | [next: 01-pg_dump](../13-backup-recovery/01-pg_dump.md)
