# Установка и настройка Prometheus

## Введение

В этом разделе мы рассмотрим различные способы установки Prometheus и детально разберём конфигурационный файл. Prometheus можно развернуть как на bare-metal серверах, так и в контейнерах Kubernetes.

## Способы установки

### 1. Установка из бинарного файла

```bash
# Скачивание последней версии
PROMETHEUS_VERSION="2.48.0"
wget https://github.com/prometheus/prometheus/releases/download/v${PROMETHEUS_VERSION}/prometheus-${PROMETHEUS_VERSION}.linux-amd64.tar.gz

# Распаковка
tar xvfz prometheus-${PROMETHEUS_VERSION}.linux-amd64.tar.gz
cd prometheus-${PROMETHEUS_VERSION}.linux-amd64

# Проверка версии
./prometheus --version

# Запуск с конфигурацией по умолчанию
./prometheus --config.file=prometheus.yml
```

### 2. Установка через Docker

```bash
# Простой запуск
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  prom/prometheus

# Запуск с кастомной конфигурацией
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v prometheus_data:/prometheus \
  prom/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/prometheus \
  --storage.tsdb.retention.time=15d
```

### 3. Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.48.0
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./rules:/etc/prometheus/rules:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
    restart: unless-stopped
    networks:
      - monitoring

  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    restart: unless-stopped
    networks:
      - monitoring

  node_exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node_exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:10.2.0
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    restart: unless-stopped
    networks:
      - monitoring

volumes:
  prometheus_data:
  alertmanager_data:
  grafana_data:

networks:
  monitoring:
    driver: bridge
```

### 4. Установка в Kubernetes с Helm

```bash
# Добавление репозитория
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Установка полного стека (Prometheus, Alertmanager, Grafana, Node Exporter)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi

# Только Prometheus
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --create-namespace
```

### 5. Systemd Service

```bash
# Создание пользователя
sudo useradd --no-create-home --shell /bin/false prometheus

# Создание директорий
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus

# Копирование файлов
sudo cp prometheus promtool /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
sudo cp -r consoles console_libraries /etc/prometheus/
sudo cp prometheus.yml /etc/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus
```

```ini
# /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus Monitoring System
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --storage.tsdb.retention.time=15d \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# Активация сервиса
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

## Структура конфигурационного файла

### Основной конфигурационный файл prometheus.yml

```yaml
# Глобальные настройки
global:
  scrape_interval: 15s          # Интервал сбора метрик
  scrape_timeout: 10s           # Таймаут скрейпинга
  evaluation_interval: 15s       # Интервал вычисления правил
  external_labels:               # Лейблы для всех метрик
    cluster: 'production'
    region: 'eu-west-1'

# Подключение файлов с правилами
rule_files:
  - '/etc/prometheus/rules/*.yml'
  - '/etc/prometheus/rules/alerts/*.yml'

# Настройки Alertmanager
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'alertmanager:9093'
      scheme: http
      timeout: 10s
      api_version: v2

# Настройки скрейпинга
scrape_configs:
  # Мониторинг самого Prometheus
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Мониторинг серверов через Node Exporter
  - job_name: 'node'
    static_configs:
      - targets:
          - 'node-exporter-1:9100'
          - 'node-exporter-2:9100'
          - 'node-exporter-3:9100'
    relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):\d+'
        target_label: instance
        replacement: '${1}'

# Remote storage (опционально)
remote_write:
  - url: "http://remote-storage:9201/write"
    queue_config:
      max_samples_per_send: 1000
      batch_send_deadline: 5s
    write_relabel_configs:
      - source_labels: [__name__]
        regex: 'expensive_metric_.*'
        action: drop

remote_read:
  - url: "http://remote-storage:9201/read"
    read_recent: true
```

## Детальная настройка scrape_configs

### Статическая конфигурация

```yaml
scrape_configs:
  - job_name: 'web-servers'
    scrape_interval: 10s
    scrape_timeout: 5s
    metrics_path: '/metrics'
    scheme: 'http'
    static_configs:
      - targets:
          - 'web-1.example.com:8080'
          - 'web-2.example.com:8080'
        labels:
          env: 'production'
          team: 'platform'
      - targets:
          - 'web-staging.example.com:8080'
        labels:
          env: 'staging'
          team: 'platform'
```

### Basic Auth и TLS

