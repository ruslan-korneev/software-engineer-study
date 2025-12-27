# Планирование (Scheduling) в Kubernetes

## Введение в планирование Kubernetes

**Планирование (Scheduling)** — это процесс назначения подов на узлы кластера. За этот процесс отвечает компонент **kube-scheduler**, который является частью control plane Kubernetes.

### Как работает планировщик

Когда создаётся новый под без указания конкретного узла, планировщик:

1. **Фильтрация (Filtering)** — отсеивает узлы, которые не подходят для пода
2. **Оценка (Scoring)** — ранжирует оставшиеся узлы по приоритету
3. **Привязка (Binding)** — назначает под на узел с наивысшим score

```
Pod создан → Фильтрация узлов → Оценка узлов → Привязка к узлу
                    ↓                  ↓
            Удаление неподходящих   Ранжирование
                  узлов            по критериям
```

### Критерии фильтрации

Планировщик учитывает множество факторов:

- **Ресурсы** — достаточно ли CPU и памяти на узле
- **Taints и Tolerations** — разрешён ли под на узле
- **Node Selectors** — соответствует ли узел меткам
- **Affinity/Anti-Affinity** — правила размещения
- **Порты** — не заняты ли требуемые порты
- **Volumes** — доступны ли необходимые тома

### Просмотр событий планирования

```bash
# События планирования для конкретного пода
kubectl describe pod <pod-name>

# Все события в namespace
kubectl get events --sort-by='.lastTimestamp'

# События планировщика
kubectl get events --field-selector reason=Scheduled
```

---

## Node Selectors

**Node Selector** — самый простой способ ограничить размещение пода определёнными узлами. Используются метки (labels) узлов.

### Добавление меток к узлам

```bash
# Добавить метку к узлу
kubectl label nodes node-1 disktype=ssd

# Просмотреть метки узла
kubectl get nodes --show-labels

# Удалить метку
kubectl label nodes node-1 disktype-
```

### Использование nodeSelector в поде

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-ssd
spec:
  nodeSelector:
    disktype: ssd
  containers:
    - name: nginx
      image: nginx:1.24
      resources:
        requests:
          memory: "64Mi"
          cpu: "100m"
```

### Node Selector в Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      nodeSelector:
        environment: production
        tier: frontend
      containers:
        - name: web
          image: nginx:1.24
```

### Встроенные метки узлов

Kubernetes автоматически добавляет метки к узлам:

```yaml
# Примеры встроенных меток
kubernetes.io/hostname: node-1
kubernetes.io/os: linux
kubernetes.io/arch: amd64
topology.kubernetes.io/zone: us-east-1a
topology.kubernetes.io/region: us-east-1
node.kubernetes.io/instance-type: m5.large
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: linux-pod
spec:
  nodeSelector:
    kubernetes.io/os: linux
    kubernetes.io/arch: amd64
  containers:
    - name: app
      image: myapp:1.0
```

### Ограничения Node Selector

- Поддерживает только **точное совпадение** (equality-based)
- Нельзя использовать **OR** логику
- Нельзя использовать **NOT** (исключение)
- Нет **мягких** (preferred) правил

Для более гибкого управления используйте **Node Affinity**.

---

## Node Affinity и Node Anti-Affinity

**Node Affinity** — расширенная версия nodeSelector с поддержкой:
- Операторов сравнения (In, NotIn, Exists, DoesNotExist, Gt, Lt)
- Мягких (preferred) и жёстких (required) правил
- Более выразительного синтаксиса

### Типы Node Affinity

| Тип | Описание |
|-----|----------|
| `requiredDuringSchedulingIgnoredDuringExecution` | **Жёсткое правило** — под не будет запланирован, если правило не выполняется |
| `preferredDuringSchedulingIgnoredDuringExecution` | **Мягкое правило** — планировщик попытается выполнить, но не гарантирует |

> **IgnoredDuringExecution** означает, что если метки узла изменятся после размещения пода, под останется на узле.

### Жёсткое правило (Required)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
                  - nvme
  containers:
    - name: nginx
      image: nginx:1.24
```

### Мягкое правило (Preferred)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-preferred
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
        - weight: 20
          preference:
            matchExpressions:
              - key: zone
                operator: In
                values:
                  - us-east-1a
  containers:
    - name: nginx
      image: nginx:1.24
```

