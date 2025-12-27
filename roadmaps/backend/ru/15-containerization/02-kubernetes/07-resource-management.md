# Resource Management в Kubernetes

Управление ресурсами — критически важный аспект работы с Kubernetes. Правильная настройка ресурсов обеспечивает стабильную работу приложений, эффективное использование кластера и предотвращает проблемы с производительностью.

## Основные концепции

### CPU и Memory — два типа ресурсов

Kubernetes управляет двумя основными типами ресурсов:

| Ресурс | Единицы измерения | Особенности |
|--------|-------------------|-------------|
| **CPU** | millicores (m) или cores | Compressible — можно ограничить (throttle) |
| **Memory** | bytes (Ki, Mi, Gi) | Incompressible — нельзя ограничить, только убить pod |

```yaml
# Примеры единиц измерения
resources:
  cpu: "500m"      # 0.5 CPU core (500 millicores)
  cpu: "2"         # 2 CPU cores
  memory: "128Mi"  # 128 мебибайт
  memory: "1Gi"    # 1 гибибайт
```

**Важно:** 1 CPU в Kubernetes = 1 vCPU/Core на облачных платформах или 1 hyperthread на bare-metal.

---

## Requests и Limits

### Что такое Requests и Limits

- **Requests** — минимальное количество ресурсов, которое контейнеру гарантировано. Используется scheduler для размещения pod на node.
- **Limits** — максимальное количество ресурсов, которое контейнер может использовать.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
```

### Как работают Requests

1. **Scheduling**: Kubernetes scheduler выбирает node, где есть достаточно свободных ресурсов (на основе requests)
2. **Гарантия**: Pod гарантированно получит запрошенные ресурсы
3. **Overcommit**: Сумма requests всех pods на node может быть меньше реальной ёмкости

```bash
# Посмотреть allocatable ресурсы node
kubectl describe node <node-name> | grep -A 5 "Allocatable"

# Посмотреть использование ресурсов на node
kubectl describe node <node-name> | grep -A 10 "Allocated resources"
```

### Как работают Limits

1. **CPU Limits**: При превышении контейнер будет throttled (замедлен)
2. **Memory Limits**: При превышении контейнер будет OOMKilled (убит из-за нехватки памяти)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    resources:
      requests:
        memory: "100Mi"
      limits:
        memory: "200Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```

### Пример полного Deployment с ресурсами

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: myapp:v1.2.3
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        # Liveness и readiness probes
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 3
      # Init container тоже нуждается в ресурсах
      initContainers:
      - name: init-db
        image: busybox:1.36
        command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
```

---

## Quality of Service (QoS) классы

Kubernetes автоматически присваивает каждому pod один из трёх QoS классов на основе настроек ресурсов.

### 1. Guaranteed (Гарантированный)

**Условия:**
- Все контейнеры имеют requests и limits для CPU и memory
- Requests равны limits для каждого ресурса

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "256Mi"
        cpu: "500m"
      limits:
        memory: "256Mi"  # Равно requests
        cpu: "500m"      # Равно requests
```

**Характеристики:**
- Наивысший приоритет
- Последними попадают под eviction (вытеснение)
- Подходит для критически важных приложений

### 2. Burstable (Пакетный)

**Условия:**
- Pod не попадает под Guaranteed
- Хотя бы один контейнер имеет request или limit для CPU или memory

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"  # Больше чем requests
        cpu: "500m"      # Больше чем requests
```

**Характеристики:**
- Средний приоритет
- Может использовать ресурсы выше requests, если они доступны
- Вытесняется перед Guaranteed, но после BestEffort

### 3. BestEffort (По возможности)

**Условия:**
- Ни один контейнер не имеет requests или limits

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
spec:
  containers:
  - name: app
    image: nginx:1.25
    # Нет секции resources
```

**Характеристики:**
- Самый низкий приоритет
- Первыми попадают под eviction при нехватке ресурсов
- Используется для некритичных задач

### Проверка QoS класса

```bash
# Посмотреть QoS класс pod
kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'

# Или через describe
kubectl describe pod <pod-name> | grep "QoS Class"
```

