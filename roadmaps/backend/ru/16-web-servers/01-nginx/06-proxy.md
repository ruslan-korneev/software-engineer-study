# Проксирование в Nginx

## Reverse Proxy — концепция

Reverse proxy принимает запросы от клиентов и перенаправляет их на backend-серверы.

```
┌─────────┐     ┌─────────┐     ┌─────────────────┐
│ Client  │────▶│  Nginx  │────▶│ Backend Server  │
│         │◀────│ (proxy) │◀────│ (app, API)      │
└─────────┘     └─────────┘     └─────────────────┘
```

**Преимущества:**
- SSL-терминация
- Кэширование
- Сжатие
- Балансировка нагрузки
- Защита backend-серверов
- Единая точка входа

## Базовое проксирование

### proxy_pass

```nginx
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
    }
}
```

### Trailing slash в proxy_pass

**Важно!** Наличие или отсутствие `/` в конце `proxy_pass` меняет поведение:

```nginx
# БЕЗ trailing slash — путь добавляется
location /api/ {
    proxy_pass http://backend;
    # /api/users → http://backend/api/users
}

# С trailing slash — путь заменяется
location /api/ {
    proxy_pass http://backend/;
    # /api/users → http://backend/users
}

# С путём — путь заменяется
location /api/ {
    proxy_pass http://backend/v2/;
    # /api/users → http://backend/v2/users
}
```

### Примеры

```nginx
# Проксирование без изменения пути
location /api/ {
    proxy_pass http://127.0.0.1:3000;
    # /api/users → http://127.0.0.1:3000/api/users
}

# Убрать /api из пути
location /api/ {
    proxy_pass http://127.0.0.1:3000/;
    # /api/users → http://127.0.0.1:3000/users
}

# Заменить /old на /new
location /old/ {
    proxy_pass http://127.0.0.1:3000/new/;
    # /old/path → http://127.0.0.1:3000/new/path
}
```

## Передача заголовков

По умолчанию nginx изменяет некоторые заголовки:
- `Host` → `$proxy_host` (хост из proxy_pass)
- `Connection` → "close"

### proxy_set_header

```nginx
location /api/ {
    proxy_pass http://backend;

    # Передать оригинальный Host
    proxy_set_header Host $host;

    # Реальный IP клиента
    proxy_set_header X-Real-IP $remote_addr;

    # Цепочка прокси
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    # Протокол (http/https)
    proxy_set_header X-Forwarded-Proto $scheme;

    # Порт
    proxy_set_header X-Forwarded-Port $server_port;
}
```

### Стандартный набор заголовков

```nginx
# snippets/proxy-headers.conf
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Forwarded-Port $server_port;

# Использование
location /api/ {
    proxy_pass http://backend;
    include /etc/nginx/snippets/proxy-headers.conf;
}
```

### Скрытие заголовков от backend

```nginx
location /api/ {
    proxy_pass http://backend;

    # Не передавать эти заголовки
    proxy_set_header Accept-Encoding "";
    proxy_set_header Cookie "";
}
```

### Скрытие заголовков от клиента

```nginx
location /api/ {
    proxy_pass http://backend;

    # Не возвращать эти заголовки клиенту
    proxy_hide_header X-Powered-By;
    proxy_hide_header Server;
}
```

## Буферизация

### Настройки буферов

```nginx
location /api/ {
    proxy_pass http://backend;

    # Включить/выключить буферизацию
    proxy_buffering on;

    # Размер буфера для первой части ответа (заголовки)
    proxy_buffer_size 4k;

    # Количество и размер буферов для тела
    proxy_buffers 8 16k;

    # Максимум данных для буферизации пока клиент читает
    proxy_busy_buffers_size 32k;

    # Временные файлы если буферы заполнены
    proxy_temp_file_write_size 64k;
    proxy_max_temp_file_size 1024m;
}
```

### Отключение буферизации

Для streaming или Server-Sent Events:

```nginx
location /stream/ {
    proxy_pass http://backend;
    proxy_buffering off;

    # Отключить буферизацию на стороне клиента тоже
    proxy_cache off;
}
```

### X-Accel-Buffering

Backend может управлять буферизацией через заголовок:

```
X-Accel-Buffering: no
```

```nginx
location /api/ {
    proxy_pass http://backend;
    # Разрешить backend управлять буферизацией
    proxy_ignore_headers X-Accel-Buffering;  # или не указывать
}
```

## Таймауты

```nginx
location /api/ {
    proxy_pass http://backend;

    # Таймаут на установку соединения
    proxy_connect_timeout 60s;

    # Таймаут на отправку запроса
    proxy_send_timeout 60s;

    # Таймаут на получение ответа
    proxy_read_timeout 60s;
}
```

### Для долгих операций

```nginx
location /api/long-operation/ {
    proxy_pass http://backend;

    proxy_connect_timeout 60s;
    proxy_send_timeout 300s;
    proxy_read_timeout 300s;
}
```

## FastCGI (PHP-FPM)

