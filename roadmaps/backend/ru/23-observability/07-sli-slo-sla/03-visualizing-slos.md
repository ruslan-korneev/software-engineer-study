# Визуализация SLO

## Введение

Визуализация SLO - критически важный аспект SRE-практик. Хорошие дашборды позволяют командам быстро оценить состояние сервисов, понять тренды и принять обоснованные решения. В этом разделе мы рассмотрим, как создавать эффективные визуализации SLO в Grafana и других инструментах.

## Ключевые метрики для визуализации

### 1. Текущее значение SLI

```promql
# Текущий SLI за последние 5 минут
1 - slo:sli_error:ratio_rate5m{sloth_slo="my-slo"}
```

### 2. Error Budget (оставшийся)

```promql
# Оставшийся error budget в процентах
(
  1 - (
    slo:sli_error:ratio_rate30d{sloth_slo="my-slo"}
    /
    slo:error_budget:ratio{sloth_slo="my-slo"}
  )
) * 100
```

### 3. Burn Rate

```promql
# Текущий burn rate
slo:current_burn_rate:ratio{sloth_slo="my-slo"}
```

### 4. Время до исчерпания бюджета

```promql
# Оставшееся время (в днях) при текущем burn rate
(
  30 * (1 - slo:sli_error:ratio_rate30d{sloth_slo="my-slo"} / slo:error_budget:ratio{sloth_slo="my-slo"})
)
/
slo:current_burn_rate:ratio{sloth_slo="my-slo"}
```

## Типы визуализаций

### 1. Stat Panel - Ключевые показатели

Для отображения одного важного значения:

```yaml
# Grafana Stat Panel Configuration
panel_type: stat
title: "Текущая доступность"
query: |
  100 * (1 - slo:sli_error:ratio_rate5m{sloth_slo="api-availability"})
unit: "percent"
thresholds:
  - value: 0
    color: "red"
  - value: 99
    color: "orange"
  - value: 99.9
    color: "green"
```

### 2. Gauge Panel - Error Budget

```yaml
# Grafana Gauge Panel Configuration
panel_type: gauge
title: "Error Budget Remaining"
query: |
  (1 - (
    slo:sli_error:ratio_rate30d{sloth_slo="api-availability"}
    /
    slo:error_budget:ratio{sloth_slo="api-availability"}
  )) * 100
unit: "percent"
min: 0
max: 100
thresholds:
  - value: 0
    color: "red"
  - value: 25
    color: "orange"
  - value: 50
    color: "yellow"
  - value: 75
    color: "green"
```

### 3. Time Series - Тренд SLI

```yaml
# Grafana Time Series Panel
panel_type: timeseries
title: "SLI Over Time"
queries:
  - name: "Текущий SLI"
    query: |
      100 * (1 - slo:sli_error:ratio_rate5m{sloth_slo="api-availability"})

  - name: "SLO Target"
    query: |
      100 * slo:objective:ratio{sloth_slo="api-availability"}

fill_below: "SLO Target"
legend:
  show: true
  placement: "bottom"
```

### 4. Heatmap - Латентность

```yaml
# Grafana Heatmap Panel
panel_type: heatmap
title: "Request Latency Distribution"
query: |
  sum(rate(http_request_duration_seconds_bucket{job="api"}[5m])) by (le)
data_format: "time_series_buckets"
color_scheme: "interpolateSpectral"
```

## Структура SLO Dashboard

### Рекомендуемая структура

