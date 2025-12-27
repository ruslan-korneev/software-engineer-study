# Developer Experience (DX) в Docker

## Введение

**Developer Experience (DX)** — это совокупность практик, инструментов и подходов, которые делают процесс разработки максимально комфортным и продуктивным. В контексте Docker это означает:

- Быстрая настройка локального окружения
- Минимальное время на пересборку и перезапуск
- Удобная отладка и логирование
- Интеграция с привычными инструментами разработки
- Консистентность между dev/staging/production

Хороший DX позволяет разработчику сосредоточиться на написании кода, а не на борьбе с инфраструктурой.

---

## Docker Compose для локальной разработки

Docker Compose — основной инструмент для организации dev-окружения. Он позволяет описать все сервисы приложения в одном файле и управлять ими одной командой.

### Настройка dev-окружения

Типичная структура проекта:

```
project/
├── docker-compose.yml          # Базовая конфигурация
├── docker-compose.override.yml # Dev-переопределения (автоматически подхватывается)
├── docker-compose.prod.yml     # Production конфигурация
├── Dockerfile
├── Dockerfile.dev              # Dev-версия Dockerfile
└── src/
```

**docker-compose.yml** (базовая конфигурация):

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    depends_on:
      - db
      - redis

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

volumes:
  postgres_data:
```

**docker-compose.override.yml** (dev-переопределения):

```yaml
version: '3.8'

services:
  app:
    build:
      dockerfile: Dockerfile.dev
    volumes:
      # Монтируем исходный код для hot reload
      - ./src:/app/src
      # Используем named volume для node_modules
      - node_modules:/app/node_modules
    environment:
      - NODE_ENV=development
      - DEBUG=*
    # Для отладки
    ports:
      - "9229:9229"
    command: npm run dev

  db:
    ports:
      # Открываем порт БД для локальных инструментов
      - "5432:5432"

  redis:
    ports:
      - "6379:6379"

  # Дополнительные dev-сервисы
  adminer:
    image: adminer
    ports:
      - "8080:8080"

  mailhog:
    image: mailhog/mailhog
    ports:
      - "1025:1025"
      - "8025:8025"

volumes:
  node_modules:
```

### Hot Reload / Live Reload

Hot reload позволяет видеть изменения кода без перезапуска контейнера. Ключевые компоненты:

1. **Volume с исходным кодом** — синхронизирует файлы между хостом и контейнером
2. **Watcher в приложении** — отслеживает изменения и перезапускает процесс

**Пример для Node.js с nodemon:**

```dockerfile
# Dockerfile.dev
FROM node:20-alpine

WORKDIR /app

# Устанавливаем зависимости отдельно для кэширования
COPY package*.json ./
RUN npm install

# nodemon для hot reload
RUN npm install -g nodemon

COPY . .

# Используем nodemon вместо node
CMD ["nodemon", "--legacy-watch", "src/index.js"]
```

**Пример для Python с Flask:**

```dockerfile
# Dockerfile.dev
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt
RUN pip install watchdog  # Для отслеживания изменений

COPY . .

ENV FLASK_ENV=development
ENV FLASK_DEBUG=1

CMD ["flask", "run", "--host=0.0.0.0", "--reload"]
```

**Пример для Go с Air:**

```dockerfile
# Dockerfile.dev
FROM golang:1.21-alpine

RUN go install github.com/cosmtrek/air@latest

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

CMD ["air", "-c", ".air.toml"]
```

**.air.toml** конфигурация:

```toml
[build]
  cmd = "go build -o ./tmp/main ."
  bin = "./tmp/main"
  delay = 1000
  exclude_dir = ["assets", "tmp", "vendor"]
  include_ext = ["go", "tpl", "tmpl", "html"]
  exclude_regex = ["_test.go"]
```

### Volumes для синхронизации кода

Существует несколько стратегий монтирования:

```yaml
services:
  app:
    volumes:
      # Bind mount - прямое монтирование директории
      - ./src:/app/src

      # Named volume для зависимостей (быстрее на macOS/Windows)
      - node_modules:/app/node_modules

      # Anonymous volume - сохраняет данные контейнера
      - /app/tmp

      # Read-only mount для конфигов
      - ./config:/app/config:ro

      # Cached mode для улучшения производительности на macOS
      - ./src:/app/src:cached

      # Delegated mode - хост видит изменения с задержкой
      - ./logs:/app/logs:delegated
