# Кэширование в Nginx

## Proxy Cache

Nginx может кэшировать ответы от upstream-серверов.

### Базовая настройка

```nginx
http {
    # Определение кэша (в http контексте)
    proxy_cache_path /var/cache/nginx/proxy
        levels=1:2
        keys_zone=my_cache:10m
        max_size=10g
        inactive=60m
        use_temp_path=off;

    server {
        location /api/ {
            proxy_pass http://backend;

            # Включение кэша
            proxy_cache my_cache;

            # Время кэширования по кодам ответа
            proxy_cache_valid 200 302 10m;
            proxy_cache_valid 404 1m;
            proxy_cache_valid any 5m;
        }
    }
}
```

### Параметры proxy_cache_path

| Параметр | Описание |
|----------|----------|
| `path` | Директория для кэша |
| `levels=1:2` | Уровни вложенности директорий (1 символ:2 символа) |
| `keys_zone=name:size` | Имя и размер зоны shared memory для ключей |
| `max_size=size` | Максимальный размер кэша |
| `inactive=time` | Время до удаления неиспользуемых данных |
| `use_temp_path=off` | Писать сразу в кэш (не через temp) |
| `loader_files=n` | Файлов за итерацию загрузки |
| `loader_sleep=time` | Пауза между итерациями |
| `loader_threshold=time` | Макс время итерации |

### Структура кэша на диске

```
/var/cache/nginx/proxy/
├── 0/
│   └── 1a/
│       └── 2d4f8a0b1c3e5f7a9b0c1d2e3f4a5b6c
├── 1/
│   └── 2b/
│       └── ...
```

## Ключ кэша

По умолчанию: `$scheme$proxy_host$request_uri`

```nginx
# Кастомный ключ
proxy_cache_key "$scheme$request_method$host$request_uri";

# Включить cookies в ключ
proxy_cache_key "$scheme$host$request_uri$cookie_sessionid";

# Исключить query параметры для некоторых запросов
map $request_uri $cache_uri {
    ~^(/api/products)(\?.*)?$ $1;
    default $request_uri;
}

proxy_cache_key "$scheme$host$cache_uri";
```

## Управление кэшированием

### Условия кэширования

```nginx
location /api/ {
    proxy_pass http://backend;
    proxy_cache my_cache;

    # Не кэшировать если есть cookie
    proxy_cache_bypass $cookie_nocache $arg_nocache;

    # Не возвращать из кэша
    proxy_no_cache $cookie_nocache $arg_nocache;

    # Не кэшировать POST запросы
    proxy_cache_methods GET HEAD;
}
```

### Map для управления кэшем

```nginx
map $request_method $skip_cache {
    POST 1;
    default 0;
}

map $cookie_auth $skip_cache {
    ~.+ 1;  # Если есть auth cookie — не кэшировать
    default $skip_cache;
}

location /api/ {
    proxy_pass http://backend;
    proxy_cache my_cache;
    proxy_no_cache $skip_cache;
    proxy_cache_bypass $skip_cache;
}
```

### proxy_cache_valid

```nginx
location /api/ {
    proxy_pass http://backend;
    proxy_cache my_cache;

    # По кодам ответа
    proxy_cache_valid 200 301 302 10m;
    proxy_cache_valid 404 1m;
    proxy_cache_valid 500 502 503 0;  # Не кэшировать ошибки
    proxy_cache_valid any 5m;  # Всё остальное
}
```

## Stale Cache

Возврат устаревшего кэша при проблемах с backend:

```nginx
location /api/ {
    proxy_pass http://backend;
    proxy_cache my_cache;

    # Возвращать stale при ошибках
    proxy_cache_use_stale error timeout http_500 http_502 http_503 http_504;

    # Обновлять кэш в фоне
    proxy_cache_background_update on;

    # Один запрос обновляет кэш, остальные получают stale
    proxy_cache_lock on;
    proxy_cache_lock_timeout 5s;
    proxy_cache_lock_age 5s;
}
```

### Параметры stale

| Значение | Когда возвращать stale |
|----------|------------------------|
| `error` | Ошибка соединения |
| `timeout` | Таймаут соединения |
| `invalid_header` | Некорректный ответ |
| `updating` | Кэш обновляется |
| `http_500`, `http_502`, etc. | HTTP ошибки |

## Cache Bypass

### Через заголовки

```nginx
location /api/ {
    proxy_pass http://backend;
    proxy_cache my_cache;

    # Bypass по заголовку X-No-Cache
    proxy_cache_bypass $http_x_no_cache;

    # Bypass по query параметру
    proxy_cache_bypass $arg_nocache;
}
```

### Purge кэша

```nginx
# Определение метода PURGE
map $request_method $purge_method {
    PURGE 1;
    default 0;
}

server {
    location /api/ {
        proxy_pass http://backend;
        proxy_cache my_cache;
        proxy_cache_key "$scheme$host$request_uri";

        # Разрешить purge только с localhost
        if ($purge_method) {
            set $purge_allowed 0;
            if ($remote_addr = 127.0.0.1) {
                set $purge_allowed 1;
            }
            if ($purge_allowed = 0) {
                return 403;
            }
            # Для purge нужен модуль ngx_cache_purge
            # или Nginx Plus
        }
    }
}
```

