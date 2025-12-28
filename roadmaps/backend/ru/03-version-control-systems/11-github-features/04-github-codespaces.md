# GitHub Codespaces

[prev: 03-github-packages](./03-github-packages.md) | [next: 05-github-sponsors](./05-github-sponsors.md)
---

## Что такое GitHub Codespaces?

**GitHub Codespaces** - это облачная среда разработки, которая позволяет писать, запускать и отлаживать код прямо в браузере или через VS Code. По сути, это полноценная виртуальная машина с настроенным окружением разработки.

### Ключевые особенности

- Полноценная VS Code в браузере
- Предустановленные инструменты и зависимости
- Настраиваемое окружение через devcontainer
- Доступ с любого устройства
- Интеграция с репозиториями GitHub
- Prebuilds для мгновенного старта

### Преимущества

| Традиционная разработка | GitHub Codespaces |
|------------------------|-------------------|
| Настройка окружения вручную | Автоматическая настройка |
| Зависимость от локальной машины | Работа с любого устройства |
| "Works on my machine" | Одинаковое окружение для всех |
| Мощный локальный компьютер | Облачные ресурсы |

## Создание Codespace

### Через веб-интерфейс GitHub

1. Откройте репозиторий на GitHub
2. Нажмите зеленую кнопку **Code**
3. Выберите вкладку **Codespaces**
4. Нажмите **Create codespace on main** (или другой ветке)

### Через GitHub CLI

```bash
# Создать codespace для репозитория
gh codespace create --repo OWNER/REPO

# Создать с конкретной веткой
gh codespace create --repo OWNER/REPO --branch feature-branch

# Создать с определенным типом машины
gh codespace create --repo OWNER/REPO --machine largePremiumLinux
```

### Через VS Code

1. Установите расширение **GitHub Codespaces**
2. Откройте Command Palette (Ctrl/Cmd + Shift + P)
3. Введите "Codespaces: Create New Codespace"
4. Выберите репозиторий и ветку

## Управление Codespaces

### Список codespaces

```bash
# Все ваши codespaces
gh codespace list

# В формате JSON
gh codespace list --json name,state,repository
```

### Подключение к codespace

```bash
# Открыть в браузере
gh codespace code --web

# Открыть в VS Code Desktop
gh codespace code

# SSH-доступ
gh codespace ssh
```

### Остановка и удаление

```bash
# Остановить codespace
gh codespace stop --codespace CODESPACE_NAME

# Удалить codespace
gh codespace delete --codespace CODESPACE_NAME

# Удалить все остановленные
gh codespace delete --all --days 0
```

## Настройка devcontainer.json

Файл `.devcontainer/devcontainer.json` определяет окружение разработки.

### Базовая структура

```json
{
    "name": "My Dev Environment",
    "image": "mcr.microsoft.com/devcontainers/base:ubuntu",

    "features": {
        "ghcr.io/devcontainers/features/python:1": {
            "version": "3.11"
        },
        "ghcr.io/devcontainers/features/node:1": {
            "version": "20"
        }
    },

    "customizations": {
        "vscode": {
            "extensions": [
                "ms-python.python",
                "ms-python.vscode-pylance",
                "esbenp.prettier-vscode"
            ],
            "settings": {
                "python.defaultInterpreterPath": "/usr/local/bin/python"
            }
        }
    },

    "postCreateCommand": "pip install -r requirements.txt",

    "forwardPorts": [8000, 5432],

    "portsAttributes": {
        "8000": {
            "label": "Application",
            "onAutoForward": "openPreview"
        }
    }
}
```

### Использование Docker Compose

```json
{
    "name": "Full Stack App",
    "dockerComposeFile": "docker-compose.yml",
    "service": "app",
    "workspaceFolder": "/workspace",

    "customizations": {
        "vscode": {
            "extensions": [
                "ms-python.python",
                "ms-azuretools.vscode-docker"
            ]
        }
    },

    "forwardPorts": [8000, 5432, 6379]
}
```

### docker-compose.yml для devcontainer

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - ..:/workspace:cached
    command: sleep infinity
    depends_on:
      - db
      - redis

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

volumes:
  postgres_data:
