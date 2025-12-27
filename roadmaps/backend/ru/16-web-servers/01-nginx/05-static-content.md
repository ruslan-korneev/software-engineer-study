# Раздача статического контента

## Базовая настройка

Nginx изначально проектировался для эффективной раздачи статического контента.

```nginx
server {
    listen 80;
    server_name static.example.com;

    # Корневая директория
    root /var/www/static;

    # Индексные файлы
    index index.html index.htm;

    location / {
        # Файлы ищутся относительно root
    }
}
```

## Root vs Alias

### Root

Путь добавляется к `root`:

```nginx
root /var/www/html;

location /images/ {
    # Запрос: /images/photo.jpg
    # Файл: /var/www/html/images/photo.jpg
}
```

### Alias

Путь **заменяется** на `alias`:

```nginx
location /images/ {
    alias /var/www/static/img/;
    # Запрос: /images/photo.jpg
    # Файл: /var/www/static/img/photo.jpg
}
```

**Важно:** При использовании `alias` убедитесь, что пути заканчиваются `/`:

```nginx
# Правильно
location /images/ {
    alias /var/www/img/;
}

# Неправильно (может сработать некорректно)
location /images {
    alias /var/www/img;
}
```

### Сравнение

| Аспект | root | alias |
|--------|------|-------|
| Работает с | URI добавляется | URI заменяется |
| Можно в | server, location | только location |
| Trailing slash | Не важен | Важен для location и alias |

```nginx
# Одинаковый результат
location /static/ {
    root /var/www;
    # /static/file.css → /var/www/static/file.css
}

location /static/ {
    alias /var/www/static/;
    # /static/file.css → /var/www/static/file.css
}
```

## Index директива

```nginx
# По умолчанию
index index.html;

# Несколько файлов (проверяются по порядку)
index index.html index.htm index.php;

# Можно указать полный путь
index /var/www/shared/index.html;
```

При запросе директории nginx ищет индексный файл:

```nginx
location / {
    root /var/www/html;
    index index.html;
    # GET /about/ → /var/www/html/about/index.html
}
```

## Try_files

Проверяет существование файлов по порядку:

```nginx
location / {
    root /var/www/html;
    try_files $uri $uri/ /index.html;
}
```

### Синтаксис

```nginx
try_files file1 file2 ... fallback;
```

- `$uri` — путь из запроса
- `$uri/` — как директория (добавляет index)
- `/fallback` — последний вариант (внутренний редирект)
- `=404` — вернуть 404

### Примеры

```nginx
# SPA (Single Page Application)
location / {
    try_files $uri $uri/ /index.html;
}

# Проверка файла или 404
location /images/ {
    try_files $uri =404;
}

# Fallback на backend
location / {
    try_files $uri $uri/ @backend;
}

location @backend {
    proxy_pass http://127.0.0.1:3000;
}

# PHP fallback
location / {
    try_files $uri $uri/ /index.php?$query_string;
}
```

### Распространённые паттерны

```nginx
# WordPress
location / {
    try_files $uri $uri/ /index.php?$args;
}

# Laravel
location / {
    try_files $uri $uri/ /index.php?$query_string;
}

# React/Vue/Angular SPA
location / {
    try_files $uri /index.html;
}

# Статика с fallback на proxy
location / {
    try_files $uri @proxy;
}
```

## Autoindex — листинг директорий

```nginx
location /files/ {
    alias /var/www/downloads/;
    autoindex on;

    # Формат вывода: html (default), xml, json, jsonp
    autoindex_format html;

    # Точный размер файлов (по умолчанию округляется)
    autoindex_exact_size off;

    # Локальное время вместо UTC
    autoindex_localtime on;
}
```

Результат при `autoindex_format html`:

```html
<html>
<head><title>Index of /files/</title></head>
<body>
<h1>Index of /files/</h1>
<pre>
<a href="../">../</a>
<a href="document.pdf">document.pdf</a>     15-Dec-2024 10:30    1.2M
<a href="image.png">image.png</a>           15-Dec-2024 09:15    256K
</pre>
</body>
</html>
```

