# Создание собственных Docker-образов

## Введение

Создание собственных Docker-образов — ключевой навык для любого разработчика, работающего с контейнерами. Хотя Docker Hub предлагает тысячи готовых образов, в реальных проектах почти всегда требуется создавать свои образы, адаптированные под конкретные нужды приложения.

### Зачем создавать свои образы?

1. **Упаковка приложения** — ваш код вместе со всеми зависимостями в одном образе
2. **Воспроизводимость** — одинаковое окружение на всех этапах разработки и в продакшене
3. **Автоматизация** — образы легко интегрируются в CI/CD пайплайны
4. **Кастомизация** — добавление специфичных настроек, библиотек, инструментов
5. **Версионирование** — контроль версий окружения через теги образов

---

## Dockerfile — основа создания образов

**Dockerfile** — это текстовый файл с инструкциями для сборки Docker-образа. Каждая инструкция создаёт новый слой в образе.

### Структура Dockerfile

```dockerfile
# Комментарий начинается с символа #
# Базовый образ
FROM python:3.11-slim

# Метаданные
LABEL maintainer="developer@example.com"
LABEL version="1.0"

# Переменные окружения
ENV PYTHONUNBUFFERED=1
ENV APP_HOME=/app

# Рабочая директория
WORKDIR $APP_HOME

# Копирование файлов
COPY requirements.txt .

# Выполнение команд
RUN pip install --no-cache-dir -r requirements.txt

# Копирование исходного кода
COPY . .

# Открытие порта
EXPOSE 8000

# Команда запуска
CMD ["python", "app.py"]
```

### Основные инструкции

#### FROM — базовый образ

```dockerfile
# Официальный образ Python
FROM python:3.11

# Минимальный образ Alpine
FROM python:3.11-alpine

# Образ без родителя (scratch)
FROM scratch

# Использование конкретного дайджеста для воспроизводимости
FROM python@sha256:abc123...
```

**Рекомендации:**
- Используйте официальные образы
- Предпочитайте slim/alpine версии для уменьшения размера
- Указывайте конкретную версию, не используйте `latest`

#### RUN — выполнение команд

```dockerfile
# Shell форма
RUN apt-get update && apt-get install -y curl

# Exec форма
RUN ["apt-get", "update"]

# Многострочные команды с очисткой кеша
RUN apt-get update && apt-get install -y \
    curl \
    git \
    vim \
    && rm -rf /var/lib/apt/lists/*
```

**Важно:** Каждая инструкция `RUN` создаёт новый слой. Объединяйте связанные команды через `&&` для оптимизации.

#### COPY и ADD — копирование файлов

```dockerfile
# COPY - простое копирование
COPY src/ /app/src/
COPY ["file with spaces.txt", "/app/"]

# Копирование с изменением владельца
COPY --chown=user:group files/ /app/

# ADD - расширенные возможности
ADD archive.tar.gz /app/              # Автоматически распаковывает архивы
ADD https://example.com/file /app/    # Скачивает файлы по URL
```

**Когда что использовать:**
- `COPY` — в большинстве случаев (проще и предсказуемее)
- `ADD` — только когда нужна распаковка архивов

#### WORKDIR — рабочая директория

```dockerfile
# Устанавливает рабочую директорию
WORKDIR /app

# Можно использовать переменные
ENV APP_HOME=/application
WORKDIR $APP_HOME

# Создаётся автоматически, если не существует
WORKDIR /path/to/new/directory
```

#### EXPOSE — объявление портов

```dockerfile
# Объявляем порт (документация)
EXPOSE 8080

# Несколько портов
EXPOSE 80 443

# UDP порт
EXPOSE 53/udp
```

**Важно:** `EXPOSE` не публикует порт! Это только документация. Для публикации используйте флаг `-p` при `docker run`.

#### ENV — переменные окружения

```dockerfile
# Одна переменная
ENV NODE_ENV=production

# Несколько переменных
ENV NODE_ENV=production \
    PORT=3000 \
    DEBUG=false

# Использование в других инструкциях
ENV APP_DIR=/app
WORKDIR $APP_DIR
```

#### ARG — аргументы сборки

```dockerfile
# Определение аргумента с значением по умолчанию
ARG VERSION=latest
ARG NODE_VERSION=18

# Использование аргумента
FROM node:${NODE_VERSION}

# Передача при сборке: docker build --build-arg VERSION=1.0 .
```

