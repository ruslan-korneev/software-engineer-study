# Viewing Commit History (Просмотр истории коммитов)

## Введение

Git хранит полную историю всех изменений в проекте. Умение эффективно просматривать и анализировать эту историю — важный навык для разработчика.

## git log — основная команда

### Базовое использование

```bash
# Показать историю коммитов
git log

# Пример вывода:
# commit a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0 (HEAD -> main)
# Author: Ivan Petrov <ivan@example.com>
# Date:   Mon Dec 25 10:30:00 2024 +0300
#
#     Add user authentication module
#
# commit b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0u1
# Author: Ivan Petrov <ivan@example.com>
# Date:   Sun Dec 24 15:20:00 2024 +0300
#
#     Fix database connection bug
```

### Форматирование вывода

```bash
# Однострочный формат
git log --oneline
# a1b2c3d Add user authentication module
# b2c3d4e Fix database connection bug
# c3d4e5f Initial commit

# С графом веток
git log --oneline --graph
# * a1b2c3d (HEAD -> main) Add user authentication
# *   b2c3d4e Merge branch 'feature'
# |\
# | * c3d4e5f Add new feature
# |/
# * d4e5f6g Fix bug

# С графом всех веток
git log --oneline --graph --all

# Кастомный формат
git log --format="%h %an %ar %s"
# a1b2c3d Ivan Petrov 2 hours ago Add user authentication

# Полный формат
git log --format=fuller
```

### Форматные плейсхолдеры

```bash
# Основные плейсхолдеры:
# %H  — полный хэш коммита
# %h  — короткий хэш
# %an — имя автора
# %ae — email автора
# %ar — относительная дата автора (2 days ago)
# %ad — абсолютная дата автора
# %s  — тема (первая строка сообщения)
# %b  — тело сообщения

# Пример красивого формата
git log --format="%C(yellow)%h%C(reset) %C(blue)%an%C(reset) %C(green)%ar%C(reset) %s"

# Сохранить как алиас
git config --global alias.lg "log --format='%C(yellow)%h%C(reset) %C(blue)%an%C(reset) %C(green)%ar%C(reset) %s' --graph"
# Теперь можно использовать: git lg
```

## Фильтрация истории

### По количеству

```bash
# Последние N коммитов
git log -5          # последние 5
git log -n 10       # последние 10
git log --max-count=3
```

### По дате

```bash
# После определённой даты
git log --after="2024-01-01"
git log --since="2024-01-01"

# До определённой даты
git log --before="2024-06-30"
git log --until="2024-06-30"

# За определённый период
git log --after="2024-01-01" --before="2024-06-30"

# Относительные даты
git log --since="2 weeks ago"
git log --since="yesterday"
git log --after="1 month ago"
```

### По автору

```bash
# По имени или email
git log --author="Ivan"
git log --author="ivan@example.com"

# Регулярные выражения
git log --author="Ivan\|Maria"  # Ivan или Maria
```

### По содержимому сообщения

```bash
# Поиск в сообщениях коммитов
git log --grep="bug"
git log --grep="fix" --grep="auth"  # fix ИЛИ auth
git log --grep="fix" --grep="auth" --all-match  # fix И auth

# Регистронезависимый поиск
git log --grep="Bug" -i
```

### По изменениям в коде

```bash
# Коммиты, где добавлена или удалена строка
git log -S "function_name"
# Пример: найти, когда была добавлена функция
git log -S "def authenticate_user"

# Поиск по регулярному выражению в коде
git log -G "def \w+_user"

# Коммиты, изменившие конкретный файл
git log -- path/to/file.py
git log -- "*.py"  # все Python файлы

# Коммиты, изменившие конкретную функцию
git log -L :function_name:file.py
```

### По веткам и диапазонам

```bash
# История конкретной ветки
git log branch_name

# Коммиты в feature, которых нет в main
git log main..feature

# Коммиты в main, которых нет в feature
git log feature..main

# Коммиты в обеих ветках (но не общие)
git log main...feature
```

## Просмотр изменений

### git log с diff

```bash
# Показать изменения в каждом коммите
git log -p
git log --patch

# Ограничить количество
git log -p -2  # последние 2 коммита с diff

# Показать статистику изменений
git log --stat
# commit a1b2c3d
# Author: Ivan Petrov
#
#     Add user authentication
#
#  src/auth.py | 45 +++++++++++++++++++++++++
#  src/api.py  | 12 ++++---
#  2 files changed, 54 insertions(+), 3 deletions(-)

# Краткая статистика
git log --shortstat

# Только имена изменённых файлов
git log --name-only

# Имена файлов со статусом (A/M/D)
git log --name-status
# A   src/auth.py      (Added)
# M   src/api.py       (Modified)
# D   src/old_file.py  (Deleted)
```

## git show — детали коммита

```bash
# Показать последний коммит с изменениями
git show

# Показать конкретный коммит
git show a1b2c3d

# Показать коммит без diff
git show --stat a1b2c3d

# Показать конкретный файл в коммите
git show a1b2c3d:path/to/file.py

# Показать только сообщение
git show -s a1b2c3d
git show --no-patch a1b2c3d
```

## git diff — сравнение версий

```bash
# Изменения между двумя коммитами
git diff a1b2c3d b2c3d4e

# Изменения между коммитом и текущим состоянием
git diff a1b2c3d

# Изменения между ветками
git diff main feature

# Изменения в конкретном файле между коммитами
git diff a1b2c3d b2c3d4e -- file.py

# Статистика изменений
git diff --stat a1b2c3d b2c3d4e

# Только имена изменённых файлов
git diff --name-only a1b2c3d b2c3d4e
```

