# Хранение в облаке и контейнерах

[prev: 06-multi-cluster](./06-multi-cluster.md) | [next: 01-admin-clients](../09-management/01-admin-clients.md)

---

## Описание

Развёртывание Kafka в облачных средах и контейнерах требует особого подхода к организации хранилища данных. Эта тема охватывает best practices для работы с Kafka в Kubernetes, Docker, а также использование облачных сервисов хранения (S3, EBS, Azure Blob, GCS).

## Ключевые концепции

### Типы хранилища в облаке

```
┌─────────────────────────────────────────────────────────────────┐
│                   Cloud Storage Options                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  Block      │  │  Object     │  │  Managed Kafka          │  │
│  │  Storage    │  │  Storage    │  │  Services               │  │
│  │             │  │             │  │                         │  │
│  │ • EBS       │  │ • S3        │  │ • Confluent Cloud       │  │
│  │ • Azure Disk│  │ • GCS       │  │ • Amazon MSK            │  │
│  │ • GCE PD    │  │ • Azure Blob│  │ • Azure Event Hubs      │  │
│  │             │  │             │  │ • Aiven for Kafka       │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
│                                                                  │
│      Hot Data           Cold Data         Fully Managed          │
│   (Primary Storage)   (Tiered Storage)    (No Operations)        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Tiered Storage

Kafka 3.0+ поддерживает tiered storage — разделение хранилища на уровни:

```
┌─────────────────────────────────────────────────────────────────┐
│                      Tiered Storage                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    Local Tier (Hot)                        │  │
│  │                                                            │  │
│  │  • Recent data (last N hours/days)                         │  │
│  │  • Fast NVMe/SSD storage                                   │  │
│  │  • Low latency reads/writes                                │  │
│  │                                                            │  │
│  │  [Segment 1000] [Segment 1001] [Segment 1002] [Active]     │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              │ Automatic offload                 │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                   Remote Tier (Cold)                       │  │
│  │                                                            │  │
│  │  • Historical data                                         │  │
│  │  • Object storage (S3, GCS, Azure Blob)                    │  │
│  │  • Cost-effective, infinite capacity                       │  │
│  │                                                            │  │
│  │  [Segment 0] [Segment 1] ... [Segment 999]                 │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Kafka в Kubernetes

### Strimzi Operator

Strimzi — популярный оператор для управления Kafka в Kubernetes:

```yaml
# kafka-cluster.yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: kafka-cluster
  namespace: kafka
spec:
  kafka:
    version: 3.6.0
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
      - name: external
        port: 9094
        type: loadbalancer
        tls: true
    config:
      # Storage settings
      log.retention.hours: 168
      log.segment.bytes: 1073741824
      log.retention.check.interval.ms: 300000

      # Performance
      num.io.threads: 8
      num.network.threads: 8
      num.partitions: 12
      default.replication.factor: 3
      min.insync.replicas: 2

      # Cleanup
      log.cleanup.policy: delete
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2

    storage:
      type: jbod
      volumes:
        - id: 0
          type: persistent-claim
          size: 500Gi
          class: gp3
          deleteClaim: false

    resources:
      requests:
        memory: 8Gi
        cpu: "2"
      limits:
        memory: 16Gi
        cpu: "4"

    jvmOptions:
      -Xms: 4096m
      -Xmx: 4096m

    rack:
      topologyKey: topology.kubernetes.io/zone

  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 100Gi
      class: gp3
      deleteClaim: false

  entityOperator:
    topicOperator: {}
    userOperator: {}
```

### StorageClass для Kafka

```yaml
# storageclass-gp3.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "16000"
  throughput: "1000"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Retain
```

### JBOD (Just a Bunch of Disks) Configuration

```yaml
# kafka-jbod.yaml
spec:
  kafka:
    storage:
      type: jbod
      volumes:
        - id: 0
          type: persistent-claim
          size: 1Ti
          class: nvme-fast
          deleteClaim: false
        - id: 1
          type: persistent-claim
          size: 1Ti
          class: nvme-fast
          deleteClaim: false
        - id: 2
          type: persistent-claim
          size: 1Ti
          class: nvme-fast
          deleteClaim: false
```

### Pod Anti-Affinity для HA

```yaml
spec:
  kafka:
    template:
      pod:
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              - labelSelector:
                  matchExpressions:
                    - key: strimzi.io/name
                      operator: In
                      values:
                        - kafka-cluster-kafka
                topologyKey: kubernetes.io/hostname
            preferredDuringSchedulingIgnoredDuringExecution:
              - weight: 100
                podAffinityTerm:
                  labelSelector:
                    matchExpressions:
                      - key: strimzi.io/name
                        operator: In
                        values:
                          - kafka-cluster-kafka
                  topologyKey: topology.kubernetes.io/zone
```

