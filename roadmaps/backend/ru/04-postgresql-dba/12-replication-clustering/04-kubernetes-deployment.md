# Развертывание PostgreSQL в Kubernetes

[prev: 03-patroni](./03-patroni.md) | [next: 05-monitoring](./05-monitoring.md)

---

## Введение

Развертывание PostgreSQL в Kubernetes требует особого подхода из-за stateful природы баз данных. В этом разделе рассмотрим различные способы развертывания, от простого StatefulSet до использования специализированных операторов.

## Архитектура PostgreSQL в Kubernetes

### Основные концепции

```
┌──────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                         │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                      Namespace: postgres                    │  │
│  │                                                             │  │
│  │  ┌─────────────────────────────────────────────────────┐   │  │
│  │  │              StatefulSet: postgres                   │   │  │
│  │  │  ┌────────────┐ ┌────────────┐ ┌────────────┐       │   │  │
│  │  │  │ Pod-0      │ │ Pod-1      │ │ Pod-2      │       │   │  │
│  │  │  │ (Leader)   │ │ (Replica)  │ │ (Replica)  │       │   │  │
│  │  │  │            │ │            │ │            │       │   │  │
│  │  │  │ ┌────────┐ │ │ ┌────────┐ │ │ ┌────────┐ │       │   │  │
│  │  │  │ │ PG     │ │ │ │ PG     │ │ │ │ PG     │ │       │   │  │
│  │  │  │ │ Patroni│ │ │ │ Patroni│ │ │ │ Patroni│ │       │   │  │
│  │  │  │ └────────┘ │ │ └────────┘ │ │ └────────┘ │       │   │  │
│  │  │  │     │      │ │     │      │ │     │      │       │   │  │
│  │  │  │ ┌───▼────┐ │ │ ┌───▼────┐ │ │ ┌───▼────┐ │       │   │  │
│  │  │  │ │  PVC   │ │ │ │  PVC   │ │ │ │  PVC   │ │       │   │  │
│  │  │  │ └────────┘ │ │ └────────┘ │ │ └────────┘ │       │   │  │
│  │  │  └────────────┘ └────────────┘ └────────────┘       │   │  │
│  │  └─────────────────────────────────────────────────────┘   │  │
│  │                                                             │  │
│  │  ┌─────────────────┐  ┌─────────────────────────────────┐  │  │
│  │  │ Service: pg-rw  │  │ Service: pg-ro                  │  │  │
│  │  │ (ClusterIP)     │  │ (ClusterIP)                     │  │  │
│  │  └─────────────────┘  └─────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

## Способы развертывания

### 1. Простой StatefulSet (без HA)

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: postgres
---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: postgres
type: Opaque
stringData:
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: secretpassword
  POSTGRES_DB: myapp
---
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: postgres
data:
  postgresql.conf: |
    listen_addresses = '*'
    max_connections = 200
    shared_buffers = 256MB
    effective_cache_size = 768MB
    maintenance_work_mem = 64MB
    checkpoint_completion_target = 0.9
    wal_buffers = 16MB
    default_statistics_target = 100
    random_page_cost = 1.1
    effective_io_concurrency = 200
    work_mem = 6553kB
    min_wal_size = 1GB
    max_wal_size = 4GB
---
# statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
          name: postgres
        envFrom:
        - secretRef:
            name: postgres-secret
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        - name: config
          mountPath: /etc/postgresql/postgresql.conf
          subPath: postgresql.conf
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: config
        configMap:
          name: postgres-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: postgres
spec:
  type: ClusterIP
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: postgres
```

### 2. Развертывание с Patroni

