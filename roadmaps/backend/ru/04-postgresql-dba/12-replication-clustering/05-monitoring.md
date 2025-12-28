# Мониторинг репликации PostgreSQL

[prev: 04-kubernetes-deployment](./04-kubernetes-deployment.md) | [next: 06-load-balancing](./06-load-balancing.md)

---

## Введение

Мониторинг репликации — критически важный аспект администрирования PostgreSQL кластеров. Эффективный мониторинг позволяет своевременно обнаруживать проблемы, предотвращать потерю данных и обеспечивать высокую доступность.

## Ключевые метрики репликации

### Основные метрики для мониторинга

```
┌──────────────────────────────────────────────────────────────────┐
│                    Метрики репликации PostgreSQL                  │
├──────────────────────────────────────────────────────────────────┤
│ Replication Lag        │ Задержка между primary и standby        │
│ WAL Generation Rate    │ Скорость генерации WAL                  │
│ WAL Send/Receive Rate  │ Скорость отправки/получения WAL         │
│ Replication Slot State │ Состояние слотов репликации             │
│ Connection Status      │ Статус подключения репликации           │
│ Timeline ID            │ Идентификатор timeline (split-brain)    │
│ Sync State             │ Состояние синхронизации (sync/async)    │
└──────────────────────────────────────────────────────────────────┘
```

## Встроенные средства мониторинга PostgreSQL

### pg_stat_replication (на Primary)

```sql
-- Базовый запрос для просмотра репликации
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    client_hostname,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    sync_state,
    sync_priority
FROM pg_stat_replication;

-- Расширенный запрос с расчетом задержки
SELECT
    application_name AS standby_name,
    client_addr,
    state,
    sync_state,
    -- Задержка в байтах
    pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS pending_bytes,
    pg_wal_lsn_diff(sent_lsn, write_lsn) AS write_lag_bytes,
    pg_wal_lsn_diff(write_lsn, flush_lsn) AS flush_lag_bytes,
    pg_wal_lsn_diff(flush_lsn, replay_lsn) AS replay_lag_bytes,
    -- Общая задержка
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS total_lag_bytes,
    -- Читаемый формат
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS total_lag_pretty,
    -- Время последнего сообщения
    reply_time,
    now() - reply_time AS time_since_reply
FROM pg_stat_replication;
```

### pg_stat_wal_receiver (на Standby)

```sql
-- Статус WAL receiver на реплике
SELECT
    pid,
    status,
    receive_start_lsn,
    receive_start_tli,
    received_lsn,
    received_tli,
    last_msg_send_time,
    last_msg_receipt_time,
    latest_end_lsn,
    latest_end_time,
    slot_name,
    sender_host,
    sender_port,
    conninfo
FROM pg_stat_wal_receiver;

-- Расчет задержки на реплике
SELECT
    CASE
        WHEN pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn() THEN 0
        ELSE EXTRACT(EPOCH FROM now() - pg_last_xact_replay_timestamp())
    END AS replication_lag_seconds;
```

### pg_replication_slots

```sql
-- Состояние слотов репликации
SELECT
    slot_name,
    plugin,
    slot_type,
    datoid,
    database,
    active,
    active_pid,
    xmin,
    catalog_xmin,
    restart_lsn,
    confirmed_flush_lsn,
    -- Размер данных, удерживаемых слотом
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS retained_bytes,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_pretty
FROM pg_replication_slots
ORDER BY pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) DESC;
```

### Мониторинг WAL

```sql
-- Размер WAL директории
SELECT
    pg_size_pretty(SUM(size)) AS total_wal_size,
    COUNT(*) AS wal_files_count
FROM pg_ls_waldir();

-- Текущая позиция WAL
SELECT
    pg_current_wal_lsn() AS current_lsn,
    pg_current_wal_insert_lsn() AS insert_lsn,
    pg_walfile_name(pg_current_wal_lsn()) AS current_wal_file;

-- Скорость генерации WAL
WITH wal_stats AS (
    SELECT
        pg_current_wal_lsn() AS current_lsn,
        pg_stat_get_wal(NULL) AS stats
)
SELECT
    (stats).wal_records AS total_records,
    (stats).wal_bytes AS total_bytes,
    pg_size_pretty((stats).wal_bytes) AS total_size,
    (stats).wal_buffers_full AS buffers_full
FROM wal_stats;
```

