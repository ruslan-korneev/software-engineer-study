# What and Why - Что такое Git Hooks

[prev: 06-tagging](../06-tagging.md) | [next: 02-client-vs-server](./02-client-vs-server.md)
---

## Концепция хуков

**Git Hooks** (хуки) — это скрипты, которые Git автоматически выполняет до или после определённых событий: коммит, пуш, мердж и т.д. Это механизм для автоматизации и кастомизации рабочего процесса Git.

Хуки позволяют "подцепиться" (hook into) к жизненному циклу Git и выполнить произвольный код в нужный момент.

```
┌─────────────────────────────────────────────────────────────┐
│                    Git Workflow с хуками                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   git commit                                                 │
│       │                                                      │
│       ▼                                                      │
│   ┌──────────────┐                                          │
│   │  pre-commit  │ ──► Запуск линтера, форматирование       │
│   └──────────────┘                                          │
│       │                                                      │
│       ▼                                                      │
│   ┌──────────────┐                                          │
│   │ prepare-msg  │ ──► Подготовка шаблона сообщения         │
│   └──────────────┘                                          │
│       │                                                      │
│       ▼                                                      │
│   ┌──────────────┐                                          │
│   │  commit-msg  │ ──► Валидация сообщения коммита          │
│   └──────────────┘                                          │
│       │                                                      │
│       ▼                                                      │
│   ┌──────────────┐                                          │
│   │ post-commit  │ ──► Уведомления, логирование             │
│   └──────────────┘                                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Зачем нужны хуки

### 1. Автоматизация проверок качества кода

```bash
# pre-commit: запуск линтера перед коммитом
#!/bin/bash
npm run lint
if [ $? -ne 0 ]; then
    echo "Lint failed. Please fix errors before committing."
    exit 1
fi
```

### 2. Стандартизация процессов в команде

```bash
# commit-msg: проверка формата сообщения коммита
#!/bin/bash
commit_regex='^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .{1,50}'
if ! grep -qE "$commit_regex" "$1"; then
    echo "Invalid commit message format!"
    echo "Format: type(scope): description"
    exit 1
fi
```

### 3. Интеграция с внешними системами

```bash
# post-commit: отправка уведомления в Slack
#!/bin/bash
commit_msg=$(git log -1 --pretty=%B)
curl -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"New commit: $commit_msg\"}" \
    $SLACK_WEBHOOK_URL
```

### 4. Защита от ошибок

```bash
# pre-push: запуск тестов перед пушем
#!/bin/bash
npm test
if [ $? -ne 0 ]; then
    echo "Tests failed. Push rejected."
    exit 1
fi
```

## Где хранятся хуки

Хуки хранятся в директории `.git/hooks/` вашего репозитория:

```bash
# Посмотреть содержимое директории хуков
ls -la .git/hooks/

# Пример содержимого (sample-файлы создаются автоматически)
applypatch-msg.sample
commit-msg.sample
fsmonitor-watchman.sample
post-update.sample
pre-applypatch.sample
pre-commit.sample
pre-merge-commit.sample
pre-push.sample
pre-rebase.sample
pre-receive.sample
prepare-commit-msg.sample
push-to-checkout.sample
update.sample
```

### Важные особенности хранения

```bash
# .git/hooks НЕ отслеживается Git!
# Поэтому хуки нужно распространять отдельно

# Способ 1: Хранить скрипты в репозитории и копировать
# Создаём директорию для хуков в репозитории
mkdir -p scripts/git-hooks