```yaml
# Полный набор манифестов для Patroni на Kubernetes

# service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: patroni
  namespace: postgres
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: patroni
  namespace: postgres
rules:
- apiGroups: [""]
  resources: ["configmaps", "endpoints", "pods"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: patroni
  namespace: postgres
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: patroni
subjects:
- kind: ServiceAccount
  name: patroni
  namespace: postgres
---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: patroni-secret
  namespace: postgres
type: Opaque
stringData:
  PATRONI_SUPERUSER_PASSWORD: postgres
  PATRONI_REPLICATION_PASSWORD: replicator
---
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: patroni-config
  namespace: postgres
data:
  patroni.yaml: |
    bootstrap:
      dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
        postgresql:
          use_pg_rewind: true
          use_slots: true
          parameters:
            wal_level: replica
            hot_standby: "on"
            max_wal_senders: 10
            max_replication_slots: 10
            wal_log_hints: "on"
            max_connections: 200
            shared_buffers: 512MB
      initdb:
        - encoding: UTF8
        - data-checksums
      pg_hba:
        - host all all 0.0.0.0/0 md5
        - host replication replicator 0.0.0.0/0 md5
    postgresql:
      authentication:
        superuser:
          username: postgres
        replication:
          username: replicator
---
# services.yaml
apiVersion: v1
kind: Service
metadata:
  name: patroni
  namespace: postgres
  labels:
    app: patroni
spec:
  type: ClusterIP
  clusterIP: None  # Headless service
  ports:
  - port: 5432
    name: postgresql
  - port: 8008
    name: patroni
  selector:
    app: patroni
---
apiVersion: v1
kind: Service
metadata:
  name: patroni-master
  namespace: postgres
  labels:
    app: patroni
spec:
  type: ClusterIP
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: patroni
    role: master
---
apiVersion: v1
kind: Service
metadata:
  name: patroni-replica
  namespace: postgres
  labels:
    app: patroni
spec:
  type: ClusterIP
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: patroni
    role: replica
---
# statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: patroni
  namespace: postgres
spec:
  serviceName: patroni
  replicas: 3
  selector:
    matchLabels:
      app: patroni
  template:
    metadata:
      labels:
        app: patroni
    spec:
      serviceAccountName: patroni
      containers:
      - name: patroni
        image: registry.opensource.zalan.do/acid/spilo-16:3.0-p1
        ports:
        - containerPort: 5432
          name: postgresql
        - containerPort: 8008
          name: patroni
        env:
        - name: PATRONI_KUBERNETES_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: PATRONI_KUBERNETES_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: PATRONI_KUBERNETES_LABELS
          value: '{app: patroni}'
        - name: PATRONI_SCOPE
          value: postgres-cluster
        - name: PATRONI_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: PATRONI_SUPERUSER_USERNAME
          value: postgres
        - name: PATRONI_SUPERUSER_PASSWORD
          valueFrom:
            secretKeyRef:
              name: patroni-secret
              key: PATRONI_SUPERUSER_PASSWORD
        - name: PATRONI_REPLICATION_USERNAME
          value: replicator
        - name: PATRONI_REPLICATION_PASSWORD
          valueFrom:
            secretKeyRef:
              name: patroni-secret
              key: PATRONI_REPLICATION_PASSWORD
        - name: PATRONI_POSTGRESQL_DATA_DIR
          value: /var/lib/postgresql/data/pgdata
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        - name: config
          mountPath: /etc/patroni
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /liveness
            port: 8008
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /readiness
            port: 8008
          initialDelaySeconds: 10
          periodSeconds: 5
      volumes:
      - name: config
        configMap:
          name: patroni-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "standard"
      resources:
        requests:
          storage: 50Gi
```

## PostgreSQL Operators

### CloudNativePG (CNPG)

CloudNativePG — это современный оператор для PostgreSQL в Kubernetes, разработанный по принципам cloud-native.

```yaml
# Установка оператора
# kubectl apply -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/main/releases/cnpg-1.22.0.yaml

# cluster.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: my-postgres
  namespace: postgres
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:16.1

  primaryUpdateStrategy: unsupervised

  postgresql:
    parameters:
      max_connections: "200"
      shared_buffers: "256MB"
      effective_cache_size: "768MB"
      maintenance_work_mem: "64MB"
      checkpoint_completion_target: "0.9"
      wal_buffers: "16MB"
      random_page_cost: "1.1"
      effective_io_concurrency: "200"
      work_mem: "6553kB"
      max_wal_size: "4GB"
    pg_hba:
      - host all all 10.0.0.0/8 md5
      - host all all 172.16.0.0/12 md5
      - host all all 192.168.0.0/16 md5

  bootstrap:
    initdb:
      database: myapp
      owner: myapp
      secret:
        name: myapp-secret

  storage:
    size: 50Gi
    storageClass: standard

  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "4Gi"
      cpu: "2000m"

  # Резервное копирование
  backup:
    barmanObjectStore:
      destinationPath: s3://my-bucket/postgres-backup
      endpointURL: https://s3.amazonaws.com
      s3Credentials:
        accessKeyId:
          name: backup-s3-creds
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: backup-s3-creds
          key: ACCESS_SECRET_KEY
      wal:
        compression: gzip
      data:
        compression: gzip
    retentionPolicy: "30d"

  # Мониторинг
  monitoring:
    enablePodMonitor: true
---
# Scheduled Backup
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: my-postgres-backup
  namespace: postgres
spec:
  schedule: "0 0 * * *"  # Ежедневно в полночь
  cluster:
    name: my-postgres
  backupOwnerReference: cluster
---
# Secret для приложения
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secret
  namespace: postgres
type: kubernetes.io/basic-auth
stringData:
  username: myapp
  password: myapp-password
```

### Zalando Postgres Operator