```

---

## Docker Dev Environments

Docker Dev Environments — экспериментальная функция Docker Desktop для быстрого создания dev-окружений.

### Создание Dev Environment

1. Через Docker Desktop GUI: Dev Environments → Create
2. Через CLI:

```bash
# Создать из Git репозитория
docker dev create https://github.com/user/repo

# Создать из локальной директории
docker dev create ./my-project

# Список dev environments
docker dev list

# Открыть в VS Code
docker dev open my-env
```

### Конфигурация через compose-dev.yaml

```yaml
# compose-dev.yaml
services:
  app:
    build:
      context: .
    init: true
    volumes:
      - type: bind
        source: .
        target: /workspace
    command: sleep infinity

    # Интеграция с VS Code
    x-develop:
      watch:
        - action: sync
          path: ./src
          target: /workspace/src
        - action: rebuild
          path: package.json
```

---

## Отладка приложений в контейнерах

### Подключение дебаггера

**Node.js (V8 Inspector):**

```yaml
# docker-compose.override.yml
services:
  app:
    command: node --inspect=0.0.0.0:9229 src/index.js
    ports:
      - "9229:9229"
```

**VS Code launch.json:**

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Docker: Attach to Node",
      "type": "node",
      "request": "attach",
      "port": 9229,
      "address": "localhost",
      "localRoot": "${workspaceFolder}/src",
      "remoteRoot": "/app/src",
      "restart": true
    }
  ]
}
```

**Python (debugpy):**

```dockerfile
# Dockerfile.dev
RUN pip install debugpy
```

```python
# В коде приложения
import debugpy
debugpy.listen(("0.0.0.0", 5678))
print("Waiting for debugger attach...")
debugpy.wait_for_client()
```

```yaml
services:
  app:
    ports:
      - "5678:5678"
```

**Go (Delve):**

```dockerfile
# Dockerfile.dev
FROM golang:1.21

RUN go install github.com/go-delve/delve/cmd/dlv@latest

WORKDIR /app
COPY . .
RUN go build -gcflags="all=-N -l" -o main .

# Запускаем через delve
CMD ["dlv", "--listen=:40000", "--headless=true", "--api-version=2", "exec", "./main"]
```

### Логирование

**Просмотр логов:**

```bash
# Логи конкретного сервиса
docker compose logs app

# Следить за логами в реальном времени
docker compose logs -f app

# Последние N строк
docker compose logs --tail=100 app

# Логи всех сервисов
docker compose logs -f

# С временными метками
docker compose logs -t app
```

**Структурированное логирование в JSON:**

```yaml
services:
  app:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

### Интерактивный режим

```bash
# Войти в работающий контейнер
docker compose exec app bash
docker compose exec app sh  # Для alpine

# Запустить одноразовый контейнер
docker compose run --rm app bash

# Выполнить команду
docker compose exec app npm test
docker compose exec db psql -U user -d myapp

# Интерактивный режим с TTY
docker compose exec -it app python manage.py shell
```

---

## IDE интеграции

### VS Code Dev Containers

Dev Containers позволяет запускать VS Code внутри контейнера с полной изоляцией окружения.

**.devcontainer/devcontainer.json:**

```json
{
  "name": "My Project Dev",

  // Использовать docker-compose
  "dockerComposeFile": ["../docker-compose.yml", "docker-compose.devcontainer.yml"],
  "service": "app",
  "workspaceFolder": "/app",

  // Или использовать Dockerfile
  // "build": {
  //   "dockerfile": "Dockerfile",
  //   "context": ".."
  // },

  // Расширения VS Code для установки в контейнере
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "ms-python.vscode-pylance",
        "esbenp.prettier-vscode",
        "dbaeumer.vscode-eslint"
      ],
      "settings": {
        "python.defaultInterpreterPath": "/usr/local/bin/python",
        "editor.formatOnSave": true
      }
    }
  },

  // Команды после создания контейнера
  "postCreateCommand": "pip install -r requirements.txt",

  // Проброс портов
  "forwardPorts": [3000, 5432],

  // Переменные окружения
  "containerEnv": {
    "DEBUG": "true"
  },

  // Монтирование
  "mounts": [
    "source=${localEnv:HOME}/.ssh,target=/home/vscode/.ssh,type=bind,readonly"
  ],

  // Запуск от имени non-root пользователя
  "remoteUser": "vscode"
}
```

**.devcontainer/docker-compose.devcontainer.yml:**

```yaml
version: '3.8'

services:
  app:
    volumes:
      - ..:/app:cached
      - vscode-extensions:/home/vscode/.vscode-server/extensions
    command: sleep infinity

volumes:
  vscode-extensions:
