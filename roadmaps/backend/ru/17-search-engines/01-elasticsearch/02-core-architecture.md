# Архитектура Elasticsearch

## Основные компоненты

Elasticsearch построен как распределённая система с несколькими ключевыми абстракциями: кластер, узлы (nodes), индексы, шарды (shards) и реплики.

## Кластер (Cluster)

Кластер — это группа из одного или нескольких узлов, которые совместно хранят данные и предоставляют возможности поиска и индексации.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Elasticsearch Cluster                         │
│                    (cluster_name: "production")                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │    Node 1    │  │    Node 2    │  │    Node 3    │          │
│  │   (Master)   │  │    (Data)    │  │    (Data)    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

### Состояния кластера

```bash
# Проверка здоровья кластера
curl -X GET "localhost:9200/_cluster/health?pretty"
```

```json
{
  "cluster_name": "production",
  "status": "green",
  "number_of_nodes": 3,
  "number_of_data_nodes": 3,
  "active_primary_shards": 10,
  "active_shards": 20,
  "relocating_shards": 0,
  "unassigned_shards": 0
}
```

| Статус | Описание |
|--------|----------|
| **green** | Все primary и replica шарды активны |
| **yellow** | Все primary шарды активны, но не все реплики |
| **red** | Некоторые primary шарды недоступны |

## Узлы (Nodes)

Узел — это отдельный экземпляр Elasticsearch. Каждый узел может выполнять одну или несколько ролей.

### Типы узлов

```yaml
# elasticsearch.yml — конфигурация ролей узла

# Master-eligible node (управление кластером)
node.roles: [ master ]

# Data node (хранение и поиск)
node.roles: [ data ]

# Ingest node (предобработка данных)
node.roles: [ ingest ]

# Coordinating-only node (маршрутизация запросов)
node.roles: [ ]

# Комбинированные роли
node.roles: [ master, data, ingest ]
```

### Роли узлов подробнее

| Роль | Назначение | Ресурсы |
|------|------------|---------|
| **master** | Управление кластером, создание/удаление индексов | CPU, RAM |
| **data** | Хранение данных, выполнение поиска и агрегаций | CPU, RAM, Disk |
| **data_content** | Хранение контентных данных | Disk, RAM |
| **data_hot** | Горячие данные (частый доступ) | Fast Disk (SSD) |
| **data_warm** | Тёплые данные (редкий доступ) | HDD |
| **data_cold** | Холодные данные (архив) | Cheap storage |
| **ingest** | Предобработка документов | CPU |
| **ml** | Machine Learning задачи | CPU, RAM |
| **transform** | Трансформация данных | CPU, RAM |
| **remote_cluster_client** | Cross-cluster search | Network |

### Типичная архитектура production-кластера

```
┌─────────────────────────────────────────────────────────────────┐
│                         Production Cluster                       │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                │
│  │  Master 1  │  │  Master 2  │  │  Master 3  │   (3 dedicated │
│  │  (voting)  │  │  (voting)  │  │  (voting)  │    masters)    │
│  └────────────┘  └────────────┘  └────────────┘                │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                │
│  │   Data 1   │  │   Data 2   │  │   Data 3   │   (hot tier)   │
│  │   (hot)    │  │   (hot)    │  │   (hot)    │                │
│  └────────────┘  └────────────┘  └────────────┘                │
│                                                                  │
│  ┌────────────┐  ┌────────────┐                                 │
│  │   Data 4   │  │   Data 5   │                (warm tier)     │
│  │   (warm)   │  │   (warm)   │                                 │
│  └────────────┘  └────────────┘                                 │
│                                                                  │
│  ┌────────────┐  ┌────────────┐                                 │
│  │ Coordinate │  │ Coordinate │               (координаторы)   │
│  │    Node    │  │    Node    │                                 │
│  └────────────┘  └────────────┘                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Индексы (Indices)

Индекс — это логическая коллекция документов с определённым маппингом и настройками.

### Создание индекса

```bash
# Создание индекса с настройками
curl -X PUT "localhost:9200/products" -H 'Content-Type: application/json' -d'
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "1s",
    "analysis": {
      "analyzer": {
        "custom_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "asciifolding"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "price": { "type": "float" },
      "created_at": { "type": "date" }
    }
  }
}'
```

### Информация об индексе

```bash
# Получить настройки индекса
curl -X GET "localhost:9200/products/_settings?pretty"