```yaml
# Установка оператора
# kubectl apply -k github.com/zalando/postgres-operator/manifests

# PostgreSQL кластер
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: my-postgres-cluster
  namespace: postgres
spec:
  teamId: "myteam"
  volume:
    size: 50Gi
    storageClass: standard
  numberOfInstances: 3

  users:
    myapp:
      - superuser
      - createdb
    readonly: []

  databases:
    myapp: myapp  # database: owner

  postgresql:
    version: "16"
    parameters:
      max_connections: "200"
      shared_buffers: "256MB"
      log_statement: "all"

  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 4Gi

  patroni:
    initdb:
      encoding: "UTF8"
      locale: "en_US.UTF-8"
      data-checksums: "true"
    pg_hba:
      - host all all 0.0.0.0/0 md5
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 33554432

  sidecars:
    - name: exporter
      image: prometheuscommunity/postgres-exporter:latest
      ports:
        - name: exporter
          containerPort: 9187
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 200m
          memory: 256Mi
      env:
        - name: DATA_SOURCE_URI
          value: "127.0.0.1:5432/postgres?sslmode=disable"
        - name: DATA_SOURCE_USER
          value: $(POSTGRES_USER)
        - name: DATA_SOURCE_PASS
          value: $(POSTGRES_PASSWORD)
```

### CrunchyData PGO

```yaml
# Установка PGO
# kubectl apply -k https://github.com/CrunchyData/postgres-operator/config/default

# PostgresCluster
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: my-postgres
  namespace: postgres
spec:
  postgresVersion: 16

  instances:
    - name: instance1
      replicas: 3
      dataVolumeClaimSpec:
        accessModes:
          - "ReadWriteOnce"
        resources:
          requests:
            storage: 50Gi
        storageClassName: standard
      resources:
        limits:
          cpu: "2"
          memory: 4Gi
        requests:
          cpu: 500m
          memory: 1Gi

  users:
    - name: myapp
      databases:
        - myapp
      options: "SUPERUSER"

  backups:
    pgbackrest:
      repos:
        - name: repo1
          s3:
            bucket: my-backup-bucket
            endpoint: s3.amazonaws.com
            region: us-east-1
          schedules:
            full: "0 1 * * 0"      # Weekly full
            differential: "0 1 * * 1-6"  # Daily differential

  patroni:
    dynamicConfiguration:
      postgresql:
        parameters:
          max_connections: 200
          shared_buffers: 512MB
          work_mem: 16MB

  monitoring:
    pgmonitor:
      exporter:
        image: registry.developers.crunchydata.com/crunchydata/crunchy-postgres-exporter:ubi8-5.4.2-0
```

## Persistent Storage

### StorageClass примеры

```yaml
# AWS EBS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: postgres-fast
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "10000"
  throughput: "500"
  encrypted: "true"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
# GCP Persistent Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: postgres-fast
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
# Azure Managed Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: postgres-fast
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### Local Persistent Volume (для высокой производительности)

```yaml
# StorageClass для local volumes
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
---
# PersistentVolume
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-1
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/postgres-data
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - worker-node-1
```

## Networking

### Network Policies

```yaml
# Разрешить доступ к PostgreSQL только из определенных подов
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-network-policy
  namespace: postgres
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Разрешить из приложений в namespace: myapp
    - from:
        - namespaceSelector:
            matchLabels:
              name: myapp
          podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 5432
    # Разрешить репликацию между подами PostgreSQL
    - from:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
        - protocol: TCP
          port: 8008
  egress:
    # Разрешить исходящий трафик на DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
    # Разрешить репликацию
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
```

### Service для внешнего доступа

```yaml
# LoadBalancer (облачные провайдеры)
apiVersion: v1
kind: Service
metadata:
  name: postgres-external
  namespace: postgres
  annotations:
    # AWS NLB
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  ports:
    - port: 5432
      targetPort: 5432
  selector:
    app: postgres
    role: master
---
# NodePort (для тестирования)
apiVersion: v1
kind: Service
metadata:
  name: postgres-nodeport
  namespace: postgres
spec:
  type: NodePort
  ports:
    - port: 5432
      targetPort: 5432
      nodePort: 30432
  selector:
    app: postgres
    role: master
```

## Backup и Recovery

### Backup Job

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: postgres
spec:
  schedule: "0 2 * * *"  # Каждый день в 2:00
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: postgres:16-alpine
              command:
                - /bin/sh
                - -c
                - |
                  pg_dump -h $PGHOST -U $PGUSER -d $PGDATABASE -Fc > /backup/backup-$(date +%Y%m%d-%H%M%S).dump
                  # Загрузить в S3
                  aws s3 cp /backup/*.dump s3://my-bucket/postgres-backups/
                  # Удалить локальный файл
                  rm /backup/*.dump
              env:
                - name: PGHOST
                  value: postgres-master
                - name: PGUSER
                  value: postgres
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgres-secret
                      key: POSTGRES_PASSWORD
                - name: PGDATABASE
                  value: myapp
              volumeMounts:
                - name: backup-volume
                  mountPath: /backup
          restartPolicy: OnFailure
          volumes:
            - name: backup-volume
              emptyDir: {}
```

