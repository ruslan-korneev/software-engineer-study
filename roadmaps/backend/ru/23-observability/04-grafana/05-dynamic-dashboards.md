# Динамические дашборды

## Введение

**Динамические дашборды** — это дашборды, которые адаптируются к данным с помощью переменных (variables), повторяющихся панелей (repeat panels) и условного отображения. Они позволяют создавать универсальные дашборды для множества серверов, сервисов или окружений.

### Зачем нужны динамические дашборды?

- **Один дашборд для многих объектов** — не нужно создавать отдельный дашборд для каждого сервера
- **Интерактивность** — пользователь выбирает нужные фильтры
- **Масштабируемость** — автоматически адаптируются при добавлении новых объектов
- **Уменьшение дублирования** — один шаблон вместо десятков похожих дашбордов

---

## Переменные (Variables)

### Типы переменных

| Тип | Описание | Пример использования |
|-----|----------|---------------------|
| **Query** | Значения из datasource | Список инстансов из Prometheus |
| **Custom** | Статический список значений | dev, staging, prod |
| **Text box** | Произвольный ввод | Поиск по имени |
| **Constant** | Константа | Datasource UID |
| **Data source** | Выбор datasource | Prometheus prod/staging |
| **Interval** | Временной интервал | 1m, 5m, 1h |
| **Ad hoc filters** | Динамические фильтры | Любые label/value |

### Query Variable

Получает значения из datasource:

```json
{
  "templating": {
    "list": [
      {
        "name": "instance",
        "type": "query",
        "datasource": {
          "type": "prometheus",
          "uid": "prometheus"
        },
        "definition": "label_values(up, instance)",
        "query": {
          "query": "label_values(up, instance)",
          "refId": "PrometheusVariableQueryEditor-VariableQuery"
        },
        "regex": "",
        "sort": 1,
        "refresh": 2,
        "multi": true,
        "includeAll": true,
        "allValue": ".*",
        "current": {
          "selected": true,
          "text": ["All"],
          "value": ["$__all"]
        },
        "hide": 0,
        "label": "Instance"
      }
    ]
  }
}
```

**PromQL запросы для переменных:**

```promql
# Список всех инстансов
label_values(up, instance)

# Список job'ов
label_values(up, job)

# Сервисы с фильтрацией по окружению
label_values(http_requests_total{environment="$environment"}, service)

# Поды в namespace
label_values(kube_pod_info{namespace="$namespace"}, pod)

# Уникальные endpoints
label_values(http_requests_total, endpoint)

# Метрики, содержащие pattern
label_values({__name__=~"http.*"}, __name__)
```

### Custom Variable

Статический список значений:

```json
{
  "name": "environment",
  "type": "custom",
  "query": "production,staging,development",
  "current": {
    "selected": true,
    "text": "production",
    "value": "production"
  },
  "options": [
    { "text": "production", "value": "production", "selected": true },
    { "text": "staging", "value": "staging", "selected": false },
    { "text": "development", "value": "development", "selected": false }
  ],
  "multi": false,
  "includeAll": false,
  "hide": 0,
  "label": "Environment"
}
```

### Interval Variable

Для динамического изменения временного окна:

```json
{
  "name": "interval",
  "type": "interval",
  "query": "1m,5m,10m,30m,1h,6h,12h,1d",
  "current": {
    "selected": true,
    "text": "5m",
    "value": "5m"
  },
  "auto": true,
  "auto_count": 30,
  "auto_min": "10s",
  "hide": 0,
  "label": "Interval"
}
```

### Data Source Variable

Позволяет переключаться между datasources:

```json
{
  "name": "datasource",
  "type": "datasource",
  "query": "prometheus",
  "regex": "/Prometheus.*/",
  "current": {
    "selected": true,
    "text": "Prometheus Prod",
    "value": "prometheus-prod"
  },
  "hide": 0,
  "label": "Data Source"
}
```

### Text Box Variable

Свободный ввод текста:

```json
{
  "name": "search",
  "type": "textbox",
  "query": "",
  "current": {
    "selected": true,
    "text": "",
    "value": ""
  },
  "hide": 0,
  "label": "Search"
}
```

