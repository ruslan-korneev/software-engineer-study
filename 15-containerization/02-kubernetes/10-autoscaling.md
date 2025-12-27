# Autoscaling в Kubernetes

Автомасштабирование — это ключевая возможность Kubernetes, позволяющая автоматически адаптировать ресурсы приложения под текущую нагрузку. Kubernetes предоставляет несколько механизмов автомасштабирования на разных уровнях: подов, ресурсов контейнеров и узлов кластера.

## Виды автомасштабирования

| Тип | Что масштабирует | Основа для решений |
|-----|------------------|-------------------|
| HPA (Horizontal Pod Autoscaler) | Количество реплик подов | Метрики (CPU, memory, custom) |
| VPA (Vertical Pod Autoscaler) | Ресурсы контейнеров (requests/limits) | Исторические данные потребления |
| Cluster Autoscaler | Количество нод в кластере | Pending поды, недоиспользование нод |
| KEDA | Количество подов | События и внешние метрики |

---

## Horizontal Pod Autoscaler (HPA)

### Принцип работы

HPA автоматически изменяет количество реплик в Deployment, ReplicaSet или StatefulSet на основе наблюдаемых метрик.

**Алгоритм расчёта:**

```
desiredReplicas = ceil[currentReplicas * (currentMetricValue / desiredMetricValue)]
```

HPA работает в цикле:
1. Каждые 15 секунд (по умолчанию) собирает метрики через Metrics API
2. Сравнивает текущее значение с целевым
3. Вычисляет необходимое количество реплик
4. Применяет изменения (с учётом стабилизации)

### Архитектура

```
┌─────────────────────────────────────────────────────────────┐
│                    HPA Controller                            │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ Metrics API │────│  Algorithm  │────│  Scale API  │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
└─────────────────────────────────────────────────────────────┘
         │                                        │
         ▼                                        ▼
┌─────────────────┐                    ┌─────────────────────┐
│ Metrics Server  │                    │ Deployment/RS/SS    │
│ (or Prometheus) │                    │ spec.replicas       │
└─────────────────┘                    └─────────────────────┘
```

### Типы метрик HPA

#### 1. Resource Metrics (CPU, Memory)

Встроенные метрики, получаемые от Metrics Server:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  # CPU — поддерживать среднюю загрузку на уровне 70%
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # Memory — среднее потребление не более 500Mi
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 500Mi
```

#### 2. Pods Metrics

Метрики, специфичные для подов (например, requests per second):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 1000  # 1000 RPS на под
```

#### 3. Object Metrics

Метрики от конкретного Kubernetes объекта:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: queue-worker
  minReplicas: 1
  maxReplicas: 100
  metrics:
  - type: Object
    object:
      metric:
        name: queue_messages_ready
      describedObject:
        apiVersion: v1
        kind: Service
        name: rabbitmq
      target:
        type: Value
        value: 100  # Scale up если в очереди > 100 сообщений
```

#### 4. External Metrics

Метрики из внешних систем мониторинга:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: external-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-processor
  minReplicas: 2
  maxReplicas: 30
  metrics:
  - type: External
    external:
      metric:
        name: pubsub.googleapis.com|subscription|num_undelivered_messages
        selector:
          matchLabels:
            resource.labels.subscription_id: payments-subscription
      target:
        type: AverageValue
        averageValue: 10
```

### Комбинирование метрик

HPA может использовать несколько метрик одновременно. Итоговое количество реплик — максимум из всех расчётов:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: multi-metric-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-service
  minReplicas: 3
  maxReplicas: 100
  metrics:
  # Метрика 1: CPU
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  # Метрика 2: Memory
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  # Метрика 3: Кастомная — RPS
  - type: Pods
    pods:
      metric:
        name: requests_per_second
      target:
        type: AverageValue
        averageValue: 500
