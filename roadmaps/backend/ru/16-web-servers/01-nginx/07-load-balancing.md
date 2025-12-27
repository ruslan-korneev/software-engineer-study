# Балансировка нагрузки

## Upstream блоки

Upstream определяет группу серверов для балансировки:

```nginx
upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
    }
}
```

## Методы балансировки

### Round Robin (по умолчанию)

Запросы распределяются по очереди между серверами:

```nginx
upstream backend {
    # Round-robin по умолчанию
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}
# Запросы: 1→A, 2→B, 3→C, 4→A, 5→B, 6→C...
```

### Weighted Round Robin

Серверы с большим весом получают больше запросов:

```nginx
upstream backend {
    server 192.168.1.10:8080 weight=5;  # 50% запросов
    server 192.168.1.11:8080 weight=3;  # 30% запросов
    server 192.168.1.12:8080 weight=2;  # 20% запросов
}
```

### Least Connections

Запрос направляется к серверу с наименьшим числом активных соединений:

```nginx
upstream backend {
    least_conn;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}
```

С весами:

```nginx
upstream backend {
    least_conn;
    server 192.168.1.10:8080 weight=5;
    server 192.168.1.11:8080 weight=3;
}
# Учитывается отношение connections/weight
```

### IP Hash

Один и тот же клиент всегда попадает на один сервер (session persistence):

```nginx
upstream backend {
    ip_hash;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}
```

**Особенности:**
- Хэшируются первые три октета IPv4 (или весь IPv6)
- Если сервер недоступен, клиент перенаправляется на другой

### Hash (произвольный ключ)

Балансировка по произвольному ключу:

```nginx
upstream backend {
    # По URI
    hash $request_uri;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

upstream backend {
    # По cookie
    hash $cookie_sessionid;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

upstream backend {
    # Consistent hashing (минимизирует перераспределение)
    hash $request_uri consistent;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}
```

### Random

Случайное распределение:

```nginx
upstream backend {
    random;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

# Two-random choices (выбор лучшего из двух случайных)
upstream backend {
    random two least_conn;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}
```

## Параметры серверов

```nginx
upstream backend {
    server 192.168.1.10:8080 weight=5;
    server 192.168.1.11:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.12:8080 backup;
    server 192.168.1.13:8080 down;
}
```

| Параметр | Описание | По умолчанию |
|----------|----------|--------------|
| `weight=n` | Вес сервера | 1 |
| `max_fails=n` | Макс. неудачных попыток | 1 |
| `fail_timeout=time` | Время "карантина" + период подсчёта ошибок | 10s |
| `backup` | Резервный сервер (только если основные недоступны) | - |
| `down` | Сервер временно отключён | - |
| `max_conns=n` | Макс. одновременных соединений | 0 (без лимита) |
| `slow_start=time` | Постепенное увеличение нагрузки (Nginx Plus) | - |

### Пример с полными настройками

```nginx
upstream backend {
    least_conn;

    # Основные серверы
    server 192.168.1.10:8080 weight=5 max_fails=3 fail_timeout=30s max_conns=100;
    server 192.168.1.11:8080 weight=3 max_fails=3 fail_timeout=30s max_conns=100;

    # Резервный
    server 192.168.1.12:8080 backup;

    # Временно отключён (на обслуживании)
    server 192.168.1.13:8080 down;
}
```

## Health Checks

### Passive Health Checks (Open Source)

Nginx отслеживает ошибки при обычных запросах:

```nginx
upstream backend {
    server 192.168.1.10:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.11:8080 max_fails=3 fail_timeout=30s;
}

server {
    location / {
        proxy_pass http://backend;

        # Что считать ошибкой для переключения на другой сервер
        proxy_next_upstream error timeout http_500 http_502 http_503 http_504;

        # Макс. количество попыток
        proxy_next_upstream_tries 3;

        # Таймаут на все попытки
        proxy_next_upstream_timeout 10s;
    }
}
```

### Active Health Checks (Nginx Plus)

