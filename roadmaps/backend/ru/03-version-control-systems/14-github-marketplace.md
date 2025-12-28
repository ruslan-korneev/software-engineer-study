# GitHub Marketplace

[prev: 13-github-security](./13-github-security.md) | [next: 01-reflog](15-advanced-git/01-reflog.md)
---

## Введение

GitHub Marketplace — это централизованная платформа для поиска, установки и управления инструментами, которые расширяют функциональность GitHub. Marketplace объединяет тысячи приложений и actions, созданных как самим GitHub, так и сторонними разработчиками.

### Основные возможности Marketplace

- **Интеграция** — приложения напрямую интегрируются с репозиториями и организациями
- **Централизованное управление** — все установленные приложения можно управлять из одного места
- **Верифицированные издатели** — проверенные разработчики имеют специальный значок
- **Бесплатные и платные планы** — многие приложения предлагают бесплатные тарифы для open-source проектов
- **Простая установка** — установка в несколько кликов с настройкой прав доступа

### Зачем использовать Marketplace

1. **Автоматизация** — CI/CD, автоматическое тестирование, деплой
2. **Качество кода** — линтеры, анализаторы, code review боты
3. **Безопасность** — сканирование уязвимостей, проверка зависимостей
4. **Управление проектами** — интеграция с task-трекерами, time-tracking
5. **Мониторинг** — отслеживание производительности, алерты

---

## Категории Marketplace

### Apps vs Actions

GitHub Marketplace предлагает два типа продуктов:

| Характеристика | GitHub Apps | GitHub Actions |
|----------------|-------------|----------------|
| Тип | Внешние приложения | Переиспользуемые шаги workflow |
| Установка | На уровне организации/репозитория | В YAML-файлах workflow |
| Права доступа | Гранулярные разрешения | Наследуют права workflow |
| Выполнение | На серверах разработчика | На GitHub runners |
| Биллинг | Подписка через GitHub | Входит в GitHub Actions minutes |

### Категории приложений

1. **Code quality** — SonarCloud, Codacy, CodeClimate
2. **Code review** — Reviewable, Pull Approve, LGTM
3. **Continuous integration** — CircleCI, Travis CI, Jenkins X
4. **Dependency management** — Dependabot, Renovate, Snyk
5. **Deployment** — Heroku, Netlify, Vercel
6. **IDEs** — GitPod, Codespaces, Replit
7. **Learning** — GitHub Learning Lab
8. **Localization** — Crowdin, Transifex
9. **Mobile CI/CD** — Bitrise, Fastlane
10. **Monitoring** — Datadog, New Relic, Sentry
11. **Project management** — ZenHub, Jira, Trello
12. **Publishing** — npm, PyPI integrations
13. **Security** — Snyk, WhiteSource, GitGuardian
14. **Support** — Freshdesk, Zendesk integrations
15. **Testing** — Codecov, Coveralls, BrowserStack

---

## Популярные приложения

### CI/CD приложения

#### CircleCI
Мощная платформа для continuous integration и delivery.

```yaml
# .circleci/config.yml
version: 2.1
jobs:
  build:
    docker:
      - image: cimg/node:18.0
    steps:
      - checkout
      - run: npm install
      - run: npm test
workflows:
  version: 2
  build_and_test:
    jobs:
      - build
```

#### Travis CI
Классическая CI-система с простой конфигурацией.

#### Jenkins X
Kubernetes-native CI/CD для облачных приложений.

### Code Quality приложения

#### SonarCloud
Автоматический анализ качества кода и поиск багов.

- Поддержка 25+ языков программирования
- Обнаружение code smells, bugs, vulnerabilities
- Quality gates для PR
- Бесплатно для open-source

#### Codacy
Автоматизированный code review с поддержкой множества языков.

#### CodeClimate
Анализ maintainability и test coverage.

### Dependency Management

#### Dependabot (встроен в GitHub)
Автоматическое обновление зависимостей.

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    reviewers:
      - "team-leads"
    labels:
      - "dependencies"
      - "automated"
