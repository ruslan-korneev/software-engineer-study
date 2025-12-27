# Мониторинг приложений в Kubernetes

## Введение

Мониторинг приложений в Kubernetes осуществляется через ServiceMonitor и PodMonitor CRDs. Эти ресурсы позволяют декларативно описать, какие сервисы и поды должен скрейпить Prometheus, и с какими параметрами.

## ServiceMonitor

### Базовый ServiceMonitor

ServiceMonitor описывает мониторинг Kubernetes Service. Prometheus находит endpoints этого сервиса и скрейпит их.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: monitoring
  labels:
    release: prometheus  # Важно! Должен совпадать с selector в Prometheus CRD
spec:
  # Namespace где искать сервисы
  namespaceSelector:
    matchNames:
      - production
      - staging
  # Или все namespaces
  # namespaceSelector:
  #   any: true

  # Селектор сервисов по labels
  selector:
    matchLabels:
      app: my-app
    # Или более сложный селектор
    # matchExpressions:
    #   - key: app
    #     operator: In
    #     values: [my-app, my-app-v2]

  # Endpoints для скрейпинга
  endpoints:
    - port: metrics  # Имя порта из Service
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s

      # Схема (http или https)
      scheme: http

      # TLS настройки
      # tlsConfig:
      #   insecureSkipVerify: true
      #   caFile: /etc/prometheus/secrets/tls/ca.crt

      # Bearer token аутентификация
      # bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token

      # Basic auth
      # basicAuth:
      #   username:
      #     name: basic-auth-secret
      #     key: username
      #   password:
      #     name: basic-auth-secret
      #     key: password

      # Добавить labels
      relabelings:
        - sourceLabels: [__meta_kubernetes_namespace]
          targetLabel: namespace
        - sourceLabels: [__meta_kubernetes_service_name]
          targetLabel: service
        - sourceLabels: [__meta_kubernetes_pod_name]
          targetLabel: pod

      # Метки метрик
      metricRelabelings:
        # Удалить ненужные метрики
        - sourceLabels: [__name__]
          regex: 'go_gc_.*'
          action: drop
        # Переименовать метрику
        - sourceLabels: [__name__]
          regex: 'http_requests_total'
          targetLabel: __name__
          replacement: 'app_http_requests_total'
```

### Пример приложения с ServiceMonitor

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
        version: v1.0.0
    spec:
      containers:
        - name: app
          image: my-app:1.0.0
          ports:
            - name: http
              containerPort: 8080
            - name: metrics
              containerPort: 9090
          # Probes
          readinessProbe:
            httpGet:
              path: /health
              port: http
          livenessProbe:
            httpGet:
              path: /health
              port: http
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app  # Этот label используется в ServiceMonitor selector
spec:
  selector:
    app: my-app
  ports:
    - name: http
      port: 80
      targetPort: http
    - name: metrics  # Этот порт указывается в ServiceMonitor
      port: 9090
      targetPort: metrics
---
# ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: monitoring
  labels:
    release: prometheus
spec:
  namespaceSelector:
    matchNames:
      - production
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics
      interval: 30s
```

## PodMonitor

### Когда использовать PodMonitor

PodMonitor используется когда:
- Под не имеет Service (например, Jobs, CronJobs)
- Нужно мониторить sidecar контейнеры
- Нужен более гранулярный контроль над отдельными подами

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: batch-jobs
  namespace: monitoring
  labels:
    release: prometheus
spec:
  namespaceSelector:
    matchNames:
      - batch-processing

  # Селектор подов
  selector:
    matchLabels:
      job-type: batch
    matchExpressions:
      - key: batch-id
        operator: Exists

  # Pod endpoints
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
      interval: 60s

      # Можно указать имя контейнера (для sidecar)
      # targetPort: 9090

      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_label_batch_id]
          targetLabel: batch_id
        - sourceLabels: [__meta_kubernetes_pod_phase]
          regex: 'Pending|Unknown'
          action: drop
```

### Мониторинг CronJobs

```yaml
# CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: data-export
  namespace: batch
spec:
  schedule: "0 */6 * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: data-export
            job-type: batch
        spec:
          containers:
            - name: exporter
              image: data-exporter:1.0.0
              ports:
                - name: metrics
                  containerPort: 9090
          restartPolicy: OnFailure
---
# PodMonitor для CronJob
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: data-export-jobs
  namespace: monitoring
  labels:
    release: prometheus
