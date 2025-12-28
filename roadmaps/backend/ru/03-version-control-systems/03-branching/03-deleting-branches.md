# Deleting Branches

[prev: 02-renaming-branches](./02-renaming-branches.md) | [next: 04-switching-branches](./04-switching-branches.md)
---

## Зачем удалять ветки?

Удаление веток — важная часть гигиены репозитория:
- **Чистота репозитория** — меньше визуального шума при просмотре веток
- **Избежание путаницы** — устаревшие ветки могут ввести в заблуждение
- **Производительность** — меньше данных для синхронизации
- **Завершённость** — ветка слита и больше не нужна

## Удаление локальных веток

### Безопасное удаление (merged ветки)

```bash
# Удалить ветку, которая уже слита
git branch -d <имя-ветки>

# Пример
git branch -d feature-login
```

Флаг `-d` (delete) проверяет, была ли ветка слита в текущую ветку или в upstream. Если нет — Git откажется удалять.

### Принудительное удаление (любая ветка)

```bash
# Удалить ветку без проверки слияния
git branch -D <имя-ветки>

# Пример
git branch -D experimental-feature
```

Флаг `-D` (эквивалент `--delete --force`) удаляет ветку даже если изменения не слиты.

**Внимание:** При принудительном удалении вы можете потерять несохранённые изменения!

### Удаление нескольких веток

```bash
# Удалить несколько веток
git branch -d branch1 branch2 branch3

# Удалить все ветки, соответствующие паттерну
git branch | grep 'feature-' | xargs git branch -d

# Удалить все слитые ветки (кроме main, master, develop)
git branch --merged | grep -v '\*\|main\|master\|develop' | xargs -n 1 git branch -d
```

## Удаление удалённых веток

### Удаление ветки на remote

```bash
# Способ 1: явное удаление
git push origin --delete <имя-ветки>

# Способ 2: короткий синтаксис (пустая ссылка)
git push origin :<имя-ветки>

# Пример
git push origin --delete feature-old
git push origin :feature-old
```

### Удаление устаревших tracking references

После удаления ветки на remote, локальные ссылки остаются. Их нужно очистить:

```bash
# Очистить устаревшие remote-tracking ветки
git fetch --prune

# Или отдельной командой
git remote prune origin

# Настроить автоматическую очистку
git config --global fetch.prune true
```

### Проверка устаревших веток

```bash
# Показать remote-tracking ветки, которых нет на remote
git remote prune origin --dry-run
```

## Удаление веток с проверками

### Проверка перед удалением

```bash
# Показать слитые ветки (можно безопасно удалить)
git branch --merged

# Показать не слитые ветки (удалять осторожно!)
git branch --no-merged

# Показать слитые ветки относительно main
git branch --merged main

# Показать все remote ветки, слитые в main
git branch -r --merged main
```

### Интерактивное удаление

```bash
# Показать ветки и предложить удаление
git branch --merged | while read branch; do
    read -p "Delete branch $branch? [y/n] " answer
    if [ "$answer" = "y" ]; then
        git branch -d "$branch"
    fi
done
```

## Восстановление удалённой ветки

Если вы случайно удалили ветку, её можно восстановить (пока не прошла сборка мусора):

### Через reflog

```bash
# Найти последний коммит удалённой ветки
git reflog

# Создать ветку заново от найденного коммита
git branch <имя-ветки> <хеш-коммита>

# Пример
git reflog | grep feature-x
# abc1234 HEAD@{5}: commit: Added feature x
git branch feature-x abc1234
```

### Через git fsck

```bash
# Найти "висящие" коммиты
git fsck --lost-found

# Восстановить из найденных
git branch recovered-branch <хеш-коммита>
```

## Best Practices

### Workflow удаления

1. **Убедитесь, что ветка слита**
   ```bash
   git branch --merged | grep feature-x
   ```

2. **Удалите локальную ветку**
   ```bash
   git branch -d feature-x
   ```

3. **Удалите remote ветку**
   ```bash
   git push origin --delete feature-x
   ```

4. **Очистите tracking**
   ```bash
   git fetch --prune
   ```

### Автоматизация очистки

Добавьте в `~/.gitconfig`:

```ini
[alias]
    # Удалить слитые ветки (кроме protected)
    cleanup = "!git branch --merged | grep -v '\\*\\|main\\|master\\|develop' | xargs -n 1 git branch -d"

    # Удалить remote слитые ветки
    cleanup-remote = "!git branch -r --merged origin/main | grep -v 'main\\|master\\|develop' | sed 's/origin\\///' | xargs -n 1 git push origin --delete"

    # Полная очистка
    cleanup-all = "!git cleanup && git cleanup-remote && git fetch --prune"
```

### Регулярная очистка

```bash
# Еженедельно проверяйте устаревшие ветки
git branch -vv | grep ': gone]'

# Удалите их
git branch -vv | grep ': gone]' | awk '{print $1}' | xargs git branch -d
```

## Типичные ошибки

### 1. Удаление текущей ветки

```bash
# ОШИБКА: нельзя удалить ветку, на которой находишься
git branch -d feature-x
# error: Cannot delete branch 'feature-x' checked out at '/path/to/repo'

# РЕШЕНИЕ: сначала переключиться на другую ветку
git checkout main
git branch -d feature-x
```

### 2. Удаление не слитой ветки с -d

```bash
# ОШИБКА: ветка не слита
git branch -d unmerged-branch
# error: The branch 'unmerged-branch' is not fully merged

# РЕШЕНИЕ 1: сначала слить ветку
git merge unmerged-branch
git branch -d unmerged-branch

# РЕШЕНИЕ 2: принудительно удалить (если изменения не нужны)
git branch -D unmerged-branch
```

### 3. Удаление защищённой ветки на remote

```bash
# ОШИБКА: ветка защищена
git push origin --delete main
# remote: error: refusing to delete the current branch: refs/heads/main

# РЕШЕНИЕ: снять защиту в настройках репозитория
# или выбрать другую default ветку
```

### 4. Путаница между локальными и remote ветками

```bash
# ОШИБКА: удалили только локальную ветку
git branch -d feature-x
# Ветка всё ещё существует на origin!

# ПРАВИЛЬНО: удалить и там, и там
git branch -d feature-x
git push origin --delete feature-x
```

## Полезные команды

```bash
# Показать все ветки с датой последнего коммита
git for-each-ref --sort=-committerdate refs/heads/ \
    --format='%(committerdate:short) %(refname:short)'

# Найти ветки старше 3 месяцев
git for-each-ref --sort=-committerdate refs/heads/ \
    --format='%(committerdate:relative) %(refname:short)' | grep 'months ago'

# Показать ветки и их авторов
git for-each-ref --format='%(refname:short) %(authorname)' refs/heads/

# Удалить все локальные ветки, которых нет на remote
git branch -vv | grep ': gone]' | awk '{print $1}' | xargs git branch -D

# Показать количество коммитов в каждой ветке
git branch -v
```

## GitHub/GitLab: автоматическое удаление

### GitHub

В настройках репозитория можно включить автоматическое удаление head-ветки после merge:

Settings → General → Pull Requests → Automatically delete head branches

### GitLab

В настройках merge request:

Settings → Merge Requests → Merge options → Delete source branch when merge request is accepted

---
[prev: 02-renaming-branches](./02-renaming-branches.md) | [next: 04-switching-branches](./04-switching-branches.md)