# Monitoring and Logging в Kubernetes

Мониторинг и логирование — критически важные компоненты production-ready кластера Kubernetes. Они позволяют отслеживать состояние приложений, выявлять проблемы и анализировать производительность.

## Содержание

1. [Встроенные инструменты](#встроенные-инструменты)
2. [Metrics Server](#metrics-server)
3. [Prometheus и Grafana](#prometheus-и-grafana)
4. [Логирование в Kubernetes](#логирование-в-kubernetes)
5. [EFK/ELK стек](#efkelk-стек)
6. [Health Probes и мониторинг](#health-probes-и-мониторинг)
7. [Alerting и уведомления](#alerting-и-уведомления)
8. [Best Practices](#best-practices)
9. [Полезные команды kubectl](#полезные-команды-kubectl)

---

## Встроенные инструменты

Kubernetes предоставляет базовые инструменты для мониторинга и просмотра логов прямо из коробки.

### kubectl logs

Команда для просмотра логов контейнеров:

```bash
# Логи пода
kubectl logs my-pod

# Логи конкретного контейнера в поде
kubectl logs my-pod -c my-container

# Последние 100 строк логов
kubectl logs my-pod --tail=100

# Логи за последний час
kubectl logs my-pod --since=1h

# Логи за последние 30 минут
kubectl logs my-pod --since=30m

# Следить за логами в реальном времени (streaming)
kubectl logs -f my-pod

# Логи предыдущего инстанса контейнера (после рестарта)
kubectl logs my-pod --previous

# Логи всех подов с определённым лейблом
kubectl logs -l app=nginx

# Логи из всех контейнеров пода
kubectl logs my-pod --all-containers=true

# Временные метки в логах
kubectl logs my-pod --timestamps=true
```

### kubectl top

Команда для просмотра потребления ресурсов (требует Metrics Server):

```bash
# Потребление ресурсов подами
kubectl top pods

# Потребление ресурсов в namespace
kubectl top pods -n my-namespace

# Потребление ресурсов нодами
kubectl top nodes

# Сортировка по CPU
kubectl top pods --sort-by=cpu

# Сортировка по памяти
kubectl top pods --sort-by=memory

# Потребление ресурсов конкретного пода
kubectl top pod my-pod

# Показать потребление по контейнерам
kubectl top pods --containers
```

### kubectl describe

Полезная информация о состоянии ресурсов:

```bash
# Детальная информация о поде
kubectl describe pod my-pod

# События в namespace
kubectl get events --sort-by='.lastTimestamp'

# События для конкретного пода
kubectl get events --field-selector involvedObject.name=my-pod

# Отслеживание событий в реальном времени
kubectl get events -w
```

---

## Metrics Server

Metrics Server — компонент Kubernetes для сбора метрик ресурсов (CPU, память) с kubelet на каждой ноде.

### Установка Metrics Server

```bash
# Установка через kubectl
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Проверка установки
kubectl get deployment metrics-server -n kube-system

# Проверка работоспособности
kubectl top nodes
```

### Конфигурация для локального кластера (Minikube, Kind)

Для локальных кластеров может потребоваться отключение проверки TLS:

```yaml
# metrics-server-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: metrics-server
        args:
        - --cert-dir=/tmp
        - --secure-port=10250
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls  # Для локальной разработки
```

### Minikube

```bash
# Включение metrics-server в Minikube
minikube addons enable metrics-server

# Проверка
minikube addons list | grep metrics
```

### Проверка работы Metrics Server

```bash
# API метрик нод
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes

# API метрик подов
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods

# Проверка логов metrics-server
kubectl logs -n kube-system -l k8s-app=metrics-server
```

---

## Prometheus и Grafana

Prometheus — система мониторинга и алертинга, де-факто стандарт для Kubernetes. Grafana — инструмент визуализации метрик.

### Архитектура Prometheus

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                        │
│                                                                   │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐        │
│  │   Pod       │     │   Pod       │     │   Pod       │        │
│  │  /metrics   │     │  /metrics   │     │  /metrics   │        │
│  └──────┬──────┘     └──────┬──────┘     └──────┬──────┘        │
│         │                   │                   │                │
│         └───────────────────┼───────────────────┘                │
│                             │                                     │
│                    ┌────────▼────────┐                           │
│                    │   Prometheus    │                           │
│                    │  (scrape, store)│                           │
│                    └────────┬────────┘                           │
│                             │                                     │
│              ┌──────────────┼──────────────┐                     │
│              │              │              │                      │
│       ┌──────▼──────┐ ┌─────▼─────┐ ┌─────▼──────┐              │
│       │   Grafana   │ │Alertmanager│ │   API     │              │
│       │(dashboards) │ │ (alerts)  │ │ (queries) │               │
│       └─────────────┘ └───────────┘ └────────────┘              │
└─────────────────────────────────────────────────────────────────┘
```

### Установка через Helm (kube-prometheus-stack)

```bash
# Добавление репозитория
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Создание namespace
kubectl create namespace monitoring

# Установка kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=admin123

# Проверка установки
kubectl get pods -n monitoring
```

### Доступ к интерфейсам

```bash
# Prometheus UI
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090

# Grafana UI (default: admin/prom-operator или заданный пароль)
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Alertmanager UI
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-alertmanager 9093:9093
```

### Конфигурация Prometheus (values.yaml)

```yaml
# prometheus-values.yaml
prometheus:
  prometheusSpec:
    # Retention period
    retention: 15d

    # Storage
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi

    # Resources
    resources:
      requests:
        memory: 400Mi
        cpu: 200m
      limits:
        memory: 2Gi
        cpu: 1000m

    # Additional scrape configs
    additionalScrapeConfigs:
    - job_name: 'my-app'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__

grafana:
  adminPassword: "securePassword123"
  persistence:
    enabled: true
    size: 10Gi

  # Дополнительные data sources
  additionalDataSources:
  - name: Loki
    type: loki
    url: http://loki:3100
    access: proxy

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
```

### ServiceMonitor для приложения

```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: monitoring
  labels:
    release: prometheus  # Важно для обнаружения Prometheus
spec:
  selector:
    matchLabels:
      app: my-app
  namespaceSelector:
    matchNames:
    - default
    - production
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
    scheme: http
```

### PodMonitor для приложения

```yaml
# podmonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: my-app-pods
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: my-app
  podMetricsEndpoints:
  - port: metrics
    interval: 15s
```

### Пример приложения с метриками (Python/FastAPI)

```python
# main.py
from fastapi import FastAPI
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST
from starlette.responses import Response
import time

app = FastAPI()

# Метрики
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint']
)

@app.middleware("http")
async def metrics_middleware(request, call_next):
    start_time = time.time()
    response = await call_next(request)

    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()

    REQUEST_LATENCY.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(time.time() - start_time)

    return response

@app.get("/metrics")
async def metrics():
    return Response(
        content=generate_latest(),
        media_type=CONTENT_TYPE_LATEST
    )

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

### Полезные PromQL запросы

```promql
# CPU usage по подам
sum(rate(container_cpu_usage_seconds_total{namespace="default"}[5m])) by (pod)

# Memory usage по подам
sum(container_memory_usage_bytes{namespace="default"}) by (pod)

# HTTP request rate
sum(rate(http_requests_total[5m])) by (endpoint)

# 95-й перцентиль латентности
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint))

# Error rate
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100

# Pod restarts
sum(increase(kube_pod_container_status_restarts_total[1h])) by (pod)

# Доступность нод
sum(kube_node_status_condition{condition="Ready",status="true"})

# Использование CPU нодой (%)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Свободная память на ноде
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100
```

---

## Логирование в Kubernetes

### Архитектура логирования

В Kubernetes логи контейнеров собираются из stdout/stderr потоков.

```
┌─────────────────────────────────────────────────────────────────┐
│                              Node                                │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                         Pod                              │    │
│  │  ┌─────────────┐    ┌─────────────┐                     │    │
│  │  │ Container   │    │ Container   │                     │    │
│  │  │ stdout ─────┼────┼─► kubelet   │                     │    │
│  │  │ stderr ─────┼────┼─► (logs)    │                     │    │
│  │  └─────────────┘    └─────────────┘                     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                │                                  │
│                                ▼                                  │
│                    /var/log/containers/*.log                     │
│                    /var/log/pods/*/*.log                         │
│                                │                                  │
│                                ▼                                  │
│                    ┌─────────────────────┐                       │
│                    │  Log Collector      │                       │
│                    │  (Fluentd/Fluent Bit)│                      │
│                    └──────────┬──────────┘                       │
└───────────────────────────────┼──────────────────────────────────┘
                                │
                                ▼
                    ┌─────────────────────┐
                    │  Centralized Storage │
                    │  (Elasticsearch/Loki)│
                    └─────────────────────┘
```

### Логирование в stdout/stderr

Правильный подход — писать логи в stdout/stderr:

```python
# Python пример
import logging
import sys

# Настройка логгера для stdout
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[logging.StreamHandler(sys.stdout)]
)

logger = logging.getLogger(__name__)

logger.info("Application started")
logger.error("An error occurred", exc_info=True)
```

```javascript
// Node.js пример
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console()
  ]
});

logger.info('Application started');
logger.error('An error occurred', { error: err.message });
```

### JSON формат логов

Рекомендуется использовать структурированные логи в JSON формате:

```json
{
  "timestamp": "2024-01-15T10:30:00.000Z",
  "level": "INFO",
  "message": "Request processed",
  "service": "api-gateway",
  "trace_id": "abc123",
  "duration_ms": 45,
  "status_code": 200
}
```

### Sidecar-контейнеры для логирования

Sidecar-контейнеры используются когда приложение пишет логи в файлы:

```yaml
# sidecar-logging.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
  # Основной контейнер приложения
  - name: app
    image: my-app:latest
    volumeMounts:
    - name: logs
      mountPath: /var/log/app

  # Sidecar для логов приложения
  - name: log-streamer
    image: busybox:latest
    args:
    - /bin/sh
    - -c
    - tail -F /var/log/app/app.log
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
      readOnly: true

  volumes:
  - name: logs
    emptyDir: {}
```

### Sidecar с несколькими лог-файлами

```yaml
# multi-log-sidecar.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-log-app
spec:
  containers:
  - name: app
    image: my-app:latest
    volumeMounts:
    - name: logs
      mountPath: /var/log/app

  # Sidecar для access логов
  - name: access-log-streamer
    image: busybox:latest
    args: ['/bin/sh', '-c', 'tail -F /var/log/app/access.log']
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
      readOnly: true

  # Sidecar для error логов
  - name: error-log-streamer
    image: busybox:latest
    args: ['/bin/sh', '-c', 'tail -F /var/log/app/error.log']
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
      readOnly: true

  volumes:
  - name: logs
    emptyDir: {}
```

---

## EFK/ELK стек

EFK (Elasticsearch, Fluentd/Fluent Bit, Kibana) и ELK (Elasticsearch, Logstash, Kibana) — стандартные решения для централизованного логирования.

### Сравнение компонентов

| Компонент | ELK | EFK | Назначение |
|-----------|-----|-----|------------|
| Collector | Logstash | Fluentd/Fluent Bit | Сбор и обработка логов |
| Storage | Elasticsearch | Elasticsearch | Хранение и индексация |
| UI | Kibana | Kibana | Визуализация и поиск |

### Fluent Bit (легковесная альтернатива Fluentd)

```yaml
# fluent-bit-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020

    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            docker
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On
        Refresh_Interval  10

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Keep_Log            Off
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On

    [OUTPUT]
        Name            es
        Match           *
        Host            elasticsearch
        Port            9200
        Logstash_Format On
        Logstash_Prefix kubernetes
        Retry_Limit     False
        Replace_Dots    On

  parsers.conf: |
    [PARSER]
        Name   docker
        Format json
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
        Time_Keep On

    [PARSER]
        Name        json
        Format      json
        Time_Key    timestamp
        Time_Format %Y-%m-%dT%H:%M:%S.%L
```

### Fluent Bit DaemonSet

```yaml
# fluent-bit-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
  labels:
    app: fluent-bit
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:latest
        ports:
        - containerPort: 2020
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: config
          mountPath: /fluent-bit/etc/
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: config
        configMap:
          name: fluent-bit-config
```

### Elasticsearch StatefulSet

```yaml
# elasticsearch.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: logging
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
        securityContext:
          privileged: true
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      - name: increase-vm-max-map
        image: busybox
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        env:
        - name: cluster.name
          value: k8s-logs
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: discovery.seed_hosts
          value: "elasticsearch-0.elasticsearch,elasticsearch-1.elasticsearch,elasticsearch-2.elasticsearch"
        - name: cluster.initial_master_nodes
          value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
        - name: ES_JAVA_OPTS
          value: "-Xms1g -Xmx1g"
        - name: xpack.security.enabled
          value: "false"
        resources:
          requests:
            cpu: 500m
            memory: 2Gi
          limits:
            cpu: 1000m
            memory: 4Gi
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Gi
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: logging
spec:
  selector:
    app: elasticsearch
  ports:
  - port: 9200
    name: http
  - port: 9300
    name: transport
  clusterIP: None
```

### Kibana Deployment

```yaml
# kibana.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: logging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:8.11.0
        ports:
        - containerPort: 5601
        env:
        - name: ELASTICSEARCH_HOSTS
          value: "http://elasticsearch:9200"
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
---
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: logging
spec:
  selector:
    app: kibana
  ports:
  - port: 5601
    targetPort: 5601
  type: ClusterIP
```

### Loki (альтернатива Elasticsearch)

Loki — легковесная система логирования от Grafana Labs:

```bash
# Установка Loki через Helm
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki-stack \
  --namespace logging \
  --create-namespace \
  --set grafana.enabled=true \
  --set prometheus.enabled=false
```

```yaml
# loki-values.yaml
loki:
  persistence:
    enabled: true
    size: 50Gi

promtail:
  config:
    clients:
    - url: http://loki:3100/loki/api/v1/push

grafana:
  enabled: true
  persistence:
    enabled: true
    size: 10Gi
```

---

## Health Probes и мониторинг

Health probes связаны с мониторингом — они позволяют Kubernetes автоматически отслеживать состояние приложений.

### Liveness Probe

Определяет, когда контейнер нужно перезапустить:

```yaml
# liveness-probe.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-liveness
spec:
  containers:
  - name: app
    image: my-app:latest
    ports:
    - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: X-Custom-Header
          value: Awesome
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      successThreshold: 1
      failureThreshold: 3
```

### Readiness Probe

Определяет, когда контейнер готов принимать трафик:

```yaml
# readiness-probe.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-readiness
spec:
  containers:
  - name: app
    image: my-app:latest
    ports:
    - containerPort: 8080
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 3
      successThreshold: 1
      failureThreshold: 3
```

### Startup Probe

Для медленно запускающихся приложений:

```yaml
# startup-probe.yaml
apiVersion: v1
kind: Pod
metadata:
  name: slow-starting-app
spec:
  containers:
  - name: app
    image: my-slow-app:latest
    ports:
    - containerPort: 8080
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
```

### Различные типы проверок

```yaml
# probe-types.yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-examples
spec:
  containers:
  - name: app
    image: my-app:latest

    # HTTP GET probe
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      periodSeconds: 10

    # TCP Socket probe
    readinessProbe:
      tcpSocket:
        port: 8080
      periodSeconds: 5

    # Exec probe
    startupProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      periodSeconds: 10

    # gRPC probe (Kubernetes 1.24+)
    # livenessProbe:
    #   grpc:
    #     port: 50051
    #   periodSeconds: 10
```

### Полный пример с всеми probes

```yaml
# complete-probes.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
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
        image: my-web-app:latest
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: metrics

        # Startup probe - ждём до 5 минут на запуск
        startupProbe:
          httpGet:
            path: /healthz
            port: http
          failureThreshold: 30
          periodSeconds: 10

        # Liveness probe - перезапускаем при зависании
        livenessProbe:
          httpGet:
            path: /healthz
            port: http
          initialDelaySeconds: 0  # startupProbe уже подождал
          periodSeconds: 15
          timeoutSeconds: 5
          failureThreshold: 3

        # Readiness probe - убираем из балансировки при проблемах
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 0
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3

        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

### Эндпоинты для проверок (Python/FastAPI)

```python
# health.py
from fastapi import FastAPI, Response, status
import asyncio

app = FastAPI()

# Глобальное состояние
app_ready = False
db_connected = False

@app.on_event("startup")
async def startup():
    global app_ready, db_connected
    # Имитация подключения к БД
    await asyncio.sleep(5)
    db_connected = True
    app_ready = True

@app.get("/healthz")
async def health_check():
    """Liveness probe - приложение живо?"""
    return {"status": "healthy"}

@app.get("/ready")
async def readiness_check(response: Response):
    """Readiness probe - приложение готово принимать трафик?"""
    if not app_ready or not db_connected:
        response.status_code = status.HTTP_503_SERVICE_UNAVAILABLE
        return {"status": "not ready", "db": db_connected}
    return {"status": "ready", "db": db_connected}

@app.get("/startup")
async def startup_check(response: Response):
    """Startup probe - приложение запустилось?"""
    if not app_ready:
        response.status_code = status.HTTP_503_SERVICE_UNAVAILABLE
        return {"status": "starting"}
    return {"status": "started"}
```

---

## Alerting и уведомления

### Alertmanager конфигурация

```yaml
# alertmanager-config.yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-config
  namespace: monitoring
stringData:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m
      smtp_smarthost: 'smtp.gmail.com:587'
      smtp_from: 'alertmanager@example.com'
      smtp_auth_username: 'alertmanager@example.com'
      smtp_auth_password: 'password'
      slack_api_url: 'https://hooks.slack.com/services/xxx/yyy/zzz'

    route:
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: 'default'
      routes:
      - match:
          severity: critical
        receiver: 'critical-alerts'
        continue: true
      - match:
          severity: warning
        receiver: 'warning-alerts'

    receivers:
    - name: 'default'
      slack_configs:
      - channel: '#alerts-default'
        send_resolved: true
        title: '{{ .Status | toUpper }}: {{ .CommonLabels.alertname }}'
        text: >-
          {{ range .Alerts }}
            *Alert:* {{ .Annotations.summary }}
            *Description:* {{ .Annotations.description }}
            *Severity:* {{ .Labels.severity }}
          {{ end }}

    - name: 'critical-alerts'
      slack_configs:
      - channel: '#alerts-critical'
        send_resolved: true
      pagerduty_configs:
      - service_key: 'your-pagerduty-key'
        severity: critical

    - name: 'warning-alerts'
      email_configs:
      - to: 'team@example.com'
        send_resolved: true

    inhibit_rules:
    - source_match:
        severity: 'critical'
      target_match:
        severity: 'warning'
      equal: ['alertname', 'namespace']
```

### PrometheusRule для алертов

```yaml
# prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: application-alerts
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
  - name: application.rules
    rules:
    # Высокий Error Rate
    - alert: HighErrorRate
      expr: |
        sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
        / sum(rate(http_requests_total[5m])) by (service) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate on {{ $labels.service }}"
        description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.service }}"

    # Высокая латентность
    - alert: HighLatency
      expr: |
        histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)) > 1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High latency on {{ $labels.service }}"
        description: "95th percentile latency is {{ $value }}s"

    # Под перезапускается
    - alert: PodCrashLooping
      expr: |
        increase(kube_pod_container_status_restarts_total[1h]) > 3
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
        description: "Pod {{ $labels.pod }} in {{ $labels.namespace }} has restarted {{ $value }} times"

    # Высокое использование CPU
    - alert: HighCPUUsage
      expr: |
        sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod, namespace)
        / sum(kube_pod_container_resource_limits{resource="cpu"}) by (pod, namespace) > 0.9
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage for {{ $labels.pod }}"
        description: "CPU usage is {{ $value | humanizePercentage }}"

    # Высокое использование памяти
    - alert: HighMemoryUsage
      expr: |
        sum(container_memory_usage_bytes{container!=""}) by (pod, namespace)
        / sum(kube_pod_container_resource_limits{resource="memory"}) by (pod, namespace) > 0.9
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High memory usage for {{ $labels.pod }}"
        description: "Memory usage is {{ $value | humanizePercentage }}"

    # Под не готов
    - alert: PodNotReady
      expr: |
        kube_pod_status_ready{condition="true"} == 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} is not ready"
        description: "Pod {{ $labels.pod }} in {{ $labels.namespace }} is not ready"

  - name: infrastructure.rules
    rules:
    # Нода недоступна
    - alert: NodeDown
      expr: up{job="node-exporter"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Node {{ $labels.instance }} is down"
        description: "Node has been unreachable for more than 5 minutes"

    # Мало места на диске
    - alert: DiskSpaceLow
      expr: |
        (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) < 0.1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Low disk space on {{ $labels.instance }}"
        description: "Disk space is below 10%"

    # Высокая нагрузка на ноду
    - alert: HighNodeLoad
      expr: node_load15 > 0.8 * count by (instance) (node_cpu_seconds_total{mode="idle"})
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "High load on {{ $labels.instance }}"
        description: "15-minute load average is high"
```

### Telegram Bot для алертов

```yaml
# alertmanager-telegram.yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-telegram
  namespace: monitoring
stringData:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m

    route:
      receiver: 'telegram'
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h

    receivers:
    - name: 'telegram'
      webhook_configs:
      - url: 'http://alertmanager-bot:8080'
        send_resolved: true
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager-bot
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alertmanager-bot
  template:
    metadata:
      labels:
        app: alertmanager-bot
    spec:
      containers:
      - name: bot
        image: metalmatze/alertmanager-bot:0.4.3
        args:
        - --alertmanager.url=http://alertmanager:9093
        - --log.level=info
        - --store=bolt
        - --bolt.path=/data/bot.db
        env:
        - name: TELEGRAM_ADMIN
          value: "YOUR_TELEGRAM_USER_ID"
        - name: TELEGRAM_TOKEN
          valueFrom:
            secretKeyRef:
              name: telegram-bot-secret
              key: token
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        emptyDir: {}
```

---

## Best Practices

### Мониторинг

1. **Используйте четыре золотых сигнала (Four Golden Signals)**:
   - **Latency** — время обработки запросов
   - **Traffic** — количество запросов
   - **Errors** — процент ошибок
   - **Saturation** — насыщение ресурсов

2. **RED метрики для сервисов**:
   - **Rate** — количество запросов в секунду
   - **Errors** — количество ошибок
   - **Duration** — время ответа

3. **USE метрики для ресурсов**:
   - **Utilization** — использование ресурса
   - **Saturation** — очередь/перегрузка
   - **Errors** — ошибки ресурса

```yaml
# golden-signals-dashboard.yaml (Grafana ConfigMap)
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: monitoring
data:
  golden-signals.json: |
    {
      "title": "Golden Signals",
      "panels": [
        {
          "title": "Request Rate",
          "targets": [{"expr": "sum(rate(http_requests_total[5m]))"}]
        },
        {
          "title": "Error Rate",
          "targets": [{"expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m]))"}]
        },
        {
          "title": "Latency P95",
          "targets": [{"expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))"}]
        },
        {
          "title": "Saturation",
          "targets": [{"expr": "sum(container_memory_usage_bytes) / sum(kube_node_status_allocatable{resource=\"memory\"})"}]
        }
      ]
    }
```

### Логирование

1. **Структурированные логи в JSON формате**
2. **Уровни логирования**: DEBUG, INFO, WARN, ERROR
3. **Контекст**: trace_id, request_id, user_id
4. **Не логируйте чувствительные данные** (пароли, токены)

```python
# structured-logging.py
import structlog
import uuid

structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer()
    ]
)

logger = structlog.get_logger()

def process_request(user_id: str):
    request_id = str(uuid.uuid4())
    log = logger.bind(request_id=request_id, user_id=user_id)

    log.info("Processing request started")
    try:
        # ... обработка
        log.info("Request processed successfully", duration_ms=45)
    except Exception as e:
        log.error("Request failed", error=str(e))
        raise
```

### Probes

1. **Используйте все три типа probes** для production
2. **Startup probe** для медленно стартующих приложений
3. **Разделяйте /healthz и /ready эндпоинты**
4. **Не делайте тяжёлые проверки** в liveness probe

```yaml
# probe-best-practices.yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 0      # startupProbe обрабатывает задержку
  periodSeconds: 15           # Не слишком часто
  timeoutSeconds: 5           # Достаточно времени для ответа
  failureThreshold: 3         # 3 неудачи = рестарт

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5            # Чаще, чем liveness
  timeoutSeconds: 3
  failureThreshold: 3

startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30        # 30 * 10 = 5 минут на запуск
  periodSeconds: 10
```

### Alerting

1. **Алерты должны быть actionable** — на каждый алерт есть конкретное действие
2. **Избегайте alert fatigue** — слишком много алертов = игнорирование
3. **Используйте severity уровни**: critical, warning, info
4. **Настройте inhibition rules** — подавление связанных алертов

```yaml
# alerting-best-practices.yaml
groups:
- name: slo.rules
  rules:
  # SLO: 99.9% availability
  - alert: SLOBreach
    expr: |
      (1 - sum(rate(http_requests_total{status=~"5.."}[30m]))
      / sum(rate(http_requests_total[30m]))) < 0.999
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "SLO breach: availability below 99.9%"
      runbook_url: "https://wiki/runbooks/slo-breach"

  # Predictive alert - диск заполнится через 4 часа
  - alert: DiskWillFillIn4Hours
    expr: |
      predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 4*3600) < 0
    for: 30m
    labels:
      severity: warning
    annotations:
      summary: "Disk {{ $labels.mountpoint }} will fill in 4 hours"