spec:
  namespaceSelector:
    matchNames:
      - batch
  selector:
    matchLabels:
      app: data-export
  podMetricsEndpoints:
    - port: metrics
      interval: 30s
      # Скрейпить только Running поды
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_phase]
          regex: 'Running'
          action: keep
```

## Аннотации для автоматического обнаружения

Альтернативный подход - использовать аннотации подов:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-app
spec:
  template:
    metadata:
      annotations:
        # Prometheus будет скрейпить этот под
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
        prometheus.io/scheme: "http"
```

Для этого нужна соответствующая конфигурация в Prometheus:

```yaml
prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: 'kubernetes-pods-annotations'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scheme]
            action: replace
            regex: (https?)
            target_label: __scheme__
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
            action: replace
            regex: ([^:]+)(?::\d+)?;(\d+)
            replacement: $1:$2
            target_label: __address__
          - action: labelmap
            regex: __meta_kubernetes_pod_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: namespace
          - source_labels: [__meta_kubernetes_pod_name]
            action: replace
            target_label: pod
```

## Мониторинг различных типов приложений

### 1. HTTP API сервис

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api-server
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: api-server
  endpoints:
    - port: metrics
      interval: 15s
      metricRelabelings:
        # Создать histogram buckets для SLO
        - sourceLabels: [__name__, le]
          regex: 'http_request_duration_seconds_bucket;0.1'
          targetLabel: slo_bucket
          replacement: 'fast'
```

### 2. gRPC сервис

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: grpc-service
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: grpc-service
  endpoints:
    - port: metrics
      interval: 30s
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_label_grpc_service]
          targetLabel: grpc_service
      metricRelabelings:
        # Оставить только grpc метрики
        - sourceLabels: [__name__]
          regex: 'grpc_.*'
          action: keep
```

### 3. База данных (например, PostgreSQL с exporter)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: postgresql
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: postgresql
  endpoints:
    - port: metrics
      interval: 60s
      # Exporter может быть медленным
      scrapeTimeout: 30s
      metricRelabelings:
        # Добавить database label
        - sourceLabels: [datname]
          targetLabel: database
        # Удалить слишком детальные метрики
        - sourceLabels: [__name__]
          regex: 'pg_stat_statements_.*'
          action: drop
```

### 4. Message Queue (RabbitMQ, Kafka)

```yaml
# RabbitMQ
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: rabbitmq
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: rabbitmq
  endpoints:
    - port: prometheus
      interval: 30s
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_label_app_kubernetes_io_instance]
          targetLabel: rabbitmq_cluster
---
# Kafka (с kafka-exporter)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kafka
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: kafka-exporter
  endpoints:
    - port: metrics
      interval: 60s
```

### 5. Redis

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: redis
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: redis
  endpoints:
    - port: metrics
      interval: 30s
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_label_redis_role]
          targetLabel: role
```

## Мониторинг sidecar контейнеров

### Envoy Proxy (Istio)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: envoy-sidecar
  namespace: monitoring
spec:
  selector:
    matchExpressions:
      - key: security.istio.io/tlsMode
        operator: Exists
  podMetricsEndpoints:
    - port: http-envoy-prom
      path: /stats/prometheus
      interval: 30s
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_container_name]
          regex: 'istio-proxy'
          action: keep
```

### Linkerd Proxy

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: linkerd-proxy
  namespace: monitoring
spec:
  selector:
    matchLabels:
      linkerd.io/control-plane-component: proxy
  podMetricsEndpoints:
    - port: linkerd-admin
      path: /metrics
      interval: 30s
```

## Relabeling и MetricRelabeling

### Разница между relabelings и metricRelabelings

```yaml
endpoints:
  - port: metrics

    # relabelings применяются ДО скрейпинга
    # Влияют на target labels и решение о скрейпинге
    relabelings:
      # Пропустить поды в состоянии Terminating
      - sourceLabels: [__meta_kubernetes_pod_phase]
        regex: 'Terminating'
        action: drop

      # Добавить labels из pod annotations
      - sourceLabels: [__meta_kubernetes_pod_annotation_team]
        targetLabel: team

      # Изменить адрес скрейпинга
      - sourceLabels: [__address__]
        regex: '(.+):.*'
        replacement: '${1}:9090'
        targetLabel: __address__

    # metricRelabelings применяются ПОСЛЕ скрейпинга
    # Влияют на сами метрики
    metricRelabelings:
      # Удалить метрики
      - sourceLabels: [__name__]
        regex: 'go_.*'
        action: drop

      # Переименовать метрику
      - sourceLabels: [__name__]
        regex: 'http_requests_total'
        replacement: 'app_requests_total'
        targetLabel: __name__

      # Удалить label из метрик
      - regex: 'instance'
        action: labeldrop

      # Сохранить только определённые labels
      - regex: '__meta_.*'
        action: labelkeep
```