```nginx
server {
    listen 80;
    server_name php.example.com;

    root /var/www/php-app/public;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        # Защита от выполнения произвольных файлов
        try_files $uri =404;

        # FastCGI к PHP-FPM
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        # или TCP: fastcgi_pass 127.0.0.1:9000;

        fastcgi_index index.php;

        # Путь к скрипту
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;

        # Остальные параметры
        include fastcgi_params;
    }

    # Запрет прямого доступа к .php файлам в определённых директориях
    location ~ ^/(vendor|storage)/.*\.php$ {
        deny all;
    }
}
```

### fastcgi_params

```nginx
# /etc/nginx/fastcgi_params (стандартный файл)
fastcgi_param QUERY_STRING       $query_string;
fastcgi_param REQUEST_METHOD     $request_method;
fastcgi_param CONTENT_TYPE       $content_type;
fastcgi_param CONTENT_LENGTH     $content_length;

fastcgi_param SCRIPT_FILENAME    $document_root$fastcgi_script_name;
fastcgi_param SCRIPT_NAME        $fastcgi_script_name;
fastcgi_param PATH_INFO          $fastcgi_path_info;
fastcgi_param REQUEST_URI        $request_uri;
fastcgi_param DOCUMENT_URI       $document_uri;
fastcgi_param DOCUMENT_ROOT      $document_root;
fastcgi_param SERVER_PROTOCOL    $server_protocol;
fastcgi_param HTTPS              $https if_not_empty;

fastcgi_param REMOTE_ADDR        $remote_addr;
fastcgi_param REMOTE_PORT        $remote_port;
fastcgi_param SERVER_ADDR        $server_addr;
fastcgi_param SERVER_PORT        $server_port;
fastcgi_param SERVER_NAME        $server_name;
```

## uWSGI (Python)

```nginx
server {
    listen 80;
    server_name python.example.com;

    location / {
        uwsgi_pass unix:/run/uwsgi/app.sock;
        # или TCP: uwsgi_pass 127.0.0.1:8000;

        include uwsgi_params;
    }

    location /static/ {
        alias /var/www/python-app/static/;
    }
}
```

### Gunicorn (HTTP proxy)

```nginx
server {
    listen 80;
    server_name django.example.com;

    location / {
        proxy_pass http://127.0.0.1:8000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /static/ {
        alias /var/www/django-app/staticfiles/;
    }

    location /media/ {
        alias /var/www/django-app/media/;
    }
}
```

## gRPC проксирование

```nginx
server {
    listen 443 ssl http2;
    server_name grpc.example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        grpc_pass grpc://127.0.0.1:50051;

        # Заголовки для gRPC
        grpc_set_header Host $host;
        grpc_set_header X-Real-IP $remote_addr;

        # Таймауты
        grpc_read_timeout 300s;
        grpc_send_timeout 300s;
    }
}
```

### gRPC с TLS к backend

```nginx
location / {
    grpc_pass grpcs://backend:50051;
    grpc_ssl_verify on;
    grpc_ssl_trusted_certificate /path/to/ca.pem;
}
```

## WebSocket проксирование

WebSocket требует специальных заголовков для upgrade соединения:

```nginx
# Map для Upgrade заголовка
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 80;
    server_name ws.example.com;

    location /ws/ {
        proxy_pass http://127.0.0.1:3000;

        # WebSocket специфичные заголовки
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        # Стандартные заголовки
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # Увеличить таймаут для долгоживущих соединений
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }
}
```

### WebSocket + обычный HTTP на одном location

```nginx
location / {
    proxy_pass http://backend;

    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_set_header Host $host;

    # Таймаут только для WebSocket
    proxy_read_timeout 86400s;
}
```

## Обработка ошибок

```nginx
location /api/ {
    proxy_pass http://backend;

    # Какие ошибки перехватывать
    proxy_intercept_errors on;

    # Следующий upstream при ошибке
    proxy_next_upstream error timeout http_502 http_503 http_504;

    # Лимит попыток
    proxy_next_upstream_tries 3;

    # Таймаут на все попытки
    proxy_next_upstream_timeout 10s;
}

# Кастомные страницы ошибок
error_page 502 503 504 /50x.html;
location = /50x.html {
    root /var/www/error-pages;
    internal;
}
```

## Полный пример

```nginx
upstream backend {
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    keepalive 32;
}

server {
    listen 80;
    server_name api.example.com;

    # Логирование
    access_log /var/log/nginx/api.access.log;
    error_log /var/log/nginx/api.error.log;

    # Размер тела запроса
    client_max_body_size 10m;

    location / {
        proxy_pass http://backend;

        # HTTP/1.1 для keepalive
        proxy_http_version 1.1;
        proxy_set_header Connection "";

        # Заголовки
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Буферизация
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 16k;

        # Таймауты
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        # Обработка ошибок
        proxy_intercept_errors on;
        proxy_next_upstream error timeout http_502 http_503;
    }

    # WebSocket endpoint
    location /socket.io/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400s;
    }

    # Health check endpoint
    location /health {
        proxy_pass http://backend;
        proxy_connect_timeout 5s;
        proxy_read_timeout 5s;
        access_log off;
    }
}
```
