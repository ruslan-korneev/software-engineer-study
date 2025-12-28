# GitHub Security

[prev: 03-webhooks](12-github-developer-tools/03-webhooks.md) | [next: 14-github-marketplace](./14-github-marketplace.md)
---

## Введение

Безопасность на GitHub — это комплексный набор инструментов и практик, направленных на защиту кода, секретов, зависимостей и всей инфраструктуры разработки. GitHub предоставляет множество встроенных функций безопасности, которые помогают разработчикам и организациям защищать свои проекты от уязвимостей, утечек данных и несанкционированного доступа.

### Почему безопасность важна

- **Защита интеллектуальной собственности** — исходный код является ценным активом
- **Предотвращение утечек секретов** — API ключи, пароли, токены не должны попадать в репозиторий
- **Управление уязвимостями** — зависимости могут содержать известные уязвимости
- **Соответствие требованиям** — многие отрасли требуют соблюдения стандартов безопасности
- **Защита цепочки поставок** — компрометация зависимостей может повлиять на ваш продукт

### Основные компоненты безопасности GitHub

```
┌─────────────────────────────────────────────────────────────┐
│                    GitHub Security                          │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │ Dependabot  │  │   Secret    │  │   Code Scanning     │ │
│  │   Alerts    │  │  Scanning   │  │      (CodeQL)       │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  Security   │  │   Branch    │  │   Signed Commits    │ │
│  │  Advisories │  │ Protection  │  │      (GPG/SSH)      │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
│  ┌─────────────────────────────────────────────────────┐   │
│  │           2FA / SSO / SAML Authentication           │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## Dependabot

Dependabot — это инструмент GitHub для автоматического управления зависимостями. Он сканирует проекты на наличие устаревших или уязвимых зависимостей и автоматически создает pull request'ы для их обновления.

### Dependabot Alerts

Оповещения Dependabot уведомляют о известных уязвимостях в зависимостях проекта.

**Как работает:**
1. GitHub сканирует манифесты зависимостей (package.json, requirements.txt, Gemfile и т.д.)
2. Сравнивает версии с базой данных GitHub Advisory Database
3. Создает alerts при обнаружении уязвимостей

**Включение alerts:**
- Settings → Security → Code security and analysis
- Включить "Dependabot alerts"

### Dependabot Security Updates

Автоматическое создание PR для исправления уязвимостей.

**Включение:**
- Settings → Security → Enable Dependabot security updates

### Dependabot Version Updates

Автоматическое обновление зависимостей до последних версий (не только для безопасности).

**Конфигурационный файл `.github/dependabot.yml`:**

```yaml
version: 2
updates:
  # Обновление npm зависимостей
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "Europe/Moscow"
    open-pull-requests-limit: 10
    reviewers:
      - "security-team"
    labels:
      - "dependencies"
      - "automated"
    commit-message:
      prefix: "deps"
      include: "scope"
    # Игнорировать определенные зависимости
    ignore:
      - dependency-name: "lodash"
        versions: ["4.x"]
    # Группировка обновлений
    groups:
      development-dependencies:
        dependency-type: "development"
      production-dependencies:
        dependency-type: "production"

  # Обновление Python зависимостей
  - package-ecosystem: "pip"
    directory: "/backend"
    schedule:
      interval: "daily"
    target-branch: "develop"

  # Обновление Docker образов
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    registries:
      - docker-hub

  # Обновление GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "monthly"

# Настройка приватных реестров
registries:
  docker-hub:
    type: docker-registry
    url: https://registry.hub.docker.com
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_PASSWORD }}
  npm-private:
    type: npm-registry
    url: https://npm.pkg.github.com
    token: ${{ secrets.NPM_TOKEN }}
