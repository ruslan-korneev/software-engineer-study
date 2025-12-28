# Переписывание истории в Git

[prev: 04-viewing-diffs](./04-viewing-diffs.md) | [next: 06-tagging](./06-tagging.md)
---

## Введение

Переписывание истории — мощная возможность Git, позволяющая изменять уже сделанные коммиты. Это полезно для:
- Исправления ошибок в коммитах
- Создания чистой, понятной истории
- Объединения мелких коммитов
- Удаления конфиденциальных данных

**Главное правило**: Никогда не переписывайте историю, которая уже была опубликована (pushed) и используется другими разработчиками!

---

## git commit --amend

Самый простой способ изменить последний коммит.

### Изменение сообщения коммита

```bash
# Изменить сообщение последнего коммита
git commit --amend -m "Новое сообщение"

# Открыть редактор для изменения сообщения
git commit --amend
```

### Добавление файлов в последний коммит

```bash
# Забыли добавить файл
git add forgotten_file.py
git commit --amend --no-edit  # --no-edit сохраняет старое сообщение
```

### Удаление файла из последнего коммита

```bash
# Случайно добавили лишний файл
git reset HEAD~1 --soft
git reset HEAD secret.txt
git commit -c ORIG_HEAD
```

### Изменение автора

```bash
git commit --amend --author="New Author <new@email.com>"
```

### Важно помнить

```bash
# --amend создаёт НОВЫЙ коммит с новым хэшем
# Старый коммит: abc123
git commit --amend -m "New message"
# Новый коммит: def456 (abc123 больше недоступен через ветку)
```

**Опасность**: Если коммит уже запушен, потребуется `force push`, что может сломать работу коллег.

```bash
# Проверить, запушен ли коммит
git log origin/main..HEAD
# Если пусто — коммит уже на сервере!
```

---

## git rebase -i (Interactive Rebase)

Интерактивный rebase — самый мощный инструмент для переписывания истории.

### Базовое использование

```bash
# Изменить последние 3 коммита
git rebase -i HEAD~3

# Изменить коммиты от определённого момента
git rebase -i abc123

# Изменить все коммиты от ответвления от main
git rebase -i main
```

### Интерфейс редактора

```bash
git rebase -i HEAD~4
```

Откроется редактор:
```
pick abc1234 Add user model
pick def5678 Add user validation
pick ghi9012 Fix typo in validation
pick jkl3456 Add user tests

# Commands:
# p, pick = use commit as is
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# d, drop = remove commit
# x, exec = run command
```

### Команды интерактивного rebase

#### pick (p) — использовать как есть

```
pick abc1234 Add user model
```

Коммит остаётся без изменений. По умолчанию все коммиты помечены как `pick`.

#### reword (r) — изменить сообщение

```
reword abc1234 Add user model
```

После сохранения откроется редактор для изменения сообщения коммита.

```bash
# Пример: исправить опечатку в сообщении 3-го коммита назад
git rebase -i HEAD~3
# Изменить pick на reword для нужного коммита
# Сохранить и закрыть
# Откроется редактор — ввести новое сообщение
```

#### edit (e) — остановиться для редактирования

```
edit abc1234 Add user model
```

Git остановится после этого коммита, позволяя внести изменения.

```bash
git rebase -i HEAD~3
# Пометить коммит как edit

# Git остановится:
# Stopped at abc1234... Add user model
# You can amend the commit now, with
#   git commit --amend

# Внести изменения
git add changed_file.py
git commit --amend

# Продолжить rebase
git rebase --continue
```

#### squash (s) — объединить с предыдущим

```
pick abc1234 Add user model
squash def5678 Add user validation
squash ghi9012 Fix typo
```

Все три коммита объединятся в один. Откроется редактор для создания объединённого сообщения.

```bash
# Результат: один коммит вместо трёх
# С объединённым сообщением:
# Add user model
#
# Add user validation
#
# Fix typo
```

#### fixup (f) — объединить, отбросив сообщение

```
pick abc1234 Add user model
fixup def5678 Small fix
fixup ghi9012 Another small fix
```

