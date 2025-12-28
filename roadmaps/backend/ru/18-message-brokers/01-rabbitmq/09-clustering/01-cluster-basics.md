# Основы кластера RabbitMQ

[prev: 04-streams-vs-queues](../08-streams/04-streams-vs-queues.md) | [next: 02-node-configuration](./02-node-configuration.md)

---

## Введение в кластеризацию

Кластеризация RabbitMQ позволяет объединить несколько узлов (nodes) в единую логическую систему обмена сообщениями. Это обеспечивает высокую доступность, масштабируемость и отказоустойчивость системы.

## Архитектура кластера

### Что такое узел (Node)?

Узел RabbitMQ — это отдельный экземпляр сервера RabbitMQ, работающий на виртуальной машине Erlang. Каждый узел имеет уникальное имя в формате `rabbit@hostname`.

```
Кластер RabbitMQ
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Node 1    │  │   Node 2    │  │   Node 3    │     │
│  │ rabbit@node1│  │ rabbit@node2│  │ rabbit@node3│     │
│  │             │  │             │  │             │     │
│  │ Queues      │  │ Queues      │  │ Queues      │     │
│  │ Exchanges   │  │ Exchanges   │  │ Exchanges   │     │
│  │ Bindings    │  │ Bindings    │  │ Bindings    │     │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘     │
│         │                │                │             │
│         └────────────────┼────────────────┘             │
│                          │                              │
│              Distributed Erlang                         │
│           (Erlang Distribution Protocol)                │
└─────────────────────────────────────────────────────────┘
```

### Что реплицируется между узлами?

В кластере RabbitMQ **реплицируются** следующие метаданные:
- Определения виртуальных хостов (vhosts)
- Определения exchanges
- Определения bindings
- Пользователи и права доступа
- Политики (policies)
- Runtime параметры

**Не реплицируются автоматически**:
- Содержимое обычных (classic) очередей — они живут только на одном узле
- Quorum queues реплицируются отдельным механизмом

## Distributed Erlang

### Что это такое?

RabbitMQ построен на платформе Erlang/OTP, которая изначально разрабатывалась для распределённых систем. Distributed Erlang — это встроенный механизм для связи между узлами Erlang.

### Как работает Distributed Erlang

```
┌─────────────────┐         ┌─────────────────┐
│     Node A      │         │     Node B      │
│                 │         │                 │
│  Erlang VM      │ ◄─────► │  Erlang VM      │
│                 │  TCP    │                 │
│  epmd (4369)    │         │  epmd (4369)    │
│  dist port      │         │  dist port      │
│  (25672)        │         │  (25672)        │
└─────────────────┘         └─────────────────┘
```

### Ключевые компоненты

**EPMD (Erlang Port Mapper Daemon)**:
- Работает на порту 4369
- Регистрирует узлы Erlang на хосте
- Помогает узлам находить друг друга

**Distribution Port**:
- По умолчанию порт 25672
- Используется для коммуникации между узлами
- Должен быть открыт в firewall

### Erlang Cookie

Для безопасности все узлы кластера должны иметь одинаковый **Erlang cookie** — секретный ключ аутентификации.

```bash
# Расположение cookie файла
# Linux
/var/lib/rabbitmq/.erlang.cookie

# Docker
/var/lib/rabbitmq/.erlang.cookie

# MacOS
~/.erlang.cookie
```

```bash
# Установка cookie
echo "my_secret_cookie_value" > /var/lib/rabbitmq/.erlang.cookie
chmod 400 /var/lib/rabbitmq/.erlang.cookie
```

## Типы узлов

### Disc Node (Дисковый узел)

Хранит метаданные на диске и в памяти.

```bash
# Создание дискового узла (по умолчанию)
rabbitmqctl join_cluster rabbit@node1
```

### RAM Node (Узел в памяти)

Хранит метаданные только в памяти (кроме сообщений в очередях).

```bash
# Создание RAM узла
rabbitmqctl join_cluster --ram rabbit@node1
```