## Оптимизация раздачи статики

### Sendfile

Передаёт файл напрямую из ядра, минуя userspace:

```nginx
http {
    # Включить sendfile
    sendfile on;

    # Размер чанка для sendfile
    sendfile_max_chunk 512k;
}
```

### TCP_nopush и TCP_nodelay

```nginx
http {
    sendfile on;

    # Отправлять заголовки и начало файла в одном пакете
    tcp_nopush on;

    # Отключить алгоритм Nagle (уменьшает latency)
    tcp_nodelay on;
}
```

**Как это работает:**
- `tcp_nopush` — собирает данные перед отправкой (оптимизирует throughput)
- `tcp_nodelay` — отправляет сразу без буферизации (оптимизирует latency)
- Nginx умеет использовать оба: `tcp_nopush` для тела, `tcp_nodelay` в конце

### Open file cache

Кэширует дескрипторы открытых файлов:

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

## Заголовки кэширования

### Expires

```nginx
location /static/ {
    # Абсолютное время
    expires 30d;

    # Относительно времени модификации файла
    expires modified +1w;

    # Запретить кэширование
    expires -1;

    # Максимальное время
    expires max;
}
```

### Cache-Control

```nginx
location /static/ {
    expires 30d;
    add_header Cache-Control "public, immutable";
}

location /api/ {
    add_header Cache-Control "no-store, no-cache, must-revalidate";
}
```

### Условное кэширование по типу файла

```nginx
# Изображения — 30 дней
location ~* \.(jpg|jpeg|png|gif|ico|svg|webp)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
    access_log off;
}

# CSS/JS — 1 год (с версионированием в имени)
location ~* \.(css|js)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}

# Шрифты — 1 год
location ~* \.(woff|woff2|ttf|otf|eot)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}

# HTML — без кэша
location ~* \.html$ {
    expires -1;
    add_header Cache-Control "no-cache";
}
```

## ETag и Last-Modified

Nginx автоматически добавляет эти заголовки для статических файлов.

```nginx
location /static/ {
    # ETag включён по умолчанию
    etag on;

    # Можно отключить если не нужен
    etag off;

    # Last-Modified добавляется автоматически
}
```

## Типы MIME

```nginx
http {
    # Загрузка типов из файла
    include /etc/nginx/mime.types;

    # Тип по умолчанию
    default_type application/octet-stream;

    # Добавление своих типов
    types {
        application/json json;
        text/plain log txt;
    }
}
```

## Полный пример конфигурации

```nginx
server {
    listen 80;
    server_name static.example.com;

    root /var/www/static;
    index index.html;

    # Оптимизации
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    # Кэш файловых дескрипторов
    open_file_cache max=1000 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;

    # Сжатие
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
    gzip_min_length 256;

    # HTML
    location ~* \.html$ {
        expires -1;
        add_header Cache-Control "no-cache";
    }

    # CSS, JS (версионированные)
    location ~* \.(css|js)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # Изображения
    location ~* \.(jpg|jpeg|png|gif|ico|svg|webp|avif)$ {
        expires 30d;
        add_header Cache-Control "public";
        access_log off;
    }

    # Шрифты
    location ~* \.(woff|woff2|ttf|otf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        add_header Access-Control-Allow-Origin "*";
        access_log off;
    }

    # Видео и аудио
    location ~* \.(mp4|webm|ogg|mp3|wav)$ {
        expires 30d;
        add_header Cache-Control "public";
    }

    # Загрузки (PDF, ZIP)
    location /downloads/ {
        alias /var/www/files/;
        autoindex on;
        expires 7d;
    }

    # Запрет доступа к скрытым файлам
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    # Fallback
    location / {
        try_files $uri $uri/ =404;
    }

    # Кастомная страница 404
    error_page 404 /404.html;
    location = /404.html {
        internal;
    }
}
```
