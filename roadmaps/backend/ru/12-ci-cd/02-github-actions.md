# GitHub Actions

[prev: 01-concepts](./01-concepts.md) | [next: 03-best-practices](./03-best-practices.md)

---

## Введение

**GitHub Actions** — это платформа для автоматизации CI/CD, встроенная непосредственно в GitHub. Она позволяет создавать автоматизированные рабочие процессы (workflows) прямо в репозитории.

## Основные концепции

### Структура GitHub Actions

```
GitHub Actions
├── Workflow (рабочий процесс)
│   ├── Event (триггер)
│   ├── Job (задача)
│   │   ├── Runner (исполнитель)
│   │   └── Steps (шаги)
│   │       ├── Action (действие)
│   │       └── Command (команда)
│   └── Job
│       └── Steps
└── Workflow
```

### Ключевые термины

| Термин | Описание |
|--------|----------|
| **Workflow** | Автоматизированный процесс, определённый в YAML-файле |
| **Event** | Событие, запускающее workflow (push, PR, schedule и т.д.) |
| **Job** | Набор шагов, выполняющихся на одном runner |
| **Step** | Отдельная задача внутри job |
| **Action** | Переиспользуемый компонент (готовый или кастомный) |
| **Runner** | Виртуальная машина, выполняющая job |

## Структура Workflow-файла

### Расположение

Все workflow-файлы должны находиться в директории:
```
.github/workflows/
├── ci.yml
├── cd.yml
└── release.yml
```

### Базовый синтаксис

```yaml
# .github/workflows/ci.yml

# Имя workflow (отображается в UI)
name: CI Pipeline

# Триггеры запуска
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

# Переменные окружения (глобальные)
env:
  NODE_VERSION: '20'

# Определение задач
jobs:
  # Имя задачи
  build:
    # На каком runner выполнять
    runs-on: ubuntu-latest

    # Шаги задачи
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build
```

## События (Events)

### Push и Pull Request

```yaml
on:
  # При push в указанные ветки
  push:
    branches:
      - main
      - 'feature/**'    # Wildcard pattern
    paths:
      - 'src/**'        # Только если изменились файлы в src/
      - '!src/**/*.md'  # Кроме markdown файлов

  # При создании/обновлении PR
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]
```

### Расписание (Cron)

```yaml
on:
  schedule:
    # Каждый день в 3:00 UTC
    - cron: '0 3 * * *'

    # Каждый понедельник в 9:00 UTC
    - cron: '0 9 * * 1'

# Формат cron: минута час день-месяца месяц день-недели
# *    *    *     *      *
# │    │    │     │      │
# │    │    │     │      └── День недели (0-6, 0 = воскресенье)
# │    │    │     └── Месяц (1-12)
# │    │    └── День месяца (1-31)
# │    └── Час (0-23)
# └── Минута (0-59)
```

### Ручной запуск

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
      debug:
        description: 'Enable debug mode'
        required: false
        type: boolean
        default: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to ${{ inputs.environment }}
        run: |
          echo "Deploying to ${{ inputs.environment }}"
          echo "Debug mode: ${{ inputs.debug }}"
```

### Другие события

```yaml
on:
  # При создании релиза
  release:
    types: [published]

  # При создании issue
  issues:
    types: [opened, labeled]

  # При изменении wiki
  gollum:

  # При изменении проекта
  project:
    types: [created]

  # Вызов из другого workflow
  workflow_call:
    inputs:
      config:
        type: string
        required: true
```

## Jobs (Задачи)

### Параллельное выполнение

По умолчанию jobs выполняются параллельно:

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test

  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
```

### Зависимости между jobs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build

  test:
    needs: build  # Ждёт завершения build
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  deploy:
    needs: [build, test]  # Ждёт обоих
    runs-on: ubuntu-latest
    steps:
      - run: npm run deploy
```

### Визуализация зависимостей

```
                    ┌───────┐
                    │ build │
                    └───┬───┘
                        │
              ┌─────────┴─────────┐
              │                   │
         ┌────▼────┐         ┌────▼────┐
         │  test   │         │  lint   │
         └────┬────┘         └────┬────┘
              │                   │
              └─────────┬─────────┘
                        │
                   ┌────▼────┐
                   │ deploy  │
                   └─────────┘
```

### Matrix Strategy

Запуск job с разными параметрами:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]
        os: [ubuntu-latest, windows-latest, macos-latest]
      fail-fast: false  # Продолжать если один упал

    steps:
      - uses: actions/checkout@v4

      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}

      - run: npm test
```

Это создаст 9 параллельных jobs (3 версии x 3 ОС).

### Условное выполнение

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    # Выполнять только для main ветки
    if: github.ref == 'refs/heads/main'
    steps:
      - run: npm run deploy

  notify:
    runs-on: ubuntu-latest
    needs: deploy
    # Выполнять даже если предыдущий job упал
    if: always()
    steps:
      - run: echo "Deploy finished"