**Важно**: В кластере должен быть минимум один disc node для сохранения метаданных.

## Docker Compose пример базового кластера

```yaml
version: '3.8'

services:
  rabbitmq1:
    image: rabbitmq:3.12-management
    hostname: rabbitmq1
    container_name: rabbitmq1
    environment:
      - RABBITMQ_ERLANG_COOKIE=secret_cookie_here
      - RABBITMQ_NODENAME=rabbit@rabbitmq1
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=admin123
    ports:
      - "5672:5672"
      - "15672:15672"
    networks:
      - rabbitmq-cluster
    volumes:
      - rabbitmq1-data:/var/lib/rabbitmq

  rabbitmq2:
    image: rabbitmq:3.12-management
    hostname: rabbitmq2
    container_name: rabbitmq2
    environment:
      - RABBITMQ_ERLANG_COOKIE=secret_cookie_here
      - RABBITMQ_NODENAME=rabbit@rabbitmq2
    depends_on:
      - rabbitmq1
    networks:
      - rabbitmq-cluster
    volumes:
      - rabbitmq2-data:/var/lib/rabbitmq

  rabbitmq3:
    image: rabbitmq:3.12-management
    hostname: rabbitmq3
    container_name: rabbitmq3
    environment:
      - RABBITMQ_ERLANG_COOKIE=secret_cookie_here
      - RABBITMQ_NODENAME=rabbit@rabbitmq3
    depends_on:
      - rabbitmq1
    networks:
      - rabbitmq-cluster
    volumes:
      - rabbitmq3-data:/var/lib/rabbitmq

networks:
  rabbitmq-cluster:
    driver: bridge

volumes:
  rabbitmq1-data:
  rabbitmq2-data:
  rabbitmq3-data:
```

## Проверка состояния кластера

```bash
# Статус кластера
rabbitmqctl cluster_status

# Список узлов
rabbitmqctl list_cluster_nodes

# Подробная информация
rabbitmq-diagnostics cluster_status
```

Пример вывода:

```
Cluster status of node rabbit@rabbitmq1 ...
Basics

Cluster name: rabbit@rabbitmq1

Disk Nodes

rabbit@rabbitmq1
rabbit@rabbitmq2
rabbit@rabbitmq3

Running Nodes

rabbit@rabbitmq1
rabbit@rabbitmq2
rabbit@rabbitmq3
```

## Best Practices для production

### 1. Количество узлов

- **Минимум 3 узла** для обеспечения кворума при использовании quorum queues
- Нечётное количество узлов предпочтительнее для избежания split-brain

### 2. Сетевые требования

- Низкая латентность между узлами (< 1 мс)
- Стабильное сетевое соединение
- Узлы должны находиться в одном дата-центре или близко расположенных

### 3. Безопасность

```bash
# Сгенерируйте надёжный cookie
openssl rand -hex 32
```

### 4. Мониторинг

- Отслеживайте состояние всех узлов
- Настройте алерты на недоступность узлов
- Мониторьте сетевые метрики между узлами

### 5. Время синхронизации

```bash
# Настройка таймаутов для net_ticktime
# В rabbitmq.conf
cluster_keepalive_interval = 10000
cluster_partition_handling = pause_minority
```

## Ограничения кластеризации

1. **Географическое расположение**: Кластеры не предназначены для WAN-сетей
2. **Classic queues**: Не реплицируются автоматически
3. **Network partitions**: Могут привести к split-brain
4. **Erlang cookie**: Все узлы должны иметь одинаковый cookie

## Заключение

Понимание основ кластеризации RabbitMQ критически важно для построения отказоустойчивых систем обмена сообщениями. Ключевые моменты:

- Кластер объединяет узлы через Distributed Erlang
- Метаданные реплицируются, очереди (classic) — нет
- Минимум 3 узла для production
- Erlang cookie обеспечивает аутентификацию между узлами

---

[prev: 04-streams-vs-queues](../08-streams/04-streams-vs-queues.md) | [next: 02-node-configuration](./02-node-configuration.md)
