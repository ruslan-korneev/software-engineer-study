# Prometheus Rules в Kubernetes

## Введение

PrometheusRule - это Custom Resource Definition (CRD), который позволяет декларативно описывать правила алертинга и записи (recording rules) для Prometheus. Prometheus Operator автоматически применяет эти правила к соответствующим экземплярам Prometheus.

## Структура PrometheusRule

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-application-rules
  namespace: monitoring
  labels:
    # Важно! Должен совпадать с ruleSelector в Prometheus CRD
    release: prometheus
    # Дополнительные метки для организации
    team: backend
    tier: application
spec:
  groups:
    - name: my-application.rules
      # Интервал оценки правил (опционально, использует глобальный по умолчанию)
      interval: 30s
      rules:
        # Recording rules
        - record: job:http_requests:rate5m
          expr: sum(rate(http_requests_total[5m])) by (job)
          labels:
            aggregation: "5m"

        # Alerting rules
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
            /
            sum(rate(http_requests_total[5m])) by (job)
            > 0.05
          for: 5m
          labels:
            severity: critical
            team: backend
          annotations:
            summary: "High error rate detected"
            description: "Error rate is {{ $value | humanizePercentage }} for job {{ $labels.job }}"
            runbook_url: "https://runbooks.example.com/high-error-rate"
            dashboard_url: "https://grafana.example.com/d/app-overview"
```

## Recording Rules

Recording rules позволяют предварительно вычислять часто используемые или ресурсоёмкие запросы и сохранять результаты как новые time series.

### Преимущества Recording Rules

1. **Производительность:** Сложные запросы вычисляются один раз
2. **Упрощение:** Сложные выражения можно использовать через простое имя метрики
3. **Консистентность:** Одинаковые вычисления во всех дашбордах и алертах

### Соглашения по именованию

```
level:metric:operations
```

- **level:** Уровень агрегации (job, instance, node, namespace)
- **metric:** Исходная метрика
- **operations:** Выполненные операции (rate, sum, avg)

### Примеры Recording Rules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: recording-rules
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    # Правила для HTTP метрик
    - name: http.rules
      interval: 30s
      rules:
        # Request rate по job
        - record: job:http_requests:rate5m
          expr: sum(rate(http_requests_total[5m])) by (job)

        # Error rate по job
        - record: job:http_errors:rate5m
          expr: sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)

        # Error ratio
        - record: job:http_error_ratio:rate5m
          expr: |
            job:http_errors:rate5m
            /
            job:http_requests:rate5m

        # Latency percentiles
        - record: job:http_request_duration_seconds:p50
          expr: histogram_quantile(0.5, sum(rate(http_request_duration_seconds_bucket[5m])) by (job, le))

        - record: job:http_request_duration_seconds:p95
          expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (job, le))

        - record: job:http_request_duration_seconds:p99
          expr: histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (job, le))

    # Правила для Kubernetes ресурсов
    - name: kubernetes.rules
      rules:
        # CPU usage по namespace
        - record: namespace:container_cpu_usage_seconds_total:sum_rate
          expr: |
            sum(rate(container_cpu_usage_seconds_total{container!="", pod!=""}[5m])) by (namespace)

        # Memory usage по namespace
        - record: namespace:container_memory_working_set_bytes:sum
          expr: |
            sum(container_memory_working_set_bytes{container!="", pod!=""}) by (namespace)

        # CPU requests по namespace
        - record: namespace:kube_pod_container_resource_requests_cpu_cores:sum
          expr: |
            sum(kube_pod_container_resource_requests{resource="cpu"}) by (namespace)

        # CPU utilization vs requests
        - record: namespace:container_cpu_utilization:ratio
          expr: |
            namespace:container_cpu_usage_seconds_total:sum_rate
            /
            namespace:kube_pod_container_resource_requests_cpu_cores:sum

    # SLO-based rules
    - name: slo.rules
      rules:
        # Error budget consumption (30 day window)
        - record: job:slo_error_budget_remaining:ratio
          expr: |
            1 - (
              sum(increase(http_requests_total{status=~"5.."}[30d])) by (job)
              /
              sum(increase(http_requests_total[30d])) by (job)
            ) / 0.001  # 99.9% SLO target

        # Burn rate
        - record: job:slo_burn_rate:ratio1h
          expr: |
            (
              sum(rate(http_requests_total{status=~"5.."}[1h])) by (job)
              /
              sum(rate(http_requests_total[1h])) by (job)
            ) / 0.001
```

## Alerting Rules

### Структура алерта