### Операторы сравнения

| Оператор | Описание | Пример |
|----------|----------|--------|
| `In` | Значение в списке | `disktype In [ssd, nvme]` |
| `NotIn` | Значение НЕ в списке | `env NotIn [dev, test]` |
| `Exists` | Метка существует | `gpu Exists` |
| `DoesNotExist` | Метка не существует | `temporary DoesNotExist` |
| `Gt` | Больше (для числовых значений) | `cores Gt 4` |
| `Lt` | Меньше (для числовых значений) | `memory Lt 32` |

### Комбинирование правил

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
spec:
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      affinity:
        nodeAffinity:
          # Обязательно: Linux с SSD
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: disktype
                    operator: In
                    values:
                      - ssd
          # Желательно: production окружение
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: environment
                    operator: In
                    values:
                      - production
      containers:
        - name: postgres
          image: postgres:15
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
```

### Node Anti-Affinity

Для исключения узлов используйте оператор `NotIn` или `DoesNotExist`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: avoid-gpu-nodes
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              # Избегать узлов с GPU
              - key: gpu
                operator: DoesNotExist
              # Избегать dev окружения
              - key: environment
                operator: NotIn
                values:
                  - development
                  - staging
  containers:
    - name: app
      image: myapp:1.0
```

---

## Pod Affinity и Pod Anti-Affinity

**Pod Affinity** позволяет размещать поды относительно других подов, а не узлов. Это полезно для:

- **Co-location** — размещение связанных сервисов рядом
- **Распределение** — размещение реплик на разных узлах/зонах
- **Изоляция** — избегание соседства с определёнными подами

### Топологические ключи

Pod Affinity работает в контексте **топологии** — логического разделения кластера:

```yaml
# Примеры топологических ключей
kubernetes.io/hostname      # Уровень узла
topology.kubernetes.io/zone # Уровень зоны доступности
topology.kubernetes.io/region # Уровень региона
```

### Pod Affinity — размещение рядом

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-frontend
  template:
    metadata:
      labels:
        app: web-frontend
    spec:
      affinity:
        podAffinity:
          # Размещать рядом с cache подами
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - redis-cache
              topologyKey: kubernetes.io/hostname
      containers:
        - name: frontend
          image: frontend:1.0
```

### Pod Anti-Affinity — распределение

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cluster
spec:
  replicas: 6
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      affinity:
        podAntiAffinity:
          # Не размещать две реплики на одном узле
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: redis
              topologyKey: kubernetes.io/hostname
      containers:
        - name: redis
          image: redis:7
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
```

### Мягкое распределение по зонам

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 6
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAntiAffinity:
          # Предпочтительно размещать в разных зонах
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: web
                topologyKey: topology.kubernetes.io/zone
          # Обязательно размещать на разных узлах
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: web
              topologyKey: kubernetes.io/hostname
      containers:
        - name: web
          image: nginx:1.24
```

### Комплексный пример: Web + Cache

```yaml
# Redis Cache
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  replicas: 3
  selector:
    matchLabels:
      app: redis-cache
  template:
    metadata:
      labels:
        app: redis-cache
        tier: cache
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: redis-cache
              topologyKey: kubernetes.io/hostname
      containers:
        - name: redis
          image: redis:7
---
# Web Application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 6
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
        tier: frontend
    spec:
      affinity:
        # Размещать рядом с redis-cache
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: redis-cache
                topologyKey: kubernetes.io/hostname
        # Распределять web поды по узлам
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 50
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: web-app
                topologyKey: kubernetes.io/hostname
      containers:
        - name: web
          image: webapp:1.0
```

### Namespace Selector

Можно искать поды в других namespace:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: database
          topologyKey: kubernetes.io/hostname
          namespaceSelector:
            matchLabels:
              environment: production
          namespaces:
            - database-prod
  containers:
    - name: app
      image: myapp:1.0
```

---

## Taints и Tolerations

**Taints** (пятна/отметки) — механизм, позволяющий узлам "отталкивать" определённые поды. **Tolerations** (допуски) — разрешение поду игнорировать taint и быть размещённым на узле.

### Концепция

```
Taint на узле:  "gpu=true:NoSchedule"
                        ↓
Pod без toleration → Отклонён
Pod с toleration  → Разрешён
```

### Эффекты Taint