## Создание представлений для мониторинга

### View для обзора репликации

```sql
-- Создать представление для мониторинга
CREATE OR REPLACE VIEW replication_status AS
SELECT
    pid,
    application_name,
    client_addr,
    state,
    sync_state,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS lag,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    reply_time,
    EXTRACT(EPOCH FROM (now() - reply_time)) AS seconds_since_reply
FROM pg_stat_replication;

-- Использование
SELECT * FROM replication_status;
```

### Функция для алертов

```sql
-- Функция проверки здоровья репликации
CREATE OR REPLACE FUNCTION check_replication_health(
    max_lag_bytes BIGINT DEFAULT 1048576,  -- 1MB
    max_lag_seconds INTEGER DEFAULT 60
)
RETURNS TABLE (
    standby_name TEXT,
    status TEXT,
    lag_bytes BIGINT,
    lag_seconds NUMERIC,
    message TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        r.application_name::TEXT,
        CASE
            WHEN r.state != 'streaming' THEN 'CRITICAL'
            WHEN pg_wal_lsn_diff(pg_current_wal_lsn(), r.replay_lsn) > max_lag_bytes THEN 'WARNING'
            WHEN EXTRACT(EPOCH FROM (now() - r.reply_time)) > max_lag_seconds THEN 'WARNING'
            ELSE 'OK'
        END,
        pg_wal_lsn_diff(pg_current_wal_lsn(), r.replay_lsn)::BIGINT,
        ROUND(EXTRACT(EPOCH FROM (now() - r.reply_time)), 2),
        CASE
            WHEN r.state != 'streaming' THEN 'Replication is not streaming'
            WHEN pg_wal_lsn_diff(pg_current_wal_lsn(), r.replay_lsn) > max_lag_bytes
                THEN 'Lag exceeds ' || pg_size_pretty(max_lag_bytes)
            WHEN EXTRACT(EPOCH FROM (now() - r.reply_time)) > max_lag_seconds
                THEN 'No response for ' || max_lag_seconds || ' seconds'
            ELSE 'Replication is healthy'
        END
    FROM pg_stat_replication r;
END;
$$ LANGUAGE plpgsql;

-- Использование
SELECT * FROM check_replication_health(5242880, 30);  -- 5MB, 30 секунд
```

## Prometheus и postgres_exporter

### Установка postgres_exporter

```bash
# Docker
docker run -d \
    --name postgres_exporter \
    -e DATA_SOURCE_NAME="postgresql://postgres:password@localhost:5432/postgres?sslmode=disable" \
    -p 9187:9187 \
    quay.io/prometheuscommunity/postgres-exporter

# Binary
wget https://github.com/prometheus-community/postgres_exporter/releases/download/v0.15.0/postgres_exporter-0.15.0.linux-amd64.tar.gz
tar xzf postgres_exporter-0.15.0.linux-amd64.tar.gz
./postgres_exporter --web.listen-address=":9187"
```

### Конфигурация postgres_exporter