```

---

## Stabilization Window и Behavior

### Проблема осцилляций

Без стабилизации HPA может постоянно масштабировать вверх-вниз при колебаниях нагрузки (flapping). Начиная с Kubernetes 1.18, появилась возможность тонкой настройки behavior.

### Конфигурация Behavior

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: stable-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: production-app
  minReplicas: 5
  maxReplicas: 100
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    # Политика scale up
    scaleUp:
      stabilizationWindowSeconds: 0  # Масштабируем сразу при нагрузке
      policies:
      - type: Percent
        value: 100          # Можем удвоить количество подов
        periodSeconds: 15
      - type: Pods
        value: 4            # Или добавить 4 пода
        periodSeconds: 15
      selectPolicy: Max     # Выбрать максимум из политик
    # Политика scale down
    scaleDown:
      stabilizationWindowSeconds: 300  # 5 минут ожидания перед scale down
      policies:
      - type: Percent
        value: 10           # Уменьшать на 10% за период
        periodSeconds: 60
      - type: Pods
        value: 2            # Или удалять по 2 пода
        periodSeconds: 60
      selectPolicy: Min     # Выбрать минимум (консервативно)
```

### Стратегии selectPolicy

| Значение | Описание |
|----------|----------|
| `Max` | Выбирает политику, дающую максимальное изменение |
| `Min` | Выбирает политику, дающую минимальное изменение |
| `Disabled` | Отключает масштабирование в этом направлении |

### Пример: Отключение Scale Down

Полезно при критичных сервисах, где уменьшение реплик нежелательно:

```yaml
behavior:
  scaleDown:
    selectPolicy: Disabled
```

---

## Vertical Pod Autoscaler (VPA)

### Когда использовать VPA

VPA автоматически подбирает оптимальные `requests` и `limits` для контейнеров:

**Используйте VPA когда:**
- Сложно предсказать потребление ресурсов
- Приложение имеет переменную нагрузку во времени
- Хотите оптимизировать утилизацию кластера
- Приложение не может горизонтально масштабироваться

**Не используйте VPA когда:**
- Приложение уже использует HPA по CPU/Memory (конфликт!)
- Требуется мгновенная реакция на нагрузку
- Недопустим перезапуск подов

### Установка VPA

VPA не входит в стандартную поставку Kubernetes:

```bash
# Клонирование репозитория
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler

# Установка
./hack/vpa-up.sh
```

### Компоненты VPA

```
┌─────────────────────────────────────────────────────────┐
│                       VPA System                         │
│  ┌────────────┐  ┌────────────┐  ┌────────────────┐    │
│  │ Recommender│  │  Updater   │  │ Admission      │    │
│  │            │  │            │  │ Controller     │    │
│  └────────────┘  └────────────┘  └────────────────┘    │
│       │               │                  │              │
│       ▼               ▼                  ▼              │
│  Анализирует      Выселяет поды      Модифицирует      │
│  метрики и        для обновления     requests при       │
│  генерирует                          создании пода      │
│  рекомендации                                           │
└─────────────────────────────────────────────────────────┘
```

### Режимы работы VPA

| Режим | Описание |
|-------|----------|
| `Off` | Только рекомендации, без изменений |
| `Initial` | Применяет рекомендации только при создании подов |
| `Recreate` | Пересоздаёт поды для применения рекомендаций |
| `Auto` | Выбирает лучший метод (сейчас = Recreate) |

### Базовый пример VPA

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: backend-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-service
  updatePolicy:
    updateMode: Auto  # Автоматически применять рекомендации
  resourcePolicy:
    containerPolicies:
    - containerName: '*'  # Для всех контейнеров
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 4
        memory: 8Gi
      controlledResources:
      - cpu
      - memory
      controlledValues: RequestsAndLimits  # Управлять и requests, и limits
```

### VPA только для рекомендаций

Для анализа без автоматических изменений:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: analysis-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Off"  # Только рекомендации
```

Просмотр рекомендаций:

