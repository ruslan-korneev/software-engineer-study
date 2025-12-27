# Продвинутые темы

## Rewrite правила

### Return vs Rewrite

```nginx
# return — простые редиректы (быстрее)
location /old {
    return 301 https://example.com/new;
}

# rewrite — для сложной логики с regex
location /products {
    rewrite ^/products/(\d+)$ /item?id=$1 last;
}
```

### Синтаксис rewrite

```nginx
rewrite regex replacement [flag];
```

| Флаг | Описание |
|------|----------|
| `last` | Остановить rewrite, начать поиск location заново |
| `break` | Остановить rewrite, продолжить в текущем location |
| `redirect` | Временный редирект 302 |
| `permanent` | Постоянный редирект 301 |

### Примеры rewrite

```nginx
server {
    # Убрать .html
    rewrite ^(.*)\.html$ $1 permanent;

    # Добавить trailing slash
    rewrite ^([^.]*[^/])$ $1/ permanent;

    # Убрать index.php
    rewrite ^/index\.php/?(.*)$ /$1 permanent;

    # SEO-friendly URLs
    location / {
        rewrite ^/product/([0-9]+)/([a-z-]+)$ /product.php?id=$1&slug=$2 last;
    }

    # Редирект с www на без www
    if ($host ~* ^www\.(.*)) {
        set $host_without_www $1;
        return 301 $scheme://$host_without_www$request_uri;
    }
}
```

### Отладка rewrite

```nginx
# Включить лог rewrite
error_log /var/log/nginx/error.log notice;
rewrite_log on;
```

## If директива

**Важно:** `if` в nginx работает не так, как в языках программирования. Используйте осторожно!

### Условия

```nginx
# Проверка переменной
if ($request_method = POST) { ... }

# Регулярное выражение
if ($http_user_agent ~* "mobile") { ... }

# Отрицание
if ($http_cookie !~ "auth") { ... }

# Существование файла
if (-f $request_filename) { ... }

# Существование директории
if (-d $request_filename) { ... }

# Файл исполняемый
if (-x $request_filename) { ... }

# Не существует
if (!-e $request_filename) { ... }
```

### Безопасные использования if

```nginx
# В server контексте
server {
    if ($host !~* ^(www\.)?example\.com$) {
        return 444;
    }
}

# return и rewrite в if — безопасно
location / {
    if ($request_method = OPTIONS) {
        return 204;
    }
}
```

### Что избегать

```nginx
# ПЛОХО — непредсказуемое поведение
location / {
    if ($slow) {
        set $limit_rate 10k;  # Может не работать как ожидается
    }

    # Другие директивы в location с if могут не работать
    proxy_pass http://backend;
}
```

## Map директива

Создание переменных на основе других переменных:

```nginx
http {
    # Простой маппинг
    map $uri $new_uri {
        /old-page /new-page;
        /legacy   /modern;
        default   $uri;
    }

    # С регулярными выражениями
    map $uri $backend {
        ~^/api/v1/  backend_v1;
        ~^/api/v2/  backend_v2;
        default     backend_default;
    }

    # Проверка User-Agent
    map $http_user_agent $is_mobile {
        ~*mobile  1;
        ~*android 1;
        ~*iphone  1;
        default   0;
    }

    # Выбор бэкенда по cookie
    map $cookie_version $upstream {
        "beta"   beta_backend;
        default  stable_backend;
    }

    server {
        location / {
            if ($is_mobile) {
                return 302 https://m.example.com$request_uri;
            }

            proxy_pass http://$upstream;
        }
    }
}
```

### Параметры map

```nginx
map $var $result {
    default       value;        # Значение по умолчанию
    hostnames;                  # Включить поддержку wildcard
    include       file.map;     # Включить файл с маппингами
    volatile;                   # Не кэшировать (для переменных)
}
```

## Geo модуль

Установка переменных на основе IP-адреса клиента:

