# Обработка HTTP-запросов в Nginx

## Фазы обработки запроса

Nginx обрабатывает каждый HTTP-запрос через последовательность фаз:

```
┌─────────────────────────────────────────────────┐
│                 HTTP Request                     │
└─────────────────────────┬───────────────────────┘
                          ▼
┌─────────────────────────────────────────────────┐
│ 1. POST_READ     - чтение заголовков запроса    │
├─────────────────────────────────────────────────┤
│ 2. SERVER_REWRITE - rewrite на уровне server    │
├─────────────────────────────────────────────────┤
│ 3. FIND_CONFIG   - поиск location               │
├─────────────────────────────────────────────────┤
│ 4. REWRITE       - rewrite на уровне location   │
├─────────────────────────────────────────────────┤
│ 5. POST_REWRITE  - после rewrite                │
├─────────────────────────────────────────────────┤
│ 6. PREACCESS     - предварительные проверки     │
│                    (limit_conn, limit_req)       │
├─────────────────────────────────────────────────┤
│ 7. ACCESS        - контроль доступа             │
│                    (allow, deny, auth_basic)     │
├─────────────────────────────────────────────────┤
│ 8. POST_ACCESS   - после проверки доступа       │
├─────────────────────────────────────────────────┤
│ 9. PRECONTENT    - try_files                    │
├─────────────────────────────────────────────────┤
│ 10. CONTENT      - генерация контента           │
│                    (proxy_pass, fastcgi, root)   │
├─────────────────────────────────────────────────┤
│ 11. LOG          - логирование                  │
└─────────────────────────────────────────────────┘
```

## Выбор виртуального хоста (Server Block)

Когда приходит запрос, nginx должен выбрать, какой `server` блок его обработает.

### Алгоритм выбора

1. **Сначала по IP:порту** (`listen` директива)
2. **Затем по имени хоста** (`server_name` директива)

```nginx
# Запрос на 192.168.1.1:80 с Host: example.com

server {
    listen 192.168.1.1:80;        # Совпадает по IP:порту
    server_name example.com;       # Совпадает по имени
}

server {
    listen 80;                     # Совпадает только по порту
    server_name example.com;
}

# Будет выбран первый server (более точный listen)
```

### Listen директива

```nginx
# Базовые варианты
listen 80;                    # Все интерфейсы, порт 80
listen 127.0.0.1:80;          # Только localhost
listen *:80;                  # Все интерфейсы (явно)
listen [::]:80;               # IPv6 все интерфейсы
listen [::1]:80;              # IPv6 localhost

# Дополнительные параметры
listen 80 default_server;     # Сервер по умолчанию
listen 443 ssl;               # HTTPS
listen 443 ssl http2;         # HTTPS + HTTP/2
listen 443 quic;              # HTTP/3 (QUIC)
listen 80 reuseport;          # Оптимизация для multi-worker
```

### Default server

Если ни один `server_name` не совпал, используется `default_server`:

```nginx
# Явный default_server
server {
    listen 80 default_server;
    server_name _;              # catch-all
    return 444;                 # Закрыть соединение без ответа
}

server {
    listen 80;
    server_name example.com;
    # ...
}
```

## Server_name — имена серверов

### Типы имён

```nginx
# 1. Точное имя
server_name example.com;
server_name example.com www.example.com;

# 2. Wildcard в начале
server_name *.example.com;     # sub.example.com, www.example.com

# 3. Wildcard в конце
server_name mail.*;            # mail.com, mail.org

# 4. Регулярное выражение (начинается с ~)
server_name ~^www\d+\.example\.com$;  # www1.example.com, www2.example.com

# 5. Пустое имя (для запросов без Host)
server_name "";

# 6. catch-all
server_name _;                 # Любое имя (не рекомендуется)
```

### Приоритеты server_name

При нескольких совпадениях выбирается в порядке приоритета:

1. **Точное имя** — `example.com`
2. **Самый длинный wildcard в начале** — `*.example.com`
3. **Самый длинный wildcard в конце** — `mail.*`
4. **Первое совпавшее регулярное выражение** (в порядке в конфиге)

```nginx
server {
    server_name example.com;              # 1-й приоритет
}

server {
    server_name *.example.com;            # 2-й приоритет
}

server {
    server_name www.*;                    # 3-й приоритет
}

server {
    server_name ~^www\.example\.(com|org)$;  # 4-й приоритет
}
```

### Regex с именованными группами

```nginx
server {
    server_name ~^(?<subdomain>.+)\.example\.com$;

    location / {
        # Используем захваченную группу
        root /var/www/sites/$subdomain;
    }
}
```

