# PR from Fork (Pull Request из форка)

[prev: 03-pull-requests](./03-pull-requests.md) | [next: 05-collaborators](./05-collaborators.md)
---

## Введение

Создание PR из форка — основной способ контрибуции в open source проекты. Этот workflow позволяет предлагать изменения в репозитории, к которым у вас нет прямого доступа на запись.

---

## Workflow "Fork → Branch → PR"

### Полная последовательность действий

```
1. Fork репозитория на GitHub
         ↓
2. Clone форка локально
         ↓
3. Настройка upstream
         ↓
4. Создание feature branch
         ↓
5. Внесение изменений
         ↓
6. Push в свой форк (origin)
         ↓
7. Создание PR в оригинальный репозиторий
         ↓
8. Code review и исправления
         ↓
9. Merge в upstream
```

### Детальное описание шагов

#### Шаг 1: Fork репозитория

1. Перейдите на страницу репозитория на GitHub
2. Нажмите кнопку "Fork" в правом верхнем углу
3. Выберите свой аккаунт как destination
4. Дождитесь создания форка

#### Шаг 2: Clone форка

```bash
# Клонируйте ваш форк (НЕ оригинал!)
git clone https://github.com/YOUR-USERNAME/project.git
cd project

# Убедитесь, что origin указывает на ваш форк
git remote -v
# origin  https://github.com/YOUR-USERNAME/project.git (fetch)
# origin  https://github.com/YOUR-USERNAME/project.git (push)
```

#### Шаг 3: Настройка upstream

```bash
# Добавьте оригинальный репозиторий как upstream
git remote add upstream https://github.com/ORIGINAL-OWNER/project.git

# Проверьте настройку
git remote -v
# origin    https://github.com/YOUR-USERNAME/project.git (fetch)
# origin    https://github.com/YOUR-USERNAME/project.git (push)
# upstream  https://github.com/ORIGINAL-OWNER/project.git (fetch)
# upstream  https://github.com/ORIGINAL-OWNER/project.git (push)
```

#### Шаг 4: Создание feature branch

```bash
# Сначала синхронизируйтесь с upstream
git fetch upstream
git checkout main
git merge upstream/main

# Создайте новую ветку для ваших изменений
git checkout -b feature/add-dark-mode

# Или для bugfix
git checkout -b fix/login-validation
```

#### Шаг 5: Внесение изменений

```bash
# Внесите изменения в код
# ...

# Проверьте статус
git status

# Добавьте изменения
git add .

# Закоммитьте с понятным сообщением
git commit -m "feat: add dark mode toggle to settings"
```

#### Шаг 6: Push в форк

```bash
# Push ветку в ваш форк (origin)
git push origin feature/add-dark-mode
```

#### Шаг 7: Создание PR

```bash
# Через GitHub CLI
gh pr create \
  --repo ORIGINAL-OWNER/project \
  --title "feat: add dark mode toggle" \
  --body "Adds dark mode support to the settings page.

Closes #42"

# Или через веб-интерфейс:
# 1. Перейдите на страницу оригинального репозитория
# 2. GitHub покажет баннер с предложением создать PR
# 3. Или: Pull requests → New pull request → compare across forks
```

---

## Настройка Upstream

### Почему это важно

- Upstream позволяет получать обновления из оригинального репозитория
- Необходим для синхронизации перед созданием PR
- Помогает избежать конфликтов

### Команды для работы с upstream

```bash
# Добавить upstream
git remote add upstream https://github.com/ORIGINAL-OWNER/repo.git

# Проверить remote-ы
git remote -v

# Получить изменения из upstream (без merge)
git fetch upstream

# Посмотреть ветки upstream
git branch -r | grep upstream

# Удалить upstream (если ошиблись)
git remote remove upstream

# Изменить URL upstream
git remote set-url upstream https://github.com/NEW-OWNER/repo.git
```

### Настройка для SSH

```bash
# Если используете SSH для GitHub
git remote add upstream git@github.com:ORIGINAL-OWNER/repo.git
```

---

## Синхронизация перед PR

### Зачем синхронизировать

- Получить последние изменения из основного проекта
- Уменьшить вероятность конфликтов
- Показать maintainers, что вы работаете с актуальным кодом

### Метод 1: Merge (проще)

```bash
# Получите изменения из upstream
git fetch upstream

# Переключитесь на main
git checkout main

# Merge upstream/main в ваш main
git merge upstream/main

# Push обновленный main в ваш форк
git push origin main

# Переключитесь на feature branch
git checkout feature/my-feature

# Merge main в feature branch
git merge main

# Решите конфликты, если есть
# ...

# Push обновленную ветку
git push origin feature/my-feature
```

### Метод 2: Rebase (чистая история)

```bash
# Получите изменения
git fetch upstream

# Переключитесь на feature branch
git checkout feature/my-feature

# Rebase на upstream/main
git rebase upstream/main

# При конфликтах:
# 1. Решите конфликт в файле
# 2. git add <file>
# 3. git rebase --continue

# Push с --force-with-lease
git push origin feature/my-feature --force-with-lease
```

### Когда какой метод использовать

| Ситуация | Рекомендация |
|----------|--------------|
| PR ещё не создан | Rebase |
| PR на review, нет комментариев | Rebase |
| PR на review, есть обсуждение | Merge (сохраняет контекст) |
| Много коммитов, хотите squash | Rebase -i |
| Не уверены | Merge (безопаснее) |

---

## Обновление PR после ревью

### Внесение исправлений

```bash
# 1. Внесите запрошенные изменения
# ...

# 2. Добавьте изменения
git add .

# 3. Создайте новый коммит
git commit -m "fix: address review comments"

# 4. Push в ту же ветку
git push origin feature/my-feature

# PR автоматически обновится!
```