```

### Поддерживаемые экосистемы

| Экосистема | Файл манифеста |
|------------|----------------|
| npm | package.json, package-lock.json |
| pip | requirements.txt, Pipfile, setup.py |
| maven | pom.xml |
| gradle | build.gradle |
| composer | composer.json |
| cargo | Cargo.toml |
| go | go.mod |
| docker | Dockerfile |
| github-actions | .github/workflows/*.yml |

---

## Secret Scanning

Secret Scanning — это функция GitHub для обнаружения секретов (токенов, паролей, ключей API), случайно закоммиченных в репозиторий.

### Как работает Secret Scanning

1. **Сканирование репозитория** — GitHub проверяет весь код на наличие паттернов секретов
2. **Партнерская программа** — GitHub сотрудничает с провайдерами (AWS, Azure, Stripe и др.)
3. **Автоматическое уведомление** — при обнаружении секрета уведомляются владелец и провайдер
4. **Отзыв токенов** — некоторые провайдеры автоматически отзывают скомпрометированные токены

### Поддерживаемые секреты

```
┌────────────────────────────────────────────────────────────┐
│              Типы обнаруживаемых секретов                  │
├────────────────────────────────────────────────────────────┤
│ • API ключи (AWS, Azure, GCP, Stripe, Twilio)             │
│ • OAuth токены (GitHub, GitLab, Slack)                    │
│ • Приватные ключи (SSH, RSA, PGP)                         │
│ • Токены доступа (npm, PyPI, Docker Hub)                  │
│ • Строки подключения к БД                                  │
│ • JWT секреты                                              │
│ • Webhook URL с токенами                                   │
│ • Ключи шифрования                                         │
└────────────────────────────────────────────────────────────┘
```

### Push Protection

Push Protection блокирует push, если обнаружены секреты — до того, как код попадет в репозиторий.

**Включение:**
- Settings → Security → Code security and analysis
- Enable "Push protection"

**Что происходит при обнаружении секрета:**

```bash
$ git push origin main
remote: error: GH013: Repository rule violations found for refs/heads/main.
remote:
remote: - GITHUB PUSH PROTECTION
remote:   —————————————————————————————————————————————————
remote:   Resolve the following violations before pushing:
remote:   —————————————————————————————————————————————————
remote:
remote:   — Push Blocked —
remote:   The following secret was detected in commit abc123:
remote:
remote:   —— AWS Access Key ID ——
remote:   locations:
remote:     - commit: abc123
remote:       path: config/settings.py:15
remote:
remote:   To push, you need to remove the secret from your commits.
```

### Custom Patterns

Для Enterprise можно настроить собственные паттерны секретов:

```yaml
# Пример кастомного паттерна для внутренних API ключей
pattern: "MYCOMPANY_API_[A-Za-z0-9]{32}"
name: "MyCompany API Key"
```

### Работа с обнаруженными секретами

```bash
# 1. Немедленно отозвать/ротировать секрет у провайдера

# 2. Удалить секрет из истории Git
# Использование git filter-repo (рекомендуется)
pip install git-filter-repo
git filter-repo --invert-paths --path config/secrets.py

# Или использование BFG Repo-Cleaner
java -jar bfg.jar --delete-files secrets.py
java -jar bfg.jar --replace-text passwords.txt

# 3. Force push
git push origin --force --all

# 4. Закрыть alert как resolved в GitHub
```

---

## Code Scanning

Code Scanning — статический анализ кода для обнаружения уязвимостей, ошибок и проблем безопасности.

### CodeQL

CodeQL — это движок семантического анализа кода, разработанный GitHub. Он позволяет находить уязвимости, анализируя код как данные.

**Поддерживаемые языки:**
- C/C++
- C#
- Go
- Java/Kotlin
- JavaScript/TypeScript
- Python
- Ruby
- Swift

### Настройка Code Scanning

**Базовая конфигурация `.github/workflows/codeql.yml`:**

```yaml
name: "CodeQL Analysis"

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    # Еженедельное сканирование
    - cron: '0 2 * * 1'
  workflow_dispatch:

jobs:
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write

    strategy:
      fail-fast: false
      matrix:
        language: ['javascript', 'python']

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
          # Использование расширенного набора запросов
          queries: +security-extended,security-and-quality
          # Кастомные конфигурации
          config-file: .github/codeql/codeql-config.yml

      # Для компилируемых языков нужен шаг сборки
      - name: Build (for compiled languages)
        if: matrix.language == 'java' || matrix.language == 'cpp'
        run: |
          make build

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: "/language:${{ matrix.language }}"
```

**Расширенная конфигурация `.github/codeql/codeql-config.yml`:**

```yaml
name: "Custom CodeQL Config"

