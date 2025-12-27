# Развёртывание kube-prometheus-stack через Helm

## Введение

kube-prometheus-stack (ранее известный как prometheus-operator) - это Helm-чарт, который устанавливает полный стек мониторинга для Kubernetes, включая Prometheus, Alertmanager, Grafana и множество предварительно настроенных дашбордов и правил алертинга.

## Состав kube-prometheus-stack

```
┌─────────────────────────────────────────────────────────────────┐
│                    kube-prometheus-stack                        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   Prometheus    │  │  Alertmanager   │  │     Grafana     │ │
│  │     Operator    │  │                 │  │                 │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   Prometheus    │  │ kube-state-     │  │  node-exporter  │ │
│  │     Server      │  │   metrics       │  │                 │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                   ServiceMonitors / PodMonitors              ││
│  │  (kubelet, apiserver, etcd, coredns, kube-controller-mgr)   ││
│  └─────────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    PrometheusRules                           ││
│  │  (alerting rules, recording rules)                          ││
│  └─────────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    Grafana Dashboards                        ││
│  │  (k8s resources, nodes, pods, networking, etcd, etc.)       ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

## Установка

### Добавление Helm репозитория

```bash
# Добавить репозиторий prometheus-community
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Посмотреть доступные версии
helm search repo prometheus-community/kube-prometheus-stack --versions
```

### Базовая установка

```bash
# Создать namespace
kubectl create namespace monitoring

# Установить с дефолтными настройками
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version 54.0.0
```

### Установка с кастомными values

```bash
# Сгенерировать файл values для редактирования
helm show values prometheus-community/kube-prometheus-stack > values.yaml

# Установить с кастомными настройками
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values.yaml \
  --version 54.0.0
```

## Базовый файл values.yaml

```yaml
# values.yaml - Основные настройки kube-prometheus-stack

# Глобальные настройки
global:
  # Частота скрейпинга по умолчанию
  scrape_interval: 30s
  # Таймаут скрейпинга
  scrape_timeout: 10s
  # Интервал оценки правил
  evaluation_interval: 30s

# ============================================================
# PROMETHEUS OPERATOR
# ============================================================
prometheusOperator:
  enabled: true

  # Количество реплик оператора
  replicas: 1

  # Ресурсы для оператора
  resources:
    requests:
      cpu: 100m
      memory: 100Mi
    limits:
      cpu: 200m
      memory: 200Mi

  # Настройки webhook для валидации CRDs
  admissionWebhooks:
    enabled: true
    patch:
      enabled: true

  # Service для доступа к оператору
  service:
    type: ClusterIP

# ============================================================
# PROMETHEUS
# ============================================================
prometheus:
  enabled: true

  # Настройки Prometheus CRD
  prometheusSpec:
    # Количество реплик
    replicas: 2

    # Время хранения метрик
    retention: 15d
    retentionSize: 50GB

    # Ресурсы
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 2000m
        memory: 8Gi

    # Persistent Volume для хранения данных
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: standard
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi

    # Селекторы для ServiceMonitor
    serviceMonitorSelectorNilUsesHelmValues: false
    serviceMonitorSelector: {}
    serviceMonitorNamespaceSelector: {}

    # Селекторы для PodMonitor
    podMonitorSelectorNilUsesHelmValues: false
    podMonitorSelector: {}
    podMonitorNamespaceSelector: {}

    # Селекторы для PrometheusRule
    ruleSelectorNilUsesHelmValues: false
    ruleSelector: {}
    ruleNamespaceSelector: {}

    # Дополнительные scrape configs
    additionalScrapeConfigs: []

    # Внешние labels (полезно для федерации)
    externalLabels:
      cluster: production

    # Pod anti-affinity для HA
    podAntiAffinity: hard
    podAntiAffinityTopologyKey: kubernetes.io/hostname

    # Security context
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      fsGroup: 2000

    # Tolerations для размещения на dedicated nodes
    tolerations: []

    # Node selector
    nodeSelector: {}

  # Ingress для Prometheus UI
  ingress:
    enabled: false
    # ingressClassName: nginx
    # hosts:
    #   - prometheus.example.com
    # tls:
    #   - secretName: prometheus-tls
    #     hosts:
    #       - prometheus.example.com

