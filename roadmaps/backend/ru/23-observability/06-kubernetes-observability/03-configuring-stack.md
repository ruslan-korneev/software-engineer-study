# Настройка стека мониторинга

## Введение

После установки kube-prometheus-stack требуется тонкая настройка под конкретные требования: настройка retention, федерации, remote write, интеграция с внешними системами и оптимизация производительности.

## Настройка Prometheus

### Retention и хранение данных

```yaml
# values.yaml
prometheus:
  prometheusSpec:
    # Хранить данные 30 дней
    retention: 30d

    # Или ограничить по размеру
    retentionSize: 100GB

    # WAL compression для экономии места
    walCompression: true

    # Настройки хранилища
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: fast-ssd  # SSD для лучшей производительности
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 200Gi

    # Или использовать emptyDir для тестовых окружений
    # storageSpec:
    #   emptyDir:
    #     medium: Memory
    #     sizeLimit: 10Gi
```

### Настройка scrape интервалов

```yaml
prometheus:
  prometheusSpec:
    # Глобальные настройки
    scrapeInterval: 30s
    scrapeTimeout: 10s
    evaluationInterval: 30s

    # Для критичных метрик можно уменьшить интервал
    additionalScrapeConfigs:
      - job_name: 'critical-services'
        scrape_interval: 10s
        scrape_timeout: 5s
        kubernetes_sd_configs:
          - role: endpoints
            namespaces:
              names:
                - critical-ns
```

### External Labels для идентификации кластера

```yaml
prometheus:
  prometheusSpec:
    # Labels добавляются ко всем метрикам
    externalLabels:
      cluster: production-eu-west-1
      environment: production
      region: eu-west-1

    # Полезно для федерации и remote write
    replicaExternalLabelName: __replica__
    prometheusExternalLabelName: prometheus
```

## Remote Write и Remote Read

### Отправка метрик в Thanos

```yaml
prometheus:
  prometheusSpec:
    remoteWrite:
      - url: http://thanos-receive.thanos.svc:19291/api/v1/receive
        name: thanos

        # Queue configuration
        queueConfig:
          capacity: 10000
          maxShards: 200
          minShards: 1
          maxSamplesPerSend: 2000
          batchSendDeadline: 5s
          minBackoff: 30ms
          maxBackoff: 5s

        # Write relabel для фильтрации
        writeRelabelConfigs:
          # Не отправлять метрики с низким приоритетом
          - sourceLabels: [__name__]
            regex: 'go_.*'
            action: drop
          # Удалить лишние labels
          - regex: 'prometheus_replica'
            action: labeldrop
```

### Отправка в Victoria Metrics

```yaml
prometheus:
  prometheusSpec:
    remoteWrite:
      - url: http://victoria-metrics:8428/api/v1/write
        name: victoriametrics

        # Basic auth
        basicAuth:
          username:
            name: vm-credentials
            key: username
          password:
            name: vm-credentials
            key: password

        # TLS
        tlsConfig:
          insecureSkipVerify: false
          ca:
            secret:
              name: vm-tls
              key: ca.crt
```

### Remote Read для исторических данных

```yaml
prometheus:
  prometheusSpec:
    remoteRead:
      - url: http://thanos-query:9090/api/v1/read
        name: thanos
        readRecent: false  # Не читать свежие данные

      # Для долгосрочного хранилища
      - url: http://long-term-storage:9090/api/v1/read
        name: long-term
        readRecent: false
        requiredMatchers:
          age: "old"  # Читать только "старые" метрики
```

## Настройка Alertmanager

### Продвинутая конфигурация маршрутизации

