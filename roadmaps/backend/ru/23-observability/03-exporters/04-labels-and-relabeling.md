# Labels и Relabeling в Prometheus

## Введение

**Labels (метки)** — это ключевая концепция в Prometheus, которая позволяет идентифицировать и группировать временные ряды. Каждая метрика в Prometheus представлена именем и набором меток.

**Relabeling** — это мощный механизм модификации меток на различных этапах сбора и хранения метрик. Он позволяет:
- Добавлять, удалять и изменять метки
- Фильтровать targets и метрики
- Динамически настраивать процесс scraping

## Метки (Labels)

### Структура метрики

```
metric_name{label1="value1", label2="value2"} value timestamp
```

Пример:
```
http_requests_total{method="GET", path="/api/users", status="200"} 1234 1640000000
```

### Типы меток

1. **Пользовательские метки** — добавляются разработчиком в коде приложения
2. **Target метки** — добавляются Prometheus при scraping
3. **Служебные метки** — автоматически добавляемые метки (`__name__`, `job`, `instance`)

### Зарезервированные метки

| Метка | Описание |
|-------|----------|
| `__name__` | Имя метрики |
| `job` | Имя job из конфигурации |
| `instance` | Адрес target (host:port) |
| `__address__` | Исходный адрес target |
| `__scheme__` | HTTP или HTTPS |
| `__metrics_path__` | Путь к метрикам |
| `__param_<name>` | URL параметры |

### Правила именования меток

```yaml
# ПРАВИЛЬНО
method="GET"
http_status="200"
kubernetes_pod_name="web-abc123"

# НЕПРАВИЛЬНО (нельзя начинать с цифры)
200_status="ok"  # Ошибка!

# НЕПРАВИЛЬНО (зарезервированный префикс)
__custom_label="value"  # __ зарезервирован для внутренних меток
```

## Relabeling: Основы

### Где применяется relabeling

1. **`relabel_configs`** — модификация меток targets перед scraping
2. **`metric_relabel_configs`** — модификация меток уже собранных метрик
3. **`alert_relabel_configs`** — модификация меток при отправке алертов
4. **`write_relabel_configs`** — модификация при remote write

### Порядок выполнения

```
Target Discovery
      ↓
relabel_configs      ← Работает с target метками
      ↓
Scraping
      ↓
metric_relabel_configs   ← Работает с метриками
      ↓
Storage (TSDB)
```

## Синтаксис Relabel Config

### Базовая структура

```yaml
relabel_configs:
  - source_labels: [label1, label2]    # Исходные метки
    separator: ";"                      # Разделитель (по умолчанию ;)
    target_label: new_label             # Целевая метка
    regex: "(.+)"                       # Регулярное выражение
    replacement: "${1}"                 # Замена
    action: replace                     # Действие
```

### Доступные действия (actions)

| Action | Описание |
|--------|----------|
| `replace` | Заменить значение target_label результатом regex |
| `keep` | Оставить target, если regex совпадает |
| `drop` | Удалить target, если regex совпадает |
| `labelmap` | Применить regex к именам меток |
| `labeldrop` | Удалить метки, соответствующие regex |
| `labelkeep` | Оставить только метки, соответствующие regex |
| `hashmod` | Установить target_label в modulus хеша |
| `lowercase` | Привести значение к нижнему регистру |
| `uppercase` | Привести значение к верхнему регистру |

## Практические примеры Relabeling

### 1. Добавление статической метки

```yaml
scrape_configs:
  - job_name: 'my-app'
    static_configs:
      - targets: ['localhost:8080']
    relabel_configs:
      - target_label: environment
        replacement: production
```

### 2. Извлечение информации из адреса

```yaml
# Извлекаем hostname из instance
relabel_configs:
  - source_labels: [__address__]
    regex: '([^:]+):\d+'
    target_label: hostname
    replacement: '${1}'
```

### 3. Переименование метки

```yaml
# Переименовываем __meta_kubernetes_pod_name в pod
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_name]
    target_label: pod
```

### 4. Объединение нескольких меток

```yaml
# Создаём метку из комбинации других
relabel_configs:
  - source_labels: [env, region, service]
    separator: '-'
    target_label: service_id
    # Результат: "prod-us-east-api"
```

### 5. Фильтрация targets (keep/drop)

```yaml
# Оставить только targets с определённой меткой
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: true

# Исключить определённые targets
relabel_configs:
  - source_labels: [__meta_kubernetes_namespace]
    action: drop
    regex: 'kube-system|monitoring'
```

### 6. Динамическое изменение порта

