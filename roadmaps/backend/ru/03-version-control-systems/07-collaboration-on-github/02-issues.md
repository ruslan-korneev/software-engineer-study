# Issues (Задачи)

[prev: 01-forking-vs-cloning](./01-forking-vs-cloning.md) | [next: 03-pull-requests](./03-pull-requests.md)
---

## Введение

**Issues** — это встроенная система отслеживания задач в GitHub. Она используется для:
- Отчетов об ошибках (bug reports)
- Запросов новых функций (feature requests)
- Обсуждения идей и улучшений
- Планирования работы
- Документирования технического долга

---

## Создание Issue

### Через веб-интерфейс

1. Перейдите в репозиторий
2. Откройте вкладку "Issues"
3. Нажмите "New issue"
4. Заполните заголовок и описание
5. Добавьте метки, назначьте исполнителя
6. Нажмите "Submit new issue"

### Через GitHub CLI

```bash
# Создание простого issue
gh issue create --title "Bug: Login not working" --body "Description of the bug"

# С указанием labels и assignee
gh issue create \
  --title "Feature: Dark mode" \
  --body "Add dark mode support" \
  --label "enhancement" \
  --assignee "@me"

# Интерактивное создание
gh issue create
```

### Через URL

```
https://github.com/owner/repo/issues/new?title=Bug&body=Description&labels=bug
```

---

## Структура хорошего Issue

### Для Bug Report

```markdown
## Описание проблемы
Кратко опишите, что работает неправильно.

## Шаги для воспроизведения
1. Перейти на страницу '...'
2. Нажать на '...'
3. Прокрутить вниз до '...'
4. Увидеть ошибку

## Ожидаемое поведение
Опишите, что должно было произойти.

## Фактическое поведение
Опишите, что произошло на самом деле.

## Скриншоты
Если применимо, добавьте скриншоты.

## Окружение
- OS: [например, macOS 14.0]
- Browser: [например, Chrome 120]
- Version: [например, 1.2.3]

## Дополнительный контекст
Любая другая информация о проблеме.
```

### Для Feature Request

```markdown
## Описание функции
Кратко опишите желаемую функцию.

## Проблема, которую это решает
Опишите проблему, которую вы испытываете.

## Предлагаемое решение
Опишите, как вы хотели бы, чтобы это работало.

## Альтернативы
Опишите альтернативные решения, которые вы рассматривали.

## Дополнительный контекст
Любая другая информация или скриншоты.
```

---

## Labels (Метки)

Labels помогают категоризировать и фильтровать issues.

### Стандартные метки GitHub

| Label | Описание |
|-------|----------|
| `bug` | Что-то работает неправильно |
| `documentation` | Улучшения документации |
| `duplicate` | Issue уже существует |
| `enhancement` | Новая функция или улучшение |
| `good first issue` | Подходит для новичков |
| `help wanted` | Нужна помощь |
| `invalid` | Некорректный issue |
| `question` | Вопрос |
| `wontfix` | Не будет исправлено |

### Создание собственных меток

```bash
# Через GitHub CLI
gh label create "priority: high" --color "FF0000" --description "High priority issue"
gh label create "priority: medium" --color "FFA500"
gh label create "priority: low" --color "008000"

# Типы меток
gh label create "type: bug" --color "d73a4a"
gh label create "type: feature" --color "0075ca"
gh label create "type: docs" --color "0052cc"
gh label create "type: refactor" --color "fbca04"

# Статусы
gh label create "status: in progress" --color "1d76db"
gh label create "status: review needed" --color "5319e7"
gh label create "status: blocked" --color "b60205"
```

### Применение меток

```bash
# Добавить метку к issue
gh issue edit 42 --add-label "bug,priority: high"

# Удалить метку
gh issue edit 42 --remove-label "wontfix"

# Фильтрация по меткам
gh issue list --label "bug"
gh issue list --label "bug,priority: high"
```

---

## Milestones (Вехи)

Milestones группируют issues по релизам или спринтам.

### Создание Milestone

```bash
# Через GitHub CLI
gh api repos/{owner}/{repo}/milestones \
  --method POST \
  --field title="v1.0.0" \
  --field description="First stable release" \
  --field due_on="2024-03-01T00:00:00Z"
```

### Через веб-интерфейс

1. Issues → Milestones → New milestone
2. Укажите название, описание и дату
3. Создайте milestone

### Привязка Issue к Milestone

```bash
# Через CLI
gh issue edit 42 --milestone "v1.0.0"
```

### Отслеживание прогресса

Milestone показывает:
- Количество открытых/закрытых issues
- Процент завершения
- Дату дедлайна

---

## Assignees (Исполнители)

Assignees — это люди, ответственные за issue.

```bash
# Назначить себя
gh issue edit 42 --add-assignee "@me"

# Назначить другого пользователя
gh issue edit 42 --add-assignee "username"

# Назначить нескольких
gh issue edit 42 --add-assignee "user1,user2"

# Снять назначение
gh issue edit 42 --remove-assignee "username"
```

### Ограничения

- Максимум 10 assignees на issue
- Можно назначать только collaborators репозитория
- Для организаций — членов организации

---

## Issue Templates

Templates стандартизируют создание issues.

### Создание шаблона

Создайте файл `.github/ISSUE_TEMPLATE/bug_report.md`:

```markdown
---
name: Bug Report
about: Create a report to help us improve
title: '[BUG] '
labels: 'bug'
assignees: ''
---

## Describe the bug
A clear description of what the bug is.

## To Reproduce
Steps to reproduce the behavior:
1. Go to '...'
2. Click on '...'
3. See error

## Expected behavior
What you expected to happen.

## Screenshots
If applicable, add screenshots.

## Environment
- OS: [e.g., macOS]
- Browser: [e.g., Chrome]
- Version: [e.g., 22]
```

