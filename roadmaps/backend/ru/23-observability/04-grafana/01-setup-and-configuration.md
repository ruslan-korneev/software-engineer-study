# Установка и настройка Grafana

## Что такое Grafana?

**Grafana** — это open-source платформа для визуализации, мониторинга и анализа данных. Она позволяет создавать интерактивные дашборды для отображения метрик из различных источников данных (Prometheus, InfluxDB, Elasticsearch, PostgreSQL и многих других).

### Основные возможности Grafana

- **Унифицированный интерфейс** — один инструмент для работы с множеством источников данных
- **Гибкая визуализация** — богатый набор панелей: графики, таблицы, heatmaps, gauges
- **Алертинг** — встроенная система оповещений
- **Темплейты и переменные** — динамические дашборды
- **Аннотации** — маркировка событий на графиках
- **Плагины** — расширяемая архитектура

---

## Способы установки Grafana

### 1. Установка через Docker (рекомендуется для разработки)

```bash
# Запуск Grafana с сохранением данных
docker run -d \
  --name grafana \
  -p 3000:3000 \
  -v grafana-storage:/var/lib/grafana \
  grafana/grafana:latest

# Или с кастомными переменными окружения
docker run -d \
  --name grafana \
  -p 3000:3000 \
  -e "GF_SECURITY_ADMIN_USER=admin" \
  -e "GF_SECURITY_ADMIN_PASSWORD=secure_password" \
  -e "GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-piechart-panel" \
  -v grafana-storage:/var/lib/grafana \
  grafana/grafana:latest
```

### 2. Docker Compose (для стека мониторинга)

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
      - GF_SERVER_ROOT_URL=http://localhost:3000
      - GF_SMTP_ENABLED=true
      - GF_SMTP_HOST=smtp.gmail.com:587
    volumes:
      - grafana-data:/var/lib/grafana
      - ./provisioning:/etc/grafana/provisioning
    networks:
      - monitoring

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'
    networks:
      - monitoring

volumes:
  grafana-data:
  prometheus-data:

networks:
  monitoring:
    driver: bridge
```

### 3. Установка на Linux (Ubuntu/Debian)

```bash
# Добавление репозитория
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

# Установка
sudo apt-get update
sudo apt-get install grafana

# Запуск и автозапуск
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server

# Проверка статуса
sudo systemctl status grafana-server
```

### 4. Установка в Kubernetes (Helm)

```bash
# Добавление репозитория
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Установка с кастомными значениями
helm install grafana grafana/grafana \
  --namespace monitoring \
  --create-namespace \
  --set persistence.enabled=true \
  --set persistence.size=10Gi \
  --set adminPassword='secure_password'

# Получение пароля admin (если не задан)
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

---

## Конфигурация Grafana

### Файл конфигурации grafana.ini

Основной конфигурационный файл находится в `/etc/grafana/grafana.ini`:

```ini
# /etc/grafana/grafana.ini

##################### Основные настройки #####################
[paths]
# Путь к данным
data = /var/lib/grafana
# Путь к логам
logs = /var/log/grafana
# Путь к плагинам
plugins = /var/lib/grafana/plugins
# Путь к провизионированию
provisioning = /etc/grafana/provisioning

##################### Настройки сервера #####################
[server]
# Протокол (http, https, socket)
protocol = http
# HTTP порт
http_port = 3000
# Домен для ссылок
domain = localhost
# Корневой URL
root_url = %(protocol)s://%(domain)s:%(http_port)s/
# Использовать подпуть (для reverse proxy)
serve_from_sub_path = false

##################### База данных #####################
[database]
# Тип БД: sqlite3, mysql, postgres
type = sqlite3
# Для PostgreSQL
# type = postgres
# host = localhost:5432
# name = grafana
# user = grafana
# password = password
# ssl_mode = disable

##################### Безопасность #####################
[security]
# Администратор по умолчанию
admin_user = admin
# Пароль администратора (изменить после первого входа!)
admin_password = admin
# Секретный ключ для подписи cookies
secret_key = SW2YcwTIb9zpOOhoPsMm
# Отключить brute-force защиту (не рекомендуется в production)
disable_brute_force_login_protection = false

##################### Пользователи #####################
[users]
# Разрешить регистрацию
allow_sign_up = false
# Разрешить создание организаций
allow_org_create = false
# Автоматически назначать новых пользователей в организацию
auto_assign_org = true
auto_assign_org_id = 1
auto_assign_org_role = Viewer

##################### Аутентификация #####################
[auth]
# Отключить страницу логина (для embed)
disable_login_form = false
# Разрешить анонимный доступ
[auth.anonymous]
enabled = false
org_name = Main Org.
org_role = Viewer

# OAuth с GitHub
[auth.github]
enabled = false
allow_sign_up = true
client_id = your_client_id
client_secret = your_client_secret
scopes = user:email,read:org
auth_url = https://github.com/login/oauth/authorize
token_url = https://github.com/login/oauth/access_token
api_url = https://api.github.com/user
allowed_organizations = your_org

# LDAP аутентификация
[auth.ldap]
enabled = false
config_file = /etc/grafana/ldap.toml

##################### SMTP / Email #####################
[smtp]
enabled = true
host = smtp.gmail.com:587
user = your_email@gmail.com
password = your_app_password
from_address = grafana@yourdomain.com
from_name = Grafana
startTLS_policy = MandatoryStartTLS

##################### Логирование #####################
[log]
# Уровень логирования: debug, info, warn, error, critical
level = info
# Режим логирования: console, file, syslog
mode = console file
# Формат: console, text, json
filters =

[log.console]
level =
format = console

[log.file]
level =
format = text
log_rotate = true
max_lines = 1000000
max_size_shift = 28
daily_rotate = true
max_days = 7

##################### Alerting #####################
[alerting]
enabled = true
execute_alerts = true

##################### Unified Alerting (Grafana 8+) #####################
[unified_alerting]
enabled = true

##################### Дашборды #####################
[dashboards]
# Минимальный интервал обновления
min_refresh_interval = 5s
# Версии дашбордов
versions_to_keep = 20
```

