# Запуск приложений в Kubernetes

Kubernetes предоставляет различные типы ресурсов (workloads) для запуска приложений. Каждый тип решает определённые задачи: от простого запуска контейнеров до управления stateful-приложениями и выполнения периодических задач.

---

## 1. Pods

**Pod** — минимальная единица развёртывания в Kubernetes. Это группа из одного или нескольких контейнеров, которые:
- Разделяют сетевое пространство (один IP-адрес)
- Могут использовать общие volumes
- Запускаются и останавливаются вместе

### Жизненный цикл Pod

Pod проходит через следующие фазы:

| Фаза | Описание |
|------|----------|
| `Pending` | Pod принят кластером, но контейнеры ещё не запущены (скачивание образов, ожидание ресурсов) |
| `Running` | Pod привязан к узлу, все контейнеры созданы, минимум один работает |
| `Succeeded` | Все контейнеры успешно завершились и не будут перезапущены |
| `Failed` | Все контейнеры завершились, минимум один с ошибкой |
| `Unknown` | Состояние Pod не может быть определено (проблемы с узлом) |

### Состояния контейнеров

Каждый контейнер внутри Pod имеет своё состояние:

- **Waiting** — контейнер ожидает запуска (pull образа, init-контейнеры)
- **Running** — контейнер выполняется
- **Terminated** — контейнер завершил работу (успешно или с ошибкой)

### Политики перезапуска (restartPolicy)

```yaml
spec:
  restartPolicy: Always  # Always | OnFailure | Never
```

- `Always` (по умолчанию) — всегда перезапускать контейнеры
- `OnFailure` — перезапускать только при ненулевом exit code
- `Never` — никогда не перезапускать

### Пример простого Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    environment: development
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### Multi-container Pods

Несколько контейнеров в одном Pod используются для паттернов:

#### Sidecar Pattern
Дополнительный контейнер расширяет функциональность основного:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-logging
spec:
  containers:
  # Основной контейнер
  - name: web-app
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx

  # Sidecar для сбора логов
  - name: log-collector
    image: fluent/fluent-bit:latest
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
      readOnly: true

  volumes:
  - name: logs
    emptyDir: {}
```

#### Ambassador Pattern
Прокси для внешних сервисов:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-ambassador
spec:
  containers:
  - name: main-app
    image: myapp:1.0
    env:
    - name: DB_HOST
      value: "localhost"  # Обращается к ambassador
    - name: DB_PORT
      value: "5432"

  # Ambassador для подключения к внешней БД
  - name: db-ambassador
    image: haproxy:2.8
    ports:
    - containerPort: 5432
```

#### Adapter Pattern
Преобразование данных в стандартный формат:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-adapter
spec:
  containers:
  - name: legacy-app
    image: legacy-app:1.0
    volumeMounts:
    - name: metrics
      mountPath: /var/metrics

  # Adapter преобразует метрики в формат Prometheus
  - name: metrics-adapter
    image: prom/statsd-exporter:latest
    ports:
    - containerPort: 9102
    volumeMounts:
    - name: metrics
      mountPath: /var/metrics
      readOnly: true

  volumes:
  - name: metrics
    emptyDir: {}
```

### Init Containers

Init-контейнеры выполняются **до** основных контейнеров (последовательно):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
  # Ожидание готовности базы данных
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nc -z postgres-svc 5432; do echo waiting...; sleep 2; done']

  # Миграции базы данных
  - name: run-migrations
    image: myapp:1.0
    command: ['./migrate', 'up']
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: url

  containers:
  - name: main-app
    image: myapp:1.0
    ports:
    - containerPort: 8080
```

### Probes (проверки здоровья)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-probes
spec:
  containers:
  - name: app
    image: myapp:1.0
    ports:
    - containerPort: 8080

    # Проверка готовности принимать трафик
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 3

    # Проверка жизнеспособности (перезапуск при неудаче)
    livenessProbe:
      httpGet:
        path: /health/live
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
      failureThreshold: 3

    # Проверка старта (для медленно стартующих приложений)
    startupProbe:
      httpGet:
        path: /health/startup
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
```

### Команды kubectl для работы с Pods

```bash
# Создание Pod из манифеста
kubectl apply -f pod.yaml

