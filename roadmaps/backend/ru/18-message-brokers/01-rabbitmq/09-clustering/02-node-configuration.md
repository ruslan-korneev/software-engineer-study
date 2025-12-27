# Конфигурация узлов RabbitMQ

## Обзор системы конфигурации

RabbitMQ использует несколько уровней конфигурации:
1. **rabbitmq.conf** — основной файл конфигурации (новый формат)
2. **advanced.config** — расширенная конфигурация на Erlang
3. **Переменные окружения** — для базовых настроек
4. **rabbitmq-env.conf** — переменные окружения специфичные для RabbitMQ

## Расположение конфигурационных файлов

```
# Linux (пакетная установка)
/etc/rabbitmq/rabbitmq.conf
/etc/rabbitmq/advanced.config
/etc/rabbitmq/rabbitmq-env.conf

# Docker
/etc/rabbitmq/rabbitmq.conf
/etc/rabbitmq/conf.d/

# Windows
%APPDATA%\RabbitMQ\rabbitmq.conf
```

## rabbitmq.conf — Основная конфигурация

Файл использует формат `key = value`. Это рекомендуемый способ конфигурации.

### Базовая конфигурация узла

```ini
# Имя узла
# nodename = rabbit@hostname

# Сетевые настройки
listeners.tcp.default = 5672
management.tcp.port = 15672

# SSL/TLS
# listeners.ssl.default = 5671
# ssl_options.cacertfile = /path/to/ca_certificate.pem
# ssl_options.certfile = /path/to/server_certificate.pem
# ssl_options.keyfile = /path/to/server_key.pem

# Логирование
log.file.level = info
log.console = true
log.console.level = info

# Путь к данным
# mnesia_base = /var/lib/rabbitmq/mnesia
```

### Конфигурация для кластера

```ini
# Настройки кластеризации
cluster_name = my_rabbitmq_cluster
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config
cluster_formation.classic_config.nodes.1 = rabbit@node1
cluster_formation.classic_config.nodes.2 = rabbit@node2
cluster_formation.classic_config.nodes.3 = rabbit@node3

# Обработка network partitions
cluster_partition_handling = pause_minority

# Keepalive между узлами (в миллисекундах)
cluster_keepalive_interval = 10000

# Автоматическое восстановление после partition
# cluster_partition_handling = autoheal
```

### Настройки памяти и дисков

```ini
# Порог памяти (относительный)
vm_memory_high_watermark.relative = 0.6

# Или абсолютный (в байтах)
# vm_memory_high_watermark.absolute = 2GB

# Порог свободного места на диске
disk_free_limit.relative = 2.0
# или абсолютный
# disk_free_limit.absolute = 5GB

# Стратегия paging
vm_memory_high_watermark_paging_ratio = 0.75
```

### Настройки соединений

```ini
# Heartbeat интервал (секунды)
heartbeat = 60

# Таймаут TCP handshake
handshake_timeout = 10000

# Максимальное количество каналов на соединение
channel_max = 2047

# Максимальный размер frame
frame_max = 131072
```

### Настройки очередей

```ini
# Quorum queues: целевой размер группы
quorum_cluster_size = 3

# Политика очередей по умолчанию
default_queue_type = quorum

# Classic queues: lazy mode
queue_master_locator = client-local
```

## advanced.config — Расширенная конфигурация

Файл advanced.config используется для настроек, которые невозможно выразить в rabbitmq.conf.

### Формат файла

```erlang
[
  {rabbit, [
    {tcp_listeners, [5672]},
    {ssl_listeners, [5671]},
    {cluster_partition_handling, pause_minority},
    {collect_statistics_interval, 5000}
  ]},
  {rabbitmq_management, [
    {listener, [{port, 15672}]},
    {load_definitions, "/etc/rabbitmq/definitions.json"}
  ]},
  {rabbitmq_shovel, [
    {shovels, []}
  ]}
].
```

### Примеры расширенных настроек

```erlang
[
  {rabbit, [
    %% Настройки SSL для кластера
    {ssl_options, [
      {cacertfile, "/path/to/ca_certificate.pem"},
      {certfile, "/path/to/server_certificate.pem"},
      {keyfile, "/path/to/server_key.pem"},
      {verify, verify_peer},
      {fail_if_no_peer_cert, true}
    ]},

    %% Настройки inter-node communication
    {cluster_nodes, {['rabbit@node1', 'rabbit@node2', 'rabbit@node3'], disc}},

    %% Кастомные политики
    {default_policies, [
      {<<"ha-all">>, <<"^">>, [{<<"ha-mode">>, <<"all">>}]}
    ]}
  ]},

  {kernel, [
    %% Настройки Erlang distribution
    {inet_dist_listen_min, 25672},
    {inet_dist_listen_max, 25672},

    %% Net ticktime для определения недоступности узла
    {net_ticktime, 60}
  ]}
].
```

## Переменные окружения

### Системные переменные

```bash
# Имя узла
export RABBITMQ_NODENAME=rabbit@myhost

# Erlang cookie
export RABBITMQ_ERLANG_COOKIE=secret_cookie

# Пути к конфигурации
export RABBITMQ_CONFIG_FILE=/etc/rabbitmq/rabbitmq
export RABBITMQ_ADVANCED_CONFIG_FILE=/etc/rabbitmq/advanced.config

# Порты
export RABBITMQ_NODE_PORT=5672
export RABBITMQ_DIST_PORT=25672

# Логи
export RABBITMQ_LOGS=/var/log/rabbitmq/rabbit.log

# Данные
export RABBITMQ_MNESIA_BASE=/var/lib/rabbitmq/mnesia
export RABBITMQ_MNESIA_DIR=/var/lib/rabbitmq/mnesia/rabbit@myhost
```

