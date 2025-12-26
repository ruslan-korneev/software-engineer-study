# Common Hooks - Распространённые хуки

## Обзор популярных хуков

В реальных проектах чаще всего используются три хука:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Most Used Git Hooks                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   pre-commit ──────► Качество кода (lint, format, secrets)     │
│                                                                  │
│   commit-msg ──────► Формат сообщения (conventional commits)   │
│                                                                  │
│   pre-push ────────► Тесты перед отправкой                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## pre-commit: Линтинг, форматирование, тесты

### Базовый пример

```bash
#!/bin/bash
# .git/hooks/pre-commit

set -e  # Остановка при первой ошибке

echo "🔍 Running pre-commit checks..."

# Получаем список staged файлов
STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACMR)

if [ -z "$STAGED_FILES" ]; then
    echo "No staged files to check"
    exit 0
fi

echo "Checking files: $STAGED_FILES"
```

### Линтинг JavaScript/TypeScript

```bash
#!/bin/bash
# .git/hooks/pre-commit - ESLint

# Получаем только JS/TS файлы
JS_FILES=$(git diff --cached --name-only --diff-filter=ACMR | grep -E '\.(js|jsx|ts|tsx)$' || true)

if [ -n "$JS_FILES" ]; then
    echo "Running ESLint..."

    # Проверяем только staged файлы
    npx eslint $JS_FILES --max-warnings=0

    if [ $? -ne 0 ]; then
        echo ""
        echo "❌ ESLint found errors. Please fix them before committing."
        echo "   Run 'npm run lint:fix' to auto-fix some issues."
        exit 1
    fi

    echo "✅ ESLint passed"
fi
```

### Линтинг Python

```bash
#!/bin/bash
# .git/hooks/pre-commit - Python linting

PYTHON_FILES=$(git diff --cached --name-only --diff-filter=ACMR | grep '\.py$' || true)

if [ -n "$PYTHON_FILES" ]; then
    echo "Running Python checks..."

    # Flake8 - стиль кода
    echo "  - flake8..."
    flake8 $PYTHON_FILES

    # MyPy - проверка типов
    echo "  - mypy..."
    mypy $PYTHON_FILES --ignore-missing-imports

    # Black - проверка форматирования (без изменений)
    echo "  - black (check)..."
    black --check $PYTHON_FILES

    if [ $? -ne 0 ]; then
        echo "❌ Python checks failed"
        exit 1
    fi

    echo "✅ Python checks passed"
fi
```

### Автоформатирование

```bash
#!/bin/bash
# .git/hooks/pre-commit - Auto-format and re-stage

# JavaScript/TypeScript с Prettier
JS_FILES=$(git diff --cached --name-only --diff-filter=ACMR | grep -E '\.(js|jsx|ts|tsx|json|css|md)$' || true)

if [ -n "$JS_FILES" ]; then
    echo "Formatting files with Prettier..."

    # Форматируем файлы
    npx prettier --write $JS_FILES

    # Добавляем отформатированные файлы обратно в staging
    git add $JS_FILES

    echo "✅ Files formatted and re-staged"
fi

# Python с Black
PYTHON_FILES=$(git diff --cached --name-only --diff-filter=ACMR | grep '\.py$' || true)

if [ -n "$PYTHON_FILES" ]; then
    echo "Formatting Python files with Black..."

    black $PYTHON_FILES

    # isort для импортов
    isort $PYTHON_FILES

    git add $PYTHON_FILES

    echo "✅ Python files formatted"
fi
```

### Проверка на секреты

```bash
#!/bin/bash
# .git/hooks/pre-commit - Secret detection

echo "Checking for secrets..."

STAGED_CONTENT=$(git diff --cached)

# Паттерны для поиска секретов
patterns=(
    'password\s*=\s*["\047][^"\047]+'
    'secret\s*=\s*["\047][^"\047]+'
    'api[_-]?key\s*=\s*["\047][^"\047]+'
    'AWS[_A-Z]*\s*=\s*["\047][^"\047]+'
    'PRIVATE[_-]?KEY'
    '-----BEGIN (RSA|DSA|EC|OPENSSH) PRIVATE KEY-----'
    'ghp_[a-zA-Z0-9]{36}'          # GitHub token
    'sk-[a-zA-Z0-9]{48}'           # OpenAI API key
    'AIza[0-9A-Za-z\\-_]{35}'      # Google API key
)

for pattern in "${patterns[@]}"; do
    if echo "$STAGED_CONTENT" | grep -qiE "$pattern"; then
        echo "❌ Potential secret detected!"
        echo "   Pattern: $pattern"
        echo ""
        echo "   If this is not a secret, you can:"
        echo "   1. Add it to .gitignore"
        echo "   2. Use environment variables"
        echo "   3. Skip with: git commit --no-verify (not recommended)"
        exit 1
    fi
done

echo "✅ No secrets detected"
```

