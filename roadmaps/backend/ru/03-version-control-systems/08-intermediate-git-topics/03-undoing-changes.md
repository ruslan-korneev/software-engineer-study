# Отмена изменений в Git

[prev: 02-history](./02-history.md) | [next: 04-viewing-diffs](./04-viewing-diffs.md)
---

## Введение

Git предоставляет несколько способов отмены изменений на разных стадиях работы. Выбор метода зависит от того, где находятся изменения:
- В рабочей директории (не staged)
- В staging area (staged)
- Уже закоммичены

Понимание различий между методами критически важно для безопасной работы с репозиторием.

---

## Отмена изменений в рабочей директории

### git checkout -- file (старый способ)

```bash
# Отменить изменения в файле (вернуть к последнему коммиту)
git checkout -- file.py

# Отменить изменения во всех файлах
git checkout -- .

# Отменить изменения в директории
git checkout -- src/
```

**Внимание**: Эта команда безвозвратно удаляет незакоммиченные изменения!

### git restore (современный способ, Git 2.23+)

```bash
# Отменить изменения в файле
git restore file.py

# Отменить изменения во всех файлах
git restore .

# Восстановить файл из конкретного коммита
git restore --source=abc123 file.py

# Восстановить из HEAD~2 (2 коммита назад)
git restore --source=HEAD~2 file.py
```

`git restore` — более понятная и безопасная альтернатива `git checkout` для восстановления файлов.

---

## Отмена staged изменений

### git reset HEAD file (старый способ)

```bash
# Убрать файл из staging area (изменения остаются в рабочей директории)
git reset HEAD file.py

# Убрать все файлы из staging
git reset HEAD
```

### git restore --staged (современный способ)

```bash
# Убрать файл из staging area
git restore --staged file.py

# Убрать все файлы из staging
git restore --staged .
```

### Комбинация: unstage + отменить изменения

```bash
# Сначала убрать из staging, потом отменить изменения
git restore --staged file.py
git restore file.py

# Или одной командой
git restore --staged --worktree file.py
# Или короче
git restore -SW file.py
```

---

## git reset — отмена коммитов

`git reset` перемещает указатель HEAD и (опционально) изменяет staging area и рабочую директорию.

### Три режима reset

```
                Working Dir    Staging Area    HEAD
--soft               -              -           ✓
--mixed (default)    -              ✓           ✓
--hard               ✓              ✓           ✓
```

### --soft (мягкий reset)

```bash
git reset --soft HEAD~1
```

- HEAD перемещается на 1 коммит назад
- Staging area **сохраняется**
- Рабочая директория **не изменяется**
- Изменения из отменённого коммита попадают в staging

**Использование**: Исправить последний коммит, объединить несколько коммитов в один.

```bash
# Сценарий: объединить последние 3 коммита
git reset --soft HEAD~3
git commit -m "Combined feature"
```

### --mixed (смешанный reset, по умолчанию)

```bash
git reset HEAD~1
# Или явно
git reset --mixed HEAD~1
```

- HEAD перемещается на 1 коммит назад
- Staging area **очищается**
- Рабочая директория **не изменяется**
- Изменения из отменённого коммита остаются в рабочей директории

**Использование**: Отменить коммит и пересмотреть, что добавлять в staging.

### --hard (жёсткий reset)

```bash
git reset --hard HEAD~1
```

- HEAD перемещается на 1 коммит назад
- Staging area **очищается**
- Рабочая директория **восстанавливается** к состоянию нового HEAD

**ОПАСНО**: Все незакоммиченные изменения будут потеряны!

**Использование**: Полностью вернуться к определённому состоянию.

### Примеры использования reset

```bash
# Отменить последний коммит, сохранить изменения staged
git reset --soft HEAD~1

# Отменить последний коммит, изменения в рабочей директории
git reset HEAD~1

# Полностью вернуться к предыдущему коммиту
git reset --hard HEAD~1

# Вернуться к конкретному коммиту
git reset --hard abc123

# Синхронизироваться с удалённой веткой
git fetch origin
git reset --hard origin/main
```

---

## git revert — безопасная отмена

`git revert` создаёт **новый коммит**, который отменяет изменения указанного коммита.

```bash
# Отменить последний коммит
git revert HEAD

# Отменить конкретный коммит
git revert abc123

# Отменить без автоматического коммита
git revert --no-commit abc123
git revert -n abc123
```

### Отмена нескольких коммитов

```bash
# Отменить диапазон (от старого к новому, не включая первый)
git revert abc123..def456

# Отменить несколько отдельных коммитов
git revert abc123 def456 ghi789

# Отменить последние 3 коммита одним revert-коммитом
git revert --no-commit HEAD~3..HEAD
git commit -m "Revert last 3 commits"
```

### Отмена merge коммита

```bash
# -m 1 указывает, какой родитель считать "основным"
git revert -m 1 <merge-commit-hash>
```

---

## Reset vs Revert: ключевые различия

