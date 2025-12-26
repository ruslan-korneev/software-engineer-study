# Collaborators (Коллабораторы)

## Введение

**Collaborators** — это пользователи GitHub, которым предоставлен доступ к репозиторию. Система управления доступом позволяет:
- Контролировать, кто может читать/писать код
- Защищать важные ветки
- Автоматизировать code review
- Организовывать командную работу

---

## Добавление коллабораторов

### Для личных репозиториев

1. Перейдите в Settings репозитория
2. Выберите "Collaborators" в левом меню
3. Нажмите "Add people"
4. Введите username или email
5. Выберите уровень доступа
6. Отправьте приглашение

### Через GitHub CLI

```bash
# Добавить коллаборатора
gh api repos/{owner}/{repo}/collaborators/{username} \
  --method PUT \
  --field permission="push"

# Проверить коллабораторов
gh api repos/{owner}/{repo}/collaborators

# Удалить коллаборатора
gh api repos/{owner}/{repo}/collaborators/{username} \
  --method DELETE
```

### Программно через API

```bash
# Добавление с помощью curl
curl -X PUT \
  -H "Authorization: token YOUR_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/OWNER/REPO/collaborators/USERNAME \
  -d '{"permission":"push"}'
```

---

## Уровни доступа (Permission Levels)

### Для личных репозиториев

| Уровень | Описание |
|---------|----------|
| **Read** | Только просмотр кода и issues |
| **Triage** | Read + управление issues и PR без доступа к коду |
| **Write** | Triage + push в репозиторий |
| **Maintain** | Write + управление настройками (без опасных) |
| **Admin** | Полный доступ, включая удаление репозитория |

### Детальные права по уровням

#### Read (Чтение)

- Клонирование репозитория
- Просмотр кода, issues, PR, wiki
- Создание issues
- Комментирование

#### Triage

- Все права Read
- Управление issues и PR (labels, assignees, milestones)
- Закрытие/переоткрытие issues
- Запрос reviewers

#### Write (Запись)

- Все права Triage
- Push в репозиторий
- Создание и удаление веток
- Merge pull requests
- Редактирование wiki

#### Maintain

- Все права Write
- Управление настройками репозитория (кроме опасных)
- Управление webhooks
- Управление deploy keys

#### Admin (Администратор)

- Все права Maintain
- Добавление/удаление коллабораторов
- Изменение visibility репозитория
- Удаление репозитория
- Управление branch protection
- Передача репозитория

---

## Teams в организациях

В организациях используются **Teams** для группового управления доступом.

### Создание Team

```bash
# Через GitHub CLI
gh api orgs/{org}/teams \
  --method POST \
  --field name="backend-developers" \
  --field description="Backend development team" \
  --field privacy="closed"
```

### Типы Team

| Тип | Описание |
|-----|----------|
| **Visible** | Видна всем членам организации |
| **Secret** | Видна только членам команды и owners |

### Добавление членов в Team

```bash
# Добавить пользователя в team
gh api orgs/{org}/teams/{team_slug}/memberships/{username} \
  --method PUT \
  --field role="member"

# Роли в team: member, maintainer
```

### Предоставление доступа Team к репозиторию

```bash
# Добавить team к репозиторию
gh api orgs/{org}/teams/{team_slug}/repos/{org}/{repo} \
  --method PUT \
  --field permission="push"
```

### Иерархия Teams

Teams могут быть вложенными:

```
Engineering (parent)
├── Backend Team
├── Frontend Team
└── DevOps Team
```

Дочерние команды наследуют доступ родительской.

---

## Protected Branches

Protected branches защищают важные ветки от нежелательных изменений.

### Настройка через UI

1. Settings → Branches
2. Add branch protection rule
3. Укажите паттерн (например, `main` или `release/*`)
4. Выберите правила защиты

### Доступные правила защиты

#### Require pull request reviews

```yaml
# Настройки
required_approving_review_count: 2  # Минимум 2 approval
dismiss_stale_reviews: true         # Сбрасывать при новых коммитах
require_code_owner_reviews: true    # Требовать review от CODEOWNERS
require_last_push_approval: true    # Последний push должен быть одобрен
```

#### Require status checks

```yaml
# Требовать прохождение CI
required_status_checks:
  strict: true  # Ветка должна быть up to date с base
  contexts:
    - "ci/test"
    - "ci/lint"
    - "ci/build"
```

#### Require conversation resolution

Все комментарии в PR должны быть resolved перед merge.

#### Require signed commits

Все коммиты должны быть подписаны GPG ключом.

#### Require linear history

Запрещает merge commits, требует squash или rebase.

#### Include administrators

Правила применяются и к администраторам.

#### Restrict pushes

Ограничить push только определённым пользователям/командам.

#### Allow force pushes

Разрешить force push (опасно, обычно отключено).

#### Allow deletions

Разрешить удаление ветки.

### Настройка через CLI

```bash
# Установить защиту ветки
gh api repos/{owner}/{repo}/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["ci/test"]}' \
  --field enforce_admins=true \
  --field required_pull_request_reviews='{"required_approving_review_count":2}' \
  --field restrictions=null
```

### Пример конфигурации для production

```bash
# main branch protection
gh api repos/{owner}/{repo}/branches/main/protection \
  --method PUT \
  -f required_status_checks='{"strict":true,"contexts":["test","lint","build"]}' \
  -f enforce_admins=true \
  -f required_pull_request_reviews='{"dismiss_stale_reviews":true,"require_code_owner_reviews":true,"required_approving_review_count":2}' \
  -f required_linear_history=true \
  -f allow_force_pushes=false \
  -f allow_deletions=false
```

---

## CODEOWNERS

**CODEOWNERS** — файл, определяющий автоматических reviewers для разных частей кода.