```

**Полезные команды VS Code:**

- `Dev Containers: Reopen in Container` — открыть проект в контейнере
- `Dev Containers: Rebuild Container` — пересобрать контейнер
- `Dev Containers: Attach to Running Container` — подключиться к существующему контейнеру

### JetBrains Docker integration

**Настройка Docker в IntelliJ/PyCharm:**

1. Settings → Build, Execution, Deployment → Docker
2. Добавить Docker connection (Docker for Mac/Windows или TCP socket)

**Настройка Remote Interpreter:**

1. Settings → Project → Python Interpreter
2. Add Interpreter → On Docker Compose
3. Выбрать service и путь к интерпретатору

**Run/Debug Configuration:**

```xml
<!-- .idea/runConfigurations/Docker_App.xml -->
<component name="ProjectRunConfigurationManager">
  <configuration name="Docker App" type="docker-deploy">
    <deployment type="docker-compose.yml">
      <settings>
        <option name="sourceFilePath" value="docker-compose.yml" />
        <option name="services">
          <list>
            <option value="app" />
          </list>
        </option>
      </settings>
    </deployment>
  </configuration>
</component>
```

---

## Ускорение сборки образов

### Кэширование слоёв

Порядок инструкций в Dockerfile критически важен:

```dockerfile
# ❌ Плохо - любое изменение кода инвалидирует кэш зависимостей
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm install

# ✅ Хорошо - зависимости кэшируются отдельно
FROM node:20-alpine
WORKDIR /app

# Сначала файлы зависимостей
COPY package.json package-lock.json ./
RUN npm ci

# Потом исходный код
COPY . .
```

**Кэширование с BuildKit mount cache:**

```dockerfile
# syntax=docker/dockerfile:1.4

FROM python:3.11-slim

WORKDIR /app

# Кэш pip между сборками
RUN --mount=type=cache,target=/root/.cache/pip \
    --mount=type=bind,source=requirements.txt,target=requirements.txt \
    pip install -r requirements.txt

COPY . .
```

### Multi-stage builds для dev

```dockerfile
# syntax=docker/dockerfile:1.4

# ===== Base stage =====
FROM node:20-alpine AS base
WORKDIR /app
COPY package*.json ./

# ===== Development stage =====
FROM base AS development
RUN npm install
RUN npm install -g nodemon
COPY . .
CMD ["nodemon", "src/index.js"]

# ===== Production dependencies =====
FROM base AS prod-deps
RUN npm ci --only=production

# ===== Build stage =====
FROM base AS build
RUN npm ci
COPY . .
RUN npm run build

# ===== Production stage =====
FROM node:20-alpine AS production
WORKDIR /app
COPY --from=prod-deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
CMD ["node", "dist/index.js"]
```

**Использование:**

```bash
# Dev
docker build --target development -t myapp:dev .

# Production
docker build --target production -t myapp:prod .
```

### BuildKit

BuildKit — современный бэкенд для сборки образов с множеством оптимизаций.

**Включение BuildKit:**

```bash
# Через переменную окружения
export DOCKER_BUILDKIT=1
docker build .

# Через docker buildx
docker buildx build .

# В Docker Desktop включён по умолчанию
```

**Возможности BuildKit:**

```dockerfile
# syntax=docker/dockerfile:1.4

FROM ubuntu:22.04

# Параллельное выполнение независимых RUN
RUN apt-get update
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get install -y curl git

# Секреты без сохранения в слоях
RUN --mount=type=secret,id=aws,target=/root/.aws/credentials \
    aws s3 cp s3://bucket/file .

# SSH для приватных репозиториев
RUN --mount=type=ssh \
    git clone git@github.com:private/repo.git
```

**Сборка с BuildKit:**

```bash
# С секретами
docker build --secret id=aws,src=$HOME/.aws/credentials .

# С SSH
docker build --ssh default .

# Параллельная сборка нескольких платформ
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest .
```

---

## Полезные инструменты

### lazydocker

Терминальный UI для управления Docker:

```bash
# Установка
brew install lazydocker           # macOS
curl https://raw.githubusercontent.com/jesseduffield/lazydocker/master/scripts/install_update_linux.sh | bash  # Linux

# Запуск
lazydocker
```

**Основные возможности:**

- Просмотр контейнеров, образов, volumes, networks
- Логи в реальном времени
- Статистика ресурсов
- Быстрый exec в контейнер
- Удаление неиспользуемых ресурсов

**Горячие клавиши:**

- `[` `]` — переключение между панелями
- `enter` — выбрать элемент
- `d` — удалить
- `s` — остановить
- `r` — перезапустить
- `x` — меню действий

### dive

Анализ слоёв Docker-образа:

```bash
# Установка
brew install dive  # macOS
docker pull wagoodman/dive