```yaml
- alert: AlertName
  expr: <PromQL expression>  # Выражение, которое должно вернуть true для срабатывания
  for: 5m                     # Сколько времени условие должно выполняться
  labels:                     # Labels добавляемые к алерту
    severity: critical
    team: backend
  annotations:                # Описательная информация
    summary: "Short description"
    description: "Detailed description with {{ $value }} and {{ $labels.instance }}"
    runbook_url: "https://..."
```

### Severity levels

```yaml
# Стандартные уровни severity:
# critical - требует немедленного внимания (PagerDuty, звонок)
# warning  - требует внимания в рабочее время (Slack, email)
# info     - информационный (для дашбордов)
```

### Примеры Alerting Rules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: application-alerts
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    # Алерты для приложений
    - name: application.alerts
      rules:
        # Высокий error rate
        - alert: HighErrorRate
          expr: |
            (
              sum(rate(http_requests_total{status=~"5.."}[5m])) by (job, namespace)
              /
              sum(rate(http_requests_total[5m])) by (job, namespace)
            ) > 0.05
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "High HTTP error rate"
            description: |
              Error rate is {{ $value | humanizePercentage }} for {{ $labels.job }}
              in namespace {{ $labels.namespace }}
            runbook_url: "https://runbooks.example.com/high-error-rate"

        # Высокая latency
        - alert: HighLatency
          expr: |
            histogram_quantile(0.95,
              sum(rate(http_request_duration_seconds_bucket[5m])) by (job, le)
            ) > 1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High request latency"
            description: "P95 latency is {{ $value | humanizeDuration }} for {{ $labels.job }}"

        # Низкий request rate (возможная проблема)
        - alert: LowRequestRate
          expr: |
            sum(rate(http_requests_total[5m])) by (job) < 1
            unless
            hour() >= 22 or hour() < 6  # Игнорировать ночью
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "Low request rate"
            description: "Request rate for {{ $labels.job }} is only {{ $value | humanize }} req/s"

        # Высокое потребление памяти
        - alert: HighMemoryUsage
          expr: |
            (
              container_memory_working_set_bytes{container!=""}
              /
              container_spec_memory_limit_bytes{container!=""}
            ) > 0.9
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Container memory usage is high"
            description: |
              Memory usage for {{ $labels.container }} in pod {{ $labels.pod }}
              is {{ $value | humanizePercentage }}

        # CPU throttling
        - alert: CPUThrottling
          expr: |
            sum(increase(container_cpu_cfs_throttled_periods_total{container!=""}[5m])) by (pod, container, namespace)
            /
            sum(increase(container_cpu_cfs_periods_total{container!=""}[5m])) by (pod, container, namespace)
            > 0.25
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Container CPU throttling"
            description: |
              {{ $labels.container }} in {{ $labels.pod }} is being throttled
              {{ $value | humanizePercentage }} of the time

    # Алерты для подов
    - name: pod.alerts
      rules:
        # Под не Ready
        - alert: PodNotReady
          expr: |
            kube_pod_status_ready{condition="true"} == 0
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Pod is not ready"
            description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is not ready"

        # Много рестартов
        - alert: PodCrashLooping
          expr: |
            increase(kube_pod_container_status_restarts_total[1h]) > 5
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Pod is crash looping"
            description: |
              Container {{ $labels.container }} in pod {{ $labels.namespace }}/{{ $labels.pod }}
              has restarted {{ $value | humanize }} times in the last hour

        # Pod в Pending слишком долго
        - alert: PodPending
          expr: |
            kube_pod_status_phase{phase="Pending"} == 1
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "Pod stuck in Pending"
            description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is pending for more than 15 minutes"

        # Terminating pod
        - alert: PodTerminating
          expr: |
            kube_pod_deletion_timestamp > 0
            and
            (time() - kube_pod_deletion_timestamp) > 300
          for: 1m
          labels:
            severity: warning
          annotations:
            summary: "Pod stuck terminating"
            description: |
              Pod {{ $labels.namespace }}/{{ $labels.pod }}
              has been terminating for more than 5 minutes

    # Алерты для deployment
    - name: deployment.alerts
      rules:
        # Deployment replicas mismatch
        - alert: DeploymentReplicasMismatch
          expr: |
            kube_deployment_spec_replicas
            !=
            kube_deployment_status_replicas_available
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "Deployment replicas mismatch"
            description: |
              Deployment {{ $labels.namespace }}/{{ $labels.deployment }}
              has {{ $value }} replicas available, expected
              {{ with printf "kube_deployment_spec_replicas{deployment='%s'}" $labels.deployment | query }}
                {{ . | first | value }}
              {{ end }}

        # Deployment generation mismatch (rollout stuck)
        - alert: DeploymentGenerationMismatch
          expr: |
            kube_deployment_status_observed_generation
            !=
            kube_deployment_metadata_generation
          for: 15m
          labels:
            severity: critical
          annotations:
            summary: "Deployment rollout stuck"
            description: |
              Deployment {{ $labels.namespace }}/{{ $labels.deployment }} rollout is stuck

    # SLO алерты (Multi-window, Multi-burn-rate)
    - name: slo.alerts
      rules:
        # Fast burn (1h window, 14.4x burn rate)
        - alert: SLOErrorBudgetFastBurn
          expr: |
            (
              sum(rate(http_requests_total{status=~"5.."}[1h])) by (job)
              /
              sum(rate(http_requests_total[1h])) by (job)
            ) > (14.4 * 0.001)
            and
            (
              sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
              /
              sum(rate(http_requests_total[5m])) by (job)
            ) > (14.4 * 0.001)
          for: 2m
          labels:
            severity: critical
            slo: "true"
          annotations:
            summary: "SLO error budget burning very fast"
            description: |
              Error budget for {{ $labels.job }} is burning at 14.4x rate.
              At this rate, the entire 30-day budget will be consumed in 2 days.

        # Slow burn (6h window, 6x burn rate)
        - alert: SLOErrorBudgetSlowBurn
          expr: |
            (
              sum(rate(http_requests_total{status=~"5.."}[6h])) by (job)
              /
              sum(rate(http_requests_total[6h])) by (job)
            ) > (6 * 0.001)
            and
            (
              sum(rate(http_requests_total{status=~"5.."}[30m])) by (job)
              /
              sum(rate(http_requests_total[30m])) by (job)
            ) > (6 * 0.001)
          for: 15m
          labels:
            severity: warning
            slo: "true"
          annotations:
            summary: "SLO error budget burning slowly"
            description: |
              Error budget for {{ $labels.job }} is burning at 6x rate.
              At this rate, the entire 30-day budget will be consumed in 5 days.
