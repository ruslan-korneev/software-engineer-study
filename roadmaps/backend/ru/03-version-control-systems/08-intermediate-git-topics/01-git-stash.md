# Git Stash

[prev: 05-collaborators](../07-collaboration-on-github/05-collaborators.md) | [next: 02-history](./02-history.md)
---

## Что такое Stash и зачем нужен

**Git Stash** — это механизм временного сохранения незавершённых изменений без создания коммита. Stash работает как "черновик", куда можно отложить текущую работу, чтобы переключиться на другую задачу.

### Типичные сценарии использования

1. **Срочный баг-фикс**: Вы работаете над фичей, но приходит срочный баг. Нужно быстро переключиться на другую ветку.
2. **Подтягивание изменений**: Вы хотите сделать `git pull`, но локальные изменения конфликтуют с удалёнными.
3. **Эксперименты**: Хотите попробовать другой подход, но не потерять текущую работу.
4. **Перенос изменений**: Случайно начали работу не в той ветке.

---

## Основные команды

### git stash (сохранение изменений)

```bash
# Сохранить все tracked изменения (staged и unstaged)
git stash

# То же самое, более явно
git stash push

# После выполнения рабочая директория становится "чистой"
git status
# On branch feature
# nothing to commit, working tree clean
```

**Что сохраняется по умолчанию:**
- Изменения в отслеживаемых (tracked) файлах
- Staged изменения (добавленные через git add)

**Что НЕ сохраняется по умолчанию:**
- Untracked файлы (новые файлы, не добавленные в git)
- Ignored файлы

### git stash pop (восстановление и удаление)

```bash
# Восстановить последний stash и удалить его из списка
git stash pop

# Восстановить конкретный stash
git stash pop stash@{2}
```

`pop` = `apply` + `drop` (применяет и удаляет)

### git stash apply (восстановление без удаления)

```bash
# Применить последний stash, но оставить его в списке
git stash apply

# Применить конкретный stash
git stash apply stash@{1}
```

Полезно, когда нужно применить одни и те же изменения в нескольких ветках.

---

## Просмотр stash-ей

### git stash list

```bash
git stash list

# Вывод:
# stash@{0}: WIP on feature: abc1234 Add user model
# stash@{1}: WIP on main: def5678 Initial commit
# stash@{2}: On bugfix: ghi9012 Fix login
```

Формат: `stash@{N}: <тип> on <ветка>: <хэш коммита> <сообщение коммита>`

### git stash show

```bash
# Показать summary изменений в последнем stash
git stash show

# Вывод:
# src/user.py | 15 +++++++++------
# tests/test_user.py | 8 ++++++++
# 2 files changed, 17 insertions(+), 6 deletions(-)

# Показать полный diff
git stash show -p

# Показать конкретный stash
git stash show stash@{2}
git stash show -p stash@{2}
```

---

## Stash с сообщением

По умолчанию stash получает автоматическое сообщение. Для лучшей организации используйте свои описания:

```bash
# Сохранить с понятным описанием
git stash push -m "WIP: Добавление валидации email"
git stash push -m "Эксперимент с новым алгоритмом"

# Список теперь более читаемый
git stash list
# stash@{0}: On feature: Эксперимент с новым алгоритмом
# stash@{1}: On feature: WIP: Добавление валидации email
```

**Best Practice**: Всегда добавляйте осмысленное сообщение, особенно если планируете вернуться к stash позже.

---

## Удаление stash-ей

### git stash drop

```bash
# Удалить последний stash
git stash drop

# Удалить конкретный stash
git stash drop stash@{2}
```

### git stash clear

```bash
# Удалить ВСЕ stash-и (осторожно!)
git stash clear
```

**Важно**: Удалённые stash-и восстановить крайне сложно. Используйте `clear` с осторожностью.

---

## Продвинутые возможности

### Частичный stash (--patch / -p)

Интерактивный режим для выборочного сохранения изменений:

```bash
git stash push -p

# Git покажет каждый hunk (блок изменений) и спросит:
# Stash this hunk [y,n,q,a,d,/,e,?]?
```

Ответы:
- `y` — да, сохранить этот hunk
- `n` — нет, пропустить
- `q` — выйти, не сохранять оставшиеся
- `a` — сохранить этот и все оставшиеся hunks в файле
- `d` — не сохранять этот и все оставшиеся hunks в файле
- `s` — разбить hunk на более мелкие части
- `e` — редактировать hunk вручную