# Список Pods
kubectl get pods
kubectl get pods -o wide  # С дополнительной информацией
kubectl get pods -w       # Отслеживание изменений в реальном времени

# Детальная информация
kubectl describe pod nginx-pod

# Логи
kubectl logs nginx-pod
kubectl logs nginx-pod -c sidecar  # Логи конкретного контейнера
kubectl logs nginx-pod --previous  # Логи предыдущего экземпляра
kubectl logs -f nginx-pod          # Streaming логов

# Выполнение команды в контейнере
kubectl exec -it nginx-pod -- /bin/bash
kubectl exec -it nginx-pod -c sidecar -- /bin/sh  # В конкретном контейнере

# Port forwarding
kubectl port-forward nginx-pod 8080:80

# Копирование файлов
kubectl cp nginx-pod:/var/log/nginx/access.log ./access.log

# Удаление
kubectl delete pod nginx-pod
kubectl delete pod nginx-pod --grace-period=0 --force  # Принудительное удаление
```

---

## 2. ReplicaSet

**ReplicaSet** гарантирует запуск указанного количества идентичных Pods. Это замена устаревшего ReplicationController.

### Как работает ReplicaSet

1. Наблюдает за Pods с определёнными labels
2. Сравнивает текущее количество с желаемым (replicas)
3. Создаёт или удаляет Pods для достижения желаемого состояния

### Пример ReplicaSet

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  labels:
    app: nginx
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

### Selector с matchExpressions

```yaml
spec:
  selector:
    matchLabels:
      app: nginx
    matchExpressions:
    - key: environment
      operator: In
      values:
      - production
      - staging
    - key: version
      operator: NotIn
      values:
      - v1
```

Операторы: `In`, `NotIn`, `Exists`, `DoesNotExist`

### Команды kubectl для ReplicaSet

```bash
# Создание
kubectl apply -f replicaset.yaml

# Просмотр
kubectl get replicasets
kubectl get rs  # Сокращённо
kubectl describe rs nginx-replicaset

# Масштабирование
kubectl scale rs nginx-replicaset --replicas=5

# Удаление
kubectl delete rs nginx-replicaset
kubectl delete rs nginx-replicaset --cascade=orphan  # Оставить Pods
```

### Когда использовать ReplicaSet напрямую

**Почти никогда!** Используйте Deployment, который управляет ReplicaSet и добавляет:
- Rolling updates
- Rollback
- История версий

ReplicaSet напрямую нужен только для:
- Кастомной логики оркестрации
- Если не нужны обновления вообще

---

## 3. Deployment

**Deployment** — рекомендуемый способ управления stateless-приложениями. Предоставляет:
- Декларативное обновление Pods
- Rolling updates и rollback
- Масштабирование
- Pause/resume обновлений

### Пример Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Стратегии обновления

#### RollingUpdate (по умолчанию)

Постепенная замена старых Pods новыми:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%  # Максимум недоступных (абсолютное число или %)
      maxSurge: 25%        # Максимум дополнительных Pods
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:2.0
```

**Как работает при 10 репликах и maxSurge=25%, maxUnavailable=25%:**
1. Можно создать до 12 Pods (10 + 25%)
2. Минимум 8 Pods должны быть доступны (10 - 25%)
3. Kubernetes создаёт новые Pods, ждёт readiness, затем удаляет старые

#### Recreate

Удаление всех старых Pods перед созданием новых:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
spec:
  replicas: 3
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: myapp
  template:
    # ...
```

**Когда использовать Recreate:**
- Приложение не поддерживает несколько версий одновременно
- Эксклюзивный доступ к ресурсам (volume в режиме RWO)
- Допустим простой при обновлении

### Расширенный пример Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app
  labels:
    app: myapp
    environment: production
  annotations:
    kubernetes.io/change-cause: "Update to version 2.1.0"
spec:
  replicas: 5
  revisionHistoryLimit: 10  # Количество хранимых ReplicaSets для rollback
  progressDeadlineSeconds: 600  # Таймаут для обновления
  minReadySeconds: 30  # Время после ready до считания available

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 2

  selector:
    matchLabels:
      app: myapp

  template:
    metadata:
      labels:
        app: myapp
        version: v2.1.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: myapp
              topologyKey: kubernetes.io/hostname

      containers:
      - name: app
        image: myapp:2.1.0
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 8080
        - name: metrics
          containerPort: 9090

        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url

        envFrom:
        - configMapRef:
            name: app-config

        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"

        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          successThreshold: 1
          failureThreshold: 3

        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3

        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
        - name: tmp
          mountPath: /tmp

      volumes:
      - name: config
        configMap:
          name: app-config-files
      - name: tmp
        emptyDir: {}

      terminationGracePeriodSeconds: 60
```

