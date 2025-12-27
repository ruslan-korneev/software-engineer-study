# Elasticsearch в Production

## Мониторинг

### Cluster APIs

```bash
# Здоровье кластера
curl -X GET "localhost:9200/_cluster/health?pretty"

# Детальная статистика кластера
curl -X GET "localhost:9200/_cluster/stats?pretty"

# Состояние кластера
curl -X GET "localhost:9200/_cluster/state?pretty"

# Pending tasks
curl -X GET "localhost:9200/_cluster/pending_tasks?pretty"
```

### Cat APIs

```bash
# Узлы кластера
curl -X GET "localhost:9200/_cat/nodes?v&h=name,role,heap.percent,cpu,load_1m,disk.used_percent"

# Индексы
curl -X GET "localhost:9200/_cat/indices?v&h=index,health,status,pri,rep,docs.count,store.size"

# Шарды
curl -X GET "localhost:9200/_cat/shards?v&h=index,shard,prirep,state,docs,store,node"

# Allocation
curl -X GET "localhost:9200/_cat/allocation?v"

# Recovery
curl -X GET "localhost:9200/_cat/recovery?v&active_only=true"

# Thread pools
curl -X GET "localhost:9200/_cat/thread_pool?v&h=node_name,name,active,queue,rejected"
```

### Nodes Stats

```bash
# Статистика всех узлов
curl -X GET "localhost:9200/_nodes/stats?pretty"

# Конкретные метрики
curl -X GET "localhost:9200/_nodes/stats/jvm,os,process?pretty"

# Hot threads
curl -X GET "localhost:9200/_nodes/hot_threads"
```

### Index Stats

```bash
# Статистика индекса
curl -X GET "localhost:9200/products/_stats?pretty"

# Сегменты
curl -X GET "localhost:9200/products/_segments?pretty"

# Recovery статус
curl -X GET "localhost:9200/products/_recovery?pretty"
```

### Ключевые метрики для мониторинга

```yaml
# Cluster level
cluster_health_status: green/yellow/red
number_of_nodes: integer
active_primary_shards: integer
unassigned_shards: integer (должен быть 0)

# Node level
jvm_heap_used_percent: < 75%
cpu_percent: < 80%
disk_used_percent: < 85%
gc_collection_time: минимизировать

# Index level
indexing_rate: docs/sec
search_rate: queries/sec
search_latency: ms
refresh_time: ms

# Thread pools
search_queue: < 1000
write_queue: < 200
rejected_count: 0
```

## Бэкапы и восстановление

### Snapshot Repository

```bash
# Регистрация repository (файловая система)
curl -X PUT "localhost:9200/_snapshot/my_backup" -H 'Content-Type: application/json' -d'
{
  "type": "fs",
  "settings": {
    "location": "/mnt/backups/elasticsearch",
    "compress": true,
    "max_snapshot_bytes_per_sec": "50mb",
    "max_restore_bytes_per_sec": "50mb"
  }
}'

# S3 repository
curl -X PUT "localhost:9200/_snapshot/s3_backup" -H 'Content-Type: application/json' -d'
{
  "type": "s3",
  "settings": {
    "bucket": "elasticsearch-backups",
    "region": "eu-west-1",
    "base_path": "snapshots",
    "compress": true
  }
}'
```

### Создание снапшотов

```bash
# Снапшот всех индексов
curl -X PUT "localhost:9200/_snapshot/my_backup/snapshot_1?wait_for_completion=true"

# Снапшот конкретных индексов
curl -X PUT "localhost:9200/_snapshot/my_backup/snapshot_2" -H 'Content-Type: application/json' -d'
{
  "indices": "products,orders",
  "ignore_unavailable": true,
  "include_global_state": false
}'

# Проверка статуса
curl -X GET "localhost:9200/_snapshot/my_backup/snapshot_1/_status?pretty"

# Список снапшотов
curl -X GET "localhost:9200/_snapshot/my_backup/_all?pretty"
```