# Отключение запросов по умолчанию
disable-default-queries: false

# Дополнительные запросы
queries:
  - uses: security-extended
  - uses: security-and-quality
  # Кастомные запросы
  - uses: ./my-custom-queries

# Пути для анализа
paths:
  - src
  - lib

# Исключение путей
paths-ignore:
  - tests
  - node_modules
  - "**/*.test.js"

# Настройки для конкретных языков
query-filters:
  - exclude:
      id: js/useless-expression
```

### SARIF (Static Analysis Results Interchange Format)

SARIF — это стандартный формат для результатов статического анализа. GitHub принимает SARIF файлы от любых инструментов.

**Загрузка SARIF результатов:**

```yaml
- name: Upload SARIF results
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: results.sarif
    category: "custom-analysis"
```

**Интеграция сторонних инструментов:**

```yaml
# Пример с Semgrep
- name: Semgrep Scan
  uses: returntocorp/semgrep-action@v1
  with:
    config: p/security-audit

# Пример с Trivy
- name: Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: 'fs'
    format: 'sarif'
    output: 'trivy-results.sarif'

- name: Upload Trivy results
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'
```

---

## Security Advisories

Security Advisories — это механизм для приватного обсуждения и исправления уязвимостей в проекте.

### Создание Security Advisory

1. **Security → Advisories → New draft advisory**
2. Заполнить информацию об уязвимости:
   - Название и описание
   - Severity (Critical, High, Medium, Low)
   - Affected versions
   - Patched versions
   - CWE (Common Weakness Enumeration)
   - CVE ID (можно запросить у GitHub)

### Процесс работы с Advisory

```
┌──────────────────────────────────────────────────────────────┐
│                 Security Advisory Workflow                    │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Создание Draft Advisory                                   │
│         ↓                                                     │
│  2. Создание приватного форка (Temporary Private Fork)        │
│         ↓                                                     │
│  3. Разработка исправления в приватном форке                  │
│         ↓                                                     │
│  4. Тестирование исправления                                  │
│         ↓                                                     │
│  5. Запрос CVE ID (опционально)                               │
│         ↓                                                     │
│  6. Публикация Advisory                                       │
│         ↓                                                     │
│  7. Merge исправления в основную ветку                        │
│         ↓                                                     │
│  8. Уведомление пользователей                                 │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Пример Security Advisory

```markdown
## Summary
SQL Injection vulnerability in user authentication module.

## Details
The `authenticate()` function in `auth/login.py` is vulnerable to SQL injection
through the `username` parameter. An attacker can bypass authentication or
extract data from the database.

## Severity
**High** (CVSS 8.1)

## Affected Versions
- >= 1.0.0
- < 2.3.5

## Patched Versions
- 2.3.5

## Workarounds
Sanitize user input before passing to `authenticate()` function.

## References
- CWE-89: SQL Injection
```

---

## Security Policy

Файл `SECURITY.md` описывает политику безопасности проекта и процесс сообщения об уязвимостях.

### Пример SECURITY.md

```markdown
# Security Policy

## Supported Versions

| Version | Supported          |
| ------- | ------------------ |
| 3.x.x   | :white_check_mark: |
| 2.x.x   | :white_check_mark: |
| 1.x.x   | :x:                |
| < 1.0   | :x:                |

## Reporting a Vulnerability

Мы серьезно относимся к безопасности нашего проекта. Если вы обнаружили
уязвимость, пожалуйста, сообщите нам об этом ответственным образом.

### Как сообщить

1. **НЕ** создавайте публичный Issue
2. Отправьте email на security@example.com
3. Или используйте GitHub Security Advisories (рекомендуется)

### Что включить в отчет

- Описание уязвимости
- Шаги для воспроизведения
- Версии, которые затронуты
- Возможное влияние
- Предлагаемое исправление (если есть)

### Что ожидать

- **24 часа** — подтверждение получения отчета
- **72 часа** — первоначальная оценка
- **7 дней** — план действий
- **90 дней** — исправление (в зависимости от сложности)

### Bug Bounty

Мы предлагаем вознаграждение за ответственное раскрытие уязвимостей:
- Critical: $1000-5000
- High: $500-1000
- Medium: $100-500
- Low: $50-100

## Security Measures

### Что мы делаем для безопасности

- Регулярный аудит зависимостей
- Автоматическое сканирование кода (CodeQL)
- Обязательные code review
- Подписанные релизы
- 2FA для всех мейнтейнеров

### Best Practices для пользователей

- Всегда используйте последнюю версию
- Проверяйте подписи релизов
- Подпишитесь на security advisories
- Сообщайте о подозрительной активности
```

