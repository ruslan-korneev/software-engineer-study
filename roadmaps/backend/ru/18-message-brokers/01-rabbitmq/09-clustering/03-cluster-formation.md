# Формирование кластера RabbitMQ

## Способы формирования кластера

RabbitMQ поддерживает несколько способов создания кластера:
1. **Ручное формирование** через CLI команды
2. **Автоматическое** через конфигурационные файлы
3. **Peer Discovery** — автоматическое обнаружение узлов

## Предварительные требования

Перед формированием кластера убедитесь:

```bash
# 1. Одинаковый Erlang cookie на всех узлах
cat /var/lib/rabbitmq/.erlang.cookie

# 2. Узлы могут разрешать hostname друг друга
ping node1
ping node2
ping node3

# 3. Открыты необходимые порты
# 4369 - EPMD
# 5672 - AMQP
# 25672 - Erlang distribution
# 15672 - Management UI
```

## Ручное формирование кластера

### Шаг 1: Подготовка узлов

На каждом узле установите RabbitMQ и убедитесь в одинаковом cookie:

```bash
# На всех узлах установите одинаковый cookie
sudo systemctl stop rabbitmq-server
echo "my_secret_cluster_cookie" | sudo tee /var/lib/rabbitmq/.erlang.cookie
sudo chmod 400 /var/lib/rabbitmq/.erlang.cookie
sudo chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
sudo systemctl start rabbitmq-server
```

### Шаг 2: Присоединение к кластеру

На узлах 2 и 3 выполните:

```bash
# Остановить приложение (не сервер!)
sudo rabbitmqctl stop_app

# Сбросить узел (удаляет данные!)
sudo rabbitmqctl reset

# Присоединиться к кластеру
sudo rabbitmqctl join_cluster rabbit@node1

# Запустить приложение
sudo rabbitmqctl start_app
```

### Шаг 3: Проверка кластера

```bash
# Проверить статус кластера
sudo rabbitmqctl cluster_status

# Вывод:
# Cluster status of node rabbit@node1 ...
# Disk Nodes
#   rabbit@node1
#   rabbit@node2
#   rabbit@node3
# Running Nodes
#   rabbit@node1
#   rabbit@node2
#   rabbit@node3
```

### Удаление узла из кластера

```bash
# На удаляемом узле
sudo rabbitmqctl stop_app
sudo rabbitmqctl reset
sudo rabbitmqctl start_app

# Или с другого узла (принудительно)
sudo rabbitmqctl forget_cluster_node rabbit@node3
```

## Автоматическое формирование через конфигурацию

### Использование Classic Config

```ini
# rabbitmq.conf на каждом узле

cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config

# Список узлов кластера
cluster_formation.classic_config.nodes.1 = rabbit@node1
cluster_formation.classic_config.nodes.2 = rabbit@node2
cluster_formation.classic_config.nodes.3 = rabbit@node3

# Тип узла (disc или ram)
cluster_formation.node_type = disc

# Период ожидания других узлов при старте
cluster_formation.target_cluster_size_hint = 3
cluster_formation.discovery_retry_limit = 10
cluster_formation.discovery_retry_interval = 500
```

## Peer Discovery механизмы

### 1. DNS-based Discovery

Использует DNS записи для обнаружения узлов:

```ini
# rabbitmq.conf
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_dns
cluster_formation.dns.hostname = rabbitmq.service.consul

# Опционально: seed hostname
cluster_formation.dns.hostname = _rabbitmq._tcp.local
```

### 2. Consul Discovery

Интеграция с HashiCorp Consul:

```ini
# rabbitmq.conf
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_consul

cluster_formation.consul.host = consul.local
cluster_formation.consul.port = 8500
cluster_formation.consul.scheme = http

cluster_formation.consul.svc = rabbitmq
cluster_formation.consul.svc_addr_auto = true
cluster_formation.consul.svc_addr_use_nodename = true

# Опционально: ACL token
# cluster_formation.consul.acl_token = your-acl-token
```

### 3. etcd Discovery

Использование etcd для обнаружения:

```ini
# rabbitmq.conf
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_etcd

cluster_formation.etcd.endpoints.1 = etcd1:2379
cluster_formation.etcd.endpoints.2 = etcd2:2379
cluster_formation.etcd.endpoints.3 = etcd3:2379

cluster_formation.etcd.key_prefix = /rabbitmq/discovery
cluster_formation.etcd.cluster_name = production
```

### 4. Kubernetes Discovery

Для Kubernetes кластеров:

```ini
# rabbitmq.conf
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_k8s

cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
cluster_formation.k8s.address_type = hostname
cluster_formation.k8s.service_name = rabbitmq-headless
cluster_formation.k8s.hostname_suffix = .rabbitmq-headless.default.svc.cluster.local
```

## Docker Compose примеры

### Автоматический кластер с Classic Config