### Проверка TODO/FIXME

```bash
#!/bin/bash
# .git/hooks/pre-commit - Block TODO comments

STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACMR)

# Ищем TODO/FIXME в staged изменениях
if git diff --cached | grep -E '^\+.*\b(TODO|FIXME|XXX|HACK)\b'; then
    echo ""
    echo "⚠️  Warning: Found TODO/FIXME comments in staged changes"
    echo ""
    read -p "Do you want to commit anyway? (y/n) " -n 1 -r
    echo
    if [[ ! $REPLY =~ ^[Yy]$ ]]; then
        exit 1
    fi
fi
```

## commit-msg: Проверка формата сообщения

### Conventional Commits

```bash
#!/bin/bash
# .git/hooks/commit-msg - Conventional Commits validation

COMMIT_MSG_FILE=$1
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

# Conventional Commits pattern
# type(scope)!: description
PATTERN='^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\([a-z0-9-]+\))?(!)?: .{1,}'

# Игнорируем merge коммиты
if echo "$COMMIT_MSG" | grep -qE '^Merge'; then
    exit 0
fi

# Игнорируем revert коммиты
if echo "$COMMIT_MSG" | grep -qE '^Revert'; then
    exit 0
fi

if ! echo "$COMMIT_MSG" | grep -qE "$PATTERN"; then
    echo "❌ Invalid commit message format!"
    echo ""
    echo "Expected: type(scope): description"
    echo ""
    echo "Types:"
    echo "  feat     - New feature"
    echo "  fix      - Bug fix"
    echo "  docs     - Documentation changes"
    echo "  style    - Code style (formatting, semicolons, etc.)"
    echo "  refactor - Code refactoring"
    echo "  perf     - Performance improvements"
    echo "  test     - Adding or fixing tests"
    echo "  build    - Build system or dependencies"
    echo "  ci       - CI/CD configuration"
    echo "  chore    - Other changes (configs, scripts)"
    echo "  revert   - Revert previous commit"
    echo ""
    echo "Examples:"
    echo "  feat(auth): add OAuth2 login"
    echo "  fix: resolve null pointer exception"
    echo "  docs(readme): update installation guide"
    echo "  feat!: breaking API change"
    echo ""
    echo "Your message: $COMMIT_MSG"
    exit 1
fi

echo "✅ Commit message is valid"
```

### Проверка длины сообщения

```bash
#!/bin/bash
# .git/hooks/commit-msg - Length validation

COMMIT_MSG_FILE=$1
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

# Первая строка (subject)
SUBJECT=$(echo "$COMMIT_MSG" | head -1)

# Длина subject
SUBJECT_LENGTH=${#SUBJECT}
MAX_SUBJECT_LENGTH=72
MIN_SUBJECT_LENGTH=10

if [ $SUBJECT_LENGTH -gt $MAX_SUBJECT_LENGTH ]; then
    echo "❌ Subject line is too long!"
    echo "   Current: $SUBJECT_LENGTH characters"
    echo "   Maximum: $MAX_SUBJECT_LENGTH characters"
    exit 1
fi

if [ $SUBJECT_LENGTH -lt $MIN_SUBJECT_LENGTH ]; then
    echo "❌ Subject line is too short!"
    echo "   Current: $SUBJECT_LENGTH characters"
    echo "   Minimum: $MIN_SUBJECT_LENGTH characters"
    exit 1
fi

# Проверка пустой второй строки (если есть body)
LINE_COUNT=$(echo "$COMMIT_MSG" | wc -l)
if [ $LINE_COUNT -gt 1 ]; then
    SECOND_LINE=$(echo "$COMMIT_MSG" | sed -n '2p')
    if [ -n "$SECOND_LINE" ]; then
        echo "❌ Second line must be empty (separates subject from body)"
        exit 1
    fi
fi
```

### Добавление номера задачи