Как squash, но сообщения fixup-коммитов отбрасываются.

```bash
# Результат: один коммит с сообщением "Add user model"
# Сообщения "Small fix" и "Another small fix" потеряны
```

#### drop (d) — удалить коммит

```
pick abc1234 Add user model
drop def5678 WIP commit  # Этот коммит будет удалён
pick ghi9012 Add tests
```

Или просто удалите строку:
```
pick abc1234 Add user model
pick ghi9012 Add tests
```

#### exec (x) — выполнить команду

```
pick abc1234 Add feature
exec npm test
pick def5678 Add another feature
exec npm test
```

Полезно для проверки, что каждый коммит проходит тесты.

### Изменение порядка коммитов

Просто поменяйте строки местами:

```
# Было:
pick abc1234 First commit
pick def5678 Second commit
pick ghi9012 Third commit

# Стало:
pick ghi9012 Third commit
pick abc1234 First commit
pick def5678 Second commit
```

### Разделение коммита

```bash
git rebase -i HEAD~3
# Пометить коммит как edit

# Когда Git остановится:
git reset HEAD~1  # Отменить коммит, сохранив изменения

# Создать несколько коммитов
git add file1.py
git commit -m "Add file1"

git add file2.py
git commit -m "Add file2"

# Продолжить rebase
git rebase --continue
```

---

## Практические сценарии

### Сценарий 1: Очистка истории перед PR

```bash
# Было много мелких коммитов
git log --oneline
# abc123 Fix typo
# def456 WIP
# ghi789 Add feature part 2
# jkl012 Add feature part 1

# Объединить в один чистый коммит
git rebase -i HEAD~4

# В редакторе:
pick jkl012 Add feature part 1
fixup ghi789 Add feature part 2
fixup def456 WIP
fixup abc123 Fix typo

# Изменить сообщение первого
reword jkl012 Add complete feature
fixup ghi789 Add feature part 2
fixup def456 WIP
fixup abc123 Fix typo

# Результат: один коммит "Add complete feature"
```

### Сценарий 2: Исправление коммита в середине истории

```bash
git log --oneline
# abc123 Latest commit
# def456 Another commit
# ghi789 Commit with bug  # <- нужно исправить
# jkl012 Earlier commit

git rebase -i jkl012
# Пометить ghi789 как edit

# Когда Git остановится на ghi789:
# Внести исправления
git add fixed_file.py
git commit --amend
git rebase --continue
```

### Сценарий 3: Удаление конфиденциальных данных

```bash
# Если секрет в последнем коммите
git reset HEAD~1
# Удалить секрет из файла
git add .
git commit -m "Add config (without secrets)"

# Если секрет глубоко в истории — используйте filter-branch или BFG
```

---

## Разрешение конфликтов при rebase

```bash
# При конфликте Git остановится:
# CONFLICT (content): Merge conflict in file.py

# 1. Разрешить конфликт в файле
# 2. Добавить разрешённый файл
git add file.py

# 3. Продолжить rebase
git rebase --continue

# Или отменить весь rebase
git rebase --abort
```

---

## git filter-branch (устаревший, но важный)

`filter-branch` позволяет массово изменять историю.

**Важно**: Эта команда считается устаревшей. Для большинства задач лучше использовать `git filter-repo` (отдельный инструмент).

### Удаление файла из всей истории

```bash
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch path/to/secret.txt' \
  --prune-empty --tag-name-filter cat -- --all
```

### Изменение email во всех коммитах

```bash
git filter-branch --env-filter '
if [ "$GIT_AUTHOR_EMAIL" = "old@email.com" ]; then
    export GIT_AUTHOR_EMAIL="new@email.com"
    export GIT_COMMITTER_EMAIL="new@email.com"
fi
' --tag-name-filter cat -- --all
```

### Рекомендация: используйте git filter-repo

```bash
# Установка
pip install git-filter-repo

# Удаление файла
git filter-repo --path secret.txt --invert-paths

# Изменение email
git filter-repo --email-callback '
    return email.replace(b"old@email.com", b"new@email.com")
'
```

---

## Опасности переписывания истории

### 1. Потеря работы коллег

