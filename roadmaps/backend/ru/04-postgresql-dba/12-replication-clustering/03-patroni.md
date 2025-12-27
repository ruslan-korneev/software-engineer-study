# Patroni для High Availability кластеров PostgreSQL

## Введение

Patroni — это open-source решение для автоматического управления высокодоступными кластерами PostgreSQL. Написан на Python, разработан компанией Zalando. Patroni обеспечивает автоматический failover, управление репликацией и интеграцию с распределенными системами хранения конфигурации (DCS).

## Архитектура Patroni

### Общая схема

```
                    ┌─────────────────────────────────────────┐
                    │     Distributed Configuration Store     │
                    │       (etcd / Consul / ZooKeeper)       │
                    └────────────────┬────────────────────────┘
                                     │
           ┌─────────────────────────┼─────────────────────────┐
           │                         │                         │
           ▼                         ▼                         ▼
    ┌──────────────┐          ┌──────────────┐          ┌──────────────┐
    │   Node 1     │          │   Node 2     │          │   Node 3     │
    │ ┌──────────┐ │          │ ┌──────────┐ │          │ ┌──────────┐ │
    │ │ Patroni  │ │          │ │ Patroni  │ │          │ │ Patroni  │ │
    │ └────┬─────┘ │          │ └────┬─────┘ │          │ └────┬─────┘ │
    │      │       │          │      │       │          │      │       │
    │ ┌────▼─────┐ │          │ ┌────▼─────┐ │          │ ┌────▼─────┐ │
    │ │PostgreSQL│ │◄────────►│ │PostgreSQL│ │◄────────►│ │PostgreSQL│ │
    │ │ (Leader) │ │ Streaming│ │(Replica) │ │Streaming │ │(Replica) │ │
    │ └──────────┘ │ Repl.    │ └──────────┘ │ Repl.    │ └──────────┘ │
    └──────────────┘          └──────────────┘          └──────────────┘
           │
           ▼
    ┌──────────────┐
    │   HAProxy/   │
    │   PgBouncer  │
    └──────────────┘
           │
           ▼
    ┌──────────────┐
    │ Application  │
    └──────────────┘
```

### Основные компоненты

1. **Patroni daemon** — управляет PostgreSQL на каждом узле
2. **DCS (Distributed Configuration Store)** — хранит состояние кластера и leader lock
3. **PostgreSQL** — база данных
4. **REST API** — интерфейс для управления и мониторинга
5. **Watchdog** — hardware/software watchdog для предотвращения split-brain

## Установка Patroni

### Установка зависимостей

```bash
# Ubuntu/Debian
apt-get update
apt-get install -y python3 python3-pip python3-dev libpq-dev

# CentOS/RHEL
yum install -y python3 python3-pip python3-devel libpq-devel

# Установка PostgreSQL
apt-get install -y postgresql-16 postgresql-client-16
```

### Установка Patroni

```bash
# Установка через pip
pip3 install patroni[etcd]

# Или с поддержкой Consul
pip3 install patroni[consul]

# Или с поддержкой ZooKeeper
pip3 install patroni[zookeeper]

# Полная установка со всеми провайдерами
pip3 install patroni[etcd,consul,zookeeper,kubernetes]
```

### Установка etcd (DCS)

```bash
# Ubuntu/Debian
apt-get install -y etcd

# Или скачать бинарник
ETCD_VER=v3.5.9
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o etcd.tar.gz
tar xzf etcd.tar.gz
mv etcd-${ETCD_VER}-linux-amd64/etcd* /usr/local/bin/

# Настройка etcd кластера
cat > /etc/etcd/etcd.conf.yml << 'EOF'
name: 'etcd1'
data-dir: /var/lib/etcd
initial-cluster-state: 'new'
initial-cluster-token: 'etcd-cluster-1'
initial-cluster: 'etcd1=http://192.168.1.1:2380,etcd2=http://192.168.1.2:2380,etcd3=http://192.168.1.3:2380'
initial-advertise-peer-urls: 'http://192.168.1.1:2380'
advertise-client-urls: 'http://192.168.1.1:2379'
listen-peer-urls: 'http://192.168.1.1:2380'
listen-client-urls: 'http://192.168.1.1:2379,http://127.0.0.1:2379'
EOF
```

## Конфигурация Patroni

### Базовый конфигурационный файл