```yaml
scrape_configs:
  - job_name: 'secure-target'
    scheme: https
    basic_auth:
      username: 'prometheus'
      password: 'secret_password'
    # Или через файл
    # basic_auth:
    #   username: 'prometheus'
    #   password_file: '/etc/prometheus/secrets/password'
    tls_config:
      ca_file: '/etc/prometheus/certs/ca.crt'
      cert_file: '/etc/prometheus/certs/client.crt'
      key_file: '/etc/prometheus/certs/client.key'
      insecure_skip_verify: false
    static_configs:
      - targets: ['secure-service:443']
```

### Bearer Token Authentication

```yaml
scrape_configs:
  - job_name: 'kubernetes-api'
    scheme: https
    bearer_token_file: '/var/run/secrets/kubernetes.io/serviceaccount/token'
    tls_config:
      ca_file: '/var/run/secrets/kubernetes.io/serviceaccount/ca.crt'
    static_configs:
      - targets: ['kubernetes.default.svc:443']
```

## Relabel Configs

Relabeling — мощный механизм для модификации лейблов перед сохранением метрик.

### Основные действия

```yaml
scrape_configs:
  - job_name: 'example'
    static_configs:
      - targets: ['app:8080']
    relabel_configs:
      # Сохранить только определённые таргеты
      - source_labels: [__address__]
        regex: '.*:8080'
        action: keep

      # Удалить таргеты
      - source_labels: [__meta_kubernetes_pod_phase]
        regex: 'Pending|Succeeded|Failed'
        action: drop

      # Заменить значение лейбла
      - source_labels: [__address__]
        regex: '(.*):\d+'
        target_label: instance
        replacement: '${1}'

      # Переименовать лейбл
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod

      # Добавить хэш
      - source_labels: [__address__]
        modulus: 4
        target_label: __tmp_hash
        action: hashmod

      # Сохранить лейблы по маске
      - regex: '__meta_kubernetes_pod_label_(.+)'
        action: labelmap
        replacement: 'pod_${1}'

      # Удалить лейблы
      - regex: 'temporary_.*'
        action: labeldrop

      # Оставить только указанные лейблы
      - regex: 'keep_this|and_this'
        action: labelkeep
```