### Сравнительная таблица QoS классов

| QoS Class | Requests | Limits | Приоритет eviction | Использование |
|-----------|----------|--------|-------------------|---------------|
| **Guaranteed** | Установлены | = Requests | Низкий (последние) | Production workloads |
| **Burstable** | Установлены | > Requests или частично | Средний | Типичные приложения |
| **BestEffort** | Нет | Нет | Высокий (первые) | Dev/test, batch jobs |

---

## ResourceQuota

ResourceQuota ограничивает общее количество ресурсов, которые могут быть использованы в namespace.

### Базовый пример

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: development
spec:
  hard:
    # Ограничения по CPU и памяти
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"

    # Ограничения по количеству объектов
    pods: "20"
    services: "10"
    secrets: "20"
    configmaps: "20"
    persistentvolumeclaims: "10"

    # Ограничения по storage
    requests.storage: "100Gi"
```

### Расширенный пример с разными типами

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-counts
  namespace: production
spec:
  hard:
    # Количество pods по QoS классам
    count/pods: "50"

    # Количество различных объектов
    count/deployments.apps: "10"
    count/replicasets.apps: "20"
    count/statefulsets.apps: "5"
    count/jobs.batch: "10"
    count/cronjobs.batch: "5"
    count/services: "15"
    count/services.loadbalancers: "2"
    count/services.nodeports: "5"

    # Ingress
    count/ingresses.networking.k8s.io: "10"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: production
spec:
  hard:
    persistentvolumeclaims: "20"
    requests.storage: "500Gi"

    # Ограничения по storage class
    ssd.storageclass.storage.k8s.io/requests.storage: "200Gi"
    ssd.storageclass.storage.k8s.io/persistentvolumeclaims: "10"
```

### ResourceQuota для QoS классов

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: guaranteed-quota
  namespace: production
spec:
  hard:
    pods: "10"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["high-priority"]
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: besteffort-quota
  namespace: development
spec:
  hard:
    pods: "5"
  scopes:
  - BestEffort
```

### Проверка ResourceQuota

```bash
# Список квот в namespace
kubectl get resourcequota -n <namespace>

# Детальная информация
kubectl describe resourcequota compute-quota -n development

# Вывод в формате YAML
kubectl get resourcequota compute-quota -n development -o yaml
```

---

## LimitRange

LimitRange устанавливает дефолтные значения и ограничения для ресурсов в namespace.

### Базовый пример

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: development
spec:
  limits:
  - type: Container
    # Значения по умолчанию (если не указаны)
    default:
      cpu: "500m"
      memory: "256Mi"
    # Дефолтные requests (если не указаны)
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    # Максимальные значения
    max:
      cpu: "2"
      memory: "2Gi"
    # Минимальные значения
    min:
      cpu: "50m"
      memory: "64Mi"
    # Максимальное соотношение limit/request
    maxLimitRequestRatio:
      cpu: "10"
      memory: "4"
```

### LimitRange для разных типов объектов

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: comprehensive-limits
  namespace: production
spec:
  limits:
  # Для контейнеров
  - type: Container
    default:
      cpu: "200m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "4"
      memory: "8Gi"
    min:
      cpu: "50m"
      memory: "64Mi"

  # Для pods (сумма всех контейнеров)
  - type: Pod
    max:
      cpu: "8"
      memory: "16Gi"
    min:
      cpu: "100m"
      memory: "128Mi"

  # Для PersistentVolumeClaim
  - type: PersistentVolumeClaim
    max:
      storage: "100Gi"
    min:
      storage: "1Gi"
    default:
      storage: "10Gi"
```

### Проверка LimitRange

```bash
# Список LimitRange в namespace
kubectl get limitrange -n <namespace>

# Детальная информация
kubectl describe limitrange default-limits -n development
```

### Как работает LimitRange

1. Если pod создаётся без resources, применяются default значения
2. Если указаны только limits, defaultRequest применяется к requests
3. Если указаны только requests, default применяется к limits
4. Создание pod будет отклонено, если значения выходят за min/max

---

## Как правильно рассчитать ресурсы

### Методология определения ресурсов

#### 1. Начните с мониторинга

```bash
# Использование ресурсов pods
kubectl top pods -n <namespace>

