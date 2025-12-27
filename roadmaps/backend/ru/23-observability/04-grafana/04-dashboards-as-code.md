# Dashboards as Code

## Введение

**Dashboards as Code** — это практика управления дашбордами Grafana через код, а не через UI. Это позволяет версионировать дашборды в Git, автоматизировать деплой и обеспечивать консистентность между окружениями.

### Преимущества подхода

- **Версионирование** — полная история изменений в Git
- **Code Review** — проверка изменений перед применением
- **Автоматизация** — CI/CD для деплоя дашбордов
- **Консистентность** — одинаковые дашборды в dev/staging/prod
- **Откат изменений** — легко вернуться к предыдущей версии
- **Документация** — код сам является документацией

---

## Способы реализации

### 1. Provisioning (встроенный в Grafana)

Grafana может автоматически загружать дашборды из файловой системы.

### 2. Grafana API

Управление через HTTP API.

### 3. Grafonnet (Jsonnet)

Генерация JSON дашбордов из кода на Jsonnet.

### 4. Terraform Provider

Управление через Terraform.

### 5. Kubernetes Operator

Grafana Operator для K8s.

---

## Provisioning

### Структура директорий

```
/etc/grafana/provisioning/
├── dashboards/
│   ├── dashboards.yaml       # Конфигурация провайдеров
│   ├── infrastructure/       # Папка с JSON дашбордами
│   │   ├── node-overview.json
│   │   └── kubernetes.json
│   └── applications/
│       ├── api-metrics.json
│       └── user-service.json
├── datasources/
│   ├── datasources.yaml
│   └── prometheus.yaml
├── alerting/
│   ├── alerting.yaml
│   └── contact-points.yaml
├── notifiers/
│   └── notifiers.yaml
└── plugins/
    └── plugins.yaml
```

### Конфигурация провайдера дашбордов

```yaml
# /etc/grafana/provisioning/dashboards/dashboards.yaml
apiVersion: 1

providers:
  # Инфраструктурные дашборды
  - name: 'Infrastructure'
    orgId: 1
    folder: 'Infrastructure'
    folderUid: 'infrastructure'
    type: file
    disableDeletion: false        # Разрешить удаление через UI
    updateIntervalSeconds: 30     # Интервал проверки изменений
    allowUiUpdates: false         # Запретить изменения через UI
    options:
      path: /var/lib/grafana/dashboards/infrastructure
      foldersFromFilesStructure: true  # Создавать папки из структуры файлов

  # Дашборды приложений
  - name: 'Applications'
    orgId: 1
    folder: 'Applications'
    folderUid: 'applications'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: false
    options:
      path: /var/lib/grafana/dashboards/applications

  # Дашборды с возможностью редактирования
  - name: 'Editable'
    orgId: 1
    folder: 'Editable'
    folderUid: 'editable'
    type: file
    disableDeletion: true
    updateIntervalSeconds: 60
    allowUiUpdates: true          # Разрешить изменения через UI
    options:
      path: /var/lib/grafana/dashboards/editable
```

### Docker Compose с provisioning

```yaml
# docker-compose.yml
version: '3.8'

services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      # Provisioning конфигурация
      - ./provisioning/dashboards:/etc/grafana/provisioning/dashboards
      - ./provisioning/datasources:/etc/grafana/provisioning/datasources
      - ./provisioning/alerting:/etc/grafana/provisioning/alerting
      # Файлы дашбордов
      - ./dashboards:/var/lib/grafana/dashboards
      # Данные Grafana
      - grafana-data:/var/lib/grafana
    networks:
      - monitoring

volumes:
  grafana-data:

networks:
  monitoring:
    driver: bridge
```

### Пример дашборда для provisioning

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
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "drawStyle": "line",
            "fillOpacity": 10,
            "lineWidth": 2
          },
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "yellow", "value": 70 },
              { "color": "red", "value": 90 }
            ]
          },
          "unit": "percent"
        }
      },
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
      "id": 1,
      "options": {
        "legend": {
          "displayMode": "table",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": { "mode": "multi" }
      },
      "targets": [
        {
          "expr": "100 - (avg by(instance) (rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
          "legendFormat": "{{instance}}",
          "refId": "A"
        }
      ],
      "title": "CPU Usage",
      "type": "timeseries"
    }
  ],
  "refresh": "30s",
  "schemaVersion": 38,
  "tags": ["provisioned", "infrastructure"],
  "templating": {
    "list": []
  },
  "time": {
    "from": "now-1h",
    "to": "now"
  },
  "title": "Node Overview",
  "uid": "node-overview-provisioned",
  "version": 1
}
```

---

## Grafonnet (Jsonnet)

### Установка

```bash
# Установка jsonnet
brew install jsonnet  # macOS
apt-get install jsonnet  # Debian/Ubuntu