```
┌─────────────────────────────────────────────────────────────────┐
│                    SLO Overview Dashboard                       │
├─────────────────────────────────────────────────────────────────┤
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ │
│ │ Current SLI │ │Error Budget │ │ Burn Rate   │ │Time to      │ │
│ │   99.95%    │ │   67.3%     │ │    1.2x     │ │Exhaustion   │ │
│ │    ✓ OK     │ │   Healthy   │ │   Normal    │ │   25 days   │ │
│ └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SLI Trend (30 days)                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │    ─────────────────────────────────────── 99.9% SLO    │   │
│  │  ∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿ Actual SLI         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Error Budget Consumption                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  ████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░   │   │
│  │  Consumed: 32.7%           Remaining: 67.3%             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Burn Rate Over Time                                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │     ─────────────────────────────────────── 14.4x Page  │   │
│  │     ─────────────────────────────────────── 6x Page     │   │
│  │     ─────────────────────────────────────── 3x Ticket   │   │
│  │  ∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿ Actual Burn Rate   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Grafana Dashboard JSON

### Полный пример dashboard

```json
{
  "dashboard": {
    "title": "SLO Dashboard - Payment API",
    "tags": ["slo", "payments"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Current Availability",
        "type": "stat",
        "gridPos": {"h": 4, "w": 6, "x": 0, "y": 0},
        "targets": [
          {
            "expr": "100 * (1 - slo:sli_error:ratio_rate5m{sloth_slo=\"payment-api-availability\"})",
            "legendFormat": "Availability"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "mode": "absolute",
              "steps": [
                {"color": "red", "value": null},
                {"color": "orange", "value": 99},
                {"color": "green", "value": 99.9}
              ]
            }
          }
        }
      },
      {
        "id": 2,
        "title": "Error Budget Remaining",
        "type": "gauge",
        "gridPos": {"h": 4, "w": 6, "x": 6, "y": 0},
        "targets": [
          {
            "expr": "(1 - (slo:sli_error:ratio_rate30d{sloth_slo=\"payment-api-availability\"} / slo:error_budget:ratio{sloth_slo=\"payment-api-availability\"})) * 100",
            "legendFormat": "Budget Remaining"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "min": 0,
            "max": 100,
            "thresholds": {
              "mode": "absolute",
              "steps": [
                {"color": "red", "value": null},
                {"color": "orange", "value": 25},
                {"color": "yellow", "value": 50},
                {"color": "green", "value": 75}
              ]
            }
          }
        }
      },
      {
        "id": 3,
        "title": "Burn Rate",
        "type": "stat",
        "gridPos": {"h": 4, "w": 6, "x": 12, "y": 0},
        "targets": [
          {
            "expr": "slo:current_burn_rate:ratio{sloth_slo=\"payment-api-availability\"}",
            "legendFormat": "Burn Rate"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "short",
            "thresholds": {
              "mode": "absolute",
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 1},
                {"color": "orange", "value": 3},
                {"color": "red", "value": 6}
              ]
            }
          }
        }
      },
      {
        "id": 4,
        "title": "SLI Trend (30 days)",
        "type": "timeseries",
        "gridPos": {"h": 8, "w": 24, "x": 0, "y": 4},
        "targets": [
          {
            "expr": "100 * (1 - slo:sli_error:ratio_rate1h{sloth_slo=\"payment-api-availability\"})",
            "legendFormat": "SLI (1h avg)"
          },
          {
            "expr": "100 * slo:objective:ratio{sloth_slo=\"payment-api-availability\"}",
            "legendFormat": "SLO Target"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "custom": {
              "fillOpacity": 10
            }
          },
          "overrides": [
            {
              "matcher": {"id": "byName", "options": "SLO Target"},
              "properties": [
                {"id": "custom.lineStyle", "value": {"fill": "dash", "dash": [10, 10]}},
                {"id": "color", "value": {"mode": "fixed", "fixedColor": "red"}}
              ]
            }
          ]
        }
      }
    ]
  }
}
```

## PromQL запросы для SLO визуализаций

### Базовые запросы

#### Текущий SLI (availability)

```promql
# За последние 5 минут
100 * (1 - slo:sli_error:ratio_rate5m{sloth_service="my-service"})

# За последний час
100 * (1 - slo:sli_error:ratio_rate1h{sloth_service="my-service"})

# За последние 30 дней
100 * (1 - slo:sli_error:ratio_rate30d{sloth_service="my-service"})
```

#### Error Budget

```promql
# Общий error budget (в процентах)
100 * slo:error_budget:ratio{sloth_service="my-service"}

# Потраченный error budget
100 * (
  slo:sli_error:ratio_rate30d{sloth_service="my-service"}
  /
  slo:error_budget:ratio{sloth_service="my-service"}
)

# Оставшийся error budget
100 * (
  1 - (
    slo:sli_error:ratio_rate30d{sloth_service="my-service"}
    /
    slo:error_budget:ratio{sloth_service="my-service"}
  )
)
```

#### Burn Rate

```promql
# Текущий burn rate (на основе 5-минутного окна)
slo:sli_error:ratio_rate5m{sloth_service="my-service"}
/
slo:error_budget:ratio{sloth_service="my-service"}

