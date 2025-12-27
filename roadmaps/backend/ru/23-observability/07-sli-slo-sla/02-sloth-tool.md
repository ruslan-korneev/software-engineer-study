# Sloth - Инструмент для генерации SLO

## Введение

**Sloth** - это open-source инструмент для генерации правил Prometheus на основе спецификации SLO. Он позволяет описывать SLO в декларативном формате YAML и автоматически генерирует все необходимые recording rules и alerting rules для мониторинга SLO.

## Зачем нужен Sloth?

### Проблемы ручного создания SLO правил

1. **Сложность расчетов**: Формулы для multi-window multi-burn-rate алертинга сложны
2. **Риск ошибок**: Легко допустить ошибку в PromQL запросах
3. **Поддержка**: Трудно поддерживать консистентность при изменениях
4. **Дублирование**: Много повторяющегося кода для каждого SLO

### Как Sloth решает эти проблемы

```yaml
# Вместо десятков строк PromQL правил - одна спецификация
version: "prometheus/v1"
service: "my-api"
slos:
  - name: "requests-availability"
    objective: 99.9
    sli:
      events:
        error_query: sum(rate(http_requests_total{status=~"5.."}[{{.window}}]))
        total_query: sum(rate(http_requests_total[{{.window}}]))
    alerting:
      page_alert:
        labels:
          severity: critical
```

## Установка Sloth

### Через Homebrew (macOS/Linux)

```bash
brew install slok/sloth/sloth
```

### Через Docker

```bash
docker pull ghcr.io/slok/sloth:latest
```

### Через Go

```bash
go install github.com/slok/sloth/cmd/sloth@latest
```

### Через Helm (для Kubernetes)

```bash
helm repo add sloth https://slok.github.io/sloth
helm install sloth sloth/sloth
```

## Структура спецификации Sloth

### Базовая структура

```yaml
version: "prometheus/v1"
service: "service-name"
labels:
  owner: "team-name"
slos:
  - name: "slo-name"
    objective: 99.9
    description: "Описание SLO"
    sli:
      # Определение SLI
    alerting:
      # Конфигурация алертов
```

### Полная спецификация SLO

```yaml
version: "prometheus/v1"
service: "payment-api"
labels:
  team: "payments"
  tier: "tier-1"

slos:
  - name: "availability"
    objective: 99.95
    description: "Доступность платежного API"
    labels:
      category: "availability"

    sli:
      events:
        error_query: |
          sum(rate(http_requests_total{
            service="payment-api",
            status=~"5.."
          }[{{.window}}]))
        total_query: |
          sum(rate(http_requests_total{
            service="payment-api"
          }[{{.window}}]))

    alerting:
      name: "PaymentAPIHighErrorRate"
      labels:
        severity: "critical"
        team: "payments"
      annotations:
        summary: "Высокий уровень ошибок в Payment API"
        runbook: "https://wiki.example.com/runbooks/payment-api-errors"
      page_alert:
        labels:
          severity: "critical"
      ticket_alert:
        labels:
          severity: "warning"
```

## Типы SLI в Sloth

### 1. Event-based SLI (на основе событий)

Самый распространенный тип - соотношение хороших событий к общему количеству.

```yaml
sli:
  events:
    error_query: |
      sum(rate(http_requests_total{status=~"5.."}[{{.window}}]))
    total_query: |
      sum(rate(http_requests_total[{{.window}}]))
```

### 2. Raw SLI (произвольный запрос)

Для случаев, когда SLI уже выражается как ratio.

```yaml
sli:
  raw:
    error_ratio_query: |
      1 - (
        sum(rate(successful_operations_total[{{.window}}]))
        /
        sum(rate(total_operations_total[{{.window}}]))
      )
```

### 3. Plugin-based SLI

Использование предопределенных плагинов для типовых сценариев.

```yaml
sli:
  plugin:
    id: "sloth-common/http-latency"
    options:
      bucket: "0.3"
      filter: 'job="my-service"'
```

## Генерация правил Prometheus

### Команда генерации