```

## Steps (Шаги)

### Использование Actions

```yaml
steps:
  # Официальный action
  - uses: actions/checkout@v4

  # Action с параметрами
  - uses: actions/setup-node@v4
    with:
      node-version: '20'
      cache: 'npm'

  # Action из другого репозитория
  - uses: owner/repo@v1

  # Action из локального пути
  - uses: ./.github/actions/my-action
```

### Выполнение команд

```yaml
steps:
  # Одна команда
  - run: npm install

  # Несколько команд
  - run: |
      npm install
      npm run build
      npm test

  # С указанием shell
  - run: echo "Hello"
    shell: bash

  # С рабочей директорией
  - run: npm install
    working-directory: ./frontend

  # С переменными окружения
  - run: npm test
    env:
      CI: true
      NODE_ENV: test
```

### Именование и условия

```yaml
steps:
  - name: Install dependencies
    id: install
    run: npm ci

  - name: Run tests
    if: steps.install.outcome == 'success'
    run: npm test

  - name: Upload coverage
    if: success()  # Только если предыдущие успешны
    uses: codecov/codecov-action@v3

  - name: Notify on failure
    if: failure()  # Только если есть ошибки
    run: echo "Something failed!"
```

## Переменные и секреты

### Переменные окружения

```yaml
# Глобальные для всего workflow
env:
  NODE_VERSION: '20'
  APP_NAME: my-app

jobs:
  build:
    # Для конкретного job
    env:
      BUILD_ENV: production

    steps:
      - name: Build
        # Для конкретного step
        env:
          SOME_VAR: value
        run: |
          echo "Node: $NODE_VERSION"
          echo "App: $APP_NAME"
          echo "Env: $BUILD_ENV"
```

### Секреты

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        env:
          # Секреты из настроек репозитория
          API_KEY: ${{ secrets.API_KEY }}
          AWS_ACCESS_KEY: ${{ secrets.AWS_ACCESS_KEY_ID }}
        run: |
          # Секреты маскируются в логах
          deploy.sh
```

### Контекстные переменные

```yaml
steps:
  - name: Show context info
    run: |
      echo "Repository: ${{ github.repository }}"
      echo "Branch: ${{ github.ref_name }}"
      echo "Commit SHA: ${{ github.sha }}"
      echo "Actor: ${{ github.actor }}"
      echo "Event: ${{ github.event_name }}"
      echo "Run ID: ${{ github.run_id }}"
      echo "Run Number: ${{ github.run_number }}"
```

### Outputs между steps

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.value }}

    steps:
      - name: Get version
        id: version
        run: |
          VERSION=$(cat package.json | jq -r .version)
          echo "value=$VERSION" >> $GITHUB_OUTPUT

      - name: Use version
        run: echo "Version is ${{ steps.version.outputs.value }}"

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy version
        run: echo "Deploying ${{ needs.build.outputs.version }}"
```

## Артефакты и кэширование

### Upload Artifacts

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: |
            dist/
            !dist/**/*.map
          retention-days: 7
```

### Download Artifacts

```yaml
jobs:
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-output
          path: ./dist

      - name: Deploy
        run: deploy.sh ./dist
```

### Кэширование зависимостей

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Кэширование npm
      - name: Cache npm
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            npm-${{ runner.os }}-

      - run: npm ci
```

### Кэширование для разных менеджеров

```yaml
# Python + pip
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: pip-${{ runner.os }}-${{ hashFiles('**/requirements.txt') }}

# Go modules
- uses: actions/cache@v4
  with:
    path: ~/go/pkg/mod
    key: go-${{ runner.os }}-${{ hashFiles('**/go.sum') }}

# Rust + Cargo
- uses: actions/cache@v4
  with:
    path: |
      ~/.cargo/registry
      ~/.cargo/git
      target/
    key: cargo-${{ runner.os }}-${{ hashFiles('**/Cargo.lock') }}
```

## Примеры пайплайнов

### Python Backend (FastAPI)

```yaml
name: Python CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  PYTHON_VERSION: '3.11'

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Install dependencies
        run: |
          pip install ruff black mypy

      - name: Run Ruff
        run: ruff check .

      - name: Run Black
        run: black --check .

      - name: Run MyPy
        run: mypy src/

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: 'pip'

      - name: Install dependencies
        run: pip install -r requirements.txt -r requirements-dev.txt

      - name: Run tests
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test_db
        run: pytest --cov=src --cov-report=xml

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.xml

  build:
    needs: [lint, test]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            user/app:latest
            user/app:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Node.js Backend (Express/NestJS)

```yaml
name: Node.js CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest

    services:
      redis:
        image: redis:7
        ports:
          - 6379:6379
      mongo:
        image: mongo:6
        ports:
          - 27017:27017

    strategy:
      matrix:
        node-version: [18, 20]

    steps:
      - uses: actions/checkout@v4

      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint

      - name: Run Prettier check
        run: npm run format:check

      - name: Run unit tests
        run: npm run test:unit

      - name: Run integration tests
        env:
          REDIS_URL: redis://localhost:6379
          MONGODB_URL: mongodb://localhost:27017/test
        run: npm run test:integration

      - name: Run e2e tests
        run: npm run test:e2e

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run npm audit
        run: npm audit --audit-level=high

      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  deploy:
    needs: [lint-and-test, security]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to production
        run: |
          # Deployment logic here
          echo "Deploying to production..."
```

