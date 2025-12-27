# Network Partitions в RabbitMQ

## Что такое Network Partition?

Network Partition (сетевой раздел) — это ситуация, когда узлы кластера теряют связь друг с другом, но продолжают работать. Это приводит к так называемому **split-brain** — состоянию, когда кластер разделяется на несколько независимых частей.

## Split-Brain проблема

```
Нормальное состояние:
┌─────────────────────────────────────────────┐
│                  Кластер                     │
│  ┌──────┐    ┌──────┐    ┌──────┐          │
│  │Node 1│◄──►│Node 2│◄──►│Node 3│          │
│  └──────┘    └──────┘    └──────┘          │
└─────────────────────────────────────────────┘

Split-Brain:
┌───────────────────┐    ┌───────────────────┐
│    Partition A    │    │    Partition B    │
│  ┌──────┐        │    │        ┌──────┐   │
│  │Node 1│        │ ✗  │        │Node 3│   │
│  └──────┘        │    │        └──────┘   │
│        ▲         │    │                    │
│        │         │    │                    │
│        ▼         │    │                    │
│  ┌──────┐        │    │                    │
│  │Node 2│        │    │                    │
│  └──────┘        │    │                    │
└───────────────────┘    └───────────────────┘
```

### Почему это опасно?

1. **Несогласованность данных** — каждая часть продолжает принимать сообщения
2. **Дублирование сообщений** — после восстановления связи
3. **Потеря сообщений** — при неправильной стратегии восстановления
4. **Конфликты** — разные версии очередей и exchanges

## Обнаружение Network Partition

### Механизм обнаружения

RabbitMQ использует **net_ticktime** для определения доступности узлов:

```erlang
% advanced.config
[
  {kernel, [
    {net_ticktime, 60}  % секунды
  ]}
].
```

```ini
# rabbitmq.conf
# Эквивалент net_ticktime не настраивается напрямую
# Используется advanced.config
```

### Как узнать о partition

```bash
# Проверка через CLI
rabbitmqctl cluster_status

# Вывод при partition:
# Network Partitions
#
# Node rabbit@node1 cannot communicate with:
#   rabbit@node3

# Через Management API
curl -u admin:password http://localhost:15672/api/nodes | jq '.[] | {name, partitions}'
```

### Мониторинг partitions

```bash
# Prometheus метрики
rabbitmq_cluster_network_partitions

# Alarms
rabbitmqctl list_alarms
```

## Стратегии обработки Network Partitions

RabbitMQ предоставляет несколько стратегий:

### 1. ignore (по умолчанию)

Игнорирует partition — не рекомендуется для production.

```ini
# rabbitmq.conf
cluster_partition_handling = ignore
```

**Когда использовать**: Только для разработки или тестирования.

### 2. pause_minority

Узлы в меньшей части кластера приостанавливаются.

```ini
# rabbitmq.conf
cluster_partition_handling = pause_minority
```

```
Пример с 3 узлами:
┌───────────────────┐    ┌───────────────────┐
│   Majority (2)    │    │   Minority (1)    │
│  ┌──────┐        │    │  ┌──────┐         │
│  │Node 1│ ACTIVE │    │  │Node 3│ PAUSED  │
│  └──────┘        │    │  └──────┘         │
│  ┌──────┐        │    │                    │
│  │Node 2│ ACTIVE │    │                    │
│  └──────┘        │    │                    │
└───────────────────┘    └───────────────────┘
```

**Преимущества**:
- Гарантирует согласованность
- Автоматическое восстановление при восстановлении сети

**Недостатки**:
- Требует нечётное количество узлов
- При равном разделении (50/50) все узлы приостанавливаются

### 3. autoheal

Автоматически выбирает "победителя" и синхронизирует данные.

```ini
# rabbitmq.conf
cluster_partition_handling = autoheal
```

**Алгоритм выбора "победителя"**:
1. Часть с большим количеством клиентских соединений
2. При равенстве — часть с большим количеством узлов
3. При равенстве — случайный выбор

**Преимущества**:
- Автоматическое восстановление
- Минимальный простой

**Недостатки**:
- Возможная потеря сообщений в "проигравшей" части
- Непредсказуемость выбора победителя

### 4. pause_if_all_down

Приостанавливает узел, если он не может связаться со списком определённых узлов.

```ini
# rabbitmq.conf
cluster_partition_handling = pause_if_all_down

# Список узлов для проверки
cluster_partition_handling.pause_if_all_down.nodes.1 = rabbit@node1
cluster_partition_handling.pause_if_all_down.nodes.2 = rabbit@node2

# Поведение при недоступности всех
cluster_partition_handling.pause_if_all_down.recover = autoheal
# или
# cluster_partition_handling.pause_if_all_down.recover = ignore
```

## Сравнение стратегий

| Стратегия | Согласованность | Доступность | Рекомендация |
|-----------|----------------|-------------|--------------|
| ignore | Низкая | Высокая | Не для production |
| pause_minority | Высокая | Средняя | **Рекомендуется** |
| autoheal | Средняя | Высокая | Если доступность важнее |
| pause_if_all_down | Высокая | Конфигурируемая | Специфичные случаи |

## Docker Compose с настройкой partition handling