| Эффект | Описание |
|--------|----------|
| `NoSchedule` | Новые поды без toleration не будут размещены |
| `PreferNoSchedule` | Планировщик попытается избежать узла, но не гарантирует |
| `NoExecute` | Существующие поды без toleration будут удалены |

### Управление Taints

```bash
# Добавить taint к узлу
kubectl taint nodes node-1 gpu=true:NoSchedule

# Просмотреть taints узла
kubectl describe node node-1 | grep Taints

# Удалить taint (добавить минус в конце)
kubectl taint nodes node-1 gpu=true:NoSchedule-

# Добавить taint без значения
kubectl taint nodes node-1 dedicated:NoSchedule
```

### Tolerations в Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  tolerations:
    - key: "gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  containers:
    - name: cuda-app
      image: nvidia/cuda:11.0-base
```

### Операторы Toleration

```yaml
# Equal — точное совпадение key и value
tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "nvidia"
    effect: "NoSchedule"

# Exists — достаточно наличия key (value игнорируется)
tolerations:
  - key: "gpu"
    operator: "Exists"
    effect: "NoSchedule"

# Toleration для всех taints с определённым key
tolerations:
  - key: "gpu"
    operator: "Exists"

# Toleration для ВСЕХ taints (не рекомендуется)
tolerations:
  - operator: "Exists"
```

### Toleration с tolerationSeconds

Для эффекта `NoExecute` можно указать время ожидания перед удалением:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tolerant-pod
spec:
  tolerations:
    - key: "node.kubernetes.io/not-ready"
      operator: "Exists"
      effect: "NoExecute"
      tolerationSeconds: 300  # 5 минут
    - key: "node.kubernetes.io/unreachable"
      operator: "Exists"
      effect: "NoExecute"
      tolerationSeconds: 300
  containers:
    - name: app
      image: myapp:1.0
```

### Практические сценарии

#### Выделенные узлы для определённых workloads

```bash
# Пометить узлы как выделенные для GPU
kubectl taint nodes gpu-node-1 gpu-node-2 dedicated=gpu:NoSchedule
kubectl label nodes gpu-node-1 gpu-node-2 hardware=gpu
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-training
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ml-training
  template:
    metadata:
      labels:
        app: ml-training
    spec:
      nodeSelector:
        hardware: gpu
      tolerations:
        - key: "dedicated"
          operator: "Equal"
          value: "gpu"
          effect: "NoSchedule"
      containers:
        - name: training
          image: ml-training:1.0
          resources:
            limits:
              nvidia.com/gpu: 1
```

#### Узлы для master компонентов

```yaml
# Control plane узлы обычно имеют taint:
# node-role.kubernetes.io/control-plane:NoSchedule

apiVersion: v1
kind: Pod
metadata:
  name: admin-tool
spec:
  tolerations:
    - key: "node-role.kubernetes.io/control-plane"
      operator: "Exists"
      effect: "NoSchedule"
  nodeSelector:
    node-role.kubernetes.io/control-plane: ""
  containers:
    - name: admin
      image: admin-tool:1.0
```

### Встроенные Taints

Kubernetes автоматически добавляет taints при проблемах с узлом:

| Taint | Причина |
|-------|---------|
| `node.kubernetes.io/not-ready` | Узел не готов |
| `node.kubernetes.io/unreachable` | Node controller не может связаться с узлом |
| `node.kubernetes.io/memory-pressure` | Нехватка памяти |
| `node.kubernetes.io/disk-pressure` | Нехватка диска |
| `node.kubernetes.io/pid-pressure` | Нехватка PID |
| `node.kubernetes.io/network-unavailable` | Сеть недоступна |
| `node.kubernetes.io/unschedulable` | Узел помечен как unschedulable |

---

## Priority и Preemption

**Priority** позволяет назначать приоритеты подам. При нехватке ресурсов поды с более высоким приоритетом могут вытеснять (preempt) поды с низким приоритетом.

### PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Высокий приоритет для критичных сервисов"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 1000
globalDefault: false
preemptionPolicy: Never  # Не вытеснять другие поды
description: "Низкий приоритет для batch jobs"
```

### Системные PriorityClass

```yaml
# Встроенные классы (не изменять!)
system-cluster-critical: 2000000000  # Критичные для кластера
system-node-critical: 2000001000     # Критичные для узла
```

### Использование приоритета

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: critical
  template:
    metadata:
      labels:
        app: critical
    spec:
      priorityClassName: high-priority
      containers:
        - name: app
          image: critical-app:1.0
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
---
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-processing
spec:
  template:
    spec:
      priorityClassName: low-priority
      restartPolicy: OnFailure
      containers:
        - name: processor
          image: batch-processor:1.0
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
```

### Preemption Policy

| Политика | Описание |
|----------|----------|
| `PreemptLowerPriority` | Может вытеснять поды с меньшим приоритетом (по умолчанию) |
| `Never` | Не вытесняет другие поды, только ожидает в очереди |

### Пример иерархии приоритетов

```yaml
# 1. Критические системные сервисы
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: platform-critical
value: 900000
description: "Платформенные сервисы (мониторинг, логирование)"
---
# 2. Production приложения
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production
value: 500000
globalDefault: true
description: "Production workloads"
---
# 3. Staging/Dev окружения
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: non-production
value: 100000
description: "Staging и development workloads"
---
# 4. Batch jobs и cron
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch
value: 10000
preemptionPolicy: Never
description: "Batch processing, может быть вытеснен"
```

---

## Resource Requests и Limits в контексте планирования

Планировщик использует **resource requests** для принятия решений о размещении подов.

### Как планировщик использует ресурсы

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
    - name: app
      image: myapp:1.0
      resources:
        requests:          # Используется планировщиком
          memory: "256Mi"
          cpu: "250m"
        limits:            # Ограничение runtime, не влияет на scheduling
          memory: "512Mi"
          cpu: "500m"
```

### Allocatable vs Capacity

```bash
# Просмотр ресурсов узла
kubectl describe node node-1

# Capacity — общие ресурсы узла
# Allocatable — доступно для подов (Capacity - системные резервы)
```

```
Capacity:
  cpu:                4
  memory:             8Gi
  pods:               110
Allocatable:
  cpu:                3800m
  memory:             7Gi
  pods:               110
```

### Quality of Service (QoS) классы

Kubernetes автоматически назначает QoS класс на основе requests/limits:

| QoS класс | Условие | Приоритет при OOM |
|-----------|---------|-------------------|
| `Guaranteed` | requests == limits для всех контейнеров | Самый высокий (последний для eviction) |
| `Burstable` | requests < limits или не у всех контейнеров | Средний |
| `BestEffort` | Нет requests и limits | Самый низкий (первый для eviction) |

```yaml
# Guaranteed QoS
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
    - name: app
      image: myapp:1.0
      resources:
        requests:
          memory: "256Mi"
          cpu: "250m"
        limits:
          memory: "256Mi"  # Равно requests
          cpu: "250m"      # Равно requests
---
# Burstable QoS
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
spec:
  containers:
    - name: app
      image: myapp:1.0
      resources:
        requests:
          memory: "256Mi"
          cpu: "250m"
        limits:
          memory: "512Mi"  # Больше requests
          cpu: "500m"      # Больше requests
---
# BestEffort QoS (не рекомендуется для production)
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
spec:
  containers:
    - name: app
      image: myapp:1.0
      # Нет resources
```

### LimitRange — ограничения по умолчанию

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:           # Limits по умолчанию
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:    # Requests по умолчанию
        cpu: "100m"
        memory: "128Mi"
      min:               # Минимальные значения
        cpu: "50m"
        memory: "64Mi"
      max:               # Максимальные значения
        cpu: "2"
        memory: "2Gi"
    - type: Pod
      max:
        cpu: "4"
        memory: "4Gi"
```

### ResourceQuota — квоты namespace

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: development
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "50"
    persistentvolumeclaims: "10"
```

---

## Custom Schedulers

Kubernetes позволяет использовать собственные планировщики вместо или вместе с kube-scheduler.

### Указание планировщика для пода

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-scheduled-pod
spec:
  schedulerName: my-custom-scheduler
  containers:
    - name: app
      image: myapp:1.0
```

