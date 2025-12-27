# Nginx

Полное руководство по веб-серверу Nginx на основе [официальной документации](https://nginx.org/ru/docs/).

## Содержание

- [x] [Введение в Nginx](./01-introduction.md) — архитектура, история, сравнение с Apache
- [x] [Установка и запуск](./02-installation.md) — установка, структура директорий, команды
- [x] [Основы конфигурации](./03-configuration.md) — контексты, директивы, переменные
- [x] [Обработка HTTP-запросов](./04-http-processing.md) — server blocks, location, приоритеты
- [x] [Статический контент](./05-static-content.md) — root/alias, try_files, кэширование
- [x] [Проксирование](./06-proxy.md) — reverse proxy, FastCGI, WebSocket, gRPC
- [x] [Балансировка нагрузки](./07-load-balancing.md) — upstream, методы балансировки
- [x] [Безопасность и HTTPS](./08-security-https.md) — SSL/TLS, Let's Encrypt, HTTP/2, security headers
- [x] [Кэширование](./09-caching.md) — proxy cache, ключи, stale cache
- [x] [Логирование и мониторинг](./10-logging.md) — форматы логов, JSON, stub_status
- [x] [Оптимизация производительности](./11-performance.md) — workers, gzip, буферы, таймауты
- [x] [Продвинутые темы](./12-advanced.md) — rewrite, map, geo, njs

## Быстрый старт

```bash
# Установка (Ubuntu)
sudo apt install nginx

# Проверка конфигурации
sudo nginx -t

# Запуск
sudo systemctl start nginx

# Статус
sudo systemctl status nginx
```

## Базовая конфигурация

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

## Полезные ссылки

- [Официальная документация](https://nginx.org/ru/docs/)
- [Nginx Wiki](https://wiki.nginx.org/)
- [Nginx Config Generator](https://nginxconfig.io/)
- [Mozilla SSL Configuration](https://ssl-config.mozilla.org/)
