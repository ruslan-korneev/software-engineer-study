# Pull Requests

[prev: 02-issues](./02-issues.md) | [next: 04-pr-from-fork](./04-pr-from-fork.md)
---

## Введение

**Pull Request (PR)** — это механизм GitHub для предложения изменений в репозиторий. PR позволяет:
- Обсуждать изменения перед merge
- Проводить code review
- Запускать автоматические проверки (CI/CD)
- Документировать историю изменений

---

## Создание Pull Request

### Через веб-интерфейс

1. Push вашу ветку на GitHub
2. Перейдите в репозиторий
3. GitHub предложит создать PR для недавно pushed веток
4. Или: Pull requests → New pull request
5. Выберите base и compare branches
6. Заполните описание
7. Нажмите "Create pull request"

### Через GitHub CLI

```bash
# Базовое создание
gh pr create --title "Add new feature" --body "Description"

# С указанием base branch
gh pr create --base main --head feature/my-feature

# Интерактивный режим
gh pr create

# С reviewers и labels
gh pr create \
  --title "Fix authentication bug" \
  --body "Fixes login issues" \
  --reviewer "user1,user2" \
  --label "bug,priority: high" \
  --assignee "@me"

# Создать и сразу открыть в браузере
gh pr create --web
```

### Автоматическое связывание с Issue

```bash
gh pr create \
  --title "Implement dark mode" \
  --body "Closes #42"
```

---

## Структура хорошего PR

### Заголовок

Используйте конвенцию Conventional Commits:

```
feat: add user authentication
fix: resolve memory leak in parser
docs: update API documentation
refactor: simplify database queries
test: add unit tests for auth module
chore: update dependencies
```

### Описание (Body)

```markdown
## Summary
Brief description of what this PR does.

## Changes
- Added new endpoint `/api/users`
- Updated User model with email validation
- Added unit tests for new functionality

## Screenshots
(if applicable)

## Testing
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Manual testing completed

## Related Issues
Closes #42
Related to #38

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Documentation updated
- [ ] Tests added/updated
```

---

## Draft Pull Requests

Draft PR — это PR, который ещё не готов к review.

### Создание Draft PR

```bash
# Через CLI
gh pr create --draft --title "WIP: New feature"

# Через веб-интерфейс
# При создании PR выберите "Create draft pull request"
```

### Когда использовать Draft

- Работа в процессе (WIP)
- Нужна ранняя обратная связь
- CI проверки без полного review
- Обсуждение архитектурных решений

### Преобразование Draft в обычный PR

```bash
# Через CLI
gh pr ready 42

# Или через веб-интерфейс — кнопка "Ready for review"
```

---

## Review Process

### Запрос Review

```bash
# Добавить reviewers
gh pr edit 42 --add-reviewer "user1,user2"

# Через CODEOWNERS (автоматически)
# Файл .github/CODEOWNERS назначает reviewers по путям
```

### Проведение Review

#### Типы review

| Тип | Описание |
|-----|----------|
| **Comment** | Общие комментарии без явного одобрения/отклонения |
| **Approve** | Одобрение изменений |
| **Request changes** | Запрос на изменения перед merge |

#### Через CLI

```bash
# Просмотр PR
gh pr view 42
gh pr diff 42

# Оставить review
gh pr review 42 --approve --body "LGTM!"
gh pr review 42 --request-changes --body "Please fix X"
gh pr review 42 --comment --body "Some thoughts..."
```

### Code Review Best Practices

#### Для автора PR

1. **Делайте PR небольшими** — легче ревьюить
2. **Описывайте контекст** — почему, а не только что
3. **Отвечайте на комментарии** оперативно
4. **Не принимайте критику лично** — это про код

#### Для reviewer

1. **Будьте конструктивны** — предлагайте решения
2. **Различайте критичное и предпочтения** — используйте "nit:" для мелочей
3. **Задавайте вопросы** — "Почему так?" лучше "Это неправильно"
4. **Хвалите хороший код** — не только критикуйте

#### Примеры комментариев

```markdown
# Хорошо
nit: можно использовать `const` вместо `let` здесь, так как значение не меняется

# Хорошо
Не понял, почему здесь используется sync вызов? Может стоит сделать async?

# Плохо
Это неправильно

# Плохо
Переделай
```

---

## Merge Options

GitHub предлагает три способа merge:

### 1. Merge Commit

```bash
git merge feature --no-ff
```

- Создает merge commit
- Сохраняет полную историю веток
- **Когда использовать**: важна история, публичные ветки

```
* Merge pull request #42
|\
| * commit 3
| * commit 2
| * commit 1
|/
* previous commit
```

### 2. Squash and Merge

```bash
git merge --squash feature
git commit -m "Feature: all changes"
```

- Объединяет все коммиты в один
- Чистая линейная история
- **Когда использовать**: много мелких коммитов, хотите чистую историю