```bash
kubectl describe vpa analysis-vpa

# Вывод включает:
# Recommendation:
#   Container Recommendations:
#     Container Name:  my-container
#     Lower Bound:
#       Cpu:     25m
#       Memory:  262144k
#     Target:
#       Cpu:     100m
#       Memory:  524288k
#     Upper Bound:
#       Cpu:     200m
#       Memory:  1Gi
```

### Исключение контейнеров

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: selective-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: multi-container-app
  resourcePolicy:
    containerPolicies:
    - containerName: main-app
      minAllowed:
        cpu: 200m
        memory: 256Mi
      maxAllowed:
        cpu: 2
        memory: 4Gi
    - containerName: sidecar-proxy
      mode: "Off"  # Не трогать этот контейнер
```

### Совместное использование VPA и HPA

VPA и HPA конфликтуют при использовании одних и тех же метрик. Безопасная комбинация:

```yaml
# VPA управляет только memory
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: memory-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hybrid-app
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      controlledResources:
      - memory  # Только память!
---
# HPA масштабирует по CPU и кастомным метрикам
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hybrid-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: requests_per_second
      target:
        type: AverageValue
        averageValue: 500
```

---

## Cluster Autoscaler

### Принцип работы

Cluster Autoscaler управляет количеством нод в кластере:

**Scale Up** происходит когда:
- Есть поды в состоянии `Pending` из-за нехватки ресурсов
- Scheduler не может разместить поды на существующих нодах

**Scale Down** происходит когда:
- Нода недоиспользуется (< 50% ресурсов по умолчанию)
- Все поды на ноде могут быть перемещены на другие ноды
- Нода не имеет "блокирующих" подов

### Архитектура

```
┌──────────────────────────────────────────────────────────────┐
│                    Cluster Autoscaler                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐  │
│  │ Scale Up    │    │ Scale Down  │    │ Cloud Provider  │  │
│  │ Logic       │    │ Logic       │    │ Interface       │  │
│  └─────────────┘    └─────────────┘    └─────────────────┘  │
│        │                  │                    │             │
│        ▼                  ▼                    ▼             │
│  Pending Pods?      Underutilized      AWS/GCP/Azure        │
│  Unschedulable?     Nodes?             API calls            │
└──────────────────────────────────────────────────────────────┘
```

### Конфигурация для AWS EKS

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - name: cluster-autoscaler
        image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.29.0
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --skip-nodes-with-system-pods=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
        - --balance-similar-node-groups
        - --scale-down-enabled=true
        - --scale-down-delay-after-add=10m
        - --scale-down-unneeded-time=10m
        - --scale-down-utilization-threshold=0.5
        env:
        - name: AWS_REGION
          value: us-west-2
```

### Expander стратегии

| Стратегия | Описание |
|-----------|----------|
| `random` | Случайный выбор node group |
| `least-waste` | Минимизация неиспользуемых ресурсов |
| `most-pods` | Максимизация размещаемых подов |
| `priority` | По приоритетам из ConfigMap |
| `price` | По цене нод (cloud-specific) |

### Priority Expander

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-priority-expander
  namespace: kube-system
data:
  priorities: |-
    10:
      - spot-node-group.*
    50:
      - on-demand-node-group.*
```

### Аннотации для защиты нод

```yaml
# Запретить scale down для конкретной ноды
kubectl annotate node node-1 cluster-autoscaler.kubernetes.io/scale-down-disabled=true

# Запретить scale down если есть критичные поды
apiVersion: v1
kind: Pod
metadata:
  name: critical-pod
  annotations:
    cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
