# Безопасность и HTTPS

## Базовая настройка SSL/TLS

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;
}
```

### Типы сертификатов

| Тип | Описание | Использование |
|-----|----------|---------------|
| Self-signed | Самоподписанный | Разработка, внутренние сервисы |
| Domain Validated (DV) | Проверка домена | Публичные сайты |
| Organization Validated (OV) | Проверка организации | Корпоративные сайты |
| Extended Validation (EV) | Расширенная проверка | Банки, финансы |

### Создание самоподписанного сертификата

```bash
# Генерация ключа и сертификата
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/nginx/ssl/selfsigned.key \
    -out /etc/nginx/ssl/selfsigned.crt \
    -subj "/C=RU/ST=Moscow/L=Moscow/O=Company/CN=example.com"

# Генерация DH-параметров
openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048
```

## Let's Encrypt + Certbot

### Установка Certbot

```bash
# Ubuntu/Debian
sudo apt install certbot python3-certbot-nginx

# CentOS/RHEL
sudo dnf install certbot python3-certbot-nginx
```

### Получение сертификата

```bash
# Автоматическая настройка nginx
sudo certbot --nginx -d example.com -d www.example.com

# Только получить сертификат (ручная настройка)
sudo certbot certonly --webroot -w /var/www/html -d example.com

# Standalone (nginx остановлен)
sudo certbot certonly --standalone -d example.com
```

### Автообновление

```bash
# Тест обновления
sudo certbot renew --dry-run

# Cron для автообновления (добавляется автоматически)
# /etc/cron.d/certbot
0 */12 * * * root certbot renew --quiet --post-hook "systemctl reload nginx"
```

### Структура сертификатов Let's Encrypt

```
/etc/letsencrypt/live/example.com/
├── cert.pem       # Сертификат домена
├── chain.pem      # Промежуточные сертификаты
├── fullchain.pem  # cert.pem + chain.pem (используйте это!)
└── privkey.pem    # Приватный ключ
```

## Оптимальная конфигурация SSL

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    # Сертификаты
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Протоколы (только современные)
    ssl_protocols TLSv1.2 TLSv1.3;

    # Шифры
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # DH параметры
    ssl_dhparam /etc/nginx/ssl/dhparam.pem;

    # ECDH кривые
    ssl_ecdh_curve secp384r1;

    # Кэш сессий
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;
}
```

### Snippet для переиспользования

```nginx
# /etc/nginx/snippets/ssl-params.conf
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;

# Использование
server {
    listen 443 ssl http2;
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    include /etc/nginx/snippets/ssl-params.conf;
}
```

## HTTP/2

```nginx
server {
    # HTTP/2 включается с ssl
    listen 443 ssl http2;

    # или отдельно (с nginx 1.25.1+)
    listen 443 ssl;
    http2 on;
}
```

### Настройки HTTP/2

```nginx
# Максимум concurrent streams
http2_max_concurrent_streams 128;

# Размер буфера для preload
http2_chunk_size 8k;
```

## HTTP/3 (QUIC)

```nginx
server {
    # HTTP/3 на UDP
    listen 443 quic reuseport;

    # HTTP/2 fallback на TCP
    listen 443 ssl http2;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    # Early data (0-RTT)
    ssl_early_data on;

    # Заголовок для браузера
    add_header Alt-Svc 'h3=":443"; ma=86400';
}
```

## Редирект HTTP → HTTPS

```nginx
# Отдельный server блок
server {
    listen 80;
    server_name example.com www.example.com;

    # Редирект всех запросов на HTTPS
    return 301 https://$host$request_uri;
}

# Или с условием
server {
    listen 80;
    listen 443 ssl;
    server_name example.com;

    if ($scheme = http) {
        return 301 https://$host$request_uri;
    }
}
```

## HSTS (HTTP Strict Transport Security)

Говорит браузеру использовать только HTTPS:

```nginx
server {
    listen 443 ssl http2;

    # max-age в секундах (1 год = 31536000)
    add_header Strict-Transport-Security "max-age=31536000" always;

    # Включая поддомены
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Для preload list
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
}
```

**Важно:** HSTS работает только для HTTPS. Не добавляйте на HTTP.

## OCSP Stapling

Ускоряет проверку сертификата:

```nginx
ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;
```

## Ограничение доступа по IP

### Allow / Deny

```nginx
location /admin/ {
    # Разрешить только эти IP
    allow 192.168.1.0/24;
    allow 10.0.0.1;
    deny all;

    proxy_pass http://backend;
}

# Или наоборот
location / {
    deny 192.168.1.100;   # Заблокировать IP
    deny 10.0.0.0/8;      # Заблокировать подсеть
    allow all;            # Остальным разрешить
}
```

