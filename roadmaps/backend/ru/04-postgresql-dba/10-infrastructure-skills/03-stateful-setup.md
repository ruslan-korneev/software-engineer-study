# Stateful настройки PostgreSQL в контейнерах

[prev: 02-logical-replication](./02-logical-replication.md) | [next: 04-helm-operators](./04-helm-operators.md)

---

## Введение

Запуск PostgreSQL в контейнерах требует особого подхода к управлению состоянием (stateful workloads). В отличие от stateless-приложений, базы данных хранят критически важные данные, которые должны пережить перезапуск контейнера, обновления и даже сбои инфраструктуры.

## Основные принципы

### Stateless vs Stateful

```
Stateless (веб-сервер):           Stateful (PostgreSQL):
┌─────────────────────┐           ┌─────────────────────┐
│     Container       │           │     Container       │
│  ┌───────────────┐  │           │  ┌───────────────┐  │
│  │  Application  │  │           │  │   PostgreSQL  │  │
│  └───────────────┘  │           │  └───────┬───────┘  │
│    (нет данных)     │           │          │          │
└─────────────────────┘           │  ┌───────▼───────┐  │
                                  │  │   Data Dir    │  │
                                  │  └───────────────┘  │
                                  └─────────┬───────────┘
                                            │
                                  ┌─────────▼───────────┐
                                  │   Persistent        │
                                  │   Volume (PV)       │
                                  └─────────────────────┘
```

### Ключевые требования для PostgreSQL

1. **Персистентное хранилище** — данные должны сохраняться
2. **Стабильная сетевая идентичность** — для репликации
3. **Упорядоченное развертывание** — primary до replicas
4. **Graceful shutdown** — корректное завершение работы

## Docker: Базовая настройка

### Простой docker-compose.yml

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16
    container_name: postgres_db
    restart: unless-stopped
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: mydb
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
    driver: local
```

### Продвинутая конфигурация

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16
    container_name: postgres_db
    restart: unless-stopped
    user: "999:999"  # UID:GID для postgres
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
      POSTGRES_DB: mydb
      PGDATA: /var/lib/postgresql/data/pgdata
    secrets:
      - db_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgresql.conf:/etc/postgresql/postgresql.conf:ro
      - ./pg_hba.conf:/etc/postgresql/pg_hba.conf:ro
    command: >
      postgres
      -c config_file=/etc/postgresql/postgresql.conf
      -c hba_file=/etc/postgresql/pg_hba.conf
    ports:
      - "5432:5432"
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
    logging:
      driver: "json-file"
      options:
        max-size: "200m"
        max-file: "10"

secrets:
  db_password:
    file: ./secrets/db_password.txt

volumes:
  postgres_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/postgres
```

## Kubernetes: StatefulSet

### Основы StatefulSet

StatefulSet обеспечивает:
- Стабильные, уникальные сетевые идентификаторы
- Стабильное, персистентное хранилище
- Упорядоченное, graceful развертывание и масштабирование
- Упорядоченные, автоматизированные rolling updates

### Пример StatefulSet для PostgreSQL

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
data:
  POSTGRES_DB: mydb
  POSTGRES_USER: myuser

---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
stringData:
  POSTGRES_PASSWORD: mysecretpassword

---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  ports:
    - port: 5432
      name: postgres
  clusterIP: None  # Headless service
  selector:
    app: postgres

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
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
      terminationGracePeriodSeconds: 60
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
              name: postgres
          envFrom:
            - configMapRef:
                name: postgres-config
            - secretRef:
                name: postgres-secret
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          readinessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - myuser
                - -d
                - mydb
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - myuser
                - -d
                - mydb
            initialDelaySeconds: 30
            periodSeconds: 30
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 10Gi
```

### Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

Headless service создает DNS-записи для каждого pod:
- `postgres-0.postgres-headless.namespace.svc.cluster.local`
- `postgres-1.postgres-headless.namespace.svc.cluster.local`

## Persistent Volumes

### StorageClass для PostgreSQL

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: postgres-storage
provisioner: kubernetes.io/aws-ebs  # или другой провизионер
parameters:
  type: gp3
  iopsPerGB: "50"
  fsType: ext4
reclaimPolicy: Retain  # Не удалять данные при удалении PVC
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: postgres-storage
  resources:
    requests:
      storage: 100Gi
```

### Рекомендации по хранилищу

| Тип хранилища | Производительность | Надежность | Стоимость |
|---------------|-------------------|------------|-----------|
| Local SSD | Высокая | Низкая (нет репликации) | Низкая |
| Network SSD (gp3, pd-ssd) | Средняя-Высокая | Высокая | Средняя |
| Network HDD | Низкая | Высокая | Низкая |

## Репликация в Kubernetes

