# Prometheus в Kubernetes

## Введение

Prometheus является де-факто стандартом мониторинга для Kubernetes-кластеров. Благодаря своей pull-based модели сбора метрик и нативной поддержке Kubernetes через Service Discovery, Prometheus идеально подходит для динамичной среды контейнерной оркестрации.

## Архитектура Prometheus в Kubernetes

### Основные компоненты

```
┌─────────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                 Monitoring Namespace                      │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │   │
│  │  │ Prometheus  │  │ Alertmanager│  │    Grafana      │  │   │
│  │  │   Server    │  │             │  │                 │  │   │
│  │  └──────┬──────┘  └──────┬──────┘  └────────┬────────┘  │   │
│  │         │                │                   │           │   │
│  │         │         ┌──────┴──────┐           │           │   │
│  │         │         │   Rules     │           │           │   │
│  │         │         └─────────────┘           │           │   │
│  └─────────┼───────────────────────────────────┼───────────┘   │
│            │                                   │                │
│  ┌─────────┴───────────────────────────────────┴───────────┐   │
│  │              Kubernetes Service Discovery                │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │   │
│  │  │kube-state-  │  │   cAdvisor  │  │   Application   │  │   │
│  │  │  metrics    │  │  (kubelet)  │  │     Pods        │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Prometheus Operator

Prometheus Operator - это Kubernetes-контроллер, который значительно упрощает развёртывание и управление Prometheus в кластере.

**Custom Resource Definitions (CRDs):**

| CRD | Описание |
|-----|----------|
| `Prometheus` | Определяет экземпляр Prometheus-сервера |
| `ServiceMonitor` | Декларативно описывает мониторинг сервисов |
| `PodMonitor` | Мониторинг подов без Service |
| `PrometheusRule` | Правила алертинга и записи |
| `Alertmanager` | Экземпляр Alertmanager |
| `AlertmanagerConfig` | Конфигурация маршрутизации алертов |

## Prometheus CRD

### Базовое определение Prometheus

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  # Версия Prometheus
  version: v2.47.0

  # Количество реплик для HA
  replicas: 2

  # Сервисный аккаунт с нужными правами
  serviceAccountName: prometheus

  # Селектор ServiceMonitor
  serviceMonitorSelector:
    matchLabels:
      release: prometheus

  # Селектор для namespaces
  serviceMonitorNamespaceSelector:
    matchLabels:
      monitoring: enabled

  # Селектор PodMonitor
  podMonitorSelector:
    matchLabels:
      release: prometheus

  # Селектор PrometheusRule
  ruleSelector:
    matchLabels:
      release: prometheus

  # Настройки хранения
  retention: 15d
  retentionSize: 50GB

  # Persistent storage
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: standard
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 100Gi

  # Ресурсы
  resources:
    requests:
      cpu: 500m
      memory: 2Gi
    limits:
      cpu: 2000m
      memory: 8Gi

  # Affinity для распределения реплик
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              prometheus: prometheus
          topologyKey: kubernetes.io/hostname

  # Включение remote write
  remoteWrite:
    - url: http://thanos-receive:19291/api/v1/receive
      queueConfig:
        capacity: 10000
        maxSamplesPerSend: 1000

  # Внешние labels
  externalLabels:
    cluster: production
    region: eu-west-1
```

