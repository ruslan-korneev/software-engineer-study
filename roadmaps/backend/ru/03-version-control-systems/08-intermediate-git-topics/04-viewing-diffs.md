# Просмотр различий (Diff) в Git

## Введение

`git diff` — одна из самых часто используемых команд Git. Она показывает различия между разными состояниями файлов: рабочей директорией, staging area, коммитами и ветками.

Понимание diff критически важно для:
- Code review
- Отладки
- Понимания изменений перед коммитом
- Анализа истории

---

## Базовые концепции

### Три области Git

```
Working Directory  →  Staging Area  →  Repository (commits)
      (WD)              (Index)            (HEAD)
```

`git diff` сравнивает эти области между собой.

### Формат вывода diff

```diff
diff --git a/file.py b/file.py
index abc123..def456 100644
--- a/file.py
+++ b/file.py
@@ -10,7 +10,8 @@ def function():
     existing line
-    removed line
+    added line
+    another added line
     unchanged line
```

Расшифровка:
- `---` — старая версия файла
- `+++` — новая версия файла
- `@@` — контекст (номера строк)
- `-` (красный) — удалённая строка
- `+` (зелёный) — добавленная строка
- Без префикса — контекстная строка (без изменений)

---

## Основные сценарии diff

### 1. git diff (Working Directory vs Staging Area)

Показывает изменения, которые **ещё НЕ добавлены** в staging:

```bash
# Все unstaged изменения
git diff

# Конкретный файл
git diff file.py

# Конкретная директория
git diff src/
```

```bash
# Пример сценария
echo "new content" >> file.py
git diff  # Покажет изменение

git add file.py
git diff  # Пусто! (изменения уже staged)
```

### 2. git diff --staged (Staging Area vs HEAD)

Показывает изменения, которые **будут закоммичены**:

```bash
# Все staged изменения
git diff --staged
# Или синоним
git diff --cached

# Конкретный файл
git diff --staged file.py
```

```bash
# Пример сценария
git add file.py
git diff         # Пусто
git diff --staged  # Покажет изменения, готовые к коммиту
```

### 3. git diff HEAD (Working Directory vs HEAD)

Показывает **все изменения** относительно последнего коммита (и staged, и unstaged):

```bash
git diff HEAD
git diff HEAD file.py
```

---

## Сравнение коммитов

### Сравнение с конкретным коммитом

```bash
# Текущее состояние vs коммит
git diff abc123

# Между двумя коммитами
git diff abc123 def456

# Более явная запись
git diff abc123..def456
```

### Сравнение с предыдущими коммитами

```bash
# С предыдущим коммитом
git diff HEAD~1
git diff HEAD^

# С позапрошлым
git diff HEAD~2

# Между двумя относительными коммитами
git diff HEAD~3..HEAD~1
```

### Что изменилось в конкретном коммите

```bash
# Показать diff конкретного коммита
git show abc123

# Только diff без метаданных
git diff abc123^..abc123
# Или
git diff abc123~1..abc123
```

---

## Сравнение веток

### Базовое сравнение веток

```bash
# Различия между ветками
git diff main feature

# Что добавлено в feature относительно main
git diff main..feature

# Что изменилось с момента ответвления feature от main
git diff main...feature  # Три точки!
```

### Разница между `..` и `...`

```bash
# Две точки: прямое сравнение
git diff main..feature
# Сравнивает текущее состояние main и текущее состояние feature

# Три точки: с общего предка
git diff main...feature
# Сравнивает общего предка main и feature с текущим состоянием feature
# Показывает только изменения, сделанные в feature
```

```
       A---B---C  (feature)
      /
 D---E---F---G  (main)

git diff main..feature   → сравнивает G и C
git diff main...feature  → сравнивает E и C (изменения только в feature)
```

### Практические примеры с ветками

```bash
# Что я изменил в своей feature-ветке?
git diff main...HEAD

# Что изменилось в main, пока я работал над feature?
git diff HEAD...main

# Подготовка к merge: что придёт из feature в main
git checkout main
git diff HEAD...feature
```

---

## Опции форматирования

### --stat (статистика)

```bash
git diff --stat

# Вывод:
# src/auth.py       | 15 +++++++++------
# src/user.py       | 23 +++++++++++++++++++++++
# tests/test_auth.py|  8 ++------
# 3 files changed, 34 insertions(+), 12 deletions(-)
```

### --shortstat (краткая статистика)

```bash
git diff --shortstat
# 3 files changed, 34 insertions(+), 12 deletions(-)
```

### --name-only (только имена файлов)

```bash
git diff --name-only
# src/auth.py
# src/user.py
# tests/test_auth.py
```

### --name-status (имена со статусом)

```bash
git diff --name-status
# M   src/auth.py      (Modified)
# A   src/user.py      (Added)
# D   old_file.py      (Deleted)
```

### --numstat (машиночитаемая статистика)

```bash
git diff --numstat
# 15    6    src/auth.py
# 23    0    src/user.py
# 2     8    tests/test_auth.py
# добавлено  удалено  файл
```

---

## Продвинутые опции

### Контекст (-U, -C)

```bash
# Показать 10 строк контекста (по умолчанию 3)
git diff -U10
git diff --unified=10

# Без контекста
git diff -U0
```

### Игнорирование whitespace