```bash
#!/bin/bash
# .git/hooks/commit-msg - Add ticket number from branch

COMMIT_MSG_FILE=$1
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

# Получаем имя ветки
BRANCH=$(git symbolic-ref --short HEAD 2>/dev/null)

# Извлекаем номер задачи (JIRA-123, PROJ-456, etc.)
TICKET=$(echo "$BRANCH" | grep -oE '[A-Z]+-[0-9]+' | head -1)

if [ -n "$TICKET" ]; then
    # Проверяем, есть ли уже номер задачи в сообщении
    if ! echo "$COMMIT_MSG" | grep -q "$TICKET"; then
        # Добавляем в конец сообщения
        echo "" >> "$COMMIT_MSG_FILE"
        echo "Refs: $TICKET" >> "$COMMIT_MSG_FILE"
        echo "✅ Added ticket reference: $TICKET"
    fi
fi
```

## pre-push: Запуск тестов перед пушем

### Базовый пример

```bash
#!/bin/bash
# .git/hooks/pre-push

REMOTE=$1
URL=$2

echo "🧪 Running pre-push checks..."

# Запуск тестов
echo "Running tests..."
npm test

if [ $? -ne 0 ]; then
    echo ""
    echo "❌ Tests failed! Push aborted."
    echo "   Fix the tests before pushing."
    exit 1
fi

echo "✅ All tests passed"
```

### Полная проверка

```bash
#!/bin/bash
# .git/hooks/pre-push - Comprehensive checks

REMOTE=$1
URL=$2

echo "🚀 Running pre-push checks..."
echo ""

# 1. Проверка типов (TypeScript)
echo "1️⃣ Type checking..."
npx tsc --noEmit
if [ $? -ne 0 ]; then
    echo "❌ Type errors found!"
    exit 1
fi
echo "✅ Types OK"
echo ""

# 2. Линтинг
echo "2️⃣ Linting..."
npm run lint
if [ $? -ne 0 ]; then
    echo "❌ Lint errors found!"
    exit 1
fi
echo "✅ Lint OK"
echo ""

# 3. Unit тесты
echo "3️⃣ Running unit tests..."
npm run test:unit
if [ $? -ne 0 ]; then
    echo "❌ Unit tests failed!"
    exit 1
fi
echo "✅ Unit tests OK"
echo ""

# 4. Сборка
echo "4️⃣ Building..."
npm run build
if [ $? -ne 0 ]; then
    echo "❌ Build failed!"
    exit 1
fi
echo "✅ Build OK"
echo ""

echo "✅ All pre-push checks passed!"
exit 0
```

### Защита веток

```bash
#!/bin/bash
# .git/hooks/pre-push - Branch protection

REMOTE=$1
URL=$2

# Читаем информацию о push
while read local_ref local_sha remote_ref remote_sha; do
    # Извлекаем имя ветки
    remote_branch=$(echo "$remote_ref" | sed 's|refs/heads/||')

    # Защищённые ветки
    protected="main master develop production"

    for branch in $protected; do
        if [ "$remote_branch" = "$branch" ]; then
            echo "⚠️  Warning: Pushing to protected branch '$branch'"
            echo ""

            # Проверяем, есть ли CI для этой ветки
            if [ "$branch" = "main" ] || [ "$branch" = "master" ]; then
                echo "Running full test suite before push to $branch..."
                npm run test:all
                if [ $? -ne 0 ]; then
                    echo "❌ Tests failed! Push to $branch aborted."
                    exit 1
                fi
            fi

            # Запрашиваем подтверждение
            read -p "Are you sure you want to push to $branch? (yes/no) " confirm
            if [ "$confirm" != "yes" ]; then
                echo "Push cancelled."
                exit 1
            fi
        fi
    done
done

exit 0
```

## Инструменты для управления хуками

### Husky (Node.js)

Самый популярный инструмент для JavaScript/TypeScript проектов.

```bash
# Установка
npm install husky --save-dev

# Инициализация
npx husky init

# Структура после инициализации:
# .husky/
#   _/
#     husky.sh
#   pre-commit

# Добавление хука
echo "npm test" > .husky/pre-commit
```

**package.json конфигурация:**

```json
{
  "scripts": {
    "prepare": "husky",
    "lint": "eslint .",
    "test": "jest"
  }
}
```

**Пример .husky/pre-commit:**

```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npm run lint
npx lint-staged
```

**lint-staged для оптимизации:**

```json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md,css}": [
      "prettier --write"
    ]
  }
}
```

### pre-commit (Python)

Популярный инструмент для Python проектов (но работает с любыми языками).

```bash
# Установка
pip install pre-commit

# Или через brew
brew install pre-commit
```

**Конфигурация .pre-commit-config.yaml:**

