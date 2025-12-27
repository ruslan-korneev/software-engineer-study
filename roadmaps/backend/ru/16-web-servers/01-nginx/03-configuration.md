# Основы конфигурации Nginx

## Структура конфигурационного файла

Конфигурация nginx состоит из **директив**, организованных в иерархические **контексты**.

```nginx
# Главный контекст (main)
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;

# Контекст events
events {
    worker_connections 1024;
    multi_accept on;
}

# Контекст http
http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Контекст server (виртуальный хост)
    server {
        listen 80;
        server_name example.com;

        # Контекст location
        location / {
            root /var/www/html;
            index index.html;
        }

        location /api/ {
            proxy_pass http://backend;
        }
    }
}
```

## Контексты

### Иерархия контекстов

```
main (глобальный)
├── events
├── http
│   ├── upstream
│   ├── server
│   │   ├── location
│   │   │   ├── location (вложенный)
│   │   │   └── if
│   │   └── if
│   └── map
├── stream (TCP/UDP)
│   ├── upstream
│   └── server
└── mail
    └── server
```

### main (глобальный контекст)

Директивы вне любых блоков. Влияют на весь nginx.

```nginx
# Пользователь и группа worker-процессов
user nginx nginx;

# Количество worker-процессов (auto = по числу CPU)
worker_processes auto;

# Привязка workers к ядрам CPU
worker_cpu_affinity auto;

# Путь к PID файлу
pid /var/run/nginx.pid;

# Максимум открытых файлов на worker
worker_rlimit_nofile 65535;

# Путь к логу ошибок
error_log /var/log/nginx/error.log warn;
```

### events

Настройки обработки соединений.

```nginx
events {
    # Максимум соединений на один worker
    worker_connections 4096;

    # Метод обработки событий (epoll для Linux, kqueue для BSD/macOS)
    use epoll;

    # Принимать все соединения сразу
    multi_accept on;

    # Использовать mutex для accept()
    accept_mutex off;
}
```

### http

Настройки HTTP-сервера.

```nginx
http {
    # Включение MIME-типов
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Логирование
    log_format main '$remote_addr - $request - $status';
    access_log /var/log/nginx/access.log main;

    # Оптимизации
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    # Таймауты
    keepalive_timeout 65;
    client_body_timeout 12;
    client_header_timeout 12;

    # Сжатие
    gzip on;
    gzip_types text/plain text/css application/json;

    # Включение конфигов сайтов
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

### server

Виртуальный хост (сайт).

```nginx
server {
    # Порт и адрес
    listen 80;
    listen [::]:80;  # IPv6

    # Имена сервера
    server_name example.com www.example.com;

    # Корневая директория
    root /var/www/example.com;

    # Индексные файлы
    index index.html index.htm;

    # Лог для этого сайта
    access_log /var/log/nginx/example.com.access.log;
    error_log /var/log/nginx/example.com.error.log;

    # Locations...
}
```

### location

Обработка URL-путей.

```nginx
server {
    # Точное совпадение
    location = / {
        # только для "/"
    }

    # Префикс
    location /images/ {
        # для /images/*, /images/photos/* и т.д.
    }

    # Регулярное выражение (регистрозависимое)
    location ~ \.php$ {
        # для *.php
    }

    # Регулярное выражение (регистронезависимое)
    location ~* \.(jpg|jpeg|png|gif)$ {
        # для изображений
    }

    # Приоритетный префикс (выше regex)
    location ^~ /static/ {
        # regex не проверяются
    }
}
```

## Синтаксис директив

### Простые директивы

```nginx
# директива параметр;
worker_processes 4;
error_log /var/log/nginx/error.log;
gzip on;
```

### Блочные директивы

```nginx
# директива параметры { ... }
server {
    listen 80;
    server_name example.com;
}

location /api/ {
    proxy_pass http://backend;
}
```

### Правила синтаксиса

1. Директивы заканчиваются точкой с запятой `;`
2. Блоки окружены фигурными скобками `{ }`
3. Комментарии начинаются с `#`
4. Пробелы и переносы строк игнорируются
5. Строки можно заключать в одинарные `'` или двойные `"` кавычки

```nginx
# Это комментарий
server {
    # Кавычки нужны если есть пробелы или спецсимволы
    server_name "my server.com";

    location ~* "\.php$" {
        # ...
    }
}
```

## Наследование директив

Директивы наследуются от родительских контекстов к дочерним.