```bash
# Игнорировать изменения в пробелах
git diff -w
git diff --ignore-all-space

# Игнорировать изменения в конце строк
git diff --ignore-space-at-eol

# Игнорировать изменения количества пробелов
git diff -b
git diff --ignore-space-change
```

### Word diff

```bash
# Показать изменения по словам, а не по строкам
git diff --word-diff

# Вывод:
# This is [-old-]{+new+} text.

# Только изменённые слова с цветом
git diff --word-diff=color
```

### Показать только изменённые файлы определённого типа

```bash
# Только Python файлы
git diff -- "*.py"

# Исключить тесты
git diff -- . ":!tests/"
git diff -- . ":(exclude)tests/"
```

### Поиск изменений по содержимому

```bash
# Показать diff только для изменений, содержащих "TODO"
git diff -S "TODO"

# С регулярным выражением
git diff -G "def \w+\("
```

---

## Инструменты для визуального diff

### Встроенные средства Git

```bash
# Использовать настроенный difftool
git difftool

# Конкретный инструмент
git difftool --tool=vimdiff
git difftool -t meld
```

### Настройка difftool

```bash
# В ~/.gitconfig
[diff]
    tool = vimdiff

[difftool]
    prompt = false

[difftool "vscode"]
    cmd = code --wait --diff $LOCAL $REMOTE

[difftool "meld"]
    cmd = meld "$LOCAL" "$REMOTE"
```

### Популярные инструменты

1. **vimdiff** — встроен в vim
   ```bash
   git difftool -t vimdiff
   ```

2. **meld** — графический инструмент для Linux
   ```bash
   sudo apt install meld  # Ubuntu/Debian
   git difftool -t meld
   ```

3. **VS Code**
   ```bash
   # Настройка
   git config --global diff.tool vscode
   git config --global difftool.vscode.cmd 'code --wait --diff $LOCAL $REMOTE'

   # Использование
   git difftool
   ```

4. **Beyond Compare** — мощный платный инструмент
   ```bash
   git config --global diff.tool bc
   git config --global difftool.bc.path "C:/Program Files/Beyond Compare/BComp.exe"
   ```

5. **Kaleidoscope** (macOS)
   ```bash
   git config --global diff.tool kaleidoscope
   ```

### Использование difftool

```bash
# Просмотр всех изменений по файлам
git difftool

# Staged изменения
git difftool --staged

# Между коммитами
git difftool abc123 def456

# Между ветками
git difftool main feature

# Конкретный файл
git difftool HEAD~1 -- file.py
```

---

## Практические примеры

### Проверка перед коммитом

```bash
# Что я добавил в staging?
git diff --staged

# Что ещё не добавлено?
git diff

# Всё вместе относительно HEAD
git diff HEAD
```

### Code Review изменений в PR

```bash
# Какие файлы изменены в feature относительно main
git diff main...feature --name-only

# Статистика изменений
git diff main...feature --stat

# Полный diff
git diff main...feature
```

### Анализ истории файла

```bash
# Как изменился файл за последние 5 коммитов
git diff HEAD~5 -- file.py

# Изменения в файле между тегами
git diff v1.0..v2.0 -- file.py
```

### Экспорт diff в файл

```bash
# Создать patch-файл
git diff > changes.patch
git diff --staged > staged_changes.patch

# Применить patch
git apply changes.patch
```

### Сравнение файла в разных ветках

```bash
# Как выглядит файл в другой ветке
git show feature:path/to/file.py

# Diff файла между ветками
git diff main:file.py feature:file.py
```

---

## Полезные алиасы

```bash
# В ~/.gitconfig
[alias]
    d = diff
    ds = diff --staged
    dc = diff --cached
    dw = diff --word-diff
    dstat = diff --stat

    # Diff с последним коммитом
    dlast = diff HEAD~1

    # Изменения в ветке относительно main
    dbranch = diff main...HEAD
```

---

## Советы и best practices

### 1. Всегда проверяйте diff перед коммитом

```bash
git diff --staged
# Убедитесь, что коммитите то, что нужно
git commit
```

### 2. Используйте --stat для overview

```bash
# Сначала общая картина
git diff --stat

# Потом детали по нужным файлам
git diff -- specific_file.py
```

### 3. Игнорируйте whitespace при review

```bash
# Если много изменений в форматировании
git diff -w
```

### 4. Используйте word-diff для текстовых файлов

```bash
git diff --word-diff README.md
```

### 5. Создавайте патчи для шаринга изменений

```bash
# Создать патч
git diff > my_changes.patch

# Отправить коллеге, он применит
git apply my_changes.patch
```

---

## Резюме команд

| Команда | Описание |
|---------|----------|
| `git diff` | Working Dir vs Staged |
| `git diff --staged` | Staged vs HEAD |
| `git diff HEAD` | Working Dir vs HEAD |
| `git diff <commit>` | Working Dir vs commit |
| `git diff A B` | Commit A vs Commit B |
| `git diff A..B` | То же, что A B |
| `git diff A...B` | Общий предок A,B vs B |
| `git diff --stat` | Статистика изменений |
| `git diff --name-only` | Только имена файлов |
| `git diff -w` | Игнорировать пробелы |
| `git diff --word-diff` | Diff по словам |
| `git difftool` | Визуальный diff |
| `git diff -- file` | Diff конкретного файла |