```

---

## Полезные команды kubectl

### Логи

```bash
# Базовые команды логов
kubectl logs pod-name                    # Логи пода
kubectl logs pod-name -c container-name  # Логи контейнера
kubectl logs -f pod-name                 # Следить за логами
kubectl logs pod-name --previous         # Логи предыдущего инстанса
kubectl logs pod-name --since=1h         # Логи за последний час
kubectl logs pod-name --tail=100         # Последние 100 строк
kubectl logs -l app=nginx                # Логи по лейблу
kubectl logs -l app=nginx --all-containers=true  # Все контейнеры

# Логи из нескольких подов одновременно
kubectl logs -l app=nginx --max-log-requests=10

# Логи с timestamps
kubectl logs pod-name --timestamps=true

# Логи в JSON (если приложение логирует в JSON)
kubectl logs pod-name | jq '.'
```

### Метрики и ресурсы

```bash
# Потребление ресурсов
kubectl top nodes                        # Метрики нод
kubectl top pods                         # Метрики подов
kubectl top pods --containers            # Метрики по контейнерам
kubectl top pods --sort-by=cpu           # Сортировка по CPU
kubectl top pods --sort-by=memory        # Сортировка по памяти

# API метрик
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods
```

### Состояние и диагностика

```bash
# Информация о подах
kubectl get pods -o wide                 # Расширенная информация
kubectl describe pod pod-name            # Детальная информация
kubectl get pod pod-name -o yaml         # YAML манифест