**Разница между ARG и ENV:**
- `ARG` — доступен только во время сборки
- `ENV` — доступен и при сборке, и при запуске контейнера

#### CMD и ENTRYPOINT

```dockerfile
# CMD - команда по умолчанию (можно переопределить при запуске)
CMD ["python", "app.py"]
CMD python app.py  # shell форма

# ENTRYPOINT - основная команда (сложнее переопределить)
ENTRYPOINT ["python", "app.py"]

# Комбинация - ENTRYPOINT + CMD
ENTRYPOINT ["python"]
CMD ["app.py"]  # аргументы по умолчанию для ENTRYPOINT
```

### Разница между CMD и ENTRYPOINT

| Характеристика | CMD | ENTRYPOINT |
|---------------|-----|------------|
| Назначение | Аргументы по умолчанию | Основная команда |
| Переопределение | Легко (любым аргументом) | Требует `--entrypoint` |
| Использование | Когда контейнер может запускаться по-разному | Когда контейнер — исполняемый файл |

**Пример комбинации:**

```dockerfile
# Dockerfile
ENTRYPOINT ["python", "manage.py"]
CMD ["runserver", "0.0.0.0:8000"]
```

```bash
# Запуск с настройками по умолчанию
docker run myapp
# Выполнит: python manage.py runserver 0.0.0.0:8000

# Переопределение CMD
docker run myapp migrate
# Выполнит: python manage.py migrate

# Переопределение ENTRYPOINT
docker run --entrypoint bash myapp
# Выполнит: bash
```

---

## Команда docker build

### Синтаксис и основные опции

```bash
# Базовый синтаксис
docker build [OPTIONS] PATH

# Примеры использования
docker build .                              # Сборка из текущей директории
docker build -t myapp:1.0 .                 # С тегом
docker build -f Dockerfile.prod .           # Указание файла
docker build --no-cache .                   # Без использования кеша
docker build --build-arg VERSION=2.0 .      # С аргументом сборки
docker build --target builder .             # Определённый этап multi-stage
```

**Основные опции:**

| Опция | Описание |
|-------|----------|
| `-t, --tag` | Имя и тег образа (name:tag) |
| `-f, --file` | Путь к Dockerfile |
| `--no-cache` | Сборка без кеша |
| `--build-arg` | Передача аргументов сборки |
| `--target` | Целевой этап в multi-stage сборке |
| `--platform` | Платформа (linux/amd64, linux/arm64) |
| `--progress` | Тип вывода (auto, plain, tty) |

### Build Context

**Build context** — это набор файлов, отправляемых Docker daemon для сборки образа.

```bash
# Текущая директория как контекст
docker build .

# Конкретная директория
docker build /path/to/context

# Git репозиторий
docker build https://github.com/user/repo.git

# Tar архив
docker build - < app.tar.gz
```

**Важно:** Все файлы из контекста отправляются на Docker daemon. Большой контекст = медленная сборка.

```bash
# Просмотр размера контекста при сборке
Sending build context to Docker daemon  2.048MB
```

### Кеширование слоёв

Docker кеширует каждый слой. При повторной сборке используются закешированные слои, если:
1. Инструкция не изменилась
2. Файлы, используемые инструкцией, не изменились

```dockerfile
# Плохо - при любом изменении кода pip install выполняется заново
COPY . /app
RUN pip install -r requirements.txt

# Хорошо - зависимости устанавливаются только при изменении requirements.txt
COPY requirements.txt /app/
RUN pip install -r requirements.txt
COPY . /app
```

**Порядок имеет значение:**
1. Редко меняющиеся инструкции — в начале
2. Часто меняющиеся — в конце

---

## Multi-stage builds

Multi-stage сборка позволяет использовать несколько этапов в одном Dockerfile, копируя только нужные артефакты между ними.

### Зачем нужны?

1. **Уменьшение размера** — в финальный образ попадает только необходимое
2. **Безопасность** — инструменты сборки не попадают в продакшен
3. **Чистота** — разделение этапов сборки и запуска
4. **Один Dockerfile** — для development и production

### Примеры использования

#### Go приложение

```dockerfile
# Этап сборки
FROM golang:1.21 AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

# Финальный образ
FROM alpine:3.18

RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/main .

EXPOSE 8080
CMD ["./main"]
```