### Ad Hoc Filters

Динамические фильтры по любым labels:

```json
{
  "name": "adhoc",
  "type": "adhoc",
  "datasource": {
    "type": "prometheus",
    "uid": "prometheus"
  },
  "hide": 0
}
```

---

## Использование переменных в запросах

### В PromQL

```promql
# Простая подстановка
up{instance="$instance"}

# Множественный выбор (multi-select)
up{instance=~"$instance"}

# С regex
http_requests_total{endpoint=~"$endpoint", status_code=~"$status_code"}

# Интервал
rate(http_requests_total[$interval])

# Автоматический интервал
rate(http_requests_total[$__rate_interval])

# Условие на основе переменной
http_requests_total{environment="$environment", service="$service"}
```

### В LogQL (Loki)

```logql
# Фильтрация по приложению
{app="$app", namespace="$namespace"} |= "$search"

# С regex
{app=~"$app"} | json | level="error"
```

### В SQL

```sql
-- PostgreSQL datasource
SELECT time, value
FROM metrics
WHERE
  instance = '$instance'
  AND $__timeFilter(time)
ORDER BY time
```

---

## Специальные переменные Grafana

| Переменная | Описание | Пример |
|------------|----------|--------|
| `$__from` | Начало временного диапазона (epoch ms) | 1609459200000 |
| `$__to` | Конец временного диапазона (epoch ms) | 1609545600000 |
| `$__interval` | Автоматический интервал | 1m |
| `$__interval_ms` | Интервал в миллисекундах | 60000 |
| `$__rate_interval` | Рекомендуемый интервал для rate() | 1m |
| `$__range` | Длительность временного диапазона | 1h |
| `$__range_s` | Диапазон в секундах | 3600 |
| `$__range_ms` | Диапазон в миллисекундах | 3600000 |
| `$__dashboard` | UID текущего дашборда | my-dashboard |
| `$__name` | Имя текущей панели | CPU Usage |
| `$__org` | ID организации | 1 |
| `$__user` | Информация о пользователе | {login: admin} |
| `$timeFilter` | Фильтр времени для SQL | time > ... |

### Примеры использования

```promql
# Автоматический интервал для rate
rate(http_requests_total[$__rate_interval])

# Прогноз на основе текущего диапазона
predict_linear(node_filesystem_avail_bytes[1h], $__range_s)

# Агрегация за весь выбранный период
increase(http_requests_total[$__range])
```

---

## Повторяющиеся панели (Repeat Panels)

### Repeat by Variable

Автоматическое создание копии панели для каждого значения переменной:

```json
{
  "id": 1,
  "title": "CPU Usage - $instance",
  "repeat": "instance",
  "repeatDirection": "h",
  "maxPerRow": 4,
  "type": "timeseries",
  "datasource": {
    "type": "prometheus",
    "uid": "prometheus"
  },
  "targets": [
    {
      "expr": "100 - (avg(rate(node_cpu_seconds_total{mode=\"idle\", instance=\"$instance\"}[5m])) * 100)",
      "legendFormat": "$instance",
      "refId": "A"
    }
  ],
  "gridPos": {
    "h": 8,
    "w": 6,
    "x": 0,
    "y": 0
  }
}
```

### Repeat Rows

Повторение целых рядов:

```json
{
  "id": 100,
  "type": "row",
  "title": "Metrics for $service",
  "repeat": "service",
  "collapsed": false,
  "panels": [
    {
      "id": 101,
      "title": "Request Rate - $service",
      "type": "timeseries",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{service=\"$service\"}[5m]))",
          "refId": "A"
        }
      ],
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 1 }
    },
    {
      "id": 102,
      "title": "Error Rate - $service",
      "type": "timeseries",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{service=\"$service\", status_code=~\"5..\"}[5m]))",
          "refId": "A"
        }
      ],
      "gridPos": { "h": 8, "w": 12, "x": 12, "y": 1 }
    }
  ],
  "gridPos": { "h": 1, "w": 24, "x": 0, "y": 0 }
}
```