```

#### Renovate
Продвинутая альтернатива Dependabot с гибкой настройкой.

#### Snyk
Поиск уязвимостей в зависимостях с автоматическими PR для исправления.

### Project Management

#### ZenHub
Agile project management прямо в GitHub.

- Kanban-доски
- Roadmaps
- Reporting и metrics
- Sprint planning

#### Jira Integration
Связь GitHub коммитов и PR с Jira issues.

### Security приложения

#### GitGuardian
Обнаружение секретов в коде (API keys, passwords, tokens).

#### WhiteSource Bolt
Сканирование open-source зависимостей на уязвимости.

---

## GitHub Apps

### Что такое GitHub Apps

GitHub Apps — это официальный способ интеграции внешних сервисов с GitHub. Они работают от своего имени (как бот), а не от имени пользователя.

### Преимущества GitHub Apps

1. **Гранулярные разрешения** — запрашивают только необходимый доступ
2. **Webhook events** — получают уведомления о событиях в репозитории
3. **Rate limits** — собственные лимиты, независимые от пользователя
4. **Installation tokens** — временные токены для API-запросов
5. **Действуют как бот** — не занимают лицензию пользователя

### Разрешения GitHub Apps

```
Repository permissions:
├── Actions: Read and write
├── Contents: Read-only
├── Issues: Read and write
├── Pull requests: Read and write
├── Metadata: Read-only (mandatory)
└── Webhooks: Read and write

Organization permissions:
├── Members: Read-only
├── Administration: Read-only
└── Projects: Read and write

Account permissions:
├── Email addresses: Read-only
└── Profile: Read-only
```

### Установка GitHub App

1. **Найти приложение** в Marketplace
2. **Выбрать план** (Free/Paid)
3. **Выбрать scope** — все репозитории или выбранные
4. **Подтвердить разрешения**
5. **Авторизовать** для своего аккаунта или организации

### Управление установленными Apps

Путь: Settings → Integrations → Applications → Installed GitHub Apps

Здесь можно:
- Изменить доступ к репозиториям
- Посмотреть используемые разрешения
- Приостановить или удалить приложение

---

## OAuth Apps

### Отличия от GitHub Apps

| Аспект | GitHub Apps | OAuth Apps |
|--------|-------------|------------|
| Действует от имени | Себя (бот) | Пользователя |
| Разрешения | Гранулярные | Широкие scopes |
| Установка | На репозиторий/организацию | На пользователя |
| Rate limits | Собственные | Делят с пользователем |
| Webhooks | Встроены | Отдельная настройка |
| Рекомендация | Предпочтительно | Legacy подход |

### OAuth Scopes

```
repo        - Full control of private repositories
repo:status - Access commit status
public_repo - Access public repositories only
admin:org   - Full control of orgs and teams
write:org   - Read and write org membership
read:org    - Read org membership
admin:repo_hook - Full control of repository hooks
user        - Full control of user profile
read:user   - Read user profile data
user:email  - Access user email addresses
gist        - Create gists
workflow    - Update GitHub Action workflows
```

### Когда использовать OAuth Apps

- Приложения, действующие от имени конкретного пользователя
- Интеграции, требующие широкий доступ
- Legacy-приложения (рекомендуется миграция на GitHub Apps)

---

## GitHub Actions в Marketplace

### Что такое Actions

GitHub Actions в Marketplace — это готовые к использованию шаги для ваших CI/CD workflows. Они инкапсулируют сложную логику в простой интерфейс.

### Поиск Actions

1. **Marketplace** — marketplace.github.com/actions
2. **GitHub Search** — поиск по репозиториям с topic `github-actions`
3. **Awesome Lists** — курируемые списки популярных actions

### Структура использования Action

```yaml
- uses: owner/repo@version
  with:
    parameter1: value1
    parameter2: value2
  env:
    ENV_VAR: value
```

### Популярные Actions

#### Checkout
```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Полная история для анализа
    token: ${{ secrets.GITHUB_TOKEN }}
```

#### Setup Node.js
```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'
```

#### Setup Python
```yaml
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'
```

#### Cache
```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

#### Upload/Download Artifacts
```yaml
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist/
    retention-days: 5

- uses: actions/download-artifact@v4
  with:
    name: build-output
    path: ./artifacts
```

### Версионирование Actions

```yaml
# Точная версия (рекомендуется для production)
- uses: actions/checkout@v4.1.1

# Major версия (получает minor и patch обновления)
- uses: actions/checkout@v4

# SHA коммита (максимальная безопасность)
- uses: actions/checkout@8ade135a41bc03ea155e62e844d188df1ea18608

# Branch (не рекомендуется)
- uses: actions/checkout@main
```

