# Grafana Loki

## Введение

**Grafana Loki** — это горизонтально масштабируемая система агрегации логов, разработанная Grafana Labs. В отличие от традиционных решений (ELK stack), Loki не индексирует содержимое логов, а индексирует только метки (labels), что делает его значительно более экономичным и простым в эксплуатации.

## Архитектура Loki

### Основные компоненты

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Promtail   │────▶│    Loki     │◀────│   Grafana   │
│  (Agent)    │     │  (Storage)  │     │    (UI)     │
└─────────────┘     └─────────────┘     └─────────────┘
       │                   │
       │                   ▼
       │            ┌─────────────┐
       │            │  Object     │
       │            │  Storage    │
       │            │  (S3/GCS)   │
       │            └─────────────┘
       │
       ▼
┌─────────────────────────────────┐
│         Приложения              │
│  (stdout/stderr, файлы логов)  │
└─────────────────────────────────┘
```

### Компоненты Loki

1. **Distributor** — принимает входящие потоки логов, валидирует и распределяет по ingester'ам
2. **Ingester** — записывает логи в память и периодически сбрасывает на диск (chunks)
3. **Querier** — выполняет запросы LogQL, читает данные из ingester'ов и storage
4. **Query Frontend** — опциональный компонент для кэширования и разделения запросов
5. **Compactor** — сжимает и дедуплицирует chunks в storage

### Сравнение с ELK Stack

| Критерий | Loki | Elasticsearch |
|----------|------|---------------|
| Индексация | Только labels | Полнотекстовая |
| Потребление ресурсов | Низкое | Высокое |
| Сложность | Простая | Сложная |
| Стоимость хранения | ~10x дешевле | Дорого |
| Поиск по содержимому | Медленнее | Быстрее |
| Интеграция с Prometheus | Нативная | Требует настройки |

## Установка и настройка

### Docker Compose (разработка)

```yaml
# docker-compose.yml
version: "3.8"

services:
  loki:
    image: grafana/loki:2.9.0
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml
      - loki-data:/loki

  promtail:
    image: grafana/promtail:2.9.0
    volumes:
      - ./promtail-config.yaml:/etc/promtail/config.yaml
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command: -config.file=/etc/promtail/config.yaml

  grafana:
    image: grafana/grafana:10.0.0
    ports:
      - "3000:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  loki-data:
  grafana-data:
```

### Конфигурация Loki

```yaml
# loki-config.yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093

limits_config:
  reject_old_samples: true
  reject_old_samples_max_age: 168h  # 7 дней
  ingestion_rate_mb: 16
  ingestion_burst_size_mb: 24
  max_streams_per_user: 10000

chunk_store_config:
  max_look_back_period: 0s

table_manager:
  retention_deletes_enabled: true
  retention_period: 720h  # 30 дней
