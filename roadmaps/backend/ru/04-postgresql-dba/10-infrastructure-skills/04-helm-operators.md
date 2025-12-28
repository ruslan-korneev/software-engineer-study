# Helm и Kubernetes Operators для PostgreSQL

[prev: 03-stateful-setup](./03-stateful-setup.md) | [next: 05-monitoring-tools](./05-monitoring-tools.md)

---

## Введение

Управление PostgreSQL в Kubernetes можно значительно упростить с помощью Helm charts и Kubernetes Operators. Эти инструменты автоматизируют развертывание, настройку, обновление и обслуживание баз данных.

## Helm для PostgreSQL

### Что такое Helm

**Helm** — это пакетный менеджер для Kubernetes, который позволяет:
- Определять, устанавливать и обновлять Kubernetes-приложения
- Управлять сложными зависимостями
- Использовать шаблонизацию для конфигурации
- Версионировать релизы и делать rollback

### Установка Helm

```bash
# macOS
brew install helm

# Linux (snap)
snap install helm --classic

# Linux (script)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Проверка установки
helm version
```

### Bitnami PostgreSQL Chart

#### Добавление репозитория

```bash
# Добавить репозиторий Bitnami
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

#### Базовая установка

```bash
# Установка с настройками по умолчанию
helm install my-postgres bitnami/postgresql

# Установка с пользовательскими параметрами
helm install my-postgres bitnami/postgresql \
  --set auth.postgresPassword=mysecretpassword \
  --set auth.database=mydb \
  --set primary.persistence.size=20Gi
```

#### Использование values.yaml

```yaml
# values.yaml
global:
  postgresql:
    auth:
      postgresPassword: "mysecretpassword"
      username: "myuser"
      password: "myuserpassword"
      database: "mydb"

primary:
  persistence:
    enabled: true
    size: 50Gi
    storageClass: "fast-storage"

  resources:
    limits:
      memory: 4Gi
      cpu: 2
    requests:
      memory: 2Gi
      cpu: 1

  extendedConfiguration: |
    max_connections = 200
    shared_buffers = 1GB
    effective_cache_size = 3GB
    work_mem = 16MB
    maintenance_work_mem = 512MB

readReplicas:
  replicaCount: 2
  persistence:
    enabled: true
    size: 50Gi

metrics:
  enabled: true
  serviceMonitor:
    enabled: true

backup:
  enabled: true
  cronjob:
    schedule: "0 2 * * *"
    storage:
      size: 100Gi
```

```bash
# Установка с файлом values
helm install my-postgres bitnami/postgresql -f values.yaml
```

### Управление релизами Helm

```bash
# Список установленных релизов
helm list

# Обновление релиза
helm upgrade my-postgres bitnami/postgresql -f values.yaml

# Откат к предыдущей версии
helm rollback my-postgres 1

# Удаление релиза
helm uninstall my-postgres

# Получение информации о релизе
helm status my-postgres

# Получение values текущего релиза
helm get values my-postgres
```

### CloudNativePG Helm Chart

```bash
# Добавить репозиторий CloudNativePG
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update

# Установка оператора
helm install cnpg cnpg/cloudnative-pg

# Создание кластера PostgreSQL
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: my-postgres-cluster
spec:
  instances: 3
  storage:
    size: 10Gi
EOF
```

## Kubernetes Operators

### Что такое Operator

**Kubernetes Operator** — это паттерн, расширяющий Kubernetes API для управления сложными приложениями. Operator использует Custom Resources (CR) и контроллеры для автоматизации операций, которые обычно выполняет человек.

### Принцип работы Operator

```
┌─────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                  │
│                                                      │
│  ┌──────────────┐       ┌───────────────────────┐   │
│  │ Custom       │       │     Operator          │   │
│  │ Resource     │◀─────▶│   Controller          │   │
│  │ (Cluster)    │       │                       │   │
│  └──────────────┘       │ - Watch CRs           │   │
│                         │ - Reconcile state     │   │
│                         │ - Manage StatefulSets │   │
│                         │ - Handle failover     │   │
│                         │ - Manage backups      │   │
│                         └───────────────────────┘   │
│                                  │                   │
│                                  ▼                   │
│  ┌──────────────────────────────────────────────┐   │
│  │           PostgreSQL Pods                     │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐    │   │
│  │  │ Primary  │  │ Replica  │  │ Replica  │    │   │
│  │  └──────────┘  └──────────┘  └──────────┘    │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### Популярные PostgreSQL Operators