---

## Установка приложений

### Процесс поиска приложений

1. Перейти на **github.com/marketplace**
2. Использовать **фильтры**:
   - Категория (CI, Security, etc.)
   - Тип (Apps/Actions)
   - Verified creators
   - Free/Paid
3. **Изучить страницу приложения**:
   - Описание функций
   - Запрашиваемые разрешения
   - Отзывы и рейтинг
   - Pricing

### Установка GitHub App

```
1. Нажать "Set up a plan"
2. Выбрать тарифный план
3. Выбрать аккаунт/организацию
4. Настроить доступ к репозиториям:
   - All repositories
   - Only select repositories
5. Review permissions
6. Confirm installation
```

### Конфигурация приложения

После установки многие приложения требуют создание конфигурационного файла:

```yaml
# .github/app-config.yml (пример)
version: 1
rules:
  - pattern: "*.js"
    reviewers:
      - frontend-team
  - pattern: "*.py"
    reviewers:
      - backend-team
settings:
  auto_merge: true
  delete_branch: true
```

### Управление биллингом

Settings → Billing → GitHub Marketplace

- Просмотр подписок
- Изменение планов
- Отмена подписки
- История платежей

---

## Создание своих приложений

### Типы приложений

1. **GitHub App** — рекомендуемый тип
2. **OAuth App** — для действий от имени пользователя
3. **GitHub Action** — для CI/CD workflows

### Создание GitHub App

#### Шаг 1: Регистрация приложения

Settings → Developer settings → GitHub Apps → New GitHub App

```
App name: my-awesome-bot
Homepage URL: https://myapp.example.com
Callback URL: https://myapp.example.com/callback
Setup URL: https://myapp.example.com/setup (optional)
Webhook URL: https://myapp.example.com/webhooks
Webhook secret: <generated-secret>
```

#### Шаг 2: Настройка разрешений

```
Repository permissions:
- Contents: Read & write
- Issues: Read & write
- Pull requests: Read & write
- Metadata: Read-only

Subscribe to events:
- Push
- Pull request
- Issues
```

#### Шаг 3: Генерация ключей

```bash
# Скачать private key (.pem файл)
# Сгенерировать client secret для OAuth flow

# Создать JWT для аутентификации
const jwt = require('jsonwebtoken');
const privateKey = fs.readFileSync('private-key.pem');

const token = jwt.sign(
  { iss: APP_ID },
  privateKey,
  { algorithm: 'RS256', expiresIn: '10m' }
);
```

#### Шаг 4: Получение installation token