## Kafka в Docker

### Docker Compose для разработки

```yaml
# docker-compose.yml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zookeeper-data:/var/lib/zookeeper/data
      - zookeeper-logs:/var/lib/zookeeper/log

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    hostname: kafka
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "9101:9101"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_JMX_PORT: 9101
      KAFKA_JMX_HOSTNAME: localhost

      # Storage settings
      KAFKA_LOG_DIRS: /var/lib/kafka/data
      KAFKA_LOG_RETENTION_HOURS: 168
      KAFKA_LOG_SEGMENT_BYTES: 1073741824
      KAFKA_LOG_RETENTION_CHECK_INTERVAL_MS: 300000

    volumes:
      - kafka-data:/var/lib/kafka/data

volumes:
  zookeeper-data:
  zookeeper-logs:
  kafka-data:
```

### Multi-broker Docker Compose

```yaml
# docker-compose-cluster.yml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    volumes:
      - zk-data:/var/lib/zookeeper/data

  kafka-1:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-1:29092,EXTERNAL://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
    volumes:
      - kafka-1-data:/var/lib/kafka/data

  kafka-2:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9093:9093"
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-2:29092,EXTERNAL://localhost:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
    volumes:
      - kafka-2-data:/var/lib/kafka/data

  kafka-3:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9094:9094"
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-3:29092,EXTERNAL://localhost:9094
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
    volumes:
      - kafka-3-data:/var/lib/kafka/data

volumes:
  zk-data:
  kafka-1-data:
  kafka-2-data:
  kafka-3-data:
```

## Tiered Storage Configuration

### Confluent Platform Tiered Storage

```properties
# server.properties

# Enable tiered storage
confluent.tier.feature = true
confluent.tier.enable = true

# S3 configuration
confluent.tier.backend = S3
confluent.tier.s3.bucket = kafka-tiered-storage
confluent.tier.s3.region = us-east-1
confluent.tier.s3.aws.access.key.id = ${secrets:aws_access_key}
confluent.tier.s3.aws.secret.access.key = ${secrets:aws_secret_key}

# Local retention (how long to keep on local disk)
confluent.tier.local.hotset.ms = 86400000  # 24 hours

# Topic-level tiered storage
# Через kafka-configs.sh или при создании топика
```

### Apache Kafka Tiered Storage (KIP-405)

```properties
# server.properties (Kafka 3.6+)

# Enable remote storage
remote.log.storage.system.enable = true

# Remote storage manager class
remote.log.storage.manager.class.name = org.apache.kafka.server.log.remote.storage.RemoteLogManager

# S3 Remote Storage Manager (plugin required)
remote.log.storage.manager.impl.class = com.example.S3RemoteStorageManager

# Remote log metadata manager
remote.log.metadata.manager.class.name = org.apache.kafka.server.log.remote.metadata.storage.TopicBasedRemoteLogMetadataManager

# Local retention before offloading
local.retention.ms = 86400000  # 1 day
local.retention.bytes = -1

# Remote retention
retention.ms = 2592000000  # 30 days
```

### Topic Configuration for Tiered Storage

```bash
# Создание топика с tiered storage
kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic events \
  --partitions 12 \
  --replication-factor 3 \
  --config remote.storage.enable=true \
  --config local.retention.ms=86400000 \
  --config retention.ms=2592000000

# Включение tiered storage для существующего топика
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name events \
  --alter \
  --add-config remote.storage.enable=true,local.retention.ms=86400000
```

## AWS EBS Optimization

### EBS Volume Types для Kafka

| Тип | IOPS | Throughput | Рекомендация |
|-----|------|------------|--------------|
| gp3 | До 16,000 | До 1,000 MB/s | Рекомендуется |
| io2 | До 64,000 | До 4,000 MB/s | High-performance |
| st1 | До 500 | До 500 MB/s | Cold data |

### Terraform для AWS Kafka

```hcl
# main.tf

resource "aws_ebs_volume" "kafka_data" {
  count             = 3
  availability_zone = element(var.availability_zones, count.index)
  size              = 1000  # 1 TB
  type              = "gp3"
  iops              = 16000
  throughput        = 1000
  encrypted         = true

  tags = {
    Name = "kafka-data-${count.index}"
  }
}

resource "aws_instance" "kafka" {
  count         = 3
  ami           = var.kafka_ami
  instance_type = "m5.2xlarge"

  root_block_device {
    volume_size = 50
    volume_type = "gp3"
  }

  # Attach EBS
  ebs_block_device {
    device_name = "/dev/sdf"
    volume_id   = aws_ebs_volume.kafka_data[count.index].id
  }

  tags = {
    Name = "kafka-broker-${count.index}"
  }
}
```