```yaml
# queries.yaml - кастомные запросы для мониторинга репликации
pg_replication:
  query: |
    SELECT
      CASE WHEN NOT pg_is_in_recovery() THEN 0 ELSE 1 END AS is_replica,
      CASE WHEN NOT pg_is_in_recovery() THEN 0 ELSE
        GREATEST(0, EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())))
      END AS lag_seconds
  master: true
  metrics:
    - is_replica:
        usage: "GAUGE"
        description: "Is this a replica (1) or primary (0)"
    - lag_seconds:
        usage: "GAUGE"
        description: "Replication lag in seconds"

pg_replication_slots:
  query: |
    SELECT
      slot_name,
      slot_type,
      active::int as active,
      pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) as lag_bytes
    FROM pg_replication_slots
  master: true
  metrics:
    - slot_name:
        usage: "LABEL"
        description: "Slot name"
    - slot_type:
        usage: "LABEL"
        description: "Slot type"
    - active:
        usage: "GAUGE"
        description: "Is slot active"
    - lag_bytes:
        usage: "GAUGE"
        description: "Replication lag in bytes"

pg_stat_replication:
  query: |
    SELECT
      application_name,
      client_addr::text,
      state,
      sync_state,
      pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) as pending_bytes,
      pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) as lag_bytes,
      EXTRACT(EPOCH FROM (now() - reply_time)) as reply_lag_seconds
    FROM pg_stat_replication
  master: true
  metrics:
    - application_name:
        usage: "LABEL"
        description: "Application name"
    - client_addr:
        usage: "LABEL"
        description: "Client address"
    - state:
        usage: "LABEL"
        description: "Replication state"
    - sync_state:
        usage: "LABEL"
        description: "Synchronization state"
    - pending_bytes:
        usage: "GAUGE"
        description: "Bytes pending to send"
    - lag_bytes:
        usage: "GAUGE"
        description: "Total replication lag in bytes"
    - reply_lag_seconds:
        usage: "GAUGE"
        description: "Time since last standby reply"
```

### Prometheus конфигурация

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'postgres'
    static_configs:
      - targets:
        - 'primary:9187'
        - 'standby1:9187'
        - 'standby2:9187'
    relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):\d+'
        target_label: instance
```

### Prometheus Alerting Rules

```yaml
# alerts.yml
groups:
  - name: postgresql_replication
    rules:
      # Критическая задержка репликации
      - alert: PostgreSQLReplicationLagCritical
        expr: pg_replication_lag_seconds > 300
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL replication lag is critical"
          description: "Replication lag on {{ $labels.instance }} is {{ $value }} seconds"

      # Предупреждение о задержке репликации
      - alert: PostgreSQLReplicationLagWarning
        expr: pg_replication_lag_seconds > 60
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL replication lag is high"
          description: "Replication lag on {{ $labels.instance }} is {{ $value }} seconds"

      # Репликация остановлена
      - alert: PostgreSQLReplicationStopped
        expr: pg_stat_replication_pg_wal_lsn_diff == 0 AND pg_stat_replication_state != "streaming"
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL replication has stopped"
          description: "Replication from {{ $labels.application_name }} is not streaming"

      # Слот репликации неактивен
      - alert: PostgreSQLReplicationSlotInactive
        expr: pg_replication_slots_active == 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL replication slot is inactive"
          description: "Replication slot {{ $labels.slot_name }} is inactive"

      # Слот репликации накапливает WAL
      - alert: PostgreSQLReplicationSlotLagHigh
        expr: pg_replication_slots_lag_bytes > 5368709120  # 5GB
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL replication slot WAL retention is high"
          description: "Slot {{ $labels.slot_name }} is retaining {{ humanize $value }} of WAL"

      # Нет реплик
      - alert: PostgreSQLNoReplicas
        expr: pg_stat_replication_count == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL has no connected replicas"
          description: "Primary {{ $labels.instance }} has no connected standbys"

      # Синхронная репликация нарушена
      - alert: PostgreSQLSyncReplicationBroken
        expr: pg_settings_synchronous_commit == 1 AND count(pg_stat_replication_sync_state{sync_state="sync"}) == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL synchronous replication is broken"
          description: "No synchronous standbys available on {{ $labels.instance }}"
```

## Grafana Dashboards

### Dashboard JSON (основные панели)

```json
{
  "title": "PostgreSQL Replication",
  "panels": [
    {
      "title": "Replication Lag (seconds)",
      "type": "graph",
      "targets": [
        {
          "expr": "pg_replication_lag_seconds",
          "legendFormat": "{{ instance }}"
        }
      ],
      "yAxes": [{"format": "s"}]
    },
    {
      "title": "Replication Lag (bytes)",
      "type": "graph",
      "targets": [
        {
          "expr": "pg_stat_replication_pg_wal_lsn_diff",
          "legendFormat": "{{ application_name }}"
        }
      ],
      "yAxes": [{"format": "bytes"}]
    },
    {
      "title": "Replication State",
      "type": "stat",
      "targets": [
        {
          "expr": "pg_stat_replication_state",
          "legendFormat": "{{ application_name }}: {{ state }}"
        }
      ]
    },
    {
      "title": "WAL Generation Rate",
      "type": "graph",
      "targets": [
        {
          "expr": "rate(pg_stat_wal_bytes_total[5m])",
          "legendFormat": "{{ instance }}"
        }
      ],
      "yAxes": [{"format": "Bps"}]
    },
    {
      "title": "Replication Slots",
      "type": "table",
      "targets": [
        {
          "expr": "pg_replication_slots_active",
          "format": "table"
        }
      ]
    }
  ]
}
```

## pg_stat_monitor (расширенный мониторинг)

```sql
-- Установка pg_stat_monitor
CREATE EXTENSION pg_stat_monitor;