### Deployment пользовательского планировщика

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-custom-scheduler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      component: my-custom-scheduler
  template:
    metadata:
      labels:
        component: my-custom-scheduler
    spec:
      serviceAccountName: my-scheduler-sa
      containers:
        - name: scheduler
          image: my-scheduler:1.0
          command:
            - /scheduler
            - --config=/etc/scheduler/config.yaml
          volumeMounts:
            - name: config
              mountPath: /etc/scheduler
      volumes:
        - name: config
          configMap:
            name: scheduler-config
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-scheduler-sa
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-scheduler-binding
subjects:
  - kind: ServiceAccount
    name: my-scheduler-sa
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: system:kube-scheduler
  apiGroup: rbac.authorization.k8s.io
```

### Использование нескольких планировщиков

```yaml
# Pod с кастомным планировщиком
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  schedulerName: gpu-scheduler
  containers:
    - name: ml-job
      image: ml-training:1.0
---
# Pod с дефолтным планировщиком
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  # schedulerName не указан — используется default-scheduler
  containers:
    - name: web
      image: nginx:1.24
```

---

## Scheduling Profiles

**Scheduling Profiles** позволяют конфигурировать поведение планировщика через профили.

### Конфигурация планировщика

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  # Профиль по умолчанию
  - schedulerName: default-scheduler
    plugins:
      score:
        enabled:
          - name: NodeResourcesBalancedAllocation
            weight: 1
          - name: ImageLocality
            weight: 1
        disabled:
          - name: NodeResourcesLeastAllocated
      filter:
        enabled:
          - name: NodePorts
          - name: NodeAffinity
          - name: TaintToleration
    pluginConfig:
      - name: NodeResourcesBalancedAllocation
        args:
          resources:
            - name: cpu
              weight: 1
            - name: memory
              weight: 1

  # Профиль для batch workloads
  - schedulerName: batch-scheduler
    plugins:
      score:
        enabled:
          - name: NodeResourcesLeastAllocated
            weight: 1
```

### Фазы планирования и плагины

| Фаза | Описание | Примеры плагинов |
|------|----------|------------------|
| `PreFilter` | Подготовка перед фильтрацией | InterPodAffinity |
| `Filter` | Отсеивание неподходящих узлов | NodePorts, TaintToleration, NodeAffinity |
| `PostFilter` | Действия после фильтрации | DefaultPreemption |
| `PreScore` | Подготовка перед оценкой | InterPodAffinity |
| `Score` | Ранжирование узлов | NodeResourcesBalancedAllocation, ImageLocality |
| `Reserve` | Резервирование ресурсов | VolumeBinding |
| `Permit` | Разрешение/отклонение | — |
| `PreBind` | Подготовка перед привязкой | VolumeBinding |
| `Bind` | Привязка пода к узлу | DefaultBinder |
| `PostBind` | Действия после привязки | — |

### Пример профиля с весами

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
    plugins:
      score:
        enabled:
          - name: NodeResourcesBalancedAllocation
            weight: 2
          - name: InterPodAffinity
            weight: 2
          - name: NodeAffinity
            weight: 1
          - name: TaintToleration
            weight: 1
```

---

## Полные примеры манифестов

### Пример 1: High Availability Web Application

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: web-production
value: 500000
globalDefault: false
description: "Production web applications"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-web-app
  namespace: production
spec:
  replicas: 6
  selector:
    matchLabels:
      app: ha-web
  template:
    metadata:
      labels:
        app: ha-web
        tier: frontend
    spec:
      priorityClassName: web-production
      affinity:
        # Узлы с SSD, не development
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: disktype
                    operator: In
                    values:
                      - ssd
                  - key: environment
                    operator: NotIn
                    values:
                      - development
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: zone
                    operator: In
                    values:
                      - us-east-1a
                      - us-east-1b
        # Распределить по зонам
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: ha-web
              topologyKey: kubernetes.io/hostname
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: ha-web
                topologyKey: topology.kubernetes.io/zone
      tolerations:
        - key: "node.kubernetes.io/not-ready"
          operator: "Exists"
          effect: "NoExecute"
          tolerationSeconds: 60
      containers:
        - name: web
          image: webapp:1.0
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          ports:
            - containerPort: 8080
```

### Пример 2: Database с выделенными узлами

