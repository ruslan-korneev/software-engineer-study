# Создание дашбордов в Grafana

## Введение в дашборды

**Дашборд (Dashboard)** — это набор панелей, организованных на одной странице для визуализации связанных метрик. Дашборды позволяют отслеживать состояние систем, анализировать тренды и быстро выявлять проблемы.

### Структура дашборда

```
Dashboard
├── Rows (опционально) — логическая группировка панелей
├── Panels — отдельные визуализации
│   ├── Query — запрос к datasource
│   ├── Visualization — тип отображения
│   └── Options — настройки панели
├── Variables — переменные для фильтрации
├── Annotations — маркеры событий
└── Time Range — временной диапазон
```

---

## Создание дашборда через UI

### Шаг 1: Создание нового дашборда

1. Нажмите **"+"** в боковом меню
2. Выберите **"Dashboard"**
3. Нажмите **"Add new panel"**

### Шаг 2: Настройка панели

1. **Query** — выберите datasource и напишите запрос
2. **Visualization** — выберите тип визуализации
3. **Panel options** — настройте заголовок, описание
4. **Apply** — сохраните панель

---

## Типы панелей (Visualizations)

### Time Series (Временные ряды)

Основной тип для метрик с временной привязкой.

```promql
# Пример PromQL запроса для CPU usage
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Настройки Time Series:**

```json
{
  "fieldConfig": {
    "defaults": {
      "color": {
        "mode": "palette-classic"
      },
      "custom": {
        "axisCenteredZero": false,
        "axisColorMode": "text",
        "axisLabel": "",
        "axisPlacement": "auto",
        "barAlignment": 0,
        "drawStyle": "line",
        "fillOpacity": 10,
        "gradientMode": "none",
        "hideFrom": {
          "legend": false,
          "tooltip": false,
          "viz": false
        },
        "lineInterpolation": "smooth",
        "lineWidth": 2,
        "pointSize": 5,
        "scaleDistribution": {
          "type": "linear"
        },
        "showPoints": "auto",
        "spanNulls": false,
        "stacking": {
          "group": "A",
          "mode": "none"
        }
      },
      "unit": "percent",
      "min": 0,
      "max": 100,
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "green", "value": null },
          { "color": "yellow", "value": 70 },
          { "color": "red", "value": 90 }
        ]
      }
    }
  }
}
```

### Stat (Статистика)

Отображение одного значения с опциональным фоном.

```promql
# Uptime в днях
(time() - node_boot_time_seconds) / 86400
```

**Настройки Stat:**

```json
{
  "options": {
    "reduceOptions": {
      "values": false,
      "calcs": ["lastNotNull"],
      "fields": ""
    },
    "orientation": "auto",
    "textMode": "auto",
    "colorMode": "value",
    "graphMode": "area",
    "justifyMode": "auto"
  },
  "fieldConfig": {
    "defaults": {
      "unit": "d",
      "decimals": 1,
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "red", "value": null },
          { "color": "yellow", "value": 1 },
          { "color": "green", "value": 7 }
        ]
      }
    }
  }
}
```

### Gauge (Датчик)

Круговой индикатор для отображения процентов или значений в диапазоне.

```promql
# Memory usage в процентах
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

**Настройки Gauge:**

```json
{
  "options": {
    "reduceOptions": {
      "values": false,
      "calcs": ["lastNotNull"],
      "fields": ""
    },
    "orientation": "auto",
    "showThresholdLabels": false,
    "showThresholdMarkers": true
  },
  "fieldConfig": {
    "defaults": {
      "unit": "percent",
      "min": 0,
      "max": 100,
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "green", "value": null },
          { "color": "yellow", "value": 70 },
          { "color": "red", "value": 85 }
        ]
      }
    }
  }
}
```

### Bar Gauge (Полосовой индикатор)

Горизонтальные или вертикальные полосы для сравнения значений.

```promql
# Disk usage по разделам
100 - ((node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes) * 100)
```

### Table (Таблица)

Табличное представление данных.

```promql
# Топ-10 endpoints по RPS
topk(10, sum by(endpoint) (rate(http_requests_total[5m])))
```

**Настройки Table:**