-- Конфигурация
ALTER SYSTEM SET pg_stat_monitor.pgsm_enable_query_plan = on;
ALTER SYSTEM SET pg_stat_monitor.pgsm_normalized_query = on;
SELECT pg_reload_conf();

-- Просмотр статистики запросов
SELECT
    bucket_start_time,
    queryid,
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    rows
FROM pg_stat_monitor
ORDER BY total_exec_time DESC
LIMIT 20;
```

## Мониторинг с pgwatch2

### Установка pgwatch2

```bash
# Docker Compose
version: '3'
services:
  pgwatch2:
    image: cybertec/pgwatch2-postgres:latest
    ports:
      - "8080:8080"
      - "3000:3000"
    environment:
      - PW2_TESTDB=false
    volumes:
      - pgwatch2-data:/pgwatch2/persistent-config

volumes:
  pgwatch2-data:
```

### Добавление мониторинга репликации

```sql
-- Кастомные метрики для pgwatch2
INSERT INTO pgwatch2.metric (m_name, m_pg_version_from, m_sql)
VALUES (
    'replication_lag',
    11.0,
    $SQL$
    SELECT
        application_name,
        client_addr::text,
        state,
        sync_state,
        pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) as lag_bytes,
        EXTRACT(EPOCH FROM (now() - reply_time))::int as reply_lag_seconds
    FROM pg_stat_replication
    $SQL$
);
```

## Скрипты мониторинга

### Shell скрипт для мониторинга

```bash
#!/bin/bash
# check_replication.sh

PGHOST="${PGHOST:-localhost}"
PGPORT="${PGPORT:-5432}"
PGUSER="${PGUSER:-postgres}"
PGDATABASE="${PGDATABASE:-postgres}"

# Пороги
LAG_WARNING=60
LAG_CRITICAL=300
LAG_BYTES_WARNING=104857600   # 100MB
LAG_BYTES_CRITICAL=1073741824 # 1GB

# Проверка задержки репликации (на primary)
check_primary() {
    result=$(psql -h $PGHOST -p $PGPORT -U $PGUSER -d $PGDATABASE -tAc "
        SELECT
            COALESCE(
                MAX(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)),
                0
            ) as lag_bytes,
            COALESCE(
                MAX(EXTRACT(EPOCH FROM (now() - reply_time)))::int,
                0
            ) as lag_seconds,
            COUNT(*) as replica_count
        FROM pg_stat_replication
    ")

    IFS='|' read -r lag_bytes lag_seconds replica_count <<< "$result"

    if [ "$replica_count" -eq 0 ]; then
        echo "CRITICAL: No replicas connected"
        exit 2
    fi

    if [ "$lag_bytes" -gt "$LAG_BYTES_CRITICAL" ] || [ "$lag_seconds" -gt "$LAG_CRITICAL" ]; then
        echo "CRITICAL: Replication lag - ${lag_bytes} bytes, ${lag_seconds} seconds"
        exit 2
    elif [ "$lag_bytes" -gt "$LAG_BYTES_WARNING" ] || [ "$lag_seconds" -gt "$LAG_WARNING" ]; then
        echo "WARNING: Replication lag - ${lag_bytes} bytes, ${lag_seconds} seconds"
        exit 1
    else
        echo "OK: Replication lag - ${lag_bytes} bytes, ${lag_seconds} seconds, ${replica_count} replicas"
        exit 0
    fi
}