```
* Feature: all changes (squashed from 3 commits)
* previous commit
```

### 3. Rebase and Merge

```bash
git rebase main
git merge feature --ff-only
```

- Перебазирует коммиты на main
- Линейная история без merge commits
- **Когда использовать**: чистые атомарные коммиты, линейная история

```
* commit 3
* commit 2
* commit 1
* previous commit
```

### Настройка в репозитории

Settings → General → Pull Requests:
- Allow merge commits
- Allow squash merging
- Allow rebase merging

---

## Разрешение конфликтов

### Через веб-интерфейс

1. GitHub покажет кнопку "Resolve conflicts"
2. Отредактируйте файлы в веб-редакторе
3. Mark as resolved
4. Commit merge

### Через командную строку

```bash
# 1. Обновите main
git checkout main
git pull origin main

# 2. Переключитесь на feature branch
git checkout feature/my-feature

# 3. Merge main в feature branch
git merge main

# 4. Решите конфликты
# Откройте файлы с конфликтами
# Удалите маркеры <<<<<<, ======, >>>>>>
# Сохраните результат

# 5. Завершите merge
git add .
git commit -m "Resolve merge conflicts"
git push origin feature/my-feature
```

### Использование rebase

```bash
# Rebase на main
git checkout feature/my-feature
git fetch origin
git rebase origin/main

# При конфликтах
# Решите конфликт в файле
git add resolved-file.js
git rebase --continue

# Если нужно отменить
git rebase --abort

# Push с force (осторожно!)
git push origin feature/my-feature --force-with-lease
```

### Инструменты для разрешения конфликтов

```bash
# Настройка merge tool
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'

# Использование
git mergetool
```

---

## PR Templates

Создайте файл `.github/PULL_REQUEST_TEMPLATE.md`:

```markdown
## Description
<!-- Describe your changes in detail -->

## Type of change
- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation update

## How Has This Been Tested?
<!-- Describe the tests you ran -->

## Checklist
- [ ] My code follows the style guidelines
- [ ] I have performed a self-review
- [ ] I have commented my code where necessary
- [ ] I have updated the documentation
- [ ] My changes generate no new warnings
- [ ] I have added tests that prove my fix/feature works
- [ ] New and existing unit tests pass locally

## Screenshots (if appropriate)

## Related Issues
<!-- Link related issues: Closes #123 -->
```

### Несколько шаблонов

```
.github/
  PULL_REQUEST_TEMPLATE/
    bug_fix.md
    feature.md
    documentation.md
```

---

## Автоматизация PR

### GitHub Actions для PR

```yaml
# .github/workflows/pr-check.yml
name: PR Check

on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: npm test

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run linter
        run: npm run lint
```

### Auto-assign Reviewers

```yaml
# .github/workflows/auto-assign.yml
name: Auto Assign

on:
  pull_request:
    types: [opened, ready_for_review]

jobs:
  assign:
    runs-on: ubuntu-latest
    steps:
      - uses: kentaro-m/auto-assign-action@v1.2.5
        with:
          configuration-path: '.github/auto-assign.yml'
```

---

## Управление PR через CLI

```bash
# Список PR
gh pr list
gh pr list --state open
gh pr list --state merged
gh pr list --author "@me"

# Просмотр PR
gh pr view 42
gh pr view 42 --web
gh pr diff 42

# Checkout PR локально
gh pr checkout 42

# Merge PR
gh pr merge 42
gh pr merge 42 --squash
gh pr merge 42 --rebase
gh pr merge 42 --merge

# Закрыть без merge
gh pr close 42

# Редактирование
gh pr edit 42 --title "New title"
gh pr edit 42 --add-label "bug"
gh pr edit 42 --add-reviewer "user1"

# Комментарии
gh pr comment 42 --body "Thanks!"
```

---

## Best Practices

### Размер PR

- **Идеально**: 200-400 строк изменений
- **Максимум**: 1000 строк
- Большие изменения разбивайте на серию PR

### Атомарность

- Один PR = одна логическая единица изменений
- Не смешивайте refactoring и новые features
- Не смешивайте форматирование и логику

### Workflow

1. Создайте ветку от актуального main
2. Делайте атомарные коммиты
3. Push и создайте PR
4. Получите review
5. Исправьте замечания
6. Merge после approval

### Описание

- Объясняйте **почему**, а не только **что**
- Добавляйте скриншоты для UI изменений
- Связывайте с issues
- Обновляйте описание при изменениях

---

## Заключение

Pull Requests — центральный механизм совместной разработки на GitHub:
- Обеспечивают code review
- Интегрируются с CI/CD
- Документируют историю решений
- Позволяют обсуждать изменения

Правильное использование PR улучшает качество кода и коммуникацию в команде.

---
[prev: 02-issues](./02-issues.md) | [next: 04-pr-from-fork](./04-pr-from-fork.md)