### Squash коммитов перед merge (если просят)

```bash
# Посмотрите историю
git log --oneline -5

# Interactive rebase для squash
git rebase -i HEAD~3  # 3 = количество коммитов

# В редакторе:
# pick abc123 feat: add dark mode
# squash def456 fix: typo
# squash ghi789 fix: review comments

# Сохраните и отредактируйте commit message

# Force push
git push origin feature/my-feature --force-with-lease
```

### Если upstream изменился во время review

```bash
# Синхронизируйте ветку
git fetch upstream
git rebase upstream/main

# Решите конфликты, если есть
# ...

# Push обновления
git push origin feature/my-feature --force-with-lease

# Сообщите в PR, что обновили ветку
```

---

## Типичные проблемы и решения

### Проблема 1: "This branch has conflicts"

```bash
# Решение: синхронизируйте с upstream
git fetch upstream
git checkout feature/my-feature
git rebase upstream/main

# Решите конфликты
git add .
git rebase --continue

git push origin feature/my-feature --force-with-lease
```

### Проблема 2: Случайно запушили в main

```bash
# Вернитесь к состоянию upstream
git checkout main
git fetch upstream
git reset --hard upstream/main
git push origin main --force

# Создайте правильную ветку
git checkout -b feature/my-feature
# ... перенесите изменения
```

### Проблема 3: Забыли добавить upstream

```bash
# Добавьте после факта
git remote add upstream https://github.com/ORIGINAL/repo.git
git fetch upstream
```

### Проблема 4: PR показывает лишние коммиты

Это происходит, когда ваш main расходится с upstream.

```bash
# Решение 1: Rebase на свежий upstream
git fetch upstream
git checkout feature/my-feature
git rebase upstream/main
git push origin feature/my-feature --force-with-lease

# Решение 2: Создать новую ветку
git fetch upstream
git checkout upstream/main
git checkout -b feature/my-feature-v2
git cherry-pick <commit-hashes>
git push origin feature/my-feature-v2
# Закройте старый PR, создайте новый
```

### Проблема 5: Нужно обновить форк, но есть локальные изменения

```bash
# Сохраните изменения
git stash

# Обновите main
git checkout main
git fetch upstream
git merge upstream/main
git push origin main

# Вернитесь к работе
git checkout feature/my-feature
git stash pop
```

### Проблема 6: CI не проходит

```bash
# 1. Посмотрите логи CI на GitHub
# 2. Исправьте проблемы локально
# 3. Запустите тесты локально
npm test  # или другая команда

# 4. Запушьте исправления
git add .
git commit -m "fix: resolve CI failures"
git push origin feature/my-feature
```

---

## Лучшие практики для PR из форка

### Перед созданием PR

1. **Прочитайте CONTRIBUTING.md** — у проекта могут быть особые требования
2. **Синхронизируйтесь с upstream** — ваш код должен быть актуальным
3. **Запустите тесты локально** — не полагайтесь только на CI
4. **Проверьте стиль кода** — используйте линтеры проекта

### При создании PR

1. **Понятный заголовок** — следуйте конвенции проекта
2. **Подробное описание** — объясните что и почему
3. **Ссылка на issue** — если исправляете известную проблему
4. **Небольшой размер** — легче ревьюить

### Во время review

1. **Отвечайте быстро** — maintainers ценят активных контрибьюторов
2. **Будьте вежливы** — даже при несогласии
3. **Задавайте вопросы** — если что-то непонятно
4. **Благодарите за feedback** — это бесплатная помощь

### После merge

```bash
# Удалите локальную ветку
git checkout main
git branch -d feature/my-feature

# Удалите remote ветку
git push origin --delete feature/my-feature

# Синхронизируйте main
git fetch upstream
git merge upstream/main
git push origin main
```

---

## Пример полного workflow

```bash
# === НАЧАЛО ===

# 1. Fork через GitHub UI
# 2. Clone форка
git clone https://github.com/myusername/awesome-project.git
cd awesome-project

# 3. Настройка upstream
git remote add upstream https://github.com/original/awesome-project.git

# 4. Синхронизация
git fetch upstream
git checkout main
git merge upstream/main

# 5. Создание ветки
git checkout -b fix/typo-in-readme

# 6. Изменения
# ... редактируем файлы ...

# 7. Коммит
git add README.md
git commit -m "docs: fix typo in installation section"

# 8. Push
git push origin fix/typo-in-readme

# 9. Создание PR
gh pr create \
  --repo original/awesome-project \
  --title "docs: fix typo in README" \
  --body "Fixed a typo in the installation section."

# === ПОСЛЕ REVIEW ===

# 10. Внесение исправлений (если нужно)
# ... редактируем ...
git add .
git commit -m "fix: address review feedback"
git push origin fix/typo-in-readme

# === ПОСЛЕ MERGE ===

# 11. Очистка
git checkout main
git branch -d fix/typo-in-readme
git push origin --delete fix/typo-in-readme
git fetch upstream
git merge upstream/main
git push origin main
```

---

## Заключение

PR из форка — стандартный способ контрибуции в open source:

1. **Fork** — создаёт вашу копию проекта
2. **Clone** — загружает код локально
3. **Upstream** — связывает с оригиналом
4. **Branch** — изолирует изменения
5. **PR** — предлагает изменения maintainers

Соблюдение best practices увеличивает шансы на принятие вашего PR и формирует репутацию надёжного контрибьютора.

---
[prev: 03-pull-requests](./03-pull-requests.md) | [next: 05-collaborators](./05-collaborators.md)