---

## Branch Protection

Branch Protection Rules — правила защиты веток от нежелательных изменений.

### Настройка Branch Protection

Settings → Branches → Add branch protection rule

### Основные правила

```yaml
# Концептуальная структура правил защиты ветки
branch_protection:
  pattern: "main"

  rules:
    # Требовать pull request перед merge
    require_pull_request:
      enabled: true
      required_approving_review_count: 2
      dismiss_stale_reviews: true
      require_code_owner_reviews: true
      require_last_push_approval: true

    # Требовать прохождение status checks
    require_status_checks:
      enabled: true
      strict: true  # Требовать актуальность ветки
      contexts:
        - "ci/tests"
        - "ci/build"
        - "CodeQL"
        - "security/secret-scanning"

    # Требовать разрешение conversations
    require_conversation_resolution: true

    # Требовать подписанные коммиты
    require_signed_commits: true

    # Требовать линейную историю
    require_linear_history: true

    # Запретить force push
    allow_force_pushes: false

    # Запретить удаление ветки
    allow_deletions: false

    # Обход правил для администраторов
    enforce_admins: true

    # Ограничить кто может push
    restrict_pushes:
      users: ["release-bot"]
      teams: ["release-team"]
```

### CODEOWNERS

Файл `CODEOWNERS` определяет, кто должен review'ить изменения в определенных файлах.

```gitignore
# .github/CODEOWNERS

# Глобальные владельцы (для всех файлов)
* @default-team

# Владельцы для конкретных директорий
/src/security/ @security-team
/src/api/ @api-team @backend-lead
/docs/ @docs-team

# Владельцы для типов файлов
*.js @frontend-team
*.py @backend-team
*.sql @dba-team

# Владельцы для конфигураций
/config/ @devops-team
/.github/ @platform-team
Dockerfile @devops-team @security-team

# Владельцы для критических файлов
/src/auth/ @security-team @tech-lead
package.json @frontend-team @security-team
requirements.txt @backend-team @security-team
```

### Rulesets (Новая функция)

Rulesets — более гибкая замена branch protection rules.

```yaml
# Концептуальная структура Ruleset
ruleset:
  name: "Production Branch Protection"
  target: branch
  enforcement: active

  conditions:
    ref_name:
      include:
        - "refs/heads/main"
        - "refs/heads/release/*"
      exclude:
        - "refs/heads/release/beta-*"

  rules:
    - type: pull_request
      parameters:
        required_approving_review_count: 2
        dismiss_stale_reviews_on_push: true
        require_code_owner_review: true

    - type: required_status_checks
      parameters:
        strict_required_status_checks_policy: true
        required_status_checks:
          - context: "ci/tests"
            integration_id: 12345

    - type: required_signatures

    - type: non_fast_forward
```

---

## Signed Commits

Подписанные коммиты подтверждают авторство и целостность изменений.

### Типы подписей

| Тип | Описание | Использование |
|-----|----------|---------------|
| GPG | Классический метод | Широко поддерживается |
| SSH | Современный метод | Проще в настройке |
| S/MIME | Корпоративный | X.509 сертификаты |

### Настройка GPG подписи

```bash
# 1. Генерация GPG ключа
gpg --full-generate-key
# Выбрать: RSA and RSA, 4096 bits, срок действия

# 2. Получение ID ключа
gpg --list-secret-keys --keyid-format=long
# sec   rsa4096/3AA5C34371567BD2 2024-01-01 [SC]

# 3. Экспорт публичного ключа
gpg --armor --export 3AA5C34371567BD2

# 4. Добавление ключа в GitHub
# Settings → SSH and GPG keys → New GPG key

# 5. Настройка Git
git config --global user.signingkey 3AA5C34371567BD2
git config --global commit.gpgsign true
git config --global tag.gpgsign true

# 6. Подписанный коммит
git commit -S -m "Подписанный коммит"

# 7. Проверка подписи
git log --show-signature
```