# Burn rate за час
slo:sli_error:ratio_rate1h{sloth_service="my-service"}
/
slo:error_budget:ratio{sloth_service="my-service"}
```

### Продвинутые запросы

#### Прогноз исчерпания бюджета

```promql
# Дней до исчерпания бюджета при текущем burn rate
(
  (1 - slo:sli_error:ratio_rate30d / slo:error_budget:ratio)
  * 30
)
/
(slo:sli_error:ratio_rate1h / slo:error_budget:ratio)
```

#### Compliance за период

```promql
# Процент времени, когда SLO выполнялось
avg_over_time(
  (slo:sli_error:ratio_rate5m{sloth_service="my-service"} < slo:error_budget:ratio{sloth_service="my-service"})
  [30d:1h]
) * 100
```

#### Multi-SLO обзор

```promql
# Все SLO одного сервиса
100 * (1 - slo:sli_error:ratio_rate30d{sloth_service="my-service"})

# Все SLO с низким error budget
100 * (1 - slo:sli_error:ratio_rate30d / slo:error_budget:ratio)
< 25
```

## Варианты визуализации Error Budget

### 1. Linear Budget Burn (простой)

```yaml
panel:
  title: "Error Budget Consumption"
  type: "timeseries"
  queries:
    - name: "Consumed"
      query: |
        100 * (
          slo:sli_error:ratio_rate30d{sloth_slo="my-slo"}
          /
          slo:error_budget:ratio{sloth_slo="my-slo"}
        )
    - name: "Expected (linear)"
      query: |
        100 * (day_of_month() / 30)
```

### 2. Budget Over Time (кумулятивный)

```promql
# Накопленный расход бюджета
sum_over_time(
  (
    slo:sli_error:ratio_rate5m{sloth_slo="my-slo"}
    * 5 * 60  # переводим в "ошибко-секунды"
  )[30d:5m]
)
/
(30 * 24 * 60 * 60 * slo:error_budget:ratio{sloth_slo="my-slo"})
```

### 3. Bar Gauge для нескольких SLO

```yaml
panel:
  title: "Error Budget by Service"
  type: "bargauge"
  query: |
    sort_desc(
      100 * (1 - slo:sli_error:ratio_rate30d / slo:error_budget:ratio)
    )
  orientation: "horizontal"
  thresholds:
    - value: 0
      color: "red"
    - value: 25
      color: "orange"
    - value: 50
      color: "yellow"
    - value: 75
      color: "green"
```

## Создание Grafana Dashboard с помощью Grafonnet

### Установка Grafonnet

```bash
# Через jsonnet-bundler
jb init
jb install github.com/grafana/grafonnet-lib/grafonnet
```

### Пример dashboard на Grafonnet

```jsonnet
local grafana = import 'grafonnet/grafana.libsonnet';
local dashboard = grafana.dashboard;
local row = grafana.row;
local singlestat = grafana.singlestat;
local graphPanel = grafana.graphPanel;
local prometheus = grafana.prometheus;

local sloLabels = {
  sloth_service: 'payment-api',
  sloth_slo: 'availability',
};

local sloSelector = std.join(',', [
  '%s="%s"' % [k, sloLabels[k]]
  for k in std.objectFields(sloLabels)
]);

dashboard.new(
  'SLO Dashboard - Payment API',
  schemaVersion=26,
  tags=['slo', 'payments'],
  time_from='now-30d',
  time_to='now',
)
.addRow(
  row.new(title='SLO Overview')
  .addPanel(
    singlestat.new(
      'Current Availability',
      datasource='Prometheus',
      format='percent',
      decimals=3,
      colorBackground=true,
      thresholds='99,99.9',
      colors=['#d44a3a', '#ff9830', '#299c46'],
    )
    .addTarget(
      prometheus.target(
        'round(100 * (1 - slo:sli_error:ratio_rate5m{%s}), 0.001)' % sloSelector,
      )
    ),
    gridPos={ h: 4, w: 6, x: 0, y: 0 }
  )
  .addPanel(
    singlestat.new(
      'Error Budget Remaining',
      datasource='Prometheus',
      format='percent',
      decimals=1,
      gaugeShow=true,
      gaugeMinValue=0,
      gaugeMaxValue=100,
      thresholds='25,50,75',
      colors=['#d44a3a', '#ff9830', '#FADE2A', '#299c46'],
    )
    .addTarget(
      prometheus.target(
        '100 * (1 - (slo:sli_error:ratio_rate30d{%s} / slo:error_budget:ratio{%s}))' % [sloSelector, sloSelector],
      )
    ),
    gridPos={ h: 4, w: 6, x: 6, y: 0 }
  )
)
.addRow(
  row.new(title='SLI Trend')
  .addPanel(
    graphPanel.new(
      'SLI Over Time',
      datasource='Prometheus',
      format='percent',
      min=99,
      max=100,
      fill=1,
      linewidth=2,
    )
    .addTarget(
      prometheus.target(
        '100 * (1 - slo:sli_error:ratio_rate1h{%s})' % sloSelector,
        legendFormat='SLI (1h)',
      )
    )
    .addTarget(
      prometheus.target(
        '100 * slo:objective:ratio{%s}' % sloSelector,
        legendFormat='SLO Target',
      )
    )
    .addSeriesOverride({
      alias: 'SLO Target',
      dashes: true,
      fill: 0,
      color: '#F2495C',
    }),
    gridPos={ h: 8, w: 24, x: 0, y: 4 }
  )
)
```

## Алерты на дашборде

### Annotations для инцидентов

```yaml
# Grafana Annotation Query
datasource: Prometheus
query: |
  ALERTS{alertname=~".*SLO.*", alertstate="firing"}