```bash
# Генерация правил из файла
sloth generate -i my-slos.yaml -o prometheus-rules.yaml

# Генерация с валидацией
sloth generate -i my-slos.yaml -o prometheus-rules.yaml --validate

# Генерация для нескольких файлов
sloth generate -i slos/*.yaml -o prometheus-rules.yaml
```

### Пример входного файла

```yaml
# slos/api-slos.yaml
version: "prometheus/v1"
service: "user-api"
slos:
  - name: "requests-availability"
    objective: 99.9
    sli:
      events:
        error_query: sum(rate(http_requests_total{job="user-api",code=~"5.."}[{{.window}}]))
        total_query: sum(rate(http_requests_total{job="user-api"}[{{.window}}]))
    alerting:
      page_alert:
        labels:
          severity: critical
      ticket_alert:
        labels:
          severity: warning
```

### Сгенерированные правила

Sloth генерирует следующие типы правил:

#### Recording Rules

```yaml
groups:
  - name: sloth-slo-sli-recordings-user-api-requests-availability
    rules:
      # SLI за разные временные окна
      - record: slo:sli_error:ratio_rate5m
        expr: |
          sum(rate(http_requests_total{job="user-api",code=~"5.."}[5m]))
          /
          sum(rate(http_requests_total{job="user-api"}[5m]))
        labels:
          sloth_service: user-api
          sloth_slo: requests-availability

      - record: slo:sli_error:ratio_rate30m
        expr: |
          # ... аналогично для 30m

      - record: slo:sli_error:ratio_rate1h
        expr: |
          # ... аналогично для 1h

      # ... и другие временные окна
```

#### Error Budget Recording Rules

```yaml
- name: sloth-slo-meta-recordings-user-api-requests-availability
  rules:
    # Целевое значение SLO
    - record: slo:objective:ratio
      expr: vector(0.999)
      labels:
        sloth_service: user-api
        sloth_slo: requests-availability

    # Error budget (оставшийся)
    - record: slo:error_budget:ratio
      expr: vector(1 - 0.999)
      labels:
        sloth_service: user-api
        sloth_slo: requests-availability

    # Процент потраченного Error Budget за 30 дней
    - record: slo:current_burn_rate:ratio
      expr: |
        slo:sli_error:ratio_rate5m{sloth_slo="requests-availability"}
        /
        slo:error_budget:ratio{sloth_slo="requests-availability"}
```

#### Alerting Rules (Multi-window Multi-burn-rate)

```yaml
- name: sloth-slo-alerts-user-api-requests-availability
  rules:
    # Page Alert - быстрое реагирование (критическая ситуация)
    - alert: UserAPIRequestsAvailabilityPageAlert
      expr: |
        (
          slo:sli_error:ratio_rate5m{sloth_slo="requests-availability"} > (14.4 * 0.001)
          and
          slo:sli_error:ratio_rate1h{sloth_slo="requests-availability"} > (14.4 * 0.001)
        )
        or
        (
          slo:sli_error:ratio_rate30m{sloth_slo="requests-availability"} > (6 * 0.001)
          and
          slo:sli_error:ratio_rate6h{sloth_slo="requests-availability"} > (6 * 0.001)
        )
      labels:
        severity: critical
        sloth_severity: page
      annotations:
        summary: "High error rate burning SLO budget"

    # Ticket Alert - медленное реагирование (требует внимания)
    - alert: UserAPIRequestsAvailabilityTicketAlert
      expr: |
        (
          slo:sli_error:ratio_rate2h{sloth_slo="requests-availability"} > (3 * 0.001)
          and
          slo:sli_error:ratio_rate1d{sloth_slo="requests-availability"} > (3 * 0.001)
        )
        or
        (
          slo:sli_error:ratio_rate6h{sloth_slo="requests-availability"} > (1 * 0.001)
          and
          slo:sli_error:ratio_rate3d{sloth_slo="requests-availability"} > (1 * 0.001)
        )
      labels:
        severity: warning
        sloth_severity: ticket
```

## Multi-window Multi-burn-rate алертинг

### Концепция

Sloth использует подход multi-window multi-burn-rate для алертинга, который описан в SRE книге Google.

```
Burn Rate = Фактический уровень ошибок / Допустимый уровень ошибок (error budget)
```