```nginx
# Только в Nginx Plus
upstream backend {
    zone backend 64k;

    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

server {
    location / {
        proxy_pass http://backend;
        health_check interval=5s fails=3 passes=2;
    }
}
```

### Самодельный health check через cron

```bash
#!/bin/bash
# /usr/local/bin/nginx-healthcheck.sh

SERVERS="192.168.1.10 192.168.1.11 192.168.1.12"
PORT=8080
CONFIG="/etc/nginx/conf.d/upstream.conf"

for server in $SERVERS; do
    if curl -sf "http://${server}:${PORT}/health" > /dev/null; then
        # Сервер здоров - убрать down если есть
        sed -i "s/server ${server}:${PORT} down;/server ${server}:${PORT};/" $CONFIG
    else
        # Сервер недоступен - пометить down
        sed -i "s/server ${server}:${PORT};/server ${server}:${PORT} down;/" $CONFIG
    fi
done

nginx -s reload
```

## Keepalive к upstream

Поддержка постоянных соединений к backend:

```nginx
upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;

    # Количество idle-соединений на worker
    keepalive 32;

    # Макс. запросов через одно соединение
    keepalive_requests 100;

    # Таймаут idle-соединения
    keepalive_timeout 60s;
}

server {
    location / {
        proxy_pass http://backend;

        # Важно для keepalive!
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

## Session Persistence

### Через IP Hash

```nginx
upstream backend {
    ip_hash;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}
```

### Через Cookie Hash

```nginx
upstream backend {
    hash $cookie_JSESSIONID consistent;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}
```

### Sticky Cookie (Nginx Plus)

```nginx
# Только в Nginx Plus
upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    sticky cookie srv_id expires=1h domain=.example.com path=/;
}
```

## Зоны для shared memory

Для корректной работы в multi-worker:

```nginx
upstream backend {
    zone backend 64k;  # Shared memory zone

    least_conn;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}
```

## Динамическое управление (Nginx Plus)

```nginx
# Только в Nginx Plus
upstream backend {
    zone backend 64k;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

server {
    location /api {
        api write=on;
    }
}
```

```bash
# Добавить сервер
curl -X POST "http://localhost/api/http/upstreams/backend/servers" \
    -d '{"server": "192.168.1.12:8080", "weight": 1}'

# Удалить сервер
curl -X DELETE "http://localhost/api/http/upstreams/backend/servers/2"
```

## Примеры конфигураций

### Базовая балансировка

```nginx
upstream app {
    least_conn;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
    keepalive 32;
}

server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://app;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Blue-Green Deployment

```nginx
upstream blue {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

upstream green {
    server 192.168.1.12:8080;
    server 192.168.1.13:8080;
}

# Переключение через map
map $cookie_deployment $backend {
    "green" green;
    default blue;
}

server {
    location / {
        proxy_pass http://$backend;
    }
}
```

### Canary Deployment

```nginx
# 10% трафика на новую версию
upstream stable {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

upstream canary {
    server 192.168.1.12:8080;
}

split_clients $request_id $backend {
    10% canary;
    *   stable;
}

server {
    location / {
        proxy_pass http://$backend;
    }
}
```

### Разные upstream для разных путей

```nginx
upstream api {
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
}

upstream web {
    server 127.0.0.1:4000;
    server 127.0.0.1:4001;
}

upstream static {
    server 127.0.0.1:5000;
}

server {
    listen 80;

    location /api/ {
        proxy_pass http://api;
    }

    location /static/ {
        proxy_pass http://static;
    }

    location / {
        proxy_pass http://web;
    }
}
```

## Мониторинг upstream

### stub_status (базовая статистика)

```nginx
server {
    listen 8080;

    location /nginx_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }
}
```

```bash
curl http://localhost:8080/nginx_status
# Active connections: 291
# server accepts handled requests
#  16630948 16630948 31070465
# Reading: 6 Writing: 179 Waiting: 106
```

### Prometheus экспортер

Использование nginx-prometheus-exporter:

```bash
docker run -p 9113:9113 nginx/nginx-prometheus-exporter:latest \
    -nginx.scrape-uri=http://nginx:8080/nginx_status
```
