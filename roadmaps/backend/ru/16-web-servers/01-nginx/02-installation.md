# Установка и запуск Nginx

## Установка из пакетов

### Ubuntu / Debian

```bash
# Обновление списка пакетов
sudo apt update

# Установка nginx
sudo apt install nginx

# Проверка версии
nginx -v
```

Для установки последней версии (mainline) используйте официальный репозиторий:

```bash
# Установка зависимостей
sudo apt install curl gnupg2 ca-certificates lsb-release

# Добавление ключа репозитория
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
    | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null

# Добавление репозитория (mainline)
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
    http://nginx.org/packages/mainline/ubuntu `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list

# Установка
sudo apt update
sudo apt install nginx
```

### CentOS / RHEL / Fedora

```bash
# CentOS/RHEL 8+
sudo dnf install nginx

# CentOS/RHEL 7
sudo yum install epel-release
sudo yum install nginx
```

### macOS (Homebrew)

```bash
brew install nginx

# Nginx будет доступен по пути /opt/homebrew/etc/nginx/
```

### Alpine Linux

```bash
apk add nginx
```

## Сборка из исходников

Сборка нужна, когда требуются специфические модули или оптимизации.

```bash
# Установка зависимостей (Ubuntu)
sudo apt install build-essential libpcre3 libpcre3-dev \
    zlib1g zlib1g-dev libssl-dev libgd-dev

# Скачивание исходников
wget https://nginx.org/download/nginx-1.26.2.tar.gz
tar -xzf nginx-1.26.2.tar.gz
cd nginx-1.26.2

# Конфигурация
./configure \
    --prefix=/etc/nginx \
    --sbin-path=/usr/sbin/nginx \
    --conf-path=/etc/nginx/nginx.conf \
    --error-log-path=/var/log/nginx/error.log \
    --http-log-path=/var/log/nginx/access.log \
    --pid-path=/var/run/nginx.pid \
    --with-http_ssl_module \
    --with-http_v2_module \
    --with-http_realip_module \
    --with-http_gzip_static_module

# Сборка и установка
make
sudo make install
```

### Полезные опции ./configure

| Опция | Описание |
|-------|----------|
| `--with-http_ssl_module` | Поддержка HTTPS |
| `--with-http_v2_module` | Поддержка HTTP/2 |
| `--with-http_v3_module` | Поддержка HTTP/3 (QUIC) |
| `--with-http_realip_module` | Получение реального IP клиента |
| `--with-http_gzip_static_module` | Раздача предварительно сжатых файлов |
| `--with-stream` | TCP/UDP проксирование |
| `--add-module=/path/to/module` | Добавление стороннего модуля |

## Структура директорий

```
/etc/nginx/                  # Конфигурация
├── nginx.conf               # Главный конфиг
├── mime.types               # MIME-типы
├── conf.d/                  # Дополнительные конфиги
│   └── default.conf
├── sites-available/         # Доступные сайты (Debian/Ubuntu)
├── sites-enabled/           # Включённые сайты (симлинки)
├── snippets/                # Переиспользуемые фрагменты
└── modules-enabled/         # Загружаемые модули

/var/log/nginx/              # Логи
├── access.log               # Логи доступа
└── error.log                # Логи ошибок

/var/www/                    # Веб-контент (по умолчанию)
└── html/
    └── index.html

/usr/share/nginx/            # Статические файлы nginx
└── html/

/var/run/nginx.pid           # PID файл процесса
```

## Параметры командной строки

```bash
# Основные команды
nginx                        # Запуск nginx
nginx -t                     # Проверка конфигурации
nginx -T                     # Проверка + вывод конфига
nginx -s signal              # Отправка сигнала

# Сигналы (-s)
nginx -s stop                # Немедленная остановка
nginx -s quit                # Graceful shutdown
nginx -s reload              # Перезагрузка конфига
nginx -s reopen              # Переоткрытие лог-файлов

# Другие опции
nginx -c /path/to/nginx.conf # Использовать другой конфиг
nginx -g "daemon off;"       # Глобальные директивы
nginx -p /path/prefix        # Установить prefix path
nginx -V                     # Версия + параметры сборки
```

### Примеры использования

```bash
# Проверка синтаксиса перед reload
sudo nginx -t && sudo nginx -s reload

# Запуск в foreground (для Docker)
nginx -g "daemon off;"

# Проверка какие модули включены
nginx -V 2>&1 | tr ' ' '\n' | grep module
```

## Управление через Systemd

```bash
# Управление сервисом
sudo systemctl start nginx    # Запуск
sudo systemctl stop nginx     # Остановка
sudo systemctl restart nginx  # Перезапуск
sudo systemctl reload nginx   # Перезагрузка конфига
sudo systemctl status nginx   # Статус

# Автозапуск
sudo systemctl enable nginx   # Включить автозапуск
sudo systemctl disable nginx  # Выключить автозапуск

# Просмотр логов
sudo journalctl -u nginx      # Все логи nginx
sudo journalctl -u nginx -f   # Следить за логами
```

### Файл systemd unit

`/lib/systemd/system/nginx.service`:

```ini
[Unit]
Description=The NGINX HTTP and reverse proxy server
After=syslog.target network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t
ExecStart=/usr/sbin/nginx
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

## Проверка работоспособности

```bash
# Проверка что nginx запущен
ps aux | grep nginx

# Проверка прослушиваемых портов
ss -tlnp | grep nginx

# HTTP-запрос к localhost
curl -I http://localhost

# Ответ должен быть примерно таким:
# HTTP/1.1 200 OK
# Server: nginx/1.26.2
# ...
```

## Docker

```dockerfile
# Минимальный Dockerfile
FROM nginx:alpine

COPY nginx.conf /etc/nginx/nginx.conf
COPY html/ /usr/share/nginx/html/

EXPOSE 80 443

CMD ["nginx", "-g", "daemon off;"]
```

```bash
# Запуск официального образа
docker run -d -p 80:80 --name nginx nginx:alpine

# С монтированием конфигов
docker run -d -p 80:80 \
    -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
    -v $(pwd)/html:/usr/share/nginx/html:ro \
    nginx:alpine
```

## Проверка установки

После установки выполните:

```bash
# 1. Проверка версии
nginx -v

# 2. Проверка конфигурации
sudo nginx -t

# 3. Запуск nginx
sudo systemctl start nginx

# 4. Проверка статуса
sudo systemctl status nginx

# 5. Открыть в браузере http://localhost
```

Вы должны увидеть страницу "Welcome to nginx!".