### Primary-Replica архитектура

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      initContainers:
        - name: init-postgres
          image: postgres:16
          command:
            - bash
            - -c
            - |
              set -ex
              [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
              ordinal=${BASH_REMATCH[1]}
              if [[ $ordinal -eq 0 ]]; then
                echo "Primary instance"
                # Primary initialization
              else
                echo "Replica instance"
                # Wait for primary and setup streaming replication
                until pg_isready -h postgres-0.postgres -p 5432; do
                  echo "Waiting for primary..."
                  sleep 2
                done
                pg_basebackup -h postgres-0.postgres -D /var/lib/postgresql/data -U replication -v -P -W
              fi
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
      containers:
        - name: postgres
          image: postgres:16
          # ... остальная конфигурация
```

### Сервисы для Primary и Replicas

```yaml
# Сервис для записи (primary)
apiVersion: v1
kind: Service
metadata:
  name: postgres-primary
spec:
  selector:
    app: postgres
    role: primary
  ports:
    - port: 5432

---
# Сервис для чтения (replicas)
apiVersion: v1
kind: Service
metadata:
  name: postgres-replicas
spec:
  selector:
    app: postgres
    role: replica
  ports:
    - port: 5432
```

## Бэкапы в контейнерах

### CronJob для бэкапов

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
spec:
  schedule: "0 2 * * *"  # Каждый день в 2:00
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: postgres:16
              command:
                - /bin/bash
                - -c
                - |
                  set -e
                  BACKUP_FILE="/backups/backup-$(date +%Y%m%d-%H%M%S).sql.gz"
                  pg_dumpall -h postgres -U postgres | gzip > $BACKUP_FILE
                  echo "Backup completed: $BACKUP_FILE"
                  # Удаление старых бэкапов (старше 7 дней)
                  find /backups -name "backup-*.sql.gz" -mtime +7 -delete
              env:
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgres-secret
                      key: POSTGRES_PASSWORD
              volumeMounts:
                - name: backup-volume
                  mountPath: /backups
          restartPolicy: OnFailure
          volumes:
            - name: backup-volume
              persistentVolumeClaim:
                claimName: postgres-backup-pvc
```

### WAL архивирование

```yaml
# ConfigMap с настройками WAL архивирования
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-wal-config
data:
  postgresql.conf: |
    archive_mode = on
    archive_command = 'cp %p /wal-archive/%f'
    archive_timeout = 60
```

## Мониторинг и Health Checks

### Liveness и Readiness Probes

```yaml
containers:
  - name: postgres
    livenessProbe:
      exec:
        command:
          - pg_isready
          - -U
          - postgres
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 6
    readinessProbe:
      exec:
        command:
          - psql
          - -U
          - postgres
          - -c
          - "SELECT 1"
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
    startupProbe:
      exec:
        command:
          - pg_isready
          - -U
          - postgres
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 30
```

### Prometheus метрики

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  template:
    spec:
      containers:
        - name: postgres
          # ... основной контейнер
        - name: postgres-exporter
          image: prometheuscommunity/postgres-exporter:latest
          ports:
            - containerPort: 9187
              name: metrics
          env:
            - name: DATA_SOURCE_NAME
              value: "postgresql://postgres:password@localhost:5432/postgres?sslmode=disable"
```

## Best Practices

### Безопасность

```yaml
# Использование секретов
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
data:
  password: base64-encoded-password

# Security Context
securityContext:
  runAsUser: 999
  runAsGroup: 999
  fsGroup: 999
  runAsNonRoot: true

# Network Policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-network-policy
spec:
  podSelector:
    matchLabels:
      app: postgres
  ingress:
    - from:
        - podSelector:
            matchLabels:
              access: postgres
      ports:
        - port: 5432
```

### Ресурсы и производительность

```yaml
resources:
  requests:
    memory: "2Gi"
    cpu: "1000m"
  limits:
    memory: "4Gi"
    cpu: "2000m"

# Quality of Service: Guaranteed
# Для этого requests == limits
```

### Graceful Shutdown

```yaml
spec:
  terminationGracePeriodSeconds: 120  # Достаточно времени для checkpoint
  containers:
    - name: postgres
      lifecycle:
        preStop:
          exec:
            command:
              - /bin/sh
              - -c
              - "pg_ctl stop -D $PGDATA -m fast"
```

## Типичные ошибки

### 1. Потеря данных при перезапуске

```yaml
# Неправильно: ephemeral storage
volumes:
  - name: data
    emptyDir: {}

# Правильно: persistent storage
volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### 2. Неправильный PGDATA

```yaml
# PGDATA должен быть в подкаталоге тома
env:
  - name: PGDATA
    value: /var/lib/postgresql/data/pgdata  # НЕ /var/lib/postgresql/data
```

### 3. Отсутствие health checks

```yaml
# Всегда добавляйте probes
readinessProbe:
  exec:
    command: ["pg_isready"]
```

### 4. Недостаточный terminationGracePeriodSeconds

```yaml
# PostgreSQL нужно время для checkpoint
terminationGracePeriodSeconds: 60  # Минимум 30 секунд
```

## Миграция существующей БД в контейнер

### Шаги миграции

```bash
# 1. Создать дамп существующей БД
pg_dumpall -h old_server -U postgres > full_backup.sql

# 2. Запустить контейнер с PostgreSQL
docker run -d --name postgres-new \
  -v postgres_data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=mysecretpassword \
  postgres:16

# 3. Восстановить данные
docker exec -i postgres-new psql -U postgres < full_backup.sql

# 4. Проверить данные
docker exec -it postgres-new psql -U postgres -c "\l"
```

## Полезные ссылки

- [Docker Hub: postgres](https://hub.docker.com/_/postgres)
- [Kubernetes StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [PostgreSQL on Kubernetes](https://www.postgresql.org/docs/current/high-availability.html)

---

[prev: 02-logical-replication](./02-logical-replication.md) | [next: 04-helm-operators](./04-helm-operators.md)