```yaml
# /etc/patroni/patroni.yml

scope: postgres-cluster    # Имя кластера
namespace: /service/       # Namespace в DCS
name: node1                # Уникальное имя узла

# REST API
restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.1.1:8008
  # Аутентификация (опционально)
  # authentication:
  #   username: admin
  #   password: admin

# Distributed Configuration Store
etcd:
  hosts:
    - 192.168.1.1:2379
    - 192.168.1.2:2379
    - 192.168.1.3:2379
  # Или использовать SRV записи
  # srv: etcd.example.com

# Bootstrap конфигурация (для инициализации нового кластера)
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576  # 1MB
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: on
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: on
        hot_standby_feedback: on
        wal_keep_size: 1GB

  # Инициализация кластера
  initdb:
    - encoding: UTF8
    - data-checksums
    - locale: en_US.UTF-8

  # Пользователи, создаваемые при инициализации
  pg_hba:
    - host replication replicator 192.168.1.0/24 scram-sha-256
    - host all all 192.168.1.0/24 scram-sha-256
    - host all all 0.0.0.0/0 scram-sha-256

  users:
    admin:
      password: admin_password
      options:
        - createrole
        - createdb
    replicator:
      password: replicator_password
      options:
        - replication

# Конфигурация PostgreSQL
postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.1.1:5432
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin
  pgpass: /var/lib/postgresql/.pgpass

  # Аутентификация
  authentication:
    replication:
      username: replicator
      password: replicator_password
    superuser:
      username: postgres
      password: postgres_password
    rewind:
      username: postgres
      password: postgres_password

  # Параметры PostgreSQL
  parameters:
    max_connections: 200
    shared_buffers: 2GB
    effective_cache_size: 6GB
    maintenance_work_mem: 512MB
    work_mem: 32MB
    max_wal_size: 4GB
    min_wal_size: 1GB
    checkpoint_completion_target: 0.9
    random_page_cost: 1.1
    effective_io_concurrency: 200
    log_destination: stderr
    logging_collector: on
    log_directory: /var/log/postgresql
    log_filename: postgresql-%Y-%m-%d.log

# Watchdog
watchdog:
  mode: automatic  # off, automatic, required
  device: /dev/watchdog
  safety_margin: 5

# Tags
tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

### Конфигурация для разных узлов

```yaml
# /etc/patroni/patroni.yml на node2
scope: postgres-cluster
namespace: /service/
name: node2  # Уникальное имя

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.1.2:8008  # IP этого узла

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.1.2:5432  # IP этого узла
  data_dir: /var/lib/postgresql/16/main
  # Остальные параметры такие же
```

## Запуск и управление кластером

### Systemd unit файл

```ini
# /etc/systemd/system/patroni.service
[Unit]
Description=Patroni - PostgreSQL HA
After=syslog.target network.target etcd.service

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
TimeoutSec=30
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Запуск кластера

```bash
# На первом узле
systemctl enable patroni
systemctl start patroni

# Проверка статуса
systemctl status patroni
journalctl -u patroni -f

# На остальных узлах после запуска первого
systemctl enable patroni
systemctl start patroni
```

### Команды patronictl

```bash
# Просмотр состояния кластера
patronictl -c /etc/patroni/patroni.yml list

# Пример вывода:
# + Cluster: postgres-cluster (7123456789012345678) ------+----+-----------+
# | Member |    Host       | Role    | State   | TL | Lag in MB  |
# +--------+---------------+---------+---------+----+------------+
# | node1  | 192.168.1.1   | Leader  | running |  1 |            |
# | node2  | 192.168.1.2   | Replica | running |  1 |          0 |
# | node3  | 192.168.1.3   | Replica | running |  1 |          0 |
# +--------+---------------+---------+---------+----+------------+

# Ручной switchover (плановое переключение)
patronictl -c /etc/patroni/patroni.yml switchover
# Интерактивный режим с подтверждением

# Switchover с указанием параметров
patronictl -c /etc/patroni/patroni.yml switchover \
    --master node1 --candidate node2 --force

# Ручной failover (принудительный)
patronictl -c /etc/patroni/patroni.yml failover

# Перезапуск PostgreSQL на узле
patronictl -c /etc/patroni/patroni.yml restart postgres-cluster node1

# Перезагрузка конфигурации
patronictl -c /etc/patroni/patroni.yml reload postgres-cluster

# Изменение динамической конфигурации
patronictl -c /etc/patroni/patroni.yml edit-config

# Просмотр конфигурации
patronictl -c /etc/patroni/patroni.yml show-config

# Пауза автоматического failover
patronictl -c /etc/patroni/patroni.yml pause

# Возобновление автоматического failover
patronictl -c /etc/patroni/patroni.yml resume

# Reinitialize узла (пересоздать из резервной копии)
patronictl -c /etc/patroni/patroni.yml reinit postgres-cluster node3
```

## REST API

### Эндпоинты для мониторинга