```

### Условия блокировки Scale Down

Нода НЕ будет удалена если:
- Есть поды с `annotation: cluster-autoscaler.kubernetes.io/safe-to-evict: "false"`
- Есть поды с local storage (без override)
- Есть поды без controller (orphan pods)
- Есть поды с restrictive PDB
- Есть системные поды из `kube-system` (без override)

---

## KEDA (Kubernetes Event-driven Autoscaling)

### Что такое KEDA

KEDA — это легковесный компонент, расширяющий HPA для event-driven масштабирования. Поддерживает 50+ scalers:
- Очереди сообщений (Kafka, RabbitMQ, SQS, Azure Queue)
- Базы данных (PostgreSQL, MySQL, MongoDB)
- Prometheus, Datadog, New Relic
- Cron (расписание)
- HTTP запросы
- И многое другое

### Установка KEDA

```bash
# Через Helm
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace

# Или через kubectl
kubectl apply -f https://github.com/kedacore/keda/releases/download/v2.13.0/keda-2.13.0.yaml
```

### Архитектура KEDA

```
┌────────────────────────────────────────────────────────────┐
│                         KEDA                                │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    │
│  │ KEDA        │    │ Metrics     │    │ Admission   │    │
│  │ Operator    │    │ Adapter     │    │ Webhooks    │    │
│  └─────────────┘    └─────────────┘    └─────────────┘    │
│        │                  │                               │
│        ▼                  ▼                               │
│  ScaledObject      External Metrics API                   │
│  ScaledJob         (for HPA consumption)                  │
└────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────┐
│ External System │
│ (Kafka, Redis,  │
│  Prometheus...) │
└─────────────────┘
```

### ScaledObject для RabbitMQ

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rabbitmq-scaledobject
  namespace: production
spec:
  scaleTargetRef:
    name: queue-consumer
  pollingInterval: 15          # Как часто проверять метрики (секунды)
  cooldownPeriod: 300          # Ожидание перед scale down до 0
  minReplicaCount: 0           # KEDA может масштабировать до 0!
  maxReplicaCount: 50
  triggers:
  - type: rabbitmq
    metadata:
      protocol: amqp
      queueName: tasks
      mode: QueueLength
      value: "10"              # Scale когда > 10 сообщений на под
    authenticationRef:
      name: rabbitmq-auth
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: rabbitmq-auth
  namespace: production
spec:
  secretTargetRef:
  - parameter: host
    name: rabbitmq-secrets
    key: connection-string
```

### ScaledObject для Kafka

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-scaledobject
spec:
  scaleTargetRef:
    name: kafka-consumer
  pollingInterval: 10
  cooldownPeriod: 60
  minReplicaCount: 1
  maxReplicaCount: 100
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka.default.svc:9092
      consumerGroup: my-consumer-group
      topic: events
      lagThreshold: "100"      # Scale при lag > 100
      activationLagThreshold: "0"  # Activate при lag > 0
```

### ScaledObject для Prometheus

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: prometheus-scaledobject
spec:
  scaleTargetRef:
    name: api-server
  pollingInterval: 15
  cooldownPeriod: 30
  minReplicaCount: 2
  maxReplicaCount: 100
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 300
          policies:
          - type: Percent
            value: 10
            periodSeconds: 60
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring.svc:9090
      metricName: http_requests_total
      query: sum(rate(http_requests_total{deployment="api-server"}[2m]))
      threshold: "100"
```

### ScaledObject для Cron (расписание)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: cron-scaledobject
spec:
  scaleTargetRef:
    name: batch-processor
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
  # Рабочие часы: 10 реплик
  - type: cron
    metadata:
      timezone: Europe/Moscow
      start: 0 9 * * 1-5   # Пн-Пт, 9:00
      end: 0 18 * * 1-5    # Пн-Пт, 18:00
      desiredReplicas: "10"
  # Ночные batch jobs: 5 реплик
  - type: cron
    metadata:
      timezone: Europe/Moscow
      start: 0 2 * * *     # Каждый день в 2:00
      end: 0 5 * * *       # До 5:00
      desiredReplicas: "5"