# Установка grafonnet
git clone https://github.com/grafana/grafonnet.git
# Или через jsonnet-bundler
jb init
jb install github.com/grafana/grafonnet/gen/grafonnet-latest@main
```

### Базовый пример

```jsonnet
// dashboards/node-overview.jsonnet
local grafana = import 'grafonnet/grafana.libsonnet';
local dashboard = grafana.dashboard;
local row = grafana.row;
local prometheus = grafana.prometheus;
local graphPanel = grafana.graphPanel;
local singlestat = grafana.singlestat;

// Определение datasource
local datasource = 'Prometheus';

// CPU Usage панель
local cpuPanel = graphPanel.new(
  title='CPU Usage',
  datasource=datasource,
  format='percent',
  min=0,
  max=100,
  legend_show=true,
  legend_values=true,
  legend_current=true,
  legend_avg=true,
  legend_max=true,
  legend_alignAsTable=true,
).addTarget(
  prometheus.target(
    expr='100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)',
    legendFormat='{{instance}}'
  )
).addSeriesOverride({
  alias: '/.*/',
  fill: 2,
});

// Memory Usage панель
local memoryPanel = graphPanel.new(
  title='Memory Usage',
  datasource=datasource,
  format='percent',
  min=0,
  max=100,
).addTarget(
  prometheus.target(
    expr='(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100',
    legendFormat='{{instance}}'
  )
);

// Uptime статистика
local uptimePanel = singlestat.new(
  title='Uptime',
  datasource=datasource,
  format='dtdurations',
  valueName='current',
  sparklineShow=true,
).addTarget(
  prometheus.target(
    expr='time() - node_boot_time_seconds',
  )
);

// Сборка дашборда
dashboard.new(
  title='Node Overview',
  uid='node-overview-jsonnet',
  tags=['infrastructure', 'nodes', 'jsonnet'],
  editable=true,
  graphTooltip='shared_crosshair',
  refresh='30s',
  time_from='now-1h',
  time_to='now',
)
.addRow(
  row.new(title='System Overview')
  .addPanel(cpuPanel, gridPos={ h: 8, w: 12, x: 0, y: 0 })
  .addPanel(memoryPanel, gridPos={ h: 8, w: 12, x: 12, y: 0 })
)
.addRow(
  row.new(title='Status')
  .addPanel(uptimePanel, gridPos={ h: 4, w: 6, x: 0, y: 8 })
)
```

### Генерация JSON

```bash
# Генерация одного дашборда
jsonnet -J vendor dashboards/node-overview.jsonnet > output/node-overview.json