```nginx
http {
    # Базовое использование
    geo $country {
        default        unknown;
        127.0.0.1      local;
        192.168.0.0/16 internal;
        10.0.0.0/8     internal;
    }

    # С диапазонами
    geo $allowed {
        default 0;
        192.168.1.0/24 1;
        10.0.0.0/8     1;
    }

    # Из файла
    geo $geo_country {
        include /etc/nginx/geo/countries.conf;
    }

    server {
        location / {
            if ($allowed = 0) {
                return 403;
            }
        }
    }
}
```

### GeoIP модуль

```nginx
# Требует ngx_http_geoip_module
load_module modules/ngx_http_geoip_module.so;

http {
    geoip_country /usr/share/GeoIP/GeoIP.dat;
    geoip_city /usr/share/GeoIP/GeoLiteCity.dat;

    # Доступные переменные:
    # $geoip_country_code (RU, US, etc.)
    # $geoip_country_name
    # $geoip_city
    # $geoip_region

    map $geoip_country_code $nearest_server {
        RU  backend_ru;
        UA  backend_ru;
        US  backend_us;
        default backend_eu;
    }
}
```

## Split Clients (A/B тестирование)

```nginx
http {
    # Разделение трафика по хэшу
    split_clients $request_id $variant {
        10%  variant_a;
        10%  variant_b;
        *    variant_default;
    }

    # По cookie/IP
    split_clients "${remote_addr}${http_user_agent}" $experiment {
        50%  "experiment";
        *    "control";
    }

    server {
        location / {
            add_header X-Variant $variant;

            if ($variant = variant_a) {
                proxy_pass http://backend_new;
            }

            proxy_pass http://backend_old;
        }
    }
}
```

## Встроенные переменные

### Полный список основных переменных

```nginx
# Запрос
$request              # Полный запрос "GET /path HTTP/1.1"
$request_method       # GET, POST, etc.
$request_uri          # URI с query string
$uri                  # URI без query string
$args                 # Query string
$arg_name             # Значение параметра ?name=value
$is_args              # "?" если есть аргументы

# Клиент
$remote_addr          # IP клиента
$remote_port          # Порт клиента
$remote_user          # Пользователь (Basic Auth)
$http_user_agent      # User-Agent
$http_referer         # Referer
$http_cookie          # Cookie заголовок
$cookie_name          # Значение cookie

# Сервер
$host                 # Имя хоста из запроса
$server_name          # server_name из конфига
$server_addr          # IP сервера
$server_port          # Порт сервера
$server_protocol      # HTTP/1.0, HTTP/1.1, HTTP/2.0

# Ответ
$status               # HTTP код ответа
$body_bytes_sent      # Размер тела ответа
$bytes_sent           # Всего отправлено

# Время
$time_local           # Локальное время
$time_iso8601         # ISO 8601 формат
$request_time         # Время обработки запроса
$msec                 # Текущее время в миллисекундах

# SSL
$ssl_protocol         # TLSv1.2, TLSv1.3
$ssl_cipher           # Используемый шифр
$ssl_client_cert      # Клиентский сертификат

# Upstream
$upstream_addr        # Адрес upstream сервера
$upstream_response_time # Время ответа upstream
$upstream_status      # Статус от upstream
$upstream_cache_status # HIT, MISS, BYPASS, etc.
```

## njs (JavaScript в nginx)

### Установка

```bash
# Ubuntu/Debian
apt install nginx-module-njs

# Загрузка модуля
load_module modules/ngx_http_js_module.so;
```

### Простой пример

```javascript
// /etc/nginx/njs/hello.js
function hello(r) {
    r.return(200, "Hello from njs!\n");
}

function headers(r) {
    var h = r.headersIn;
    var result = {};
    for (var name in h) {
        result[name] = h[name];
    }
    r.return(200, JSON.stringify(result, null, 2));
}

export default { hello, headers };
```

```nginx
load_module modules/ngx_http_js_module.so;

http {
    js_import main from /etc/nginx/njs/hello.js;

    server {
        location /hello {
            js_content main.hello;
        }

        location /headers {
            js_content main.headers;
        }
    }
}
```

### Модификация запросов