```yaml
version: '3.8'

x-rabbitmq-common: &rabbitmq-common
  image: rabbitmq:3.12-management
  environment:
    - RABBITMQ_ERLANG_COOKIE=SECRETCOOKIE123
  networks:
    - rabbitmq-net
  healthcheck:
    test: rabbitmq-diagnostics -q ping
    interval: 30s
    timeout: 10s
    retries: 5

services:
  rabbitmq1:
    <<: *rabbitmq-common
    hostname: rabbitmq1
    container_name: rabbitmq1
    environment:
      - RABBITMQ_ERLANG_COOKIE=SECRETCOOKIE123
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=admin123
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - ./cluster-config/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro
      - rabbitmq1-data:/var/lib/rabbitmq

  rabbitmq2:
    <<: *rabbitmq-common
    hostname: rabbitmq2
    container_name: rabbitmq2
    volumes:
      - ./cluster-config/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro
      - rabbitmq2-data:/var/lib/rabbitmq
    depends_on:
      rabbitmq1:
        condition: service_healthy

  rabbitmq3:
    <<: *rabbitmq-common
    hostname: rabbitmq3
    container_name: rabbitmq3
    volumes:
      - ./cluster-config/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro
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

### cluster-config/rabbitmq.conf

```ini
# Peer Discovery
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config
cluster_formation.classic_config.nodes.1 = rabbit@rabbitmq1
cluster_formation.classic_config.nodes.2 = rabbit@rabbitmq2
cluster_formation.classic_config.nodes.3 = rabbit@rabbitmq3

# Cluster settings
cluster_name = docker-cluster
cluster_partition_handling = pause_minority
cluster_formation.node_type = disc

# Wait for peers
cluster_formation.target_cluster_size_hint = 3
cluster_formation.discovery_retry_limit = 20
cluster_formation.discovery_retry_interval = 1000

# Default queue type
default_queue_type = quorum

# Memory
vm_memory_high_watermark.relative = 0.6
disk_free_limit.relative = 2.0
```

### Скрипт для проверки формирования кластера

```bash
#!/bin/bash
# check-cluster.sh

echo "Ожидание формирования кластера..."

for i in {1..30}; do
    NODES=$(docker exec rabbitmq1 rabbitmqctl cluster_status --formatter json 2>/dev/null | jq -r '.running_nodes | length')

    if [ "$NODES" == "3" ]; then
        echo "Кластер сформирован успешно! Узлов: $NODES"
        docker exec rabbitmq1 rabbitmqctl cluster_status
        exit 0
    fi

    echo "Попытка $i: Узлов в кластере: ${NODES:-0}"
    sleep 5
done

echo "Ошибка: кластер не сформирован за отведённое время"
exit 1
```

## Kubernetes StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
spec:
  serviceName: rabbitmq-headless
  replicas: 3
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      serviceAccountName: rabbitmq
      containers:
      - name: rabbitmq
        image: rabbitmq:3.12-management
        ports:
        - containerPort: 5672
          name: amqp
        - containerPort: 15672
          name: management
        - containerPort: 25672
          name: dist
        env:
        - name: RABBITMQ_ERLANG_COOKIE
          valueFrom:
            secretKeyRef:
              name: rabbitmq-secret
              key: erlang-cookie
        - name: RABBITMQ_USE_LONGNAME
          value: "true"
        - name: K8S_SERVICE_NAME
          value: "rabbitmq-headless"
        - name: RABBITMQ_NODENAME
          value: "rabbit@$(POD_NAME).$(K8S_SERVICE_NAME).$(POD_NAMESPACE).svc.cluster.local"
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: config
          mountPath: /etc/rabbitmq/rabbitmq.conf
          subPath: rabbitmq.conf
        - name: data
          mountPath: /var/lib/rabbitmq
      volumes:
      - name: config
        configMap:
          name: rabbitmq-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-headless
spec:
  clusterIP: None
  selector:
    app: rabbitmq
  ports:
  - port: 5672
    name: amqp
  - port: 25672
    name: dist
```

## Best Practices

### 1. Порядок запуска

- Запускайте seed-узел первым
- Используйте health checks для зависимостей
- Настройте достаточные retry интервалы

### 2. Надёжность формирования

```ini
# Увеличьте лимиты для нестабильных сетей
cluster_formation.discovery_retry_limit = 30
cluster_formation.discovery_retry_interval = 2000
```

### 3. Мониторинг формирования

```bash
# Логи формирования кластера
journalctl -u rabbitmq-server -f | grep -i cluster

# Или в Docker
docker logs -f rabbitmq1 2>&1 | grep -i cluster
```

### 4. Избегайте race conditions

- Не запускайте все узлы одновременно
- Используйте health checks с задержками
- Проверяйте успешность формирования кластера

## Устранение неполадок

### Узел не присоединяется

```bash
# Проверьте cookie
rabbitmqctl eval 'erlang:get_cookie().'

# Проверьте DNS
nslookup node1

# Проверьте сетевую связность
rabbitmq-diagnostics check_port_connectivity
```

### Ошибки в логах

```bash
# Распространённые ошибки:
# - "Connection refused" - проблема с портами
# - "Cookie mismatch" - разные cookies
# - "Hostname mismatch" - неправильное имя узла
```

### Сброс проблемного узла

```bash
# Полный сброс узла
rabbitmqctl stop_app
rabbitmqctl force_reset
rabbitmqctl start_app
```