```bash
# Проверка статуса узла
curl http://192.168.1.1:8008/

# Ответ лидера (HTTP 200):
# {
#   "state": "running",
#   "postmaster_start_time": "2024-01-15 10:00:00.000 UTC",
#   "role": "master",
#   "server_version": 160001,
#   "cluster_unlocked": false,
#   "xlog": {"location": 100663296},
#   "timeline": 1,
#   "patroni": {"version": "3.0.0", "scope": "postgres-cluster"}
# }

# Ответ реплики (HTTP 200):
# {
#   "state": "running",
#   "role": "replica",
#   "xlog": {"received_location": 100663296, "replayed_location": 100663296}
# }

# Проверка, является ли узел лидером
curl -s http://192.168.1.1:8008/leader
# HTTP 200 - лидер, HTTP 503 - не лидер

# Проверка, является ли узел репликой
curl -s http://192.168.1.1:8008/replica
# HTTP 200 - реплика, HTTP 503 - не реплика

# Проверка для read-only запросов
curl -s http://192.168.1.1:8008/read-only
# HTTP 200 - доступен для чтения (лидер или реплика)

# Проверка здоровья кластера
curl http://192.168.1.1:8008/cluster
```

### Эндпоинты для HAProxy

```bash
# Проверка лидера (для балансировки записи)
# /master или /leader или /primary - HTTP 200 только для лидера

# Проверка реплики (для балансировки чтения)
# /replica - HTTP 200 только для реплик
# /read-only - HTTP 200 для лидера и реплик

# Проверка синхронной реплики
# /synchronous - HTTP 200 только для синхронной реплики

# Проверка асинхронной реплики
# /asynchronous - HTTP 200 только для асинхронных реплик
```

## Интеграция с HAProxy

### Конфигурация HAProxy

```haproxy
# /etc/haproxy/haproxy.cfg

global
    log /dev/log local0
    maxconn 1000
    user haproxy
    group haproxy

defaults
    log global
    mode tcp
    retries 3
    timeout connect 10s
    timeout client 30m
    timeout server 30m
    timeout check 5s

# Статистика HAProxy
listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

# PostgreSQL master (read-write)
listen postgresql-master
    bind *:5000
    option httpchk OPTIONS /master
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions

    server node1 192.168.1.1:5432 check port 8008
    server node2 192.168.1.2:5432 check port 8008
    server node3 192.168.1.3:5432 check port 8008

# PostgreSQL replicas (read-only)
listen postgresql-replica
    bind *:5001
    option httpchk OPTIONS /replica
    http-check expect status 200
    balance roundrobin
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions

    server node1 192.168.1.1:5432 check port 8008
    server node2 192.168.1.2:5432 check port 8008
    server node3 192.168.1.3:5432 check port 8008

# PostgreSQL all (для чтения с любого узла)
listen postgresql-read
    bind *:5002
    option httpchk OPTIONS /read-only
    http-check expect status 200
    balance roundrobin
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions

    server node1 192.168.1.1:5432 check port 8008
    server node2 192.168.1.2:5432 check port 8008
    server node3 192.168.1.3:5432 check port 8008
```

## Синхронная репликация в Patroni

### Настройка синхронной репликации

```yaml
# В bootstrap.dcs или через patronictl edit-config
bootstrap:
  dcs:
    synchronous_mode: true           # Включить синхронную репликацию
    synchronous_mode_strict: false   # Если true, запись блокируется без синхронных реплик
    synchronous_node_count: 1        # Количество синхронных реплик

    postgresql:
      parameters:
        synchronous_commit: on
```

### Управление синхронными репликами

```bash
# Проверка синхронного режима
patronictl -c /etc/patroni/patroni.yml show-config | grep synchronous

# Включить/выключить синхронный режим
patronictl -c /etc/patroni/patroni.yml edit-config
# Изменить synchronous_mode: true/false

# Проверка какой узел синхронный
SELECT application_name, sync_state
FROM pg_stat_replication;
```

## Callbacks и скрипты

### Настройка callbacks

```yaml
# patroni.yml
postgresql:
  callbacks:
    on_start: /scripts/on_start.sh
    on_stop: /scripts/on_stop.sh
    on_restart: /scripts/on_restart.sh
    on_reload: /scripts/on_reload.sh
    on_role_change: /scripts/on_role_change.sh
```

### Пример callback скрипта

```bash
#!/bin/bash
# /scripts/on_role_change.sh

ACTION=$1  # on_role_change
ROLE=$2    # master, replica, или standby_leader

case $ROLE in
    master)
        echo "$(date): Node became master" >> /var/log/patroni/callbacks.log
        # Например, обновить DNS запись
        # curl -X POST "https://api.cloudflare.com/..."
        ;;
    replica)
        echo "$(date): Node became replica" >> /var/log/patroni/callbacks.log
        ;;
    *)
        echo "$(date): Unknown role: $ROLE" >> /var/log/patroni/callbacks.log
        ;;
esac
```

## Резервное копирование с Patroni