# Устанавливаем хуки
cp scripts/git-hooks/* .git/hooks/
chmod +x .git/hooks/*

# Способ 2: Использовать git config для указания другой директории
git config core.hooksPath scripts/git-hooks

# Способ 3: Использовать специальные инструменты (husky, pre-commit)
```

## Как активировать хук

### Шаг 1: Создать файл хука

```bash
# Создаём pre-commit хук
nano .git/hooks/pre-commit
```

### Шаг 2: Написать скрипт

```bash
#!/bin/bash
# Пример простого pre-commit хука

echo "Running pre-commit checks..."

# Проверяем синтаксис Python файлов
python_files=$(git diff --cached --name-only --diff-filter=ACM | grep '\.py$')

if [ -n "$python_files" ]; then
    echo "Checking Python files..."
    python -m py_compile $python_files
    if [ $? -ne 0 ]; then
        echo "Python syntax check failed!"
        exit 1
    fi
fi

echo "Pre-commit checks passed!"
exit 0
```

### Шаг 3: Сделать файл исполняемым

```bash
# ОБЯЗАТЕЛЬНО! Без этого хук не запустится
chmod +x .git/hooks/pre-commit
```

### Шаг 4: Проверить работу

```bash
# Хук запустится автоматически при коммите
git add .
git commit -m "Test commit"

# Для обхода хука (не рекомендуется!)
git commit --no-verify -m "Skip hooks"
# или
git commit -n -m "Skip hooks"
```

## Примеры использования

### Пример 1: Автоформатирование кода

```bash
#!/bin/bash
# .git/hooks/pre-commit - автоформатирование Python кода

# Получаем список изменённых .py файлов
staged_files=$(git diff --cached --name-only --diff-filter=ACM | grep '\.py$')

if [ -z "$staged_files" ]; then
    exit 0
fi

echo "Formatting Python files with Black..."

# Форматируем файлы
black $staged_files

# Добавляем отформатированные файлы обратно в staging
git add $staged_files

echo "Files formatted and staged."
exit 0
```

### Пример 2: Проверка размера файлов

```bash
#!/bin/bash
# .git/hooks/pre-commit - запрет больших файлов

max_size=5242880  # 5MB в байтах

for file in $(git diff --cached --name-only --diff-filter=ACM); do
    if [ -f "$file" ]; then
        size=$(wc -c < "$file")
        if [ $size -gt $max_size ]; then
            echo "Error: $file is larger than 5MB ($size bytes)"
            exit 1
        fi
    fi
done

exit 0
```

### Пример 3: Добавление номера задачи в коммит

```bash
#!/bin/bash
# .git/hooks/prepare-commit-msg

# Извлекаем номер задачи из имени ветки (например: feature/TASK-123-add-login)
branch_name=$(git symbolic-ref --short HEAD)
task_number=$(echo "$branch_name" | grep -oE '[A-Z]+-[0-9]+')

if [ -n "$task_number" ]; then
    # Добавляем номер задачи в начало сообщения, если его там нет
    if ! grep -q "$task_number" "$1"; then
        sed -i.bak "1s/^/[$task_number] /" "$1"
    fi
fi
```

### Пример 4: Защита main ветки

```bash
#!/bin/bash
# .git/hooks/pre-commit - запрет прямых коммитов в main

branch=$(git symbolic-ref --short HEAD)

if [ "$branch" = "main" ] || [ "$branch" = "master" ]; then
    echo "Error: Direct commits to $branch are not allowed!"
    echo "Please create a feature branch and submit a pull request."
    exit 1
fi

exit 0
```

## Exit коды хуков

```bash
# exit 0 - успех, Git продолжает операцию
exit 0

# exit 1 (или любой ненулевой код) - ошибка, Git отменяет операцию
exit 1

# Это работает для pre-* хуков (pre-commit, pre-push и т.д.)
# post-* хуки не могут отменить операцию (она уже выполнена)
```

## Best Practices

### 1. Делайте хуки быстрыми

```bash
# Плохо - проверяем ВСЕ файлы
npm run lint

# Хорошо - проверяем только изменённые файлы
staged_files=$(git diff --cached --name-only --diff-filter=ACM | grep '\.js$')
npx eslint $staged_files
```

### 2. Выводите понятные сообщения об ошибках

```bash
if [ $? -ne 0 ]; then
    echo ""
    echo "╔════════════════════════════════════════════╗"
    echo "║  COMMIT REJECTED: Lint errors found        ║"
    echo "║                                            ║"
    echo "║  Run 'npm run lint:fix' to auto-fix       ║"
    echo "╚════════════════════════════════════════════╝"
    echo ""
    exit 1
fi
```

### 3. Делитесь хуками с командой

```bash
# Создайте setup скрипт в репозитории
# scripts/setup-hooks.sh

#!/bin/bash
echo "Setting up Git hooks..."
git config core.hooksPath scripts/git-hooks
echo "Done! Git hooks configured."
```

### 4. Документируйте хуки

```bash
#!/bin/bash
# =============================================================================
# pre-commit hook
# =============================================================================
# Purpose: Ensures code quality before commits
#
# Checks:
#   1. Runs ESLint on staged JavaScript files
#   2. Runs Prettier for code formatting
#   3. Blocks commits with TODO/FIXME comments
#
# To skip: git commit --no-verify (not recommended)
# =============================================================================
```

## Полезные команды

```bash
# Просмотр всех доступных хуков
ls .git/hooks/*.sample

# Создание хука из шаблона
cp .git/hooks/pre-commit.sample .git/hooks/pre-commit

# Временное отключение хуков
git commit --no-verify -m "message"

# Изменение директории хуков
git config core.hooksPath ./my-hooks

# Сброс директории хуков на default
git config --unset core.hooksPath
```

---
[prev: 06-tagging](../06-tagging.md) | [next: 02-client-vs-server](./02-client-vs-server.md)