color: red
```

### Отображение алертов на графике

```promql
# Показать моменты, когда burn rate превышал порог
(
  slo:sli_error:ratio_rate5m{sloth_slo="my-slo"}
  /
  slo:error_budget:ratio{sloth_slo="my-slo"}
) > 14.4
```

## Agile SLO Dashboard

### Дашборд для ежедневных стендапов

```yaml
panels:
  - title: "SLO Status (All Services)"
    type: "table"
    query: |
      sort_desc(
        label_replace(
          100 * (1 - slo:sli_error:ratio_rate30d / slo:error_budget:ratio),
          "status",
          "$1",
          "sloth_service",
          "(.*)"
        )
      )
    columns:
      - "Service"
      - "SLO"
      - "Budget Remaining"
      - "Burn Rate"
      - "Status"

  - title: "Services at Risk"
    type: "stat"
    query: |
      count(
        (slo:sli_error:ratio_rate30d / slo:error_budget:ratio) > 0.5
      )
    thresholds:
      - value: 0
        color: "green"
        text: "All Clear"
      - value: 1
        color: "yellow"
        text: "Monitor"
      - value: 3
        color: "red"
        text: "Action Required"
```

## Лучшие практики визуализации

### 1. Используйте правильные временные окна

```yaml
# Для текущего статуса - короткие окна
current_status: 5m или 1h

# Для трендов - средние окна
trends: 1h или 6h

# Для compliance - длинные окна
compliance: 7d или 30d
```

### 2. Цветовые схемы

```yaml
# Traffic light для статуса
green: "SLO выполняется с запасом"
yellow: "SLO выполняется, но бюджет расходуется"
orange: "SLO под угрозой"
red: "SLO нарушен"

# Для error budget
green: "> 50% осталось"
yellow: "25-50% осталось"
orange: "10-25% осталось"
red: "< 10% осталось или превышен"
```

### 3. Контекстная информация

```yaml
# Добавляйте links к другим дашбордам
links:
  - title: "Service Dashboard"
    url: "/d/service-dashboard?var-service=${sloth_service}"
  - title: "Runbook"
    url: "https://wiki.example.com/runbooks/${sloth_service}"
  - title: "Incidents"
    url: "https://incidents.example.com?service=${sloth_service}"
```

### 4. Responsive дизайн

```yaml
# Адаптируйте панели для разных экранов
stat_panels:
  desktop: { w: 6, h: 4 }
  mobile: { w: 24, h: 3 }

graph_panels:
  desktop: { w: 12, h: 8 }
  mobile: { w: 24, h: 6 }
```

## Инструменты для визуализации

### 1. Grafana (основной)
- Нативная поддержка Prometheus
- Богатый набор визуализаций
- Dashboard as Code (JSON, Grafonnet)

### 2. Nobl9
- Специализированный инструмент для SLO
- Автоматические дашборды
- Интеграция с множеством источников

### 3. Datadog SLO
- Встроенные SLO виджеты
- Error Budget отслеживание
- Интеграция с APM

### 4. Google Cloud SLO Monitoring
- Нативная интеграция с GCP
- Автоматические алерты
- Error budget отслеживание

## Заключение

Эффективная визуализация SLO требует:

1. **Правильный выбор метрик** - показывайте то, что важно для принятия решений
2. **Понятные визуализации** - используйте подходящие типы графиков
3. **Контекст** - добавляйте пороговые значения, аннотации, ссылки
4. **Иерархия** - от общего к частному (overview -> details)
5. **Автоматизация** - используйте Dashboard as Code для консистентности

Хороший SLO дашборд должен отвечать на вопросы:
- "Как сейчас работает сервис?"
- "Сколько ошибок мы можем себе позволить?"
- "Нужно ли что-то делать прямо сейчас?"