### Команды kubectl для Deployment

```bash
# Создание и обновление
kubectl apply -f deployment.yaml

# Просмотр
kubectl get deployments
kubectl get deploy  # Сокращённо
kubectl describe deployment nginx-deployment

# Масштабирование
kubectl scale deployment nginx-deployment --replicas=5

# Автомасштабирование
kubectl autoscale deployment nginx-deployment --min=3 --max=10 --cpu-percent=80

# Обновление образа
kubectl set image deployment/nginx-deployment nginx=nginx:1.26
kubectl set image deployment/nginx-deployment nginx=nginx:1.26 --record  # С записью причины

# Обновление ресурсов
kubectl set resources deployment/nginx-deployment -c=nginx --limits=cpu=200m,memory=512Mi

# Статус обновления
kubectl rollout status deployment/nginx-deployment

# История обновлений
kubectl rollout history deployment/nginx-deployment
kubectl rollout history deployment/nginx-deployment --revision=2

# Откат
kubectl rollout undo deployment/nginx-deployment
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# Пауза/возобновление
kubectl rollout pause deployment/nginx-deployment
kubectl rollout resume deployment/nginx-deployment

# Перезапуск всех Pods
kubectl rollout restart deployment/nginx-deployment
```

### Blue-Green и Canary Deployments

Kubernetes не имеет встроенной поддержки, но можно реализовать:

#### Blue-Green через Service

```yaml
# Blue deployment (текущая версия)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: app
        image: myapp:1.0
---
# Green deployment (новая версия)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: app
        image: myapp:2.0
---
# Service переключается между blue и green
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    version: blue  # Изменить на green для переключения
  ports:
  - port: 80
    targetPort: 8080
```

#### Canary через несколько Deployments

```yaml
# Stable deployment (90% трафика)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      track: stable
  template:
    metadata:
      labels:
        app: myapp
        track: stable
    spec:
      containers:
      - name: app
        image: myapp:1.0
---
# Canary deployment (10% трафика)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      track: canary
  template:
    metadata:
      labels:
        app: myapp
        track: canary
    spec:
      containers:
      - name: app
        image: myapp:2.0
---
# Service направляет на оба
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp  # Без track — охватывает оба
  ports:
  - port: 80
    targetPort: 8080
```

---

## 4. StatefulSet

**StatefulSet** — для stateful-приложений, требующих:
- Стабильные сетевые идентификаторы
- Стабильное персистентное хранилище
- Упорядоченное развёртывание и масштабирование
- Упорядоченные, graceful обновления

### Отличия от Deployment

| Аспект | Deployment | StatefulSet |
|--------|------------|-------------|
| Имена Pods | Случайный суффикс (app-7d4f9c5b-x2k) | Порядковый индекс (app-0, app-1, app-2) |
| Хранилище | Общий или без состояния | Индивидуальный PVC для каждого Pod |
| Порядок создания | Параллельно | Последовательно (0, 1, 2...) |
| Порядок удаления | Параллельно | Обратный (2, 1, 0...) |
| DNS | Только через Service | Индивидуальный DNS для каждого Pod |

### Пример StatefulSet

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
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
  serviceName: postgres-headless  # Обязательно для StatefulSet
  replicas: 3
  selector:
    matchLabels:
      app: postgres

  template:
    metadata:
      labels:
        app: postgres
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
          name: postgres

        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata

        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data

        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 5
          periodSeconds: 10

  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 10Gi
```

### DNS-имена в StatefulSet

Каждый Pod получает стабильное DNS-имя:
```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

Для примера выше:
- `postgres-0.postgres-headless.default.svc.cluster.local`
- `postgres-1.postgres-headless.default.svc.cluster.local`
- `postgres-2.postgres-headless.default.svc.cluster.local`