# ============================================================
# ALERTMANAGER
# ============================================================
alertmanager:
  enabled: true

  alertmanagerSpec:
    replicas: 3

    # Ресурсы
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi

    # Storage
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: standard
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi

    # Pod anti-affinity
    podAntiAffinity: soft

  # Конфигурация Alertmanager
  config:
    global:
      resolve_timeout: 5m
      smtp_smarthost: 'smtp.example.com:587'
      smtp_from: 'alertmanager@example.com'
      smtp_auth_username: 'alertmanager@example.com'
      smtp_auth_password: 'password'

    route:
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'default-receiver'
      routes:
        - match:
            severity: critical
          receiver: 'pagerduty'
        - match:
            severity: warning
          receiver: 'slack'

    receivers:
      - name: 'default-receiver'
        email_configs:
          - to: 'team@example.com'

      - name: 'slack'
        slack_configs:
          - api_url: 'https://hooks.slack.com/services/xxx'
            channel: '#alerts'
            send_resolved: true
            title: '{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}'
            text: |
              {{ range .Alerts }}
              *Alert:* {{ .Annotations.summary }}
              *Details:* {{ .Annotations.description }}
              {{ end }}

      - name: 'pagerduty'
        pagerduty_configs:
          - service_key: 'your-pagerduty-key'
            severity: '{{ .CommonLabels.severity }}'

    inhibit_rules:
      - source_match:
          severity: 'critical'
        target_match:
          severity: 'warning'
        equal: ['alertname', 'namespace']

# ============================================================
# GRAFANA
# ============================================================
grafana:
  enabled: true

  # Логин/пароль администратора
  adminUser: admin
  adminPassword: admin  # В production использовать secret

  # Ресурсы
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

  # Persistence
  persistence:
    enabled: true
    size: 10Gi
    storageClassName: standard

  # Дополнительные datasources
  additionalDataSources:
    - name: Loki
      type: loki
      url: http://loki:3100
      access: proxy
    - name: Tempo
      type: tempo
      url: http://tempo:3200
      access: proxy

  # Настройки Grafana
  grafana.ini:
    server:
      root_url: https://grafana.example.com
    auth.anonymous:
      enabled: false
    security:
      admin_password: "${GRAFANA_ADMIN_PASSWORD}"

  # Sidecar для автоматической загрузки дашбордов
  sidecar:
    dashboards:
      enabled: true
      searchNamespace: ALL
      label: grafana_dashboard
    datasources:
      enabled: true
      searchNamespace: ALL

  # Ingress
  ingress:
    enabled: false
    # hosts:
    #   - grafana.example.com

# ============================================================
# NODE EXPORTER
# ============================================================
nodeExporter:
  enabled: true

  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 250m
      memory: 128Mi

# ============================================================
# KUBE-STATE-METRICS
# ============================================================
kubeStateMetrics:
  enabled: true

kube-state-metrics:
  resources:
    requests:
      cpu: 10m
      memory: 32Mi
    limits:
      cpu: 100m
      memory: 128Mi

# ============================================================
# КОМПОНЕНТЫ КЛАСТЕРА
# ============================================================

# Мониторинг kubelet
kubelet:
  enabled: true
  serviceMonitor:
    https: true
    cAdvisor: true
    probes: true

# Мониторинг API сервера
kubeApiServer:
  enabled: true

# Мониторинг Controller Manager
kubeControllerManager:
  enabled: true
  # Endpoints могут отличаться в managed Kubernetes
  endpoints: []

# Мониторинг Scheduler
kubeScheduler:
  enabled: true
  endpoints: []

# Мониторинг CoreDNS
coreDns:
  enabled: true

# Мониторинг etcd
kubeEtcd:
  enabled: true
  # В managed K8s обычно недоступен
  endpoints: []

# Мониторинг Kube Proxy
kubeProxy:
  enabled: true

# ============================================================
# ДЕФОЛТНЫЕ ПРАВИЛА
# ============================================================
defaultRules:
  create: true
  rules:
    alertmanager: true
    etcd: true
    configReloaders: true
    general: true
    k8s: true
    kubeApiserverAvailability: true
    kubeApiserverBurnrate: true
    kubeApiserverHistogram: true
    kubeApiserverSlos: true
    kubeControllerManager: true
    kubelet: true
    kubeProxy: true
    kubePrometheusGeneral: true
    kubePrometheusNodeRecording: true
    kubernetesApps: true
    kubernetesResources: true
    kubernetesStorage: true
    kubernetesSystem: true
    kubeSchedulerAlerting: true
    kubeSchedulerRecording: true
    kubeStateMetrics: true
    network: true
    node: true
    nodeExporterAlerting: true
    nodeExporterRecording: true
    prometheus: true
    prometheusOperator: true
```

## Обновление и управление

### Обновление релиза

```bash
# Обновить репозиторий
helm repo update