## git blame — кто изменил строку

```bash
# Показать, кто и когда изменил каждую строку
git blame file.py

# Пример вывода:
# a1b2c3d4 (Ivan Petrov 2024-12-25 10:30:00 +0300  1) def authenticate():
# a1b2c3d4 (Ivan Petrov 2024-12-25 10:30:00 +0300  2)     """Authenticate user"""
# b2c3d4e5 (Maria Ivanova 2024-12-26 09:15:00 +0300  3)     if not user:
# b2c3d4e5 (Maria Ivanova 2024-12-26 09:15:00 +0300  4)         return None

# Показать конкретные строки
git blame -L 10,20 file.py

# Показать только автора и строку
git blame -s file.py

# Игнорировать пробельные изменения
git blame -w file.py

# Показать email вместо имени
git blame -e file.py
```

## git shortlog — сводка по авторам

```bash
# Группировка коммитов по авторам
git shortlog
# Ivan Petrov (15):
#       Add user authentication
#       Fix login bug
#       Update documentation
#
# Maria Ivanova (8):
#       Add new feature
#       Refactor database layer

# Только количество коммитов
git shortlog -s
# 15  Ivan Petrov
#  8  Maria Ivanova

# Сортировка по количеству
git shortlog -sn
# 15  Ivan Petrov
#  8  Maria Ivanova

# С email
git shortlog -sne
```

## git reflog — история всех действий

```bash
# Показать историю перемещений HEAD
git reflog
# a1b2c3d HEAD@{0}: commit: Add user authentication
# b2c3d4e HEAD@{1}: checkout: moving from feature to main
# c3d4e5f HEAD@{2}: commit: Add new feature
# d4e5f6g HEAD@{3}: reset: moving to HEAD~1

# Полезно для восстановления потерянных коммитов!
# Если случайно сделали reset --hard:
git reflog  # найти нужный коммит
git checkout a1b2c3d  # вернуться к нему
```

## Практические сценарии

### Сценарий 1: Найти, когда появился баг

```bash
# 1. Найти коммиты, изменявшие подозрительный файл
git log --oneline -- src/auth.py

# 2. Посмотреть изменения в каждом коммите
git log -p -- src/auth.py

# 3. Найти, кто изменил конкретную строку
git blame src/auth.py

# 4. Использовать git bisect для автоматического поиска
git bisect start
git bisect bad          # текущая версия — плохая
git bisect good v1.0    # версия v1.0 — хорошая
# Git будет предлагать коммиты для проверки
```

### Сценарий 2: Посмотреть свои коммиты за неделю

```bash
git log --author="$(git config user.name)" --since="1 week ago" --oneline
```

### Сценарий 3: Найти удалённый код

```bash
# Найти, когда была удалена функция
git log -S "def deleted_function" --oneline

# Посмотреть коммит
git show a1b2c3d
```

### Сценарий 4: Сравнить версии

```bash
# Что изменилось между релизами
git log v1.0..v2.0 --oneline

# Какие файлы изменились
git diff v1.0 v2.0 --stat

# Полные изменения
git diff v1.0 v2.0
```

## Полезные алиасы

```bash
# Добавить в ~/.gitconfig
git config --global alias.lg "log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"

git config --global alias.last "log -1 HEAD --stat"

git config --global alias.hist "log --pretty=format:'%h %ad | %s%d [%an]' --graph --date=short"

# Использование
git lg        # красивый лог с графом
git last      # последний коммит
git hist      # компактная история
```

## Best Practices

### 1. Используйте алиасы для частых команд
```bash
git config --global alias.lg "log --oneline --graph --all"
```

### 2. Комбинируйте фильтры
```bash
# Конкретный автор, за период, в определённых файлах
git log --author="Ivan" --since="2024-01-01" -- src/
```

### 3. Используйте --follow для переименованных файлов
```bash
# Отслеживать историю даже через переименования
git log --follow -- new_name.py
```

### 4. Изучите git reflog
```bash
# Это спасёт вас при ошибках с reset
git reflog
```

## Типичные ошибки

### 1. Забыли -- перед путём к файлу
```bash
# Неоднозначно — feature это ветка или файл?
git log feature

# Явно указываем, что это файл
git log -- feature
```

### 2. Путаница с .. и ...
```bash
# A..B — коммиты в B, которых нет в A
git log main..feature  # что добавится в main при merge

# A...B — коммиты, уникальные для каждой ветки
git log main...feature  # что отличается между ветками
```

### 3. Поиск в merged истории
```bash
# По умолчанию log показывает линейную историю
# Добавьте --all чтобы видеть все ветки
git log --all --oneline --graph
```

## Резюме

- **`git log`** — основная команда для просмотра истории
- **`--oneline --graph`** — компактный вывод с визуализацией веток
- **Фильтры:** `--author`, `--since`, `--until`, `--grep`, `-S`
- **`git log -p`** — показать изменения в каждом коммите
- **`git show`** — детали конкретного коммита
- **`git blame`** — кто изменил каждую строку файла
- **`git reflog`** — история всех перемещений HEAD (спасение от ошибок)
- **`git shortlog -sn`** — статистика по авторам
- Используйте алиасы для часто используемых комбинаций
