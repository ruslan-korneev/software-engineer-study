# GitHub Actions

[prev: 09-github-cli](./09-github-cli.md) | [next: 01-github-pages](11-github-features/01-github-pages.md)
---

## Что такое GitHub Actions

**GitHub Actions** — это встроенная CI/CD платформа GitHub, которая позволяет автоматизировать процессы сборки, тестирования и деплоя приложений прямо из репозитория. С помощью GitHub Actions можно создавать автоматизированные рабочие процессы (workflows), которые запускаются в ответ на различные события в репозитории.

### Преимущества GitHub Actions

- **Интеграция с GitHub** — не нужны сторонние CI/CD сервисы
- **Бесплатный тарифный план** — для публичных репозиториев неограниченно, для приватных — 2000 минут/месяц
- **Marketplace** — тысячи готовых actions от сообщества
- **Матричные сборки** — тестирование на разных ОС и версиях
- **Секреты** — безопасное хранение чувствительных данных

---

## Основные концепции

### Workflows (Рабочие процессы)

**Workflow** — это автоматизированный процесс, описанный в YAML-файле. Один репозиторий может содержать несколько workflows для разных целей (тестирование, деплой, линтинг и т.д.).

```
.github/
└── workflows/
    ├── ci.yml           # Continuous Integration
    ├── deploy.yml       # Деплой на продакшн
    └── lint.yml         # Проверка кода
```

### Events (События-триггеры)

**Events** — это события в репозитории, которые запускают workflow. Примеры:
- `push` — push в ветку
- `pull_request` — создание/обновление PR
- `schedule` — запуск по расписанию (cron)
- `workflow_dispatch` — ручной запуск

### Jobs (Задачи)

**Job** — это набор шагов, выполняемых на одном runner. Jobs по умолчанию выполняются параллельно, но можно настроить зависимости между ними.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    # шаги build

  test:
    runs-on: ubuntu-latest
    needs: build  # test запустится только после build
    # шаги test
```

### Steps (Шаги)

**Step** — это отдельная команда или action внутри job. Шаги выполняются последовательно в рамках одного job.

```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v4

  - name: Run tests
    run: npm test
```

### Actions (Переиспользуемые действия)

**Action** — это переиспользуемый модуль, который можно подключить в workflow. Actions бывают:
- **Официальные** (`actions/checkout`, `actions/setup-node`)
- **От сообщества** (из GitHub Marketplace)
- **Собственные** (можно создать свои)

```yaml
# Использование action
- uses: actions/checkout@v4

# Использование action с параметрами
- uses: actions/setup-node@v4
  with:
    node-version: '20'
```

### Runners (Исполнители)

**Runner** — это сервер, на котором выполняются jobs. GitHub предоставляет hosted runners:
- `ubuntu-latest` — Ubuntu Linux
- `windows-latest` — Windows Server
- `macos-latest` — macOS

Также можно настроить **self-hosted runners** — собственные серверы.

---

## Структура workflow файла

Workflow файлы располагаются в директории `.github/workflows/` и используют YAML синтаксис.

### Базовая структура

```yaml
# Название workflow (отображается в интерфейсе GitHub)
name: CI Pipeline

# События, запускающие workflow
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

# Переменные окружения для всего workflow
env:
  NODE_ENV: test

# Определение задач
jobs:
  # Уникальный идентификатор job
  build:
    # Тип runner
    runs-on: ubuntu-latest

    # Шаги выполнения
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
```

### Ключевые элементы YAML

| Элемент | Описание |
|---------|----------|
| `name` | Название workflow |
| `on` | События-триггеры |
| `env` | Переменные окружения |
| `jobs` | Определение задач |
| `runs-on` | Тип runner для job |
| `steps` | Последовательность шагов |
| `uses` | Использование action |
| `run` | Выполнение shell-команды |
| `with` | Параметры для action |
| `needs` | Зависимость от других jobs |
| `if` | Условное выполнение |

---

## События и триггеры

### push

Запуск при push в указанные ветки или теги:

```yaml
on:
  push:
    branches:
      - main
      - 'release/**'  # Все ветки, начинающиеся с release/
    tags:
      - 'v*'          # Все теги, начинающиеся с v
    paths:
      - 'src/**'      # Только если изменения в src/
    paths-ignore:
      - '**.md'       # Игнорировать изменения в .md файлах
```

### pull_request

Запуск при событиях с pull request:

```yaml
on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]
```

Типы событий PR:
- `opened` — PR открыт
- `synchronize` — новые коммиты в PR
- `reopened` — PR переоткрыт
- `closed` — PR закрыт
- `merged` — PR смержен

### schedule (cron)

Запуск по расписанию с использованием cron-синтаксиса:

```yaml
on:
  schedule:
    # Каждый день в 6:00 UTC
    - cron: '0 6 * * *'
    # Каждый понедельник в 9:00 UTC
    - cron: '0 9 * * 1'