```yaml
# .pre-commit-config.yaml
repos:
  # Python formatting
  - repo: https://github.com/psf/black
    rev: 23.12.1
    hooks:
      - id: black

  # Python imports sorting
  - repo: https://github.com/pycqa/isort
    rev: 5.13.2
    hooks:
      - id: isort

  # Python linting
  - repo: https://github.com/pycqa/flake8
    rev: 7.0.0
    hooks:
      - id: flake8

  # Python type checking
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy

  # General hooks
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-added-large-files
        args: ['--maxkb=1000']
      - id: detect-private-key

  # Commit message
  - repo: https://github.com/commitizen-tools/commitizen
    rev: v3.13.0
    hooks:
      - id: commitizen
        stages: [commit-msg]
```

```bash
# Установка хуков
pre-commit install
pre-commit install --hook-type commit-msg

# Запуск на всех файлах
pre-commit run --all-files

# Обновление версий хуков
pre-commit autoupdate
```

### Lefthook (Go)

Быстрый и гибкий инструмент, написанный на Go.

```bash
# Установка
brew install lefthook

# Или через npm
npm install lefthook --save-dev

# Инициализация
lefthook install
```

**Конфигурация lefthook.yml:**

```yaml
# lefthook.yml
pre-commit:
  parallel: true
  commands:
    lint:
      glob: "*.{js,ts,jsx,tsx}"
      run: npx eslint {staged_files}

    prettier:
      glob: "*.{js,ts,jsx,tsx,json,md,css}"
      run: npx prettier --check {staged_files}

    python-lint:
      glob: "*.py"
      run: flake8 {staged_files}

    secrets:
      run: git diff --cached | grep -E "(password|secret|api_key)" && exit 1 || exit 0

commit-msg:
  commands:
    conventional:
      run: |
        commit_regex='^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .+'
        if ! grep -qE "$commit_regex" {1}; then
          echo "Invalid commit message format"
          exit 1
        fi

pre-push:
  commands:
    test:
      run: npm test

    build:
      run: npm run build
```

```bash
# Полезные команды lefthook
lefthook run pre-commit    # Запустить вручную
lefthook install           # Установить хуки
lefthook uninstall         # Удалить хуки
```

## Примеры скриптов

### Полный pre-commit скрипт

```bash
#!/bin/bash
# .git/hooks/pre-commit - Production-ready hook

set -e

# Цвета для вывода
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${YELLOW}Running pre-commit checks...${NC}"
echo ""

# Получаем staged файлы
STAGED=$(git diff --cached --name-only --diff-filter=ACMR)

if [ -z "$STAGED" ]; then
    echo "No staged files"
    exit 0
fi

# 1. Проверка на конфликтные маркеры
echo "Checking for merge conflict markers..."
if git diff --cached | grep -E '^[+].*(<<<<<<<|=======|>>>>>>>)'; then
    echo -e "${RED}ERROR: Merge conflict markers found!${NC}"
    exit 1
fi
echo -e "${GREEN}OK${NC}"

# 2. Проверка на отладочный код
echo "Checking for debug statements..."
JS_DEBUG=$(echo "$STAGED" | xargs grep -l -E '\b(debugger|console\.(log|debug|info))\b' 2>/dev/null || true)
PY_DEBUG=$(echo "$STAGED" | xargs grep -l -E '\b(breakpoint\(\)|pdb\.set_trace\(\)|print\()' 2>/dev/null || true)

if [ -n "$JS_DEBUG" ] || [ -n "$PY_DEBUG" ]; then
    echo -e "${YELLOW}WARNING: Debug statements found in:${NC}"
    [ -n "$JS_DEBUG" ] && echo "$JS_DEBUG"
    [ -n "$PY_DEBUG" ] && echo "$PY_DEBUG"
    read -p "Continue anyway? (y/n) " -n 1 -r
    echo
    [[ ! $REPLY =~ ^[Yy]$ ]] && exit 1
fi
echo -e "${GREEN}OK${NC}"

# 3. JavaScript/TypeScript checks
JS_FILES=$(echo "$STAGED" | grep -E '\.(js|jsx|ts|tsx)$' || true)
if [ -n "$JS_FILES" ]; then
    echo "Running ESLint..."
    npx eslint $JS_FILES --max-warnings=0 || exit 1
    echo -e "${GREEN}OK${NC}"

    echo "Running Prettier..."
    npx prettier --check $JS_FILES || {
        echo -e "${YELLOW}Running Prettier fix...${NC}"
        npx prettier --write $JS_FILES
        git add $JS_FILES
    }
    echo -e "${GREEN}OK${NC}"
fi

# 4. Python checks
PY_FILES=$(echo "$STAGED" | grep '\.py$' || true)
if [ -n "$PY_FILES" ]; then
    echo "Running Black..."
    black --check $PY_FILES || {
        black $PY_FILES
        git add $PY_FILES
    }
    echo -e "${GREEN}OK${NC}"

    echo "Running Flake8..."
    flake8 $PY_FILES || exit 1
    echo -e "${GREEN}OK${NC}"
fi

# 5. Проверка размера файлов
echo "Checking file sizes..."
MAX_SIZE=1048576  # 1MB
for file in $STAGED; do
    if [ -f "$file" ]; then
        size=$(wc -c < "$file")
        if [ $size -gt $MAX_SIZE ]; then
            echo -e "${RED}ERROR: $file is too large ($(($size/1024))KB > 1MB)${NC}"
            exit 1
        fi
    fi
done
echo -e "${GREEN}OK${NC}"

echo ""
echo -e "${GREEN}All pre-commit checks passed!${NC}"
exit 0
```