| Аспект | git reset | git revert |
|--------|-----------|------------|
| История | **Изменяет** (переписывает) | **Сохраняет** (добавляет новый коммит) |
| Безопасность | Опасен для shared веток | Безопасен всегда |
| Использование | Локальные ветки | Публичные ветки |
| Результат | Коммиты "исчезают" | Остаётся след отмены |

### Когда использовать reset

- Локальная ветка, не запушенная
- Нужно "почистить" историю перед push
- Объединение коммитов
- Исправление последнего коммита

### Когда использовать revert

- Ветка уже запушена / shared
- Нужно отменить изменения в main/master
- Важно сохранить историю для аудита
- Отмена конкретного коммита из середины истории

```bash
# Плохо: reset на публичной ветке
git checkout main
git reset --hard HEAD~3  # НЕ ДЕЛАЙТЕ ТАК!
git push --force         # Сломает работу коллег

# Хорошо: revert на публичной ветке
git checkout main
git revert HEAD~3..HEAD  # Создаст revert-коммиты
git push                 # Безопасно
```

---

## Восстановление удалённых коммитов (reflog)

### Что такое reflog

`reflog` (reference log) — это журнал всех изменений HEAD. Даже после `reset --hard` коммиты не удаляются сразу — они остаются в reflog примерно 90 дней.

```bash
git reflog

# Вывод:
# abc123d HEAD@{0}: reset: moving to HEAD~3
# def456g HEAD@{1}: commit: Add feature
# ghi789j HEAD@{2}: commit: Fix bug
# jkl012m HEAD@{3}: commit: Initial work
```

### Восстановление после reset --hard

```bash
# Ой, случайно сделали reset --hard!
git reset --hard HEAD~5

# Найти потерянный коммит
git reflog

# Восстановить
git reset --hard HEAD@{1}
# Или по хэшу
git reset --hard def456g
```

### Восстановление удалённой ветки

```bash
# Удалили ветку
git branch -D feature

# Найти последний коммит ветки
git reflog | grep feature
# Или
git reflog --all

# Восстановить ветку
git checkout -b feature abc123
```

### Важные команды reflog

```bash
# Показать reflog
git reflog

# Reflog для конкретной ветки
git reflog show feature

# С датами
git reflog --date=iso

# Удалить старые записи (осторожно!)
git reflog expire --expire=now --all
git gc --prune=now
```

---

## Практические сценарии

### Сценарий 1: Отменить последний коммит, сохранив изменения

```bash
# Мягкий reset
git reset --soft HEAD~1

# Теперь можно исправить и закоммитить заново
git commit -m "Better commit message"
```

### Сценарий 2: Исправить сообщение последнего коммита

```bash
git commit --amend -m "New message"
```

### Сценарий 3: Добавить файл в последний коммит

```bash
git add forgotten_file.py
git commit --amend --no-edit
```

### Сценарий 4: Полностью начать сначала

```bash
# Отменить ВСЕ незакоммиченные изменения
git reset --hard HEAD

# Удалить untracked файлы
git clean -fd
```

### Сценарий 5: Отменить изменения в одном файле из коммита

```bash
# Восстановить файл из предыдущего коммита
git checkout HEAD~1 -- file.py
# Или
git restore --source=HEAD~1 file.py

# Закоммитить это изменение
git commit -m "Revert changes to file.py"
```

### Сценарий 6: Синхронизация с remote после force push коллеги

```bash
git fetch origin
git reset --hard origin/main
```

---

## Безопасные практики

### 1. Перед опасными операциями создайте backup

```bash
# Создать backup-ветку
git branch backup-before-reset

# Теперь можно экспериментировать
git reset --hard HEAD~5

# Если что-то пошло не так
git reset --hard backup-before-reset
```

### 2. Используйте stash перед reset

```bash
git stash
git reset --hard origin/main
git stash pop  # если нужно вернуть изменения
```

### 3. Проверяйте перед commit --amend

```bash
# Убедитесь, что коммит НЕ запушен
git log origin/main..HEAD
# Если пусто — коммит уже на сервере, amend опасен
```

### 4. Используйте revert для публичных веток

```bash
# Всегда безопасно
git revert HEAD
git push
```

---

## Резюме команд

| Задача | Команда |
|--------|---------|
| Отменить изменения в файле | `git restore file.py` |
| Убрать файл из staging | `git restore --staged file.py` |
| Отменить последний коммит (сохранить staged) | `git reset --soft HEAD~1` |
| Отменить последний коммит (сохранить в working dir) | `git reset HEAD~1` |
| Полностью отменить последний коммит | `git reset --hard HEAD~1` |
| Безопасно отменить коммит | `git revert <commit>` |
| Исправить сообщение коммита | `git commit --amend -m "new msg"` |
| Добавить в последний коммит | `git add file && git commit --amend` |
| Восстановить после reset | `git reflog` + `git reset --hard HEAD@{n}` |
| Отменить все незакоммиченные | `git reset --hard HEAD && git clean -fd` |

---
[prev: 02-history](./02-history.md) | [next: 04-viewing-diffs](./04-viewing-diffs.md)