# История коммитов в Git

## Введение

История коммитов — это журнал всех изменений в репозитории. Git предоставляет мощные инструменты для просмотра, поиска и анализа истории. Понимание этих инструментов критически важно для эффективной работы с проектом.

---

## git log — основная команда

### Базовое использование

```bash
git log

# Вывод:
# commit abc123def456... (HEAD -> main, origin/main)
# Author: John Doe <john@example.com>
# Date:   Mon Dec 25 10:30:00 2024 +0300
#
#     Add user authentication
#
# commit def456ghi789...
# Author: Jane Smith <jane@example.com>
# Date:   Sun Dec 24 15:45:00 2024 +0300
#
#     Initial commit
```

Навигация:
- `Space` / `Page Down` — следующая страница
- `b` / `Page Up` — предыдущая страница
- `q` — выход
- `/pattern` — поиск
- `n` — следующее совпадение

---

## Форматирование вывода

### --oneline (компактный формат)

```bash
git log --oneline

# Вывод:
# abc123d Add user authentication
# def456g Fix login bug
# ghi789j Initial commit
```

Показывает только короткий хэш и первую строку сообщения.

### --graph (визуализация веток)

```bash
git log --graph

# Вывод с ASCII-графикой:
# * commit abc123...
# |   Add feature X
# |
# *   commit def456...
# |\  Merge branch 'feature'
# | |
# | * commit ghi789...
# |     Add feature component
# |
# * commit jkl012...
#     Fix bug in main
```

### Комбинации

```bash
# Компактный граф всех веток
git log --oneline --graph --all

# Вывод:
# * abc123d (HEAD -> main) Add authentication
# *   def456g Merge feature branch
# |\
# | * ghi789j (feature) Add user model
# | * jkl012m Add tests
# |/
# * mno345p Initial commit
```

### --all (все ветки)

```bash
# Показать историю всех веток, не только текущей
git log --all
git log --oneline --all
git log --graph --all
```

### --decorate (ссылки)

```bash
# Показать теги, ветки, HEAD
git log --decorate

# В современных версиях Git включён по умолчанию
# commit abc123... (HEAD -> main, tag: v1.0, origin/main)
```

---

## Фильтрация истории

### По автору (--author)

```bash
# Коммиты конкретного автора
git log --author="John"
git log --author="john@example.com"

# Регулярные выражения
git log --author="John\|Jane"  # John ИЛИ Jane
```

### По дате (--since, --until)

```bash
# С определённой даты
git log --since="2024-01-01"
git log --since="2 weeks ago"
git log --since="yesterday"

# До определённой даты
git log --until="2024-06-01"
git log --until="3 days ago"

# Комбинация
git log --since="2024-01-01" --until="2024-06-30"

# Другие форматы дат
git log --since="2024-12-25 10:00"
git log --after="1 month ago"
git log --before="last Friday"
```

### По сообщению коммита (--grep)

```bash
# Поиск по сообщению
git log --grep="fix"
git log --grep="bug"

# Регистронезависимый поиск
git log --grep="FIX" -i

# Несколько паттернов (OR)
git log --grep="fix" --grep="bug"

# Все паттерны (AND)
git log --grep="fix" --grep="bug" --all-match
```

### По файлу

```bash
# История конкретного файла
git log -- path/to/file.py

# История директории
git log -- src/

# Несколько файлов
git log -- file1.py file2.py
```

### По содержимому изменений (-S, -G)

```bash
# Pickaxe: когда строка была добавлена/удалена
git log -S "function_name"
git log -S "TODO"

# Регулярное выражение в содержимом
git log -G "def \w+\("
```

### Ограничение количества (-n)

```bash
# Последние N коммитов
git log -5
git log -n 5
git log --max-count=5
```

### Комбинирование фильтров

```bash
# Коммиты John за последний месяц с "fix" в сообщении
git log --author="John" --since="1 month ago" --grep="fix"

# Последние 10 коммитов, затрагивающих src/
git log -10 -- src/
```

---

## Просмотр изменений

### git log -p (patch)

```bash
# Показать diff для каждого коммита
git log -p

# Ограничить количество
git log -p -3

# Для конкретного файла
git log -p -- file.py
```

### git log --stat

```bash
# Статистика изменений
git log --stat

# Вывод:
# commit abc123...
# Author: John Doe
# Date:   Mon Dec 25 10:30:00 2024
#
#     Add user authentication
#
#  src/auth.py   | 45 +++++++++++++++++++++++++++
#  tests/test_auth.py | 23 ++++++++++++++
#  2 files changed, 68 insertions(+)
```

### git log --shortstat

```bash
# Только суммарная статистика
git log --shortstat

# commit abc123...
#  2 files changed, 68 insertions(+)
```

### git log --name-only

```bash
# Только имена изменённых файлов
git log --name-only

# src/auth.py
# tests/test_auth.py
```

### git log --name-status

```bash
# Имена файлов со статусом
git log --name-status

# A   src/new_file.py      (Added)
# M   src/modified.py      (Modified)
# D   src/deleted.py       (Deleted)
# R   old_name.py -> new.py (Renamed)
```

---

## git shortlog

Группировка коммитов по авторам:

```bash
git shortlog

# Вывод:
# John Doe (15):
#       Add authentication
#       Fix login bug
#       Update dependencies
#
# Jane Smith (8):
#       Add user model
#       Improve tests
```

### Опции shortlog