```

### ScaledObject с множественными триггерами

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: multi-trigger-scaler
spec:
  scaleTargetRef:
    name: hybrid-worker
  pollingInterval: 10
  cooldownPeriod: 60
  minReplicaCount: 1
  maxReplicaCount: 50
  triggers:
  # HTTP нагрузка
  - type: prometheus
    metadata:
      serverAddress: http://prometheus:9090
      query: sum(rate(http_requests_total[1m]))
      threshold: "500"
  # Длина очереди
  - type: redis
    metadata:
      address: redis.default.svc:6379
      listName: tasks
      listLength: "100"
    authenticationRef:
      name: redis-auth
  # CPU fallback
  - type: cpu
    metricType: Utilization
    metadata:
      value: "70"
```

### ScaledJob для batch-обработки

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: batch-job-scaler
spec:
  jobTargetRef:
    parallelism: 1
    completions: 1
    backoffLimit: 3
    template:
      spec:
        containers:
        - name: processor
          image: batch-processor:latest
          env:
          - name: QUEUE_NAME
            value: batch-tasks
        restartPolicy: Never
  pollingInterval: 30
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  maxReplicaCount: 10
  scalingStrategy:
    strategy: accurate   # Точное количество job'ов = количество сообщений
  triggers:
  - type: rabbitmq
    metadata:
      queueName: batch-tasks
      mode: QueueLength
      value: "1"
    authenticationRef:
      name: rabbitmq-auth
```

---

## Настройка кастомных метрик

### Prometheus Adapter

Для использования кастомных метрик с HPA нужен Prometheus Adapter:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  --namespace monitoring \
  -f custom-metrics-config.yaml
```

### Конфигурация Prometheus Adapter

```yaml
# custom-metrics-config.yaml
prometheus:
  url: http://prometheus.monitoring.svc
  port: 9090

rules:
  default: false
  custom:
  # HTTP RPS метрика
  - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      matches: "^(.*)_total$"
      as: "${1}_per_second"
    metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'

  # Кастомная бизнес-метрика
  - seriesQuery: 'active_sessions{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      matches: "^(.*)$"
      as: "$1"
    metricsQuery: 'avg(<<.Series>>{<<.LabelMatchers>>}) by (<<.GroupBy>>)'

  # Queue depth для workers
  - seriesQuery: 'queue_messages_ready{queue_name!=""}'
    resources:
      template: <<.Resource>>
    name:
      matches: ".*"
      as: "queue_depth"
    metricsQuery: 'sum(<<.Series>>{<<.LabelMatchers>>})'
```

### Проверка кастомных метрик

```bash
# Список доступных кастомных метрик
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq

# Получить конкретную метрику для пода
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests_per_second" | jq

# Внешние метрики
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1" | jq
```

### HPA с кастомными метриками

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: custom-metrics-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-service
  minReplicas: 2
  maxReplicas: 50
  metrics:
  # Кастомная метрика: HTTP requests per second
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 1000
  # Кастомная метрика: Active sessions
  - type: Object
    object:
      metric:
        name: active_sessions
      describedObject:
        apiVersion: v1
        kind: Service
        name: web-service
      target:
        type: Value
        value: 1000
```

---

## Команды kubectl для работы с автоскейлингом

### Работа с HPA

```bash
# Создание HPA императивно
kubectl autoscale deployment web-app --cpu-percent=50 --min=2 --max=20

# Список всех HPA
kubectl get hpa
kubectl get hpa -A  # Во всех namespace

# Детальная информация
kubectl describe hpa web-app-hpa

# Текущие метрики и состояние
kubectl get hpa web-app-hpa -o yaml

# Наблюдение за изменениями в реальном времени
kubectl get hpa -w

# Удаление HPA
kubectl delete hpa web-app-hpa

# Редактирование HPA
kubectl edit hpa web-app-hpa

# Получить события связанные с HPA
kubectl get events --field-selector involvedObject.kind=HorizontalPodAutoscaler
```

### Работа с VPA

```bash
# Список VPA
kubectl get vpa
kubectl get vpa -A

# Детали и рекомендации
kubectl describe vpa backend-vpa