```yaml
alertmanager:
  config:
    global:
      resolve_timeout: 5m
      # HTTP настройки для webhooks
      http_config:
        follow_redirects: true
        enable_http2: true

      # SMTP
      smtp_smarthost: 'smtp.gmail.com:587'
      smtp_from: 'alerts@company.com'
      smtp_auth_username: 'alerts@company.com'
      smtp_auth_password_file: /etc/alertmanager/secrets/smtp/password
      smtp_require_tls: true

    # Шаблоны
    templates:
      - '/etc/alertmanager/config/*.tmpl'

    # Основной маршрут
    route:
      receiver: 'default'
      group_by: ['alertname', 'namespace', 'severity']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h

      routes:
        # Критические алерты - PagerDuty + Slack
        - match:
            severity: critical
          receiver: 'critical-alerts'
          group_wait: 10s
          repeat_interval: 1h
          continue: true  # Продолжить проверку других routes

        # Warning - только Slack
        - match:
            severity: warning
          receiver: 'slack-warnings'
          group_wait: 1m

        # Инфраструктурные алерты
        - match_re:
            alertname: 'Kube.*|Node.*|Container.*'
          receiver: 'infrastructure-team'
          group_by: ['alertname', 'node']

        # Алерты приложений по namespace
        - match_re:
            namespace: 'app-.*'
          receiver: 'application-team'
          routes:
            - match:
                team: backend
              receiver: 'backend-team'
            - match:
                team: frontend
              receiver: 'frontend-team'

        # Тихие часы для non-critical
        - match:
            severity: info
          receiver: 'null'
          mute_time_intervals:
            - nights-and-weekends

    # Receivers
    receivers:
      - name: 'default'
        email_configs:
          - to: 'oncall@company.com'
            send_resolved: true

      - name: 'null'
        # Пустой receiver для подавления

      - name: 'critical-alerts'
        pagerduty_configs:
          - service_key_file: /etc/alertmanager/secrets/pagerduty/key
            severity: critical
            client: 'Prometheus {{ template "prometheus.external.url" . }}'
            client_url: '{{ template "prometheus.external.url" . }}'
            description: '{{ .CommonAnnotations.summary }}'
            details:
              firing: '{{ template "pagerduty.default.instances" .Alerts.Firing }}'
              num_firing: '{{ .Alerts.Firing | len }}'

        slack_configs:
          - api_url_file: /etc/alertmanager/secrets/slack/webhook
            channel: '#critical-alerts'
            send_resolved: true
            color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
            title: '[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
            text: |
              {{ range .Alerts }}
              *Cluster:* {{ .Labels.cluster }}
              *Namespace:* {{ .Labels.namespace }}
              *Summary:* {{ .Annotations.summary }}
              *Description:* {{ .Annotations.description }}
              {{ end }}
            actions:
              - type: button
                text: 'Runbook'
                url: '{{ (index .Alerts 0).Annotations.runbook_url }}'
              - type: button
                text: 'Dashboard'
                url: '{{ (index .Alerts 0).Annotations.dashboard_url }}'

      - name: 'slack-warnings'
        slack_configs:
          - api_url_file: /etc/alertmanager/secrets/slack/webhook
            channel: '#monitoring-warnings'
            send_resolved: true
            color: 'warning'

      - name: 'infrastructure-team'
        slack_configs:
          - channel: '#infra-alerts'
            api_url_file: /etc/alertmanager/secrets/slack/webhook

        opsgenie_configs:
          - api_key_file: /etc/alertmanager/secrets/opsgenie/key
            message: '{{ .CommonLabels.alertname }}'
            priority: '{{ if eq .CommonLabels.severity "critical" }}P1{{ else }}P3{{ end }}'

      - name: 'application-team'
        webhook_configs:
          - url: 'http://alert-router.internal/webhook'
            send_resolved: true

      - name: 'backend-team'
        slack_configs:
          - channel: '#backend-alerts'
            api_url_file: /etc/alertmanager/secrets/slack/webhook

      - name: 'frontend-team'
        slack_configs:
          - channel: '#frontend-alerts'
            api_url_file: /etc/alertmanager/secrets/slack/webhook

    # Inhibition rules
    inhibit_rules:
      # Critical подавляет warning для того же алерта
      - source_matchers:
          - severity = critical
        target_matchers:
          - severity = warning
        equal: ['alertname', 'namespace', 'pod']

      # Недоступность ноды подавляет алерты подов на ней
      - source_matchers:
          - alertname = NodeDown
        target_matchers:
          - alertname =~ "Pod.*|Container.*"
        equal: ['node']

      # Cluster-level issues подавляют namespace alerts
      - source_matchers:
          - alertname = KubeAPIDown
        target_matchers:
          - alertname =~ "Kube.*"

    # Time intervals
    time_intervals:
      - name: nights-and-weekends
        time_intervals:
          - times:
              - start_time: '22:00'
                end_time: '08:00'
          - weekdays: ['saturday', 'sunday']
```

### Секреты для Alertmanager

```yaml
# Создание секретов
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-secrets
  namespace: monitoring
type: Opaque
stringData:
  slack-webhook: "https://hooks.slack.com/services/xxx"
  pagerduty-key: "your-service-key"
  smtp-password: "your-smtp-password"
---
# Подключение в values.yaml
alertmanager:
  alertmanagerSpec:
    secrets:
      - alertmanager-secrets
```