### Geo модуль для блокировки

```nginx
geo $blocked {
    default 0;
    192.168.1.100 1;
    10.0.0.0/8 1;
}

server {
    if ($blocked) {
        return 403;
    }
}
```

## HTTP Basic Auth

```bash
# Создание файла паролей
sudo apt install apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd admin
sudo htpasswd /etc/nginx/.htpasswd user2
```

```nginx
location /admin/ {
    auth_basic "Restricted Area";
    auth_basic_user_file /etc/nginx/.htpasswd;

    proxy_pass http://backend;
}
```

## Rate Limiting

### Ограничение запросов

```nginx
# Определение зоны (в http контексте)
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

server {
    location /api/ {
        # Применение лимита
        limit_req zone=api;

        # С burst (очередью)
        limit_req zone=api burst=20;

        # С nodelay (не задерживать burst)
        limit_req zone=api burst=20 nodelay;

        proxy_pass http://backend;
    }
}
```

### Ограничение соединений

```nginx
# Определение зоны
limit_conn_zone $binary_remote_addr zone=addr:10m;

server {
    # Макс 10 соединений с одного IP
    limit_conn addr 10;

    # Или на уровне location
    location /download/ {
        limit_conn addr 1;  # Только 1 загрузка
        limit_rate 100k;    # Скорость 100KB/s
    }
}
```

### Ответы при превышении лимита

```nginx
# HTTP код ошибки (по умолчанию 503)
limit_req_status 429;
limit_conn_status 429;

# Логирование
limit_req_log_level warn;
limit_conn_log_level warn;
```

## Security Headers

```nginx
server {
    # Предотвращение clickjacking
    add_header X-Frame-Options "SAMEORIGIN" always;

    # XSS защита (для старых браузеров)
    add_header X-XSS-Protection "1; mode=block" always;

    # Предотвращение MIME-sniffing
    add_header X-Content-Type-Options "nosniff" always;

    # Referrer policy
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Content Security Policy
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';" always;

    # Permissions Policy
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
}
```

### Snippet для security headers

```nginx
# /etc/nginx/snippets/security-headers.conf
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# Использование
server {
    include /etc/nginx/snippets/security-headers.conf;
}
```

## Защита от атак

### Скрытие версии nginx

```nginx
http {
    server_tokens off;
}
```

### Ограничение методов

```nginx
location / {
    # Разрешить только GET, POST, HEAD
    if ($request_method !~ ^(GET|POST|HEAD)$) {
        return 405;
    }
}
```

### Защита от slowloris

```nginx
http {
    # Таймауты
    client_body_timeout 10s;
    client_header_timeout 10s;
    send_timeout 10s;

    # Размер буферов
    client_body_buffer_size 1k;
    client_header_buffer_size 1k;
    large_client_header_buffers 2 1k;
    client_max_body_size 1m;
}
```

### Защита от сканирования

```nginx
# Блокировка типичных сканеров
if ($http_user_agent ~* (nmap|nikto|wikto|sf|sqlmap|bsqlbf|w3af|acunetix|havij|appscan)) {
    return 403;
}

# Запрет доступа к скрытым файлам
location ~ /\. {
    deny all;
    access_log off;
    log_not_found off;
}

# Запрет доступа к бэкапам
location ~* \.(bak|conf|dist|fla|in[ci]|log|psd|sh|sql|sw[op])$ {
    deny all;
}
```

## Полный пример безопасной конфигурации

```nginx
# HTTP → HTTPS редирект
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;
}

# WWW → non-WWW редирект
server {
    listen 443 ssl http2;
    server_name www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    return 301 https://example.com$request_uri;
}

# Основной сервер
server {
    listen 443 ssl http2;
    server_name example.com;

    # SSL
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    include /etc/nginx/snippets/ssl-params.conf;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Скрыть версию
    server_tokens off;

    # Rate limiting
    limit_req zone=api burst=10 nodelay;

    # Защита от скрытых файлов
    location ~ /\. {
        deny all;
    }

    # Admin с IP ограничением и auth
    location /admin/ {
        allow 192.168.1.0/24;
        deny all;
        auth_basic "Admin";
        auth_basic_user_file /etc/nginx/.htpasswd;
        proxy_pass http://backend;
    }

    # API с rate limiting
    location /api/ {
        limit_req zone=api burst=20;
        proxy_pass http://backend;
    }

    location / {
        root /var/www/html;
        index index.html;
    }
}
```