### Таблица Burn Rate

| Burn Rate | Бюджет израсходуется за | Тип алерта |
|-----------|------------------------|------------|
| 14.4x | 1 час | Page (critical) |
| 6x | 4 часа | Page (critical) |
| 3x | 10 часов | Ticket (warning) |
| 1x | 30 дней | Без алерта |

### Временные окна для алертов

```yaml
page_alerts:
  # Быстрые окна - обнаружение острых проблем
  - short_window: 5m
    long_window: 1h
    burn_rate_multiplier: 14.4

  # Средние окна - устойчивые проблемы
  - short_window: 30m
    long_window: 6h
    burn_rate_multiplier: 6

ticket_alerts:
  # Медленные окна - постепенная деградация
  - short_window: 2h
    long_window: 1d
    burn_rate_multiplier: 3

  # Очень медленные окна - тренды
  - short_window: 6h
    long_window: 3d
    burn_rate_multiplier: 1
```

## Интеграция с Kubernetes

### Kubernetes Operator

Sloth предоставляет Kubernetes оператор для декларативного управления SLO.

```bash
# Установка оператора
kubectl apply -f https://raw.githubusercontent.com/slok/sloth/main/deploy/kubernetes/raw/sloth.yaml
```

### PrometheusServiceLevel CRD

```yaml
apiVersion: sloth.slok.dev/v1
kind: PrometheusServiceLevel
metadata:
  name: user-api-slos
  namespace: monitoring
spec:
  service: "user-api"
  labels:
    team: "platform"
  slos:
    - name: "requests-availability"
      objective: 99.9
      description: "Доступность User API"
      sli:
        events:
          errorQuery: |
            sum(rate(http_requests_total{job="user-api",code=~"5.."}[{{.window}}]))
          totalQuery: |
            sum(rate(http_requests_total{job="user-api"}[{{.window}}]))
      alerting:
        pageAlert:
          labels:
            severity: critical
        ticketAlert:
          labels:
            severity: warning
```

### Применение CRD

```bash
kubectl apply -f user-api-slos.yaml

# Проверка созданных правил
kubectl get prometheusrules -n monitoring
```

## Плагины Sloth

### Встроенные плагины

Sloth поставляется с набором готовых плагинов для типовых сценариев.

#### HTTP Availability

```yaml
sli:
  plugin:
    id: "sloth-common/http-availability"
    options:
      filter: 'job="my-service"'
      error_status_regex: "5.."
```

#### HTTP Latency

```yaml
sli:
  plugin:
    id: "sloth-common/http-latency"
    options:
      filter: 'job="my-service"'
      bucket: "0.3"  # 300ms
```

#### Kubernetes Pod Availability

```yaml
sli:
  plugin:
    id: "sloth-common/kubernetes/pod-availability"
    options:
      namespace: "production"
      pod_regex: "my-app-.*"
```

### Создание собственных плагинов

```yaml
# plugins/custom-availability.yaml
apiVersion: sloth.slok.dev/v1
kind: SLIPlugin
metadata:
  name: custom-availability
spec:
  meta:
    displayName: "Custom Availability"
    description: "Custom availability SLI for our services"
  options:
    - key: "service_name"
      description: "Name of the service"
      required: true
    - key: "namespace"
      description: "Kubernetes namespace"
      default: "default"
  sli:
    events:
      errorQuery: |
        sum(rate(app_requests_total{
          service="{{.options.service_name}}",
          namespace="{{.options.namespace}}",
          result="error"
        }[{{.window}}]))
      totalQuery: |
        sum(rate(app_requests_total{
          service="{{.options.service_name}}",
          namespace="{{.options.namespace}}"
        }[{{.window}}]))
```

## Примеры SLO для различных сервисов

### REST API