## Настройка Grafana

### Дополнительные Data Sources

```yaml
grafana:
  additionalDataSources:
    # Prometheus (основной)
    - name: Prometheus
      type: prometheus
      url: http://prometheus-kube-prometheus-prometheus:9090
      access: proxy
      isDefault: true
      jsonData:
        timeInterval: "30s"
        httpMethod: POST

    # Thanos для исторических данных
    - name: Thanos
      type: prometheus
      url: http://thanos-query:9090
      access: proxy
      jsonData:
        timeInterval: "30s"

    # Loki для логов
    - name: Loki
      type: loki
      url: http://loki:3100
      access: proxy
      jsonData:
        derivedFields:
          - datasourceUid: tempo
            matcherRegex: "traceID=(\\w+)"
            name: TraceID
            url: "$${__value.raw}"

    # Tempo для трейсов
    - name: Tempo
      type: tempo
      url: http://tempo:3200
      access: proxy
      uid: tempo
      jsonData:
        httpMethod: GET
        tracesToLogs:
          datasourceUid: loki
          filterByTraceID: true
          filterBySpanID: false
        serviceMap:
          datasourceUid: prometheus
        nodeGraph:
          enabled: true

    # Jaeger
    - name: Jaeger
      type: jaeger
      url: http://jaeger-query:16686
      access: proxy

    # InfluxDB
    - name: InfluxDB
      type: influxdb
      url: http://influxdb:8086
      access: proxy
      database: metrics
      jsonData:
        httpMode: POST
```

### Кастомные дашборды

```yaml
grafana:
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: 'custom'
          orgId: 1
          folder: 'Custom'
          type: file
          disableDeletion: false
          editable: true
          options:
            path: /var/lib/grafana/dashboards/custom

        - name: 'infrastructure'
          orgId: 1
          folder: 'Infrastructure'
          type: file
          disableDeletion: true
          editable: false
          options:
            path: /var/lib/grafana/dashboards/infrastructure

  # Дашборды из ConfigMaps
  dashboardsConfigMaps:
    custom: "grafana-dashboards-custom"
    infrastructure: "grafana-dashboards-infra"

  # Дашборды из grafana.com
  dashboards:
    infrastructure:
      kubernetes-cluster:
        gnetId: 7249
        revision: 1
        datasource: Prometheus

      node-exporter:
        gnetId: 1860
        revision: 31
        datasource: Prometheus

      nginx-ingress:
        gnetId: 9614
        revision: 1
        datasource: Prometheus
```

### Настройки безопасности

```yaml
grafana:
  # Аутентификация через OIDC
  grafana.ini:
    server:
      root_url: https://grafana.company.com

    security:
      admin_user: admin
      admin_password: ${GRAFANA_ADMIN_PASSWORD}
      secret_key: ${GRAFANA_SECRET_KEY}
      disable_gravatar: true

    users:
      auto_assign_org: true
      auto_assign_org_role: Viewer

    auth:
      disable_login_form: false

    auth.generic_oauth:
      enabled: true
      name: SSO
      allow_sign_up: true
      client_id: ${OAUTH_CLIENT_ID}
      client_secret: ${OAUTH_CLIENT_SECRET}
      scopes: openid profile email groups
      auth_url: https://sso.company.com/oauth2/authorize
      token_url: https://sso.company.com/oauth2/token
      api_url: https://sso.company.com/oauth2/userinfo
      role_attribute_path: "contains(groups[*], 'admins') && 'Admin' || contains(groups[*], 'editors') && 'Editor' || 'Viewer'"

    auth.ldap:
      enabled: false
```

## Настройка kube-state-metrics

### Кастомные метрики