# Посмотреть текущие values
helm get values prometheus -n monitoring

# Обновить с новыми values
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values.yaml \
  --version 55.0.0

# Обновить только отдельные параметры
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --reuse-values \
  --set prometheus.prometheusSpec.retention=30d
```

### Откат к предыдущей версии

```bash
# Посмотреть историю релизов
helm history prometheus -n monitoring

# Откатиться к предыдущей версии
helm rollback prometheus 1 -n monitoring
```

## Проверка установки

```bash
# Проверить поды
kubectl get pods -n monitoring

# Ожидаемый вывод:
# NAME                                                     READY   STATUS    RESTARTS   AGE
# alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   0          5m
# alertmanager-prometheus-kube-prometheus-alertmanager-1   2/2     Running   0          5m
# prometheus-grafana-xxx                                   3/3     Running   0          5m
# prometheus-kube-prometheus-operator-xxx                  1/1     Running   0          5m
# prometheus-kube-state-metrics-xxx                        1/1     Running   0          5m
# prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   0          5m
# prometheus-prometheus-kube-prometheus-prometheus-1       2/2     Running   0          5m
# prometheus-prometheus-node-exporter-xxx                  1/1     Running   0          5m

# Проверить CRDs
kubectl get crd | grep monitoring

# Проверить ServiceMonitors
kubectl get servicemonitors -n monitoring

# Проверить PrometheusRules
kubectl get prometheusrules -n monitoring
```

### Доступ к UI

```bash
# Prometheus UI
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
# Открыть http://localhost:9090

# Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
# Открыть http://localhost:3000 (admin/admin)

# Alertmanager
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-alertmanager 9093:9093
# Открыть http://localhost:9093
```

## Удаление

```bash
# Удалить Helm релиз
helm uninstall prometheus -n monitoring

# CRDs не удаляются автоматически! Удалить вручную:
kubectl delete crd alertmanagerconfigs.monitoring.coreos.com
kubectl delete crd alertmanagers.monitoring.coreos.com
kubectl delete crd podmonitors.monitoring.coreos.com
kubectl delete crd probes.monitoring.coreos.com
kubectl delete crd prometheuses.monitoring.coreos.com
kubectl delete crd prometheusrules.monitoring.coreos.com
kubectl delete crd servicemonitors.monitoring.coreos.com
kubectl delete crd thanosrulers.monitoring.coreos.com

# Удалить namespace
kubectl delete namespace monitoring
```

## Best Practices

### 1. Версионирование values

```bash
# Хранить values в Git
my-cluster/
├── monitoring/
│   ├── values.yaml
│   ├── values-staging.yaml
│   └── values-production.yaml
```

### 2. Использование секретов

```yaml
# Не хранить пароли в values.yaml
alertmanager:
  config:
    global:
      slack_api_url_file: /etc/alertmanager/secrets/slack-webhook/url

# Создать секрет
kubectl create secret generic alertmanager-slack \
  --from-literal=url='https://hooks.slack.com/services/xxx' \
  -n monitoring
```

### 3. Отдельные релизы для разных окружений

```bash
# Production
helm install prometheus-prod prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values-production.yaml

# Staging (в другом namespace или кластере)
helm install prometheus-staging prometheus-community/kube-prometheus-stack \
  --namespace monitoring-staging \
  --values values-staging.yaml
```

## Распространённые ошибки

### 1. Prometheus не видит ServiceMonitor

```bash
# Проверить labels селектора
kubectl get prometheus -n monitoring -o yaml | grep -A5 serviceMonitorSelector

# Убедиться что ServiceMonitor имеет правильные labels
kubectl get servicemonitor my-app -n app -o yaml | grep -A5 labels
```

### 2. Нехватка ресурсов

```bash
# Проверить events
kubectl get events -n monitoring --sort-by='.lastTimestamp'

# Увеличить ресурсы в values.yaml
prometheus:
  prometheusSpec:
    resources:
      requests:
        memory: 4Gi
      limits:
        memory: 8Gi
```

### 3. PVC не создаётся

```bash
# Проверить StorageClass
kubectl get storageclass

# Убедиться что storageClassName правильный
kubectl get pvc -n monitoring
```

## Заключение

kube-prometheus-stack предоставляет готовое production-ready решение для мониторинга Kubernetes. Helm-чарт позволяет гибко настраивать все компоненты и легко управлять обновлениями. Важно правильно настроить селекторы для ServiceMonitor/PodMonitor и выделить достаточно ресурсов для Prometheus.