### Пример Kubernetes relabeling

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Сохранить только поды с аннотацией prometheus.io/scrape: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

      # Использовать аннотацию для пути метрик
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

      # Использовать аннотацию для порта
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__

      # Добавить namespace как лейбл
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace

      # Добавить имя пода
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod

      # Скопировать все pod labels
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
```

## Recording Rules

Recording rules позволяют предварительно вычислять часто используемые или ресурсоёмкие выражения.

```yaml
# /etc/prometheus/rules/recording_rules.yml
groups:
  - name: node_recording_rules
    interval: 15s
    rules:
      # CPU usage percentage
      - record: instance:node_cpu_utilization:ratio
        expr: |
          1 - avg by (instance) (
            rate(node_cpu_seconds_total{mode="idle"}[5m])
          )

      # Memory usage percentage
      - record: instance:node_memory_utilization:ratio
        expr: |
          1 - (
            node_memory_MemAvailable_bytes
            / node_memory_MemTotal_bytes
          )

      # Disk usage percentage
      - record: instance:node_filesystem_utilization:ratio
        expr: |
          1 - (
            node_filesystem_avail_bytes{mountpoint="/"}
            / node_filesystem_size_bytes{mountpoint="/"}
          )

  - name: http_recording_rules
    rules:
      # Request rate per second
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      # Error rate
      - record: job:http_errors:ratio5m
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
          / sum by (job) (rate(http_requests_total[5m]))

      # 99th percentile latency
      - record: job:http_request_duration_seconds:p99
        expr: |
          histogram_quantile(0.99,
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          )
```

## Alerting Rules

```yaml
# /etc/prometheus/rules/alerting_rules.yml
groups:
  - name: node_alerts
    rules:
      - alert: HighCPUUsage
        expr: instance:node_cpu_utilization:ratio > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is above 90% for more than 5 minutes (current: {{ $value | printf \"%.2f\" }})"

      - alert: HighMemoryUsage
        expr: instance:node_memory_utilization:ratio > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is above 90% for more than 5 minutes"

      - alert: DiskSpaceLow
        expr: instance:node_filesystem_utilization:ratio > 0.85
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Disk usage is above 85%"

      - alert: InstanceDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
          description: "{{ $labels.job }} instance {{ $labels.instance }} has been down for more than 2 minutes"

  - name: http_alerts
    rules:
      - alert: HighErrorRate
        expr: job:http_errors:ratio5m > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate in {{ $labels.job }}"
          description: "Error rate is above 5%"

      - alert: HighLatency
        expr: job:http_request_duration_seconds:p99 > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency in {{ $labels.job }}"
          description: "99th percentile latency is above 1 second"
```

## Конфигурация Alertmanager

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: 'password'
  slack_api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default-receiver'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true
    - match:
        severity: critical
      receiver: 'slack-critical'
    - match:
        severity: warning
      receiver: 'slack-warnings'
    - match:
        team: database
      receiver: 'database-team'

receivers:
  - name: 'default-receiver'
    email_configs:
      - to: 'team@example.com'
        send_resolved: true

  - name: 'slack-critical'
    slack_configs:
      - channel: '#alerts-critical'
        send_resolved: true
        title: '{{ template "slack.default.title" . }}'
        text: '{{ template "slack.default.text" . }}'

  - name: 'slack-warnings'
    slack_configs:
      - channel: '#alerts-warnings'
        send_resolved: true

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'your-pagerduty-service-key'
        send_resolved: true

  - name: 'database-team'
    email_configs:
      - to: 'database-team@example.com'
    slack_configs:
      - channel: '#database-alerts'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster', 'service']
```

## Флаги командной строки

```bash
# Основные флаги
prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --storage.tsdb.retention.time=15d \
  --storage.tsdb.retention.size=50GB \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle \
  --web.enable-admin-api \
  --log.level=info \
  --log.format=json

# Просмотр всех флагов
prometheus --help
```

## Валидация конфигурации

```bash
# Проверка конфигурации prometheus.yml
promtool check config prometheus.yml

# Проверка правил
promtool check rules /etc/prometheus/rules/*.yml

# Тестирование правил
promtool test rules tests.yml

# Проверка конфигурации alertmanager
amtool check-config alertmanager.yml

# Проверка конфигурации через API
curl -X POST http://localhost:9090/-/reload
```

## Горячая перезагрузка конфигурации

```bash
# Через SIGHUP
kill -HUP $(pgrep prometheus)

# Через HTTP API (требуется --web.enable-lifecycle)
curl -X POST http://localhost:9090/-/reload

# Проверка статуса
curl http://localhost:9090/-/ready
curl http://localhost:9090/-/healthy
```

## Best Practices

### 1. Организация файлов

```
/etc/prometheus/
├── prometheus.yml            # Основная конфигурация
├── rules/
│   ├── recording_rules.yml   # Recording rules
│   └── alerting_rules.yml    # Alerting rules
├── targets/
│   ├── web-servers.yml       # File-based service discovery
│   └── databases.yml
└── certs/
    ├── ca.crt
    └── client.crt
```

### 2. File-based Service Discovery

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'web-servers'
    file_sd_configs:
      - files:
          - '/etc/prometheus/targets/web-servers.yml'
        refresh_interval: 30s
```

```yaml
# /etc/prometheus/targets/web-servers.yml
- targets:
    - 'web-1.example.com:8080'
    - 'web-2.example.com:8080'
  labels:
    env: production
    role: webserver
```

### 3. Мониторинг самого Prometheus

Всегда мониторьте сам Prometheus:

```yaml
# Важные метрики для мониторинга Prometheus
prometheus_tsdb_head_samples_appended_total    # Количество добавленных сэмплов
prometheus_tsdb_compaction_duration_seconds    # Время компактации
prometheus_rule_evaluation_duration_seconds    # Время вычисления правил
prometheus_target_scrape_pool_sync_total       # Синхронизация целей
prometheus_config_last_reload_successful       # Успешность перезагрузки конфига
```

## Частые ошибки

1. **Не указан scrape_timeout** — если таргет отвечает медленно, скрейпинг может не успевать

2. **Слишком короткий scrape_interval** — увеличивает нагрузку на Prometheus и таргеты

3. **Не настроен retention** — диск может быстро заполниться

4. **Отсутствие резервного копирования** — TSDB нужно регулярно бэкапить

5. **Игнорирование remote_write** — для долгосрочного хранения нужен remote storage

## Заключение

Правильная настройка Prometheus — ключ к эффективному мониторингу. Используйте recording rules для оптимизации запросов, настройте alerting rules для своевременного оповещения о проблемах и не забывайте про регулярную валидацию конфигурации.