```json
{
  "options": {
    "showHeader": true,
    "cellHeight": "sm",
    "footer": {
      "show": true,
      "reducer": ["sum"],
      "countRows": false
    }
  },
  "fieldConfig": {
    "overrides": [
      {
        "matcher": { "id": "byName", "options": "Value" },
        "properties": [
          { "id": "displayName", "value": "RPS" },
          { "id": "unit", "value": "reqps" },
          { "id": "decimals", "value": 2 }
        ]
      }
    ]
  }
}
```

### Heatmap (Тепловая карта)

Визуализация распределения данных.

```promql
# Histogram для latency
sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
```

### Pie Chart (Круговая диаграмма)

Распределение по категориям.

```promql
# Распределение HTTP статусов
sum by(status_code) (rate(http_requests_total[1h]))
```

### Logs (Логи)

Отображение логов из Loki.

```logql
# LogQL запрос
{namespace="production", app="api"} |= "error" | json | line_format "{{.level}}: {{.message}}"
```

### Node Graph (Граф узлов)

Визуализация связей между сервисами.

### Geomap (Географическая карта)

Отображение данных на карте.

---

## Пример полного дашборда (JSON)

```json
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": {
          "type": "grafana",
          "uid": "-- Grafana --"
        },
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "type": "dashboard"
      }
    ]
  },
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 1,
  "id": null,
  "links": [],
  "liveNow": false,
  "panels": [
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "palette-classic" },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": { "legend": false, "tooltip": false, "viz": false },
            "lineInterpolation": "smooth",
            "lineWidth": 2,
            "pointSize": 5,
            "scaleDistribution": { "type": "linear" },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": { "group": "A", "mode": "none" }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "yellow", "value": 70 },
              { "color": "red", "value": 90 }
            ]
          },
          "unit": "percent"
        },
        "overrides": []
      },
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
      "id": 1,
      "options": {
        "legend": {
          "calcs": ["mean", "max"],
          "displayMode": "table",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": { "mode": "multi", "sort": "desc" }
      },
      "targets": [
        {
          "datasource": { "type": "prometheus", "uid": "prometheus" },
          "editorMode": "code",
          "expr": "100 - (avg by(instance) (rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
          "legendFormat": "{{instance}}",
          "range": true,
          "refId": "A"
        }
      ],
      "title": "CPU Usage",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "yellow", "value": 70 },
              { "color": "red", "value": 85 }
            ]
          },
          "unit": "percent"
        },
        "overrides": []
      },
      "gridPos": { "h": 8, "w": 6, "x": 12, "y": 0 },
      "id": 2,
      "options": {
        "orientation": "auto",
        "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false },
        "showThresholdLabels": false,
        "showThresholdMarkers": true
      },
      "targets": [
        {
          "datasource": { "type": "prometheus", "uid": "prometheus" },
          "editorMode": "code",
          "expr": "(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100",
          "legendFormat": "Memory",
          "range": true,
          "refId": "A"
        }
      ],
      "title": "Memory Usage",
      "type": "gauge"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "red", "value": null },
              { "color": "yellow", "value": 1 },
              { "color": "green", "value": 7 }
            ]
          },
          "unit": "d"
        },
        "overrides": []
      },
      "gridPos": { "h": 4, "w": 6, "x": 18, "y": 0 },
      "id": 3,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false },
        "textMode": "auto"
      },
      "targets": [
        {
          "datasource": { "type": "prometheus", "uid": "prometheus" },
          "editorMode": "code",
          "expr": "(time() - node_boot_time_seconds) / 86400",
          "legendFormat": "Uptime",
          "range": true,
          "refId": "A"
        }
      ],
      "title": "Uptime",
      "type": "stat"
    }
  ],
  "refresh": "30s",
  "schemaVersion": 38,
  "style": "dark",
  "tags": ["infrastructure", "nodes"],
  "templating": { "list": [] },
  "time": { "from": "now-1h", "to": "now" },
  "timepicker": {
    "refresh_intervals": ["5s", "10s", "30s", "1m", "5m", "15m", "30m", "1h"]
  },
  "timezone": "browser",
  "title": "Node Overview",
  "uid": "node-overview",
  "version": 1,
  "weekStart": ""
}
```

---

## Работа с запросами (Queries)

### PromQL примеры