```

### Полный пример для Python-проекта

```json
{
    "name": "Python FastAPI",
    "build": {
        "dockerfile": "Dockerfile",
        "context": ".."
    },

    "features": {
        "ghcr.io/devcontainers/features/python:1": {
            "version": "3.11",
            "installTools": true
        },
        "ghcr.io/devcontainers/features/docker-in-docker:2": {},
        "ghcr.io/devcontainers/features/git:1": {}
    },

    "customizations": {
        "vscode": {
            "extensions": [
                "ms-python.python",
                "ms-python.vscode-pylance",
                "ms-python.black-formatter",
                "charliermarsh.ruff",
                "tamasfe.even-better-toml",
                "redhat.vscode-yaml"
            ],
            "settings": {
                "python.defaultInterpreterPath": "/usr/local/bin/python",
                "python.formatting.provider": "black",
                "editor.formatOnSave": true,
                "[python]": {
                    "editor.defaultFormatter": "ms-python.black-formatter"
                }
            }
        }
    },

    "postCreateCommand": "pip install -e '.[dev]'",
    "postStartCommand": "git fetch --all",

    "forwardPorts": [8000],

    "portsAttributes": {
        "8000": {
            "label": "FastAPI",
            "onAutoForward": "openBrowser"
        }
    },

    "remoteUser": "vscode",

    "containerEnv": {
        "PYTHONDONTWRITEBYTECODE": "1",
        "PYTHONUNBUFFERED": "1"
    }
}
```

### Dockerfile для devcontainer

```dockerfile
# .devcontainer/Dockerfile
FROM mcr.microsoft.com/devcontainers/python:3.11

# Установка системных зависимостей
RUN apt-get update && apt-get install -y \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Установка Poetry
RUN pip install poetry

# Настройка Poetry
ENV POETRY_VIRTUALENVS_CREATE=false

# Копирование файлов зависимостей
COPY pyproject.toml poetry.lock* /tmp/
WORKDIR /tmp

# Установка зависимостей
RUN poetry install --no-root

WORKDIR /workspace
```

## Prebuilds

**Prebuilds** позволяют заранее собрать окружение, чтобы codespace запускался мгновенно.

### Настройка prebuilds

1. Перейдите в **Settings** > **Codespaces** репозитория
2. В разделе **Prebuilds** нажмите **Set up prebuild**
3. Выберите ветки и регионы
4. Настройте триггеры

### Конфигурация в devcontainer.json

```json
{
    "name": "Prebuild Ready",
    "image": "mcr.microsoft.com/devcontainers/python:3.11",

    "onCreateCommand": "pip install -r requirements.txt",
    "updateContentCommand": "pip install -r requirements.txt",
    "postCreateCommand": "git config --global core.editor 'code --wait'",

    "waitFor": "onCreateCommand"
}
```

### Lifecycle команды

| Команда | Когда выполняется |
|---------|-------------------|
| `onCreateCommand` | При создании контейнера (включая prebuild) |
| `updateContentCommand` | После обновления source content |
| `postCreateCommand` | После первого подключения пользователя |
| `postStartCommand` | При каждом запуске |
| `postAttachCommand` | При каждом подключении к codespace |

### GitHub Actions для prebuilds

```yaml
# .github/workflows/codespaces-prebuild.yml
name: Codespaces Prebuilds

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  create_codespace:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Create Codespace
        run: |
          gh codespace create \
            --repo ${{ github.repository }} \
            --branch main \
            --status
        env:
          GITHUB_TOKEN: ${{ secrets.CODESPACES_TOKEN }}
```

## VS Code в браузере

### Горячие клавиши

| Действие | Windows/Linux | Mac |
|----------|--------------|-----|
| Command Palette | Ctrl+Shift+P | Cmd+Shift+P |
| Quick Open | Ctrl+P | Cmd+P |
| Terminal | Ctrl+` | Cmd+` |
| Split Editor | Ctrl+\ | Cmd+\ |
| Toggle Sidebar | Ctrl+B | Cmd+B |
| Go to Definition | F12 | F12 |
| Find in Files | Ctrl+Shift+F | Cmd+Shift+F |

### Работа с терминалом

```bash
# Создать новый терминал
# Через меню Terminal > New Terminal
# Или Ctrl/Cmd + Shift + `

# Разделить терминал
# Нажать иконку split в панели терминала

