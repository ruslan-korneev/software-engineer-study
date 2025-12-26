# Reset Modes - Режимы Reset

## Три дерева Git

Чтобы понять `git reset`, нужно понимать три "дерева" (три состояния файлов) в Git:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Three Trees of Git                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│   │  Working    │    │   Staging   │    │    HEAD     │        │
│   │  Directory  │    │    Area     │    │  (Commit)   │        │
│   │             │    │   (Index)   │    │             │        │
│   └─────────────┘    └─────────────┘    └─────────────┘        │
│         │                  │                  │                 │
│         │    git add       │   git commit     │                 │
│         │ ───────────────► │ ───────────────► │                 │
│         │                  │                  │                 │
│   Ваши файлы         Подготовлено        Зафиксировано         │
│   на диске           для коммита         в истории             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Working Directory (Рабочая директория)

```bash
# Это ваши файлы на диске
# Всё, что вы видите в редакторе и файловом менеджере

ls -la
# Это Working Directory
```

### Staging Area (Index, Индекс)

```bash
# Файлы, подготовленные для следующего коммита
git add file.txt
# file.txt теперь в staging area

# Посмотреть содержимое staging area
git ls-files --stage
```

### HEAD (последний коммит)

```bash
# Снимок файлов в последнем коммите текущей ветки
git show HEAD:file.txt
# Показывает file.txt из последнего коммита
```

## Обзор git reset

`git reset` перемещает HEAD (и ветку) на указанный коммит и может изменять staging area и working directory.

```bash
git reset [--soft | --mixed | --hard] <commit>
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    git reset modes                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                Working Dir    Staging    HEAD                   │
│                                                                  │
│   --soft           ✗            ✗         ✓                     │
│   --mixed          ✗            ✓         ✓                     │
│   --hard           ✓            ✓         ✓                     │
│                                                                  │
│   ✓ = изменяется                                                │
│   ✗ = не изменяется                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## git reset --soft

**Перемещает только HEAD** на указанный коммит. Staging area и working directory остаются без изменений.

```bash
git reset --soft <commit>
```

### Что происходит

```
До reset --soft HEAD~:

Working Dir    Staging       HEAD (main)
    │             │              │
    ▼             ▼              ▼
┌───────┐    ┌───────┐      ┌───────┐
│ file: │    │ file: │      │ file: │  Commit C
│  "C"  │    │  "C"  │      │  "C"  │
└───────┘    └───────┘      └───┬───┘
                                │
                            ┌───▼───┐
                            │ file: │  Commit B
                            │  "B"  │
                            └───────┘

После reset --soft HEAD~:

Working Dir    Staging       HEAD (main)
    │             │              │
    ▼             ▼              │
┌───────┐    ┌───────┐          │
│ file: │    │ file: │          │
│  "C"  │    │  "C"  │          │  (изменения "staged")
└───────┘    └───────┘          │
                            ┌───▼───┐
                            │ file: │  Commit B
                            │  "B"  │
                            └───────┘
```

### Примеры использования

```bash
# Отменить последний коммит, сохранив изменения в staging
git reset --soft HEAD~

# Теперь можно:
# 1. Изменить сообщение коммита
git commit -m "Новое сообщение"

# 2. Добавить забытые файлы
git add forgotten_file.txt
git commit -m "Полный коммит"

# 3. Объединить несколько коммитов в один (squash)
git reset --soft HEAD~3
git commit -m "Объединённый коммит"
```

### Use cases для --soft

```bash
# 1. Исправить сообщение последнего коммита (альтернатива --amend)
git reset --soft HEAD~
git commit -m "Исправленное сообщение"

# 2. Объединить коммиты (squash) без интерактивного rebase
git reset --soft HEAD~5
git commit -m "feat: big feature complete"

