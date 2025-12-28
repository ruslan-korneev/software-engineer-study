# Quorum Queues и High Availability

[prev: 04-network-partitions](./04-network-partitions.md) | [next: 01-authentication](../10-security/01-authentication.md)

---

## Введение в Quorum Queues

Quorum Queues — это современный тип очередей в RabbitMQ, разработанный для обеспечения высокой доступности (HA) и надёжности данных. Они используют алгоритм консенсуса **Raft** для репликации данных между узлами.

## Сравнение с Classic Mirrored Queues

| Характеристика | Quorum Queues | Classic Mirrored |
|----------------|---------------|------------------|
| Алгоритм репликации | Raft | Синхронная репликация |
| Согласованность | Строгая (strong) | Eventual |
| Производительность | Лучше | Хуже при синхронизации |
| Потеря сообщений | Минимальная | Возможна |
| Поддержка | Активная | **Устарел** (deprecated) |
| Рекомендация | **Использовать** | Не рекомендуется |

## Архитектура Quorum Queues

```
Quorum Queue "orders" (replication factor = 3)
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │    Node 1    │  │    Node 2    │  │    Node 3    │  │
│  │              │  │              │  │              │  │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │  │
│  │  │ LEADER │  │  │  │FOLLOWER│  │  │  │FOLLOWER│  │  │
│  │  │  WAL   │  │  │  │  WAL   │  │  │  │  WAL   │  │  │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │  │
│  │              │  │              │  │              │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│         │                 │                 │          │
│         └─────────────────┼─────────────────┘          │
│                           │                            │
│                    Raft Consensus                      │
└─────────────────────────────────────────────────────────┘
```

### Ключевые концепции

**Leader (Лидер)**:
- Обрабатывает все операции записи
- Координирует репликацию
- Один на очередь

**Follower (Последователь)**:
- Реплицирует данные от лидера
- Может стать лидером при отказе

**WAL (Write-Ahead Log)**:
- Журнал упреждающей записи
- Обеспечивает durability

## Создание Quorum Queue

### Через CLI

```bash
# Создание quorum queue
rabbitmqctl declare_queue name=my-queue queue_type=quorum

# С дополнительными параметрами
rabbitmqctl eval 'rabbit_amqqueue:declare(
  rabbit_misc:r(<<"/">>, queue, <<"my-queue">>),
  true, false,
  [{<<"x-queue-type">>, longstr, <<"quorum">>}],
  none, <<"guest">>
).'
```

### Через Management UI

1. Перейдите в Queues
2. Add a new queue
3. Выберите Type: Quorum
4. Укажите параметры

### Через код (Python/pika)

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Создание quorum queue
args = {
    'x-queue-type': 'quorum',
    'x-quorum-initial-group-size': 3  # размер реплики
}

channel.queue_declare(
    queue='orders',
    durable=True,
    arguments=args
)

print("Quorum queue 'orders' создана")
connection.close()
```

### Через Policy

```bash
# Создание политики для всех очередей
rabbitmqctl set_policy quorum-policy ".*" \
  '{"queue-type": "quorum"}' \
  --priority 1 \
  --apply-to queues
```

## Конфигурация Quorum Queues

### rabbitmq.conf

```ini
# Тип очередей по умолчанию
default_queue_type = quorum

# Размер группы по умолчанию
quorum_cluster_size = 3

# Путь к сегментам WAL
quorum_commands_soft_limit = 32
quorum_commands_hard_limit = 512

# Максимальный размер сегмента WAL (в байтах)
raft.segment_max_entries = 32768

# Интервал snapshot
raft.wal_max_entries = 32768
```

### Параметры очереди

```bash
# x-queue-type - тип очереди
# x-quorum-initial-group-size - начальный размер группы
# x-max-length - максимальное количество сообщений
# x-max-length-bytes - максимальный размер в байтах
# x-overflow - политика при переполнении (drop-head, reject-publish)
# x-delivery-limit - лимит повторных доставок
# x-dead-letter-exchange - DLX для отклонённых сообщений
```

## Docker Compose для Quorum Queues

```yaml
version: '3.8'

