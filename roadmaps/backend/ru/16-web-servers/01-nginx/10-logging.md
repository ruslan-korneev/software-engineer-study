# Логирование и мониторинг

## Access Log

### Базовая настройка

```nginx
http {
    # Формат по умолчанию (combined)
    access_log /var/log/nginx/access.log;

    server {
        # Свой лог для сервера
        access_log /var/log/nginx/example.com.access.log;

        location /api/ {
            # Свой лог для location
            access_log /var/log/nginx/api.access.log;
        }
    }
}
```

### Отключение логирования

```nginx
# Для статики
location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
    access_log off;
}

# Для health checks
location = /health {
    access_log off;
    return 200 "OK";
}
```

## Log Format

### Встроенные форматы

```nginx
# combined (по умолчанию)
log_format combined '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';
```

### Кастомные форматы

```nginx
http {
    # Простой формат
    log_format simple '$remote_addr - $request - $status';

    # Расширенный с временем
    log_format detailed '$remote_addr - $remote_user [$time_local] '
                        '"$request" $status $body_bytes_sent '
                        '"$http_referer" "$http_user_agent" '
                        'rt=$request_time uct=$upstream_connect_time '
                        'uht=$upstream_header_time urt=$upstream_response_time';

    # Для отладки прокси
    log_format proxy '$remote_addr - [$time_local] "$request" '
                     '$status $body_bytes_sent '
                     'upstream: $upstream_addr '
                     'cache: $upstream_cache_status '
                     'time: $request_time';

    server {
        access_log /var/log/nginx/access.log detailed;
    }
}
```

### JSON формат

```nginx
log_format json escape=json '{'
    '"time": "$time_iso8601",'
    '"remote_addr": "$remote_addr",'
    '"remote_user": "$remote_user",'
    '"request": "$request",'
    '"status": $status,'
    '"body_bytes_sent": $body_bytes_sent,'
    '"request_time": $request_time,'
    '"http_referrer": "$http_referer",'
    '"http_user_agent": "$http_user_agent",'
    '"http_x_forwarded_for": "$http_x_forwarded_for",'
    '"upstream_addr": "$upstream_addr",'
    '"upstream_response_time": "$upstream_response_time",'
    '"upstream_cache_status": "$upstream_cache_status"'
'}';

access_log /var/log/nginx/access.json json;
```

### Полезные переменные для логов

| Переменная | Описание |
|------------|----------|
| `$remote_addr` | IP клиента |
| `$remote_user` | Пользователь (Basic Auth) |
| `$time_local` | Локальное время |
| `$time_iso8601` | Время в ISO 8601 |
| `$request` | Полный запрос (метод + URI + протокол) |
| `$request_method` | GET, POST, etc. |
| `$request_uri` | URI с query string |
| `$status` | HTTP код ответа |
| `$body_bytes_sent` | Размер тела ответа |
| `$bytes_sent` | Всего отправлено байт |
| `$request_time` | Время обработки (в секундах) |
| `$request_length` | Размер запроса |
| `$http_referer` | Referer заголовок |
| `$http_user_agent` | User-Agent |
| `$upstream_addr` | Адрес upstream |
| `$upstream_response_time` | Время ответа upstream |
| `$upstream_cache_status` | Статус кэша |
| `$ssl_protocol` | TLS версия |
| `$ssl_cipher` | Используемый шифр |

## Условное логирование

```nginx
# Не логировать успешные запросы
map $status $loggable {
    ~^[23]  0;
    default 1;
}

access_log /var/log/nginx/error-only.log combined if=$loggable;

# Не логировать ботов
map $http_user_agent $is_bot {
    ~*bot 1;
    ~*crawler 1;
    ~*spider 1;
    default 0;
}

map $is_bot $log_ua {
    1 0;
    default 1;
}

access_log /var/log/nginx/access.log combined if=$log_ua;
```

## Error Log

### Уровни логирования

```nginx
# Синтаксис: error_log path [level];
error_log /var/log/nginx/error.log warn;
```

| Уровень | Описание |
|---------|----------|
| `debug` | Отладочная информация (требует сборки с `--with-debug`) |
| `info` | Информационные сообщения |
| `notice` | Важные события |
| `warn` | Предупреждения (по умолчанию) |
| `error` | Ошибки обработки |
| `crit` | Критические ошибки |
| `alert` | Требуется немедленное действие |
| `emerg` | Система неработоспособна |

### Раздельные логи ошибок

```nginx
http {
    error_log /var/log/nginx/error.log warn;

    server {
        server_name example.com;
        error_log /var/log/nginx/example.com.error.log;

        location /api/ {
            error_log /var/log/nginx/api.error.log debug;
        }
    }
}
```

### Debug лог для конкретных клиентов

```nginx
events {
    debug_connection 192.168.1.100;
    debug_connection 192.168.1.0/24;
}
```

## Буферизация логов

```nginx
http {
    # Буфер 32KB, сброс каждые 5 секунд или при переполнении
    access_log /var/log/nginx/access.log combined buffer=32k flush=5s;

    # Gzip сжатие (требует gzip=on)
    access_log /var/log/nginx/access.log.gz combined gzip=9 buffer=64k;
}
```

## Syslog