```yaml
# Заменяем порт на основе аннотации
relabel_configs:
  - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
    regex: '([^:]+)(?::\d+)?;(\d+)'
    replacement: '${1}:${2}'
    target_label: __address__
```

### 7. Labelmap для Service Discovery

```yaml
# Преобразуем метки Kubernetes в обычные
relabel_configs:
  - action: labelmap
    regex: '__meta_kubernetes_pod_label_(.+)'
    # __meta_kubernetes_pod_label_app -> app
```

### 8. Удаление ненужных меток

```yaml
metric_relabel_configs:
  # Удалить все метки, начинающиеся с kubernetes_
  - action: labeldrop
    regex: 'kubernetes_.*'

  # Оставить только определённые метки
  - action: labelkeep
    regex: 'job|instance|method|status'
```

## Metric Relabeling

`metric_relabel_configs` применяется к метрикам после их сбора, но до сохранения в TSDB.

### Фильтрация метрик

```yaml
metric_relabel_configs:
  # Исключить метрики по имени
  - source_labels: [__name__]
    action: drop
    regex: 'go_.*'

  # Оставить только нужные метрики
  - source_labels: [__name__]
    action: keep
    regex: 'http_requests_total|http_request_duration_.*'
```

### Переименование метрик

```yaml
metric_relabel_configs:
  - source_labels: [__name__]
    regex: 'old_metric_name'
    target_label: __name__
    replacement: 'new_metric_name'
```

### Удаление высококардинальных меток

```yaml
metric_relabel_configs:
  # Удаляем метку user_id, которая создаёт слишком много временных рядов
  - action: labeldrop
    regex: 'user_id'
```

## Service Discovery и Relabeling

### Kubernetes SD

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod

    relabel_configs:
      # Scrape только pods с аннотацией prometheus.io/scrape: "true"
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

      # Добавить namespace как метку
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace

      # Добавить имя pod как метку
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name

      # Скопировать все pod labels
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
```

### EC2 SD (AWS)

```yaml
scrape_configs:
  - job_name: 'ec2-instances'
    ec2_sd_configs:
      - region: us-east-1
        access_key: XXX
        secret_key: XXX
        port: 9100

    relabel_configs:
      # Использовать приватный IP
      - source_labels: [__meta_ec2_private_ip]
        target_label: __address__
        replacement: '${1}:9100'

      # Добавить instance ID
      - source_labels: [__meta_ec2_instance_id]
        target_label: instance_id

      # Добавить имя из тега Name
      - source_labels: [__meta_ec2_tag_Name]
        target_label: instance_name

      # Фильтрация по тегу Environment
      - source_labels: [__meta_ec2_tag_Environment]
        action: keep
        regex: 'production|staging'
```

### File SD

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'file-sd'
    file_sd_configs:
      - files:
          - '/etc/prometheus/targets/*.yml'
        refresh_interval: 30s

    relabel_configs:
      - source_labels: [role]
        target_label: job
        replacement: '${1}-exporter'
```

```yaml
# /etc/prometheus/targets/web-servers.yml
- targets:
    - 'web-01:9100'
    - 'web-02:9100'
  labels:
    role: 'web'
    env: 'production'

- targets:
    - 'web-staging-01:9100'
  labels:
    role: 'web'
    env: 'staging'
```

## Продвинутые паттерны

### 1. Hashmod для шардирования

Распределение targets между несколькими Prometheus серверами:

```yaml
# Prometheus сервер 1 (shard 0)
relabel_configs:
  - source_labels: [__address__]
    modulus: 3
    target_label: __tmp_hash
    action: hashmod
  - source_labels: [__tmp_hash]
    regex: ^0$
    action: keep
```

```yaml
# Prometheus сервер 2 (shard 1)
relabel_configs:
  - source_labels: [__address__]
    modulus: 3
    target_label: __tmp_hash
    action: hashmod
  - source_labels: [__tmp_hash]
    regex: ^1$
    action: keep
```

### 2. Условное relabeling

```yaml
# Добавить метку только если другая метка имеет определённое значение
relabel_configs:
  - source_labels: [environment]
    regex: 'production'
    target_label: priority
    replacement: 'high'

  - source_labels: [environment]
    regex: 'staging|development'
    target_label: priority
    replacement: 'low'
```

### 3. Сложные регулярные выражения

```yaml
# Извлечение нескольких групп
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_name]
    regex: '(.+)-([a-z0-9]+)-([a-z0-9]+)'
    target_label: deployment
    replacement: '${1}'

  - source_labels: [__meta_kubernetes_pod_name]
    regex: '(.+)-([a-z0-9]+)-([a-z0-9]+)'
    target_label: replica_set
    replacement: '${1}-${2}'
```