# Использование ресурсов nodes
kubectl top nodes

# История использования ресурсов (требуется Prometheus/Grafana)
# Или использовать metrics-server
```

#### 2. Анализ приложения

```bash
# Локальное профилирование
# Для JVM приложений
java -Xms256m -Xmx512m -jar app.jar

# Для Node.js
node --max-old-space-size=512 app.js

# Для Go (анализ pprof)
go tool pprof http://localhost:6060/debug/pprof/heap
```

### Формулы расчёта

#### CPU

```
CPU Request = Средняя загрузка за период наблюдения + 10-20% буфер
CPU Limit = Пиковая загрузка + 20-30% буфер (или = Request для Guaranteed)
```

#### Memory

```
Memory Request = Стабильное потребление памяти + 10-20% буфер
Memory Limit = Максимальное потребление + буфер для GC (если применимо)
```

### Примеры для разных типов приложений

#### Web-сервер (Node.js/Python)

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

#### Java приложение (Spring Boot)

```yaml
resources:
  requests:
    memory: "512Mi"    # JVM Heap + Metaspace + Native
    cpu: "200m"
  limits:
    memory: "1Gi"      # Учитываем GC overhead
    cpu: "1000m"
```

#### База данных (PostgreSQL)

```yaml
resources:
  requests:
    memory: "1Gi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "2000m"
```

#### Batch Job

```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "2000m"   # Позволяем burst для быстрого завершения
```

### Vertical Pod Autoscaler (VPA) для рекомендаций

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Off"  # Только рекомендации, без автоизменения
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: "50m"
        memory: "64Mi"
      maxAllowed:
        cpu: "2"
        memory: "4Gi"
```

```bash
# Посмотреть рекомендации VPA
kubectl describe vpa my-app-vpa
```

---

## Мониторинг использования ресурсов

### Встроенные инструменты

#### kubectl top

```bash
# Требуется установленный metrics-server

# Ресурсы pods
kubectl top pods -n production
kubectl top pods --all-namespaces
kubectl top pods -l app=web-app

# Ресурсы nodes
kubectl top nodes

# Сортировка по использованию
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory

# С контейнерами
kubectl top pods --containers
```

#### kubectl describe

```bash
# Ресурсы на node
kubectl describe node <node-name>

# Посмотреть requests/limits pod
kubectl describe pod <pod-name> | grep -A 10 "Limits\|Requests"

# Allocated resources на node
kubectl describe node <node-name> | grep -A 20 "Allocated resources"
```

### Prometheus метрики

```yaml
# Основные метрики для мониторинга

# Использование CPU контейнером
container_cpu_usage_seconds_total

# Использование памяти контейнером
container_memory_usage_bytes
container_memory_working_set_bytes

# Requests и Limits
kube_pod_container_resource_requests
kube_pod_container_resource_limits

# OOMKilled события
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}

# CPU throttling
container_cpu_cfs_throttled_seconds_total
```

### Grafana дашборды

```yaml
# Полезные запросы PromQL

# CPU использование vs request
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod)
/
sum(kube_pod_container_resource_requests{resource="cpu"}) by (pod)

# Memory использование vs limit
sum(container_memory_working_set_bytes{container!=""}) by (pod)
/
sum(kube_pod_container_resource_limits{resource="memory"}) by (pod)

# Pods с высоким CPU throttling
sum(rate(container_cpu_cfs_throttled_seconds_total[5m])) by (pod) > 0.1
```

### Alerting rules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: resource-alerts
spec:
  groups:
  - name: resource.rules
    rules:
    - alert: PodHighMemoryUsage
      expr: |
        sum(container_memory_working_set_bytes{container!=""}) by (pod, namespace)
        /
        sum(kube_pod_container_resource_limits{resource="memory"}) by (pod, namespace)
        > 0.9
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} использует >90% memory limit"

    - alert: PodHighCPUThrottling
      expr: |
        sum(rate(container_cpu_cfs_throttled_seconds_total[5m])) by (pod, namespace)
        /
        sum(rate(container_cpu_cfs_periods_total[5m])) by (pod, namespace)
        > 0.25
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} throttled >25% времени"

    - alert: PodOOMKilled
      expr: |
        kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
      for: 0m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} был OOMKilled"