### Полный commit-msg скрипт

```bash
#!/bin/bash
# .git/hooks/commit-msg - Complete message validation

COMMIT_MSG_FILE=$1
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

# Цвета
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

# Пропускаем merge и revert коммиты
if echo "$COMMIT_MSG" | head -1 | grep -qE '^(Merge|Revert)'; then
    exit 0
fi

echo -e "${YELLOW}Validating commit message...${NC}"

# 1. Conventional Commits format
PATTERN='^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\([a-z0-9/_-]+\))?(!)?: .{3,}'

if ! echo "$COMMIT_MSG" | head -1 | grep -qE "$PATTERN"; then
    echo -e "${RED}ERROR: Invalid commit message format!${NC}"
    echo ""
    echo "Format: type(scope): description"
    echo ""
    echo "Types: feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert"
    echo ""
    echo -e "Your message: ${YELLOW}$(echo "$COMMIT_MSG" | head -1)${NC}"
    exit 1
fi

# 2. Subject length (max 72)
SUBJECT=$(echo "$COMMIT_MSG" | head -1)
if [ ${#SUBJECT} -gt 72 ]; then
    echo -e "${RED}ERROR: Subject too long (${#SUBJECT} > 72 chars)${NC}"
    exit 1
fi

# 3. No period at end of subject
if echo "$SUBJECT" | grep -qE '\.$'; then
    echo -e "${RED}ERROR: Subject should not end with a period${NC}"
    exit 1
fi

# 4. Capitalize first letter after type
if ! echo "$SUBJECT" | grep -qE '^[a-z]+(\([a-z0-9/_-]+\))?(!)?: [A-Z]'; then
    echo -e "${YELLOW}WARNING: Description should start with capital letter${NC}"
fi

# 5. Body formatting (if present)
LINE_COUNT=$(echo "$COMMIT_MSG" | wc -l)
if [ $LINE_COUNT -gt 1 ]; then
    SECOND_LINE=$(echo "$COMMIT_MSG" | sed -n '2p')
    if [ -n "$SECOND_LINE" ]; then
        echo -e "${RED}ERROR: Second line must be blank${NC}"
        exit 1
    fi

    # Check body line length (max 100)
    while IFS= read -r line; do
        if [ ${#line} -gt 100 ]; then
            echo -e "${YELLOW}WARNING: Line too long (${#line} > 100 chars)${NC}"
        fi
    done < <(echo "$COMMIT_MSG" | tail -n +3)
fi

echo -e "${GREEN}Commit message is valid!${NC}"
exit 0
```

## Best Practices

### 1. Делайте хуки быстрыми

```bash
# Плохо - проверяем все файлы
npm run lint  # 30 секунд

# Хорошо - только staged файлы
npx eslint $(git diff --cached --name-only | grep '\.js$')  # 2 секунды
```

### 2. Выводите понятные сообщения

```bash
echo "❌ Lint errors found!"
echo ""
echo "Affected files:"
echo "$FAILED_FILES"
echo ""
echo "To fix automatically, run:"
echo "  npm run lint:fix"
```

### 3. Позволяйте пропустить в экстренных случаях

```bash
# Документируйте возможность пропуска
echo "To skip this check (emergency only):"
echo "  git commit --no-verify"
```

### 4. Используйте инструменты вместо самописных скриптов

```bash
# Вместо самописного хука
# Используйте husky + lint-staged для JS
# Используйте pre-commit для Python
# Используйте lefthook для мультиязычных проектов
```

### 5. Тестируйте хуки локально

```bash
# Запуск pre-commit вручную
.git/hooks/pre-commit

# Запуск commit-msg с тестовым сообщением
echo "test: example message" > /tmp/test-msg
.git/hooks/commit-msg /tmp/test-msg
```