# Получить маппинг
curl -X GET "localhost:9200/products/_mapping?pretty"

# Статистика индекса
curl -X GET "localhost:9200/products/_stats?pretty"
```

## Шарды (Shards)

Шард — это базовая единица масштабирования. Каждый индекс разбивается на шарды, которые распределяются по узлам.

### Типы шардов

```
┌─────────────────────────────────────────────────────────────────┐
│                     Index: "products"                            │
│                     (3 primary shards, 1 replica)                │
│                                                                  │
│  Node 1              Node 2              Node 3                  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │ P0 (primary)│    │ P1 (primary)│    │ P2 (primary)│         │
│  │ R1 (replica)│    │ R2 (replica)│    │ R0 (replica)│         │
│  └─────────────┘    └─────────────┘    └─────────────┘         │
│                                                                  │
│  P = Primary shard (запись + чтение)                            │
│  R = Replica shard (только чтение + failover)                   │
└─────────────────────────────────────────────────────────────────┘
```

### Primary vs Replica шарды

| Характеристика | Primary Shard | Replica Shard |
|---------------|---------------|---------------|
| **Запись** | Да | Нет (получает копию) |
| **Чтение** | Да | Да |
| **Failover** | — | Становится primary при отказе |
| **Расположение** | На любой data-ноде | На другой ноде (не той же, что primary) |

### Расчёт количества шардов

```python
# Рекомендации по размеру шарда
# Оптимально: 10-50 GB на шард
# Максимум: ~65 GB (иначе recovery медленный)

# Формула расчёта количества шардов
total_data_size = 500  # GB
shard_size = 30        # GB (целевой размер)
primary_shards = ceil(total_data_size / shard_size)  # = 17 шардов

# С учётом роста
growth_factor = 1.5  # 50% роста
future_shards = ceil(total_data_size * growth_factor / shard_size)  # = 25 шардов
```

### Просмотр распределения шардов

```bash
# Все шарды в кластере
curl -X GET "localhost:9200/_cat/shards?v"

# Шарды конкретного индекса
curl -X GET "localhost:9200/_cat/shards/products?v"
```

**Вывод:**
```
index    shard prirep state   docs   store ip         node
products 0     p      STARTED 10000  50mb  172.18.0.2 node-1
products 0     r      STARTED 10000  50mb  172.18.0.3 node-2
products 1     p      STARTED 10500  52mb  172.18.0.3 node-2
products 1     r      STARTED 10500  52mb  172.18.0.4 node-3
```

## Реплики (Replicas)

Реплики обеспечивают высокую доступность и увеличивают пропускную способность чтения.

### Настройка реплик

```bash
# Изменение количества реплик (можно на лету)
curl -X PUT "localhost:9200/products/_settings" -H 'Content-Type: application/json' -d'
{
  "number_of_replicas": 2
}'
```

### Allocation awareness

```yaml
# elasticsearch.yml — размещение реплик в разных зонах доступности
cluster.routing.allocation.awareness.attributes: zone

# На каждой ноде указать зону
node.attr.zone: zone-a  # или zone-b, zone-c
```

```bash
# Настройка force awareness
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "cluster.routing.allocation.awareness.attributes": "zone",
    "cluster.routing.allocation.awareness.force.zone.values": ["zone-a", "zone-b"]
  }
}'
```

## Сегменты (Segments)

Внутри каждого шарда данные хранятся в неизменяемых сегментах Lucene.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Shard                                     │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐       │
│  │ Segment 0 │ │ Segment 1 │ │ Segment 2 │ │ Segment 3 │       │
│  │ (merged)  │ │  (5 docs) │ │ (10 docs) │ │ (15 docs) │       │
│  └───────────┘ └───────────┘ └───────────┘ └───────────┘       │
│                                                                  │
│  + Translog (write-ahead log для durability)                    │
│  + In-memory buffer (новые документы до commit)                 │
└─────────────────────────────────────────────────────────────────┘
```

