# Основы Docker

Docker — это платформа для разработки, доставки и запуска приложений в контейнерах. Контейнеры позволяют упаковать приложение со всеми его зависимостями в изолированную среду, которая работает одинаково на любом хосте.

---

## 1. Работа с образами (Images)

**Образ (Image)** — это шаблон, из которого создаются контейнеры. Образ содержит файловую систему, зависимости, конфигурации и метаданные.

### 1.1 Загрузка образов (docker pull)

```bash
# Загрузить образ из Docker Hub
docker pull nginx

# Загрузить конкретную версию (тег)
docker pull nginx:1.25

# Загрузить из другого registry
docker pull gcr.io/google-containers/busybox

# Загрузить все теги образа
docker pull --all-tags ubuntu
```

### 1.2 Просмотр локальных образов (docker images)

```bash
# Список всех образов
docker images

# Пример вывода:
# REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
# nginx        latest    a6bd71f48f68   2 days ago     187MB
# python       3.11      7d5b7c2e9d8a   1 week ago     1.01GB
# node         20        abc123def456   3 days ago     1.1GB

# Показать только ID образов
docker images -q

# Фильтрация по имени
docker images nginx

# Показать все образы включая промежуточные слои
docker images -a

# Фильтрация с флагом --filter
docker images --filter "dangling=true"  # "висячие" образы без тега
```

### 1.3 Удаление образов (docker rmi)

```bash
# Удалить образ по имени
docker rmi nginx

# Удалить образ по ID
docker rmi a6bd71f48f68

# Удалить несколько образов
docker rmi nginx python:3.11 node:20

# Принудительное удаление (если образ используется)
docker rmi -f nginx

# Удалить все неиспользуемые образы
docker image prune

# Удалить ВСЕ образы (осторожно!)
docker rmi $(docker images -q)
```

### 1.4 Сборка образов (docker build)

```bash
# Собрать образ из Dockerfile в текущей директории
docker build .

# Собрать с указанием имени и тега
docker build -t myapp:1.0 .

# Собрать из конкретного Dockerfile
docker build -f Dockerfile.prod -t myapp:prod .

# Собрать с передачей аргументов сборки
docker build --build-arg VERSION=1.0 -t myapp .

# Сборка без использования кеша
docker build --no-cache -t myapp .

# Мультиплатформенная сборка
docker build --platform linux/amd64,linux/arm64 -t myapp .
```

### 1.5 Тегирование образов (docker tag)

```bash
# Добавить тег к существующему образу
docker tag myapp:latest myapp:v1.0

# Тегирование для отправки в registry
docker tag myapp:latest username/myapp:latest
docker tag myapp:latest registry.example.com/myapp:v1.0
```

### 1.6 Отправка образов в registry (docker push)

```bash
# Сначала нужно авторизоваться
docker login
docker login registry.example.com

# Отправить образ в Docker Hub
docker push username/myapp:latest

# Отправить в приватный registry
docker push registry.example.com/myapp:v1.0

# Отправить все теги образа
docker push --all-tags username/myapp
```

### 1.7 Понимание слоёв образа

Образ Docker состоит из **слоёв (layers)**. Каждая инструкция в Dockerfile создаёт новый слой:

```dockerfile
FROM ubuntu:22.04          # Слой 1: базовый образ
RUN apt-get update         # Слой 2: обновление пакетов
RUN apt-get install -y vim # Слой 3: установка vim
COPY app.py /app/          # Слой 4: копирование файлов
```

**Особенности слоёв:**
- Слои кешируются и переиспользуются между образами
- Изменение слоя инвалидирует все последующие слои
- Только верхний слой контейнера доступен для записи
- Размер образа = сумма всех слоёв

```bash
# Посмотреть историю слоёв образа
docker history nginx

# Пример вывода:
# IMAGE          CREATED       CREATED BY                                      SIZE
# a6bd71f48f68   2 days ago    CMD ["nginx" "-g" "daemon off;"]                0B
# <missing>      2 days ago    EXPOSE map[80/tcp:{}]                           0B
# <missing>      2 days ago    STOPSIGNAL SIGQUIT                              0B
# <missing>      2 days ago    RUN /bin/sh -c set -x && addgroup...            61.1MB

# Детальная информация об образе
docker inspect nginx
```