### Политики управления Pods

```yaml
spec:
  podManagementPolicy: OrderedReady  # OrderedReady | Parallel
```

- **OrderedReady** (по умолчанию): последовательное создание и удаление
- **Parallel**: параллельное создание (полезно для распределённых систем)

### Стратегии обновления StatefulSet

```yaml
spec:
  updateStrategy:
    type: RollingUpdate  # RollingUpdate | OnDelete
    rollingUpdate:
      partition: 2  # Обновлять только Pods с индексом >= partition
      maxUnavailable: 1  # Kubernetes 1.24+
```

**Partition** — для канареечных обновлений:
- `partition: 2` при 3 репликах обновит только `app-2`
- После проверки уменьшить partition для обновления остальных

### Пример: Redis Cluster

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
data:
  redis.conf: |
    cluster-enabled yes
    cluster-config-file nodes.conf
    cluster-node-timeout 5000
    appendonly yes
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  serviceName: redis-cluster
  replicas: 6
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      containers:
      - name: redis
        image: redis:7
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        command: ["redis-server", "/conf/redis.conf"]
        volumeMounts:
        - name: conf
          mountPath: /conf
        - name: data
          mountPath: /data
      volumes:
      - name: conf
        configMap:
          name: redis-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
```

### Команды kubectl для StatefulSet

```bash
# Создание
kubectl apply -f statefulset.yaml

# Просмотр
kubectl get statefulsets
kubectl get sts  # Сокращённо
kubectl describe sts postgres

# Масштабирование
kubectl scale sts postgres --replicas=5

# Обновление (с partition)
kubectl patch sts postgres -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'

# Rollout (аналогично Deployment)
kubectl rollout status sts postgres
kubectl rollout history sts postgres

# Удаление
kubectl delete sts postgres
kubectl delete sts postgres --cascade=orphan  # Оставить Pods и PVCs
```

---

## 5. DaemonSet

**DaemonSet** гарантирует запуск копии Pod на каждом (или выбранных) узле кластера.

### Use Cases

- **Сбор логов**: Fluentd, Filebeat
- **Мониторинг**: Node Exporter, Datadog Agent
- **Сетевые плагины**: Calico, Weave, Cilium
- **Storage daemons**: Ceph, Gluster

### Пример DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true

      tolerations:
      # Запуск на всех узлах, включая master
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule

      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.6.1
        ports:
        - containerPort: 9100
          hostPort: 9100

        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root

        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: root
          mountPath: /host/root
          readOnly: true

        resources:
          limits:
            memory: 180Mi
          requests:
            cpu: 102m
            memory: 180Mi

      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /
```

### Запуск на определённых узлах

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: gpu-driver
spec:
  selector:
    matchLabels:
      app: gpu-driver
  template:
    metadata:
      labels:
        app: gpu-driver
    spec:
      nodeSelector:
        gpu: nvidia  # Только узлы с этим label

      containers:
      - name: nvidia-driver
        image: nvidia/driver:525
```

Или через affinity:

```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-type
                operator: In
                values:
                - gpu
                - high-memory
```

### Стратегия обновления DaemonSet

```yaml
spec:
  updateStrategy:
    type: RollingUpdate  # RollingUpdate | OnDelete
    rollingUpdate:
      maxUnavailable: 1  # Или процент: "10%"
      maxSurge: 0  # По умолчанию 0, можно увеличить в 1.22+
```

### Пример: Fluentd для сбора логов

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccountName: fluentd
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule

      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.16-debian-elasticsearch8

        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch.logging.svc.cluster.local"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
        - name: FLUENT_CONTAINER_TAIL_PARSER_TYPE
          value: "cri"

        resources:
          limits:
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 200Mi

        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: containers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluentd-config
          mountPath: /fluentd/etc/conf.d

      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: containers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluentd-config
        configMap:
          name: fluentd-config
```

### Команды kubectl для DaemonSet