**Результат:** образ ~15MB вместо ~1GB с Go toolchain

#### React приложение

```dockerfile
# Этап сборки
FROM node:18 AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Финальный образ с Nginx
FROM nginx:alpine

COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### Java приложение (Maven)

```dockerfile
# Этап сборки
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline

COPY src ./src
RUN mvn package -DskipTests

# Финальный образ
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Именованные этапы и их использование

```dockerfile
FROM node:18 AS deps
COPY package*.json ./
RUN npm ci

FROM node:18 AS builder
COPY --from=deps /node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:18 AS tester
COPY --from=deps /node_modules ./node_modules
COPY . .
RUN npm test

FROM node:18-alpine AS production
COPY --from=builder /app/dist ./dist
COPY --from=deps /node_modules ./node_modules
CMD ["node", "dist/index.js"]
```

```bash
# Сборка только до определённого этапа
docker build --target tester -t myapp:test .
docker build --target production -t myapp:prod .
```

---

## Практические примеры

### Образ для Python приложения (FastAPI)

```dockerfile
# Dockerfile для FastAPI приложения
FROM python:3.11-slim AS base

# Настройка Python
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app

# Установка системных зависимостей
FROM base AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Создание виртуального окружения
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Установка Python зависимостей
COPY requirements.txt .
RUN pip install -r requirements.txt

# Финальный образ
FROM base AS production

# Копирование виртуального окружения
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Создание непривилегированного пользователя
RUN useradd --create-home --shell /bin/bash appuser
USER appuser

# Копирование приложения
COPY --chown=appuser:appuser . .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Образ для Node.js приложения (Express)

```dockerfile
# Dockerfile для Node.js приложения
FROM node:18-alpine AS base

WORKDIR /app

# Установка зависимостей
FROM base AS deps

COPY package.json package-lock.json ./
RUN npm ci --only=production

# Сборка (если есть TypeScript)
FROM base AS builder

COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build

# Продакшен образ
FROM base AS production

ENV NODE_ENV=production

# Создание пользователя
RUN addgroup --system --gid 1001 nodejs \
    && adduser --system --uid 1001 nodeuser

WORKDIR /app

# Копирование только необходимого
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package.json ./

USER nodeuser

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

### Образ для Go приложения

```dockerfile
# Dockerfile для Go приложения
FROM golang:1.21-alpine AS builder

# Установка сертификатов и git
RUN apk add --no-cache ca-certificates git

WORKDIR /app

# Загрузка зависимостей
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# Сборка приложения
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-w -s" -o /app/server ./cmd/server

# Минимальный образ
FROM scratch

# Копирование сертификатов для HTTPS запросов
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Копирование бинарника
COPY --from=builder /app/server /server

EXPOSE 8080

ENTRYPOINT ["/server"]
```

**Размер финального образа:** 5-15MB (только бинарник + сертификаты)

---

## Best Practices написания Dockerfile

### 1. Оптимизация слоёв

```dockerfile
# Плохо - много слоёв
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN rm -rf /var/lib/apt/lists/*

# Хорошо - один слой
RUN apt-get update && apt-get install -y \
    curl \
    git \
    && rm -rf /var/lib/apt/lists/*
```

### 2. Порядок инструкций (кеширование)

```dockerfile
# Хорошо - зависимости редко меняются
COPY package.json package-lock.json ./
RUN npm ci

# Код меняется часто - копируем в конце
COPY . .
```

### 3. Использование .dockerignore

```dockerignore
# .dockerignore
.git
.gitignore
.env
.env.*
*.md
README*
LICENSE
Dockerfile*
docker-compose*
.dockerignore

# Python
__pycache__
*.pyc
*.pyo
.pytest_cache
.coverage
.venv
venv
env

# Node.js
node_modules
npm-debug.log
.npm

# IDE
.idea
.vscode
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Tests
tests/
test/
*_test.go
*_test.py

# Documentation
docs/
```

### 4. Безопасность

```dockerfile
# 1. Используйте конкретные версии
FROM python:3.11.4-slim  # Не FROM python:latest

# 2. Создавайте непривилегированного пользователя
RUN useradd --create-home --shell /bin/bash appuser
USER appuser

# 3. Не храните секреты в образе
# Плохо
ENV API_KEY=secret123

# Хорошо - передавать при запуске
# docker run -e API_KEY=secret123 myapp

# 4. Используйте COPY вместо ADD
COPY requirements.txt .  # Предсказуемо

# 5. Проверяйте образы на уязвимости
# docker scan myimage:tag
# trivy image myimage:tag

# 6. Минимизируйте установленные пакеты
RUN apt-get install --no-install-recommends -y package
```