```

### Конфигурация Promtail

```yaml
# promtail-config.yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # Сбор логов из Docker контейнеров
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: 'logstream'
      - source_labels: ['__meta_docker_container_label_com_docker_compose_service']
        target_label: 'service'

  # Сбор логов из файлов
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*log

  # Сбор логов приложения
  - job_name: application
    static_configs:
      - targets:
          - localhost
        labels:
          job: app
          environment: production
          __path__: /var/log/myapp/*.log
    pipeline_stages:
      - json:
          expressions:
            level: level
            message: message
            timestamp: timestamp
            trace_id: trace_id
      - labels:
          level:
          trace_id:
      - timestamp:
          source: timestamp
          format: RFC3339Nano
```

## Pipeline Stages

Promtail использует pipeline stages для обработки логов перед отправкой в Loki.

### Основные stages

```yaml
pipeline_stages:
  # 1. Парсинг JSON
  - json:
      expressions:
        level: level
        msg: message
        user_id: context.user_id

  # 2. Парсинг регулярными выражениями
  - regex:
      expression: '^(?P<ip>\S+) \S+ \S+ \[(?P<timestamp>[\w:/]+\s[+\-]\d{4})\] "(?P<method>\S+)\s?(?P<url>\S+)?\s?(?P<protocol>\S+)?" (?P<status>\d{3}) (?P<size>\d+)'

  # 3. Добавление labels
  - labels:
      level:
      method:
      status:

  # 4. Изменение timestamp
  - timestamp:
      source: timestamp
      format: '02/Jan/2006:15:04:05 -0700'

  # 5. Вывод (изменение сообщения)
  - output:
      source: msg

  # 6. Фильтрация
  - match:
      selector: '{job="app"}'
      stages:
        - drop:
            expression: ".*healthcheck.*"

  # 7. Метрики (для Prometheus)
  - metrics:
      http_requests_total:
        type: Counter
        description: "Total HTTP requests"
        source: status
        config:
          action: inc
```

### Пример комплексной обработки

```yaml
# promtail-config.yaml для JSON логов приложения
scrape_configs:
  - job_name: myapp
    static_configs:
      - targets:
          - localhost
        labels:
          job: myapp
          env: production
          __path__: /var/log/myapp/*.log

    pipeline_stages:
      # Парсим JSON
      - json:
          expressions:
            level: level
            message: message
            timestamp: timestamp
            service: service
            trace_id: trace_id
            user_id: user_id
            duration_ms: duration_ms
            error: error

      # Извлекаем labels
      - labels:
          level:
          service:
          trace_id:

      # Устанавливаем timestamp из лога
      - timestamp:
          source: timestamp
          format: RFC3339Nano

      # Меняем формат сообщения
      - template:
          source: output
          template: '{{ .message }}{{ if .error }} | error: {{ .error }}{{ end }}'

      - output:
          source: output

      # Фильтруем health check логи
      - match:
          selector: '{job="myapp"}'
          stages:
            - drop:
                expression: "health_check"
                drop_counter_reason: healthcheck

      # Создаём метрики
      - metrics:
          request_duration_seconds:
            type: Histogram
            description: "Request duration in seconds"
            source: duration_ms
            config:
              buckets: [10, 50, 100, 250, 500, 1000, 2500, 5000]
```

## LogQL — язык запросов

### Основы синтаксиса

LogQL похож на PromQL и состоит из двух частей:
1. **Log Stream Selector** — выбор потоков логов по labels
2. **Filter Expression** — фильтрация содержимого

```logql
# Базовый запрос
{job="myapp"}

# С фильтрацией по labels
{job="myapp", level="error"}

# С фильтрацией по содержимому
{job="myapp"} |= "error"

# Регулярное выражение
{job="myapp"} |~ "order_id=\\d+"

# Отрицание (не содержит)
{job="myapp"} != "healthcheck"
{job="myapp"} !~ "debug|trace"
```

### Операторы фильтрации

| Оператор | Описание |
|----------|----------|
| `\|=` | Содержит строку |
| `!=` | Не содержит строку |
| `\|~` | Соответствует regex |
| `!~` | Не соответствует regex |

### Парсеры

```logql
# JSON парсер
{job="myapp"} | json

# С извлечением полей
{job="myapp"} | json | level="error"

# Regex парсер
{job="nginx"} | regexp `(?P<ip>\S+) .* "(?P<method>\S+) (?P<path>\S+)"`

# Logfmt парсер
{job="myapp"} | logfmt

# Pattern парсер (быстрее regex)
{job="nginx"} | pattern `<ip> - - [<_>] "<method> <path> <_>" <status> <_>`
```

### Line Format

```logql
# Изменение формата вывода
{job="myapp"}
  | json
  | line_format "{{.level}} - {{.message}} (user: {{.user_id}})"

# С условиями
{job="myapp"}
  | json
  | line_format `{{ if eq .level "error" }}🔴{{ else }}🟢{{ end }} {{.message}}`
```

### Агрегирующие функции (Metric Queries)

```logql
# Количество логов в секунду
rate({job="myapp"}[5m])

# Количество ошибок в минуту
sum(rate({job="myapp", level="error"}[1m])) by (service)

# Топ 10 endpoint'ов по количеству запросов
topk(10, sum(rate({job="nginx"} | pattern `<_> "<_> <path> <_>"` [5m])) by (path))

# Процентиль времени ответа
quantile_over_time(0.95,
  {job="myapp"}
  | json
  | unwrap duration_ms [5m]
) by (endpoint)

# Среднее значение
avg_over_time(
  {job="myapp"}
  | json
  | unwrap response_time [5m]
)

# Количество уникальных значений
count_over_time({job="myapp"} | json | user_id != "" [1h])
```

### Примеры практических запросов

```logql
# Все ошибки за последний час
{job="myapp", level="error"} | json

# Ошибки с определённым trace_id
{job="myapp"} | json | trace_id="abc123" level="error"

# Медленные запросы (> 1 секунды)
{job="myapp"} | json | duration_ms > 1000

# Логи конкретного пользователя
{job="myapp"} | json | user_id="12345"

# Ошибки платежей сгруппированные по типу
sum by (error_type) (
  rate({job="payment-service", level="error"} | json [5m])
)

# Поиск паттернов ошибок
{job="myapp", level="error"}
  | json
  | pattern `<error_type>: <error_message>`
  | line_format "{{.error_type}}: {{.error_message}}"
```

## Отправка логов напрямую из приложения

### Python с python-logging-loki

```python
import logging
import logging_loki

# Настройка Loki handler
loki_handler = logging_loki.LokiHandler(
    url="http://loki:3100/loki/api/v1/push",
    tags={"application": "my-app", "environment": "production"},
    version="1",
)

# Настройка логгера
logger = logging.getLogger("my-app")
logger.addHandler(loki_handler)
logger.setLevel(logging.INFO)

# Использование
logger.info(
    "Order processed",
    extra={"tags": {"order_id": "12345", "user_id": "67890"}},
)
```

### Python с собственной реализацией

```python
import requests
import time
import json
from datetime import datetime
import logging
from queue import Queue
from threading import Thread

class LokiHandler(logging.Handler):
    def __init__(self, url: str, labels: dict, batch_size: int = 100, flush_interval: int = 5):
        super().__init__()
        self.url = url
        self.labels = labels
        self.batch_size = batch_size
        self.flush_interval = flush_interval
        self.queue = Queue()
        self._start_worker()

    def _start_worker(self):
        def worker():
            batch = []
            last_flush = time.time()

            while True:
                try:
                    record = self.queue.get(timeout=1)
                    batch.append(self._format_record(record))

                    if len(batch) >= self.batch_size or (time.time() - last_flush) > self.flush_interval:
                        self._send_batch(batch)
                        batch = []
                        last_flush = time.time()
                except:
                    if batch and (time.time() - last_flush) > self.flush_interval:
                        self._send_batch(batch)
                        batch = []
                        last_flush = time.time()

        thread = Thread(target=worker, daemon=True)
        thread.start()

    def _format_record(self, record: logging.LogRecord) -> tuple:
        timestamp = str(int(record.created * 1e9))  # nanoseconds
        message = json.dumps({
            "message": record.getMessage(),
            "level": record.levelname,
            "logger": record.name,
            "module": record.module,
            "function": record.funcName,
            **getattr(record, 'extra', {})
        })
        return (timestamp, message)

    def _send_batch(self, batch: list):
        if not batch:
            return

        payload = {
            "streams": [{
                "stream": self.labels,
                "values": batch
            }]
        }

        try:
            response = requests.post(
                f"{self.url}/loki/api/v1/push",
                json=payload,
                headers={"Content-Type": "application/json"},
                timeout=5
            )
            response.raise_for_status()
        except Exception as e:
            print(f"Failed to send logs to Loki: {e}")

    def emit(self, record: logging.LogRecord):
        self.queue.put(record)

# Использование
handler = LokiHandler(
    url="http://localhost:3100",
    labels={"job": "myapp", "env": "production"}
)
logger = logging.getLogger("myapp")
logger.addHandler(handler)
logger.setLevel(logging.INFO)

logger.info("Test message", extra={"extra": {"user_id": 123}})
```

### Go с promtail-client

```go
package main

import (
    "time"

    "github.com/afiskon/promtail-client/promtail"
)

func main() {
    // Настройка клиента
    conf := promtail.ClientConfig{
        PushURL:            "http://localhost:3100/api/prom/push",
        Labels:             "{job=\"myapp\",env=\"production\"}",
        BatchWait:          5 * time.Second,
        BatchEntriesNumber: 10000,
    }

    client, err := promtail.NewClientJson(conf)
    if err != nil {
        panic(err)
    }
    defer client.Shutdown()

    // Отправка логов
    client.Infof("User %d logged in", 12345)
    client.Errorf("Failed to process order: %v", err)
}
```

### Go с собственной реализацией

```go
package loki

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
    "strconv"
    "sync"
    "time"
)

type LokiClient struct {
    url       string
    labels    map[string]string
    batch     [][]string
    batchSize int
    mu        sync.Mutex
    client    *http.Client
}

type lokiPayload struct {
    Streams []lokiStream `json:"streams"`
}

type lokiStream struct {
    Stream map[string]string `json:"stream"`
    Values [][]string        `json:"values"`
}

func NewLokiClient(url string, labels map[string]string) *LokiClient {
    lc := &LokiClient{
        url:       url,
        labels:    labels,
        batchSize: 100,
        client:    &http.Client{Timeout: 5 * time.Second},
    }

    go lc.flushPeriodically()
    return lc
}

func (lc *LokiClient) Log(level, message string, fields map[string]interface{}) {
    lc.mu.Lock()
    defer lc.mu.Unlock()

    fields["level"] = level
    fields["message"] = message

    jsonMsg, _ := json.Marshal(fields)

    timestamp := strconv.FormatInt(time.Now().UnixNano(), 10)
    lc.batch = append(lc.batch, []string{timestamp, string(jsonMsg)})

    if len(lc.batch) >= lc.batchSize {
        lc.flush()
    }
}

func (lc *LokiClient) flush() {
    if len(lc.batch) == 0 {
        return
    }

    payload := lokiPayload{
        Streams: []lokiStream{{
            Stream: lc.labels,
            Values: lc.batch,
        }},
    }

    body, _ := json.Marshal(payload)

    req, _ := http.NewRequest("POST", lc.url+"/loki/api/v1/push", bytes.NewBuffer(body))
    req.Header.Set("Content-Type", "application/json")

    resp, err := lc.client.Do(req)
    if err != nil {
        fmt.Printf("Failed to send logs: %v\n", err)
        return
    }
    defer resp.Body.Close()

    lc.batch = nil
}

func (lc *LokiClient) flushPeriodically() {
    ticker := time.NewTicker(5 * time.Second)
    for range ticker.C {
        lc.mu.Lock()
        lc.flush()
        lc.mu.Unlock()
    }
}

// Удобные методы
func (lc *LokiClient) Info(msg string, fields map[string]interface{}) {
    lc.Log("info", msg, fields)
}

func (lc *LokiClient) Error(msg string, fields map[string]interface{}) {
    lc.Log("error", msg, fields)
}

func (lc *LokiClient) Debug(msg string, fields map[string]interface{}) {
    lc.Log("debug", msg, fields)
}
```

## Настройка в Kubernetes

### Helm Chart

```bash
# Добавление репозитория
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Установка Loki Stack (Loki + Promtail + Grafana)
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.enabled=true \
  --set promtail.enabled=true
```

### Values для production

```yaml
# loki-values.yaml
loki:
  auth_enabled: false

  storage:
    type: s3
    s3:
      endpoint: s3.amazonaws.com
      bucketnames: my-loki-bucket
      region: us-east-1
      access_key_id: ${AWS_ACCESS_KEY_ID}
      secret_access_key: ${AWS_SECRET_ACCESS_KEY}

  limits_config:
    enforce_metric_name: false
    reject_old_samples: true
    reject_old_samples_max_age: 168h
    max_entries_limit_per_query: 5000

  schema_config:
    configs:
      - from: 2020-10-24
        store: boltdb-shipper
        object_store: s3
        schema: v11
        index:
          prefix: loki_index_
          period: 24h

  compactor:
    working_directory: /data/loki/compactor
    shared_store: s3
    compaction_interval: 10m
    retention_enabled: true
    retention_delete_delay: 2h
    retention_delete_worker_count: 150

promtail:
  config:
    clients:
      - url: http://loki:3100/loki/api/v1/push

    scrape_configs:
      - job_name: kubernetes-pods
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            target_label: app
          - source_labels: [__meta_kubernetes_pod_label_version]
            target_label: version
          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace
          - source_labels: [__meta_kubernetes_pod_name]
            target_label: pod
```

## Alerting с Loki

### Настройка Ruler

```yaml
# loki-config.yaml
ruler:
  storage:
    type: local
    local:
      directory: /loki/rules
  rule_path: /loki/rules-temp
  alertmanager_url: http://alertmanager:9093
  ring:
    kvstore:
      store: inmemory
  enable_api: true
```

### Правила алертинга

```yaml
# /loki/rules/myapp/alerts.yaml
groups:
  - name: myapp-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate({job="myapp", level="error"}[5m])) by (service) > 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate in {{ $labels.service }}"
          description: "Error rate is {{ $value }} errors/sec"

      - alert: PaymentFailures
        expr: |
          sum(rate({job="payment-service"} |= "payment_failed" [5m])) > 5
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High payment failure rate"

      - alert: SlowRequests
        expr: |
          quantile_over_time(0.95,
            {job="myapp"} | json | unwrap duration_ms [5m]
          ) > 5000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "95th percentile latency is above 5 seconds"
```

## Grafana Dashboards

### Добавление Data Source

```yaml
# grafana-datasources.yaml
apiVersion: 1
datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    jsonData:
      maxLines: 1000
```

### Пример Dashboard (JSON)

```json
{
  "panels": [
    {
      "title": "Log Volume",
      "type": "timeseries",
      "targets": [
        {
          "expr": "sum(rate({job=\"myapp\"}[5m])) by (level)",
          "legendFormat": "{{level}}"
        }
      ]
    },
    {
      "title": "Error Logs",
      "type": "logs",
      "targets": [
        {
          "expr": "{job=\"myapp\", level=\"error\"}"
        }
      ],
      "options": {
        "showTime": true,
        "wrapLogMessage": true
      }
    },
    {
      "title": "Top Error Types",
      "type": "piechart",
      "targets": [
        {
          "expr": "sum by (error_type) (count_over_time({job=\"myapp\", level=\"error\"} | json [1h]))"
        }
      ]
    }
  ]
}
```

## Best Practices

### 1. Правильное использование Labels

```yaml
# Хорошо - низкая кардинальность
labels:
  job: myapp
  environment: production
  service: payment-service
  level: error

# Плохо - высокая кардинальность (создаёт много потоков)
labels:
  user_id: "12345"          # Уникальный для каждого пользователя
  request_id: "abc-123"     # Уникальный для каждого запроса
  timestamp: "2024-01-15"   # Меняется каждый день
```

### 2. Оптимизация запросов

```logql
# Плохо - полное сканирование
{job="myapp"} |= "error"

# Хорошо - сначала фильтрация по labels
{job="myapp", level="error"}

# Плохо - сложный regex на большом объёме
{job="myapp"} |~ "user_id=\\d+ order_id=\\d+ .* error"

# Хорошо - сначала json парсинг, потом фильтрация
{job="myapp"} | json | level="error" | user_id != ""
```

### 3. Retention и хранение

```yaml
# Настройка retention в зависимости от важности
limits_config:
  retention_period: 720h  # 30 дней для обычных логов

# Разные retention для разных логов через разные tenant'ы
# или через compactor configuration
```

### 4. Мониторинг самого Loki

```logql
# Метрики Loki для мониторинга
loki_ingester_streams_created_total
loki_distributor_bytes_received_total
loki_request_duration_seconds_bucket
```

## Типичные ошибки

### 1. Слишком много уникальных labels

```yaml
# Ошибка: высокая кардинальность
labels:
  trace_id: "unique-value"  # Создаёт новый stream для каждого запроса

# Решение: trace_id должен быть в содержимом лога
{"trace_id": "unique-value", "message": "..."}
```

### 2. Неправильный timestamp

```python
# Ошибка: отправка логов с неправильным временем
# Loki отклонит логи слишком старые или из будущего

# Решение: всегда используйте текущее время или настройте reject_old_samples
```

### 3. Отсутствие batch отправки

```python
# Ошибка: отправка каждого лога отдельно
for log in logs:
    send_to_loki(log)  # N HTTP запросов

# Решение: батчинг
batch = []
for log in logs:
    batch.append(log)
    if len(batch) >= 100:
        send_batch_to_loki(batch)
        batch = []
```

## Вывод

Grafana Loki — это мощное и экономичное решение для агрегации логов, особенно хорошо интегрируемое с экосистемой Prometheus и Grafana. Ключевые преимущества:

- Простота эксплуатации по сравнению с ELK
- Низкое потребление ресурсов
- Нативная интеграция с Kubernetes
- Мощный язык запросов LogQL
- Горизонтальная масштабируемость

Правильно настроенный Loki в сочетании с структурированным логированием и Grafana dashboards обеспечивает полноценную observability для современных распределённых систем.