```

Формат cron: `минуты часы день_месяца месяц день_недели`

```
┌───────────── минуты (0-59)
│ ┌───────────── часы (0-23)
│ │ ┌───────────── день месяца (1-31)
│ │ │ ┌───────────── месяц (1-12)
│ │ │ │ ┌───────────── день недели (0-6, 0 = воскресенье)
│ │ │ │ │
* * * * *
```

### workflow_dispatch

Ручной запуск workflow с возможностью передачи параметров:

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Окружение для деплоя'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
      debug:
        description: 'Включить отладку'
        required: false
        type: boolean
        default: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to ${{ github.event.inputs.environment }}
        run: |
          echo "Deploying to ${{ github.event.inputs.environment }}"
          echo "Debug mode: ${{ github.event.inputs.debug }}"
```

### release

Запуск при создании релиза:

```yaml
on:
  release:
    types: [published, created, released]
```

### Комбинирование триггеров

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:
  schedule:
    - cron: '0 0 * * 0'  # Каждое воскресенье в полночь
```

---

## Секреты и переменные окружения

### Секреты (Secrets)

Секреты используются для хранения чувствительных данных (токены, пароли, ключи API). Они шифруются и не отображаются в логах.

**Добавление секретов:**
1. Settings → Secrets and variables → Actions
2. New repository secret
3. Указать имя и значение

**Использование в workflow:**

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to server
        env:
          API_TOKEN: ${{ secrets.API_TOKEN }}
          SSH_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
        run: |
          echo "Using secret token"
          # Секреты автоматически маскируются в логах
```

### Уровни секретов

| Уровень | Область видимости |
|---------|-------------------|
| Repository | Один репозиторий |
| Environment | Конкретное окружение (staging, production) |
| Organization | Все репозитории организации |

### Environment Secrets

```yaml
jobs:
  deploy-prod:
    runs-on: ubuntu-latest
    environment: production  # Использует секреты окружения production
    steps:
      - name: Deploy
        env:
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
        run: ./deploy.sh
```

### Переменные окружения

**Встроенные переменные GitHub:**

```yaml
steps:
  - name: Show context
    run: |
      echo "Repository: ${{ github.repository }}"
      echo "Branch: ${{ github.ref_name }}"
      echo "Commit SHA: ${{ github.sha }}"
      echo "Actor: ${{ github.actor }}"
      echo "Run ID: ${{ github.run_id }}"
      echo "Workflow: ${{ github.workflow }}"
```

**Определение собственных переменных:**

```yaml
# На уровне workflow
env:
  APP_NAME: my-app

jobs:
  build:
    runs-on: ubuntu-latest
    # На уровне job
    env:
      BUILD_ENV: production
    steps:
      - name: Build
        # На уровне step
        env:
          STEP_VAR: value
        run: echo "$APP_NAME in $BUILD_ENV with $STEP_VAR"
```

### Variables (не секретные)

Для несекретных данных используйте Variables (Settings → Variables):

```yaml
steps:
  - name: Use variable
    run: echo "Version: ${{ vars.APP_VERSION }}"
```

---

## Матричные сборки

Матрицы позволяют запускать job с разными комбинациями параметров — например, тестировать на разных версиях языка или ОС.

### Базовая матрица

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [16, 18, 20]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci
      - run: npm test
```

Этот workflow создаст 3 параллельных job: для Node.js 16, 18 и 20.

### Многомерная матрица

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.9', '3.10', '3.11', '3.12']

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install -r requirements.txt
      - run: pytest
```

Это создаст 12 jobs (3 ОС × 4 версии Python).

### Исключение и добавление комбинаций

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [16, 18, 20]

    # Исключить определённые комбинации
    exclude:
      - os: windows-latest
        node: 16

    # Добавить специфичные комбинации
    include:
      - os: ubuntu-latest
        node: 20
        experimental: true
```

### Параметры стратегии

```yaml
strategy:
  # Продолжать другие jobs при падении одного
  fail-fast: false

  # Максимальное количество параллельных jobs
  max-parallel: 2

  matrix:
    version: [1, 2, 3]
```

---

## Артефакты и кэширование

### Артефакты (Artifacts)

Артефакты — это файлы, созданные во время выполнения workflow, которые нужно сохранить (логи, бинарники, отчёты о тестах).

**Загрузка артефактов:**

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
          retention-days: 5  # Хранить 5 дней
```

**Скачивание артефактов в другом job:**

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build
          path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download build
        uses: actions/download-artifact@v4
        with:
          name: build
          path: dist/

      - name: Deploy
        run: ./deploy.sh dist/
```

### Кэширование зависимостей

Кэширование ускоряет workflow, сохраняя зависимости между запусками.

**Кэширование npm:**

```yaml
steps:
  - uses: actions/checkout@v4

  - name: Cache node modules
    uses: actions/cache@v4
    with:
      path: ~/.npm
      key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-node-

  - run: npm ci
