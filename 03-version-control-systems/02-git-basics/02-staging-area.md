# Staging Area (Область подготовки)

## Что такое Staging Area?

**Staging Area** (также называется **Index**) — это промежуточная область между вашим рабочим каталогом и репозиторием Git. Здесь вы собираете изменения, которые войдут в следующий коммит.

Представьте staging area как "черновик" вашего будущего коммита. Вы можете:
- Добавлять изменения частями
- Убирать то, что добавили по ошибке
- Формировать логически связанные коммиты

## Зачем нужна Staging Area?

### 1. Выборочное добавление изменений
Вы изменили 5 файлов, но хотите закоммитить только 2 из них:

```bash
# Добавляем только нужные файлы
git add file1.py file2.py
git commit -m "Fix authentication bug"

# Остальные 3 файла останутся в working directory
```

### 2. Разделение большого изменения на логические коммиты
```bash
# Сначала коммитим изменения в базе данных
git add models/user.py migrations/001_users.sql
git commit -m "Add user model and migration"

# Затем коммитим API эндпоинты
git add api/users.py tests/test_users.py
git commit -m "Add user API endpoints with tests"
```

### 3. Проверка перед коммитом
```bash
# Смотрим, что будет закоммичено
git diff --staged
```

## Основные команды

### git add — добавление в staging area

```bash
# Добавить конкретный файл
git add file.py

# Добавить несколько файлов
git add file1.py file2.py file3.py

# Добавить все файлы в директории
git add src/

# Добавить все изменённые и новые файлы
git add .

# Добавить все файлы определённого типа
git add "*.py"
git add "**/*.js"  # рекурсивно во всех поддиректориях

# Добавить все изменённые файлы (без новых)
git add -u
# или
git add --update

# Добавить все изменения (modified + new + deleted)
git add -A
# или
git add --all
```

### Интерактивное добавление (git add -p)

Самая мощная функция staging area — добавление по частям:

```bash
git add -p file.py
# или
git add --patch file.py
```

Git покажет каждое изменение (hunk) и спросит, что делать:
- `y` — добавить этот hunk в staging
- `n` — не добавлять
- `s` — разделить hunk на меньшие части
- `e` — редактировать hunk вручную
- `q` — выйти
- `?` — показать справку

Пример:
```bash
$ git add -p

diff --git a/app.py b/app.py
@@ -10,6 +10,8 @@ def main():
     print("Starting app")
+    # New feature
+    init_database()
     run_server()
Stage this hunk [y,n,q,a,d,s,e,?]?
```

### git restore --staged — убрать из staging area

```bash
# Убрать файл из staging (изменения останутся в working directory)
git restore --staged file.py

# Убрать все файлы из staging
git restore --staged .

# Старый способ (всё ещё работает)
git reset HEAD file.py
git reset HEAD .
```

### git diff --staged — просмотр staged изменений

```bash
# Показать все изменения, добавленные в staging
git diff --staged
# или (синоним)
git diff --cached

# Показать staged изменения для конкретного файла
git diff --staged file.py
```

## Состояния файла и Staging Area

```
┌─────────────────────────────────────────────────────────────┐
│                    Working Directory                         │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │Untracked │  │Unmodified│  │ Modified │  │ Deleted  │    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘    │
│       │             │             │             │           │
└───────┼─────────────┼─────────────┼─────────────┼───────────┘
        │             │             │             │
        │   git add   │   git add   │   git add   │
        ▼             │             ▼             ▼
┌───────────────────────────────────────────────────────────┐
│                     Staging Area                           │
│                                                            │
│  ┌──────────────────────────────────────────────────┐     │
│  │         Изменения для следующего коммита          │     │
│  └──────────────────────────────────────────────────┘     │
│                                                            │
└───────────────────────────────────────────────────────────┘
                            │
                            │ git commit
                            ▼
┌───────────────────────────────────────────────────────────┐
│                      Repository                            │
│                   (История коммитов)                       │
└───────────────────────────────────────────────────────────┘
```

## Практические примеры

### Пример 1: Разделение изменений на коммиты

```bash
# Вы изменили 3 файла, но это разные задачи
$ git status
modified: auth.py      # исправление бага
modified: config.py    # исправление бага
modified: README.md    # обновление документации

# Коммитим баг-фикс
$ git add auth.py config.py
$ git commit -m "Fix authentication timeout bug"

# Коммитим документацию отдельно
$ git add README.md
$ git commit -m "Update installation instructions"
```

### Пример 2: Частичное добавление файла

```bash
# В одном файле есть и нужные изменения, и отладочный код
$ git add -p server.py

# Добавляем только нужные части
# y - для production кода
# n - для debug print statements
```

### Пример 3: Отмена случайного добавления

```bash
# Случайно добавили файл с паролями
$ git add .
$ git status
# new file: config/secrets.py  # Ой!

# Убираем из staging
$ git restore --staged config/secrets.py

# И добавляем в .gitignore
$ echo "config/secrets.py" >> .gitignore
```

## Best Practices

### 1. Проверяйте staged изменения перед коммитом
```bash
# Всегда смотрите, что собираетесь закоммитить
git diff --staged
```

### 2. Используйте `git add -p` для чистых коммитов
```bash
# Это помогает не добавлять случайный код
git add -p
```

### 3. Не используйте `git add .` бездумно
```bash
# Плохо — добавляет всё подряд
git add .

# Лучше — явно указывать файлы
git add src/feature.py tests/test_feature.py

# Или хотя бы проверить перед добавлением
git status
git add .
git diff --staged
```

### 4. Коммитьте атомарно
Один коммит = одна логическая единица изменений.

### 5. Используйте `git stash` для временных изменений
```bash
# Если нужно переключиться на другую задачу
git stash
# ... работаете над другим
git stash pop  # вернуть изменения
```

## Типичные ошибки

### 1. Добавление чувствительных данных
```bash
# ПЛОХО
git add config.py  # содержит пароли
git commit

# Даже если потом удалить файл, он останется в истории!
# Решение: использовать .gitignore ЗАРАНЕЕ
```

### 2. Коммит без проверки
```bash
# ПЛОХО — не видим, что коммитим
git add .
git commit -m "Quick fix"

# ХОРОШО
git add .
git diff --staged  # проверяем
git commit -m "Fix user validation in signup form"
```

### 3. Путаница между staged и unstaged
```bash
# Файл изменён, добавлен в staging, и изменён снова
$ git status
Changes to be committed:
    modified: app.py       # версия 1 (staged)

Changes not staged for commit:
    modified: app.py       # версия 2 (working directory)

# git commit закоммитит версию 1!
# Нужно снова сделать git add, чтобы закоммитить версию 2
```

## Полезные команды

```bash
# Показать staged файлы (только имена)
git diff --staged --name-only

# Показать статистику staged изменений
git diff --staged --stat

# Добавить изменения и сразу открыть редактор для коммита
git commit -v
# (покажет diff в комментарии редактора)

# Добавить все изменённые tracked файлы и закоммитить
git commit -a -m "Message"
# Но это НЕ добавит новые (untracked) файлы!
```

## Резюме

- **Staging Area** — промежуточная область для формирования коммита
- `git add` — добавляет изменения в staging
- `git add -p` — добавляет изменения по частям (очень полезно!)
- `git restore --staged` — убирает из staging (изменения сохраняются)
- `git diff --staged` — показывает, что будет закоммичено
- Staging area позволяет создавать чистые, атомарные коммиты
- Всегда проверяйте `git diff --staged` перед коммитом