# Просмотр рекомендаций в JSON
kubectl get vpa backend-vpa -o jsonpath='{.status.recommendation}'

# Форматированный вывод рекомендаций
kubectl get vpa backend-vpa -o jsonpath='{range .status.recommendation.containerRecommendations[*]}{.containerName}: CPU={.target.cpu}, Memory={.target.memory}{"\n"}{end}'
```

### Отладка автоскейлинга

```bash
# Проверка Metrics Server
kubectl top nodes
kubectl top pods

# Проверка API metrics
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes"
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/pods"

# Логи HPA контроллера
kubectl logs -n kube-system -l app=kube-controller-manager | grep -i horizontal

# Логи Cluster Autoscaler
kubectl logs -n kube-system -l app=cluster-autoscaler -f

# Логи KEDA
kubectl logs -n keda -l app=keda-operator -f

# События кластера
kubectl get events --sort-by='.lastTimestamp' | grep -i scale

# Проверка pending подов (для Cluster Autoscaler)
kubectl get pods --all-namespaces --field-selector=status.phase=Pending
```

### Тестирование автоскейлинга

```bash
# Генерация нагрузки для тестирования HPA
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://web-app; done"

# Мониторинг во время теста
watch kubectl get hpa,pods

# Проверка статуса развёртывания
kubectl rollout status deployment/web-app
```

---

## Best Practices

### 1. Настройка ресурсов

```yaml
# Всегда устанавливайте requests для HPA по CPU/Memory
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            cpu: 100m      # Обязательно для HPA по CPU!
            memory: 128Mi  # Обязательно для HPA по Memory!
          limits:
            cpu: 500m
            memory: 512Mi
```

### 2. Правильные пороги

```yaml
# CPU utilization 50-70% — оптимальный диапазон
# Слишком низкий (30%) — преждевременный scale up, перерасход ресурсов
# Слишком высокий (90%) — приложение может не успеть масштабироваться
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 60  # Рекомендуемое значение
```

### 3. Стабилизация для production

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 60    # Не спешить с масштабированием вверх
    policies:
    - type: Percent
      value: 50                        # Максимум +50% за период
      periodSeconds: 60
  scaleDown:
    stabilizationWindowSeconds: 300   # 5 минут стабилизации
    policies:
    - type: Percent
      value: 10                        # Консервативное уменьшение
      periodSeconds: 120
```

### 4. Минимальные реплики для High Availability

```yaml
# Для production сервисов минимум 2-3 реплики
spec:
  minReplicas: 3  # Не 1! Иначе нет HA
  maxReplicas: 100
```

### 5. Readiness Probes обязательны

```yaml
# Новые поды должны быть ready перед получением трафика
spec:
  containers:
  - name: app
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 3
```

### 6. Pod Disruption Budget

```yaml
# Защита от агрессивного scale down
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 2  # Или maxUnavailable: 1
  selector:
    matchLabels:
      app: web-app
```

### 7. Preemptible/Spot ноды для экономии

```yaml
# Cluster Autoscaler: приоритет spot инстансам
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-priority-expander
  namespace: kube-system
data:
  priorities: |-
    10:
      - .*spot.*
    50:
      - .*on-demand.*
```

---

## Типичные ошибки

### 1. Отсутствие resource requests

```yaml
# НЕПРАВИЛЬНО: HPA не будет работать без requests
containers:
- name: app
  image: myapp:latest
  # resources не указаны!

# ПРАВИЛЬНО
containers:
- name: app
  image: myapp:latest
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
```

### 2. Конфликт VPA и HPA

```yaml
# НЕПРАВИЛЬНО: VPA и HPA оба управляют CPU
# VPA
controlledResources: [cpu, memory]
# HPA
metrics:
- type: Resource
  resource:
    name: cpu  # Конфликт!

# ПРАВИЛЬНО: Разделить ответственность
# VPA управляет только memory
controlledResources: [memory]
# HPA масштабирует по CPU или кастомным метрикам
```