| Operator | Разработчик | Особенности |
|----------|-------------|-------------|
| CloudNativePG | CNCF Sandbox | Нативный для Kubernetes, активная разработка |
| Zalando Postgres Operator | Zalando | Зрелый, поддержка Patroni |
| CrunchyData PGO | Crunchy Data | Enterprise-функции, мониторинг |
| Percona Operator | Percona | Резервное копирование, мониторинг |

## CloudNativePG

### Установка

```bash
# Установка через kubectl
kubectl apply -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.22/releases/cnpg-1.22.0.yaml

# Или через Helm
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm install cnpg cnpg/cloudnative-pg -n cnpg-system --create-namespace
```

### Создание кластера

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: my-cluster
  namespace: default
spec:
  instances: 3

  imageName: ghcr.io/cloudnative-pg/postgresql:16.1

  postgresql:
    parameters:
      max_connections: "200"
      shared_buffers: "256MB"
      effective_cache_size: "768MB"
      work_mem: "16MB"
      maintenance_work_mem: "128MB"
      log_statement: "ddl"

  bootstrap:
    initdb:
      database: mydb
      owner: myuser
      secret:
        name: my-cluster-user-secret

  storage:
    size: 10Gi
    storageClass: standard

  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "1"

  monitoring:
    enablePodMonitor: true

  backup:
    barmanObjectStore:
      destinationPath: s3://my-bucket/backups
      s3Credentials:
        accessKeyId:
          name: s3-creds
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: s3-creds
          key: SECRET_ACCESS_KEY
    retentionPolicy: "30d"

---
apiVersion: v1
kind: Secret
metadata:
  name: my-cluster-user-secret
type: kubernetes.io/basic-auth
stringData:
  username: myuser
  password: mypassword
```

### Резервное копирование

```yaml
# Scheduled backup
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: my-cluster-backup
spec:
  schedule: "0 0 * * *"
  backupOwnerReference: self
  cluster:
    name: my-cluster

---
# On-demand backup
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: my-cluster-backup-now
spec:
  cluster:
    name: my-cluster
```

### Восстановление

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: my-cluster-restored
spec:
  instances: 3

  storage:
    size: 10Gi

  bootstrap:
    recovery:
      source: my-cluster
      # Point-in-time recovery
      recoveryTarget:
        targetTime: "2024-01-15T10:00:00Z"

  externalClusters:
    - name: my-cluster
      barmanObjectStore:
        destinationPath: s3://my-bucket/backups
        s3Credentials:
          accessKeyId:
            name: s3-creds
            key: ACCESS_KEY_ID
          secretAccessKey:
            name: s3-creds
            key: SECRET_ACCESS_KEY
```

## Zalando Postgres Operator

### Установка

```bash
# Клонировать репозиторий
git clone https://github.com/zalando/postgres-operator.git
cd postgres-operator

# Установка с ConfigMap
kubectl create -f manifests/configmap.yaml
kubectl create -f manifests/operator-service-account-rbac.yaml
kubectl create -f manifests/postgres-operator.yaml

# Или через Helm
helm repo add postgres-operator-charts https://opensource.zalando.com/postgres-operator/charts/postgres-operator
helm install postgres-operator postgres-operator-charts/postgres-operator
```

### Создание кластера

```yaml
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: acid-minimal-cluster
  namespace: default
spec:
  teamId: "acid"
  volume:
    size: 10Gi
    storageClass: standard
  numberOfInstances: 3

  users:
    myuser:
      - superuser
      - createdb
    app_user: []

  databases:
    mydb: myuser

  postgresql:
    version: "16"
    parameters:
      max_connections: "200"
      shared_buffers: "256MB"

  resources:
    requests:
      cpu: 500m
      memory: 500Mi
    limits:
      cpu: "1"
      memory: 1Gi

  patroni:
    initdb:
      encoding: "UTF8"
      locale: "en_US.UTF-8"
    pg_hba:
      - host all all 0.0.0.0/0 md5
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 33554432

  enableLogicalBackup: true
  logicalBackupSchedule: "0 0 * * *"
```

### Управление пользователями

```yaml
spec:
  users:
    # Суперпользователь с правом создания БД
    admin:
      - superuser
      - createdb

    # Обычный пользователь приложения
    app_user:
      - login

    # Пользователь только для чтения
    readonly_user:
      - login

  databases:
    production_db: admin
    analytics_db: admin

  # Дополнительные права
  preparedDatabases:
    production_db:
      schemas:
        public:
          defaultUsers: true
          defaultRoles: true
```

## Crunchy Data PGO

### Установка