```

**Кэширование pip:**

```yaml
steps:
  - uses: actions/checkout@v4

  - name: Cache pip
    uses: actions/cache@v4
    with:
      path: ~/.cache/pip
      key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
      restore-keys: |
        ${{ runner.os }}-pip-

  - run: pip install -r requirements.txt
```

**Встроенное кэширование в setup actions:**

```yaml
# Node.js с кэшированием
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'

# Python с кэшированием pip
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'
```

---

## Примеры workflow

### CI для Node.js

```yaml
name: Node.js CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - run: npm ci
      - run: npm test

      - name: Upload coverage
        if: matrix.node-version == 20
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/

  build:
    needs: [lint, test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build

      - uses: actions/upload-artifact@v4
        with:
          name: build
          path: dist/
```

### CI для Python

```yaml
name: Python CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.10', '3.11', '3.12']

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Lint with ruff
        run: ruff check .

      - name: Type check with mypy
        run: mypy src/

      - name: Test with pytest
        run: |
          pytest --cov=src --cov-report=xml

      - name: Upload coverage to Codecov
        if: matrix.python-version == '3.12'
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage.xml
```

### Автодеплой на сервер

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build

      - uses: actions/upload-artifact@v4
        with:
          name: production-build
          path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Download build
        uses: actions/download-artifact@v4
        with:
          name: production-build
          path: dist/

      - name: Deploy to server via SSH
        env:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
          SERVER_HOST: ${{ secrets.SERVER_HOST }}
          SERVER_USER: ${{ secrets.SERVER_USER }}
        run: |
          mkdir -p ~/.ssh
          echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan -H $SERVER_HOST >> ~/.ssh/known_hosts

          rsync -avz --delete dist/ $SERVER_USER@$SERVER_HOST:/var/www/app/

          ssh $SERVER_USER@$SERVER_HOST "sudo systemctl restart app"
```

### Docker Build и Push

```yaml
name: Docker Build

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: username/app
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Автоматический релиз

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Generate changelog
        id: changelog
        uses: orhun/git-cliff-action@v3
        with:
          config: cliff.toml
          args: --latest --strip header

      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          body: ${{ steps.changelog.outputs.content }}
          draft: false
          prerelease: ${{ contains(github.ref, 'alpha') || contains(github.ref, 'beta') }}
```

---

## Полезные паттерны

### Условное выполнение

```yaml
steps:
  # Выполнить только на main ветке
  - name: Deploy
    if: github.ref == 'refs/heads/main'
    run: ./deploy.sh

  # Выполнить только при успехе предыдущих шагов
  - name: Notify success
    if: success()
    run: echo "All good!"

  # Выполнить при ошибке
  - name: Notify failure
    if: failure()
    run: echo "Something went wrong"

  # Выполнить всегда
  - name: Cleanup
    if: always()
    run: ./cleanup.sh
```

### Переиспользование workflow (Reusable Workflows)

**Определение переиспользуемого workflow (.github/workflows/reusable-build.yml):**

```yaml
name: Reusable Build

on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string
    secrets:
      npm-token:
        required: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
      - run: npm ci
        env:
          NPM_TOKEN: ${{ secrets.npm-token }}
      - run: npm run build
```

**Вызов переиспользуемого workflow:**

```yaml
name: CI

on: push

jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      node-version: '20'
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
```

### Concurrency (предотвращение параллельных запусков)

```yaml
name: Deploy

on:
  push:
    branches: [main]

concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: true  # Отменить предыдущий запуск

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

---

## Отладка и мониторинг

### Просмотр логов

- В интерфейсе GitHub: Actions → выбрать workflow → выбрать job
- Логи можно скачать как zip-архив

### Debug режим

Для включения подробного логирования добавьте секреты:
- `ACTIONS_RUNNER_DEBUG`: `true`
- `ACTIONS_STEP_DEBUG`: `true`

### Локальное тестирование с act

[act](https://github.com/nektos/act) — инструмент для локального запуска GitHub Actions:

```bash
# Установка
brew install act  # macOS

# Запуск всех workflows
act

# Запуск конкретного события
act push

# Запуск конкретного job
act -j build
```

---

## Лучшие практики

1. **Используйте конкретные версии actions** — `@v4` вместо `@main`
2. **Кэшируйте зависимости** — значительно ускоряет выполнение
3. **Разделяйте jobs** — параллельное выполнение быстрее
4. **Используйте environments** — для разных окружений (staging, production)
5. **Храните секреты безопасно** — никогда не хардкодьте чувствительные данные
6. **Ограничивайте permissions** — минимально необходимые права
7. **Используйте concurrency** — предотвращайте конфликты деплоя
8. **Документируйте workflows** — комментарии в YAML

---

## Дополнительные ресурсы

- [Официальная документация GitHub Actions](https://docs.github.com/en/actions)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)
- [Awesome Actions](https://github.com/sdras/awesome-actions) — коллекция полезных actions
- [act](https://github.com/nektos/act) — локальный запуск actions

---
[prev: 09-github-cli](./09-github-cli.md) | [next: 01-github-pages](11-github-features/01-github-pages.md)