# 3. Добавить изменения к последнему коммиту
git reset --soft HEAD~
git add new_file.txt
git commit -m "Исходное сообщение + новый файл"
```

## git reset --mixed (default)

**Перемещает HEAD и сбрасывает staging area**, но оставляет working directory без изменений.

```bash
git reset --mixed <commit>
# или просто
git reset <commit>
```

### Что происходит

```
До reset --mixed HEAD~:

Working Dir    Staging       HEAD (main)
    │             │              │
    ▼             ▼              ▼
┌───────┐    ┌───────┐      ┌───────┐
│ file: │    │ file: │      │ file: │  Commit C
│  "C"  │    │  "C"  │      │  "C"  │
└───────┘    └───────┘      └───┬───┘
                                │
                            ┌───▼───┐
                            │ file: │  Commit B
                            │  "B"  │
                            └───────┘

После reset --mixed HEAD~:

Working Dir    Staging       HEAD (main)
    │             │              │
    ▼             │              │
┌───────┐        │              │
│ file: │        │              │  (изменения "unstaged")
│  "C"  │        │              │
└───────┘    ┌───▼───┐      ┌───▼───┐
             │ file: │      │ file: │  Commit B
             │  "B"  │      │  "B"  │
             └───────┘      └───────┘
```

### Примеры использования

```bash
# Отменить последний коммит, оставив изменения unstaged
git reset HEAD~

# Убрать файл из staging area (unstage)
git reset HEAD file.txt
# или современный способ
git restore --staged file.txt

# Отменить git add
git add .
# Упс, добавил лишнее
git reset
# Все файлы убраны из staging
```

### Use cases для --mixed

```bash
# 1. Unstage всех файлов
git add .
git reset  # Убирает всё из staging

# 2. Unstage конкретного файла
git reset HEAD -- path/to/file.txt

# 3. Разбить коммит на несколько
git reset HEAD~
# Теперь файлы unstaged, можно добавлять по частям
git add feature_part1.txt
git commit -m "feat: part 1"
git add feature_part2.txt
git commit -m "feat: part 2"

# 4. Пересмотреть, что войдёт в коммит
git reset HEAD~
git status  # Смотрим все изменения
git add -p  # Добавляем интерактивно
git commit -m "Тщательно отобранные изменения"
```

## git reset --hard

**Перемещает HEAD, сбрасывает staging area И working directory**. Это "полный откат" — изменения теряются!

```bash
git reset --hard <commit>
```

### Что происходит

```
До reset --hard HEAD~:

Working Dir    Staging       HEAD (main)
    │             │              │
    ▼             ▼              ▼
┌───────┐    ┌───────┐      ┌───────┐
│ file: │    │ file: │      │ file: │  Commit C
│  "C"  │    │  "C"  │      │  "C"  │
└───────┘    └───────┘      └───┬───┘
                                │
                            ┌───▼───┐
                            │ file: │  Commit B
                            │  "B"  │
                            └───────┘

После reset --hard HEAD~:

Working Dir    Staging       HEAD (main)
    │             │              │
    ▼             ▼              ▼
┌───────┐    ┌───────┐      ┌───────┐
│ file: │    │ file: │      │ file: │  Commit B
│  "B"  │    │  "B"  │      │  "B"  │
└───────┘    └───────┘      └───────┘

                            ┌ ─ ─ ─ ┐
                              file:    Commit C
                            │  "C"  │  (ПОТЕРЯН!)
                            └ ─ ─ ─ ┘
```

### Примеры использования

```bash
# Полностью отменить последний коммит
git reset --hard HEAD~
# ⚠️ Изменения из коммита ПОТЕРЯНЫ

# Сбросить до состояния remote
git fetch origin
git reset --hard origin/main
# ⚠️ Все локальные изменения ПОТЕРЯНЫ

# Очистить working directory (альтернатива git checkout .)
git reset --hard HEAD
# ⚠️ Все несохранённые изменения ПОТЕРЯНЫ
```

### Use cases для --hard

```bash
# 1. Начать с чистого листа (сбросить к remote)
git fetch origin
git reset --hard origin/main