### RBAC для Prometheus

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
  # Доступ к метрикам API сервера
  - apiGroups: [""]
    resources:
      - nodes
      - nodes/metrics
      - nodes/proxy
      - services
      - endpoints
      - pods
    verbs: ["get", "list", "watch"]

  # Доступ к configmaps для service discovery
  - apiGroups: [""]
    resources:
      - configmaps
    verbs: ["get"]

  # Доступ к networking resources
  - apiGroups: ["networking.k8s.io"]
    resources:
      - ingresses
    verbs: ["get", "list", "watch"]

  # Доступ к метрикам через API
  - nonResourceURLs: ["/metrics", "/metrics/cadvisor"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: monitoring
```

## Service Discovery в Kubernetes

### Типы Service Discovery

Prometheus поддерживает несколько типов автоматического обнаружения целей в Kubernetes:

```yaml
# Пример scrape_config для различных SD
scrape_configs:
  # Мониторинг API серверов
  - job_name: 'kubernetes-apiservers'
    kubernetes_sd_configs:
      - role: endpoints
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      insecure_skip_verify: true
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https

  # Мониторинг нод через kubelet
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
      - role: node
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      insecure_skip_verify: true
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics

  # Мониторинг подов с аннотациями
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Скрейпить только поды с аннотацией prometheus.io/scrape: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      # Путь из аннотации
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      # Порт из аннотации
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      # Добавить labels из pod
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      # Добавить namespace и pod name
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name
```

### Meta Labels

При Service Discovery Prometheus предоставляет метки для relabeling:

**Node SD (`role: node`):**
- `__meta_kubernetes_node_name`
- `__meta_kubernetes_node_label_<labelname>`
- `__meta_kubernetes_node_annotation_<annotationname>`
- `__meta_kubernetes_node_address_<type>` (InternalIP, ExternalIP, etc.)

**Pod SD (`role: pod`):**
- `__meta_kubernetes_pod_name`
- `__meta_kubernetes_pod_ip`
- `__meta_kubernetes_pod_label_<labelname>`
- `__meta_kubernetes_pod_annotation_<annotationname>`
- `__meta_kubernetes_pod_container_name`
- `__meta_kubernetes_pod_container_port_number`
- `__meta_kubernetes_pod_ready`
- `__meta_kubernetes_pod_phase`
- `__meta_kubernetes_namespace`

## Источники метрик в Kubernetes

### 1. kube-state-metrics

Генерирует метрики о состоянии Kubernetes-объектов:

```yaml
# Примеры метрик kube-state-metrics
kube_deployment_status_replicas_available{namespace="app", deployment="web"}
kube_pod_status_phase{namespace="app", pod="web-xxx", phase="Running"}
kube_pod_container_status_restarts_total{namespace="app", pod="web-xxx", container="app"}
kube_node_status_condition{node="node-1", condition="Ready", status="true"}
kube_persistentvolumeclaim_status_phase{namespace="app", persistentvolumeclaim="data", phase="Bound"}
```

### 2. cAdvisor (Container Advisor)

Встроен в kubelet, предоставляет метрики контейнеров:

```yaml
# Метрики потребления ресурсов
container_cpu_usage_seconds_total{container="app", pod="web-xxx", namespace="app"}
container_memory_usage_bytes{container="app", pod="web-xxx", namespace="app"}
container_network_receive_bytes_total{pod="web-xxx", namespace="app"}
container_fs_usage_bytes{container="app", pod="web-xxx", namespace="app"}
```

### 3. metrics-server

Предоставляет метрики для HPA и VPA через Metrics API:

```bash
# Получить метрики через kubectl
kubectl top pods -n app
kubectl top nodes
```

### 4. kubelet метрики

```yaml
# Метрики kubelet
kubelet_running_pods{node="node-1"}
kubelet_running_containers{node="node-1", container_state="running"}
kubelet_volume_stats_available_bytes{namespace="app", persistentvolumeclaim="data"}
```

## Best Practices

### 1. High Availability

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  replicas: 2
  # Thanos Sidecar для дедупликации
  thanos:
    version: v0.32.0
    objectStorageConfig:
      key: thanos.yaml
      name: thanos-objstore-config
```

### 2. Управление ресурсами

```yaml
# Правильный расчёт ресурсов
# Memory: ~2-3 GB на 100k активных time series
# CPU: зависит от количества targets и частоты scrape
resources:
  requests:
    cpu: "1"
    memory: 4Gi
  limits:
    cpu: "4"
    memory: 16Gi
```

### 3. Селективный мониторинг

```yaml
# Мониторинг только нужных namespace
spec:
  serviceMonitorNamespaceSelector:
    matchExpressions:
      - key: kubernetes.io/metadata.name
        operator: In
        values:
          - production
          - staging
```

## Распространённые ошибки

### 1. Недостаточные права RBAC

```bash
# Ошибка в логах Prometheus
level=error msg="Error listing pods" err="pods is forbidden:
User \"system:serviceaccount:monitoring:prometheus\" cannot list resource \"pods\""
```

**Решение:** Проверить ClusterRole и ClusterRoleBinding.

### 2. Проблемы с Service Discovery

```bash
# Проверка targets
kubectl port-forward -n monitoring svc/prometheus 9090:9090
# Открыть http://localhost:9090/targets
```

### 3. Большое потребление памяти

```yaml
# Добавить лимиты на количество series
spec:
  additionalScrapeConfigs:
    name: additional-scrape-configs
    key: prometheus-additional.yaml
# Использовать sample_limit и series_limit
```

## Проверка работоспособности

```bash
# Проверить статус Prometheus pods
kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus

# Проверить CRDs
kubectl get servicemonitors -A
kubectl get podmonitors -A
kubectl get prometheusrules -A

# Проверить логи
kubectl logs -n monitoring prometheus-prometheus-0 -c prometheus

# Проверить targets через API
kubectl exec -n monitoring prometheus-prometheus-0 -c prometheus -- \
  wget -qO- localhost:9090/api/v1/targets | jq '.data.activeTargets | length'
```

## Заключение

Prometheus в Kubernetes требует правильной настройки RBAC, понимания механизмов Service Discovery и выбора подходящих источников метрик. Использование Prometheus Operator значительно упрощает управление и позволяет декларативно описывать конфигурацию мониторинга через Kubernetes CRDs.