# События
kubectl get events                       # Все события
kubectl get events --sort-by='.lastTimestamp'  # По времени
kubectl get events --field-selector type=Warning  # Только warnings
kubectl get events -w                    # Watch события

# Состояние deployment
kubectl rollout status deployment/name
kubectl rollout history deployment/name

# Проверка endpoints
kubectl get endpoints
kubectl describe endpoints service-name
```

### Отладка

```bash
# Выполнение команд в контейнере
kubectl exec -it pod-name -- /bin/sh
kubectl exec -it pod-name -c container -- /bin/bash

# Копирование файлов
kubectl cp pod-name:/path/to/file ./local-file
kubectl cp ./local-file pod-name:/path/to/file

# Port forwarding
kubectl port-forward pod-name 8080:80
kubectl port-forward svc/service-name 8080:80

# Проверка DNS
kubectl run test --image=busybox --rm -it -- nslookup kubernetes

# Debug контейнер (Kubernetes 1.25+)
kubectl debug pod-name -it --image=busybox --target=container-name
```

### Мониторинг в реальном времени

```bash
# Watch подов
kubectl get pods -w

# Watch с подробностями
watch -n 2 kubectl get pods

# Stern для логов нескольких подов (требует установки)
stern app-name
stern -l app=nginx --since 1h