---

## Chained Variables (Связанные переменные)

Значения одной переменной зависят от выбора другой:

```json
{
  "templating": {
    "list": [
      {
        "name": "namespace",
        "type": "query",
        "query": "label_values(kube_pod_info, namespace)",
        "refresh": 2,
        "sort": 1
      },
      {
        "name": "deployment",
        "type": "query",
        "query": "label_values(kube_deployment_labels{namespace=\"$namespace\"}, deployment)",
        "refresh": 2,
        "sort": 1
      },
      {
        "name": "pod",
        "type": "query",
        "query": "label_values(kube_pod_info{namespace=\"$namespace\", created_by_name=~\"$deployment.*\"}, pod)",
        "refresh": 2,
        "sort": 1
      }
    ]
  }
}
```

**Порядок обновления:**
1. Пользователь выбирает `namespace`
2. Grafana обновляет список `deployment` для этого namespace
3. Grafana обновляет список `pod` для выбранного deployment

---

## Пример полного динамического дашборда

```json
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": { "type": "grafana", "uid": "-- Grafana --" },
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "type": "dashboard"
      },
      {
        "datasource": { "type": "prometheus", "uid": "${datasource}" },
        "enable": true,
        "expr": "ALERTS{alertstate=\"firing\", service=\"$service\"}",
        "iconColor": "red",
        "name": "Alerts",
        "tagKeys": "alertname,severity",
        "titleFormat": "{{ alertname }}"
      }
    ]
  },
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 1,
  "id": null,
  "links": [
    {
      "asDropdown": true,
      "icon": "external link",
      "includeVars": true,
      "keepTime": true,
      "tags": ["service"],
      "targetBlank": true,
      "title": "Related Dashboards",
      "type": "dashboards"
    }
  ],
  "liveNow": false,
  "panels": [
    {
      "collapsed": false,
      "gridPos": { "h": 1, "w": 24, "x": 0, "y": 0 },
      "id": 1,
      "panels": [],
      "title": "Overview - $service ($environment)",
      "type": "row"
    },
    {
      "datasource": { "type": "prometheus", "uid": "${datasource}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [
            { "options": { "0": { "text": "DOWN" } }, "type": "value" },
            { "options": { "1": { "text": "UP" } }, "type": "value" }
          ],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "red", "value": null },
              { "color": "green", "value": 1 }
            ]
          }
        }
      },
      "gridPos": { "h": 4, "w": 4, "x": 0, "y": 1 },
      "id": 2,
      "options": {
        "colorMode": "background",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false },
        "textMode": "auto"
      },
      "targets": [
        {
          "expr": "up{service=\"$service\", environment=\"$environment\"}",
          "legendFormat": "Status",
          "refId": "A"
        }
      ],
      "title": "Service Status",
      "type": "stat"
    },
    {
      "datasource": { "type": "prometheus", "uid": "${datasource}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "palette-classic" },
          "custom": {
            "drawStyle": "line",
            "fillOpacity": 10,
            "lineWidth": 2,
            "showPoints": "never"
          },
          "unit": "reqps"
        }
      },
      "gridPos": { "h": 8, "w": 10, "x": 4, "y": 1 },
      "id": 3,
      "options": {
        "legend": { "displayMode": "table", "placement": "bottom", "showLegend": true },
        "tooltip": { "mode": "multi", "sort": "desc" }
      },
      "targets": [
        {
          "expr": "sum by(status_code) (rate(http_requests_total{service=\"$service\", environment=\"$environment\"}[$__rate_interval]))",
          "legendFormat": "{{status_code}}",
          "refId": "A"
        }
      ],
      "title": "Request Rate by Status",
      "type": "timeseries"
    },
    {
      "datasource": { "type": "prometheus", "uid": "${datasource}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "palette-classic" },
          "custom": {
            "drawStyle": "line",
            "fillOpacity": 10,
            "lineWidth": 2
          },
          "unit": "s"
        }
      },
      "gridPos": { "h": 8, "w": 10, "x": 14, "y": 1 },
      "id": 4,
      "options": {
        "legend": { "displayMode": "table", "placement": "bottom", "showLegend": true },
        "tooltip": { "mode": "multi", "sort": "desc" }
      },
      "targets": [
        {
          "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket{service=\"$service\", environment=\"$environment\"}[$__rate_interval])) by (le))",
          "legendFormat": "p50",
          "refId": "A"
        },
        {
          "expr": "histogram_quantile(0.90, sum(rate(http_request_duration_seconds_bucket{service=\"$service\", environment=\"$environment\"}[$__rate_interval])) by (le))",
          "legendFormat": "p90",
          "refId": "B"
        },
        {
          "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{service=\"$service\", environment=\"$environment\"}[$__rate_interval])) by (le))",
          "legendFormat": "p99",
          "refId": "C"
        }
      ],
      "title": "Latency Percentiles",
      "type": "timeseries"
    },
    {
      "collapsed": false,
      "gridPos": { "h": 1, "w": 24, "x": 0, "y": 9 },
      "id": 5,
      "panels": [],
      "repeat": "instance",
      "repeatDirection": "h",
      "title": "Instance: $instance",
      "type": "row"
    },
    {
      "datasource": { "type": "prometheus", "uid": "${datasource}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "palette-classic" },
          "custom": { "drawStyle": "line", "fillOpacity": 10 },
          "unit": "percent",
          "min": 0,
          "max": 100
        }
      },
      "gridPos": { "h": 6, "w": 12, "x": 0, "y": 10 },
      "id": 6,
      "repeat": "instance",
      "repeatDirection": "h",
      "maxPerRow": 2,
      "targets": [
        {
          "expr": "100 - (avg(rate(node_cpu_seconds_total{mode=\"idle\", instance=\"$instance\"}[$__rate_interval])) * 100)",
          "legendFormat": "CPU",
          "refId": "A"
        }
      ],
      "title": "CPU Usage - $instance",
      "type": "timeseries"
    }
  ],
  "refresh": "30s",
  "schemaVersion": 38,
  "tags": ["dynamic", "service", "template"],
  "templating": {
    "list": [
      {
        "current": { "selected": false, "text": "Prometheus", "value": "prometheus" },
        "hide": 0,
        "includeAll": false,
        "label": "Data Source",
        "multi": false,
        "name": "datasource",
        "query": "prometheus",
        "queryValue": "",
        "refresh": 1,
        "regex": "",
        "type": "datasource"
      },
      {
        "current": { "selected": true, "text": "production", "value": "production" },
        "hide": 0,
        "includeAll": false,
        "label": "Environment",
        "multi": false,
        "name": "environment",
        "options": [
          { "selected": true, "text": "production", "value": "production" },
          { "selected": false, "text": "staging", "value": "staging" },
          { "selected": false, "text": "development", "value": "development" }
        ],
        "query": "production,staging,development",
        "type": "custom"
      },
      {
        "current": { "selected": false, "text": "All", "value": "$__all" },
        "datasource": { "type": "prometheus", "uid": "${datasource}" },
        "definition": "label_values(up{environment=\"$environment\"}, service)",
        "hide": 0,
        "includeAll": true,
        "label": "Service",
        "multi": true,
        "name": "service",
        "query": { "query": "label_values(up{environment=\"$environment\"}, service)", "refId": "Query" },
        "refresh": 2,
        "regex": "",
        "sort": 1,
        "type": "query"
      },
      {
        "current": { "selected": false, "text": "All", "value": "$__all" },
        "datasource": { "type": "prometheus", "uid": "${datasource}" },
        "definition": "label_values(up{environment=\"$environment\", service=~\"$service\"}, instance)",
        "hide": 0,
        "includeAll": true,
        "label": "Instance",
        "multi": true,
        "name": "instance",
        "query": { "query": "label_values(up{environment=\"$environment\", service=~\"$service\"}, instance)", "refId": "Query" },
        "refresh": 2,
        "regex": "",
        "sort": 1,
        "type": "query"
      },
      {
        "auto": true,
        "auto_count": 30,
        "auto_min": "10s",
        "current": { "selected": false, "text": "auto", "value": "$__auto_interval_interval" },
        "hide": 0,
        "label": "Interval",
        "name": "interval",
        "query": "1m,5m,10m,30m,1h",
        "refresh": 2,
        "type": "interval"
      }
    ]
  },
  "time": { "from": "now-1h", "to": "now" },
  "timepicker": {
    "refresh_intervals": ["5s", "10s", "30s", "1m", "5m", "15m", "30m", "1h"]
  },
  "timezone": "browser",
  "title": "Service Dashboard (Dynamic)",
  "uid": "service-dynamic",
  "version": 1,
  "weekStart": ""
}
```