```

## Kubernetes Infrastructure Alerts

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-infrastructure-alerts
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    # Node алерты
    - name: node.alerts
      rules:
        - alert: NodeNotReady
          expr: kube_node_status_condition{condition="Ready",status="true"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Node not ready"
            description: "Node {{ $labels.node }} is not ready"

        - alert: NodeHighCPU
          expr: |
            (1 - avg by(node) (rate(node_cpu_seconds_total{mode="idle"}[5m]))) > 0.9
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "Node high CPU usage"
            description: "Node {{ $labels.node }} CPU usage is {{ $value | humanizePercentage }}"

        - alert: NodeHighMemory
          expr: |
            (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) > 0.9
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "Node high memory usage"
            description: "Node {{ $labels.node }} memory usage is {{ $value | humanizePercentage }}"

        - alert: NodeDiskPressure
          expr: kube_node_status_condition{condition="DiskPressure",status="true"} == 1
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Node disk pressure"
            description: "Node {{ $labels.node }} is experiencing disk pressure"

        - alert: NodeFilesystemFull
          expr: |
            (1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) > 0.85
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "Node filesystem almost full"
            description: |
              Filesystem {{ $labels.device }} on node {{ $labels.node }}
              is {{ $value | humanizePercentage }} full

    # PVC алерты
    - name: pvc.alerts
      rules:
        - alert: PersistentVolumeClaimPending
          expr: kube_persistentvolumeclaim_status_phase{phase="Pending"} == 1
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "PVC pending"
            description: |
              PVC {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }}
              is pending for more than 15 minutes

        - alert: PersistentVolumeFull
          expr: |
            kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.9
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Persistent volume almost full"
            description: |
              PVC {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }}
              is {{ $value | humanizePercentage }} full

    # Control plane алерты
    - name: control-plane.alerts
      rules:
        - alert: KubeAPIDown
          expr: up{job="kubernetes-apiservers"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Kubernetes API server is down"
            description: "Kubernetes API server {{ $labels.instance }} is down"

        - alert: KubeControllerManagerDown
          expr: up{job="kube-controller-manager"} == 0
          for: 15m
          labels:
            severity: critical
          annotations:
            summary: "Kubernetes controller manager is down"

        - alert: KubeSchedulerDown
          expr: up{job="kube-scheduler"} == 0
          for: 15m
          labels:
            severity: critical
          annotations:
            summary: "Kubernetes scheduler is down"

        - alert: EtcdMembersDown
          expr: |
            count(up{job="etcd"} == 0) > (count(up{job="etcd"}) / 2 - 1)
          for: 3m
          labels:
            severity: critical
          annotations:
            summary: "Etcd cluster has too many members down"
            description: "etcd cluster has lost quorum, {{ $value }} members are down"
```