### Восстановление

```bash
# Восстановление всего снапшота
curl -X POST "localhost:9200/_snapshot/my_backup/snapshot_1/_restore"

# Восстановление с переименованием
curl -X POST "localhost:9200/_snapshot/my_backup/snapshot_1/_restore" -H 'Content-Type: application/json' -d'
{
  "indices": "products",
  "ignore_unavailable": true,
  "rename_pattern": "(.+)",
  "rename_replacement": "restored_$1",
  "index_settings": {
    "index.number_of_replicas": 0
  }
}'
```

### Snapshot Lifecycle Management (SLM)

```bash
# Политика автоматических снапшотов
curl -X PUT "localhost:9200/_slm/policy/daily-snapshots" -H 'Content-Type: application/json' -d'
{
  "schedule": "0 30 1 * * ?",
  "name": "<daily-snap-{now/d}>",
  "repository": "my_backup",
  "config": {
    "indices": ["*"],
    "ignore_unavailable": true,
    "include_global_state": false
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 50
  }
}'

# Запуск вручную
curl -X POST "localhost:9200/_slm/policy/daily-snapshots/_execute"
```

## Безопасность

### Включение безопасности

```yaml
# elasticsearch.yml
xpack.security.enabled: true
xpack.security.enrollment.enabled: true

# Transport layer TLS
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12

# HTTP layer TLS
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: http.p12
```

### Генерация сертификатов

```bash
# CA
bin/elasticsearch-certutil ca

# Сертификаты для узлов
bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12

# HTTP сертификат
bin/elasticsearch-certutil http
```

### Управление пользователями

```bash
# Создание пользователя
curl -X POST "localhost:9200/_security/user/app_user" -H 'Content-Type: application/json' -d'
{
  "password": "secure_password",
  "roles": ["app_role"],
  "full_name": "Application User"
}'

# Создание роли
curl -X PUT "localhost:9200/_security/role/app_role" -H 'Content-Type: application/json' -d'
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["products*"],
      "privileges": ["read", "write", "create_index"],
      "field_security": {
        "grant": ["*"],
        "except": ["internal_*"]
      }
    }
  ]
}'
```

### API Keys

```bash
# Создание API key
curl -X POST "localhost:9200/_security/api_key" -H 'Content-Type: application/json' -d'
{
  "name": "my-api-key",
  "expiration": "1d",
  "role_descriptors": {
    "search_role": {
      "cluster": ["monitor"],
      "indices": [
        {
          "names": ["products"],
          "privileges": ["read"]
        }
      ]
    }
  }
}'

# Использование
curl -X GET "localhost:9200/products/_search" -H "Authorization: ApiKey base64_encoded_key"
```

## Масштабирование

### Горизонтальное масштабирование

```yaml
# elasticsearch.yml — добавление новых узлов

# Data nodes
node.roles: [ data_content, data_hot ]
node.name: data-node-4

# Discovery
discovery.seed_hosts: ["master-1", "master-2", "master-3"]
```

### Shard Allocation

```bash
# Настройка allocation awareness
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "cluster.routing.allocation.awareness.attributes": "zone",
    "cluster.routing.allocation.awareness.force.zone.values": ["zone-a", "zone-b"]
  }
}'

# Исключение узла из allocation
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "transient": {
    "cluster.routing.allocation.exclude._name": "node-to-remove"
  }
}'

# Ребалансировка
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "cluster.routing.rebalance.enable": "all",
    "cluster.routing.allocation.cluster_concurrent_rebalance": 2
  }
}'
```

### Index Lifecycle Management (ILM)