# Анализ образа
dive myapp:latest

# Или через Docker
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive:latest myapp:latest
```

**Что показывает dive:**

- Размер каждого слоя
- Какие файлы добавлены/изменены/удалены
- Эффективность использования места
- Потенциальная экономия при оптимизации

**CI-режим:**

```bash
# Проверка эффективности образа
dive myapp:latest --ci

# С пользовательскими порогами
CI=true dive myapp:latest \
  --highestUserWastedPercent=0.1 \
  --lowestEfficiency=0.95
```

### ctop

Мониторинг контейнеров в реальном времени (как htop для Docker):

```bash
# Установка
brew install ctop  # macOS

# Или через Docker
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  quay.io/vektorlab/ctop:latest

# Запуск
ctop
```

**Отображаемые метрики:**

- CPU usage
- Memory usage
- Network I/O
- Block I/O
- PIDs

### Другие полезные инструменты

```bash
# docker-compose-viz - визуализация docker-compose
docker run --rm -it --name dcv \
  -v $(pwd):/input pmsipilot/docker-compose-viz \
  render -m image docker-compose.yml

# dockle - линтер безопасности образов
dockle myapp:latest

# hadolint - линтер Dockerfile
hadolint Dockerfile

# docker-slim - оптимизация размера образов
docker-slim build myapp:latest
```

---

## Типичные проблемы и их решения

### Медленные volumes на macOS/Windows

Docker на macOS и Windows использует виртуализацию, что замедляет файловые операции.

**Проблема:** Node.js проект с 100k файлов в node_modules работает медленно.

**Решения:**

1. **Named volumes для зависимостей:**

```yaml
services:
  app:
    volumes:
      - ./src:/app/src           # Только исходный код
      - node_modules:/app/node_modules  # Named volume для deps

volumes:
  node_modules:
```

2. **Consistency modes (устаревшее, но может помочь):**

```yaml
volumes:
  - ./src:/app/src:cached     # Хост — источник правды
  - ./logs:/app/logs:delegated  # Контейнер — источник правды
```

3. **Docker Desktop VirtioFS (macOS):**

Settings → General → Choose file sharing implementation → VirtioFS

4. **Mutagen для синхронизации:**

```yaml
# docker-compose.yml
x-mutagen:
  sync:
    defaults:
      mode: "two-way-resolved"
    code:
      alpha: "./src"
      beta: "volume://code-sync"

services:
  app:
    volumes:
      - code-sync:/app/src

volumes:
  code-sync:
```

5. **Установка зависимостей внутри контейнера:**

```bash
# Не монтировать node_modules с хоста
docker compose exec app npm install
```

### Права доступа к файлам

**Проблема:** Файлы, созданные в контейнере, принадлежат root на хосте.

**Решение 1: Запуск от имени текущего пользователя:**

```yaml
services:
  app:
    user: "${UID:-1000}:${GID:-1000}"
    volumes:
      - ./src:/app/src
```

```bash
# Экспорт переменных
export UID=$(id -u)
export GID=$(id -g)
docker compose up
```

**Решение 2: Создание пользователя в Dockerfile:**

```dockerfile
FROM node:20-alpine

# Создаём пользователя с тем же UID, что и на хосте
ARG UID=1000
ARG GID=1000

RUN addgroup -g $GID appgroup && \
    adduser -u $UID -G appgroup -D appuser

WORKDIR /app
RUN chown -R appuser:appgroup /app

USER appuser

COPY --chown=appuser:appgroup . .
```

```yaml
services:
  app:
    build:
      args:
        UID: ${UID:-1000}
        GID: ${GID:-1000}
```

**Решение 3: fixuid (для dev containers):**

```dockerfile
FROM node:20

# Установка fixuid
RUN curl -SsL https://github.com/boxboat/fixuid/releases/download/v0.6.0/fixuid-0.6.0-linux-amd64.tar.gz | tar -C /usr/local/bin -xzf -

RUN addgroup --gid 1000 docker && \
    adduser --uid 1000 --ingroup docker --home /home/docker docker && \
    chown -R docker:docker /home/docker

RUN printf "user: docker\ngroup: docker\n" > /etc/fixuid/config.yml

USER docker:docker
ENTRYPOINT ["fixuid", "-q"]
```

### Другие частые проблемы

**Порт уже занят:**

```bash
# Найти процесс, использующий порт
lsof -i :3000