services:
  rabbitmq1:
    image: rabbitmq:3.12-management
    hostname: rabbitmq1
    container_name: rabbitmq1
    environment:
      - RABBITMQ_ERLANG_COOKIE=QUORUMQUEUESCOOKIE
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=admin123
    ports:
      - "5672:5672"
      - "15672:15672"
    networks:
      - rabbitmq-net
    volumes:
      - ./config/rabbitmq-quorum.conf:/etc/rabbitmq/rabbitmq.conf:ro
      - ./definitions.json:/etc/rabbitmq/definitions.json:ro
      - rabbitmq1-data:/var/lib/rabbitmq
    healthcheck:
      test: rabbitmq-diagnostics -q ping
      interval: 30s
      timeout: 10s
      retries: 5

  rabbitmq2:
    image: rabbitmq:3.12-management
    hostname: rabbitmq2
    container_name: rabbitmq2
    environment:
      - RABBITMQ_ERLANG_COOKIE=QUORUMQUEUESCOOKIE
    networks:
      - rabbitmq-net
    volumes:
      - ./config/rabbitmq-quorum.conf:/etc/rabbitmq/rabbitmq.conf:ro
      - rabbitmq2-data:/var/lib/rabbitmq
    depends_on:
      rabbitmq1:
        condition: service_healthy

  rabbitmq3:
    image: rabbitmq:3.12-management
    hostname: rabbitmq3
    container_name: rabbitmq3
    environment:
      - RABBITMQ_ERLANG_COOKIE=QUORUMQUEUESCOOKIE
    networks:
      - rabbitmq-net
    volumes:
      - ./config/rabbitmq-quorum.conf:/etc/rabbitmq/rabbitmq.conf:ro
      - rabbitmq3-data:/var/lib/rabbitmq
    depends_on:
      rabbitmq1:
        condition: service_healthy

networks:
  rabbitmq-net:
    driver: bridge

volumes:
  rabbitmq1-data:
  rabbitmq2-data:
  rabbitmq3-data:
```

### config/rabbitmq-quorum.conf

```ini
# Cluster formation
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config
cluster_formation.classic_config.nodes.1 = rabbit@rabbitmq1
cluster_formation.classic_config.nodes.2 = rabbit@rabbitmq2
cluster_formation.classic_config.nodes.3 = rabbit@rabbitmq3

# Partition handling
cluster_partition_handling = pause_minority

# Quorum queues settings
default_queue_type = quorum
quorum_cluster_size = 3

# Memory and disk
vm_memory_high_watermark.relative = 0.6
disk_free_limit.relative = 2.0

# Management plugin - загрузка definitions
management.load_definitions = /etc/rabbitmq/definitions.json

# Logging
log.console = true
log.console.level = info
```

### definitions.json (предопределённые очереди)

```json
{
  "queues": [
    {
      "name": "orders",
      "vhost": "/",
      "durable": true,
      "auto_delete": false,
      "arguments": {
        "x-queue-type": "quorum",
        "x-quorum-initial-group-size": 3,
        "x-max-length": 1000000,
        "x-delivery-limit": 5,
        "x-dead-letter-exchange": "dlx"
      }
    },
    {
      "name": "notifications",
      "vhost": "/",
      "durable": true,
      "auto_delete": false,
      "arguments": {
        "x-queue-type": "quorum",
        "x-quorum-initial-group-size": 3
      }
    }
  ],
  "exchanges": [
    {
      "name": "dlx",
      "vhost": "/",
      "type": "fanout",
      "durable": true
    }
  ]
}
```

## Отказоустойчивость Quorum Queues

### Сценарий: Отказ одного узла

```
До отказа:
┌──────┐  ┌──────┐  ┌──────┐
│Node 1│  │Node 2│  │Node 3│
│LEADER│  │FOLLOW│  │FOLLOW│
└──────┘  └──────┘  └──────┘

После отказа Node 1:
          ┌──────┐  ┌──────┐
    ✗     │Node 2│  │Node 3│
          │LEADER│  │FOLLOW│  ← Выборы нового лидера
          └──────┘  └──────┘
```

### Тест отказоустойчивости

```bash
#!/bin/bash
# test-failover.sh

echo "=== Тест Failover Quorum Queue ==="

# 1. Проверка состояния очереди
echo "Начальное состояние:"
docker exec rabbitmq1 rabbitmqctl list_queues name type leader members

# 2. Публикация тестовых сообщений
echo "Публикация 100 сообщений..."
for i in {1..100}; do
  docker exec rabbitmq1 rabbitmqadmin publish \
    exchange=amq.default \
    routing_key=orders \
    payload="Message $i"
done

# 3. Проверка количества сообщений
echo "Сообщений в очереди:"
docker exec rabbitmq1 rabbitmqctl list_queues name messages