### Примеры relabeling

```yaml
relabelings:
  # 1. Добавить все pod labels
  - action: labelmap
    regex: __meta_kubernetes_pod_label_(.+)

  # 2. Заменить значение label
  - sourceLabels: [__meta_kubernetes_namespace]
    regex: 'production'
    replacement: 'prod'
    targetLabel: env

  # 3. Объединить labels
  - sourceLabels: [__meta_kubernetes_namespace, __meta_kubernetes_pod_name]
    separator: '/'
    targetLabel: instance

  # 4. Hashmod для шардирования
  - sourceLabels: [__address__]
    modulus: 3
    targetLabel: __tmp_hash
    action: hashmod
  - sourceLabels: [__tmp_hash]
    regex: '0'
    action: keep  # Этот Prometheus скрейпит только 1/3 targets

  # 5. Keep/drop по условию
  - sourceLabels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    regex: 'true'
    action: keep

  # 6. Replace с regex groups
  - sourceLabels: [__meta_kubernetes_pod_name]
    regex: '(.+)-[a-z0-9]+-[a-z0-9]+'
    replacement: '${1}'
    targetLabel: deployment
```

## Troubleshooting

### Проверка обнаружения ServiceMonitor

```bash
# Проверить что ServiceMonitor создан
kubectl get servicemonitors -A

# Проверить labels ServiceMonitor
kubectl get servicemonitor my-app -n monitoring -o yaml | grep -A5 labels

# Проверить selector Prometheus
kubectl get prometheus -n monitoring -o yaml | grep -A10 serviceMonitorSelector

# Проверить targets в Prometheus UI
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090
# Открыть http://localhost:9090/targets
```

### Проверка endpoints

```bash
# Проверить endpoints сервиса
kubectl get endpoints my-app -n production

# Проверить что порт metrics доступен
kubectl exec -n production deployment/my-app -- wget -qO- localhost:9090/metrics | head

# Проверить сетевую доступность из Prometheus
kubectl exec -n monitoring prometheus-prometheus-0 -c prometheus -- \
  wget -qO- my-app.production.svc:9090/metrics | head
```

### Распространённые проблемы

**1. ServiceMonitor не обнаруживается:**
```bash
# Проверить labels
kubectl get servicemonitor my-app -o jsonpath='{.metadata.labels}'
# Должен содержать release: prometheus (или другой label из selector)
```

**2. Нет targets в Prometheus:**
```bash
# Проверить namespaceSelector
kubectl get servicemonitor my-app -o yaml | grep -A5 namespaceSelector
# Убедиться что namespace указан правильно
```

**3. Targets в состоянии DOWN:**
```bash
# Проверить логи Prometheus
kubectl logs -n monitoring prometheus-prometheus-0 -c prometheus | grep -i error

# Проверить сетевую политику
kubectl get networkpolicies -n production
```

## Best Practices

### 1. Организация ServiceMonitors

```
monitoring/
├── servicemonitors/
│   ├── infrastructure/
│   │   ├── ingress-nginx.yaml
│   │   └── cert-manager.yaml
│   ├── databases/
│   │   ├── postgresql.yaml
│   │   └── redis.yaml
│   └── applications/
│       ├── api-gateway.yaml
│       └── user-service.yaml
```

### 2. Стандартные labels

```yaml
# Все ServiceMonitors должны иметь стандартные labels
metadata:
  labels:
    release: prometheus
    team: platform
    tier: infrastructure
```

### 3. Ограничение количества метрик

```yaml
metricRelabelings:
  # Лимит на количество labels
  - sourceLabels: [__name__]
    regex: '.*'
    action: keep
    # Добавить sample_limit
endpoints:
  - port: metrics
    sampleLimit: 10000
```

## Заключение

ServiceMonitor и PodMonitor - основные инструменты для мониторинга приложений в Kubernetes. Правильное использование selectors, relabeling и metricRelabeling позволяет гибко настроить сбор метрик. Важно следовать best practices: стандартизировать labels, ограничивать количество метрик и организовывать конфигурацию в понятную структуру.