### 5. Метаданные и документация

```dockerfile
# Информация об образе
LABEL org.opencontainers.image.title="My Application"
LABEL org.opencontainers.image.description="Production-ready app"
LABEL org.opencontainers.image.version="1.0.0"
LABEL org.opencontainers.image.authors="dev@example.com"
LABEL org.opencontainers.image.source="https://github.com/user/repo"

# Документирование портов
EXPOSE 8080

# Healthcheck
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1
```

### 6. Использование HEALTHCHECK

```dockerfile
# HTTP проверка
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# TCP проверка
HEALTHCHECK --interval=30s --timeout=3s \
    CMD nc -z localhost 8080 || exit 1

# Кастомный скрипт
COPY healthcheck.sh /usr/local/bin/
HEALTHCHECK --interval=30s CMD ["healthcheck.sh"]
```

---

## Типичные ошибки

### 1. Использование latest тега

```dockerfile
# Плохо - непредсказуемо
FROM python:latest

# Хорошо - воспроизводимо
FROM python:3.11.4-slim
```

### 2. Запуск от root

```dockerfile
# Плохо - всё от root
CMD ["python", "app.py"]

# Хорошо - непривилегированный пользователь
RUN useradd -m appuser
USER appuser
CMD ["python", "app.py"]
```

### 3. Большой build context

```bash
# Проблема: весь проект (~500MB) отправляется daemon
Sending build context to Docker daemon  524.3MB

# Решение: создать .dockerignore
```

### 4. Секреты в образе

```dockerfile
# Плохо - секреты останутся в слоях
COPY .env /app/
ENV DATABASE_PASSWORD=secret123

# Хорошо - секреты передаются при запуске
# docker run -e DATABASE_PASSWORD=secret myapp
# docker run --env-file .env myapp
```

### 5. Неоптимальный порядок инструкций

```dockerfile
# Плохо - при любом изменении кода кеш инвалидируется
COPY . .
RUN npm install

# Хорошо - зависимости кешируются отдельно
COPY package*.json ./
RUN npm install
COPY . .
```

### 6. Отсутствие очистки кеша

```dockerfile
# Плохо - кеш apt остаётся в образе
RUN apt-get update && apt-get install -y curl

# Хорошо - очистка после установки
RUN apt-get update && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
```

### 7. Игнорирование .dockerignore

Без `.dockerignore`:
- `node_modules` (сотни мегабайт) попадают в контекст
- `.git` история увеличивает контекст
- Локальные `.env` файлы с секретами копируются

### 8. Один слой на каждую команду

```dockerfile
# Плохо - 4 слоя
RUN apt-get update
RUN apt-get install -y curl
RUN curl -o file.txt https://example.com/file
RUN rm -rf /var/lib/apt/lists/*

# Хорошо - 1 слой
RUN apt-get update \
    && apt-get install -y curl \
    && curl -o file.txt https://example.com/file \
    && rm -rf /var/lib/apt/lists/*
```

---

## Полезные команды

```bash
# Просмотр слоёв образа
docker history myimage:tag

# Анализ размера образа
docker images myimage:tag
docker image inspect myimage:tag

# Инструменты анализа
dive myimage:tag  # Интерактивный анализ слоёв

# Очистка неиспользуемых образов
docker image prune -a

# Экспорт/импорт образа
docker save myimage:tag > image.tar
docker load < image.tar

# Сканирование на уязвимости
docker scout cves myimage:tag
trivy image myimage:tag
```

---

## Резюме

1. **Dockerfile** — декларативный способ описания сборки образа
2. **Каждая инструкция** создаёт новый слой — объединяйте связанные команды
3. **Порядок важен** — помещайте редко меняющиеся инструкции вверху
4. **Multi-stage builds** — уменьшают размер и повышают безопасность
5. **Безопасность** — не запускайте от root, не храните секреты в образе
6. **.dockerignore** — обязателен для любого проекта
7. **Конкретные версии** — гарантируют воспроизводимость сборки