---

## Dashboard Links

### Типы ссылок

```json
{
  "links": [
    {
      "asDropdown": false,
      "icon": "external link",
      "includeVars": true,
      "keepTime": true,
      "tags": [],
      "targetBlank": true,
      "title": "Node Details",
      "tooltip": "Go to node details dashboard",
      "type": "link",
      "url": "/d/node-details/node-details?var-instance=$instance"
    },
    {
      "asDropdown": true,
      "icon": "dashboard",
      "includeVars": true,
      "keepTime": true,
      "tags": ["service"],
      "targetBlank": false,
      "title": "Related Dashboards",
      "type": "dashboards"
    }
  ]
}
```

### Data Links в панелях

```json
{
  "fieldConfig": {
    "defaults": {
      "links": [
        {
          "title": "View in Logs",
          "url": "/explore?orgId=1&left=%5B%22now-1h%22,%22now%22,%22Loki%22,%7B%22expr%22:%22%7Bservice%3D%5C%22${__field.labels.service}%5C%22%7D%22%7D%5D",
          "targetBlank": true
        },
        {
          "title": "View Traces",
          "url": "/explore?orgId=1&left=%5B%22now-1h%22,%22now%22,%22Tempo%22,%7B%22query%22:%22${__data.fields.traceID}%22%7D%5D",
          "targetBlank": true
        }
      ]
    }
  }
}
```

