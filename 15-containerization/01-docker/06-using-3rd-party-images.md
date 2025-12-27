# Использование сторонних Docker-образов

## Введение

Сторонние Docker-образы — это готовые к использованию образы, созданные другими разработчиками или организациями и опубликованные в публичных или приватных реестрах. Они позволяют быстро развернуть базы данных, веб-серверы, инструменты разработки и многое другое без необходимости создавать образы с нуля.

### Зачем использовать сторонние образы?

1. **Экономия времени** — не нужно писать Dockerfile для популярного ПО
2. **Проверенные решения** — официальные образы поддерживаются и обновляются
3. **Стандартизация** — единообразная конфигурация в команде
4. **Безопасность** — официальные образы регулярно сканируются на уязвимости
5. **Документация** — подробное описание использования и настройки

---

## Docker Hub — основной источник образов

[Docker Hub](https://hub.docker.com) — это крупнейший публичный реестр Docker-образов. Здесь можно найти образы практически для любого программного обеспечения.

### Поиск образов

#### Через командную строку

```bash
# Поиск образов по ключевому слову
docker search nginx

# Поиск с фильтрами
docker search --filter is-official=true nginx
docker search --filter stars=100 nginx

# Ограничение количества результатов
docker search --limit 5 postgres
```

#### Через веб-интерфейс

На сайте Docker Hub доступен расширенный поиск с фильтрами:
- По категориям (Databases, Languages, Tools и т.д.)
- По количеству скачиваний
- По дате обновления
- По наличию официального статуса

### Official Images vs Community Images

#### Official Images (Официальные образы)

Официальные образы отмечены значком "Docker Official Image" и имеют ряд преимуществ:

```bash
# Официальные образы не имеют префикса пользователя
docker pull nginx
docker pull postgres
docker pull redis
docker pull node
```

**Характеристики официальных образов:**
- Поддерживаются командой Docker или партнёрами
- Регулярно обновляются и сканируются на уязвимости
- Имеют качественную документацию
- Следуют лучшим практикам Dockerfile
- Поддерживают несколько архитектур (amd64, arm64 и др.)

#### Community Images (Сообщественные образы)

```bash
# Сообщественные образы имеют префикс с именем пользователя/организации
docker pull bitnami/postgresql
docker pull linuxserver/nginx
docker pull grafana/grafana
```

**Когда использовать сообщественные образы:**
- Нужна специфическая конфигурация
- Официальный образ не существует
- Требуется интеграция с определённой экосистемой (например, Bitnami)

### Проверка безопасности образов

#### Docker Scout

```bash
# Анализ уязвимостей образа
docker scout cves nginx:latest

# Быстрый обзор безопасности
docker scout quickview nginx:latest

# Сравнение версий
docker scout compare nginx:1.24 nginx:1.25
```

#### Проверка перед использованием

1. **Проверьте автора** — кто создал образ
2. **Дата обновления** — когда последний раз обновлялся
3. **Количество скачиваний** — популярность образа
4. **Наличие Dockerfile** — можно ли посмотреть, как собран образ
5. **Документация** — качество описания

```bash
# Просмотр информации об образе
docker inspect nginx:latest

# История слоёв образа
docker history nginx:latest
```

---

## Теги образов и версионирование

### Что такое теги?

Тег — это метка, указывающая на конкретную версию образа. Один образ может иметь несколько тегов.

```bash
# Формат: имя_образа:тег
docker pull nginx:1.25.3
docker pull nginx:1.25
docker pull nginx:latest
docker pull nginx:alpine
```

### latest vs конкретные версии

#### Тег `latest`

```bash
# По умолчанию используется тег latest
docker pull nginx
# Эквивалентно:
docker pull nginx:latest
```

**Проблемы с `latest`:**
- Не гарантирует стабильность — может измениться в любой момент
- Нет воспроизводимости сборок
- Сложно отследить, какая версия используется
- Может содержать breaking changes

#### Конкретные версии

```bash
# Полная версия (наиболее стабильно)
docker pull nginx:1.25.3

# Мажорная.минорная версия (получит патчи)
docker pull nginx:1.25

# Только мажорная версия
docker pull nginx:1
```

### Semantic Versioning в Docker

Большинство образов следуют схеме семантического версионирования:

```
MAJOR.MINOR.PATCH
  1   .  25  . 3
```

- **MAJOR** — несовместимые изменения API
- **MINOR** — новая функциональность с обратной совместимостью
- **PATCH** — исправления ошибок

#### Специальные теги

```bash
# Alpine-версии (минимальный размер)
docker pull nginx:1.25-alpine
docker pull node:20-alpine

# Slim-версии (уменьшенный размер)
docker pull python:3.12-slim

# Версии с указанием ОС
docker pull node:20-bookworm
docker pull node:20-bullseye

# Дайджесты (SHA256, максимальная точность)
docker pull nginx@sha256:abc123...
```

---

## Практические примеры использования популярных образов

### Nginx — веб-сервер

```bash
# Запуск простого веб-сервера
docker run -d \
  --name my-nginx \
  -p 8080:80 \
  nginx:1.25-alpine

# Монтирование своих статических файлов
docker run -d \
  --name my-nginx \
  -p 8080:80 \
  -v $(pwd)/html:/usr/share/nginx/html:ro \
  nginx:1.25-alpine

# Монтирование конфигурации
docker run -d \
  --name my-nginx \
  -p 8080:80 \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v $(pwd)/html:/usr/share/nginx/html:ro \
  nginx:1.25-alpine
```

#### Пример nginx.conf

```nginx
events {
    worker_connections 1024;
}

http {
    server {
        listen 80;
        server_name localhost;

        location / {
            root /usr/share/nginx/html;
            index index.html;
        }

        location /api {
            proxy_pass http://backend:3000;
        }
    }
}
```

### PostgreSQL — реляционная база данных

```bash
# Простой запуск
docker run -d \
  --name my-postgres \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -p 5432:5432 \
  postgres:16-alpine

# С сохранением данных
docker run -d \
  --name my-postgres \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=mydb \
  -v postgres_data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16-alpine

# С начальными скриптами
docker run -d \
  --name my-postgres \
  -e POSTGRES_PASSWORD=mypassword \
  -v $(pwd)/init-scripts:/docker-entrypoint-initdb.d:ro \
  -v postgres_data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16-alpine
```

#### Подключение к PostgreSQL

```bash
# Через psql внутри контейнера
docker exec -it my-postgres psql -U postgres

# Или с указанием базы данных
docker exec -it my-postgres psql -U myuser -d mydb
```

### Redis — кэш и брокер сообщений

```bash
# Простой запуск
docker run -d \
  --name my-redis \
  -p 6379:6379 \
  redis:7-alpine

# С паролем
docker run -d \
  --name my-redis \
  -p 6379:6379 \
  redis:7-alpine \
  redis-server --requirepass mysecretpassword

# С сохранением данных и конфигурацией
docker run -d \
  --name my-redis \
  -p 6379:6379 \
  -v redis_data:/data \
  -v $(pwd)/redis.conf:/usr/local/etc/redis/redis.conf:ro \
  redis:7-alpine \
  redis-server /usr/local/etc/redis/redis.conf
```

#### Подключение к Redis

```bash
# Redis CLI внутри контейнера
docker exec -it my-redis redis-cli

# С паролем
docker exec -it my-redis redis-cli -a mysecretpassword

# Проверка работы
docker exec -it my-redis redis-cli ping
# Ответ: PONG
```

### Node.js — среда выполнения JavaScript

```bash
# Запуск приложения
docker run -d \
  --name my-node-app \
  -p 3000:3000 \
  -v $(pwd):/app \
  -w /app \
  node:20-alpine \
  node server.js

# Разработка с nodemon
docker run -it \
  --name my-node-dev \
  -p 3000:3000 \
  -v $(pwd):/app \
  -w /app \
  node:20-alpine \
  sh -c "npm install && npx nodemon server.js"

# Интерактивный режим для отладки
docker run -it \
  --rm \
  -v $(pwd):/app \
  -w /app \
  node:20-alpine \
  sh
```

---

## Переменные окружения и конфигурация

### Передача переменных окружения

```bash
# Через флаг -e
docker run -d \
  -e DATABASE_URL=postgres://user:pass@host:5432/db \
  -e NODE_ENV=production \
  -e API_KEY=secret123 \
  my-app

# Через файл .env
docker run -d \
  --env-file .env \
  my-app
```

#### Пример файла .env

```env
# .env
DATABASE_URL=postgres://user:pass@host:5432/db
NODE_ENV=production
API_KEY=secret123
LOG_LEVEL=info
```

### Типичные переменные окружения популярных образов

#### PostgreSQL

| Переменная | Описание | Пример |
|------------|----------|--------|
| `POSTGRES_USER` | Имя пользователя | `myuser` |
| `POSTGRES_PASSWORD` | Пароль (обязательный) | `mysecret` |
| `POSTGRES_DB` | Имя базы данных | `mydb` |
| `PGDATA` | Путь к данным | `/var/lib/postgresql/data` |

#### MySQL

| Переменная | Описание | Пример |
|------------|----------|--------|
| `MYSQL_ROOT_PASSWORD` | Пароль root | `rootpass` |
| `MYSQL_USER` | Имя пользователя | `myuser` |
| `MYSQL_PASSWORD` | Пароль пользователя | `mypass` |
| `MYSQL_DATABASE` | Имя базы данных | `mydb` |

#### MongoDB

| Переменная | Описание | Пример |
|------------|----------|--------|
| `MONGO_INITDB_ROOT_USERNAME` | Имя администратора | `admin` |
| `MONGO_INITDB_ROOT_PASSWORD` | Пароль администратора | `adminpass` |
| `MONGO_INITDB_DATABASE` | Начальная БД | `mydb` |

### Конфигурационные файлы

Многие образы поддерживают монтирование конфигурационных файлов:

```bash
# Nginx
-v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro

# PostgreSQL
-v $(pwd)/postgresql.conf:/etc/postgresql/postgresql.conf:ro

# Redis
-v $(pwd)/redis.conf:/usr/local/etc/redis/redis.conf:ro

# MySQL
-v $(pwd)/my.cnf:/etc/mysql/my.cnf:ro
```

---

## Best Practices при использовании сторонних образов

### 1. Всегда указывайте конкретную версию

```dockerfile
# Плохо
FROM node:latest
FROM postgres

# Хорошо
FROM node:20.10-alpine
FROM postgres:16.1-alpine
```

### 2. Предпочитайте Alpine-версии

Alpine Linux значительно меньше по размеру и имеет меньшую поверхность атаки:

```bash
# Сравнение размеров
docker images | grep node
# node:20          ~1GB
# node:20-slim     ~200MB
# node:20-alpine   ~140MB
```

### 3. Используйте официальные образы когда возможно

```bash
# Предпочтительно
docker pull nginx
docker pull postgres
docker pull redis

# Только если нужна специфическая функциональность
docker pull bitnami/nginx
```

### 4. Проверяйте образы на уязвимости

```bash
# Используйте Docker Scout
docker scout cves nginx:1.25-alpine

# Или сторонние инструменты
trivy image nginx:1.25-alpine
```

### 5. Не храните секреты в образах

```bash
# Плохо — секрет в команде
docker run -e PASSWORD=secret123 myapp

# Лучше — через файл
docker run --env-file .env myapp

# Ещё лучше — через Docker secrets (Swarm) или внешний менеджер секретов
```

### 6. Используйте read-only монтирование для конфигов

```bash
# Флаг :ro предотвращает изменение файлов
docker run -v $(pwd)/config:/app/config:ro myapp
```

### 7. Ограничивайте ресурсы контейнеров

```bash
docker run -d \
  --memory="512m" \
  --cpus="1.0" \
  --name my-app \
  my-image
```

### 8. Используйте multi-stage builds для своих образов на базе сторонних

```dockerfile
# Сборка
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Продакшен
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/main.js"]
```

---

## Типичные ошибки и как их избежать

### 1. Использование `latest` в продакшене

**Проблема:**
```dockerfile
FROM node:latest  # Версия может измениться в любой момент
```

**Решение:**
```dockerfile
FROM node:20.10.0-alpine  # Конкретная версия
```

### 2. Игнорирование архитектуры

**Проблема:**
```bash
# На Mac M1/M2 некоторые образы не работают
docker run --platform linux/amd64 old-image
```

**Решение:**
Выбирайте образы с поддержкой нескольких архитектур или используйте флаг `--platform`:
```bash
docker run --platform linux/arm64 nginx:alpine
```

### 3. Потеря данных при удалении контейнера

**Проблема:**
```bash
# Данные будут потеряны при удалении контейнера
docker run -d postgres:16
docker rm -f <container_id>  # Данные потеряны!
```

**Решение:**
```bash
# Используйте volumes
docker run -d \
  -v postgres_data:/var/lib/postgresql/data \
  postgres:16
```

### 4. Запуск от root без необходимости

**Проблема:**
По умолчанию многие образы запускаются от root.

**Решение:**
```bash
# Запуск от непривилегированного пользователя
docker run -u 1000:1000 myapp
```

Или в Dockerfile:
```dockerfile
FROM node:20-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

### 5. Отсутствие health checks

**Проблема:**
Docker не знает, работает ли приложение внутри контейнера корректно.

**Решение:**
```bash
docker run -d \
  --health-cmd="curl -f http://localhost/ || exit 1" \
  --health-interval=30s \
  --health-timeout=10s \
  --health-retries=3 \
  nginx
```

### 6. Неправильное использование entrypoint скриптов

**Проблема:**
Попытка переопределить команду без понимания entrypoint.

```bash
# Не сработает как ожидается
docker run postgres:16 --help
```

**Решение:**
```bash
# Просмотр entrypoint образа
docker inspect --format='{{.Config.Entrypoint}}' postgres:16

# Переопределение entrypoint
docker run --entrypoint="" postgres:16 cat /etc/os-release
```

### 7. Игнорирование документации образа

**Совет:** Всегда читайте документацию на Docker Hub перед использованием образа. Там описаны:
- Доступные переменные окружения
- Пути для монтирования volumes
- Примеры использования
- Известные ограничения

---

## Полезные команды для работы со сторонними образами

```bash
# Скачать образ без запуска
docker pull nginx:1.25-alpine

# Посмотреть все скачанные образы
docker images

# Удалить образ
docker rmi nginx:1.25-alpine

# Удалить неиспользуемые образы
docker image prune

# Посмотреть историю образа
docker history nginx:1.25-alpine

# Информация об образе
docker inspect nginx:1.25-alpine

# Посмотреть переменные окружения образа
docker inspect --format='{{range .Config.Env}}{{println .}}{{end}}' nginx:1.25-alpine

# Посмотреть exposed порты
docker inspect --format='{{.Config.ExposedPorts}}' nginx:1.25-alpine
```

---

## Заключение

Использование сторонних Docker-образов — это эффективный способ быстро развернуть необходимую инфраструктуру. Ключевые принципы:

1. **Используйте официальные образы** когда это возможно
2. **Фиксируйте версии** для воспроизводимости
3. **Проверяйте безопасность** перед использованием
4. **Читайте документацию** каждого образа
5. **Используйте volumes** для персистентных данных
6. **Настраивайте через переменные окружения** вместо изменения образов