```nginx
http {
    # Применяется ко всем servers
    gzip on;
    root /var/www/default;

    server {
        server_name site1.com;
        # Наследует gzip on; и root

        location /images/ {
            # Наследует всё сверху
        }
    }

    server {
        server_name site2.com;
        root /var/www/site2;  # Переопределяет root
        gzip off;              # Переопределяет gzip

        location /api/ {
            # Наследует gzip off; и root /var/www/site2
        }
    }
}
```

### Типы наследования

| Тип | Описание | Пример директив |
|-----|----------|-----------------|
| Стандартное | Наследуется если не переопределено | `root`, `index`, `error_page` |
| Массивное | Добавляется к родительскому | `add_header` (но с нюансами!) |
| Заменяющее | Полностью заменяет родительское | `try_files` |

**Важно:** `add_header` полностью заменяется если определён в дочернем контексте:

```nginx
server {
    add_header X-Frame-Options "SAMEORIGIN";

    location / {
        # X-Frame-Options НЕ наследуется!
        add_header X-Content-Type-Options "nosniff";
    }
}
```

## Include и организация конфигов

### Структура для production

```
/etc/nginx/
├── nginx.conf              # Главный конфиг
├── mime.types
├── conf.d/                 # Глобальные настройки
│   ├── gzip.conf
│   ├── security.conf
│   └── logging.conf
├── snippets/               # Переиспользуемые блоки
│   ├── ssl-params.conf
│   ├── proxy-params.conf
│   └── php-fpm.conf
└── sites-enabled/          # Сайты
    ├── example.com.conf
    └── api.example.com.conf
```

### nginx.conf

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Общие настройки
    include /etc/nginx/conf.d/*.conf;

    # Сайты
    include /etc/nginx/sites-enabled/*.conf;
}
```

### snippets/ssl-params.conf

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
ssl_prefer_server_ciphers off;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_session_tickets off;
```

### sites-enabled/example.com.conf

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    include /etc/nginx/snippets/ssl-params.conf;

    root /var/www/example.com;
    index index.html;
}
```

## Единицы измерения

### Размеры

```nginx
# Без суффикса — байты
client_max_body_size 1048576;   # 1 МБ в байтах

# k или K — килобайты
client_max_body_size 1024k;     # 1 МБ

# m или M — мегабайты
client_max_body_size 1m;        # 1 МБ

# g или G — гигабайты
client_max_body_size 1g;        # 1 ГБ
```

### Время

```nginx
# Без суффикса — секунды
keepalive_timeout 65;

# ms — миллисекунды
resolver_timeout 100ms;

# s — секунды
client_body_timeout 60s;

# m — минуты
proxy_read_timeout 5m;

# h — часы
proxy_cache_valid 200 1h;

# d — дни
ssl_session_timeout 1d;

# Комбинации
proxy_read_timeout 1h30m;       # 1 час 30 минут
```

## Переменные

Nginx предоставляет множество встроенных переменных.

### Информация о запросе

```nginx
$request_method     # GET, POST, PUT...
$request_uri        # /path?query (оригинальный URI)
$uri                # /path (нормализованный, без query string)
$args               # query string без ?
$arg_name           # значение параметра ?name=value
$is_args            # "?" если есть аргументы, иначе пусто
$query_string       # то же что $args
```

### Информация о клиенте

```nginx
$remote_addr        # IP клиента
$remote_port        # Порт клиента
$remote_user        # Имя пользователя (Basic Auth)
$http_user_agent    # User-Agent
$http_referer       # Referer
$http_cookie        # Cookie заголовок
$cookie_name        # Значение cookie с именем name
```

### Информация о сервере

```nginx
$host               # Имя хоста из запроса
$server_name        # Имя сервера из конфига
$server_addr        # IP сервера
$server_port        # Порт сервера
$server_protocol    # HTTP/1.0 или HTTP/1.1
$scheme             # http или https
```

### Пример использования

```nginx
server {
    # Логирование с переменными
    log_format detailed '$remote_addr - $host - $request_uri - $status';

    # Условная логика
    if ($request_method = POST) {
        return 405;
    }

    # Передача заголовков
    location /api/ {
        proxy_pass http://backend;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $host;
    }

    # Редирект с сохранением пути
    location /old/ {
        return 301 https://newdomain.com$request_uri;
    }
}
```

## Валидация конфигурации

Всегда проверяйте конфигурацию перед применением:

```bash
# Проверка синтаксиса
sudo nginx -t

# Проверка + вывод конфига
sudo nginx -T

# Проверка конкретного файла
sudo nginx -t -c /path/to/nginx.conf

# Безопасный reload
sudo nginx -t && sudo nginx -s reload
```

Типичные ошибки:
- Забытая точка с запятой `;`
- Незакрытые скобки `{ }`
- Неверный путь в `include`
- Дублирование `listen` директив
