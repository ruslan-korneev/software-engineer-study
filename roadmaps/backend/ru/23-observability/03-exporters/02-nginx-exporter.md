# Nginx Exporter

## Введение

**Nginx Exporter** — это экспортёр метрик для Prometheus, который собирает статистику работы веб-сервера Nginx. Он позволяет мониторить ключевые показатели производительности: количество активных подключений, скорость обработки запросов, статусы upstream серверов и многое другое.

Существует несколько версий экспортёров для Nginx:
- **nginx-prometheus-exporter** — официальный экспортёр от NGINX Inc.
- **nginx-vts-exporter** — экспортёр для модуля nginx-module-vts (более детальные метрики)
- **nginx-exporter** — альтернативные реализации

В этом руководстве мы рассмотрим официальный **nginx-prometheus-exporter**.

## Предварительные требования

Для работы экспортёра необходимо включить в Nginx модуль статистики:

### Для Nginx Open Source (stub_status)

```nginx
# /etc/nginx/conf.d/status.conf
server {
    listen 127.0.0.1:8080;
    server_name _;

    location /nginx_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }
}
```

Проверка:
```bash
curl http://127.0.0.1:8080/nginx_status
```

Ответ:
```
Active connections: 291
server accepts handled requests
 16630948 16630948 31070465
Reading: 6 Writing: 179 Waiting: 106
```

### Для Nginx Plus (API модуль)

```nginx
# /etc/nginx/conf.d/api.conf
server {
    listen 127.0.0.1:8080;
    server_name _;

    location /api {
        api write=off;
        allow 127.0.0.1;
        deny all;
    }
}
```

## Установка Nginx Exporter

### Способ 1: Бинарный файл

```bash
# Скачиваем последнюю версию
wget https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v1.1.0/nginx-prometheus-exporter_1.1.0_linux_amd64.tar.gz

# Распаковываем
tar xvfz nginx-prometheus-exporter_1.1.0_linux_amd64.tar.gz

# Перемещаем
sudo mv nginx-prometheus-exporter /usr/local/bin/

# Проверяем
nginx-prometheus-exporter --version
```

### Способ 2: Docker

```bash
# Для Nginx Open Source (stub_status)
docker run -d \
  --name nginx-exporter \
  -p 9113:9113 \
  nginx/nginx-prometheus-exporter:1.1.0 \
  --nginx.scrape-uri=http://host.docker.internal:8080/nginx_status

# Для Nginx Plus
docker run -d \
  --name nginx-exporter \
  -p 9113:9113 \
  nginx/nginx-prometheus-exporter:1.1.0 \
  --nginx.plus \
  --nginx.scrape-uri=http://host.docker.internal:8080/api
```

### Способ 3: Docker Compose

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./conf.d:/etc/nginx/conf.d:ro

  nginx-exporter:
    image: nginx/nginx-prometheus-exporter:1.1.0
    ports:
      - "9113:9113"
    command:
      - '--nginx.scrape-uri=http://nginx:8080/nginx_status'
    depends_on:
      - nginx
```

## Настройка systemd

```bash
sudo nano /etc/systemd/system/nginx-exporter.service
```

```ini
[Unit]
Description=Nginx Prometheus Exporter
Documentation=https://github.com/nginxinc/nginx-prometheus-exporter
Wants=network-online.target
After=network-online.target nginx.service

[Service]
User=nginx-exporter
Group=nginx-exporter
Type=simple
ExecStart=/usr/local/bin/nginx-prometheus-exporter \
    --nginx.scrape-uri=http://127.0.0.1:8080/nginx_status \
    --web.listen-address=:9113

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# Создаём пользователя
sudo useradd -rs /bin/false nginx-exporter

# Применяем
sudo systemctl daemon-reload
sudo systemctl enable nginx-exporter
sudo systemctl start nginx-exporter

# Проверяем
curl http://localhost:9113/metrics
```

## Флаги командной строки

| Флаг | Описание | Значение по умолчанию |
|------|----------|----------------------|
| `--nginx.scrape-uri` | URI для получения статистики | `http://127.0.0.1:8080/stub_status` |
| `--nginx.plus` | Использовать Nginx Plus API | `false` |
| `--nginx.retries` | Количество повторных попыток | `0` |
| `--nginx.retry-interval` | Интервал между попытками | `5s` |
| `--nginx.timeout` | Таймаут запроса | `5s` |
| `--nginx.ssl-verify` | Проверять SSL сертификат | `true` |
| `--web.listen-address` | Адрес для прослушивания | `:9113` |
| `--web.telemetry-path` | Путь для метрик | `/metrics` |