```javascript
const response = await fetch(
  `https://api.github.com/app/installations/${installationId}/access_tokens`,
  {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${jwt}`,
      'Accept': 'application/vnd.github+json'
    }
  }
);
const { token } = await response.json();
```

### Создание GitHub Action

#### Структура Action

```
my-action/
├── action.yml          # Метаданные
├── dist/
│   └── index.js        # Скомпилированный код
├── src/
│   └── index.ts        # Исходный код
├── package.json
├── README.md
└── LICENSE
```

#### action.yml

```yaml
name: 'My Awesome Action'
description: 'Does something awesome'
author: 'Your Name'

branding:
  icon: 'zap'
  color: 'blue'

inputs:
  api-key:
    description: 'API key for the service'
    required: true
  config-file:
    description: 'Path to config file'
    required: false
    default: '.config.yml'

outputs:
  result:
    description: 'The result of the action'

runs:
  using: 'node20'
  main: 'dist/index.js'
```

#### Реализация Action

```javascript
// src/index.js
const core = require('@actions/core');
const github = require('@actions/github');

async function run() {
  try {
    const apiKey = core.getInput('api-key', { required: true });
    const configFile = core.getInput('config-file');

    core.info(`Using config file: ${configFile}`);

    // Основная логика
    const result = await doSomething(apiKey);

    core.setOutput('result', result);
  } catch (error) {
    core.setFailed(`Action failed: ${error.message}`);
  }
}

run();
```

---

## Публикация в Marketplace

### Требования для публикации

#### Для GitHub Apps

1. **Публичный профиль** с верифицированным email
2. **Документация** — README с инструкциями
3. **Terms of Service** и **Privacy Policy**
4. **Logo** — минимум 256x256 пикселей
5. **Pricing plans** — как минимум один план
6. **Webhook URL** — рабочий endpoint

#### Для GitHub Actions

1. **Публичный репозиторий**
2. **action.yml** в корне репозитория
3. **README.md** с документацией
4. **Branding** — icon и color в action.yml
5. **Release** — опубликованный релиз с semver тегом

### Процесс публикации GitHub App

```
1. Developer settings → GitHub Apps → Your App
2. Нажать "Publish to Marketplace"
3. Заполнить информацию:
   - Categories
   - Short description
   - Full description (Markdown)
   - Pricing plans
   - Screenshots
4. Submit for review
5. Ожидать approval (обычно 1-3 дня)
```

### Процесс публикации Action

```
1. Создать Release в репозитории
2. Перейти в Marketplace listing
3. "Draft a release"
4. Выбрать категории
5. Publish release
```

### Best practices для Marketplace

1. **Четкое описание** — что делает приложение
2. **Документация** — подробные инструкции по установке
3. **Screenshots** — визуальные примеры работы
4. **Responsive support** — быстрые ответы на issues
5. **Regular updates** — поддержка актуальной версии
6. **Semantic versioning** — понятные версии

---

## Безопасность

### Оценка приложений перед установкой

#### Checklist безопасности

```
[ ] Проверить издателя (Verified creator badge)
[ ] Изучить запрашиваемые разрешения
[ ] Прочитать Privacy Policy
[ ] Проверить репутацию (звезды, отзывы)
[ ] Посмотреть историю обновлений
[ ] Проверить исходный код (если open-source)
[ ] Оценить necessity — нужны ли все запрашиваемые права
```

#### Red flags

- Запрос избыточных разрешений
- Нет Privacy Policy
- Неизвестный разработчик без истории
- Давно не обновлялось
- Много негативных отзывов
- Закрытый исходный код для критичных операций

### Принцип минимальных привилегий

```yaml
# Плохо — слишком широкий доступ
permissions:
  contents: write  # Зачем write, если только читаем?
  admin: write     # Опасно!

# Хорошо — только необходимое
permissions:
  contents: read
  pull-requests: write
```

### Безопасность секретов

```yaml
# Никогда не хардкодить секреты
- uses: some-action@v1
  with:
    api-key: sk-12345  # ПЛОХО!

# Использовать GitHub Secrets
- uses: some-action@v1
  with:
    api-key: ${{ secrets.API_KEY }}  # Хорошо
```

### Аудит установленных приложений

Регулярно проверяйте:

1. **Settings → Applications** — список установленных apps
2. **Security log** — история действий приложений
3. **Unused apps** — удалить неиспользуемые
4. **Permission changes** — уведомления об изменении прав

### Revoke доступа

При подозрении на компрометацию:

1. Settings → Applications → Authorized OAuth Apps
2. Revoke access для подозрительного приложения
3. Сменить затронутые токены/секреты
4. Проверить audit log на подозрительные действия

---

## Заключение

### Ключевые выводы

1. **GitHub Marketplace** — централизованная платформа для расширения возможностей GitHub
2. **GitHub Apps** предпочтительнее OAuth Apps благодаря гранулярным разрешениям
3. **GitHub Actions** в Marketplace ускоряют создание CI/CD pipelines
4. При установке приложений всегда проверяйте **запрашиваемые разрешения**
5. Создание собственных Apps и Actions позволяет **автоматизировать** уникальные процессы

### Рекомендации

- Используйте **verified publishers** когда возможно
- Регулярно **аудируйте** установленные приложения
- Применяйте принцип **минимальных привилегий**
- Держите Actions на **фиксированных версиях**
- Читайте **Privacy Policy** перед установкой

### Полезные ссылки

- [GitHub Marketplace](https://github.com/marketplace)
- [GitHub Apps Documentation](https://docs.github.com/en/apps)
- [Creating Actions](https://docs.github.com/en/actions/creating-actions)
- [Marketplace Developer Agreement](https://docs.github.com/en/site-policy/github-terms/github-marketplace-developer-agreement)
- [Awesome Actions](https://github.com/sdras/awesome-actions)

---
[prev: 13-github-security](./13-github-security.md) | [next: 01-reflog](15-advanced-git/01-reflog.md)