# 2. Отменить неудачный merge
git merge feature-branch
# Много конфликтов, не хочу разбираться
git reset --hard HEAD~

# 3. Отменить все локальные изменения
git reset --hard HEAD

# 4. Вернуться к определённому коммиту
git reset --hard abc1234

# 5. Очистить после экспериментов
git reset --hard origin/main
```

## Сравнительная таблица

| Режим | HEAD | Staging | Working Dir | Безопасность |
|-------|------|---------|-------------|--------------|
| --soft | Перемещается | Не меняется | Не меняется | Безопасно |
| --mixed | Перемещается | Сбрасывается | Не меняется | Безопасно |
| --hard | Перемещается | Сбрасывается | Сбрасывается | ОПАСНО |

## Визуальное объяснение

### Начальное состояние

```bash
# Сделали 3 коммита
git log --oneline
# c3 (HEAD -> main) Third commit
# c2 Second commit
# c1 First commit

# В файле написано "Third"
cat file.txt
# Third

# В staging пусто
git status
# nothing to commit
```

### После reset --soft HEAD~2

```bash
git reset --soft HEAD~2

# HEAD переместился
git log --oneline
# c1 (HEAD -> main) First commit

# Working directory НЕ изменился
cat file.txt
# Third

# Изменения в STAGING
git status
# Changes to be committed:
#   modified: file.txt

git diff --cached
# -First
# +Third
```

### После reset --mixed HEAD~2 (или reset HEAD~2)

```bash
git reset HEAD~2  # --mixed по умолчанию

# HEAD переместился
git log --oneline
# c1 (HEAD -> main) First commit

# Working directory НЕ изменился
cat file.txt
# Third

# Изменения НЕ в staging (unstaged)
git status
# Changes not staged for commit:
#   modified: file.txt

git diff
# -First
# +Third
```

### После reset --hard HEAD~2

```bash
git reset --hard HEAD~2

# HEAD переместился
git log --oneline
# c1 (HEAD -> main) First commit

# Working directory ИЗМЕНИЛСЯ!
cat file.txt
# First

# Staging пуст
git status
# nothing to commit, working tree clean

# Изменения ПОТЕРЯНЫ!
```

## Опасности --hard

### Потеря незакоммиченных изменений

```bash
# У вас есть несохранённые изменения
echo "Important work" >> important.txt

# Случайный reset --hard
git reset --hard HEAD~

# important.txt ПОТЕРЯН навсегда!
# Git его никогда не видел, reflog не поможет
```

### Потеря коммитов

```bash
# Сделали важные коммиты
git log --oneline
# abc123 Important feature
# def456 Another commit
# ...

# Reset --hard
git reset --hard HEAD~5

# Коммиты abc123, def456 и т.д. "потеряны"
# (можно восстановить через reflog в течение ~30 дней)
```

## Восстановление после неудачного reset

### 1. Reflog — ваш спаситель

```bash
# Reflog хранит историю всех изменений HEAD
git reflog
# abc1234 HEAD@{0}: reset: moving to HEAD~3
# def5678 HEAD@{1}: commit: Important commit
# ...

# Вернуться к состоянию до reset
git reset --hard def5678
# или
git reset --hard HEAD@{1}
```

### 2. Пример восстановления

```bash
# Было
git log --oneline
# abc Important feature
# def Some work
# ghi Initial

# Ошибочный reset
git reset --hard HEAD~2

# Теперь
git log --oneline
# ghi Initial

# Паника! Но...
git reflog
# ghi HEAD@{0}: reset: moving to HEAD~2
# abc HEAD@{1}: commit: Important feature
# def HEAD@{2}: commit: Some work
# ghi HEAD@{3}: commit: Initial

# Восстановление
git reset --hard HEAD@{1}
# или
git reset --hard abc