# 4. Остановка лидера
LEADER=$(docker exec rabbitmq1 rabbitmqctl list_queues name leader --formatter json | jq -r '.[0].leader')
echo "Текущий лидер: $LEADER"
echo "Остановка лидера..."
docker stop ${LEADER#rabbit@}

# 5. Ожидание выборов
echo "Ожидание выборов нового лидера..."
sleep 10

# 6. Проверка нового состояния
ACTIVE_NODE=$(docker ps --filter "name=rabbitmq" --format "{{.Names}}" | head -1)
echo "Проверка через $ACTIVE_NODE:"
docker exec $ACTIVE_NODE rabbitmqctl list_queues name type leader messages

# 7. Восстановление узла
echo "Восстановление остановленного узла..."
docker start ${LEADER#rabbit@}
sleep 15

# 8. Финальная проверка
echo "Финальное состояние:"
docker exec rabbitmq1 rabbitmqctl list_queues name type leader members messages
```

## Мониторинг Quorum Queues

### CLI команды

```bash
# Информация о quorum queues
rabbitmqctl list_queues name type leader members messages

# Детальная информация
rabbitmq-diagnostics quorum_status --queue orders

# Статистика Raft
rabbitmq-diagnostics quorum_status

# Проверка здоровья
rabbitmq-diagnostics check_if_any_quorum_queues_are_replicated_to_a_minority
```

### Prometheus метрики

```yaml
# Ключевые метрики для мониторинга
- rabbitmq_queue_messages  # Количество сообщений
- rabbitmq_queue_consumers  # Количество consumers
- rabbitmq_queue_messages_ready  # Готовые к доставке
- rabbitmq_queue_messages_unacked  # Неподтверждённые

# Специфичные для quorum queues
- rabbitmq_raft_term_total  # Текущий Raft term
- rabbitmq_raft_log_snapshot_index  # Индекс snapshot
- rabbitmq_raft_log_last_written_index  # Последний записанный индекс
```

### Grafana dashboard

```json
{
  "panels": [
    {
      "title": "Quorum Queue Messages",
      "targets": [
        {
          "expr": "sum(rabbitmq_queue_messages{queue_type=\"quorum\"}) by (queue)"
        }
      ]
    },
    {
      "title": "Quorum Queue Leaders",
      "targets": [
        {
          "expr": "count(rabbitmq_queue_info{queue_type=\"quorum\"}) by (node)"
        }
      ]
    }
  ]
}
```

## Масштабирование и управление репликами

### Добавление реплик

```bash
# Увеличить размер группы
rabbitmqctl grow_quorum_queue orders rabbit@node4
```

### Удаление реплик

```bash
# Уменьшить размер группы
rabbitmqctl shrink_quorum_queue orders rabbit@node4
```

### Перебалансировка лидеров

```bash
# Передать лидерство
rabbitmqctl transfer_leadership orders rabbit@node2

# Автоматическая перебалансировка
rabbitmq-queues rebalance quorum
```

## Best Practices для Production

### 1. Размер группы репликации

```ini
# Рекомендация: 3 или 5 узлов
quorum_cluster_size = 3

# Формула для отказоустойчивости:
# Выдерживает отказ (N-1)/2 узлов
# 3 узла = 1 отказ
# 5 узлов = 2 отказа
```

### 2. Настройка delivery limits

```python
# Предотвращение бесконечных retry
args = {
    'x-queue-type': 'quorum',
    'x-delivery-limit': 5,  # Максимум 5 попыток
    'x-dead-letter-exchange': 'dlx',
    'x-dead-letter-routing-key': 'failed'
}
```

### 3. Ограничение размера очереди

```python
args = {
    'x-queue-type': 'quorum',
    'x-max-length': 1000000,  # Максимум сообщений
    'x-max-length-bytes': 1073741824,  # 1GB
    'x-overflow': 'reject-publish'  # Отклонять новые при переполнении
}
```

### 4. Настройка publishers

```python
# Включить publisher confirms
channel.confirm_delivery()

# Публикация с подтверждением
try:
    channel.basic_publish(
        exchange='',
        routing_key='orders',
        body=message,
        properties=pika.BasicProperties(
            delivery_mode=2,  # Persistent
        ),
        mandatory=True
    )
    print("Сообщение подтверждено")
except pika.exceptions.UnroutableError:
    print("Сообщение не доставлено")
```

### 5. Мониторинг и алерты

```yaml
# Prometheus alerts
groups:
- name: rabbitmq-quorum
  rules:
  - alert: QuorumQueueNoLeader
    expr: rabbitmq_queue_info{queue_type="quorum"} == 0
    for: 30s
    labels:
      severity: critical

  - alert: QuorumQueueMinority
    expr: |
      rabbitmq_queue_members{queue_type="quorum"} <
      rabbitmq_queue_quorum_target{queue_type="quorum"}
    for: 5m
    labels:
      severity: warning
```

## Ограничения Quorum Queues

1. **Не поддерживают**: TTL для сообщений (только для очереди), priorities, lazy mode
2. **Больше памяти**: Требуют больше ресурсов из-за репликации
3. **Сетевые требования**: Критична низкая latency между узлами
4. **Именование**: Должны быть durable, non-exclusive, non-autodeleted

## Заключение

Quorum Queues — рекомендуемый тип очередей для production систем, требующих высокой доступности. Ключевые преимущества:

1. **Строгая согласованность** благодаря Raft
2. **Автоматическое восстановление** при отказах
3. **Лучшая производительность** по сравнению с mirrored queues
4. **Активная поддержка** от команды RabbitMQ

При использовании:
- Минимум 3 узла в кластере
- Настройте delivery limits и DLX
- Мониторьте состояние реплик
- Используйте publisher confirms

---

[prev: 04-network-partitions](./04-network-partitions.md) | [next: 01-authentication](../10-security/01-authentication.md)