## Best Practices

### 1. Выбор типа хранилища

```
┌─────────────────────────────────────────────────────────────┐
│                 Storage Selection Guide                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Workload                    Recommended Storage             │
│  ─────────────────────────   ──────────────────────────────  │
│  Development/Testing         Local SSD, gp2                  │
│  Production (standard)       gp3, Azure Premium SSD          │
│  High-throughput             io2, NVMe instance storage      │
│  Cost-sensitive archival     Tiered Storage + S3/GCS         │
│  Kubernetes                  CSI with gp3/Premium SSD        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 2. Kubernetes Storage Best Practices

```yaml
# Рекомендуемые настройки для production

# 1. Используйте отдельные StorageClass для Kafka
# 2. Включите volumeBindingMode: WaitForFirstConsumer
# 3. Используйте reclaimPolicy: Retain
# 4. Настройте allowVolumeExpansion: true

# 5. Мониторинг использования дисков
resources:
  requests:
    storage: 500Gi  # Планируйте 70% заполнение
```

### 3. Docker Volume Performance

```bash
# Используйте local driver с правильными опциями
docker volume create --driver local \
  --opt type=none \
  --opt device=/dev/nvme0n1p1 \
  --opt o=bind \
  kafka-data

# Или named volume с tmpfs для тестов
docker volume create --driver local \
  --opt type=tmpfs \
  --opt device=tmpfs \
  kafka-test-data
```

### 4. Tiered Storage Considerations

```properties
# Когда использовать tiered storage:
# - Retention > 7 дней
# - Редкий доступ к историческим данным
# - Экономия на локальном хранилище

# Когда НЕ использовать:
# - Частый доступ к старым данным (replay)
# - Latency-критичные приложения
# - Небольшие объёмы данных
```

### 5. Мониторинг хранилища

```yaml
# Prometheus метрики для мониторинга

# Disk usage
kafka_log_log_size{topic="events"}

# Remote storage metrics (tiered storage)
kafka_server_remote_storage_manager_fetch_rate
kafka_server_remote_storage_manager_copy_rate
kafka_server_remote_storage_manager_delete_rate

# Kubernetes PV usage
kubelet_volume_stats_used_bytes
kubelet_volume_stats_capacity_bytes
```

### 6. Backup и Disaster Recovery

```bash
#!/bin/bash
# backup-to-s3.sh

KAFKA_DATA="/var/lib/kafka/data"
S3_BUCKET="s3://kafka-backup"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)

# Остановить broker перед backup
kafka-server-stop.sh

# Создать snapshot
aws s3 sync $KAFKA_DATA $S3_BUCKET/snapshot-$TIMESTAMP \
  --storage-class STANDARD_IA

# Запустить broker
kafka-server-start.sh -daemon config/server.properties
```

## Managed Kafka Services

### Amazon MSK

```yaml
# msk-cluster.yaml (CloudFormation)
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  MSKCluster:
    Type: AWS::MSK::Cluster
    Properties:
      ClusterName: production-kafka
      KafkaVersion: '3.5.1'
      NumberOfBrokerNodes: 3
      BrokerNodeGroupInfo:
        InstanceType: kafka.m5.2xlarge
        StorageInfo:
          EBSStorageInfo:
            VolumeSize: 1000
            ProvisionedThroughput:
              Enabled: true
              VolumeThroughput: 250
        ClientSubnets:
          - !Ref SubnetA
          - !Ref SubnetB
          - !Ref SubnetC
        SecurityGroups:
          - !Ref KafkaSecurityGroup
```

### Confluent Cloud

```bash
# Создание кластера через CLI
confluent kafka cluster create production-cluster \
  --cloud aws \
  --region us-east-1 \
  --type dedicated \
  --cku 4 \
  --availability multi-zone
```

## Связанные темы

- [Срок хранения данных](./01-data-retention.md) — retention в облаке
- [Перемещение данных](./02-data-movement.md) — backup и restore
- [Мульти-кластер](./06-multi-cluster.md) — cross-region deployment
- [Архитектуры](./05-architectures.md) — cloud-native архитектуры

---

[prev: 06-multi-cluster](./06-multi-cluster.md) | [next: 01-admin-clients](../09-management/01-admin-clients.md)