---

## 2. Работа с контейнерами (Containers)

**Контейнер** — это запущенный экземпляр образа. Это изолированный процесс со своей файловой системой, сетью и ресурсами.

### 2.1 Запуск контейнеров (docker run)

```bash
# Базовый запуск
docker run nginx

# Важные флаги:

# -d (detach) — запуск в фоновом режиме
docker run -d nginx

# -it (interactive + tty) — интерактивный режим
docker run -it ubuntu bash
docker run -it python:3.11 python

# -p (port) — проброс портов (host:container)
docker run -d -p 8080:80 nginx           # localhost:8080 → container:80
docker run -d -p 3000:3000 node-app      # одинаковые порты
docker run -d -p 127.0.0.1:8080:80 nginx # привязка к конкретному IP

# -v (volume) — монтирование томов
docker run -v /host/path:/container/path nginx      # bind mount
docker run -v myvolume:/data nginx                   # named volume
docker run -v $(pwd)/config:/etc/nginx/conf.d nginx # текущая директория

# --name — задать имя контейнеру
docker run -d --name my-nginx nginx
docker run -d --name web-server -p 80:80 nginx

# -e (environment) — переменные окружения
docker run -e MYSQL_ROOT_PASSWORD=secret mysql
docker run -e NODE_ENV=production -e PORT=3000 node-app
docker run --env-file .env myapp  # из файла

# --rm — автоматическое удаление после остановки
docker run --rm ubuntu echo "Hello"
docker run --rm -it python:3.11 python -c "print('test')"

# Комбинация флагов — типичные примеры:
docker run -d --name nginx-web -p 8080:80 -v ./html:/usr/share/nginx/html nginx

docker run -d \
  --name postgres-db \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=mydb \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:15

docker run -d \
  --name redis-cache \
  -p 6379:6379 \
  -v redis-data:/data \
  --rm \
  redis:7 redis-server --appendonly yes
```

### 2.2 Просмотр контейнеров (docker ps)

```bash
# Показать запущенные контейнеры
docker ps

# Пример вывода:
# CONTAINER ID   IMAGE   COMMAND                  CREATED         STATUS         PORTS                  NAMES
# abc123def456   nginx   "/docker-entrypoint.…"   5 minutes ago   Up 5 minutes   0.0.0.0:8080->80/tcp   my-nginx

# Показать ВСЕ контейнеры (включая остановленные)
docker ps -a

# Показать только ID контейнеров
docker ps -q
docker ps -aq  # все контейнеры, только ID

# Показать последний созданный контейнер
docker ps -l

# Показать размер контейнеров
docker ps -s

# Фильтрация
docker ps --filter "status=exited"
docker ps --filter "name=nginx"
docker ps --filter "ancestor=ubuntu"

# Форматированный вывод
docker ps --format "{{.ID}}: {{.Names}} ({{.Status}})"
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### 2.3 Управление жизненным циклом контейнеров

```bash
# Остановка контейнера (SIGTERM, затем SIGKILL)
docker stop my-nginx
docker stop abc123def456  # по ID
docker stop $(docker ps -q)  # остановить все

# Остановка с таймаутом
docker stop -t 30 my-nginx  # ждать 30 секунд перед SIGKILL

# Запуск остановленного контейнера
docker start my-nginx
docker start -a my-nginx  # с attach к stdout

# Перезапуск контейнера
docker restart my-nginx
docker restart -t 10 my-nginx  # с таймаутом

# Принудительная остановка (SIGKILL)
docker kill my-nginx
```

### 2.4 Удаление контейнеров (docker rm)

```bash
# Удалить остановленный контейнер
docker rm my-nginx

# Удалить несколько контейнеров
docker rm container1 container2 container3

# Принудительное удаление (работающего контейнера)
docker rm -f my-nginx

# Удалить контейнер и его анонимные тома
docker rm -v my-nginx

# Удалить все остановленные контейнеры
docker container prune

# Удалить все контейнеры (осторожно!)
docker rm -f $(docker ps -aq)
```

### 2.5 Выполнение команд в контейнере (docker exec)

```bash
# Выполнить команду в работающем контейнере
docker exec my-nginx ls -la