# K9s - TUI для Kubernetes (требует установки)
k9s

# Kubetail для логов (требует установки)
kubetail app-name
```

### Алерты и проблемы

```bash
# Поды с проблемами
kubectl get pods --field-selector=status.phase!=Running
kubectl get pods | grep -v Running

# Неготовые поды
kubectl get pods -o json | jq '.items[] | select(.status.conditions[] | select(.type=="Ready" and .status!="True")) | .metadata.name'

# Поды с высоким restart count
kubectl get pods -o json | jq '.items[] | select(.status.containerStatuses[].restartCount > 3) | .metadata.name'

# Проверка ресурсов
kubectl describe nodes | grep -A 5 "Allocated resources"

# PVC проблемы
kubectl get pvc | grep -v Bound
```

---

## Резюме

| Компонент | Инструмент | Назначение |
|-----------|------------|------------|
| Метрики | Prometheus | Сбор и хранение метрик |
| Визуализация | Grafana | Дашборды и графики |
| Логи | EFK/Loki | Централизованное логирование |
| Алерты | Alertmanager | Уведомления |
| Базовые метрики | Metrics Server | kubectl top |

### Минимальный стек для production

1. **Metrics Server** — для kubectl top и HPA
2. **Prometheus + Grafana** — для метрик и визуализации
3. **Alertmanager** — для алертов
4. **Loki или EFK** — для централизованных логов
5. **Правильно настроенные probes** — для health checking

### Чек-лист мониторинга

- [ ] Metrics Server установлен
- [ ] Prometheus собирает метрики
- [ ] Grafana настроена с дашбордами
- [ ] Alertmanager настроен с каналами уведомлений
- [ ] Централизованное логирование работает
- [ ] Все приложения имеют liveness/readiness probes
- [ ] SLO алерты настроены
- [ ] Runbooks созданы для критических алертов