```bash
# Вы переписали историю и сделали force push
git push --force

# Коллега, работавший на основе старой истории:
git pull
# error: Your local changes would be overwritten
# Или ещё хуже — молчаливое перезатирание его работы
```

### 2. Сломанные ссылки

- CI/CD пайплайны, ссылающиеся на старые коммиты
- Issue и PR, связанные с удалёнными коммитами
- Внешние ссылки на определённые хэши

### 3. Потеря истории аудита

В некоторых проектах история изменений важна для аудита и compliance.

---

## Когда можно переписывать историю

### Безопасно:

1. **Локальные коммиты** — ещё не запушены
   ```bash
   git log origin/main..HEAD  # Покажет незапушенные коммиты
   ```

2. **Персональные ветки** — никто кроме вас не использует
   ```bash
   git push --force-with-lease origin my-feature
   ```

3. **Перед созданием PR** — подготовка чистой истории

### Опасно/Запрещено:

1. **main/master** — основная ветка
2. **Shared ветки** — используемые другими
3. **После merge в main** — история уже интегрирована
4. **Публичные репозитории** — внешние зависимости

---

## Безопасный force push

```bash
# --force-with-lease безопаснее чем --force
# Не перезапишет, если remote изменился
git push --force-with-lease origin feature-branch

# Ещё безопаснее: указать ожидаемый коммит
git push --force-with-lease=feature-branch:abc123 origin feature-branch
```

---

## Best Practices

### 1. Создавайте backup перед rebase

```bash
git branch backup-before-rebase
git rebase -i HEAD~5
# Если что-то пошло не так:
git reset --hard backup-before-rebase
```

### 2. Используйте fixup commits во время работы

```bash
# Создать fixup для конкретного коммита
git commit --fixup=abc123

# Позже применить все fixup автоматически
git rebase -i --autosquash HEAD~10
```

### 3. Настройте autosquash по умолчанию

```bash
git config --global rebase.autosquash true
```

### 4. Проверяйте историю после rebase

```bash
git log --oneline
git diff ORIG_HEAD  # Сравнить с состоянием до rebase
```

### 5. Используйте rebase.missingCommitsCheck

```bash
git config --global rebase.missingCommitsCheck warn
# Предупредит, если вы случайно удалили коммит
```

---

## Восстановление после неудачного rebase

```bash
# Во время rebase — отмена
git rebase --abort

# После завершения rebase
git reflog
# Найти коммит до rebase
git reset --hard HEAD@{5}

# Или использовать ORIG_HEAD
git reset --hard ORIG_HEAD
```

---

## Резюме команд

| Команда | Описание |
|---------|----------|
| `git commit --amend` | Изменить последний коммит |
| `git commit --amend -m "msg"` | Изменить сообщение |
| `git commit --amend --no-edit` | Добавить файлы без изменения сообщения |
| `git rebase -i HEAD~n` | Интерактивный rebase последних n коммитов |
| `git rebase -i main` | Rebase от ветки main |
| `git rebase --continue` | Продолжить после разрешения конфликта |
| `git rebase --abort` | Отменить rebase |
| `git commit --fixup=<hash>` | Создать fixup-коммит |
| `git rebase -i --autosquash` | Автоматически применить fixup |
| `git push --force-with-lease` | Безопасный force push |
| `git filter-repo` | Массовое изменение истории |

---

## Таблица решений

| Задача | Решение |
|--------|---------|
| Изменить сообщение последнего коммита | `git commit --amend` |
| Добавить файл в последний коммит | `git add file && git commit --amend` |
| Изменить сообщение старого коммита | `git rebase -i` + `reword` |
| Объединить коммиты | `git rebase -i` + `squash/fixup` |
| Удалить коммит | `git rebase -i` + `drop` |
| Разделить коммит | `git rebase -i` + `edit` + `reset` |
| Изменить порядок коммитов | `git rebase -i` + переставить строки |
| Удалить файл из всей истории | `git filter-repo` |

---
[prev: 04-viewing-diffs](./04-viewing-diffs.md) | [next: 06-tagging](./06-tagging.md)