# Интерактивный shell в контейнере
docker exec -it my-nginx bash
docker exec -it my-nginx sh  # если bash недоступен

# Выполнить команду от другого пользователя
docker exec -u root my-nginx whoami
docker exec -u 1000:1000 my-nginx id

# Установить переменные окружения
docker exec -e MY_VAR=value my-nginx env

# Работа в определённой директории
docker exec -w /var/log my-nginx ls -la

# Примеры практического использования:
docker exec -it postgres-db psql -U postgres
docker exec -it redis-cache redis-cli
docker exec my-app python manage.py migrate
```

### 2.6 Подключение к контейнеру (docker attach)

```bash
# Подключиться к stdin/stdout контейнера
docker attach my-nginx

# ВАЖНО: Ctrl+C остановит контейнер!
# Для отключения без остановки: Ctrl+P, Ctrl+Q

# Подключиться без stdin (только чтение)
docker attach --no-stdin my-nginx
```

**Разница между exec и attach:**
- `exec` — создаёт новый процесс внутри контейнера
- `attach` — подключается к главному процессу (PID 1)

### 2.7 Просмотр логов (docker logs)

```bash
# Показать логи контейнера
docker logs my-nginx

# Следить за логами в реальном времени
docker logs -f my-nginx

# Показать последние N строк
docker logs --tail 100 my-nginx

# Логи с временными метками
docker logs -t my-nginx

# Логи за определённый период
docker logs --since 2h my-nginx      # за последние 2 часа
docker logs --since "2024-01-01" my-nginx
docker logs --until "2024-01-01T12:00:00" my-nginx

# Комбинация флагов
docker logs -f --tail 50 -t my-nginx
```

---

## 3. Основы Dockerfile

**Dockerfile** — это текстовый файл с инструкциями для сборки образа.

### 3.1 FROM — базовый образ

```dockerfile
# Указание базового образа (обязательная первая инструкция)
FROM ubuntu:22.04
FROM python:3.11-slim
FROM node:20-alpine

# Multi-stage build
FROM golang:1.21 AS builder
# ... сборка ...
FROM alpine:3.18
COPY --from=builder /app/binary /app/

# Минимальный образ
FROM scratch
```

### 3.2 RUN — выполнение команд

```dockerfile
# Выполнение shell-команды
RUN apt-get update