### 3. minReplicas: 0 без KEDA

```yaml
# HPA не может масштабировать до 0!
# Минимум для HPA: 1
spec:
  minReplicas: 0  # Ошибка!

# Для scale-to-zero используйте KEDA
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
spec:
  minReplicaCount: 0  # KEDA поддерживает!
```

### 4. Игнорирование cooldown

```yaml
# НЕПРАВИЛЬНО: Нет стабилизации — постоянные колебания
behavior:
  scaleDown:
    stabilizationWindowSeconds: 0

# ПРАВИЛЬНО: Минимум 5 минут для production
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300
```

### 5. Неправильный target для метрик

```yaml
# НЕПРАВИЛЬНО: Utilization для memory обычно плохая идея
# Приложения часто держат память allocated даже без нагрузки
metrics:
- type: Resource
  resource:
    name: memory
    target:
      type: Utilization
      averageUtilization: 70

# ПРАВИЛЬНО: Использовать AverageValue для memory
metrics:
- type: Resource
  resource:
    name: memory
    target:
      type: AverageValue
      averageValue: 500Mi
```

### 6. Отсутствие readiness probe

```yaml
# НЕПРАВИЛЬНО: Без readiness probe поды получают трафик до готовности
# Это может вызывать каскадный scale up

# ПРАВИЛЬНО: Всегда настраивайте readiness probe
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### 7. Слишком агрессивный scale up

```yaml
# НЕПРАВИЛЬНО: Мгновенное утроение подов при пике
behavior:
  scaleUp:
    policies:
    - type: Percent
      value: 200
      periodSeconds: 15

# ПРАВИЛЬНО: Постепенное наращивание
behavior:
  scaleUp:
    stabilizationWindowSeconds: 30
    policies:
    - type: Percent
      value: 50
      periodSeconds: 60
    - type: Pods
      value: 4
      periodSeconds: 60
    selectPolicy: Max
```

---

## Мониторинг автоскейлинга

### Prometheus метрики для HPA

```yaml
# Полезные метрики для дашбордов
- kube_horizontalpodautoscaler_status_current_replicas
- kube_horizontalpodautoscaler_status_desired_replicas
- kube_horizontalpodautoscaler_spec_max_replicas
- kube_horizontalpodautoscaler_spec_min_replicas
- kube_horizontalpodautoscaler_status_condition{condition="ScalingActive"}
- kube_horizontalpodautoscaler_status_condition{condition="AbleToScale"}
```

### Grafana Dashboard queries

```promql
# Текущие vs желаемые реплики
sum by (horizontalpodautoscaler) (
  kube_horizontalpodautoscaler_status_current_replicas
)
vs
sum by (horizontalpodautoscaler) (
  kube_horizontalpodautoscaler_status_desired_replicas
)

# Процент утилизации относительно лимитов
100 * (
  kube_horizontalpodautoscaler_status_current_replicas
  / kube_horizontalpodautoscaler_spec_max_replicas
)
```

### Алерты

```yaml
groups:
- name: autoscaling
  rules:
  - alert: HPAMaxedOut
    expr: |
      kube_horizontalpodautoscaler_status_current_replicas
      == kube_horizontalpodautoscaler_spec_max_replicas
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "HPA {{ $labels.horizontalpodautoscaler }} at max replicas"

  - alert: HPAScalingIssue
    expr: |
      kube_horizontalpodautoscaler_status_condition{condition="ScalingActive",status="false"} == 1
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "HPA {{ $labels.horizontalpodautoscaler }} cannot scale"
```

---

## Полезные ресурсы

- [Kubernetes HPA Documentation](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [VPA GitHub Repository](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [Cluster Autoscaler FAQ](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md)
- [KEDA Documentation](https://keda.sh/docs/)
- [Prometheus Adapter](https://github.com/kubernetes-sigs/prometheus-adapter)