### Настройка SSH подписи

```bash
# 1. Использование существующего SSH ключа или создание нового
ssh-keygen -t ed25519 -C "your_email@example.com"

# 2. Добавление ключа в GitHub как signing key
# Settings → SSH and GPG keys → New SSH key → Key type: Signing Key

# 3. Настройка Git
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true

# 4. Настройка allowed_signers для верификации
echo "your_email@example.com $(cat ~/.ssh/id_ed25519.pub)" >> ~/.ssh/allowed_signers
git config --global gpg.ssh.allowedSignersFile ~/.ssh/allowed_signers
```

### Vigilant Mode

Vigilant Mode помечает все неподписанные коммиты как "Unverified".

**Включение:**
- Settings → SSH and GPG keys → Enable Vigilant mode

### Статусы верификации

| Статус | Значение |
|--------|----------|
| Verified | Подпись действительна и соответствует email |
| Partially verified | Подпись действительна, но email не подтвержден |
| Unverified | Нет подписи или она недействительна |

---

## 2FA и SSO

### Двухфакторная аутентификация (2FA)

2FA добавляет дополнительный уровень защиты аккаунта.

**Поддерживаемые методы:**
- TOTP приложения (Google Authenticator, Authy, 1Password)
- SMS (не рекомендуется)
- Security keys (WebAuthn/FIDO2)
- GitHub Mobile

**Настройка 2FA:**

```
Settings → Password and authentication → Enable two-factor authentication

Рекомендуемый порядок:
1. Настроить TOTP приложение (основной метод)
2. Добавить Security Key (аппаратный ключ)
3. Сохранить recovery codes в безопасном месте
4. Настроить GitHub Mobile как резервный метод
```

**Recovery Codes:**
```
# Храните в безопасном месте (password manager, сейф)
# Каждый код можно использовать только один раз

a1b2c-d3e4f
g5h6i-j7k8l
m9n0o-p1q2r
...
```

### Принудительное 2FA для организации

```
Organization Settings → Authentication security → Require two-factor authentication
```

### SAML Single Sign-On (SSO)

SAML SSO позволяет использовать корпоративный identity provider для аутентификации.

**Поддерживаемые IdP:**
- Azure Active Directory
- Okta
- OneLogin
- PingOne
- Другие SAML 2.0 совместимые

**Настройка SAML SSO:**

```yaml
# Концептуальная настройка (выполняется через UI)
saml_sso:
  enabled: true
  idp:
    sso_url: "https://idp.example.com/sso/saml"
    issuer: "https://idp.example.com"
    certificate: |
      -----BEGIN CERTIFICATE-----
      MIIDpDCCAoygAwIBAgIGAX...
      -----END CERTIFICATE-----

  # Маппинг атрибутов
  attribute_mapping:
    username: "urn:oid:0.9.2342.19200300.100.1.1"
    email: "urn:oid:0.9.2342.19200300.100.1.3"
    full_name: "urn:oid:2.5.4.3"

  # Требовать SSO для всех членов организации
  require_sso: true
```

### SCIM Provisioning

SCIM автоматически синхронизирует пользователей и группы с IdP.

```
Organization Settings → Security → SCIM provisioning
```

---

## Best Practices

### Общие рекомендации

```
┌────────────────────────────────────────────────────────────────┐
│                  Security Best Practices                        │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✓ Включите 2FA для всех участников                            │
│  ✓ Используйте SAML SSO для организаций                        │
│  ✓ Требуйте подписанные коммиты                                │
│  ✓ Настройте branch protection для main/master                 │
│  ✓ Включите Dependabot alerts и security updates               │
│  ✓ Активируйте Secret scanning с push protection               │
│  ✓ Настройте Code scanning (CodeQL)                            │
│  ✓ Создайте SECURITY.md с политикой безопасности               │
│  ✓ Используйте CODEOWNERS для критических файлов               │
│  ✓ Регулярно проводите аудит доступов                          │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### Чеклист безопасности репозитория

```markdown
## Repository Security Checklist