# Цепочка команд (уменьшает количество слоёв)
RUN apt-get update && \
    apt-get install -y \
        curl \
        vim \
        git && \
    rm -rf /var/lib/apt/lists/*

# Exec-форма (не использует shell)
RUN ["apt-get", "install", "-y", "nginx"]

# Установка пакетов Python
RUN pip install --no-cache-dir -r requirements.txt

# Установка npm зависимостей
RUN npm ci --only=production
```

### 3.3 COPY и ADD — копирование файлов

```dockerfile
# COPY — копирование локальных файлов
COPY app.py /app/
COPY . /app
COPY package*.json ./
COPY --chown=user:group files/ /app/

# ADD — расширенное копирование (с распаковкой архивов и URL)
ADD archive.tar.gz /app/         # автоматически распаковывает
ADD https://example.com/file.txt /app/  # скачивает из URL

# ВАЖНО: предпочитайте COPY вместо ADD
# ADD используйте только для распаковки архивов
```

### 3.4 WORKDIR — рабочая директория

```dockerfile
# Установка рабочей директории
WORKDIR /app

# Последующие команды выполняются в /app
COPY . .
RUN npm install

# Можно использовать несколько раз
WORKDIR /app/src
RUN make build
```

### 3.5 ENV — переменные окружения

```dockerfile
# Установка переменных окружения
ENV NODE_ENV=production
ENV PORT=3000

# Несколько переменных
ENV APP_HOME=/app \
    APP_USER=appuser \
    PATH="$PATH:/app/bin"

# Использование в других инструкциях
WORKDIR $APP_HOME
```

### 3.6 ARG — аргументы сборки

```dockerfile
# Определение аргумента
ARG VERSION=latest
ARG BASE_IMAGE=python:3.11

# Использование аргумента
FROM ${BASE_IMAGE}
RUN echo "Building version: ${VERSION}"

# Аргумент только для сборки (не доступен в runtime)
ARG BUILD_DATE
LABEL build_date=$BUILD_DATE

# Передача при сборке:
# docker build --build-arg VERSION=1.0 .
```

**Разница между ARG и ENV:**
- `ARG` — доступен только во время сборки
- `ENV` — доступен и при сборке, и при запуске контейнера

### 3.7 EXPOSE — объявление портов

```dockerfile
# Документирование портов (не пробрасывает автоматически!)
EXPOSE 80
EXPOSE 443
EXPOSE 3000/tcp
EXPOSE 53/udp

# Несколько портов
EXPOSE 80 443

# Для реального проброса используйте docker run -p
```

### 3.8 CMD — команда по умолчанию

```dockerfile
# Shell-форма
CMD python app.py

# Exec-форма (рекомендуется)
CMD ["python", "app.py"]
CMD ["nginx", "-g", "daemon off;"]

# CMD можно переопределить при запуске:
# docker run myimage другая_команда
```

### 3.9 ENTRYPOINT — точка входа

```dockerfile
# Exec-форма (рекомендуется)
ENTRYPOINT ["python"]

# С CMD как аргументы по умолчанию
ENTRYPOINT ["python"]
CMD ["app.py"]
# Результат: python app.py
# docker run myimage другой.py → python другой.py

# Shell-форма
ENTRYPOINT python app.py
```

### 3.10 Разница между CMD и ENTRYPOINT

| Аспект | CMD | ENTRYPOINT |
|--------|-----|------------|
| Переопределение | Легко заменяется аргументами docker run | Нужен флаг `--entrypoint` |
| Назначение | Аргументы по умолчанию | Основная команда |
| Комбинация | Может служить аргументами для ENTRYPOINT | Выполняется всегда |

```dockerfile
# Пример 1: только CMD
FROM ubuntu
CMD ["echo", "Hello"]
# docker run myimage → "Hello"
# docker run myimage echo "World" → "World"

# Пример 2: только ENTRYPOINT
FROM ubuntu
ENTRYPOINT ["echo"]
# docker run myimage → ""
# docker run myimage "Hello" → "Hello"

# Пример 3: ENTRYPOINT + CMD (рекомендуемый паттерн)
FROM ubuntu
ENTRYPOINT ["echo"]
CMD ["Hello"]
# docker run myimage → "Hello"
# docker run myimage "World" → "World"
```

---

## 4. Примеры Dockerfile

### 4.1 Python приложение (Flask)

```dockerfile
# Dockerfile для Flask приложения
FROM python:3.11-slim

# Метаданные
LABEL maintainer="developer@example.com"
LABEL version="1.0"

# Переменные окружения
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Создание пользователя (не root)
RUN groupadd --gid 1000 appgroup && \
    useradd --uid 1000 --gid appgroup --shell /bin/bash --create-home appuser

# Рабочая директория
WORKDIR /app

# Установка зависимостей (отдельный слой для кеширования)
COPY requirements.txt .
RUN pip install -r requirements.txt

# Копирование приложения
COPY --chown=appuser:appgroup . .

# Переключение на непривилегированного пользователя
USER appuser

# Объявление порта
EXPOSE 5000

# Команда запуска
CMD ["python", "-m", "flask", "run", "--host=0.0.0.0"]
```

```python
# requirements.txt
Flask==3.0.0
gunicorn==21.2.0
```

```python
# app.py
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello, Docker!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### 4.2 Python приложение (FastAPI) с multi-stage build

```dockerfile
# Multi-stage build для FastAPI
# Стадия 1: сборка зависимостей
FROM python:3.11-slim AS builder

WORKDIR /app

# Установка build-зависимостей
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc && \
    rm -rf /var/lib/apt/lists/*

# Создание виртуального окружения
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Установка зависимостей
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt


# Стадия 2: финальный образ
FROM python:3.11-slim

# Копирование виртуального окружения из builder
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Создание пользователя
RUN useradd --create-home appuser
WORKDIR /app
USER appuser

# Копирование приложения
COPY --chown=appuser:appuser . .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 4.3 Node.js приложение (Express)

```dockerfile
# Dockerfile для Node.js приложения
FROM node:20-alpine

# Создание директории приложения
WORKDIR /app

# Переменные окружения
ENV NODE_ENV=production

# Копирование package.json для кеширования слоя зависимостей
COPY package*.json ./

# Установка зависимостей (только production)
RUN npm ci --only=production && \
    npm cache clean --force

# Копирование исходного кода
COPY . .

# Непривилегированный пользователь (встроен в node-образы)
USER node

# Объявление порта
EXPOSE 3000

# Healthcheck
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://localhost:3000/health || exit 1

# Запуск приложения
CMD ["node", "server.js"]
```

```javascript
// server.js
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.json({ message: 'Hello, Docker!' });
});

app.get('/health', (req, res) => {
  res.status(200).send('OK');
});

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### 4.4 Node.js с multi-stage build

```dockerfile
# Multi-stage build для Node.js с TypeScript
# Стадия 1: сборка
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build


# Стадия 2: production
FROM node:20-alpine

WORKDIR /app

ENV NODE_ENV=production

COPY package*.json ./
RUN npm ci --only=production && \
    npm cache clean --force

# Копирование скомпилированного кода
COPY --from=builder /app/dist ./dist

USER node

EXPOSE 3000

CMD ["node", "dist/server.js"]
```

---

## 5. Best Practices

### 5.1 Оптимизация размера образа

```dockerfile
# Используйте slim/alpine образы
FROM python:3.11-slim    # вместо python:3.11
FROM node:20-alpine      # вместо node:20

# Объединяйте RUN команды
RUN apt-get update && \
    apt-get install -y package && \
    rm -rf /var/lib/apt/lists/*

# Используйте multi-stage builds
FROM golang:1.21 AS builder
RUN go build -o app .

FROM alpine:3.18
COPY --from=builder /app/app /app
```

### 5.2 Кеширование слоёв

```dockerfile
# Сначала копируйте файлы зависимостей
COPY requirements.txt .
RUN pip install -r requirements.txt

# Затем остальной код (меняется чаще)
COPY . .
```

### 5.3 Безопасность

```dockerfile
# Создавайте непривилегированного пользователя
RUN useradd --create-home appuser
USER appuser

# Используйте конкретные версии образов
FROM python:3.11.5-slim  # вместо python:latest

# Сканируйте образы на уязвимости
# docker scout cves myimage
```

### 5.4 .dockerignore

```plaintext
# .dockerignore
.git
.gitignore
node_modules
__pycache__
*.pyc
.env
.env.*
Dockerfile
docker-compose*.yml
README.md
.vscode
.idea
tests/
docs/
```

---

## 6. Полезные команды

```bash
# Информация о Docker
docker version
docker info

# Очистка системы
docker system prune           # удалить неиспользуемые данные
docker system prune -a        # включая все неиспользуемые образы
docker system prune --volumes # включая тома
docker system df              # использование диска

# Копирование файлов из/в контейнер
docker cp mycontainer:/path/file.txt ./local/
docker cp ./local/file.txt mycontainer:/path/

# Сохранение/загрузка образов
docker save -o myimage.tar myimage:latest
docker load -i myimage.tar

# Экспорт/импорт контейнеров
docker export mycontainer > container.tar
docker import container.tar mynewimage

# Просмотр изменений в файловой системе контейнера
docker diff mycontainer

# Статистика ресурсов
docker stats
docker stats mycontainer

# Просмотр процессов в контейнере
docker top mycontainer
```

---

## Резюме

Основные концепции Docker:
1. **Образы** — неизменяемые шаблоны, состоящие из слоёв
2. **Контейнеры** — запущенные изолированные экземпляры образов
3. **Dockerfile** — декларативное описание сборки образа

Ключевые команды:
- `docker build` — сборка образа
- `docker run` — создание и запуск контейнера
- `docker ps` — список контейнеров
- `docker exec` — выполнение команд в контейнере
- `docker logs` — просмотр логов

Best practices:
- Используйте multi-stage builds
- Минимизируйте количество слоёв
- Запускайте от непривилегированного пользователя
- Используйте `.dockerignore`
- Указывайте конкретные версии образов