```yaml
version: '3.8'

services:
  rabbitmq1:
    image: rabbitmq:3.12-management
    hostname: rabbitmq1
    container_name: rabbitmq1
    environment:
      - RABBITMQ_ERLANG_COOKIE=PARTITIONTESTCOOKIE
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=admin123
    ports:
      - "5672:5672"
      - "15672:15672"
    networks:
      rabbitmq-net:
        ipv4_address: 172.28.0.10
    volumes:
      - ./config/rabbitmq-partition.conf:/etc/rabbitmq/rabbitmq.conf:ro
      - ./config/advanced.config:/etc/rabbitmq/advanced.config:ro
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
      - RABBITMQ_ERLANG_COOKIE=PARTITIONTESTCOOKIE
    networks:
      rabbitmq-net:
        ipv4_address: 172.28.0.11
    volumes:
      - ./config/rabbitmq-partition.conf:/etc/rabbitmq/rabbitmq.conf:ro
      - ./config/advanced.config:/etc/rabbitmq/advanced.config:ro
    depends_on:
      rabbitmq1:
        condition: service_healthy

  rabbitmq3:
    image: rabbitmq:3.12-management
    hostname: rabbitmq3
    container_name: rabbitmq3
    environment:
      - RABBITMQ_ERLANG_COOKIE=PARTITIONTESTCOOKIE
    networks:
      rabbitmq-net:
        ipv4_address: 172.28.0.12
    volumes:
      - ./config/rabbitmq-partition.conf:/etc/rabbitmq/rabbitmq.conf:ro
      - ./config/advanced.config:/etc/rabbitmq/advanced.config:ro
    depends_on:
      rabbitmq1:
        condition: service_healthy

networks:
  rabbitmq-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16
```

### config/rabbitmq-partition.conf

```ini
# Cluster formation
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config
cluster_formation.classic_config.nodes.1 = rabbit@rabbitmq1
cluster_formation.classic_config.nodes.2 = rabbit@rabbitmq2
cluster_formation.classic_config.nodes.3 = rabbit@rabbitmq3

# Partition handling - ВАЖНО для production!
cluster_partition_handling = pause_minority

# Cluster name
cluster_name = partition-test-cluster

# Default queue type
default_queue_type = quorum

# Memory and disk
vm_memory_high_watermark.relative = 0.6
disk_free_limit.relative = 2.0

# Logging
log.console = true
log.console.level = info
```

### config/advanced.config

```erlang
[
  {kernel, [
    %% Уменьшено для тестирования (production: 60+)
    {net_ticktime, 30}
  ]}
].
```

## Симуляция Network Partition

### Использование iptables (Linux)

```bash
# Изолировать node3 от node1 и node2
# На node3:
iptables -A INPUT -s <node1-ip> -j DROP
iptables -A INPUT -s <node2-ip> -j DROP
iptables -A OUTPUT -d <node1-ip> -j DROP
iptables -A OUTPUT -d <node2-ip> -j DROP

# Восстановление
iptables -F
```

### Использование Docker network

```bash
# Отключить node3 от сети
docker network disconnect rabbitmq-net rabbitmq3

# Проверить partition
docker exec rabbitmq1 rabbitmqctl cluster_status

# Восстановить соединение
docker network connect rabbitmq-net rabbitmq3
```

### Скрипт тестирования partition

```bash
#!/bin/bash
# test-partition.sh

echo "=== Тест Network Partition ==="

# Проверка начального состояния
echo "Начальное состояние кластера:"
docker exec rabbitmq1 rabbitmqctl cluster_status

# Создание partition
echo "Создание partition (отключение rabbitmq3)..."
docker network disconnect rabbitmq_rabbitmq-net rabbitmq3

# Ожидание обнаружения partition
echo "Ожидание 60 секунд для обнаружения partition..."
sleep 60

# Проверка состояния
echo "Состояние после partition:"
docker exec rabbitmq1 rabbitmqctl cluster_status

# Проверка alarms
echo "Alarms:"
docker exec rabbitmq1 rabbitmqctl list_alarms

# Восстановление
echo "Восстановление соединения..."
docker network connect rabbitmq_rabbitmq-net rabbitmq3

# Ожидание восстановления
echo "Ожидание восстановления..."
sleep 30

# Финальная проверка
echo "Финальное состояние:"
docker exec rabbitmq1 rabbitmqctl cluster_status
```

## Ручное восстановление после Partition

### Если автоматическое восстановление не сработало

```bash
# 1. Определить "правильную" часть кластера
rabbitmqctl cluster_status

# 2. На "неправильных" узлах:
rabbitmqctl stop_app
rabbitmqctl force_reset
rabbitmqctl join_cluster rabbit@correct_node
rabbitmqctl start_app
```

### Синхронизация очередей после partition

```bash
# Для classic mirrored queues
rabbitmqctl sync_queue <queue_name>

# Для всех очередей
rabbitmqctl list_queues name slave_pids synchronised_slave_pids | \
  while read queue slaves synced; do
    if [ "$slaves" != "$synced" ]; then
      rabbitmqctl sync_queue "$queue"
    fi
  done
```

## Best Practices для предотвращения Partition

### 1. Сетевая инфраструктура

- Используйте надёжные сетевые соединения
- Избегайте WAN между узлами
- Настройте мониторинг сетевой задержки

### 2. Количество узлов

- Используйте **нечётное** количество узлов (3, 5, 7)
- Минимум **3 узла** для pause_minority

### 3. Географическое расположение

- Все узлы в одном дата-центре
- Для geo-репликации используйте Federation или Shovel

### 4. Мониторинг и алерты

```yaml
# Prometheus alerting rule
groups:
- name: rabbitmq
  rules:
  - alert: RabbitMQNetworkPartition
    expr: rabbitmq_cluster_network_partitions > 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "RabbitMQ network partition detected"
      description: "Cluster has network partitions"
```

### 5. Использование Quorum Queues

Quorum Queues лучше справляются с partition благодаря алгоритму Raft.

## Заключение

Network Partitions — критическая проблема в распределённых системах. Для production рекомендуется:

1. Использовать `pause_minority` стратегию
2. Иметь нечётное количество узлов (минимум 3)
3. Использовать Quorum Queues вместо Classic Mirrored
4. Настроить мониторинг и алерты
5. Регулярно тестировать восстановление после partition
