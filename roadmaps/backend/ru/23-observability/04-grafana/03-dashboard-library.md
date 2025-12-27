# Библиотека дашбордов (Dashboard Library)

## Введение

**Библиотека дашбордов** — это функционал Grafana, который позволяет переиспользовать панели и дашборды между разными проектами, командами и экземплярами Grafana. Это помогает стандартизировать визуализации и сократить время на создание новых дашбордов.

### Зачем нужна библиотека?

- **Консистентность** — единый стиль визуализаций во всей организации
- **Экономия времени** — не нужно создавать панели с нуля
- **Простота обновлений** — изменение в библиотеке распространяется на все дашборды
- **Лучшие практики** — библиотека содержит проверенные решения

---

## Library Panels (Библиотечные панели)

### Что такое Library Panel?

**Library Panel** — это панель, сохранённая в центральной библиотеке Grafana. Она может быть добавлена на несколько дашбордов и обновляется централизованно.

### Создание Library Panel через UI

1. Создайте или выберите существующую панель на дашборде
2. Нажмите на заголовок панели
3. Выберите **More... → Create library panel**
4. Укажите имя и папку для сохранения
5. Нажмите **Create library panel**

### Добавление Library Panel на дашборд

1. В режиме редактирования дашборда нажмите **Add panel**
2. Выберите **Add a panel from the panel library**
3. Найдите нужную панель в списке
4. Панель будет связана с библиотечной версией

### API для работы с Library Panels

```bash
# Получить список всех library panels
curl -X GET "http://localhost:3000/api/library-elements" \
  -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json"

# Создать library panel
curl -X POST "http://localhost:3000/api/library-elements" \
  -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "folderUid": "general",
    "name": "CPU Usage Panel",
    "model": {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "yellow", "value": 70 },
              { "color": "red", "value": 90 }
            ]
          }
        }
      },
      "targets": [
        {
          "expr": "100 - (avg by(instance) (rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
          "refId": "A"
        }
      ],
      "title": "CPU Usage",
      "type": "timeseries"
    },
    "kind": 1
  }'

# Обновить library panel
curl -X PATCH "http://localhost:3000/api/library-elements/{uid}" \
  -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "CPU Usage Panel (Updated)",
    "model": { ... }
  }'

# Удалить library panel
curl -X DELETE "http://localhost:3000/api/library-elements/{uid}" \
  -H "Authorization: Bearer $GRAFANA_TOKEN"

# Получить дашборды, использующие panel
curl -X GET "http://localhost:3000/api/library-elements/{uid}/connections" \
  -H "Authorization: Bearer $GRAFANA_TOKEN"
```

---

## JSON Model Library Panel

```json
{
  "uid": "cpu-usage-panel",
  "name": "CPU Usage Panel",
  "kind": 1,
  "type": "timeseries",
  "description": "Стандартная панель для отображения CPU usage",
  "model": {
    "datasource": {
      "type": "prometheus",
      "uid": "${DS_PROMETHEUS}"
    },
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
          },
          "thresholdsStyle": {
            "mode": "area"
          }
        },
        "mappings": [],
        "thresholds": {
          "mode": "absolute",
          "steps": [
            {
              "color": "green",
              "value": null
            },
            {
              "color": "yellow",
              "value": 70
            },
            {
              "color": "red",
              "value": 90
            }
          ]
        },
        "unit": "percent",
        "min": 0,
        "max": 100
      },
      "overrides": []
    },
    "options": {
      "legend": {
        "calcs": [
          "mean",
          "max",
          "last"
        ],
        "displayMode": "table",
        "placement": "bottom",
        "showLegend": true
      },
      "tooltip": {
        "mode": "multi",
        "sort": "desc"
      }
    },
    "targets": [
      {
        "datasource": {
          "type": "prometheus",
          "uid": "${DS_PROMETHEUS}"
        },
        "editorMode": "code",
        "expr": "100 - (avg by(instance) (rate(node_cpu_seconds_total{mode=\"idle\"}[$__rate_interval])) * 100)",
        "legendFormat": "{{instance}}",
        "range": true,
        "refId": "A"
      }
    ],
    "title": "CPU Usage",
    "type": "timeseries"
  },
  "version": 1,
  "meta": {
    "folderName": "Infrastructure",
    "folderUid": "infrastructure",
    "connectedDashboards": 5,
    "created": "2024-01-15T10:00:00Z",
    "updated": "2024-01-20T14:30:00Z",
    "createdBy": {
      "id": 1,
      "name": "admin"
    },
    "updatedBy": {
      "id": 1,
      "name": "admin"
    }
  }
}
```

---

## Готовые дашборды с Grafana.com

### Каталог дашбордов

