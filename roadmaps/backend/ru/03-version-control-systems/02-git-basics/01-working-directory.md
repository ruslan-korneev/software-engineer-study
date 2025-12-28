# Working Directory (Рабочий каталог)

[prev: 05-repository-initialization](../01-learn-the-basics/05-repository-initialization.md) | [next: 02-staging-area](./02-staging-area.md)
---

## Что такое Working Directory?

**Working Directory** (рабочий каталог) — это директория на вашем компьютере, где находятся все файлы проекта, с которыми вы непосредственно работаете. Это "рабочая копия" вашего репозитория.

В Git существует три основных области:
1. **Working Directory** — где вы редактируете файлы
2. **Staging Area** (Index) — промежуточная область для подготовки коммита
3. **Repository** (.git) — где Git хранит всю историю изменений

## Состояния файлов в Working Directory

Файлы в рабочем каталоге могут находиться в одном из следующих состояний:

### 1. Untracked (Неотслеживаемые)
Файлы, о которых Git ещё не знает. Они только что созданы и никогда не добавлялись в Git.

```bash
# Создаём новый файл
touch new_file.txt

# Git покажет его как untracked
git status
# On branch main
# Untracked files:
#   new_file.txt
```

### 2. Tracked (Отслеживаемые)
Файлы, которые Git отслеживает. Они могут быть:

- **Unmodified** — файл не изменялся с последнего коммита
- **Modified** — файл изменён, но изменения ещё не добавлены в staging area
- **Staged** — файл изменён и добавлен в staging area для следующего коммита

```bash
# Файл отслеживается и был изменён
git status
# Changes not staged for commit:
#   modified: app.py
```

## Основные команды для работы с Working Directory

### git status — проверка состояния

```bash
# Полный статус
git status

# Краткий формат
git status -s
# или
git status --short

# Пример вывода краткого формата:
# M  file1.txt    # Изменён и добавлен в staging
#  M file2.txt    # Изменён, но не добавлен в staging
# MM file3.txt    # Изменён, добавлен, и снова изменён
# ?? file4.txt    # Untracked файл
# A  file5.txt    # Новый файл добавлен в staging
# D  file6.txt    # Файл удалён
```

### git diff — просмотр изменений

```bash
# Показать изменения в working directory (не staged)
git diff

# Показать изменения конкретного файла
git diff path/to/file.py

# Показать изменения между working directory и последним коммитом
git diff HEAD

# Показать только имена изменённых файлов
git diff --name-only
```

### git checkout / git restore — отмена изменений

```bash
# Современный способ (Git 2.23+)
# Отменить изменения в файле (вернуть к состоянию последнего коммита)
git restore file.txt

# Отменить изменения во всех файлах
git restore .

# Старый способ (всё ещё работает)
git checkout -- file.txt
git checkout -- .
```

### git clean — удаление untracked файлов

```bash
# Показать, что будет удалено (dry run)
git clean -n

# Удалить untracked файлы
git clean -f

# Удалить untracked файлы и директории
git clean -fd

# Удалить untracked файлы, включая игнорируемые
git clean -fx
```

## Практический пример рабочего процесса

```bash
# 1. Проверяем текущее состояние
git status

# 2. Смотрим, что изменилось в файлах
git diff

# 3. Если изменения нужны — добавляем в staging
git add modified_file.py

# 4. Если изменения не нужны — отменяем
git restore unwanted_changes.py

# 5. Если нужно удалить новые ненужные файлы
git clean -n  # сначала проверяем
git clean -f  # затем удаляем
```

## Best Practices

### 1. Регулярно проверяйте статус
```bash
# Делайте это перед каждым коммитом
git status
```

### 2. Используйте .gitignore
Настройте `.gitignore` чтобы временные файлы не засоряли вывод `git status`.

### 3. Коммитьте атомарно
Группируйте связанные изменения в один коммит. Не смешивайте разные задачи.

### 4. Не храните секреты в working directory
Используйте переменные окружения или специальные файлы конфигурации, добавленные в `.gitignore`.

## Типичные ошибки

### 1. Потеря изменений при checkout/restore
```bash
# ОСТОРОЖНО! Эта команда безвозвратно удалит ваши изменения
git restore important_file.py

# Лучше сначала сохранить изменения
git stash
# Потом можно вернуть
git stash pop
```

### 2. Случайное удаление файлов через git clean
```bash
# ВСЕГДА сначала проверяйте с -n
git clean -n
# И только потом удаляйте
git clean -f
```

### 3. Работа с неправильной веткой
```bash
# Всегда проверяйте, в какой ветке находитесь
git branch
# или
git status  # показывает текущую ветку
```

## Связь с другими областями Git

```
Working Directory  →  git add  →  Staging Area  →  git commit  →  Repository
        ↑                              ↑                              ↑
    Ваши файлы              Подготовка к коммиту          История изменений

        ←  git restore  ←          ←  git restore --staged  ←
```

## Полезные алиасы

```bash
# Добавить в ~/.gitconfig или через git config
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.df diff

# Теперь можно использовать
git st    # вместо git status
git df    # вместо git diff
```

## Резюме

- **Working Directory** — это ваша рабочая папка с файлами проекта
- Файлы могут быть **untracked**, **unmodified**, **modified** или **staged**
- `git status` — главная команда для понимания текущего состояния
- `git diff` — показывает конкретные изменения в файлах
- `git restore` — отменяет изменения (осторожно, данные теряются!)
- `git clean` — удаляет неотслеживаемые файлы

---
[prev: 05-repository-initialization](../01-learn-the-basics/05-repository-initialization.md) | [next: 02-staging-area](./02-staging-area.md)