```bash
# Создание политики
curl -X PUT "localhost:9200/_ilm/policy/logs_policy" -H 'Content-Type: application/json' -d'
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_age": "1d",
            "max_primary_shard_size": "50gb"
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          },
          "set_priority": {
            "priority": 50
          },
          "allocate": {
            "require": {
              "data": "warm"
            }
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "allocate": {
            "require": {
              "data": "cold"
            }
          },
          "set_priority": {
            "priority": 0
          }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}'

# Применение к index template
curl -X PUT "localhost:9200/_index_template/logs_template" -H 'Content-Type: application/json' -d'
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "index.lifecycle.name": "logs_policy",
      "index.lifecycle.rollover_alias": "logs"
    }
  }
}'
```

## Настройки производительности

### JVM Settings

```bash
# jvm.options
-Xms16g
-Xmx16g

# G1GC (рекомендуется)
-XX:+UseG1GC
-XX:G1ReservePercent=25
-XX:InitiatingHeapOccupancyPercent=30

# GC logging
-Xlog:gc*,gc+age=trace,safepoint:file=logs/gc.log:utctime,pid,tags:filecount=32,filesize=64m
```

### System Settings

```bash
# /etc/sysctl.conf
vm.max_map_count=262144
vm.swappiness=1

# /etc/security/limits.conf
elasticsearch soft nofile 65535
elasticsearch hard nofile 65535
elasticsearch soft memlock unlimited
elasticsearch hard memlock unlimited
```

### Elasticsearch Settings

```yaml
# elasticsearch.yml
bootstrap.memory_lock: true

# Thread pools
thread_pool.write.queue_size: 1000
thread_pool.search.queue_size: 1000

# Indexing
indices.memory.index_buffer_size: 20%
indices.queries.cache.size: 10%

# Network
http.max_content_length: 100mb
```

### Index Settings

```bash
curl -X PUT "localhost:9200/products/_settings" -H 'Content-Type: application/json' -d'
{
  "index": {
    "refresh_interval": "30s",
    "translog.durability": "async",
    "translog.sync_interval": "30s",
    "merge.scheduler.max_thread_count": 1
  }
}'
```

## Troubleshooting

### Allocation Problems

```bash
# Почему шард не назначен
curl -X GET "localhost:9200/_cluster/allocation/explain?pretty" -H 'Content-Type: application/json' -d'
{
  "index": "products",
  "shard": 0,
  "primary": true
}'

# Принудительное перераспределение
curl -X POST "localhost:9200/_cluster/reroute" -H 'Content-Type: application/json' -d'
{
  "commands": [
    {
      "allocate_stale_primary": {
        "index": "products",
        "shard": 0,
        "node": "node-1",
        "accept_data_loss": true
      }
    }
  ]
}'
```

### Slow Queries

```yaml
# elasticsearch.yml — slow log
index.search.slowlog.threshold.query.warn: 10s
index.search.slowlog.threshold.query.info: 5s
index.search.slowlog.threshold.fetch.warn: 1s
index.indexing.slowlog.threshold.index.warn: 10s
```

### Circuit Breakers

```bash
# Настройка breakers
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "indices.breaker.total.limit": "70%",
    "indices.breaker.fielddata.limit": "40%",
    "indices.breaker.request.limit": "40%"
  }
}'

# Проверка состояния
curl -X GET "localhost:9200/_nodes/stats/breaker?pretty"
```

## Best Practices

### Production Checklist

```
✅ Минимум 3 master-eligible nodes
✅ Dedicated master nodes для кластеров > 10 nodes
✅ Memory lock включён
✅ Swap отключён или minimized
✅ TLS для transport и HTTP
✅ Автоматические бэкапы настроены
✅ ILM для управления жизненным циклом
✅ Мониторинг и alerting настроены
✅ Slow logs включены
✅ Регулярное тестирование восстановления
```

### Рекомендации по ресурсам

```
RAM:
- 50% системной памяти для JVM heap (max 31GB)
- Остальное для filesystem cache

CPU:
- Минимум 2 cores для production
- 4-8 cores для data nodes

Disk:
- SSD для hot data
- HDD допустим для warm/cold
- RAID 0 или JBOD (ES сам реплицирует)
- Минимум 15% свободного места
```