```bash
# Очистка конкретного URL
curl -X PURGE http://localhost/api/users

# Очистка всего кэша (через скрипт)
rm -rf /var/cache/nginx/proxy/*
nginx -s reload
```

## FastCGI Cache (PHP)

```nginx
http {
    fastcgi_cache_path /var/cache/nginx/fastcgi
        levels=1:2
        keys_zone=php_cache:10m
        max_size=1g
        inactive=60m;

    server {
        location ~ \.php$ {
            fastcgi_pass unix:/run/php/php-fpm.sock;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;

            # Кэширование
            fastcgi_cache php_cache;
            fastcgi_cache_key "$scheme$request_method$host$request_uri";
            fastcgi_cache_valid 200 10m;
            fastcgi_cache_use_stale error timeout;

            # Не кэшировать для залогиненных
            fastcgi_no_cache $cookie_logged_in;
            fastcgi_cache_bypass $cookie_logged_in;
        }
    }
}
```

## Мониторинг кэша

### Переменная $upstream_cache_status

```nginx
location /api/ {
    proxy_pass http://backend;
    proxy_cache my_cache;

    # Добавить статус в заголовок ответа
    add_header X-Cache-Status $upstream_cache_status;
}
```

| Значение | Описание |
|----------|----------|
| `MISS` | Не найдено в кэше, запрос к backend |
| `HIT` | Возвращено из кэша |
| `EXPIRED` | Кэш истёк, запрос к backend |
| `STALE` | Возвращён устаревший кэш |
| `UPDATING` | Кэш обновляется, возвращён stale |
| `REVALIDATED` | Кэш валидирован через If-Modified-Since |
| `BYPASS` | Кэш пропущен (bypass) |

### Логирование статуса кэша

```nginx
log_format cache '$remote_addr - [$time_local] '
                 '"$request" $status $body_bytes_sent '
                 '"$upstream_cache_status" '
                 'rt=$request_time uct=$upstream_connect_time '
                 'uht=$upstream_header_time urt=$upstream_response_time';

access_log /var/log/nginx/cache.log cache;
```

### Анализ эффективности

```bash
# Подсчёт HIT/MISS
grep -o 'X-Cache-Status: [A-Z]*' /var/log/nginx/access.log | sort | uniq -c

# Или через awk
awk '{print $NF}' /var/log/nginx/cache.log | sort | uniq -c
```

## Слоёный кэш

```nginx
# Кэш в памяти (маленький, быстрый)
proxy_cache_path /dev/shm/nginx/cache
    levels=1:2
    keys_zone=memory_cache:10m
    max_size=100m
    inactive=10m;

# Кэш на диске (большой)
proxy_cache_path /var/cache/nginx
    levels=1:2
    keys_zone=disk_cache:100m
    max_size=10g
    inactive=60m;

server {
    location /api/hot/ {
        proxy_pass http://backend;
        proxy_cache memory_cache;  # В памяти
        proxy_cache_valid 200 5m;
    }

    location /api/ {
        proxy_pass http://backend;
        proxy_cache disk_cache;  # На диске
        proxy_cache_valid 200 1h;
    }
}
```

## Полный пример

```nginx
http {
    # Определение кэша
    proxy_cache_path /var/cache/nginx/api
        levels=1:2
        keys_zone=api_cache:50m
        max_size=5g
        inactive=24h
        use_temp_path=off;

    # Map для bypass
    map $cookie_auth $skip_cache {
        ~.+ 1;
        default 0;
    }

    map $request_method $skip_cache {
        POST 1;
        PUT 1;
        DELETE 1;
        default $skip_cache;
    }

    server {
        listen 80;
        server_name api.example.com;

        location /api/ {
            proxy_pass http://backend;

            # Кэширование
            proxy_cache api_cache;
            proxy_cache_key "$scheme$host$request_uri";
            proxy_cache_valid 200 10m;
            proxy_cache_valid 404 1m;

            # Bypass для авторизованных и мутирующих запросов
            proxy_cache_bypass $skip_cache;
            proxy_no_cache $skip_cache;

            # Stale при ошибках
            proxy_cache_use_stale error timeout updating http_500 http_502;
            proxy_cache_background_update on;
            proxy_cache_lock on;

            # Минимальное время использования для кэширования
            proxy_cache_min_uses 2;

            # Статус кэша в заголовке
            add_header X-Cache-Status $upstream_cache_status;

            # Заголовки
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # Статика с агрессивным кэшированием
        location /static/ {
            proxy_pass http://backend;
            proxy_cache api_cache;
            proxy_cache_valid 200 30d;
            proxy_cache_use_stale error timeout;
            add_header X-Cache-Status $upstream_cache_status;
        }
    }
}
```