## Location — обработка путей

### Синтаксис location

```nginx
location [ = | ~ | ~* | ^~ ] uri { ... }
```

| Модификатор | Описание | Пример |
|-------------|----------|--------|
| (нет) | Префикс | `location /api/` |
| `=` | Точное совпадение | `location = /` |
| `~` | Regex (регистрозависимый) | `location ~ \.php$` |
| `~*` | Regex (регистронезависимый) | `location ~* \.(jpg\|png)$` |
| `^~` | Приоритетный префикс | `location ^~ /static/` |

### Алгоритм поиска location

```
┌────────────────────────────────────────────────┐
│ Запрос: GET /images/photo.jpg                  │
└─────────────────────┬──────────────────────────┘
                      ▼
┌────────────────────────────────────────────────┐
│ 1. Проверка точных совпадений (=)              │
│    location = /images/photo.jpg               │
│    → Если найдено, СТОП                        │
└─────────────────────┬──────────────────────────┘
                      ▼
┌────────────────────────────────────────────────┐
│ 2. Поиск самого длинного префикса              │
│    location /images/                           │
│    location /images/photo                      │
│    → Запоминаем лучший                         │
└─────────────────────┬──────────────────────────┘
                      ▼
┌────────────────────────────────────────────────┐
│ 3. Если лучший префикс имеет ^~, СТОП          │
│    location ^~ /images/                        │
└─────────────────────┬──────────────────────────┘
                      ▼
┌────────────────────────────────────────────────┐
│ 4. Проверка regex в порядке из конфига         │
│    location ~ \.jpg$                          │
│    → Если найдено совпадение, используем его   │
└─────────────────────┬──────────────────────────┘
                      ▼
┌────────────────────────────────────────────────┐
│ 5. Если regex не найден, используем            │
│    запомненный префикс                         │
└────────────────────────────────────────────────┘
```

### Примеры приоритетов

```nginx
server {
    # 1. Точное совпадение — высший приоритет
    location = / {
        # Только GET /
    }

    # 2. Приоритетный префикс — выше regex
    location ^~ /static/ {
        # /static/*, regex проверяться не будут
    }

    # 3. Regex в порядке объявления
    location ~ \.php$ {
        # *.php
    }

    location ~* \.(jpg|png|gif)$ {
        # Изображения
    }

    # 4. Обычные префиксы — по длине
    location /documents/ {
        # /documents/*
    }

    location /documents/archive/ {
        # /documents/archive/* (приоритет выше — длиннее)
    }

    # 5. Fallback
    location / {
        # Всё остальное
    }
}
```

### Тестирование location

```bash
# Запрос: /
curl http://localhost/                  # location = /

# Запрос: /static/style.css
curl http://localhost/static/style.css  # location ^~ /static/

# Запрос: /api/users.php
curl http://localhost/api/users.php     # location ~ \.php$

# Запрос: /images/photo.jpg
curl http://localhost/images/photo.jpg  # location ~* \.(jpg|png|gif)$

# Запрос: /about
curl http://localhost/about             # location /
```

## Вложенные location

```nginx
server {
    location /api/ {
        # Общие настройки для /api/*

        location /api/v1/ {
            # Специфично для /api/v1/*
            proxy_pass http://backend_v1;
        }

        location /api/v2/ {
            # Специфично для /api/v2/*
            proxy_pass http://backend_v2;
        }
    }
}
```

**Ограничения:**
- Вложенный location не может быть именованным (`@name`)
- `=` location не может содержать вложенных locations
- Regex location может содержать только regex locations

## Именованные location

Используются для внутренних переходов (error_page, try_files).

```nginx
server {
    location / {
        try_files $uri $uri/ @fallback;
    }

    location @fallback {
        proxy_pass http://backend;
    }

    error_page 404 = @notfound;

    location @notfound {
        return 404 "Page not found\n";
    }
}
```

## Практический пример

```nginx
server {
    listen 80;
    server_name example.com www.example.com;

    root /var/www/example.com/public;
    index index.html index.php;

    # Точное совпадение для главной
    location = / {
        try_files /index.html =404;
    }

    # Статика с кэшированием
    location ^~ /static/ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # PHP через FastCGI
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # API проксирование
    location /api/ {
        proxy_pass http://127.0.0.1:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Изображения
    location ~* \.(jpg|jpeg|png|gif|ico|svg|webp)$ {
        expires 7d;
        access_log off;
    }

    # Запрет доступа к скрытым файлам
    location ~ /\. {
        deny all;
    }

    # Fallback для SPA
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```