```

---

## Команды kubectl для работы с ресурсами

### Просмотр ресурсов

```bash
# Использование ресурсов pods
kubectl top pods -n <namespace>
kubectl top pods --all-namespaces --sort-by=memory

# Использование ресурсов nodes
kubectl top nodes

# Ресурсы конкретного pod
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].resources}'

# Форматированный вывод
kubectl get pods -n <namespace> -o custom-columns=\
"NAME:.metadata.name,\
CPU_REQ:.spec.containers[0].resources.requests.cpu,\
CPU_LIM:.spec.containers[0].resources.limits.cpu,\
MEM_REQ:.spec.containers[0].resources.requests.memory,\
MEM_LIM:.spec.containers[0].resources.limits.memory"
```

### Работа с ResourceQuota

```bash
# Создать
kubectl apply -f quota.yaml

# Список
kubectl get resourcequota -n <namespace>

# Детали
kubectl describe resourcequota <name> -n <namespace>

# Удалить
kubectl delete resourcequota <name> -n <namespace>
```

### Работа с LimitRange

```bash
# Создать
kubectl apply -f limitrange.yaml

# Список
kubectl get limitrange -n <namespace>

# Детали
kubectl describe limitrange <name> -n <namespace>

# Удалить
kubectl delete limitrange <name> -n <namespace>
```

### Анализ node capacity

```bash
# Подробная информация о ресурсах node
kubectl describe node <node-name> | grep -A 20 "Capacity\|Allocatable\|Allocated"

# JSON вывод capacity
kubectl get node <node-name> -o jsonpath='{.status.capacity}'

# Все nodes с capacity
kubectl get nodes -o custom-columns=\
"NAME:.metadata.name,\
CPU:.status.capacity.cpu,\
MEMORY:.status.capacity.memory,\
PODS:.status.capacity.pods"
```

### Проверка events

```bash
# События связанные с ресурсами
kubectl get events -n <namespace> --field-selector reason=FailedScheduling
kubectl get events -n <namespace> --field-selector reason=OOMKilling
kubectl get events -n <namespace> --field-selector reason=Evicted

# Все события pod
kubectl describe pod <pod-name> | grep -A 20 "Events:"
```

---

## Best Practices

### 1. Всегда устанавливайте requests и limits

```yaml
# ХОРОШО
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"

# ПЛОХО - нет ресурсов
# containers:
# - name: app
#   image: myapp
```

### 2. Начинайте с консервативных значений

```yaml
# Начните с небольших значений и увеличивайте по мере необходимости
resources:
  requests:
    memory: "128Mi"
    cpu: "50m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

### 3. Используйте Guaranteed QoS для production

```yaml
# Для критичных приложений requests = limits
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### 4. Учитывайте особенности языка

```yaml
# Java - учитывайте JVM overhead
# Heap = 75% от memory limit
resources:
  requests:
    memory: "1Gi"   # -Xmx768m для heap
  limits:
    memory: "1Gi"

# Go - низкий overhead
resources:
  requests:
    memory: "128Mi"
  limits:
    memory: "256Mi"
```

### 5. Настраивайте LimitRange для namespace

```yaml
# Защита от pods без ресурсов
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    default:
      cpu: "200m"
      memory: "256Mi"
```

### 6. Используйте ResourceQuota для multi-tenant кластеров

```yaml
# Ограничение ресурсов для команды/проекта
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
```

### 7. Мониторьте и оптимизируйте

```bash
# Регулярно проверяйте использование
kubectl top pods -n production

# Ищите pods с низким использованием (overprovisioned)
# Ищите pods с высоким throttling (underprovisioned)
```

---

## Типичные ошибки

### 1. Слишком высокие limits без requests

```yaml
# ПЛОХО - неэффективное scheduling
resources:
  limits:
    memory: "4Gi"
    cpu: "2"
