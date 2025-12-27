# Node Exporter

## Введение

**Node Exporter** — это официальный экспортёр Prometheus для сбора метрик аппаратного обеспечения и операционной системы на Unix-подобных системах. Он предоставляет детальную информацию о состоянии сервера: использование CPU, памяти, дискового пространства, сетевых интерфейсов и многого другого.

Node Exporter является одним из самых популярных экспортёров в экосистеме Prometheus и используется практически в каждой инфраструктуре, где применяется мониторинг на базе Prometheus.

## Архитектура

Node Exporter работает как демон на каждом сервере, который нужно мониторить. Он:

1. Читает данные из файловой системы `/proc` и `/sys`
2. Преобразует их в формат метрик Prometheus
3. Предоставляет HTTP-эндпоинт `/metrics` для Prometheus

```
┌─────────────────────────────────────────────────────────────┐
│                         Server                               │
│  ┌──────────────┐     ┌─────────────────┐                   │
│  │  /proc, /sys │────▶│  Node Exporter  │──▶ :9100/metrics  │
│  └──────────────┘     └─────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
                                │
                                ▼
                    ┌─────────────────────┐
                    │     Prometheus      │
                    │   (scrapes :9100)   │
                    └─────────────────────┘
```

## Установка

### Способ 1: Скачивание бинарного файла

```bash
# Скачиваем последнюю версию
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz

# Распаковываем
tar xvfz node_exporter-1.7.0.linux-amd64.tar.gz

# Перемещаем бинарник
sudo mv node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/

# Проверяем
node_exporter --version
```

### Способ 2: Через пакетный менеджер (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install prometheus-node-exporter
```

### Способ 3: Docker

```bash
docker run -d \
  --name node_exporter \
  --net="host" \
  --pid="host" \
  -v "/:/host:ro,rslave" \
  quay.io/prometheus/node-exporter:latest \
  --path.rootfs=/host
```

### Способ 4: Docker Compose

```yaml
version: '3.8'

services:
  node-exporter:
    image: quay.io/prometheus/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    network_mode: host
    pid: host
    volumes:
      - /:/host:ro,rslave
    command:
      - '--path.rootfs=/host'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
```

## Настройка systemd

Создайте файл сервиса:

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Node Exporter
Documentation=https://prometheus.io/docs/guides/node-exporter/
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
    --collector.systemd \
    --collector.processes \
    --web.listen-address=:9100

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# Создаём пользователя
sudo useradd -rs /bin/false node_exporter

# Применяем конфигурацию
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter

# Проверяем статус
sudo systemctl status node_exporter
```

## Основные коллекторы (Collectors)

Node Exporter использует систему коллекторов для сбора различных типов метрик. По умолчанию включены наиболее полезные коллекторы.

### Включённые по умолчанию

| Коллектор | Описание |
|-----------|----------|
| `cpu` | Статистика CPU |
| `diskstats` | Статистика дисковых устройств |
| `filesystem` | Статистика файловых систем |
| `loadavg` | Средняя нагрузка системы |
| `meminfo` | Статистика памяти |
| `netdev` | Статистика сетевых интерфейсов |
| `netstat` | Сетевая статистика из /proc/net/netstat |
| `uname` | Информация о системе (ядро, ОС) |
| `vmstat` | Статистика виртуальной памяти |

### Отключённые по умолчанию (требуют явного включения)

| Коллектор | Описание | Флаг |
|-----------|----------|------|
| `systemd` | Состояние systemd сервисов | `--collector.systemd` |
| `processes` | Статистика процессов | `--collector.processes` |
| `textfile` | Пользовательские метрики | `--collector.textfile` |
| `wifi` | Информация о WiFi | `--collector.wifi` |

## Основные метрики

### CPU

```promql
# Использование CPU в процентах (по ядрам)
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Использование CPU по режимам
rate(node_cpu_seconds_total[5m])
```

### Память

```promql
# Доступная память в процентах
(node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Использованная память
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

# Использование swap
(1 - (node_memory_SwapFree_bytes / node_memory_SwapTotal_bytes)) * 100
```

### Диск

```promql
# Использование диска в процентах
(1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100

# Свободное место на диске
node_filesystem_avail_bytes

# Скорость чтения/записи
rate(node_disk_read_bytes_total[5m])
rate(node_disk_written_bytes_total[5m])

# IOPS
rate(node_disk_reads_completed_total[5m])
rate(node_disk_writes_completed_total[5m])
```

### Сеть

```promql
# Входящий трафик (bytes/sec)
rate(node_network_receive_bytes_total[5m])

# Исходящий трафик (bytes/sec)
rate(node_network_transmit_bytes_total[5m])

# Ошибки сетевых интерфейсов
rate(node_network_receive_errs_total[5m])
rate(node_network_transmit_errs_total[5m])
```

### Нагрузка системы

```promql
# Load Average (1, 5, 15 минут)
node_load1
node_load5
node_load15

# Нормализованный load average (относительно количества CPU)
node_load1 / count by(instance) (node_cpu_seconds_total{mode="idle"})
```

## Конфигурация Prometheus

Добавьте job в `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']

  # Для нескольких серверов
  - job_name: 'node-cluster'
    static_configs:
      - targets:
          - 'server1:9100'
          - 'server2:9100'
          - 'server3:9100'
        labels:
          env: 'production'

  # С Service Discovery (file-based)
  - job_name: 'node-discovery'
    file_sd_configs:
      - files:
          - '/etc/prometheus/targets/nodes.yml'
        refresh_interval: 30s
```