### Конфигурация через переменные окружения

Все параметры можно переопределить через переменные окружения с префиксом `GF_`:

```bash
# Формат: GF_<SECTION>_<PARAMETER>
# Пример конфигурации через ENV
export GF_SECURITY_ADMIN_USER=admin
export GF_SECURITY_ADMIN_PASSWORD=secure_password
export GF_SERVER_ROOT_URL=https://grafana.mycompany.com
export GF_DATABASE_TYPE=postgres
export GF_DATABASE_HOST=postgres:5432
export GF_DATABASE_NAME=grafana
export GF_DATABASE_USER=grafana
export GF_DATABASE_PASSWORD=db_password
export GF_SMTP_ENABLED=true
export GF_SMTP_HOST=smtp.gmail.com:587
export GF_USERS_ALLOW_SIGN_UP=false
export GF_AUTH_ANONYMOUS_ENABLED=false
export GF_INSTALL_PLUGINS=grafana-piechart-panel,grafana-clock-panel
```

---

## Настройка источников данных (Data Sources)

### Prometheus

```yaml
# provisioning/datasources/prometheus.yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
    jsonData:
      httpMethod: POST
      manageAlerts: true
      prometheusType: Prometheus
      prometheusVersion: 2.40.0
      timeInterval: 15s
      exemplarTraceIdDestinations:
        - name: traceID
          datasourceUid: tempo
```

### InfluxDB

```yaml
# provisioning/datasources/influxdb.yaml
apiVersion: 1

datasources:
  - name: InfluxDB
    type: influxdb
    access: proxy
    url: http://influxdb:8086
    jsonData:
      version: Flux
      organization: myorg
      defaultBucket: mybucket
    secureJsonData:
      token: your_influxdb_token
```

### PostgreSQL

```yaml
# provisioning/datasources/postgres.yaml
apiVersion: 1

datasources:
  - name: PostgreSQL
    type: postgres
    url: postgres:5432
    database: mydb
    user: grafana_reader
    jsonData:
      sslmode: disable
      maxOpenConns: 10
      maxIdleConns: 5
      connMaxLifetime: 14400
      postgresVersion: 1500
      timescaledb: false
    secureJsonData:
      password: reader_password
```

### Loki (для логов)

```yaml
# provisioning/datasources/loki.yaml
apiVersion: 1

datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    jsonData:
      maxLines: 1000
      derivedFields:
        - datasourceUid: tempo
          matcherRegex: "traceID=(\\w+)"
          name: TraceID
          url: '$${__value.raw}'
```

---

## Управление плагинами

### Установка плагинов

```bash
# CLI установка
grafana-cli plugins install grafana-piechart-panel
grafana-cli plugins install grafana-clock-panel
grafana-cli plugins install grafana-worldmap-panel
grafana-cli plugins install grafana-polystat-panel

# Список установленных плагинов
grafana-cli plugins ls

# Обновление плагинов
grafana-cli plugins update-all

# Удаление плагина
grafana-cli plugins remove grafana-piechart-panel
```

### Установка через Docker

```dockerfile
# Dockerfile
FROM grafana/grafana:latest

# Установка плагинов при сборке
RUN grafana-cli plugins install grafana-piechart-panel && \
    grafana-cli plugins install grafana-clock-panel && \
    grafana-cli plugins install grafana-polystat-panel
```

Или через переменную окружения:

```yaml
# docker-compose.yml
services:
  grafana:
    image: grafana/grafana:latest
    environment:
      - GF_INSTALL_PLUGINS=grafana-piechart-panel,grafana-clock-panel,grafana-worldmap-panel
```

---

## Best Practices

### 1. Безопасность

- **Изменить пароль admin** сразу после установки
- **Использовать HTTPS** в production (через reverse proxy)
- **Отключить регистрацию** (`allow_sign_up = false`)
- **Настроить LDAP/OAuth** для корпоративной аутентификации
- **Ограничить анонимный доступ**

### 2. Production настройки

- **Использовать PostgreSQL/MySQL** вместо SQLite для хранения данных
- **Настроить backup** базы данных и `/var/lib/grafana`
- **Включить TLS** для datasources
- **Настроить rate limiting** для API
- **Использовать provisioning** для автоматизации

### 3. Производительность

- **Настроить caching** для datasources
- **Ограничить время запросов** (`dataproxy.timeout`)
- **Использовать recording rules** в Prometheus для тяжелых запросов
- **Настроить min_refresh_interval** для предотвращения частых обновлений

---

## Типичные ошибки

1. **Забыли изменить пароль admin** — безопасность под угрозой
2. **SQLite в production** — не масштабируется, нет отказоустойчивости
3. **Неправильный URL datasource** — использовать внутренние DNS имена в Docker/K8s
4. **Отсутствие backup** — потеря дашбордов при сбое
5. **Слишком частый refresh дашбордов** — нагрузка на datasources

---

## Полезные ссылки

- [Официальная документация Grafana](https://grafana.com/docs/grafana/latest/)
- [Grafana на GitHub](https://github.com/grafana/grafana)
- [Каталог плагинов](https://grafana.com/grafana/plugins/)
- [Grafana Community](https://community.grafana.com/)
