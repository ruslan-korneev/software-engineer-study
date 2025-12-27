# Оптимизация производительности

## Worker Processes

### Базовые настройки

```nginx
# Количество worker процессов
worker_processes auto;  # По числу CPU ядер

# Привязка к ядрам CPU
worker_cpu_affinity auto;

# Приоритет процесса (nice)
worker_priority -5;  # от -20 до 20, меньше = выше приоритет

# Максимум открытых файлов на worker
worker_rlimit_nofile 65535;
```

### Определение оптимального числа workers

```bash
# Количество CPU ядер
nproc

# Или
grep -c processor /proc/cpuinfo
```

**Рекомендации:**
- Для CPU-bound задач: `worker_processes = число ядер`
- Для I/O-bound задач: `worker_processes = число ядер * 2`
- В большинстве случаев: `auto`

## Events

```nginx
events {
    # Максимум соединений на worker
    worker_connections 4096;

    # Принимать все соединения сразу
    multi_accept on;

    # Метод обработки событий
    use epoll;  # Linux
    # use kqueue;  # FreeBSD, macOS

    # Отключить mutex (для multi_accept)
    accept_mutex off;
}
```

### Расчёт максимальных соединений

```
max_clients = worker_processes × worker_connections

# Для reverse proxy (каждое клиентское = 2 соединения):
max_clients = (worker_processes × worker_connections) / 2
```

### Методы обработки событий

| Метод | ОС | Описание |
|-------|-----|----------|
| `epoll` | Linux 2.6+ | Самый эффективный для Linux |
| `kqueue` | FreeBSD, macOS | Самый эффективный для BSD |
| `eventport` | Solaris | Для Solaris |
| `select` | Все | Устаревший, ограничен 1024 соединениями |
| `poll` | Все | Устаревший |

## Keepalive

### Client keepalive

```nginx
http {
    # Таймаут keepalive соединения с клиентом
    keepalive_timeout 65s;

    # Второй параметр - значение в заголовке Keep-Alive
    keepalive_timeout 65s 60s;

    # Максимум запросов на keepalive соединение
    keepalive_requests 1000;
}
```

### Upstream keepalive