## Проверка правил

### Валидация синтаксиса

```bash
# С помощью promtool
kubectl exec -n monitoring prometheus-prometheus-0 -c prometheus -- \
  promtool check rules /etc/prometheus/rules/*.yaml

# Локально
promtool check rules my-rules.yaml
```

### Unit тесты для правил

```yaml
# test-rules.yaml
rule_files:
  - my-rules.yaml

evaluation_interval: 1m

tests:
  - interval: 1m
    input_series:
      - series: 'http_requests_total{job="api", status="500"}'
        values: '0+10x10'  # От 0, увеличиваясь на 10 каждую минуту
      - series: 'http_requests_total{job="api", status="200"}'
        values: '0+100x10'

    alert_rule_test:
      - eval_time: 10m
        alertname: HighErrorRate
        exp_alerts:
          - exp_labels:
              job: api
              severity: critical
            exp_annotations:
              summary: "High error rate detected"

    promql_expr_test:
      - expr: job:http_error_ratio:rate5m
        eval_time: 5m
        exp_samples:
          - labels: 'job:http_error_ratio:rate5m{job="api"}'
            value: 0.1
```

```bash
# Запуск тестов
promtool test rules test-rules.yaml
```

## Best Practices

### 1. Организация правил

```
rules/
├── recording/
│   ├── http.yaml
│   ├── kubernetes.yaml
│   └── slo.yaml
├── alerting/
│   ├── application.yaml
│   ├── infrastructure.yaml
│   └── slo.yaml
└── tests/
    └── test-all.yaml
```

### 2. Стандартизация labels

```yaml
# Всегда добавлять severity
labels:
  severity: critical|warning|info

# Добавлять team для маршрутизации
labels:
  team: backend|frontend|platform|sre

# Добавлять tier для приоритезации
labels:
  tier: critical|standard|auxiliary
```

### 3. Полезные annotations

```yaml
annotations:
  summary: "Краткое описание (1 строка)"
  description: "Детальное описание с {{ $value }} и {{ $labels.pod }}"
  runbook_url: "https://runbooks.example.com/alert-name"
  dashboard_url: "https://grafana.example.com/d/xxx?var-pod={{ $labels.pod }}"
  playbook: |
    1. Проверить логи: kubectl logs {{ $labels.pod }}
    2. Проверить ресурсы: kubectl top pod {{ $labels.pod }}
    3. При необходимости перезапустить: kubectl delete pod {{ $labels.pod }}
```

### 4. Избегание alert fatigue

```yaml
# Использовать for для избежания ложных срабатываний
for: 5m

# Группировать похожие алерты
# В Alertmanager config
group_by: ['alertname', 'namespace']

# Использовать inhibition rules
inhibit_rules:
  - source_matchers: [severity="critical"]
    target_matchers: [severity="warning"]
    equal: ['alertname']
```

## Распространённые ошибки

### 1. Отсутствие for clause

```yaml
# Плохо - срабатывает мгновенно
- alert: HighCPU
  expr: cpu_usage > 0.9

# Хорошо - ждёт 5 минут
- alert: HighCPU
  expr: cpu_usage > 0.9
  for: 5m
```

### 2. Неправильные labels

```yaml
# Плохо - label без значения
labels:
  release: prometheus
  team:  # Пустое значение

# Хорошо
labels:
  release: prometheus
  team: backend
```

### 3. Сложные выражения без recording rules

```yaml
# Плохо - вычисляется при каждом evaluation
- alert: ComplexAlert
  expr: |
    sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
    /
    sum(rate(http_requests_total[5m])) by (job)
    > 0.05

# Хорошо - использует recording rule
- record: job:http_error_ratio:rate5m
  expr: |
    sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
    /
    sum(rate(http_requests_total[5m])) by (job)

- alert: HighErrorRate
  expr: job:http_error_ratio:rate5m > 0.05
```

## Заключение

PrometheusRule позволяет декларативно управлять правилами алертинга и recording rules в Kubernetes. Важно следовать best practices: использовать recording rules для сложных вычислений, стандартизировать labels и annotations, писать unit тесты для правил и организовывать их в понятную структуру. Правильно настроенные алерты должны быть actionable и не создавать alert fatigue.