### Access Control
- [ ] Минимальные необходимые права для collaborators
- [ ] Регулярный review членов организации и команд
- [ ] Удаление неактивных пользователей
- [ ] Аудит deploy keys и OAuth apps

### Branch Protection
- [ ] Защита main/master ветки
- [ ] Требование PR reviews (минимум 2)
- [ ] Требование прохождения CI
- [ ] Запрет force push
- [ ] CODEOWNERS для критических путей

### Secrets Management
- [ ] Secret scanning включен
- [ ] Push protection активирован
- [ ] Нет секретов в истории Git
- [ ] Используются GitHub Secrets для CI/CD
- [ ] Регулярная ротация секретов

### Dependencies
- [ ] Dependabot alerts включены
- [ ] Dependabot security updates включены
- [ ] Настроен dependabot.yml
- [ ] Регулярный review и обновление зависимостей

### Code Analysis
- [ ] CodeQL настроен для всех языков
- [ ] Интегрированы дополнительные SAST инструменты
- [ ] Code review процесс обязателен
- [ ] Линтеры настроены и запускаются в CI

### Documentation
- [ ] SECURITY.md создан и актуален
- [ ] Процесс disclosure описан
- [ ] Контакты для security issues указаны
```

### Безопасность GitHub Actions

```yaml
# Пример безопасного workflow

name: Secure CI

on:
  pull_request:
    branches: [main]

# Минимальные права
permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          # Не включать полную историю
          fetch-depth: 1
          # Не использовать persist-credentials
          persist-credentials: false

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          # Использовать кеш безопасно
          cache: 'npm'

      # Никогда не выводить секреты в логи
      - name: Build
        run: npm run build
        env:
          # Маскировать значения
          API_KEY: ${{ secrets.API_KEY }}

      # Проверка зависимостей
      - name: Audit dependencies
        run: npm audit --audit-level=high
```

### Аудит и мониторинг

```bash
# Просмотр audit log организации
# Organization Settings → Logs → Audit log

# Экспорт audit log через API
gh api \
  -H "Accept: application/vnd.github+json" \
  /orgs/{org}/audit-log \
  --paginate > audit_log.json

# Фильтрация по действиям
gh api "/orgs/{org}/audit-log?phrase=action:repo.destroy"
```

---

## Заключение

Безопасность на GitHub — это многоуровневая система защиты, требующая комплексного подхода. Основные принципы:

### Ключевые выводы

1. **Defense in Depth** — используйте несколько уровней защиты
2. **Principle of Least Privilege** — минимальные необходимые права
3. **Shift Left** — интегрируйте безопасность на ранних этапах разработки
4. **Automation** — автоматизируйте проверки безопасности
5. **Continuous Monitoring** — постоянно отслеживайте угрозы

### Порядок внедрения

```
Этап 1: Базовая защита
├── Включить 2FA для всех
├── Настроить branch protection
└── Создать SECURITY.md

Этап 2: Автоматизация
├── Включить Dependabot
├── Настроить Secret scanning
└── Внедрить Code scanning

Этап 3: Расширенная защита
├── Настроить SAML SSO
├── Внедрить подписанные коммиты
└── Настроить SCIM provisioning

Этап 4: Мониторинг
├── Настроить audit log streaming
├── Внедрить SIEM интеграцию
└── Регулярные security reviews
```

### Полезные ресурсы

- [GitHub Security Documentation](https://docs.github.com/en/security)
- [GitHub Advisory Database](https://github.com/advisories)
- [CodeQL Documentation](https://codeql.github.com/docs/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [CWE (Common Weakness Enumeration)](https://cwe.mitre.org/)

Безопасность — это непрерывный процесс, требующий регулярного внимания и обновления практик в соответствии с новыми угрозами и возможностями платформы.

---
[prev: 03-webhooks](12-github-developer-tools/03-webhooks.md) | [next: 14-github-marketplace](./14-github-marketplace.md)