```nginx
upstream backend {
    server 127.0.0.1:3000;

    # Количество idle-соединений на worker
    keepalive 32;

    # Максимум запросов через одно соединение
    keepalive_requests 1000;

    # Таймаут idle-соединения
    keepalive_timeout 60s;
}

server {
    location / {
        proxy_pass http://backend;

        # Важно для keepalive к upstream!
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

## Gzip сжатие

```nginx
http {
    # Включить gzip
    gzip on;

    # Минимальный размер для сжатия
    gzip_min_length 256;

    # Уровень сжатия (1-9)
    gzip_comp_level 5;

    # Типы файлов для сжатия
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml
        application/xml+rss
        application/x-javascript
        image/svg+xml;

    # Сжимать для всех прокси
    gzip_proxied any;

    # Добавить Vary: Accept-Encoding
    gzip_vary on;

    # Отключить для IE6
    gzip_disable "msie6";

    # Буферы для сжатия
    gzip_buffers 16 8k;
}
```

### Gzip Static

Раздача предварительно сжатых файлов:

```nginx
http {
    gzip_static on;

    # Nginx будет искать .gz файлы:
    # /path/to/file.js.gz для /path/to/file.js
}
```

```bash
# Предварительное сжатие
gzip -k -9 /var/www/html/static/*.js
gzip -k -9 /var/www/html/static/*.css
```

### Brotli сжатие

```nginx
# Требует модуль ngx_brotli
load_module modules/ngx_http_brotli_filter_module.so;
load_module modules/ngx_http_brotli_static_module.so;

http {
    brotli on;
    brotli_comp_level 6;
    brotli_types text/plain text/css application/json application/javascript;
    brotli_static on;
}
```

## Open File Cache

Кэширование информации о файлах:

```nginx
http {
    # Кэш на 1000 файлов, неактивные удаляются через 20 сек
    open_file_cache max=1000 inactive=20s;

    # Проверять актуальность каждые 30 сек
    open_file_cache_valid 30s;

    # Минимум обращений для кэширования
    open_file_cache_min_uses 2;

    # Кэшировать ошибки (файл не найден)
    open_file_cache_errors on;
}
```

## Буферы

### Client буферы

```nginx
http {
    # Буфер для тела запроса
    client_body_buffer_size 16k;

    # Буфер для заголовков
    client_header_buffer_size 1k;

    # Буферы для больших заголовков
    large_client_header_buffers 4 8k;

    # Максимальный размер тела запроса
    client_max_body_size 100m;
}
```

### Proxy буферы

```nginx
location /api/ {
    proxy_pass http://backend;

    # Буфер для первой части ответа
    proxy_buffer_size 4k;

    # Количество и размер буферов
    proxy_buffers 8 16k;

    # Макс буферов для отправки клиенту
    proxy_busy_buffers_size 32k;

    # Временные файлы
    proxy_temp_file_write_size 64k;
}
```

## Таймауты

```nginx
http {
    # Таймаут на чтение тела запроса
    client_body_timeout 12s;

    # Таймаут на чтение заголовков
    client_header_timeout 12s;

    # Таймаут на отправку ответа
    send_timeout 10s;

    # Keepalive таймаут
    keepalive_timeout 65s;
}
```

### Proxy таймауты

```nginx
location /api/ {
    proxy_pass http://backend;

    proxy_connect_timeout 60s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
}
```

## Sendfile и TCP оптимизации

```nginx
http {
    # Использовать sendfile() для отправки файлов
    sendfile on;

    # Размер чанка для sendfile
    sendfile_max_chunk 512k;

    # Отправлять заголовки и начало файла вместе
    tcp_nopush on;

    # Отключить алгоритм Nagle
    tcp_nodelay on;
}
```

## Системные настройки (Linux)

### Лимиты файловых дескрипторов

```bash
# /etc/security/limits.conf
nginx soft nofile 65535
nginx hard nofile 65535

# Или через systemd
# /etc/systemd/system/nginx.service.d/override.conf
[Service]
LimitNOFILE=65535
```

### Сетевые параметры ядра

```bash
# /etc/sysctl.conf

# Увеличить backlog
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 65535

# TCP буферы
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# TCP оптимизации
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 15

# Быстрое открытие TCP
net.ipv4.tcp_fastopen = 3

# Применить
sysctl -p
```

## Мониторинг производительности

### Метрики nginx

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

### Анализ времени запросов

```nginx
log_format timing '$remote_addr - [$time_local] "$request" '
                  '$status $body_bytes_sent '
                  'rt=$request_time '
                  'uct=$upstream_connect_time '
                  'uht=$upstream_header_time '
                  'urt=$upstream_response_time';

access_log /var/log/nginx/timing.log timing;
```

```bash
# Средне время запроса
awk -F'rt=' '{print $2}' /var/log/nginx/timing.log | awk '{sum+=$1; count++} END {print sum/count}'

# Запросы дольше 1 секунды
awk -F'rt=' '$2 > 1 {print $0}' /var/log/nginx/timing.log
```

## Полная конфигурация для высоких нагрузок

```nginx
# Глобальные настройки
user nginx;
worker_processes auto;
worker_cpu_affinity auto;
worker_rlimit_nofile 65535;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
    accept_mutex off;
}

http {
    # Базовые настройки
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Логирование
    log_format main '$remote_addr - $request_time - "$request" $status';
    access_log /var/log/nginx/access.log main buffer=32k flush=5s;

    # Файловые операции
    sendfile on;
    sendfile_max_chunk 512k;
    tcp_nopush on;
    tcp_nodelay on;

    # Open file cache
    open_file_cache max=10000 inactive=30s;
    open_file_cache_valid 60s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;

    # Keepalive
    keepalive_timeout 65s;
    keepalive_requests 1000;

    # Таймауты
    client_body_timeout 12s;
    client_header_timeout 12s;
    send_timeout 10s;

    # Буферы
    client_body_buffer_size 16k;
    client_header_buffer_size 1k;
    large_client_header_buffers 4 8k;
    client_max_body_size 100m;

    # Gzip
    gzip on;
    gzip_comp_level 5;
    gzip_min_length 256;
    gzip_proxied any;
    gzip_vary on;
    gzip_types text/plain text/css application/json application/javascript
               application/xml application/xml+rss text/javascript image/svg+xml;

    # Upstream
    upstream backend {
        least_conn;
        server 127.0.0.1:3000;
        server 127.0.0.1:3001;
        keepalive 32;
    }

    server {
        listen 80 reuseport;
        server_name example.com;

        location / {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;

            proxy_buffer_size 4k;
            proxy_buffers 8 16k;
            proxy_busy_buffers_size 32k;

            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }

        location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2)$ {
            root /var/www/static;
            expires 30d;
            access_log off;
            add_header Cache-Control "public, immutable";
        }
    }
}
```

## Бенчмаркинг

### wrk

```bash
# Установка
apt install wrk

# Базовый тест
wrk -t12 -c400 -d30s http://localhost/

# С keepalive
wrk -t12 -c400 -d30s -H "Connection: keep-alive" http://localhost/
```

### ab (Apache Bench)

```bash
# 10000 запросов, 100 одновременных
ab -n 10000 -c 100 http://localhost/

# С keepalive
ab -n 10000 -c 100 -k http://localhost/
```

### siege

```bash
# 100 пользователей на 1 минуту
siege -c100 -t1M http://localhost/
```