### Merge process

```bash
# Принудительный merge сегментов
curl -X POST "localhost:9200/products/_forcemerge?max_num_segments=1"

# Статистика сегментов
curl -X GET "localhost:9200/products/_segments?pretty"
```

## Процесс записи документа

```
1. Клиент → Coordinating Node (любой узел)
2. Coordinating Node → вычисляет shard_id = hash(doc_id) % num_shards
3. Coordinating Node → Primary Shard
4. Primary Shard:
   a. Записывает в translog
   b. Добавляет в in-memory buffer
   c. После refresh (по умолчанию 1s) — документ доступен для поиска
   d. После flush — данные на диске
5. Primary → реплицирует на все Replica Shards
6. Ответ клиенту после подтверждения от реплик
```

### Настройки durability

```bash
# Уровни гарантии записи
curl -X PUT "localhost:9200/products/_doc/1?wait_for_active_shards=all" -d'
{
  "name": "Product 1"
}'

# wait_for_active_shards варианты:
# 1 — только primary (fastest, least safe)
# all — все шарды включая реплики (slowest, safest)
# quorum — (replicas / 2) + 1
```

## Процесс поиска

```
1. Клиент → Coordinating Node
2. Coordinating Node → broadcast запроса на все шарды индекса
3. Каждый шард:
   a. Выполняет локальный поиск
   b. Возвращает отсортированный список (doc_id, score)
4. Coordinating Node:
   a. Собирает результаты со всех шардов
   b. Merge-sort по score
   c. Fetch фаза — получает полные документы
5. Ответ клиенту
```

```
┌──────────────────────────────────────────────────────────────────┐
│                        Search Flow                                │
│                                                                   │
│  Client                                                           │
│    │                                                              │
│    ▼                                                              │
│  Coordinating Node                                                │
│    │                                                              │
│    ├──────────────┬──────────────┤  (Query Phase)                │
│    ▼              ▼              ▼                                │
│  Shard 0       Shard 1       Shard 2                             │
│    │              │              │                                │
│    └──────────────┴──────────────┘                                │
│                   │                                               │
│                   ▼                                               │
│           Merge & Sort                                            │
│                   │                                               │
│                   ▼  (Fetch Phase)                                │
│           Get full docs                                           │
│                   │                                               │
│                   ▼                                               │
│             Response                                              │
└──────────────────────────────────────────────────────────────────┘
```

## Best Practices

### Планирование шардов

```
✅ Один шард = 10-50 GB данных
✅ Количество шардов кратно количеству data-нод
✅ Учитывать рост данных при создании индекса
✅ Использовать ILM для ротации индексов (time-based)

❌ Слишком много маленьких шардов (overhead)
❌ Слишком большие шарды (медленный recovery)
❌ Менять number_of_shards после создания (нельзя!)
```

### Конфигурация кластера

```yaml
# elasticsearch.yml — рекомендуемые настройки

# Кластер
cluster.name: production-cluster

# Узел
node.name: node-1
node.roles: [ data, ingest ]

# Discovery
discovery.seed_hosts: ["node-1", "node-2", "node-3"]
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]

# Memory
bootstrap.memory_lock: true  # отключить swap

# Network
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300
```

### JVM настройки

```bash
# jvm.options
-Xms16g  # 50% RAM, но не более 32GB
-Xmx16g  # Xms = Xmx (избежать resize)

# Для heap > 32GB теряется compressed oops
# Оптимально: 26-31 GB
```

## Мониторинг архитектуры

```bash
# Статистика узлов
curl -X GET "localhost:9200/_cat/nodes?v&h=name,role,heap.percent,cpu,load_1m"

# Allocation explain (почему шард не назначен)
curl -X GET "localhost:9200/_cluster/allocation/explain?pretty"

# Hot threads (диагностика нагрузки)
curl -X GET "localhost:9200/_nodes/hot_threads"
```