```nginx
# Отправка в syslog
access_log syslog:server=192.168.1.10:514,facility=local7,tag=nginx,severity=info combined;
error_log syslog:server=unix:/dev/log,facility=local7,tag=nginx_error;

# Несколько destinations
access_log /var/log/nginx/access.log combined;
access_log syslog:server=logserver:514 combined;
```

### Параметры syslog

| Параметр | Описание |
|----------|----------|
| `server=addr` | Адрес syslog сервера (IP:port или unix socket) |
| `facility=` | Facility (local0-local7, auth, daemon, etc.) |
| `tag=` | Тег сообщения |
| `severity=` | Severity (debug, info, notice, warn, error, crit) |
| `nohostname` | Не добавлять hostname |

## Ротация логов

### Logrotate

```bash
# /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}
```

### Сигнал для переоткрытия логов

```bash
# После ротации
nginx -s reopen

# Или
kill -USR1 $(cat /var/run/nginx.pid)
```

## Stub Status

Базовая статистика nginx:

```nginx
server {
    listen 8080;

    location /nginx_status {
        stub_status;
        allow 127.0.0.1;
        allow 192.168.1.0/24;
        deny all;
    }
}
```

```bash
curl http://localhost:8080/nginx_status

# Вывод:
# Active connections: 291
# server accepts handled requests
#  16630948 16630948 31070465
# Reading: 6 Writing: 179 Waiting: 106
```

| Метрика | Описание |
|---------|----------|
| Active connections | Текущие активные соединения |
| accepts | Всего принятых соединений |
| handled | Всего обработанных соединений |
| requests | Всего запросов |
| Reading | Соединений, читающих заголовки |
| Writing | Соединений, отправляющих ответ |
| Waiting | Keep-alive соединений в ожидании |

## Интеграция с системами мониторинга

### Prometheus + nginx-prometheus-exporter

```nginx
# Включить stub_status
server {
    listen 8080;
    location /nginx_status {
        stub_status;
    }
}
```

```bash
# Запуск экспортера
docker run -p 9113:9113 nginx/nginx-prometheus-exporter:latest \
    -nginx.scrape-uri=http://nginx:8080/nginx_status
```

### Telegraf

```toml
# /etc/telegraf/telegraf.d/nginx.conf
[[inputs.nginx]]
  urls = ["http://localhost:8080/nginx_status"]
```

### Filebeat (ELK)

```yaml
# /etc/filebeat/filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/nginx/access.log
  json.keys_under_root: true
  json.add_error_key: true

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "nginx-%{+yyyy.MM.dd}"
```

### Vector

```toml
# /etc/vector/vector.toml
[sources.nginx_logs]
type = "file"
include = ["/var/log/nginx/access.log"]

[transforms.parse_nginx]
type = "remap"
inputs = ["nginx_logs"]
source = '''
. = parse_json!(.message)
'''

[sinks.elasticsearch]
type = "elasticsearch"
inputs = ["parse_nginx"]
endpoint = "http://elasticsearch:9200"
```

## Анализ логов

### GoAccess (real-time)

```bash
# Интерактивный режим
goaccess /var/log/nginx/access.log -c

# HTML отчёт
goaccess /var/log/nginx/access.log -o report.html --log-format=COMBINED

# Real-time HTML
goaccess /var/log/nginx/access.log -o /var/www/html/report.html \
    --log-format=COMBINED --real-time-html
```

### Bash однострочники

```bash
# Топ IP по запросам
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20

# Топ URL
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20

# Коды ответов
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Запросы в секунду
awk '{print $4}' /var/log/nginx/access.log | cut -d: -f1-3 | uniq -c | tail -10

# Медленные запросы (если есть request_time)
awk '$NF > 1 {print $0}' /var/log/nginx/access.log | tail -20

# 5xx ошибки
grep '" 5[0-9][0-9] ' /var/log/nginx/access.log | tail -50
```

## Полный пример

```nginx
http {
    # JSON формат для ELK
    log_format json escape=json '{'
        '"@timestamp": "$time_iso8601",'
        '"client_ip": "$remote_addr",'
        '"method": "$request_method",'
        '"uri": "$request_uri",'
        '"status": $status,'
        '"bytes": $body_bytes_sent,'
        '"duration": $request_time,'
        '"user_agent": "$http_user_agent",'
        '"referer": "$http_referer",'
        '"upstream": "$upstream_addr",'
        '"upstream_time": "$upstream_response_time",'
        '"cache": "$upstream_cache_status"'
    '}';

    # Буферизированное логирование
    access_log /var/log/nginx/access.json json buffer=32k flush=5s;
    error_log /var/log/nginx/error.log warn;

    server {
        listen 80;
        server_name example.com;

        # Отдельные логи для сервера
        access_log /var/log/nginx/example.com.access.json json;
        error_log /var/log/nginx/example.com.error.log;

        # Не логировать статику
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
            access_log off;
            expires 30d;
        }

        # Не логировать health checks
        location = /health {
            access_log off;
            return 200;
        }

        location / {
            proxy_pass http://backend;
        }
    }

    # Статистика
    server {
        listen 8080;
        allow 127.0.0.1;
        allow 10.0.0.0/8;
        deny all;

        location /nginx_status {
            stub_status;
        }
    }
}
```