# requests будут равны limits = перерасход

# ХОРОШО
resources:
  requests:
    memory: "512Mi"
    cpu: "200m"
  limits:
    memory: "1Gi"
    cpu: "500m"
```

### 2. Игнорирование CPU throttling

```bash
# Проверьте throttling
kubectl top pods
# Если CPU usage близок к limit, но приложение тормозит - throttling

# Решение: увеличьте CPU limit или уберите его
```

### 3. Неправильный расчёт памяти для JVM

```yaml
# ПЛОХО - memory limit = Xmx
# JVM использует больше памяти чем heap
env:
- name: JAVA_OPTS
  value: "-Xmx1g"
resources:
  limits:
    memory: "1Gi"  # OOMKilled!

# ХОРОШО - memory limit > Xmx
env:
- name: JAVA_OPTS
  value: "-Xmx768m -XX:MaxMetaspaceSize=128m"
resources:
  limits:
    memory: "1Gi"  # 768m + 128m + native = ~1Gi
```

### 4. Одинаковые ресурсы для разных workloads

```yaml
# ПЛОХО - один размер для всех
# Web server и batch job с одинаковыми ресурсами

# ХОРОШО - разные профили
# Web server
resources:
  requests:
    cpu: "100m"
    memory: "256Mi"

# Batch job
resources:
  requests:
    cpu: "500m"
    memory: "1Gi"
```

### 5. Отсутствие LimitRange в namespace

```bash
# Pods без resources попадают в BestEffort
# и будут первыми evicted при нехватке памяти

# Решение: всегда создавайте LimitRange
kubectl apply -f - <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: default
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    default:
      cpu: "200m"
      memory: "256Mi"
EOF
```

### 6. Слишком маленький memory limit

```bash
# Симптомы: частые OOMKilled, рестарты

# Диагностика
kubectl describe pod <pod-name> | grep -i oom
kubectl get events --field-selector reason=OOMKilling

# Решение: увеличьте memory limit
```

### 7. CPU limit без понимания throttling

```yaml
# CPU limits вызывают throttling даже при свободном CPU на node
# Это увеличивает latency

# Для latency-sensitive приложений
# Вариант 1: убрать CPU limit
resources:
  requests:
    cpu: "500m"
  # Нет CPU limit

# Вариант 2: использовать Guaranteed QoS
resources:
  requests:
    cpu: "1"
  limits:
    cpu: "1"
```

---

## Полезные скрипты

### Отчёт по использованию ресурсов

```bash
#!/bin/bash
# resource-report.sh

echo "=== Node Resources ==="
kubectl top nodes

echo -e "\n=== Pods by CPU Usage ==="
kubectl top pods --all-namespaces --sort-by=cpu | head -20

echo -e "\n=== Pods by Memory Usage ==="
kubectl top pods --all-namespaces --sort-by=memory | head -20

echo -e "\n=== ResourceQuotas ==="
kubectl get resourcequota --all-namespaces

echo -e "\n=== LimitRanges ==="
kubectl get limitrange --all-namespaces
```

### Поиск pods без ресурсов

```bash
#!/bin/bash
# find-pods-without-resources.sh

kubectl get pods --all-namespaces -o json | jq -r '
  .items[] |
  select(.spec.containers[].resources.requests == null or .spec.containers[].resources.limits == null) |
  "\(.metadata.namespace)/\(.metadata.name)"
'
```

---

## Заключение

Правильное управление ресурсами в Kubernetes включает:

1. **Установку requests и limits** для всех контейнеров
2. **Выбор правильного QoS класса** в зависимости от критичности приложения
3. **Использование ResourceQuota** для ограничения ресурсов namespace
4. **Настройку LimitRange** для дефолтных значений
5. **Мониторинг и оптимизацию** на основе реального использования
6. **Избежание типичных ошибок** с памятью и CPU

Ключевой принцип: начинайте с мониторинга, устанавливайте консервативные значения, и оптимизируйте итеративно.