### Интеграция с pgBackRest

```yaml
# patroni.yml
postgresql:
  create_replica_methods:
    - pgbackrest
    - basebackup

  pgbackrest:
    command: /usr/bin/pgbackrest --stanza=main --delta restore
    keep_data: true
    no_params: true

  recovery_conf:
    restore_command: pgbackrest --stanza=main archive-get %f %p
```

### Интеграция с WAL-G

```yaml
# patroni.yml
postgresql:
  create_replica_methods:
    - walg
    - basebackup

  walg:
    command: /scripts/walg_restore.sh
    keep_data: true
    no_params: true

  parameters:
    archive_command: wal-g wal-push %p
    archive_mode: on

  recovery_conf:
    restore_command: wal-g wal-fetch %f %p
```

## Watchdog

### Настройка hardware watchdog

```bash
# Загрузить модуль watchdog
modprobe softdog
echo "softdog" >> /etc/modules-load.d/softdog.conf

# Установить права
chown postgres:postgres /dev/watchdog

# Или использовать udev правило
cat > /etc/udev/rules.d/99-watchdog.rules << 'EOF'
KERNEL=="watchdog", OWNER="postgres", GROUP="postgres"
EOF
```

```yaml
# patroni.yml
watchdog:
  mode: automatic  # off | automatic | required
  device: /dev/watchdog
  safety_margin: 5
```

## Мониторинг Patroni

### Prometheus метрики

Patroni экспортирует метрики в формате Prometheus:

```bash
# Получить метрики
curl http://192.168.1.1:8008/metrics

# Примеры метрик:
# patroni_version{version="3.0.0"} 1
# patroni_running 1
# patroni_postmaster_running 1
# patroni_postgres_running 1
# patroni_master 1
# patroni_xlog_location 100663296
# patroni_xlog_received_location 100663296
# patroni_xlog_replayed_location 100663296
# patroni_replication_slots{slot_name="node2"} 1
```

### Grafana Dashboard

```json
{
  "title": "Patroni Cluster",
  "panels": [
    {
      "title": "Cluster State",
      "targets": [
        {"expr": "patroni_master", "legendFormat": "{{instance}}"}
      ]
    },
    {
      "title": "Replication Lag",
      "targets": [
        {"expr": "patroni_xlog_location - patroni_xlog_replayed_location", "legendFormat": "{{instance}}"}
      ]
    }
  ]
}
```

## Типичные проблемы и решения

### Проблема: Split-brain

```bash
# Причина: DCS недоступен, несколько узлов считают себя лидером
# Решение 1: Настроить watchdog
watchdog:
  mode: required

# Решение 2: Увеличить кворум etcd
# Используйте минимум 3 узла etcd
```

### Проблема: Долгий failover

```yaml
# Уменьшить TTL и loop_wait
bootstrap:
  dcs:
    ttl: 20          # Время жизни лидер-лока
    loop_wait: 5     # Интервал проверки
    retry_timeout: 5 # Таймаут повторных попыток
```

### Проблема: Реплика не может догнать лидера

```bash
# Reinitialize реплику
patronictl -c /etc/patroni/patroni.yml reinit postgres-cluster node3

# Или через REST API
curl -XPOST http://192.168.1.3:8008/reinitialize
```

### Проблема: pg_rewind не работает

```bash
# Убедитесь, что включены checksums или wal_log_hints
# При initdb:
initdb --data-checksums

# Или в postgresql.conf:
wal_log_hints = on
```

## Best Practices

### 1. Всегда используйте нечетное количество узлов DCS (etcd)

```
3 узла etcd = кворум 2, выдерживает потерю 1 узла
5 узлов etcd = кворум 3, выдерживает потерю 2 узлов
```

### 2. Разделяйте DCS и PostgreSQL на разные серверы

```
Для production:
- 3 узла etcd на отдельных серверах
- 3+ узла PostgreSQL/Patroni
```

### 3. Настройте мониторинг и алерты

```yaml
# Критические метрики:
- patroni_master == 0 для всех узлов  # Нет лидера
- patroni_running == 0                # Patroni не запущен
- replication_lag > threshold         # Задержка репликации
```

### 4. Регулярно тестируйте failover

```bash
# Тестовое переключение
patronictl -c /etc/patroni/patroni.yml switchover --force
```

## Заключение

Patroni обеспечивает:
- Автоматический failover за секунды
- Простое управление кластером
- Интеграцию с HAProxy и PgBouncer
- Поддержку синхронной репликации
- REST API для мониторинга и управления

Ключевые рекомендации:
- Используйте watchdog для предотвращения split-brain
- Настройте минимум 3 узла etcd
- Регулярно тестируйте процедуры failover
- Мониторьте задержку репликации и состояние кластера