## Метрики Nginx Open Source (stub_status)

При использовании `stub_status` доступны следующие метрики:

| Метрика | Тип | Описание |
|---------|-----|----------|
| `nginx_connections_accepted` | counter | Общее количество принятых соединений |
| `nginx_connections_active` | gauge | Текущее количество активных соединений |
| `nginx_connections_handled` | counter | Общее количество обработанных соединений |
| `nginx_connections_reading` | gauge | Соединения, читающие заголовок запроса |
| `nginx_connections_waiting` | gauge | Keep-alive соединения в ожидании |
| `nginx_connections_writing` | gauge | Соединения, отправляющие ответ |
| `nginx_http_requests_total` | counter | Общее количество HTTP запросов |
| `nginx_up` | gauge | Доступность Nginx (1 = работает) |

### Примеры PromQL запросов

```promql
# Количество запросов в секунду
rate(nginx_http_requests_total[5m])

# Текущие активные соединения
nginx_connections_active

# Соотношение принятых/обработанных соединений
nginx_connections_handled / nginx_connections_accepted

# Запросов на соединение (keep-alive эффективность)
rate(nginx_http_requests_total[5m]) / rate(nginx_connections_accepted[5m])

# Все метрики Nginx
{__name__=~"nginx_.*"}
```

## Метрики Nginx Plus (расширенные)

Nginx Plus предоставляет значительно больше метрик:

### Upstream метрики

| Метрика | Описание |
|---------|----------|
| `nginxplus_upstream_server_active` | Активные соединения к upstream серверу |
| `nginxplus_upstream_server_requests` | Запросы к upstream серверу |
| `nginxplus_upstream_server_responses` | Ответы от upstream сервера (по кодам) |
| `nginxplus_upstream_server_response_time` | Время ответа upstream сервера |
| `nginxplus_upstream_server_health_checks_fails` | Неуспешные health checks |
| `nginxplus_upstream_server_state` | Состояние сервера (up/down/draining) |

### HTTP Zone метрики

| Метрика | Описание |
|---------|----------|
| `nginxplus_server_zone_requests` | Запросы к server zone |
| `nginxplus_server_zone_responses` | Ответы по кодам (1xx, 2xx, 3xx, 4xx, 5xx) |
| `nginxplus_server_zone_received` | Получено байт |
| `nginxplus_server_zone_sent` | Отправлено байт |

### Примеры PromQL для Nginx Plus

```promql
# Запросы в секунду по upstream серверам
rate(nginxplus_upstream_server_requests[5m])

# Среднее время ответа upstream
rate(nginxplus_upstream_server_response_time_sum[5m]) / rate(nginxplus_upstream_server_requests[5m])

# Процент ошибок 5xx
rate(nginxplus_server_zone_responses{code="5xx"}[5m]) / rate(nginxplus_server_zone_requests[5m]) * 100

# Upstream серверы в состоянии down
nginxplus_upstream_server_state{state="down"}

# Неуспешные health checks
rate(nginxplus_upstream_server_health_checks_fails[5m])
```

## Конфигурация Prometheus

```yaml
# /etc/prometheus/prometheus.yml
scrape_configs:
  - job_name: 'nginx'
    static_configs:
      - targets: ['localhost:9113']
        labels:
          server: 'web-01'

  # Несколько Nginx серверов
  - job_name: 'nginx-cluster'
    static_configs:
      - targets:
          - 'nginx-01:9113'
          - 'nginx-02:9113'
          - 'nginx-03:9113'
        labels:
          environment: 'production'

  # С relabeling
  - job_name: 'nginx-discovery'
    file_sd_configs:
      - files:
          - '/etc/prometheus/targets/nginx.yml'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
```

## Алерты для Nginx

```yaml
# /etc/prometheus/rules/nginx_alerts.yml
groups:
  - name: nginx_alerts
    rules:
      # Nginx недоступен
      - alert: NginxDown
        expr: nginx_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Nginx is down on {{ $labels.instance }}"
          description: "Nginx has been down for more than 1 minute"

      # Высокое количество активных соединений
      - alert: NginxHighConnections
        expr: nginx_connections_active > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High number of connections on {{ $labels.instance }}"
          description: "Active connections: {{ $value }}"

      # Низкая пропускная способность
      - alert: NginxLowRequestRate
        expr: rate(nginx_http_requests_total[5m]) < 1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Low request rate on {{ $labels.instance }}"
          description: "Request rate is below 1 req/s (current: {{ $value }})"

      # Много соединений в состоянии waiting
      - alert: NginxHighWaitingConnections
        expr: nginx_connections_waiting / nginx_connections_active > 0.7
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High waiting connections ratio on {{ $labels.instance }}"
          description: "{{ $value | humanizePercentage }} of connections are waiting"

      # Nginx Plus: Upstream сервер недоступен
      - alert: NginxUpstreamDown
        expr: nginxplus_upstream_server_state{state="unhealthy"} == 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Upstream server {{ $labels.server }} is unhealthy"
          description: "Server in upstream {{ $labels.upstream }} is down"

      # Nginx Plus: Высокое время ответа upstream
      - alert: NginxUpstreamHighLatency
        expr: |
          rate(nginxplus_upstream_server_response_time_sum[5m])
          / rate(nginxplus_upstream_server_requests[5m]) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High upstream latency for {{ $labels.upstream }}"
          description: "Average response time is {{ $value }}s"
```