### Feature Request Template

`.github/ISSUE_TEMPLATE/feature_request.md`:

```markdown
---
name: Feature Request
about: Suggest an idea for this project
title: '[FEATURE] '
labels: 'enhancement'
assignees: ''
---

## Is your feature request related to a problem?
A clear description of the problem.

## Describe the solution you'd like
A clear description of what you want.

## Describe alternatives you've considered
Alternative solutions you've considered.

## Additional context
Any other context or screenshots.
```

### YAML-based Templates (рекомендуется)

`.github/ISSUE_TEMPLATE/bug_report.yml`:

```yaml
name: Bug Report
description: Report a bug to help us improve
title: "[Bug]: "
labels: ["bug", "triage"]
assignees:
  - maintainer1
body:
  - type: markdown
    attributes:
      value: |
        Thanks for taking the time to fill out this bug report!

  - type: input
    id: version
    attributes:
      label: Version
      description: What version are you running?
      placeholder: "1.0.0"
    validations:
      required: true

  - type: textarea
    id: description
    attributes:
      label: Description
      description: Describe the bug
      placeholder: Tell us what happened
    validations:
      required: true

  - type: dropdown
    id: severity
    attributes:
      label: Severity
      options:
        - Low
        - Medium
        - High
        - Critical
    validations:
      required: true

  - type: checkboxes
    id: terms
    attributes:
      label: Checklist
      options:
        - label: I have searched existing issues
          required: true
        - label: I have read the documentation
          required: false
```

### Config для выбора шаблона

`.github/ISSUE_TEMPLATE/config.yml`:

```yaml
blank_issues_enabled: false
contact_links:
  - name: Documentation
    url: https://docs.example.com
    about: Check our documentation first
  - name: Discord
    url: https://discord.gg/example
    about: Ask questions in our Discord
```

---

## Связывание Issues с PR

### Ключевые слова для закрытия

Используйте в описании PR или коммите:

```
Closes #42
Fixes #42
Resolves #42

Close #42
Fix #42
Resolve #42

Closed #42
Fixed #42
Resolved #42
```

### Примеры

```bash
# В commit message
git commit -m "Fix login validation, closes #42"

# Несколько issues
git commit -m "Refactor auth module, fixes #42, closes #43"

# В описании PR
# PR Description:
# This PR implements dark mode.
#
# Closes #42
# Related to #38
```

### Связывание без закрытия

```markdown
# Просто ссылка (не закрывает)
Related to #42
See #42
Part of #42
```

---

## Закрытие Issues через коммиты

### Автоматическое закрытие

При merge PR с ключевыми словами:

```bash
# Коммит в main/master автоматически закроет issue
git commit -m "Add user authentication, fixes #15"
git push origin main
```

### Настройка default branch

Issues закрываются автоматически только при merge в default branch (обычно `main` или `master`).

### Ручное закрытие

```bash
# Через CLI
gh issue close 42

# С комментарием
gh issue close 42 --comment "Fixed in v1.2.0"

# Закрыть как дубликат
gh issue close 42 --reason "duplicate"

# Закрыть как "not planned"
gh issue close 42 --reason "not planned"
```

---

## Управление Issues через CLI

```bash
# Список issues
gh issue list
gh issue list --state open
gh issue list --state closed
gh issue list --state all

# Фильтрация
gh issue list --assignee "@me"
gh issue list --author "username"
gh issue list --label "bug"
gh issue list --milestone "v1.0"

# Просмотр issue
gh issue view 42
gh issue view 42 --web  # Открыть в браузере

# Редактирование
gh issue edit 42 --title "New title"
gh issue edit 42 --body "New body"

# Комментарии
gh issue comment 42 --body "Thanks for reporting!"

# Переоткрытие
gh issue reopen 42

# Поиск
gh issue list --search "bug in:title"
gh issue list --search "is:open label:bug"
```

---

## Best Practices

### Для создателей Issues

1. **Проверьте существующие issues** перед созданием нового
2. **Используйте понятный заголовок** — кратко и информативно
3. **Предоставьте достаточно деталей** для воспроизведения
4. **Прикрепите скриншоты/логи** когда это уместно
5. **Используйте шаблоны** если они есть

### Для maintainers

1. **Отвечайте быстро** — хотя бы acknowledge
2. **Используйте labels** для организации
3. **Закрывайте неактуальные issues** с объяснением
4. **Связывайте дубликаты** вместо закрытия без объяснений
5. **Отмечайте "good first issue"** для привлечения контрибьюторов

### Организация

1. **Используйте milestones** для планирования релизов
2. **Создайте систему labels** и документируйте её
3. **Используйте Projects** для kanban-досок
4. **Автоматизируйте** с помощью GitHub Actions

---

## Интеграция с Projects

Issues можно добавлять в GitHub Projects для визуального управления:

```bash
# Добавить issue в project
gh project item-add 1 --owner "@me" --url https://github.com/owner/repo/issues/42
```

---

## Заключение

Issues — мощный инструмент для:
- Отслеживания багов и задач
- Планирования разработки
- Коммуникации в команде
- Организации open source проектов

Правильное использование labels, milestones и templates значительно улучшает workflow и помогает поддерживать порядок в проекте.

---
[prev: 01-forking-vs-cloning](./01-forking-vs-cloning.md) | [next: 03-pull-requests](./03-pull-requests.md)