```javascript
// /etc/nginx/njs/auth.js
function validateToken(r) {
    var token = r.headersIn['Authorization'];

    if (!token || !token.startsWith('Bearer ')) {
        r.return(401, 'Unauthorized');
        return;
    }

    // Простая проверка (в реальности - JWT decode)
    var jwt = token.slice(7);
    if (jwt.length < 10) {
        r.return(403, 'Invalid token');
        return;
    }

    // Добавить заголовок для backend
    r.headersOut['X-User-Id'] = 'extracted-user-id';
    r.return(200);
}

export default { validateToken };
```

```nginx
http {
    js_import auth from /etc/nginx/njs/auth.js;

    server {
        location /api/ {
            auth_request /auth;
            proxy_pass http://backend;
        }

        location = /auth {
            internal;
            js_content auth.validateToken;
        }
    }
}
```

## Отладка

### Debug лог

```nginx
# Требует сборки с --with-debug
error_log /var/log/nginx/debug.log debug;

# Для конкретных клиентов
events {
    debug_connection 192.168.1.100;
    debug_connection 10.0.0.0/24;
}
```

### Вывод переменных

```nginx
location /debug {
    add_header X-Debug-Host $host;
    add_header X-Debug-URI $uri;
    add_header X-Debug-Args $args;
    return 200 "Debug info in headers";
}
```

### Echo модуль (для отладки)

```nginx
# Требует установки nginx-module-echo
location /echo {
    echo "host: $host";
    echo "uri: $uri";
    echo "args: $args";
    echo "remote_addr: $remote_addr";
}
```

## Типичные ошибки и решения

### 502 Bad Gateway

```nginx
# Причины: backend недоступен, таймаут
# Решения:
proxy_connect_timeout 60s;
proxy_send_timeout 60s;
proxy_read_timeout 60s;

# Увеличить буферы
proxy_buffer_size 128k;
proxy_buffers 4 256k;
proxy_busy_buffers_size 256k;
```

### 504 Gateway Timeout

```nginx
# Увеличить таймауты
proxy_read_timeout 300s;
fastcgi_read_timeout 300s;
```

### 413 Request Entity Too Large

```nginx
# Увеличить лимит
client_max_body_size 100m;
```

### Redirect loop

```nginx
# Проверить правила rewrite
# Избегать конфликтов между server блоками
# Использовать return вместо rewrite для редиректов
```

### Upstream sent too big header

```nginx
proxy_buffer_size 16k;
proxy_busy_buffers_size 24k;
fastcgi_buffer_size 16k;
fastcgi_buffers 16 16k;
```

### Cannot allocate memory

```nginx
# Уменьшить буферы или увеличить память
# Проверить лимиты ОС
worker_rlimit_nofile 65535;

# Уменьшить количество workers если много
worker_processes 4;
```

## Практические рецепты

### Защита от DDoS

```nginx
limit_req_zone $binary_remote_addr zone=req:10m rate=10r/s;
limit_conn_zone $binary_remote_addr zone=conn:10m;

server {
    limit_req zone=req burst=20 nodelay;
    limit_conn conn 10;

    # Блокировка по User-Agent
    if ($http_user_agent ~* (wget|curl|libwww)) {
        return 403;
    }
}
```

### Canary deployment

```nginx
split_clients $request_id $backend {
    5%  canary;
    *   stable;
}

upstream stable {
    server 192.168.1.10:8080;
}

upstream canary {
    server 192.168.1.20:8080;
}

server {
    location / {
        proxy_pass http://$backend;
    }
}
```

### Maintenance mode

```nginx
server {
    # Проверка файла maintenance
    if (-f /var/www/maintenance.flag) {
        return 503;
    }

    error_page 503 @maintenance;

    location @maintenance {
        root /var/www/html;
        rewrite ^(.*)$ /maintenance.html break;
    }
}
```

### Защита статики по токену

```nginx
location /protected/ {
    secure_link $arg_md5,$arg_expires;
    secure_link_md5 "$secure_link_expires$uri$remote_addr secret";

    if ($secure_link = "") {
        return 403;
    }

    if ($secure_link = "0") {
        return 410;
    }

    alias /var/www/protected/;
}
```