```bash
# Только количество коммитов
git shortlog -s
# 15  John Doe
# 8   Jane Smith

# Сортировка по количеству
git shortlog -sn
# 15  John Doe
#  8  Jane Smith

# С email
git shortlog -sne
# 15  John Doe <john@example.com>
#  8  Jane Smith <jane@example.com>
```

---

## Форматирование вывода (--format / --pretty)

### Встроенные форматы

```bash
# Короткий
git log --pretty=short

# Полный
git log --pretty=full

# Ещё более полный
git log --pretty=fuller

# Только хэши
git log --pretty=format:"%H"
```

### Кастомный формат

```bash
git log --pretty=format:"%h - %an, %ar : %s"

# Вывод:
# abc123d - John Doe, 2 days ago : Add authentication
# def456g - Jane Smith, 5 days ago : Fix bug
```

### Плейсхолдеры

| Плейсхолдер | Описание |
|-------------|----------|
| `%H` | Полный хэш коммита |
| `%h` | Короткий хэш |
| `%T` | Хэш дерева |
| `%P` | Хэши родителей |
| `%an` | Имя автора |
| `%ae` | Email автора |
| `%ad` | Дата автора |
| `%ar` | Дата автора (относительная) |
| `%cn` | Имя коммиттера |
| `%ce` | Email коммиттера |
| `%cd` | Дата коммита |
| `%cr` | Дата коммита (относительная) |
| `%s` | Сообщение (первая строка) |
| `%b` | Тело сообщения |
| `%d` | Декорации (ветки, теги) |

### Примеры форматов

```bash
# Формат для changelog
git log --pretty=format:"* %s (%h)" --no-merges

# С цветами
git log --pretty=format:"%C(yellow)%h%C(reset) %C(blue)%an%C(reset) %s"

# JSON-подобный
git log --pretty=format:'{"hash":"%h","author":"%an","date":"%ad","message":"%s"}'

# Для скриптов
git log --pretty=format:"%h|%an|%ad|%s" --date=short
```

### Формат даты (--date)

```bash
git log --date=short        # 2024-12-25
git log --date=relative     # 2 days ago
git log --date=local        # Mon Dec 25 10:30:00 2024
git log --date=iso          # 2024-12-25 10:30:00 +0300
git log --date=format:"%Y-%m-%d %H:%M"
```

---

## Специальные представления

### Просмотр истории диапазона

```bash
# От коммита до коммита
git log abc123..def456

# Коммиты в feature, которых нет в main
git log main..feature

# Коммиты в main, которых нет в feature
git log feature..main

# Коммиты, уникальные для каждой ветки
git log main...feature
```

### Просмотр merge коммитов

```bash
# Только merge коммиты
git log --merges

# Без merge коммитов
git log --no-merges

# Показать первого родителя (основную линию)
git log --first-parent
```

### История одной строки файла

```bash
# Кто изменял строки 10-20 в файле
git log -L 10,20:file.py

# История функции (Git пытается найти её границы)
git log -L :function_name:file.py
```

---

## Другие команды для истории

### git show

```bash
# Показать конкретный коммит
git show abc123

# Показать файл в коммите
git show abc123:path/to/file.py

# Показать изменения в теге
git show v1.0
```

### git blame

```bash
# Кто изменял каждую строку файла
git blame file.py

# С датами
git blame -e file.py

# Определённые строки
git blame -L 10,20 file.py
```

### git reflog

```bash
# История всех действий с HEAD
git reflog

# Вывод:
# abc123d HEAD@{0}: commit: Add feature
# def456g HEAD@{1}: checkout: moving from main to feature
# ghi789j HEAD@{2}: commit: Fix bug
```

---

## Полезные алиасы

```bash
# Добавить в ~/.gitconfig
[alias]
    lg = log --oneline --graph --all --decorate
    hist = log --pretty=format:'%h %ad | %s%d [%an]' --graph --date=short
    last = log -1 HEAD
    today = log --since=midnight --author='Your Name'
    contrib = shortlog -sn
```

---

## Практические примеры

### Найти, когда появился баг

```bash
# Поиск по содержимому
git log -S "buggy_function" --oneline

# Поиск по сообщению
git log --grep="feature X" --oneline

# Bisect для автоматического поиска
git bisect start
git bisect bad HEAD
git bisect good v1.0
# Git бинарным поиском найдёт проблемный коммит
```

### Анализ активности

```bash
# Кто больше всех коммитил
git shortlog -sn

# Активность за месяц
git log --since="1 month ago" --oneline | wc -l

# Изменения по дням
git log --format="%ad" --date=short | sort | uniq -c
```

### Генерация changelog

```bash
# Простой changelog
git log v1.0..v2.0 --pretty=format:"* %s" --no-merges

# С категориями (если используете conventional commits)
git log v1.0..v2.0 --pretty=format:"%s" | grep "^feat:"
```

---

## Резюме команд

| Команда | Описание |
|---------|----------|
| `git log` | Полная история |
| `git log --oneline` | Компактный вывод |
| `git log --graph` | С графом веток |
| `git log --all` | Все ветки |
| `git log -n 5` | Последние 5 коммитов |
| `git log -p` | С diff-ами |
| `git log --stat` | Со статистикой |
| `git log --author="X"` | По автору |
| `git log --since="date"` | С даты |
| `git log --grep="text"` | По сообщению |
| `git log -S "text"` | По содержимому |
| `git log -- file` | История файла |
| `git shortlog -sn` | Статистика по авторам |
| `git show <commit>` | Детали коммита |
| `git blame file` | Авторство строк |
| `git reflog` | Журнал действий |