### rabbitmq-env.conf

```bash
# /etc/rabbitmq/rabbitmq-env.conf

# Имя узла
NODENAME=rabbit@myhost

# Erlang cookie (лучше использовать файл .erlang.cookie)
# RABBITMQ_ERLANG_COOKIE=secret

# Конфигурационные файлы
CONFIG_FILE=/etc/rabbitmq/rabbitmq
ADVANCED_CONFIG_FILE=/etc/rabbitmq/advanced.config

# Сеть
NODE_IP_ADDRESS=0.0.0.0
NODE_PORT=5672

# Erlang VM параметры
SERVER_ERL_ARGS="+K true +P 1048576 -kernel inet_default_connect_options [{nodelay,true}]"

# Memory
RABBITMQ_SERVER_ERL_ARGS="+K true +A 128"
```

## Docker конфигурация кластера

### docker-compose.yml с полной конфигурацией

```yaml
version: '3.8'

services:
  rabbitmq1:
    image: rabbitmq:3.12-management
    hostname: rabbitmq1
    container_name: rabbitmq1
    environment:
      - RABBITMQ_ERLANG_COOKIE=SWQOKODSQALRPCLNMEQG
      - RABBITMQ_NODENAME=rabbit@rabbitmq1
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=admin123
      - RABBITMQ_DEFAULT_VHOST=/
    ports:
      - "5672:5672"
      - "15672:15672"
      - "25672:25672"
    networks:
      - rabbitmq-cluster
    volumes:
      - ./config/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro
      - ./config/advanced.config:/etc/rabbitmq/advanced.config:ro
      - ./config/enabled_plugins:/etc/rabbitmq/enabled_plugins:ro
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
      - RABBITMQ_ERLANG_COOKIE=SWQOKODSQALRPCLNMEQG
      - RABBITMQ_NODENAME=rabbit@rabbitmq2
    ports:
      - "5673:5672"
      - "15673:15672"
    networks:
      - rabbitmq-cluster
    volumes:
      - ./config/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro
      - ./config/advanced.config:/etc/rabbitmq/advanced.config:ro
      - rabbitmq2-data:/var/lib/rabbitmq
    depends_on:
      rabbitmq1:
        condition: service_healthy

  rabbitmq3:
    image: rabbitmq:3.12-management
    hostname: rabbitmq3
    container_name: rabbitmq3
    environment:
      - RABBITMQ_ERLANG_COOKIE=SWQOKODSQALRPCLNMEQG
      - RABBITMQ_NODENAME=rabbit@rabbitmq3
    ports:
      - "5674:5672"
      - "15674:15672"
    networks:
      - rabbitmq-cluster
    volumes:
      - ./config/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro
      - ./config/advanced.config:/etc/rabbitmq/advanced.config:ro
      - rabbitmq3-data:/var/lib/rabbitmq
    depends_on:
      rabbitmq1:
        condition: service_healthy

networks:
  rabbitmq-cluster:
    driver: bridge

volumes:
  rabbitmq1-data:
  rabbitmq2-data:
  rabbitmq3-data:
```

### config/rabbitmq.conf

```ini
# Кластер
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config
cluster_formation.classic_config.nodes.1 = rabbit@rabbitmq1
cluster_formation.classic_config.nodes.2 = rabbit@rabbitmq2
cluster_formation.classic_config.nodes.3 = rabbit@rabbitmq3

cluster_partition_handling = pause_minority
cluster_name = production-cluster

# Память и диск
vm_memory_high_watermark.relative = 0.6
disk_free_limit.relative = 2.0

# Очереди по умолчанию
default_queue_type = quorum

# Сеть
heartbeat = 60

# Логирование
log.console = true
log.console.level = info
log.file.level = info
```

### config/enabled_plugins

```
[rabbitmq_management,rabbitmq_prometheus,rabbitmq_peer_discovery_common].
```

## Проверка конфигурации

```bash
# Проверка эффективной конфигурации
rabbitmq-diagnostics environment

# Проверка параметров памяти
rabbitmq-diagnostics memory_breakdown

# Проверка конфигурации кластера
rabbitmqctl cluster_status

# Валидация конфигурационного файла
rabbitmq-diagnostics check_local_alarms
```

## Best Practices

### 1. Разделение конфигурации

```
/etc/rabbitmq/
├── rabbitmq.conf          # Основные настройки
├── advanced.config        # Erlang-специфичные настройки
├── conf.d/                # Дополнительные конфигурации
│   ├── 10-cluster.conf
│   ├── 20-memory.conf
│   └── 30-logging.conf
└── enabled_plugins        # Список плагинов
```

### 2. Безопасность

- Не храните пароли в конфигурационных файлах
- Используйте переменные окружения для secrets
- Ограничьте доступ к файлу .erlang.cookie

### 3. Мониторинг конфигурации

```bash
# Экспорт текущих определений
rabbitmqctl export_definitions /backup/definitions.json

# Импорт определений
rabbitmqctl import_definitions /backup/definitions.json
```

### 4. Версионирование

- Храните конфигурации в системе контроля версий
- Используйте шаблоны для разных окружений
- Документируйте изменения конфигурации

## Частые ошибки конфигурации

1. **Разные Erlang cookies** — узлы не смогут соединиться
2. **Неправильные hostname** — DNS должен корректно разрешать имена
3. **Закрытые порты** — 4369, 5672, 25672 должны быть доступны
4. **Несовместимые версии** — все узлы должны иметь одинаковую версию