# Проверка состояния реплики (на standby)
check_standby() {
    is_replica=$(psql -h $PGHOST -p $PGPORT -U $PGUSER -d $PGDATABASE -tAc "SELECT pg_is_in_recovery()")

    if [ "$is_replica" != "t" ]; then
        echo "CRITICAL: Not a replica"
        exit 2
    fi

    lag_seconds=$(psql -h $PGHOST -p $PGPORT -U $PGUSER -d $PGDATABASE -tAc "
        SELECT COALESCE(
            EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))::int,
            0
        )
    ")

    receiver_status=$(psql -h $PGHOST -p $PGPORT -U $PGUSER -d $PGDATABASE -tAc "
        SELECT status FROM pg_stat_wal_receiver
    ")

    if [ "$receiver_status" != "streaming" ]; then
        echo "CRITICAL: WAL receiver is not streaming (status: $receiver_status)"
        exit 2
    fi

    if [ "$lag_seconds" -gt "$LAG_CRITICAL" ]; then
        echo "CRITICAL: Replication lag - ${lag_seconds} seconds"
        exit 2
    elif [ "$lag_seconds" -gt "$LAG_WARNING" ]; then
        echo "WARNING: Replication lag - ${lag_seconds} seconds"
        exit 1
    else
        echo "OK: Replication lag - ${lag_seconds} seconds"
        exit 0
    fi
}

# Определить роль и выполнить проверку
is_primary=$(psql -h $PGHOST -p $PGPORT -U $PGUSER -d $PGDATABASE -tAc "SELECT NOT pg_is_in_recovery()")

if [ "$is_primary" == "t" ]; then
    check_primary
else
    check_standby
fi
```

### Python скрипт для мониторинга

```python
#!/usr/bin/env python3
"""
PostgreSQL Replication Monitor
"""

import psycopg2
import argparse
import sys
from datetime import datetime

class ReplicationMonitor:
    def __init__(self, host, port, user, password, database):
        self.conn_params = {
            'host': host,
            'port': port,
            'user': user,
            'password': password,
            'database': database
        }

    def get_connection(self):
        return psycopg2.connect(**self.conn_params)

    def is_primary(self):
        with self.get_connection() as conn:
            with conn.cursor() as cur:
                cur.execute("SELECT NOT pg_is_in_recovery()")
                return cur.fetchone()[0]

    def get_replication_status(self):
        """Получить статус репликации на primary"""
        query = """
        SELECT
            application_name,
            client_addr::text,
            state,
            sync_state,
            pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) as pending_bytes,
            pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) as lag_bytes,
            EXTRACT(EPOCH FROM (now() - reply_time))::int as reply_lag_seconds
        FROM pg_stat_replication
        """
        with self.get_connection() as conn:
            with conn.cursor() as cur:
                cur.execute(query)
                columns = [desc[0] for desc in cur.description]
                return [dict(zip(columns, row)) for row in cur.fetchall()]

    def get_standby_status(self):
        """Получить статус на standby"""
        with self.get_connection() as conn:
            with conn.cursor() as cur:
                cur.execute("""
                    SELECT
                        status,
                        sender_host,
                        sender_port,
                        EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))::int as lag_seconds
                    FROM pg_stat_wal_receiver
                """)
                row = cur.fetchone()
                if row:
                    return {
                        'status': row[0],
                        'sender_host': row[1],
                        'sender_port': row[2],
                        'lag_seconds': row[3]
                    }
                return None

    def get_replication_slots(self):
        """Получить информацию о слотах репликации"""
        query = """
        SELECT
            slot_name,
            slot_type,
            active,
            pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) as retained_bytes
        FROM pg_replication_slots
        """
        with self.get_connection() as conn:
            with conn.cursor() as cur:
                cur.execute(query)
                columns = [desc[0] for desc in cur.description]
                return [dict(zip(columns, row)) for row in cur.fetchall()]

    def check_health(self, lag_warning=60, lag_critical=300):
        """Проверить здоровье репликации"""
        results = []

        if self.is_primary():
            replicas = self.get_replication_status()
            if not replicas:
                results.append(('CRITICAL', 'No replicas connected'))
            else:
                for replica in replicas:
                    if replica['state'] != 'streaming':
                        results.append(('CRITICAL', f"Replica {replica['application_name']} is not streaming"))
                    elif replica['reply_lag_seconds'] > lag_critical:
                        results.append(('CRITICAL', f"Replica {replica['application_name']} lag: {replica['reply_lag_seconds']}s"))
                    elif replica['reply_lag_seconds'] > lag_warning:
                        results.append(('WARNING', f"Replica {replica['application_name']} lag: {replica['reply_lag_seconds']}s"))
                    else:
                        results.append(('OK', f"Replica {replica['application_name']} healthy, lag: {replica['reply_lag_seconds']}s"))
        else:
            status = self.get_standby_status()
            if not status:
                results.append(('CRITICAL', 'WAL receiver not running'))
            elif status['status'] != 'streaming':
                results.append(('CRITICAL', f"WAL receiver status: {status['status']}"))
            elif status['lag_seconds'] > lag_critical:
                results.append(('CRITICAL', f"Replication lag: {status['lag_seconds']}s"))
            elif status['lag_seconds'] > lag_warning:
                results.append(('WARNING', f"Replication lag: {status['lag_seconds']}s"))
            else:
                results.append(('OK', f"Replication healthy, lag: {status['lag_seconds']}s"))

        return results