# Генерация всех дашбордов
for f in dashboards/*.jsonnet; do
  jsonnet -J vendor "$f" > "output/$(basename "$f" .jsonnet).json"
done
```

### Makefile для автоматизации

```makefile
# Makefile
JSONNET = jsonnet
JSONNET_FMT = jsonnetfmt
JB = jb

DASHBOARDS_DIR = dashboards
OUTPUT_DIR = output
VENDOR_DIR = vendor

.PHONY: all clean deps fmt lint build

all: deps build

deps:
	$(JB) install

fmt:
	find $(DASHBOARDS_DIR) -name '*.jsonnet' -exec $(JSONNET_FMT) -i {} \;
	find $(DASHBOARDS_DIR) -name '*.libsonnet' -exec $(JSONNET_FMT) -i {} \;

lint:
	find $(DASHBOARDS_DIR) -name '*.jsonnet' -exec $(JSONNET) --lint {} \;

build: $(OUTPUT_DIR)
	for f in $(DASHBOARDS_DIR)/*.jsonnet; do \
		$(JSONNET) -J $(VENDOR_DIR) "$$f" > $(OUTPUT_DIR)/$$(basename "$$f" .jsonnet).json; \
	done

$(OUTPUT_DIR):
	mkdir -p $(OUTPUT_DIR)

clean:
	rm -rf $(OUTPUT_DIR)
	rm -rf $(VENDOR_DIR)

deploy: build
	./scripts/deploy-dashboards.sh $(OUTPUT_DIR)
```

### Переиспользуемые компоненты

```jsonnet
// lib/panels.libsonnet
local grafana = import 'grafonnet/grafana.libsonnet';
local prometheus = grafana.prometheus;
local graphPanel = grafana.graphPanel;
local singlestat = grafana.singlestat;
local gauge = grafana.gauge;

{
  // Стандартная панель CPU
  cpuUsagePanel(datasource, instance_filter='')::
    graphPanel.new(
      title='CPU Usage',
      datasource=datasource,
      format='percent',
      min=0,
      max=100,
      legend_alignAsTable=true,
      legend_values=true,
      legend_current=true,
      legend_avg=true,
      legend_max=true,
    ).addTarget(
      prometheus.target(
        expr='100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"' + instance_filter + '}[5m])) * 100)',
        legendFormat='{{instance}}'
      )
    ).addThreshold({
      value: 70,
      colorMode: 'warning',
      op: 'gt',
      fill: true,
      line: true,
    }).addThreshold({
      value: 90,
      colorMode: 'critical',
      op: 'gt',
      fill: true,
      line: true,
    }),

  // Стандартная панель Memory
  memoryUsagePanel(datasource, instance_filter='')::
    gauge.new(
      title='Memory Usage',
      datasource=datasource,
    ).addTarget(
      prometheus.target(
        expr='(1 - (node_memory_MemAvailable_bytes' + instance_filter + ' / node_memory_MemTotal_bytes' + instance_filter + ')) * 100',
      )
    ),

  // Панель для HTTP метрик
  httpRequestRatePanel(datasource, service_filter='')::
    graphPanel.new(
      title='Request Rate',
      datasource=datasource,
      format='reqps',
      legend_alignAsTable=true,
    ).addTarget(
      prometheus.target(
        expr='sum by(status_code) (rate(http_requests_total{' + service_filter + '}[5m]))',
        legendFormat='{{status_code}}'
      )
    ),
}
```

```jsonnet
// dashboards/service-dashboard.jsonnet
local grafana = import 'grafonnet/grafana.libsonnet';
local dashboard = grafana.dashboard;
local row = grafana.row;
local template = grafana.template;
local panels = import '../lib/panels.libsonnet';

local datasource = 'Prometheus';

// Переменные
local serviceVar = template.custom(
  name='service',
  query='api-gateway,user-service,payment-service',
  current='api-gateway',
  label='Service',
);

local environmentVar = template.custom(
  name='environment',
  query='production,staging,development',
  current='production',
  label='Environment',
);

// Дашборд
dashboard.new(
  title='Service Dashboard',
  uid='service-dashboard-jsonnet',
  tags=['application', 'jsonnet'],
  editable=true,
  refresh='30s',
)
.addTemplate(serviceVar)
.addTemplate(environmentVar)
.addRow(
  row.new(title='HTTP Metrics')
  .addPanel(
    panels.httpRequestRatePanel(
      datasource=datasource,
      service_filter='service="$service", environment="$environment"'
    ),
    gridPos={ h: 8, w: 12, x: 0, y: 0 }
  )
)
```

---

## Terraform Provider

### Установка

```hcl
# versions.tf
terraform {
  required_providers {
    grafana = {
      source  = "grafana/grafana"
      version = "~> 2.0"
    }
  }
}

# provider.tf
provider "grafana" {
  url  = "http://localhost:3000"
  auth = var.grafana_auth  # API key или basic auth
}

variable "grafana_auth" {
  type      = string
  sensitive = true
}
```

### Управление дашбордами

```hcl
# dashboards.tf

# Папка для дашбордов
resource "grafana_folder" "infrastructure" {
  title = "Infrastructure"
  uid   = "infrastructure"
}

resource "grafana_folder" "applications" {
  title = "Applications"
  uid   = "applications"
}

# Дашборд из JSON файла
resource "grafana_dashboard" "node_overview" {
  folder      = grafana_folder.infrastructure.id
  config_json = file("${path.module}/dashboards/node-overview.json")

  overwrite = true
}

# Дашборд inline
resource "grafana_dashboard" "simple" {
  folder = grafana_folder.infrastructure.id

  config_json = jsonencode({
    title = "Simple Dashboard"
    uid   = "simple-terraform"
    tags  = ["terraform"]

    panels = [
      {
        id    = 1
        title = "CPU Usage"
        type  = "timeseries"
        gridPos = {
          h = 8
          w = 12
          x = 0
          y = 0
        }
        datasource = {
          type = "prometheus"
          uid  = grafana_data_source.prometheus.uid
        }
        targets = [
          {
            expr         = "100 - (avg by(instance) (rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)"
            legendFormat = "{{instance}}"
            refId        = "A"
          }
        ]
      }
    ]

    time = {
      from = "now-1h"
      to   = "now"
    }

    refresh = "30s"
  })
}

# Datasource
resource "grafana_data_source" "prometheus" {
  type = "prometheus"
  name = "Prometheus"
  uid  = "prometheus"
  url  = "http://prometheus:9090"

  is_default = true

  json_data_encoded = jsonencode({
    httpMethod = "POST"
    timeInterval = "15s"
  })
}

# Alert rule
resource "grafana_rule_group" "cpu_alerts" {
  name             = "CPU Alerts"
  folder_uid       = grafana_folder.infrastructure.uid
  interval_seconds = 60
  org_id           = 1

  rule {
    name      = "High CPU Usage"
    condition = "B"

    data {
      ref_id = "A"

      relative_time_range {
        from = 600
        to   = 0
      }

      datasource_uid = grafana_data_source.prometheus.uid
      model = jsonencode({
        expr = "100 - (avg by(instance) (rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)"
        refId = "A"
      })
    }

    data {
      ref_id = "B"

      relative_time_range {
        from = 0
        to   = 0
      }

      datasource_uid = "__expr__"
      model = jsonencode({
        type       = "threshold"
        refId      = "B"
        expression = "A"
        conditions = [
          {
            type = "gt"
            evaluator = {
              type = "gt"
              params = [90]
            }
          }
        ]
      })
    }

    no_data_state  = "NoData"
    exec_err_state = "Error"

    for = "5m"

    labels = {
      severity = "critical"
    }

    annotations = {
      summary     = "High CPU usage detected"
      description = "CPU usage is above 90% on {{ $labels.instance }}"
    }
  }
}
```

### Terraform module для дашбордов

```hcl
# modules/grafana-dashboard/main.tf
variable "folder_uid" {
  type = string
}

variable "dashboard_file" {
  type = string
}

variable "datasource_uid" {
  type    = string
  default = "prometheus"
}

resource "grafana_dashboard" "this" {
  folder = var.folder_uid

  config_json = templatefile(var.dashboard_file, {
    datasource_uid = var.datasource_uid
  })

  overwrite = true
}

output "uid" {
  value = jsondecode(grafana_dashboard.this.config_json).uid
}
```

```hcl
# main.tf
module "node_dashboard" {
  source = "./modules/grafana-dashboard"

  folder_uid     = grafana_folder.infrastructure.uid
  dashboard_file = "${path.module}/dashboards/node-overview.json.tpl"
  datasource_uid = grafana_data_source.prometheus.uid
}
```

---

## CI/CD Pipeline

### GitHub Actions

```yaml
# .github/workflows/grafana-dashboards.yml
name: Deploy Grafana Dashboards

on:
  push:
    branches: [main]
    paths:
      - 'dashboards/**'
      - 'provisioning/**'
  pull_request:
    branches: [main]
    paths:
      - 'dashboards/**'

env:
  GRAFANA_URL: ${{ secrets.GRAFANA_URL }}
  GRAFANA_TOKEN: ${{ secrets.GRAFANA_TOKEN }}

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Validate JSON
        run: |
          for f in dashboards/*.json; do
            echo "Validating $f..."
            jq empty "$f" || exit 1
          done

      - name: Check required fields
        run: |
          for f in dashboards/*.json; do
            uid=$(jq -r '.uid' "$f")
            title=$(jq -r '.title' "$f")

            if [ -z "$uid" ] || [ "$uid" = "null" ]; then
              echo "Error: $f missing uid"
              exit 1
            fi

            if [ -z "$title" ] || [ "$title" = "null" ]; then
              echo "Error: $f missing title"
              exit 1
            fi
          done

  lint:
    runs-on: ubuntu-latest
    needs: validate
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install dashboard-linter
        run: npm install -g @grafana/dashboard-linter

      - name: Lint dashboards
        run: |
          for f in dashboards/*.json; do
            echo "Linting $f..."
            dashboard-lint lint "$f"
          done

  deploy:
    runs-on: ubuntu-latest
    needs: [validate, lint]
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - name: Deploy dashboards
        run: |
          for f in dashboards/*.json; do
            echo "Deploying $f..."

            dashboard=$(cat "$f")
            payload=$(jq -n --argjson dashboard "$dashboard" '{
              dashboard: $dashboard,
              overwrite: true,
              message: "Deployed via CI/CD"
            }')

            curl -X POST "$GRAFANA_URL/api/dashboards/db" \
              -H "Authorization: Bearer $GRAFANA_TOKEN" \
              -H "Content-Type: application/json" \
              -d "$payload" \
              --fail
          done

      - name: Notify on success
        if: success()
        run: |
          echo "Dashboards deployed successfully"

      - name: Notify on failure
        if: failure()
        run: |
          echo "Dashboard deployment failed"
```

### GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - lint
  - deploy

variables:
  GRAFANA_URL: ${GRAFANA_URL}
  GRAFANA_TOKEN: ${GRAFANA_TOKEN}

validate:
  stage: validate
  image: stedolan/jq
  script:
    - |
      for f in dashboards/*.json; do
        echo "Validating $f..."
        jq empty "$f" || exit 1
      done

lint:
  stage: lint
  image: node:18
  script:
    - npm install -g @grafana/dashboard-linter
    - |
      for f in dashboards/*.json; do
        echo "Linting $f..."
        dashboard-lint lint "$f"
      done

deploy:
  stage: deploy
  image: curlimages/curl
  only:
    - main
  script:
    - |
      for f in dashboards/*.json; do
        echo "Deploying $f..."
        curl -X POST "$GRAFANA_URL/api/dashboards/db" \
          -H "Authorization: Bearer $GRAFANA_TOKEN" \
          -H "Content-Type: application/json" \
          -d "{\"dashboard\": $(cat $f), \"overwrite\": true}" \
          --fail
      done
```

---

## Best Practices

### 1. Структура репозитория

```
grafana-dashboards/
├── dashboards/
│   ├── infrastructure/
│   ├── applications/
│   └── databases/
├── provisioning/
│   ├── dashboards/
│   └── datasources/
├── lib/                    # Переиспользуемые компоненты
├── scripts/
│   ├── validate.sh
│   ├── deploy.sh
│   └── export.sh
├── .github/workflows/
├── Makefile
└── README.md
```

### 2. Соглашения

- **Уникальные UID** — используйте понятные uid (не auto-generated)
- **Версионирование** — включайте version в JSON
- **Теги** — добавляйте теги для фильтрации
- **Документация** — описывайте назначение в description

### 3. Безопасность

- **Не храните секреты** в JSON дашбордов
- **Используйте переменные** для datasource UID
- **API keys** храните в секретах CI/CD
- **Ограничьте права** — отдельный API key для деплоя

### 4. Тестирование

- **Валидация JSON** — синтаксическая проверка
- **Lint** — проверка best practices
- **Preview** — деплой в staging перед production

---

## Типичные ошибки

1. **Hardcoded datasource UID** — используйте переменные `${DS_PROMETHEUS}`
2. **Отсутствие uid** — Grafana сгенерирует случайный при импорте
3. **Конфликты версий** — не указан version или overwrite
4. **Забытые секреты** — API keys в коде вместо CI/CD секретов
5. **Отсутствие валидации** — невалидный JSON ломает деплой

---

## Полезные ссылки

- [Grafana Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/)
- [Grafonnet Documentation](https://grafana.github.io/grafonnet/index.html)
- [Terraform Grafana Provider](https://registry.terraform.io/providers/grafana/grafana/latest/docs)
- [Grafana API](https://grafana.com/docs/grafana/latest/developers/http_api/)