# Ура!
git log --oneline
# abc Important feature
# def Some work
# ghi Initial
```

### 3. FSCK для потерянных объектов

```bash
# Если reflog не помог, можно поискать "болтающиеся" коммиты
git fsck --lost-found

# Показывает dangling commits
# dangling commit abc1234...

# Можно посмотреть их содержимое
git show abc1234
```

### 4. Когда восстановление невозможно

```bash
# 1. Незакоммиченные изменения после --hard — ПОТЕРЯНЫ НАВСЕГДА
# 2. После git gc --prune=now — старые коммиты удалены
# 3. После истечения срока reflog (~30 дней) — коммиты могут быть удалены
```

## Reset vs других команд

### Reset vs Revert

```bash
# Reset — изменяет историю
git reset --hard HEAD~
# Коммит "исчезает" из истории

# Revert — создаёт новый коммит, отменяющий изменения
git revert HEAD
# История сохраняется, добавляется "отменяющий" коммит

# Для публичных веток используйте revert!
```

### Reset vs Checkout

```bash
# Reset перемещает ветку
git reset --hard abc123
# main теперь указывает на abc123

# Checkout перемещает HEAD (без ветки)
git checkout abc123
# HEAD на abc123, main остаётся где был (detached HEAD)
```

### Reset vs Restore

```bash
# Reset для staging area
git reset HEAD file.txt  # Старый способ

# Restore — современная альтернатива
git restore --staged file.txt  # Убрать из staging
git restore file.txt  # Отменить изменения в working directory
```

## Практические сценарии

### Сценарий 1: Исправить последний коммит

```bash
# Забыли добавить файл
git reset --soft HEAD~
git add forgotten_file.txt
git commit -m "Complete commit with all files"
```

### Сценарий 2: Разделить большой коммит

```bash
git reset HEAD~
# Теперь все изменения unstaged

git add -p  # Интерактивно выбираем
git commit -m "Part 1: database changes"

git add -p
git commit -m "Part 2: API changes"

git add .
git commit -m "Part 3: frontend changes"
```

### Сценарий 3: Синхронизация с remote

```bash
git fetch origin
git reset --hard origin/main
# Локальная ветка теперь идентична remote
```

### Сценарий 4: Отмена merge

```bash
# После неудачного merge
git reset --hard ORIG_HEAD
# ORIG_HEAD сохраняется перед "опасными" операциями
```

### Сценарий 5: Быстрый squash

```bash
# Вместо интерактивного rebase
git reset --soft HEAD~5
git commit -m "feat: complete feature"
```

## Best Practices

### 1. Никогда не используйте --hard для публичных веток

```bash
# Плохо
git push origin main
git reset --hard HEAD~
git push --force  # Ломает историю для всех!

# Хорошо — используйте revert
git revert HEAD
git push origin main
```

### 2. Всегда проверяйте reflog перед --hard

```bash
# Сохраните текущее состояние
git reflog | head -5

# Теперь безопасно делать reset
git reset --hard HEAD~3

# Если что-то пошло не так — восстановите
git reset --hard HEAD@{1}
```

### 3. Используйте stash перед reset

```bash
# Сохраните несохранённую работу
git stash

# Безопасный reset
git reset --hard origin/main

# Верните работу
git stash pop
```

### 4. Создайте backup ветку

```bash
# Перед опасными операциями
git branch backup-before-reset

# Делайте reset
git reset --hard HEAD~5

# Если нужно — восстановите
git reset --hard backup-before-reset
```

## Шпаргалка

```bash
# Отменить последний коммит, сохранить изменения staged
git reset --soft HEAD~

# Отменить последний коммит, изменения unstaged
git reset HEAD~

# Полностью удалить последний коммит
git reset --hard HEAD~

# Убрать файл из staging
git reset HEAD -- file.txt

# Сбросить к remote
git reset --hard origin/main

# Восстановить после reset
git reflog
git reset --hard HEAD@{N}
```