---

## Best Practices

### 1. Проектирование переменных

- **Используйте понятные имена** — `environment` вместо `env`
- **Добавляйте labels** — понятные для пользователей подписи
- **Include All** — позволяйте выбрать все значения
- **Сортировка** — алфавитная или по значению
- **Refresh** — при изменении времени или загрузке

### 2. Производительность

- **Ограничивайте repeat panels** — не более 10-15 копий
- **Используйте $__rate_interval** — вместо hardcoded интервалов
- **Кэширование** — настройте min_refresh_interval
- **Оптимизируйте запросы** — избегайте regex где возможно

### 3. UX

- **Логичный порядок переменных** — от общего к частному
- **Значения по умолчанию** — разумные defaults
- **Скрывайте технические переменные** — hide для datasource
- **Используйте All Value** — `.*` для regex фильтров

### 4. Связность

- **Chained variables** — связанные переменные для drill-down
- **Dashboard links** — переходы между дашбордами с сохранением контекста
- **Data links** — переход к логам/трейсам из графиков

---

## Типичные ошибки

1. **Слишком много repeat panels** — дашборд тормозит
2. **Неправильный regex** — `$instance` вместо `=~"$instance"` для multi-select
3. **Отсутствие All option** — нельзя посмотреть общую картину
4. **Hardcoded intervals** — `[5m]` вместо `[$__rate_interval]`
5. **Неправильный refresh** — данные в dropdown устаревают
6. **Забытые переменные** — $var в запросе, но переменная не создана

---

## Полезные ссылки

- [Grafana Variables](https://grafana.com/docs/grafana/latest/dashboards/variables/)
- [Repeat Panels](https://grafana.com/docs/grafana/latest/panels-visualizations/configure-panel-options/#configure-repeating-panels)
- [Data Links](https://grafana.com/docs/grafana/latest/panels-visualizations/configure-data-links/)
- [Dashboard Links](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/manage-dashboard-links/)