# Использовать другой порт
docker compose up -d
# Error: port 3000 is already allocated

# Изменить маппинг в docker-compose.yml
ports:
  - "3001:3000"
```

**Устаревший кэш сборки:**

```bash
# Пересборка без кэша
docker compose build --no-cache

# Удаление всего build cache
docker builder prune
```

**Контейнер сразу останавливается:**

```bash
# Проверить логи
docker compose logs app

# Запустить интерактивно для отладки
docker compose run --rm app sh

# Проверить exit code
docker compose ps -a
```

---

## Best practices для комфортной разработки

### 1. Структура проекта

```
project/
├── .devcontainer/
│   └── devcontainer.json
├── docker/
│   ├── dev/
│   │   └── Dockerfile
│   └── prod/
│       └── Dockerfile
├── docker-compose.yml
├── docker-compose.override.yml      # Dev (автоподхватывается)
├── docker-compose.prod.yml
├── docker-compose.test.yml
├── .dockerignore
├── .env.example
└── src/
```

### 2. Makefile для частых команд

```makefile
.PHONY: up down build logs shell test

# Запуск dev окружения
up:
	docker compose up -d

# Остановка
down:
	docker compose down

# Пересборка
build:
	docker compose build

# Логи
logs:
	docker compose logs -f

# Shell в app контейнере
shell:
	docker compose exec app sh

# Запуск тестов
test:
	docker compose exec app npm test

# Очистка всего
clean:
	docker compose down -v --rmi local
	docker system prune -f

# Полная пересборка с нуля
rebuild: clean build up
```

### 3. Быстрый старт для новых разработчиков

```bash
# Один файл для старта
# scripts/dev-setup.sh

#!/bin/bash
set -e

echo "🐳 Setting up development environment..."

# Проверка Docker
if ! command -v docker &> /dev/null; then
    echo "❌ Docker is not installed"
    exit 1
fi

# Копирование env файла
if [ ! -f .env ]; then
    cp .env.example .env
    echo "✅ Created .env file"
fi

# Сборка и запуск
docker compose build
docker compose up -d

# Ожидание готовности сервисов
echo "⏳ Waiting for services..."
sleep 5

# Миграции и seed
docker compose exec app npm run db:migrate
docker compose exec app npm run db:seed

echo "✅ Development environment is ready!"
echo "🌐 App: http://localhost:3000"
echo "📊 Adminer: http://localhost:8080"
```

### 4. Health checks для dev

```yaml
services:
  app:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s

  db:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 5s
      timeout: 5s
      retries: 5
```

### 5. Профили для разных сценариев

```yaml
services:
  app:
    # Всегда запускается

  db:
    # Всегда запускается

  adminer:
    profiles: ["debug"]

  prometheus:
    profiles: ["monitoring"]

  grafana:
    profiles: ["monitoring"]

  test-runner:
    profiles: ["test"]
```

```bash
# Только основные сервисы
docker compose up

# С инструментами отладки
docker compose --profile debug up

# С мониторингом
docker compose --profile monitoring up

# Для тестов
docker compose --profile test up
```

### 6. Удобные алиасы

```bash
# ~/.bashrc или ~/.zshrc

# Docker Compose
alias dc='docker compose'
alias dcu='docker compose up -d'
alias dcd='docker compose down'
alias dcl='docker compose logs -f'
alias dce='docker compose exec'
alias dcr='docker compose run --rm'
alias dcb='docker compose build'
alias dcps='docker compose ps'

# Docker
alias dps='docker ps'
alias dpsa='docker ps -a'
alias dimg='docker images'
alias drm='docker rm'
alias drmi='docker rmi'
alias dprune='docker system prune -f'

# Быстрый вход в контейнер
dsh() {
    docker compose exec "$1" sh
}
```

### 7. Git hooks для Docker

**.husky/pre-commit:**

```bash
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

# Линтинг Dockerfile
hadolint Dockerfile* || exit 1

# Проверка docker-compose
docker compose config -q || exit 1
```

---

## Заключение

Хороший Developer Experience в Docker строится на нескольких принципах:

1. **Минимальное время на настройку** — новый разработчик должен запустить проект одной командой
2. **Быстрая обратная связь** — hot reload, быстрые пересборки
3. **Удобная отладка** — интеграция с IDE, понятные логи
4. **Документированность** — README, Makefile, скрипты
5. **Консистентность** — одинаковое окружение у всех разработчиков

Инвестиции в DX окупаются многократно: меньше времени на "борьбу с Docker", больше — на написание кода.