# Переименовать терминал
# Правый клик > Rename
```

### Port Forwarding

Codespaces автоматически пробрасывает порты:

```json
// В devcontainer.json
{
    "forwardPorts": [3000, 5000, 8000],
    "portsAttributes": {
        "3000": {
            "label": "Frontend",
            "onAutoForward": "openBrowser"
        },
        "5000": {
            "label": "API",
            "onAutoForward": "notify"
        },
        "8000": {
            "label": "Docs",
            "onAutoForward": "silent"
        }
    }
}
```

### Visibility портов

- **Private** (по умолчанию) - только вы
- **Org members** - члены организации
- **Public** - все (для демо)

```bash
# Изменить visibility через CLI
gh codespace ports visibility 3000:public -c CODESPACE_NAME
```

## Работа с VS Code Desktop

### Подключение к Codespace

1. Установите расширение **GitHub Codespaces**
2. Нажмите на иконку Remote Explorer
3. Выберите "GitHub Codespaces"
4. Нажмите Connect на нужном codespace

### Синхронизация настроек

VS Code синхронизирует:
- Расширения
- Настройки
- Keybindings
- Snippets
- UI State

```json
// Settings Sync в Codespaces
{
    "settingsSync.ignoredExtensions": [],
    "settingsSync.ignoredSettings": []
}
```

## Secrets и переменные окружения

### Настройка secrets

1. **Settings** > **Codespaces** > **Secrets**
2. **New secret**
3. Укажите имя и значение
4. Выберите репозитории с доступом

### Использование в devcontainer

```json
{
    "containerEnv": {
        "DATABASE_URL": "${localEnv:DATABASE_URL}"
    },
    "remoteEnv": {
        "API_KEY": "${localEnv:API_KEY}"
    }
}
```

### Доступ в коде

```python
import os

database_url = os.environ.get("DATABASE_URL")
api_key = os.environ.get("API_KEY")
```

## Типы машин

| Тип | vCPUs | RAM | Storage |
|-----|-------|-----|---------|
| 2-core | 2 | 8 GB | 32 GB |
| 4-core | 4 | 16 GB | 32 GB |
| 8-core | 8 | 32 GB | 64 GB |
| 16-core | 16 | 64 GB | 128 GB |
| 32-core | 32 | 128 GB | 128 GB |

### Выбор типа машины

```json
// В devcontainer.json
{
    "hostRequirements": {
        "cpus": 4,
        "memory": "16gb",
        "storage": "64gb"
    }
}
```

## Лимиты и тарификация

### Бесплатный план

- 120 core-hours/месяц для личных аккаунтов
- 15 GB storage/месяц

### Расчет core-hours

```
Core-hours = количество ядер × часы использования

Пример:
- 2-core машина, 10 часов = 20 core-hours
- 4-core машина, 5 часов = 20 core-hours
```

### Экономия ресурсов

```json
// Автоматическая остановка
{
    "codespaces.defaultIdleTimeout": 30
}
```

## Полезные практики

### 1. Используйте .devcontainer

```
.devcontainer/
├── devcontainer.json
├── Dockerfile
├── docker-compose.yml (опционально)
└── scripts/
    └── post-create.sh
```

### 2. Оптимизируйте prebuilds

```json
{
    // Тяжелые операции в onCreate (включается в prebuild)
    "onCreateCommand": "npm ci && npm run build",

    // Легкие операции в postCreate
    "postCreateCommand": "npm run prepare"
}
```

### 3. Используйте features вместо ручной установки

```json
{
    "features": {
        "ghcr.io/devcontainers/features/python:1": {},
        "ghcr.io/devcontainers/features/node:1": {},
        "ghcr.io/devcontainers/features/docker-in-docker:2": {},
        "ghcr.io/devcontainers/features/github-cli:1": {}
    }
}
```

### 4. Настройте dotfiles

В **Settings** > **Codespaces** > **Dotfiles** укажите репозиторий с вашими dotfiles.

```bash
# Пример структуры dotfiles репозитория
dotfiles/
├── .bashrc
├── .gitconfig
├── .vimrc
└── install.sh
```

### 5. Используйте GitHub CLI внутри Codespace

```bash
# Уже аутентифицирован!
gh pr create
gh issue list
gh repo view
```

## Полезные ссылки

- [GitHub Codespaces Documentation](https://docs.github.com/en/codespaces)
- [Dev Container Specification](https://containers.dev/)
- [Available Features](https://containers.dev/features)
- [VS Code Remote Development](https://code.visualstudio.com/docs/remote/codespaces)

---
[prev: 03-github-packages](./03-github-packages.md) | [next: 05-github-sponsors](./05-github-sponsors.md)