```bash
# Создание
kubectl apply -f daemonset.yaml

# Просмотр
kubectl get daemonsets
kubectl get ds  # Сокращённо
kubectl describe ds node-exporter

# Статус rollout
kubectl rollout status ds/node-exporter
kubectl rollout history ds/node-exporter

# Обновление
kubectl set image ds/node-exporter node-exporter=prom/node-exporter:v1.7.0

# Откат
kubectl rollout undo ds/node-exporter

# Удаление
kubectl delete ds node-exporter
```

---

## 6. Jobs и CronJobs

### Job

**Job** — одноразовая задача, которая выполняется до успешного завершения.

#### Простой Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-database
spec:
  template:
    spec:
      containers:
      - name: backup
        image: postgres:15
        command: ["pg_dump"]
        args:
        - -h
        - postgres-primary
        - -U
        - postgres
        - -d
        - mydb
        - -f
        - /backup/dump.sql
        env:
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: backup
          mountPath: /backup

      volumes:
      - name: backup
        persistentVolumeClaim:
          claimName: backup-pvc

      restartPolicy: OnFailure  # Обязательно: Never или OnFailure
```

#### Параллельный Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-processing
spec:
  completions: 10     # Сколько раз выполнить успешно
  parallelism: 3      # Сколько Pods одновременно
  backoffLimit: 5     # Максимум повторов при неудаче
  activeDeadlineSeconds: 600  # Таймаут для всего Job

  template:
    spec:
      containers:
      - name: worker
        image: worker:1.0
        command: ["./process-item"]
      restartPolicy: Never
```

**Режимы выполнения:**

| completions | parallelism | Поведение |
|-------------|-------------|-----------|
| 1 | 1 | Один Pod, одно выполнение |
| N | 1 | Последовательно N раз |
| N | M | До M Pods параллельно, всего N успешных |
| не указано | M | Work queue — работа до исчерпания очереди |

#### Job с Indexed Completion Mode

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: indexed-job
spec:
  completions: 5
  parallelism: 3
  completionMode: Indexed  # Каждый Pod получает уникальный индекс

  template:
    spec:
      containers:
      - name: worker
        image: worker:1.0
        command: ["./process"]
        env:
        - name: JOB_COMPLETION_INDEX  # Автоматически 0, 1, 2, 3, 4
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
      restartPolicy: Never
```

#### TTL для автоудаления

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: auto-cleanup-job
spec:
  ttlSecondsAfterFinished: 3600  # Удалить через час после завершения
  template:
    spec:
      containers:
      - name: task
        image: busybox
        command: ["echo", "Hello World"]
      restartPolicy: Never
```

### CronJob

**CronJob** — периодическое выполнение Job по расписанию (cron-формат).

#### Пример CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"  # Каждый день в 2:00
  timeZone: "Europe/Moscow"  # Kubernetes 1.27+

  concurrencyPolicy: Forbid  # Allow | Forbid | Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  startingDeadlineSeconds: 600  # Время на запуск после пропуска
  suspend: false  # Приостановка расписания

  jobTemplate:
    spec:
      backoffLimit: 3
      activeDeadlineSeconds: 3600

      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:1.0
            command: ["./backup.sh"]
            env:
            - name: BACKUP_TARGET
              value: "s3://my-bucket/backups"
          restartPolicy: OnFailure
```

#### Cron-расписание

```
┌───────────── минута (0 - 59)
│ ┌───────────── час (0 - 23)
│ │ ┌───────────── день месяца (1 - 31)
│ │ │ ┌───────────── месяц (1 - 12)
│ │ │ │ ┌───────────── день недели (0 - 6, 0 = воскресенье)
│ │ │ │ │
* * * * *
```

**Примеры расписаний:**

| Расписание | Описание |
|------------|----------|
| `*/15 * * * *` | Каждые 15 минут |
| `0 * * * *` | Каждый час |
| `0 0 * * *` | Каждый день в полночь |
| `0 0 * * 0` | Каждое воскресенье в полночь |
| `0 0 1 * *` | Первый день каждого месяца |
| `0 9-17 * * 1-5` | Каждый час с 9 до 17 в будни |

#### Политики конкурентности

- **Allow**: разрешить параллельные Job (по умолчанию)
- **Forbid**: пропустить, если предыдущий ещё выполняется
- **Replace**: остановить предыдущий и запустить новый

```yaml
spec:
  concurrencyPolicy: Forbid