### 4. Blackbox Exporter relabeling

```yaml
scrape_configs:
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://example.com
          - https://api.myservice.com
    relabel_configs:
      # Сохраняем оригинальный target в параметр
      - source_labels: [__address__]
        target_label: __param_target

      # Сохраняем target как instance метку
      - source_labels: [__param_target]
        target_label: instance

      # Перенаправляем scrape на Blackbox Exporter
      - target_label: __address__
        replacement: 'blackbox-exporter:9115'

      # Извлекаем домен из URL
      - source_labels: [__param_target]
        regex: 'https?://([^/]+).*'
        target_label: domain
        replacement: '${1}'
```

## Отладка Relabeling

### Prometheus API

```bash
# Просмотр активных targets
curl http://localhost:9090/api/v1/targets

# Метаданные service discovery
curl http://localhost:9090/api/v1/targets/metadata
```

### Prometheus UI

1. Откройте Status → Targets
2. Посмотрите "Discovered Labels" и "Target Labels"
3. Проверьте, какие метки были добавлены/удалены

### Логирование

```yaml
# prometheus.yml
global:
  # Увеличить уровень логирования для отладки
  scrape_interval: 15s

# Запуск с debug логами
# prometheus --log.level=debug
```

## Best Practices

### 1. Минимизируйте количество меток

```yaml
# ПЛОХО - слишком много меток, высокая кардинальность
http_requests_total{user_id="12345", session_id="abc", request_id="xyz"}

# ХОРОШО - только необходимые метки
http_requests_total{method="GET", status="200", endpoint="/api/users"}
```

### 2. Используйте согласованные имена меток

```yaml
# Создайте стандарт именования
relabel_configs:
  # Все метки в snake_case
  - action: labelmap
    regex: '([A-Z])([a-z]+)'
    replacement: '${1}_${2}'
```

### 3. Удаляйте ненужные метрики на ранней стадии

```yaml
metric_relabel_configs:
  # Удаляем метрики Go runtime, если они не нужны
  - source_labels: [__name__]
    regex: 'go_gc_.*|go_memstats_.*'
    action: drop
```

### 4. Документируйте relabeling правила

```yaml
relabel_configs:
  # Извлекаем имя сервиса из Kubernetes metadata
  # Исходная метка: __meta_kubernetes_service_name
  # Результат: service="my-app"
  - source_labels: [__meta_kubernetes_service_name]
    target_label: service
```

### 5. Тестируйте правила перед применением

```bash
# Используйте promtool для проверки конфигурации
promtool check config prometheus.yml
```

## Типичные ошибки

### 1. Неправильный порядок действий

```yaml
# НЕПРАВИЛЬНО - drop выполнится раньше replace
relabel_configs:
  - source_labels: [__meta_custom_label]
    action: drop
    regex: 'skip'
  - source_labels: [__meta_custom_label]
    target_label: custom  # Это правило не применится к dropped targets

# ПРАВИЛЬНО - сначала преобразуем, потом фильтруем
relabel_configs:
  - source_labels: [__meta_custom_label]
    target_label: custom
  - source_labels: [custom]
    action: drop
    regex: 'skip'
```

### 2. Забытый regex по умолчанию

```yaml
# regex по умолчанию = ".*"
# Это правило заменит ВСЕ метки!
relabel_configs:
  - source_labels: [__address__]
    target_label: instance
    replacement: 'fixed-value'
    # Добавьте regex, если нужно условное применение
    regex: 'specific-host.*'
```

### 3. Неэкранированные специальные символы

```yaml
# НЕПРАВИЛЬНО
regex: 'api.example.com'  # Точка - любой символ!

# ПРАВИЛЬНО
regex: 'api\.example\.com'  # Экранированная точка
```

### 4. Использование __tmp_ меток без очистки

```yaml
relabel_configs:
  - source_labels: [__address__]
    target_label: __tmp_port
    regex: '[^:]+:(\d+)'
    replacement: '${1}'

  - source_labels: [__tmp_port]
    target_label: port

  # __tmp_ метки автоматически удаляются после relabeling
  # Но лучше явно указывать временные метки с префиксом __tmp_
```

## Полезные ссылки

- [Prometheus Relabeling](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config)
- [Life of a Label](https://www.robustperception.io/life-of-a-label)
- [Relabeling Tricks](https://medium.com/quiq-blog/prometheus-relabeling-tricks-6ae62c56cbda)
- [PromLabs Blog](https://promlabs.com/blog/)