```yaml
kube-state-metrics:
  # Добавить custom resource state metrics
  customResourceState:
    enabled: true
    config:
      spec:
        resources:
          # Мониторинг Certificates от cert-manager
          - groupVersionKind:
              group: cert-manager.io
              version: v1
              kind: Certificate
            labelsFromPath:
              name: [metadata, name]
              namespace: [metadata, namespace]
            metrics:
              - name: "certmanager_certificate_ready_status"
                help: "The ready status of the certificate"
                each:
                  type: Gauge
                  gauge:
                    path: [status, conditions]
                    labelsFromPath:
                      type: [type]
                    valueFrom: [status]

          # Мониторинг ArgoCD Applications
          - groupVersionKind:
              group: argoproj.io
              version: v1alpha1
              kind: Application
            labelsFromPath:
              name: [metadata, name]
              namespace: [metadata, namespace]
              project: [spec, project]
            metrics:
              - name: "argocd_app_sync_status"
                help: "ArgoCD Application sync status"
                each:
                  type: StateSet
                  stateSet:
                    path: [status, sync, status]
                    list: ["Synced", "OutOfSync", "Unknown"]

  # Метрики для мониторинга
  metricLabelsAllowlist:
    - pods=[app,version,team]
    - deployments=[app,version]
    - namespaces=[team,environment]

  # Collectors
  collectors:
    - certificatesigningrequests
    - configmaps
    - cronjobs
    - daemonsets
    - deployments
    - endpoints
    - horizontalpodautoscalers
    - ingresses
    - jobs
    - leases
    - limitranges
    - mutatingwebhookconfigurations
    - namespaces
    - networkpolicies
    - nodes
    - persistentvolumeclaims
    - persistentvolumes
    - poddisruptionbudgets
    - pods
    - replicasets
    - replicationcontrollers
    - resourcequotas
    - secrets
    - services
    - statefulsets
    - storageclasses
    - validatingwebhookconfigurations
    - volumeattachments
```

## Настройка Node Exporter

```yaml
nodeExporter:
  # Сбор дополнительных метрик
  extraArgs:
    - --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/.+)($|/)
    - --collector.filesystem.fs-types-exclude=^(autofs|binfmt_misc|bpf|cgroup2?|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|iso9660|mqueue|nsfs|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|selinuxfs|squashfs|sysfs|tracefs)$
    - --collector.netclass.ignored-devices=^(veth.*|docker.*|br-.*)$
    - --collector.netdev.device-exclude=^(veth.*|docker.*|br-.*)$
    # Включить textfile collector для custom метрик
    - --collector.textfile.directory=/var/lib/node_exporter/textfile_collector

  # Монтирование директории для textfile collector
  extraHostVolumeMounts:
    - name: textfile-collector
      hostPath: /var/lib/node_exporter/textfile_collector
      mountPath: /var/lib/node_exporter/textfile_collector
      readOnly: true

  # Tolerations для мониторинга всех нод
  tolerations:
    - effect: NoSchedule
      operator: Exists
    - effect: NoExecute
      operator: Exists
```

## Оптимизация производительности

### Prometheus

```yaml
prometheus:
  prometheusSpec:
    # Query settings
    query:
      maxConcurrency: 20
      maxSamples: 50000000
      timeout: 2m

    # Limits
    enforcedSampleLimit: 100000
    enforcedTargetLimit: 1000
    enforcedLabelLimit: 64
    enforcedLabelNameLengthLimit: 1024
    enforcedLabelValueLengthLimit: 2048

    # Sharding для распределения нагрузки
    shards: 2

    # Дополнительные аргументы
    additionalArgs:
      - name: storage.tsdb.min-block-duration
        value: 2h
      - name: storage.tsdb.max-block-duration
        value: 2h
      - name: query.max-samples
        value: "50000000"
```

### Grafana

```yaml
grafana:
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 2Gi

  # Caching
  grafana.ini:
    database:
      type: postgres
      host: postgres:5432
      name: grafana
      user: grafana
      password: ${DB_PASSWORD}

    caching:
      enabled: true
      backend: redis

    remote_cache:
      type: redis
      connstr: addr=redis:6379
```

## Проверка конфигурации

```bash
# Проверить конфигурацию Prometheus
kubectl exec -n monitoring prometheus-prometheus-0 -c prometheus -- \
  promtool check config /etc/prometheus/config_out/prometheus.env.yaml

# Проверить правила
kubectl exec -n monitoring prometheus-prometheus-0 -c prometheus -- \
  promtool check rules /etc/prometheus/rules/*

# Проверить конфигурацию Alertmanager
kubectl exec -n monitoring alertmanager-prometheus-alertmanager-0 -c alertmanager -- \
  amtool check-config /etc/alertmanager/config_out/alertmanager.env.yaml

# Проверить маршрутизацию алертов
kubectl exec -n monitoring alertmanager-prometheus-alertmanager-0 -c alertmanager -- \
  amtool config routes show --config.file=/etc/alertmanager/config_out/alertmanager.env.yaml
```

## Заключение

Правильная настройка стека мониторинга включает множество аспектов: от конфигурации хранения и retention до сложной маршрутизации алертов. Важно настроить интеграцию с внешними системами через remote write/read, правильно сконфигурировать Alertmanager для разных команд и настроить Grafana с нужными data sources.