```

#### Пример: очистка старых данных

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-old-data
spec:
  schedule: "0 3 * * *"  # 3:00 каждый день
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 3

  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: cleanup-sa
          containers:
          - name: cleanup
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              kubectl delete pods --field-selector=status.phase=Succeeded -n default
              kubectl delete pods --field-selector=status.phase=Failed -n default
          restartPolicy: OnFailure
```

### Команды kubectl для Jobs и CronJobs

```bash
# Jobs
kubectl apply -f job.yaml
kubectl get jobs
kubectl describe job backup-database
kubectl logs job/backup-database

# Просмотр Pods от Job
kubectl get pods --selector=job-name=backup-database

# Удаление Job (и его Pods)
kubectl delete job backup-database

# CronJobs
kubectl apply -f cronjob.yaml
kubectl get cronjobs
kubectl get cj  # Сокращённо
kubectl describe cj daily-backup

# Ручной запуск CronJob
kubectl create job --from=cronjob/daily-backup manual-backup

# Приостановка/возобновление
kubectl patch cronjob daily-backup -p '{"spec":{"suspend":true}}'
kubectl patch cronjob daily-backup -p '{"spec":{"suspend":false}}'

# Удаление CronJob (и всех созданных Jobs)
kubectl delete cronjob daily-backup
```

---

## Best Practices

### Общие рекомендации

1. **Всегда указывайте resource requests и limits**
   ```yaml
   resources:
     requests:
       memory: "128Mi"
       cpu: "100m"
     limits:
       memory: "256Mi"
       cpu: "200m"
   ```

2. **Используйте liveness и readiness probes**
   - Readiness: определяет, когда Pod готов принимать трафик
   - Liveness: определяет, нужно ли перезапустить контейнер

3. **Задавайте labels для всех ресурсов**
   ```yaml
   labels:
     app: myapp
     version: v1.0
     environment: production
     team: backend
   ```

4. **Используйте конкретные версии образов**
   ```yaml
   image: nginx:1.25.3  # Хорошо
   image: nginx:latest  # Плохо — непредсказуемо
   image: nginx         # Плохо — эквивалентно latest
   ```

5. **Настройте PodDisruptionBudget для критичных приложений**
   ```yaml
   apiVersion: policy/v1
   kind: PodDisruptionBudget
   metadata:
     name: app-pdb
   spec:
     minAvailable: 2  # или maxUnavailable: 1
     selector:
       matchLabels:
         app: myapp
   ```

### По типам ресурсов

#### Pods
- Не создавайте "голые" Pods — используйте контроллеры
- Используйте init-контейнеры для подготовительных задач
- Настройте terminationGracePeriodSeconds под ваше приложение

#### Deployments
- Используйте RollingUpdate с правильными maxSurge и maxUnavailable
- Храните достаточную историю для rollback (revisionHistoryLimit)
- Используйте pod anti-affinity для распределения по узлам

#### StatefulSets
- Обязательно создавайте Headless Service
- Используйте volumeClaimTemplates для персистентного хранилища
- Помните о порядке удаления — начинайте с конца

#### DaemonSets
- Добавляйте tolerations для control-plane узлов при необходимости
- Используйте resource limits для защиты узлов
- Настройте hostNetwork только если действительно нужно

#### Jobs/CronJobs
- Устанавливайте backoffLimit и activeDeadlineSeconds
- Используйте ttlSecondsAfterFinished для автоочистки
- Для CronJobs выбирайте правильную concurrencyPolicy

---

## Сравнительная таблица

| Ресурс | Назначение | Идентификаторы | Хранилище | Масштабирование |
|--------|------------|----------------|-----------|-----------------|
| Pod | Единица запуска | Случайные | Эфемерное | Нет |
| ReplicaSet | Поддержание реплик | Случайные | Эфемерное | Ручное |
| Deployment | Stateless apps | Случайные | Эфемерное | Авто/ручное |
| StatefulSet | Stateful apps | Порядковые | Персистентное | Ручное |
| DaemonSet | По одному на узел | На каждом узле | Host или PVC | По узлам |
| Job | Одноразовая задача | Случайные | Эфемерное | completions |
| CronJob | Периодическая задача | Случайные | Эфемерное | По расписанию |