```yaml
version: "prometheus/v1"
service: "rest-api"
slos:
  - name: "availability"
    objective: 99.9
    sli:
      events:
        error_query: |
          sum(rate(http_server_requests_seconds_count{
            job="rest-api",
            status=~"5.."
          }[{{.window}}]))
        total_query: |
          sum(rate(http_server_requests_seconds_count{
            job="rest-api"
          }[{{.window}}]))
    alerting:
      page_alert:
        labels:
          severity: critical

  - name: "latency-p99"
    objective: 99
    description: "99% запросов за менее 500ms"
    sli:
      events:
        error_query: |
          sum(rate(http_server_requests_seconds_count{
            job="rest-api"
          }[{{.window}}]))
          -
          sum(rate(http_server_requests_seconds_bucket{
            job="rest-api",
            le="0.5"
          }[{{.window}}]))
        total_query: |
          sum(rate(http_server_requests_seconds_count{
            job="rest-api"
          }[{{.window}}]))
```

### gRPC сервис

```yaml
version: "prometheus/v1"
service: "grpc-service"
slos:
  - name: "grpc-availability"
    objective: 99.95
    sli:
      events:
        error_query: |
          sum(rate(grpc_server_handled_total{
            grpc_service="myservice.MyService",
            grpc_code!="OK"
          }[{{.window}}]))
        total_query: |
          sum(rate(grpc_server_handled_total{
            grpc_service="myservice.MyService"
          }[{{.window}}]))
```

### Kafka Consumer

```yaml
version: "prometheus/v1"
service: "kafka-consumer"
slos:
  - name: "processing-success"
    objective: 99.9
    sli:
      events:
        error_query: |
          sum(rate(kafka_consumer_messages_failed_total{
            consumer_group="my-consumer"
          }[{{.window}}]))
        total_query: |
          sum(rate(kafka_consumer_messages_processed_total{
            consumer_group="my-consumer"
          }[{{.window}}]))

  - name: "lag"
    objective: 99
    description: "Лаг обработки менее 1000 сообщений"
    sli:
      raw:
        error_ratio_query: |
          (
            max(kafka_consumer_group_lag{consumer_group="my-consumer"}) > 1000
          ) or vector(0)
```

## Валидация и тестирование

### Валидация спецификаций

```bash
# Валидация файла
sloth validate -i my-slos.yaml

# Валидация всех файлов в директории
sloth validate -i slos/
```

### Dry-run генерации

```bash
# Показать сгенерированные правила без записи в файл
sloth generate -i my-slos.yaml --dry-run
```

### Тестирование правил

```bash
# Проверка синтаксиса PromQL
promtool check rules prometheus-rules.yaml

# Тестирование правил с примерами данных
promtool test rules tests.yaml
```

## Best Practices использования Sloth

### 1. Организация файлов

```
slos/
├── platform/
│   ├── api-gateway.yaml
│   └── auth-service.yaml
├── payments/
│   ├── payment-api.yaml
│   └── refund-service.yaml
└── common/
    └── plugins/
        └── custom-plugins.yaml
```

### 2. Использование labels

```yaml
labels:
  team: "payments"
  tier: "tier-1"
  environment: "production"
```

### 3. Документирование SLO

```yaml
slos:
  - name: "availability"
    objective: 99.9
    description: |
      Доступность платежного API. Измеряется как процент
      успешных HTTP запросов (не 5xx). Критично для бизнеса,
      так как напрямую влияет на конверсию.
```

### 4. Настройка алертов

```yaml
alerting:
  name: "PaymentAPIAvailability"
  labels:
    team: "payments"
    runbook: "payment-api-availability"
  annotations:
    summary: "Payment API availability SLO at risk"
    description: "Error budget is being consumed faster than expected"
    runbook_url: "https://wiki.example.com/runbooks/payment-api"
    dashboard_url: "https://grafana.example.com/d/payment-api"
```

## Заключение

Sloth значительно упрощает внедрение SLO-ориентированного мониторинга:

1. **Декларативный подход**: Описывайте SLO, а не правила Prometheus
2. **Автоматическая генерация**: Получайте правильные recording и alerting rules
3. **Multi-window алертинг**: Встроенная поддержка передовых практик
4. **Kubernetes-native**: Интеграция через CRD и оператор
5. **Расширяемость**: Создавайте собственные плагины для типовых сценариев

Использование Sloth позволяет сосредоточиться на определении правильных SLO, а не на технических деталях их реализации.