```promql
# Rate — скорость изменения счётчика за период
rate(http_requests_total[5m])

# Increase — абсолютный прирост за период
increase(http_requests_total[1h])

# Sum by — агрегация с группировкой
sum by(service, method) (rate(http_requests_total[5m]))

# Histogram quantile — перцентили
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# Topk — топ N значений
topk(10, sum by(endpoint) (rate(http_requests_total[5m])))

# Absent — проверка отсутствия метрики (для алертов)
absent(up{job="api"})

# Predict linear — прогнозирование
predict_linear(node_filesystem_avail_bytes[1h], 24*3600)
```

### Трансформации данных

Grafana позволяет преобразовывать данные после получения от datasource:

1. **Reduce** — агрегирование рядов в одно значение
2. **Merge** — объединение нескольких запросов
3. **Filter by name** — фильтрация по имени поля
4. **Organize fields** — переименование и скрытие полей
5. **Join by field** — объединение по ключу
6. **Group by** — группировка
7. **Sort by** — сортировка
8. **Calculate field** — вычисляемые поля

```json
{
  "transformations": [
    {
      "id": "reduce",
      "options": {
        "reducers": ["mean", "max", "min"],
        "mode": "reduceFields",
        "includeTimeField": false
      }
    },
    {
      "id": "organize",
      "options": {
        "renameByName": {
          "Mean": "Average",
          "Max": "Peak"
        },
        "excludeByName": {
          "Min": true
        }
      }
    }
  ]
}
```

---

## Настройка рядов (Rows)

Rows позволяют организовать панели в логические группы:

```json
{
  "panels": [
    {
      "collapsed": false,
      "gridPos": { "h": 1, "w": 24, "x": 0, "y": 0 },
      "id": 100,
      "panels": [],
      "title": "System Metrics",
      "type": "row"
    },
    {
      "collapsed": true,
      "gridPos": { "h": 1, "w": 24, "x": 0, "y": 10 },
      "id": 101,
      "panels": [
        {
          "gridPos": { "h": 8, "w": 12, "x": 0, "y": 11 },
          "id": 10,
          "title": "Hidden Panel",
          "type": "timeseries"
        }
      ],
      "title": "Application Metrics (collapsed)",
      "type": "row"
    }
  ]
}
```

---

## Best Practices

### 1. Организация дашбордов

- **Один дашборд — одна тема** (сервис, команда, слой инфраструктуры)
- **Используйте папки** для группировки
- **Добавляйте теги** для поиска
- **Документируйте** в описании дашборда

### 2. Дизайн панелей

- **Размещайте важные метрики вверху**
- **Используйте единообразные цвета** для однотипных метрик
- **Добавляйте thresholds** для визуализации проблем
- **Указывайте единицы измерения** (unit)

### 3. Производительность

- **Ограничивайте количество панелей** (10-15 на дашборд)
- **Используйте разумный refresh interval** (не менее 30s)
- **Оптимизируйте запросы** — избегайте тяжёлых regex
- **Используйте recording rules** для часто используемых запросов

### 4. Переиспользование

- **Используйте переменные** для фильтрации
- **Создавайте библиотеку панелей** для типовых визуализаций
- **Экспортируйте дашборды** в JSON и храните в Git

---

## Типичные ошибки

1. **Слишком много панелей** — дашборд тормозит и сложно читается
2. **Отсутствие thresholds** — непонятно, когда значение плохое
3. **Жёстко заданные значения** вместо переменных
4. **Неоптимальные запросы** — нагрузка на datasource
5. **Отсутствие единиц измерения** — непонятно, что показывает график
6. **Слишком частое обновление** — refresh каждые 5 секунд без необходимости

---

## Экспорт и импорт

### Экспорт дашборда

1. Откройте дашборд
2. Нажмите на иконку шестерёнки (Settings)
3. Выберите **JSON Model**
4. Скопируйте JSON или нажмите **Save to file**

### Импорт дашборда

1. Нажмите **"+"** в боковом меню
2. Выберите **"Import"**
3. Вставьте JSON, загрузите файл или укажите ID с Grafana.com
4. Выберите папку и datasource
5. Нажмите **Import**

---

## Полезные ссылки

- [Grafana Dashboards на Grafana.com](https://grafana.com/grafana/dashboards/)
- [PromQL Cheat Sheet](https://promlabs.com/promql-cheat-sheet/)
- [Grafana Best Practices](https://grafana.com/docs/grafana/latest/best-practices/)