[Grafana Dashboards](https://grafana.com/grafana/dashboards/) — официальный каталог с тысячами готовых дашбордов.

### Популярные дашборды

| ID | Название | Описание |
|----|----------|----------|
| 1860 | Node Exporter Full | Полный мониторинг Linux серверов |
| 3662 | Prometheus 2.0 Overview | Обзор самого Prometheus |
| 7362 | MySQL Overview | Мониторинг MySQL |
| 9628 | PostgreSQL Database | Мониторинг PostgreSQL |
| 12740 | Kubernetes Cluster | Обзор Kubernetes кластера |
| 14055 | Nginx Ingress Controller | Мониторинг Nginx Ingress |
| 11074 | Node Exporter for Prometheus | Альтернативный дашборд для node_exporter |
| 13639 | Docker Containers | Мониторинг Docker контейнеров |
| 10000 | Cluster Monitoring for Kubernetes | Детальный мониторинг K8s |

### Импорт дашборда по ID

**Через UI:**
1. Нажмите **+** → **Import**
2. Введите ID дашборда (например, 1860)
3. Нажмите **Load**
4. Выберите datasource
5. Нажмите **Import**

**Через API:**

```bash
# Скачать JSON дашборда
curl -o dashboard.json https://grafana.com/api/dashboards/1860/revisions/latest/download

# Импортировать в Grafana
curl -X POST "http://localhost:3000/api/dashboards/db" \
  -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "dashboard": '"$(cat dashboard.json)"',
    "overwrite": false,
    "inputs": [
      {
        "name": "DS_PROMETHEUS",
        "type": "datasource",
        "pluginId": "prometheus",
        "value": "prometheus"
      }
    ],
    "folderId": 0
  }'
```

**Через Provisioning:**

```yaml
# provisioning/dashboards/dashboards.yaml
apiVersion: 1

providers:
  - name: 'default'
    orgId: 1
    folder: 'Imported'
    folderUid: 'imported'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
```

Затем скачайте JSON файлы в `/var/lib/grafana/dashboards/`.

---

## Создание собственной библиотеки

### Структура проекта

```
grafana-library/
├── panels/
│   ├── infrastructure/
│   │   ├── cpu-usage.json
│   │   ├── memory-usage.json
│   │   ├── disk-usage.json
│   │   └── network-traffic.json
│   ├── application/
│   │   ├── request-rate.json
│   │   ├── error-rate.json
│   │   ├── latency-percentiles.json
│   │   └── active-connections.json
│   └── database/
│       ├── query-duration.json
│       ├── connection-pool.json
│       └── replication-lag.json
├── dashboards/
│   ├── node-overview.json
│   ├── api-metrics.json
│   └── database-health.json
├── variables/
│   └── common-variables.json
└── README.md
```

### Стандартизированная панель

```json
{
  "_comment": "Standard Request Rate Panel",
  "_version": "1.0.0",
  "_author": "Platform Team",
  "_tags": ["application", "http", "rate"],

  "type": "timeseries",
  "title": "Request Rate",
  "description": "HTTP requests per second by status code",

  "datasource": {
    "type": "prometheus",
    "uid": "${DS_PROMETHEUS}"
  },

  "fieldConfig": {
    "defaults": {
      "color": {
        "mode": "palette-classic"
      },
      "custom": {
        "axisCenteredZero": false,
        "axisColorMode": "text",
        "axisLabel": "requests/s",
        "axisPlacement": "auto",
        "drawStyle": "line",
        "fillOpacity": 10,
        "gradientMode": "none",
        "lineInterpolation": "smooth",
        "lineWidth": 2,
        "showPoints": "never",
        "spanNulls": false,
        "stacking": {
          "group": "A",
          "mode": "none"
        }
      },
      "mappings": [],
      "unit": "reqps",
      "decimals": 2
    },
    "overrides": [
      {
        "matcher": {
          "id": "byRegexp",
          "options": ".*5..$"
        },
        "properties": [
          {
            "id": "color",
            "value": {
              "fixedColor": "red",
              "mode": "fixed"
            }
          }
        ]
      },
      {
        "matcher": {
          "id": "byRegexp",
          "options": ".*4..$"
        },
        "properties": [
          {
            "id": "color",
            "value": {
              "fixedColor": "yellow",
              "mode": "fixed"
            }
          }
        ]
      },
      {
        "matcher": {
          "id": "byRegexp",
          "options": ".*2..$"
        },
        "properties": [
          {
            "id": "color",
            "value": {
              "fixedColor": "green",
              "mode": "fixed"
            }
          }
        ]
      }
    ]
  },

  "options": {
    "legend": {
      "calcs": ["mean", "max", "last"],
      "displayMode": "table",
      "placement": "bottom",
      "showLegend": true
    },
    "tooltip": {
      "mode": "multi",
      "sort": "desc"
    }
  },

  "targets": [
    {
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "editorMode": "code",
      "expr": "sum by(status_code) (rate(http_requests_total{service=\"$service\", environment=\"$environment\"}[$__rate_interval]))",
      "legendFormat": "{{status_code}}",
      "range": true,
      "refId": "A"
    }
  ]
}
```

### Скрипт для загрузки библиотеки

```python
#!/usr/bin/env python3
"""
Скрипт для загрузки библиотеки панелей в Grafana
"""

import os
import json
import requests
from pathlib import Path

GRAFANA_URL = os.getenv("GRAFANA_URL", "http://localhost:3000")
GRAFANA_TOKEN = os.getenv("GRAFANA_TOKEN")
LIBRARY_PATH = Path("./panels")

def create_library_panel(panel_json: dict, folder_uid: str = "general") -> dict:
    """Создаёт library panel в Grafana"""

    headers = {
        "Authorization": f"Bearer {GRAFANA_TOKEN}",
        "Content-Type": "application/json"
    }

    payload = {
        "folderUid": folder_uid,
        "name": panel_json.get("title", "Unnamed Panel"),
        "model": panel_json,
        "kind": 1  # Panel type
    }

    response = requests.post(
        f"{GRAFANA_URL}/api/library-elements",
        headers=headers,
        json=payload
    )

    if response.status_code == 200:
        return response.json()
    else:
        print(f"Error: {response.status_code} - {response.text}")
        return None

def load_library():
    """Загружает все панели из библиотеки"""

    for panel_file in LIBRARY_PATH.rglob("*.json"):
        folder_name = panel_file.parent.name

        with open(panel_file) as f:
            panel_json = json.load(f)

        print(f"Loading: {panel_file.name} -> {folder_name}")
        result = create_library_panel(panel_json, folder_name)

        if result:
            print(f"  Created: {result.get('result', {}).get('uid')}")

if __name__ == "__main__":
    if not GRAFANA_TOKEN:
        print("Error: GRAFANA_TOKEN environment variable is required")
        exit(1)

    load_library()
```

---

## Организация папок

### Структура папок в Grafana

```
Grafana/
├── General/
│   └── (дашборды по умолчанию)
├── Infrastructure/
│   ├── Nodes/
│   ├── Kubernetes/
│   └── Networking/
├── Applications/
│   ├── API Gateway/
│   ├── User Service/
│   └── Payment Service/
├── Databases/
│   ├── PostgreSQL/
│   └── Redis/
└── Library/
    ├── Infrastructure Panels/
    └── Application Panels/
```

### Создание папок через API

```bash
# Создать папку
curl -X POST "http://localhost:3000/api/folders" \
  -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "uid": "infrastructure",
    "title": "Infrastructure"
  }'

# Создать вложенную папку (Grafana 9+)
curl -X POST "http://localhost:3000/api/folders" \
  -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "uid": "infrastructure-nodes",
    "title": "Nodes",
    "parentUid": "infrastructure"
  }'
```

---

## Версионирование дашбордов

### История версий в Grafana

Grafana автоматически сохраняет историю изменений дашбордов:

1. Откройте дашборд
2. Нажмите **Dashboard settings** (шестерёнка)
3. Выберите **Versions**
4. Просмотрите изменения и откатите при необходимости

### API для версий

```bash
# Получить историю версий
curl -X GET "http://localhost:3000/api/dashboards/uid/{dashboard_uid}/versions" \
  -H "Authorization: Bearer $GRAFANA_TOKEN"

# Получить конкретную версию
curl -X GET "http://localhost:3000/api/dashboards/uid/{dashboard_uid}/versions/{version}" \
  -H "Authorization: Bearer $GRAFANA_TOKEN"

# Восстановить версию
curl -X POST "http://localhost:3000/api/dashboards/uid/{dashboard_uid}/restore" \
  -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "version": 5
  }'
```

---

## Best Practices

### 1. Организация библиотеки

- **Группируйте по домену** — Infrastructure, Application, Database
- **Используйте понятные имена** — "CPU Usage (Node)" вместо "Panel 1"
- **Добавляйте описания** — что показывает панель, какие метрики
- **Версионируйте** — храните JSON в Git

### 2. Стандартизация

- **Единые цветовые схемы** — зелёный=хорошо, красный=плохо
- **Одинаковые thresholds** — 70% warning, 90% critical
- **Общие переменные** — $environment, $service, $instance
- **Единицы измерения** — всегда указывайте unit

### 3. Переиспользование

- **Library panels для типовых метрик** — CPU, Memory, Disk, Network
- **Template dashboards** — базовые дашборды для микросервисов
- **Общие переменные** — datasource, time range, environment

### 4. Документация

- **README для библиотеки** — как использовать, как контрибьютить
- **Описания в панелях** — что показывает, какие данные нужны
- **Changelog** — что изменилось в новых версиях

---

## Типичные ошибки

1. **Дублирование панелей** — копирование вместо использования library panels
2. **Жёсткие datasource UID** — используйте переменные `${DS_PROMETHEUS}`
3. **Отсутствие версионирования** — нет истории изменений
4. **Неструктурированные папки** — все дашборды в General
5. **Устаревшие дашборды** — импортированные дашборды не обновляются автоматически

---

## Полезные ссылки

- [Grafana Dashboard Library](https://grafana.com/docs/grafana/latest/dashboards/dashboard-library/)
- [Library Panels API](https://grafana.com/docs/grafana/latest/developers/http_api/library_element/)
- [Grafana.com Dashboards](https://grafana.com/grafana/dashboards/)
- [Dashboard JSON Model](https://grafana.com/docs/grafana/latest/dashboards/json-model/)