```bash
# Через kubectl
kubectl apply -k https://github.com/CrunchyData/postgres-operator-examples/kustomize/install

# Через Helm
helm repo add crunchydata https://crunchydata.github.io/postgres-operator-examples/helm/install
helm install pgo crunchydata/pgo
```

### Создание кластера

```yaml
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: hippo
spec:
  image: registry.developers.crunchydata.com/crunchydata/crunchy-postgres:ubi8-16.1-0
  postgresVersion: 16

  instances:
    - name: instance1
      replicas: 3
      dataVolumeClaimSpec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
      resources:
        limits:
          cpu: "2"
          memory: "4Gi"

  users:
    - name: hippo
      databases:
        - hippo
      options: "CREATEROLE"

  backups:
    pgbackrest:
      image: registry.developers.crunchydata.com/crunchydata/crunchy-pgbackrest:ubi8-2.47-2
      repos:
        - name: repo1
          schedules:
            full: "0 1 * * 0"
            differential: "0 1 * * 1-6"
          volume:
            volumeClaimSpec:
              accessModes:
                - ReadWriteOnce
              resources:
                requests:
                  storage: 50Gi

  monitoring:
    pgmonitor:
      exporter:
        image: registry.developers.crunchydata.com/crunchydata/crunchy-postgres-exporter:ubi8-5.5.0-0
```

## Сравнение Operators

| Функция | CloudNativePG | Zalando | CrunchyData |
|---------|---------------|---------|-------------|
| High Availability | Да | Да (Patroni) | Да (Patroni) |
| Автоматический failover | Да | Да | Да |
| Point-in-time recovery | Да | Да | Да |
| S3 бэкапы | Да | Да | Да |
| Мониторинг | PodMonitor | Нет | Встроенный |
| Connection pooling | Нет (внешний) | Нет | PgBouncer |
| Лицензия | Apache 2.0 | MIT | Apache 2.0 |

## Best Practices

### Выбор Operator

1. **CloudNativePG** — для новых проектов, активная разработка, CNCF
2. **Zalando** — если уже используете Patroni
3. **CrunchyData** — для enterprise с поддержкой

### Безопасность

```yaml
# Использование секретов для паролей
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
type: Opaque
stringData:
  username: admin
  password: $(openssl rand -base64 32)

# Network Policies
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-network-policy
spec:
  podSelector:
    matchLabels:
      cnpg.io/cluster: my-cluster
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: my-app
      ports:
        - port: 5432
```

### Мониторинг

```yaml
# ServiceMonitor для Prometheus
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: postgres-metrics
spec:
  selector:
    matchLabels:
      cnpg.io/cluster: my-cluster
  endpoints:
    - port: metrics
      interval: 30s
```

### Резервное копирование

```yaml
# Настройка retention policy
backup:
  barmanObjectStore:
    destinationPath: s3://my-bucket/backups
    retentionPolicy: "30d"  # Хранить 30 дней
    wal:
      compression: gzip
```

## Типичные ошибки

### 1. Отсутствие резервных копий

```yaml
# Всегда настраивайте автоматические бэкапы
backup:
  barmanObjectStore:
    destinationPath: s3://backups
  retentionPolicy: "7d"
```

### 2. Недостаточное количество реплик

```yaml
# Минимум 3 инстанса для HA
spec:
  instances: 3  # Не 1 или 2
```

### 3. Отсутствие resource limits

```yaml
# Всегда указывайте лимиты
resources:
  requests:
    memory: "1Gi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "1"
```

### 4. Игнорирование мониторинга

```yaml
# Включайте мониторинг
monitoring:
  enablePodMonitor: true
```

## Полезные команды

```bash
# CloudNativePG
kubectl cnpg status my-cluster
kubectl cnpg promote my-cluster my-cluster-2

# Zalando
kubectl get postgresql
kubectl describe postgresql acid-minimal-cluster

# Логи оператора
kubectl logs -n cnpg-system deployment/cnpg-controller-manager

# Подключение к БД
kubectl cnpg psql my-cluster -- -c "SELECT version();"
```

## Полезные ссылки

- [Helm Documentation](https://helm.sh/docs/)
- [CloudNativePG](https://cloudnative-pg.io/)
- [Zalando Postgres Operator](https://github.com/zalando/postgres-operator)
- [CrunchyData PGO](https://access.crunchydata.com/documentation/postgres-operator/)
- [Bitnami PostgreSQL Chart](https://github.com/bitnami/charts/tree/main/bitnami/postgresql)

---

[prev: 03-stateful-setup](./03-stateful-setup.md) | [next: 05-monitoring-tools](./05-monitoring-tools.md)