### Go Backend

```yaml
name: Go CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'
          cache: true

      - name: Verify dependencies
        run: go mod verify

      - name: Run golangci-lint
        uses: golangci/golangci-lint-action@v4
        with:
          version: latest

      - name: Run tests
        run: go test -race -coverprofile=coverage.out -covermode=atomic ./...

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.out

      - name: Build
        run: go build -v ./...

  release:
    needs: build
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/')

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'

      - name: Run GoReleaser
        uses: goreleaser/goreleaser-action@v5
        with:
          version: latest
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Переиспользуемые Workflows

### Создание reusable workflow

```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      version:
        required: true
        type: string
    secrets:
      deploy_key:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}

    steps:
      - uses: actions/checkout@v4

      - name: Deploy
        env:
          DEPLOY_KEY: ${{ secrets.deploy_key }}
        run: |
          echo "Deploying ${{ inputs.version }} to ${{ inputs.environment }}"
```

### Использование reusable workflow

```yaml
# .github/workflows/main.yml
name: Main Pipeline

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.value }}
    steps:
      - id: version
        run: echo "value=1.0.0" >> $GITHUB_OUTPUT

  deploy-staging:
    needs: build
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
      version: ${{ needs.build.outputs.version }}
    secrets:
      deploy_key: ${{ secrets.STAGING_DEPLOY_KEY }}

  deploy-production:
    needs: [build, deploy-staging]
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
      version: ${{ needs.build.outputs.version }}
    secrets:
      deploy_key: ${{ secrets.PROD_DEPLOY_KEY }}
```

## Environments и защита

### Настройка окружений

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com

    steps:
      - name: Deploy
        run: deploy.sh
```

### Защита окружений (настройка в UI)

1. **Required reviewers** — обязательное одобрение
2. **Wait timer** — задержка перед деплоем
3. **Deployment branches** — ограничение веток
4. **Environment secrets** — секреты для окружения

## Self-hosted Runners

### Когда использовать

- Нужно специфичное железо (GPU, много RAM)
- Доступ к внутренней сети
- Снижение затрат на минуты
- Соблюдение требований безопасности

### Конфигурация

```yaml
jobs:
  build:
    runs-on: self-hosted  # Использовать self-hosted runner

  test:
    runs-on: [self-hosted, linux, gpu]  # С labels
```

## Отладка и мониторинг

### Debug logging

```yaml
# Включить debug логи
env:
  ACTIONS_STEP_DEBUG: true
  ACTIONS_RUNNER_DEBUG: true
```

### Просмотр контекста

```yaml
steps:
  - name: Dump context
    env:
      GITHUB_CONTEXT: ${{ toJson(github) }}
      JOB_CONTEXT: ${{ toJson(job) }}
      STEPS_CONTEXT: ${{ toJson(steps) }}
    run: |
      echo "$GITHUB_CONTEXT"
      echo "$JOB_CONTEXT"
```

## Типичные ошибки

### 1. Неправильное кэширование

```yaml
# Плохо: кэш никогда не инвалидируется
- uses: actions/cache@v4
  with:
    path: node_modules
    key: node-modules

# Хорошо: кэш привязан к lock-файлу
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('package-lock.json') }}
```

### 2. Утечка секретов

```yaml
# Плохо: секрет может попасть в логи
- run: curl -H "Authorization: ${{ secrets.TOKEN }}" https://api.example.com

# Хорошо: используйте маскирование
- run: |
    echo "::add-mask::${{ secrets.TOKEN }}"
    curl -H "Authorization: ${{ secrets.TOKEN }}" https://api.example.com
```

### 3. Отсутствие таймаутов

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30  # Ограничение времени

    steps:
      - name: Long running step
        timeout-minutes: 10  # Для конкретного шага
        run: npm test
```

## Полезные Actions

| Action | Описание |
|--------|----------|
| `actions/checkout` | Клонирование репозитория |
| `actions/setup-node` | Настройка Node.js |
| `actions/setup-python` | Настройка Python |
| `actions/cache` | Кэширование |
| `actions/upload-artifact` | Загрузка артефактов |
| `docker/build-push-action` | Сборка и push Docker |
| `softprops/action-gh-release` | Создание GitHub Release |
| `codecov/codecov-action` | Загрузка coverage |

## Заключение

GitHub Actions — мощный и гибкий инструмент для CI/CD. Основные преимущества:

1. **Интеграция с GitHub** — единая экосистема
2. **YAML-конфигурация** — понятный и версионируемый формат
3. **Marketplace** — тысячи готовых actions
4. **Matrix builds** — тестирование на разных платформах
5. **Бесплатный tier** — для публичных репозиториев

## Дополнительные ресурсы

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)
- [Awesome Actions](https://github.com/sdras/awesome-actions)

---

[prev: 01-concepts](./01-concepts.md) | [next: 03-best-practices](./03-best-practices.md)