### Расположение файла

```
# Один из вариантов:
.github/CODEOWNERS
CODEOWNERS
docs/CODEOWNERS
```

### Синтаксис

```gitignore
# Это комментарий

# Владелец по умолчанию для всего репозитория
*       @default-owner

# Владельцы конкретных файлов
README.md    @docs-team

# Владельцы директорий
/src/        @backend-team
/frontend/   @frontend-team @lead-developer

# Паттерны с wildcards
*.js         @js-team
*.py         @python-team

# Вложенные директории
/apps/api/   @api-team
/apps/web/   @web-team

# Владельцы по расширению в конкретной директории
/docs/*.md   @tech-writers

# GitHub teams (с @)
/infrastructure/  @org/devops-team

# Несколько владельцев
/critical/   @lead @senior @cto
```

### Правила работы CODEOWNERS

1. **Последнее совпадение выигрывает** — более специфичные правила внизу файла
2. **Автоматический запрос review** — при создании PR владельцы добавляются как reviewers
3. **Требование approval** — при включенной опции "Require review from Code Owners"
4. **Обязательные reviewers** — нельзя merge без их approval (если настроено)

### Примеры паттернов

```gitignore
# Все файлы
*                          @global-owner

# Конкретный файл в корне
/README.md                 @readme-owner

# Все MD файлы везде
*.md                       @docs-team

# Все файлы в директории
/src/                      @src-owner

# Рекурсивно все JS файлы
**/*.js                    @js-team

# Файлы с определённым именем везде
**/package.json            @deps-team

# Исключение (пустой владелец)
/generated/                # никто
```

### Пример реального CODEOWNERS

```gitignore
# Default owners
*                           @myorg/core-team

# Documentation
*.md                        @myorg/docs-team
/docs/                      @myorg/docs-team

# Frontend
/frontend/                  @myorg/frontend-team
*.tsx                       @myorg/frontend-team
*.css                       @myorg/frontend-team

# Backend
/backend/                   @myorg/backend-team
/api/                       @myorg/backend-team
*.py                        @myorg/backend-team

# Infrastructure
/terraform/                 @myorg/devops-team
/kubernetes/                @myorg/devops-team
/.github/workflows/         @myorg/devops-team
Dockerfile                  @myorg/devops-team

# Security-sensitive
/auth/                      @myorg/security-team @myorg/backend-team
/.env.example               @myorg/security-team

# Dependencies (require senior review)
package.json                @lead-developer
package-lock.json           @lead-developer
requirements.txt            @lead-developer
go.mod                      @lead-developer

# Critical configuration
/.github/CODEOWNERS         @admin
/config/production.yaml     @admin @myorg/devops-team
```

---

## Управление доступом: Best Practices

### Принцип минимальных привилегий

```
✓ Давайте минимально необходимый уровень доступа
✓ Используйте Read для внешних контракторов
✓ Write только для активных разработчиков
✓ Admin только для tech leads / maintainers
```

### Организация команд

```
# Хорошая структура
├── Owners (2-3 человека)
├── Maintainers
│   ├── Backend Maintainers
│   └── Frontend Maintainers
├── Developers
│   ├── Backend Developers
│   └── Frontend Developers
└── External Contributors (Read only)
```

### Защита веток

```
main/master:
  - Require PR
  - Require 2+ approvals
  - Require CI pass
  - Require CODEOWNERS review
  - No direct push
  - No force push

develop:
  - Require PR
  - Require 1+ approval
  - Require CI pass

feature/*:
  - No restrictions (developers can manage their branches)

release/*:
  - Same as main
```

### Аудит доступа

```bash
# Регулярно проверяйте коллабораторов
gh api repos/{owner}/{repo}/collaborators

# Проверяйте inactive пользователей
# Удаляйте доступ у уволившихся сотрудников
# Пересматривайте уровни доступа

# Просмотр audit log (для организаций)
# Settings → Audit log
```

---

## Repository Rulesets (новая функция)

GitHub Rulesets — более гибкая система правил, заменяющая branch protection.

### Преимущества Rulesets

- Применение к нескольким веткам/тегам
- Bypass lists для определённых пользователей
- Организационный уровень правил
- История изменений правил

### Создание Ruleset

```bash
# Через API
gh api repos/{owner}/{repo}/rulesets \
  --method POST \
  --field name="production-rules" \
  --field target="branch" \
  --field enforcement="active" \
  --field conditions='{"ref_name":{"include":["refs/heads/main","refs/heads/release/*"]}}' \
  --field rules='[{"type":"pull_request","parameters":{"required_approving_review_count":2}}]'
```

---

## Автоматизация управления доступом

### GitHub Actions для приветствия

```yaml
# .github/workflows/welcome.yml
name: Welcome New Contributor

on:
  pull_request_target:
    types: [opened]

jobs:
  welcome:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/first-interaction@v1
        with:
          repo-token: ${{ secrets.GITHUB_TOKEN }}
          pr-message: |
            Thanks for your first PR! 🎉
            A maintainer will review it soon.
```

### Auto-assign reviewers

```yaml
# .github/auto-assign.yml
addReviewers: true
addAssignees: author
reviewers:
  - reviewer1
  - reviewer2
numberOfReviewers: 2
```

---

## Заключение

Правильное управление доступом критически важно для:

1. **Безопасности** — защита от несанкционированных изменений
2. **Качества кода** — обязательный code review
3. **Организации работы** — чёткие зоны ответственности
4. **Масштабирования** — Teams для больших организаций

Используйте:
- **Collaborators** для простых проектов
- **Teams** для организаций
- **Protected Branches** для защиты важных веток
- **CODEOWNERS** для автоматизации review
- **Rulesets** для сложных сценариев