def main():
    parser = argparse.ArgumentParser(description='PostgreSQL Replication Monitor')
    parser.add_argument('-H', '--host', default='localhost')
    parser.add_argument('-p', '--port', default=5432, type=int)
    parser.add_argument('-U', '--user', default='postgres')
    parser.add_argument('-P', '--password', default='')
    parser.add_argument('-d', '--database', default='postgres')
    parser.add_argument('--warning', default=60, type=int)
    parser.add_argument('--critical', default=300, type=int)

    args = parser.parse_args()

    monitor = ReplicationMonitor(
        host=args.host,
        port=args.port,
        user=args.user,
        password=args.password,
        database=args.database
    )

    try:
        results = monitor.check_health(args.warning, args.critical)

        exit_code = 0
        for status, message in results:
            print(f"{status}: {message}")
            if status == 'CRITICAL':
                exit_code = 2
            elif status == 'WARNING' and exit_code < 2:
                exit_code = 1

        sys.exit(exit_code)

    except Exception as e:
        print(f"CRITICAL: {str(e)}")
        sys.exit(2)


if __name__ == '__main__':
    main()
```

## Best Practices

### 1. Многоуровневый мониторинг

```
Уровень 1: Базовые проверки
- pg_isready (доступность)
- pg_is_in_recovery() (роль узла)

Уровень 2: Метрики репликации
- Задержка репликации (bytes и seconds)
- Состояние WAL receiver/sender
- Слоты репликации

Уровень 3: Детальный анализ
- Статистика запросов
- WAL generation rate
- Connection pools
```

### 2. Правильные пороги алертов

```yaml
# Рекомендуемые пороги
replication_lag_seconds:
  warning: 60    # 1 минута
  critical: 300  # 5 минут

replication_lag_bytes:
  warning: 104857600    # 100MB
  critical: 1073741824  # 1GB

slot_retained_wal:
  warning: 5368709120   # 5GB
  critical: 10737418240 # 10GB
```

### 3. Регулярная проверка

```bash
# Cron job для проверки
*/5 * * * * /usr/local/bin/check_replication.sh >> /var/log/replication_check.log 2>&1
```

## Заключение

Эффективный мониторинг репликации PostgreSQL включает:
- Использование встроенных системных представлений
- Интеграцию с Prometheus и Grafana
- Настройку алертов для критических метрик
- Автоматизацию проверок через скрипты

Ключевые метрики для мониторинга:
- Задержка репликации (в байтах и секундах)
- Состояние подключения репликации
- Размер WAL, удерживаемого слотами
- Timeline ID для обнаружения split-brain

---

[prev: 04-kubernetes-deployment](./04-kubernetes-deployment.md) | [next: 06-load-balancing](./06-load-balancing.md)