### Stash untracked файлов

```bash
# Включить untracked файлы
git stash push -u
git stash push --include-untracked

# Включить ВСЁ (untracked + ignored)
git stash push -a
git stash push --all
```

Пример:
```bash
# Создали новый файл
touch new_feature.py
git status
# Untracked files: new_feature.py

# Обычный stash НЕ сохранит его
git stash
git status
# Untracked files: new_feature.py  (всё ещё здесь!)

# С флагом -u сохранит
git stash -u
git status
# nothing to commit, working tree clean
```

### Stash конкретных файлов

```bash
# Сохранить только указанные файлы
git stash push -m "Only config changes" config.py settings.py

# Сохранить с pathspec
git stash push -- "src/*.py"
```

### Создание ветки из stash

```bash
# Создать новую ветку и применить stash
git stash branch new-feature-branch

# Из конкретного stash
git stash branch experiment stash@{2}
```

Полезно, когда stash конфликтует с текущим состоянием ветки.

---

## Практические примеры

### Сценарий 1: Срочное переключение

```bash
# Работаем над фичей
git checkout feature/user-profile
# ... вносим изменения ...

# Приходит срочный баг!
git stash push -m "WIP: профиль пользователя"
git checkout main
git checkout -b hotfix/critical-bug
# ... фиксим баг ...
git commit -m "Fix critical bug"
git checkout feature/user-profile
git stash pop
# Продолжаем работу
```

### Сценарий 2: Подтягивание изменений

```bash
# Хотим получить свежие изменения
git pull
# error: Your local changes would be overwritten by merge

# Решение:
git stash
git pull
git stash pop
# Возможен конфликт, который нужно разрешить вручную
```

### Сценарий 3: Работа в неправильной ветке

```bash
# Ой, работали в main вместо feature!
git stash
git checkout feature/my-feature
git stash pop
# Изменения теперь в правильной ветке
```

---

## Best Practices

1. **Не накапливайте stash-и**: Stash — временное решение. Если изменения важны, создайте коммит или ветку.

2. **Используйте описательные сообщения**:
   ```bash
   git stash push -m "WIP: API endpoint для загрузки файлов"
   ```

3. **Регулярно проверяйте список**:
   ```bash
   git stash list
   ```

4. **Предпочитайте apply перед pop**, если не уверены:
   ```bash
   git stash apply  # Можно откатить, если что-то пошло не так
   git stash drop   # Удалить, когда убедились что всё ок
   ```

5. **Не используйте stash для долгосрочного хранения**: Для этого есть ветки и коммиты.

---

## Типичные ошибки

### Потеря untracked файлов

```bash
# Неправильно (новые файлы не сохранятся):
git stash

# Правильно:
git stash -u
```

### Конфликты при pop

```bash
git stash pop
# CONFLICT (content): Merge conflict in file.py

# Решите конфликты вручную, затем:
git add file.py
# Stash НЕ удалился автоматически из-за конфликта
git stash drop  # Удалить вручную после разрешения
```

### Забытые stash-и

```bash
# Проверяйте перед удалением репозитория или ветки
git stash list
```

---

## Полезные алиасы

```bash
# Добавить в ~/.gitconfig
[alias]
    st = stash
    stp = stash pop
    stl = stash list
    sts = stash show -p
    sta = stash apply
```

---

## Резюме команд

| Команда | Описание |
|---------|----------|
| `git stash` | Сохранить tracked изменения |
| `git stash -u` | Сохранить включая untracked |
| `git stash -m "msg"` | Сохранить с сообщением |
| `git stash -p` | Интерактивный частичный stash |
| `git stash list` | Список всех stash-ей |
| `git stash show` | Показать изменения в stash |
| `git stash show -p` | Показать полный diff |
| `git stash pop` | Применить и удалить |
| `git stash apply` | Применить без удаления |
| `git stash drop` | Удалить stash |
| `git stash clear` | Удалить все stash-и |
| `git stash branch <name>` | Создать ветку из stash |

---
[prev: 05-collaborators](../07-collaboration-on-github/05-collaborators.md) | [next: 02-history](./02-history.md)