```bash
# Подготовка узлов
kubectl taint nodes db-node-1 db-node-2 dedicated=database:NoSchedule
kubectl label nodes db-node-1 db-node-2 workload=database disktype=ssd
```

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: database-critical
value: 800000
description: "Critical database workloads"
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-cluster
  namespace: database
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
      priorityClassName: database-critical
      nodeSelector:
        workload: database
        disktype: ssd
      tolerations:
        - key: "dedicated"
          operator: "Equal"
          value: "database"
          effect: "NoSchedule"
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: postgres
              topologyKey: kubernetes.io/hostname
      containers:
        - name: postgres
          image: postgres:15
          resources:
            requests:
              memory: "2Gi"
              cpu: "1"
            limits:
              memory: "4Gi"
              cpu: "2"
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 100Gi
```

### Пример 3: ML Training Jobs

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: ml-training
value: 300000
preemptionPolicy: PreemptLowerPriority
description: "ML training jobs"
---
apiVersion: batch/v1
kind: Job
metadata:
  name: model-training
  namespace: ml
spec:
  parallelism: 4
  completions: 4
  template:
    spec:
      priorityClassName: ml-training
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: nvidia.com/gpu.present
                    operator: Exists
                  - key: gpu-type
                    operator: In
                    values:
                      - a100
                      - v100
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    job-name: model-training
                topologyKey: kubernetes.io/hostname
      tolerations:
        - key: "nvidia.com/gpu"
          operator: "Exists"
          effect: "NoSchedule"
        - key: "dedicated"
          operator: "Equal"
          value: "gpu"
          effect: "NoSchedule"
      restartPolicy: OnFailure
      containers:
        - name: trainer
          image: ml-training:1.0
          resources:
            requests:
              memory: "16Gi"
              cpu: "4"
              nvidia.com/gpu: 1
            limits:
              memory: "32Gi"
              cpu: "8"
              nvidia.com/gpu: 1
```

### Пример 4: Микросервисная архитектура

```yaml
# API Gateway — на edge узлах
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  replicas: 4
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
        tier: edge
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node-type
                    operator: In
                    values:
                      - edge
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: api-gateway
              topologyKey: kubernetes.io/hostname
      containers:
        - name: gateway
          image: api-gateway:1.0
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
---
# Backend Service — рядом с cache
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-service
spec:
  replicas: 6
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        tier: backend
    spec:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: redis
                topologyKey: kubernetes.io/hostname
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 50
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: backend
                topologyKey: kubernetes.io/hostname
      containers:
        - name: backend
          image: backend:1.0
          resources:
            requests:
              memory: "256Mi"
              cpu: "200m"
---
# Redis Cache
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
        tier: cache
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: redis
              topologyKey: kubernetes.io/hostname
      containers:
        - name: redis
          image: redis:7
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
```

---

## Troubleshooting планирования

### Частые проблемы

```bash
# Под в статусе Pending
kubectl describe pod <pod-name>
# Смотрим Events секцию

# Типичные причины:
# - Insufficient cpu/memory
# - No nodes match nodeSelector
# - Taint not tolerated
# - Node affinity not matched
```

### Полезные команды

```bash
# Ресурсы узлов
kubectl top nodes

# Allocatable ресурсы
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
CPU:.status.allocatable.cpu,\
MEMORY:.status.allocatable.memory

# Поды на узле
kubectl get pods --all-namespaces --field-selector spec.nodeName=node-1

# Taints узла
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
TAINTS:.spec.taints

# События планировщика
kubectl get events --field-selector reason=FailedScheduling
```

### Дебаг планировщика

```bash
# Логи планировщика
kubectl logs -n kube-system -l component=kube-scheduler

# Verbose логирование
kubectl logs -n kube-system kube-scheduler-master --v=4
```

---

## Best Practices

1. **Всегда указывайте resource requests** — без них планировщик не может принимать оптимальные решения

2. **Используйте Pod Anti-Affinity для HA** — распределяйте реплики по узлам и зонам

3. **Применяйте Taints для изоляции** — выделяйте узлы для специфических workloads

4. **Настройте PriorityClass** — определите иерархию важности приложений

5. **Предпочитайте мягкие правила** — используйте `preferred` когда строгое размещение не критично

6. **Мониторьте pending поды** — это индикатор проблем с планированием

7. **Тестируйте scheduling правила** — проверяйте поведение в staging окружении

8. **Документируйте стратегию** — объясняйте причины placement ограничений

---

## Ссылки

- [Kubernetes Scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
- [Scheduler Configuration](https://kubernetes.io/docs/reference/scheduling/config/)