## Nginx VTS Exporter (альтернатива)

Для более детальных метрик можно использовать модуль **nginx-module-vts** и соответствующий экспортёр.

### Установка модуля VTS

```bash
# Компиляция Nginx с модулем VTS
./configure --add-module=/path/to/nginx-module-vts
make
make install
```

### Конфигурация VTS

```nginx
http {
    vhost_traffic_status_zone;

    server {
        listen 80;

        location /status {
            vhost_traffic_status_display;
            vhost_traffic_status_display_format html;
            allow 127.0.0.1;
            deny all;
        }
    }
}
```

### Метрики VTS

VTS предоставляет метрики по:
- Отдельным виртуальным хостам (server names)
- URI путям
- Upstream группам и серверам
- Кодам ответов
- Cache статусам

## Best Practices

### 1. Безопасность эндпоинта статистики

```nginx
# Ограничиваем доступ по IP
location /nginx_status {
    stub_status;
    allow 127.0.0.1;
    allow 10.0.0.0/8;
    deny all;
}

# Или используем basic auth
location /nginx_status {
    stub_status;
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```

### 2. Мониторинг нескольких инстансов

```yaml
# docker-compose.yml с метками
services:
  nginx-exporter-web:
    image: nginx/nginx-prometheus-exporter
    command: ['--nginx.scrape-uri=http://nginx-web:8080/nginx_status']
    labels:
      - "prometheus.io/scrape=true"
      - "prometheus.io/port=9113"

  nginx-exporter-api:
    image: nginx/nginx-prometheus-exporter
    command: ['--nginx.scrape-uri=http://nginx-api:8080/nginx_status']
    labels:
      - "prometheus.io/scrape=true"
      - "prometheus.io/port=9113"
```

### 3. Граничные значения для алертов

Установите пороги на основе базовых значений вашей системы:

```yaml
# Запишите базовые значения
recording_rules:
  - name: nginx_baseline
    rules:
      - record: nginx:requests_per_second:rate5m
        expr: rate(nginx_http_requests_total[5m])

      - record: nginx:connections_per_second:rate5m
        expr: rate(nginx_connections_accepted[5m])
```

## Типичные ошибки

### 1. stub_status не включён

```bash
# Проверьте, что модуль скомпилирован
nginx -V 2>&1 | grep -o with-http_stub_status_module

# Если нет — переустановите nginx с нужным модулем
```

### 2. Экспортёр не может подключиться к Nginx

```bash
# Проверьте доступность статуса
curl http://127.0.0.1:8080/nginx_status

# Проверьте логи экспортёра
journalctl -u nginx-exporter -f
```

### 3. Неправильный URI для Nginx Plus

```bash
# Для Nginx Plus используйте /api, а не /stub_status
nginx-prometheus-exporter \
  --nginx.plus \
  --nginx.scrape-uri=http://127.0.0.1:8080/api
```

## Grafana Dashboard

Рекомендуемые дашборды для Grafana:

- **Nginx Exporter** (ID: 12708) — для stub_status
- **NGINX Plus Dashboard** (ID: 13397) — для Nginx Plus
- **Nginx VTS Stats** (ID: 2949) — для VTS модуля

Импорт:
1. Откройте Grafana
2. Dashboards → Import
3. Введите ID дашборда
4. Выберите источник данных Prometheus
5. Импортируйте

## Полезные ссылки

- [Официальный репозиторий nginx-prometheus-exporter](https://github.com/nginxinc/nginx-prometheus-exporter)
- [Nginx stub_status документация](https://nginx.org/en/docs/http/ngx_http_stub_status_module.html)
- [Nginx Plus API](https://nginx.org/en/docs/http/ngx_http_api_module.html)
- [nginx-module-vts](https://github.com/vozlt/nginx-module-vts)
