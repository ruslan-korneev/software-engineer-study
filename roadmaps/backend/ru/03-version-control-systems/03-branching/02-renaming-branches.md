# Renaming Branches

## Зачем переименовывать ветки?

Переименование веток может понадобиться в различных ситуациях:
- Опечатка в названии ветки
- Изменение конвенции именования в команде
- Уточнение назначения ветки
- Переименование `master` в `main` (современный стандарт)

## Переименование локальной ветки

### Переименование текущей ветки

```bash
# Переименовать ветку, на которой находишься
git branch -m <новое-имя>

# Пример: переименовать текущую ветку в feature-auth
git branch -m feature-auth
```

Флаг `-m` означает "move" (перемещение), что в контексте веток равносильно переименованию.

### Переименование любой ветки

```bash
# Переименовать ветку, находясь на другой ветке
git branch -m <старое-имя> <новое-имя>

# Пример
git branch -m old-feature new-feature
```

### Принудительное переименование

Если ветка с новым именем уже существует, используйте `-M` (заглавная):

```bash
# Принудительное переименование (перезапишет существующую ветку!)
git branch -M <старое-имя> <новое-имя>

# Пример
git branch -M feature-old feature-new
```

**Внимание:** `-M` удалит существующую ветку с новым именем без предупреждения!

## Переименование удалённой ветки

Git не имеет прямой команды для переименования удалённой ветки. Процесс состоит из нескольких шагов:

### Пошаговый процесс

```bash
# 1. Переименовать локальную ветку
git branch -m old-name new-name

# 2. Удалить старую ветку на remote
git push origin --delete old-name

# 3. Запушить новую ветку
git push origin new-name

# 4. Настроить отслеживание (tracking)
git push --set-upstream origin new-name
```

### Сокращённый вариант

```bash
# Всё в несколько команд
git branch -m old-name new-name
git push origin :old-name new-name
git push origin -u new-name
```

Синтаксис `:old-name` в push означает "удалить old-name на remote".

## Переименование main/master ветки

### Локальное переименование master в main

```bash
# 1. Переключиться на ветку master
git checkout master

# 2. Переименовать её в main
git branch -m master main

# 3. Запушить новую ветку
git push -u origin main

# 4. Удалить старую ветку на remote
git push origin --delete master
```

### Изменение default branch на GitHub/GitLab

После переименования нужно изменить default branch в настройках репозитория:

**GitHub:**
1. Settings → Branches → Default branch
2. Выбрать `main`
3. Update

**GitLab:**
1. Settings → Repository → Default branch
2. Выбрать `main`
3. Save changes

### Обновление у других участников команды

После переименования ветки на remote, другие участники должны выполнить:

```bash
# Получить изменения
git fetch origin

# Переключиться на новую ветку
git checkout main

# Настроить tracking
git branch -u origin/main main

# Удалить локальную ссылку на старую ветку
git remote prune origin
```

## Best Practices

### Когда переименовывать

1. **Сразу после создания** — если заметили опечатку
2. **До первого push** — пока ветка локальная, переименование безболезненно
3. **Согласованно с командой** — предупредите коллег перед переименованием общих веток

### Рекомендации

```bash
# Проверить текущее имя ветки перед переименованием
git branch --show-current

# Убедиться, что ветка не используется другими
git branch -r | grep <имя-ветки>

# После переименования проверить результат
git branch -a
```

### Документирование изменений

При переименовании важных веток (особенно default branch):
- Обновите README с информацией об изменении
- Уведомите команду через Slack/email
- Обновите CI/CD конфигурации
- Проверьте защищённые ветки (branch protection rules)

## Типичные ошибки

### 1. Переименование без обновления tracking

```bash
# ОШИБКА: переименовали локально, но tracking указывает на старую
git branch -m old-name new-name
git push origin new-name
# Ветка теперь не отслеживает remote!

# ПРАВИЛЬНО: обновить tracking
git push -u origin new-name
```

### 2. Забыли удалить старую ветку на remote

```bash
# После переименования на remote остаётся старая ветка
git branch -m old-name new-name
git push origin new-name
# old-name всё ещё существует на origin!

# ПРАВИЛЬНО: удалить старую ветку
git push origin --delete old-name
```

### 3. Переименование защищённой ветки

```bash
# Попытка удалить защищённую ветку
git push origin --delete main
# error: refusing to delete the current branch

# РЕШЕНИЕ: сначала снять защиту в настройках репозитория
```

### 4. Конфликт имён

```bash
# ОШИБКА: ветка с таким именем уже существует
git branch -m old-branch existing-branch
# fatal: A branch named 'existing-branch' already exists

# РЕШЕНИЕ 1: выбрать другое имя
git branch -m old-branch new-unique-name

# РЕШЕНИЕ 2: принудительно перезаписать (осторожно!)
git branch -M old-branch existing-branch
```

## Полезные команды

```bash
# Проверить, какие ветки отслеживают какие remote
git branch -vv

# Показать все ветки с датой последнего коммита
git branch -v --sort=-committerdate

# Найти ветки, содержащие определённое слово
git branch --list '*feature*'

# Очистить устаревшие ссылки на удалённые ветки
git remote prune origin
git fetch --prune
```

## Автоматизация через Git aliases

Добавьте в `~/.gitconfig`:

```ini
[alias]
    # Переименовать текущую ветку
    rename = branch -m

    # Переименовать и запушить
    rename-remote = "!f() { \
        old=$(git branch --show-current); \
        git branch -m $1; \
        git push origin :$old $1; \
        git push -u origin $1; \
    }; f"
```

Использование:

```bash
git rename new-name                    # локальное переименование
git rename-remote new-name             # переименование с обновлением remote
```