Пример файла `nodes.yml`:

```yaml
- targets:
    - 'web-01:9100'
    - 'web-02:9100'
  labels:
    role: 'web'
    env: 'production'

- targets:
    - 'db-01:9100'
    - 'db-02:9100'
  labels:
    role: 'database'
    env: 'production'
```

## Textfile Collector

Textfile collector позволяет добавлять собственные метрики через файлы.

### Настройка

```bash
# Создаём директорию для метрик
sudo mkdir -p /var/lib/node_exporter/textfile_collector

# Запускаем с collector
node_exporter --collector.textfile.directory=/var/lib/node_exporter/textfile_collector
```

### Пример пользовательской метрики

```bash
# Создаём скрипт для сбора метрик
cat > /usr/local/bin/backup_metrics.sh << 'EOF'
#!/bin/bash
METRICS_DIR=/var/lib/node_exporter/textfile_collector
BACKUP_LOG=/var/log/backup/last_backup.log

# Время последнего успешного бэкапа
if [ -f "$BACKUP_LOG" ]; then
    LAST_BACKUP=$(stat -c %Y "$BACKUP_LOG")
else
    LAST_BACKUP=0
fi

cat > "${METRICS_DIR}/backup.prom.$$" << METRICS
# HELP backup_last_success_timestamp_seconds Unix timestamp of last successful backup
# TYPE backup_last_success_timestamp_seconds gauge
backup_last_success_timestamp_seconds $LAST_BACKUP
METRICS

mv "${METRICS_DIR}/backup.prom.$$" "${METRICS_DIR}/backup.prom"
EOF

chmod +x /usr/local/bin/backup_metrics.sh

# Добавляем в cron
echo "*/5 * * * * root /usr/local/bin/backup_metrics.sh" >> /etc/crontab
```

## Алерты для Node Exporter

Создайте файл правил алертинга:

```yaml
# /etc/prometheus/rules/node_alerts.yml
groups:
  - name: node_alerts
    rules:
      # Высокое использование CPU
      - alert: HighCpuUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is above 80% (current: {{ $value }}%)"

      # Высокое использование памяти
      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is above 90% (current: {{ $value }}%)"

      # Мало места на диске
      - alert: DiskSpaceLow
        expr: (1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk space low on {{ $labels.instance }}"
          description: "Disk usage on {{ $labels.mountpoint }} is above 85%"

      # Сервер недоступен
      - alert: NodeDown
        expr: up{job="node"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Node {{ $labels.instance }} is down"
          description: "Node Exporter is not responding"

      # Высокий Load Average
      - alert: HighLoadAverage
        expr: node_load15 / count by(instance) (node_cpu_seconds_total{mode="idle"}) > 2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High load average on {{ $labels.instance }}"
          description: "Load average is very high (current: {{ $value }})"
```

## Best Practices

### 1. Безопасность

```bash
# Запускайте от непривилегированного пользователя
sudo useradd -rs /bin/false node_exporter

# Ограничьте доступ через firewall
sudo ufw allow from 10.0.0.0/8 to any port 9100

# Или используйте TLS
node_exporter \
  --web.config.file=/etc/node_exporter/web-config.yml
```

Пример `web-config.yml`:

```yaml
tls_server_config:
  cert_file: /etc/node_exporter/cert.pem
  key_file: /etc/node_exporter/key.pem
basic_auth_users:
  prometheus: $2y$10$...  # bcrypt hash пароля
```

### 2. Оптимизация сбора метрик

```bash
# Отключите ненужные коллекторы для снижения нагрузки
node_exporter \
  --no-collector.arp \
  --no-collector.bcache \
  --no-collector.bonding \
  --no-collector.infiniband \
  --no-collector.ipvs \
  --no-collector.nfs \
  --no-collector.nfsd \
  --no-collector.zfs
```

### 3. Фильтрация файловых систем

```bash
# Исключаем виртуальные и временные ФС
node_exporter \
  --collector.filesystem.mount-points-exclude='^/(sys|proc|dev|run|snap)($|/)' \
  --collector.filesystem.fs-types-exclude='^(tmpfs|autofs|binfmt_misc|cgroup|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|mqueue|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|sysfs|tracefs)$'
```

## Типичные ошибки

### 1. Неправильные права доступа в Docker

```bash
# НЕПРАВИЛЬНО - нет доступа к /proc хоста
docker run -d quay.io/prometheus/node-exporter

# ПРАВИЛЬНО - монтируем корневую ФС
docker run -d \
  --net="host" \
  --pid="host" \
  -v "/:/host:ro,rslave" \
  quay.io/prometheus/node-exporter \
  --path.rootfs=/host
```

### 2. Не включён нужный коллектор

```bash
# Если нужны метрики systemd, но их нет:
node_exporter --collector.systemd
```

### 3. Prometheus не может подключиться

```bash
# Проверьте, что порт открыт
curl http://localhost:9100/metrics

# Проверьте firewall
sudo ufw status
sudo iptables -L -n
```

## Полезные ссылки

- [Официальная документация](https://prometheus.io/docs/guides/node-exporter/)
- [GitHub репозиторий](https://github.com/prometheus/node_exporter)
- [Awesome Prometheus Alerts](https://awesome-prometheus-alerts.grep.to/rules#host-and-hardware)