### Восстановление из бэкапа

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: postgres-restore
  namespace: postgres
spec:
  template:
    spec:
      containers:
        - name: restore
          image: postgres:16-alpine
          command:
            - /bin/sh
            - -c
            - |
              aws s3 cp s3://my-bucket/postgres-backups/backup-20240115.dump /restore/
              pg_restore -h $PGHOST -U $PGUSER -d $PGDATABASE -c /restore/backup-20240115.dump
          env:
            - name: PGHOST
              value: postgres-master
            - name: PGUSER
              value: postgres
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
            - name: PGDATABASE
              value: myapp
          volumeMounts:
            - name: restore-volume
              mountPath: /restore
      restartPolicy: OnFailure
      volumes:
        - name: restore-volume
          emptyDir: {}
```

## Monitoring

### ServiceMonitor для Prometheus

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: postgres-monitor
  namespace: postgres
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: postgres
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
---
# PodMonitor для CloudNativePG
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: cnpg-cluster-monitor
  namespace: postgres
spec:
  selector:
    matchLabels:
      cnpg.io/cluster: my-postgres
  podMetricsEndpoints:
    - port: metrics
```

### Prometheus Rules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: postgres-alerts
  namespace: postgres
spec:
  groups:
    - name: postgres
      rules:
        - alert: PostgresDown
          expr: pg_up == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: PostgreSQL is down
            description: PostgreSQL instance {{ $labels.instance }} is down

        - alert: PostgresReplicationLag
          expr: pg_replication_lag > 60
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: PostgreSQL replication lag is high
            description: Replication lag is {{ $value }} seconds

        - alert: PostgresConnectionsHigh
          expr: sum(pg_stat_activity_count) / pg_settings_max_connections > 0.8
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: PostgreSQL connections approaching limit
```

## Best Practices

### 1. Используйте операторы для production

```bash
# CloudNativePG (рекомендуется для новых проектов)
kubectl apply -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/main/releases/cnpg-1.22.0.yaml

# Или Zalando Postgres Operator (проверенный временем)
kubectl apply -k github.com/zalando/postgres-operator/manifests
```

### 2. Правильно настройте ресурсы

```yaml
resources:
  requests:
    memory: "1Gi"    # Минимум для shared_buffers
    cpu: "500m"
  limits:
    memory: "4Gi"    # shared_buffers + work_mem * max_connections
    cpu: "2000m"
```

### 3. Используйте Pod Disruption Budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
  namespace: postgres
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: postgres
```

### 4. Настройте Pod Anti-Affinity

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - postgres
          topologyKey: "kubernetes.io/hostname"
```

### 5. Используйте Init Containers для проверок

```yaml
initContainers:
  - name: init-permissions
    image: busybox
    command: ['sh', '-c', 'chown -R 999:999 /var/lib/postgresql/data']
    volumeMounts:
      - name: data
        mountPath: /var/lib/postgresql/data
```

## Типичные проблемы

### Проблема: Pod в CrashLoopBackOff

```bash
# Проверить логи
kubectl logs -n postgres postgres-0

# Часто проблема в правах на volume
# Решение: initContainer для chown
```

### Проблема: Нет места на диске

```bash
# Расширить PVC (если storageClass поддерживает)
kubectl patch pvc data-postgres-0 -n postgres -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'

# Удалить WAL файлы (опасно!)
kubectl exec -it postgres-0 -n postgres -- pg_archivecleanup /var/lib/postgresql/data/pg_wal 000000010000000000000010
```

### Проблема: Медленные запросы

```bash
# Проверить ресурсы
kubectl top pod -n postgres

# Проверить IOPS хранилища
kubectl exec -it postgres-0 -n postgres -- fio --name=test --size=1G --rw=randread --bs=4k
```

## Заключение

Развертывание PostgreSQL в Kubernetes требует:
- Выбора подходящего оператора (CloudNativePG, Zalando, CrunchyData)
- Правильной настройки хранилища с высоким IOPS
- Настройки бэкапов и мониторинга
- Понимания особенностей stateful приложений в Kubernetes

Рекомендации:
- Начните с оператора CloudNativePG для новых проектов
- Используйте SSD хранилище с низкой латентностью
- Настройте автоматические бэкапы с первого дня
- Мониторьте метрики через Prometheus/Grafana

---

[prev: 03-patroni](./03-patroni.md) | [next: 05-monitoring](./05-monitoring.md)
