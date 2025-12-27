# Client vs Server Hooks - Клиентские и серверные хуки

## Обзор

Git хуки делятся на две категории в зависимости от того, где они выполняются:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Git Hooks Architecture                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   LOCAL MACHINE                           REMOTE SERVER                  │
│   (Developer)                             (GitHub/GitLab/etc)           │
│                                                                          │
│   ┌────────────────────┐                 ┌────────────────────┐         │
│   │  Client-side Hooks │    git push     │ Server-side Hooks  │         │
│   │                    │ ───────────────►│                    │         │
│   │  - pre-commit      │                 │  - pre-receive     │         │
│   │  - commit-msg      │                 │  - update          │         │
│   │  - pre-push        │                 │  - post-receive    │         │
│   │  - post-commit     │                 │                    │         │
│   │  - pre-rebase      │                 │                    │         │
│   └────────────────────┘                 └────────────────────┘         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## Клиентские хуки (Client-side)

Выполняются на машине разработчика. Хранятся в `.git/hooks/` локального репозитория.

### Хуки коммитов

#### pre-commit

Запускается **перед** созданием коммита, до открытия редактора для сообщения.

```bash
#!/bin/bash
# .git/hooks/pre-commit

# Типичные задачи:
# - Линтинг кода
# - Проверка форматирования
# - Запуск быстрых тестов
# - Проверка на наличие секретов

echo "Running pre-commit checks..."

# Проверка на забытые console.log
if git diff --cached | grep -E '\bconsole\.(log|debug)\b'; then
    echo "Error: console.log/debug found in staged files"
    exit 1
fi

# Линтинг
npm run lint --quiet
exit $?
```

#### prepare-commit-msg

Запускается **после** создания дефолтного сообщения, но **до** открытия редактора.

```bash
#!/bin/bash
# .git/hooks/prepare-commit-msg
# Аргументы: $1 = путь к файлу с сообщением, $2 = тип, $3 = SHA (для amend)

COMMIT_MSG_FILE=$1
COMMIT_SOURCE=$2

# Добавляем шаблон сообщения
if [ -z "$COMMIT_SOURCE" ]; then
    # Получаем имя текущей ветки
    branch=$(git symbolic-ref --short HEAD 2>/dev/null)

    # Извлекаем номер задачи из имени ветки
    ticket=$(echo "$branch" | grep -oE '[A-Z]+-[0-9]+')

    if [ -n "$ticket" ]; then
        # Добавляем номер задачи в начало сообщения
        sed -i.bak "1s/^/[$ticket] /" "$COMMIT_MSG_FILE"
    fi
fi
```

#### commit-msg

Запускается **после** ввода сообщения коммита. Используется для валидации.

```bash
#!/bin/bash
# .git/hooks/commit-msg
# Аргумент: $1 = путь к файлу с сообщением коммита

COMMIT_MSG_FILE=$1
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

# Conventional Commits validation
commit_regex='^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\(.+\))?(!)?: .{1,}'

if ! echo "$COMMIT_MSG" | grep -qE "$commit_regex"; then
    echo "ERROR: Invalid commit message format!"
    echo ""
    echo "Expected format: type(scope): description"
    echo ""
    echo "Types: feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert"
    echo ""
    echo "Examples:"
    echo "  feat(auth): add login functionality"
    echo "  fix: resolve memory leak in parser"
    echo "  docs(readme): update installation steps"
    exit 1
fi

# Проверка минимальной длины
min_length=10
if [ ${#COMMIT_MSG} -lt $min_length ]; then
    echo "ERROR: Commit message is too short (min $min_length characters)"
    exit 1
fi

exit 0
```

#### post-commit

Запускается **после** успешного создания коммита. Не может отменить коммит.

```bash
#!/bin/bash
# .git/hooks/post-commit

# Получаем информацию о коммите
commit_hash=$(git rev-parse HEAD)
commit_msg=$(git log -1 --pretty=%B)
author=$(git log -1 --pretty=%an)

echo "Commit $commit_hash created successfully!"

# Отправка уведомления
# curl -X POST -d "New commit by $author: $commit_msg" $WEBHOOK_URL

# Логирование
echo "$(date): $commit_hash - $commit_msg" >> ~/.git-commits.log
```

### Хуки push/pull

#### pre-push

Запускается **перед** отправкой данных на сервер. Может отменить push.

```bash
#!/bin/bash
# .git/hooks/pre-push
# Аргументы: $1 = имя remote, $2 = URL remote
# stdin: <local ref> <local sha> <remote ref> <remote sha>

remote="$1"
url="$2"

echo "Running pre-push checks..."

# Запуск тестов
npm test
if [ $? -ne 0 ]; then
    echo "Tests failed! Push aborted."
    exit 1
fi

# Запрет пуша в main без PR
while read local_ref local_sha remote_ref remote_sha; do
    if [[ "$remote_ref" == "refs/heads/main" ]] || [[ "$remote_ref" == "refs/heads/master" ]]; then
        echo "ERROR: Direct push to $remote_ref is not allowed!"
        echo "Please create a pull request instead."
        exit 1
    fi
done

exit 0
```

#### pre-rebase

Запускается **перед** rebase. Может запретить rebase.

```bash
#!/bin/bash
# .git/hooks/pre-rebase
# Аргументы: $1 = upstream branch, $2 = rebased branch (пусто если HEAD)

upstream=$1
branch=${2:-HEAD}

# Запрет rebase на публичных ветках
current_branch=$(git symbolic-ref --short HEAD)

protected_branches="main master develop"

for protected in $protected_branches; do
    if [ "$current_branch" = "$protected" ]; then
        echo "ERROR: Rebase of '$protected' branch is not allowed!"
        echo "Protected branches should not be rebased."
        exit 1
    fi
done

exit 0
```

#### post-merge

Запускается **после** успешного merge.

```bash
#!/bin/bash
# .git/hooks/post-merge
# Аргумент: $1 = squash merge flag (0 или 1)

squash_merge=$1

echo "Merge completed!"

# Проверяем, изменился ли package.json
changed_files=$(git diff-tree -r --name-only --no-commit-id ORIG_HEAD HEAD)

if echo "$changed_files" | grep -q "package.json"; then
    echo "package.json changed. Running npm install..."
    npm install
fi

if echo "$changed_files" | grep -q "requirements.txt"; then
    echo "requirements.txt changed. Running pip install..."
    pip install -r requirements.txt
fi
```

#### post-checkout

Запускается **после** успешного checkout.

```bash
#!/bin/bash
# .git/hooks/post-checkout
# Аргументы: $1 = prev HEAD, $2 = new HEAD, $3 = branch flag (1=branch, 0=file)

prev_head=$1
new_head=$2
is_branch_checkout=$3

if [ "$is_branch_checkout" = "1" ]; then
    # Это переключение ветки
    branch=$(git symbolic-ref --short HEAD)
    echo "Switched to branch: $branch"

    # Проверяем изменения в зависимостях
    if git diff "$prev_head" "$new_head" --name-only | grep -q "package-lock.json"; then
        echo "Dependencies changed. Run 'npm install' to update."
    fi
fi
```

### Другие клиентские хуки

```bash
# pre-auto-gc - перед автоматической сборкой мусора
# post-rewrite - после команд, переписывающих историю (rebase, amend)
# pre-applypatch - перед применением патча (git am)
# post-applypatch - после применения патча
```

## Серверные хуки (Server-side)

Выполняются на сервере при получении push. Используются для политик и автоматизации.

### pre-receive

Запускается **один раз** перед обновлением любых refs. Может отклонить весь push.

```bash
#!/bin/bash
# hooks/pre-receive (на сервере)
# stdin: <old-value> <new-value> <ref-name>

echo "Running pre-receive hook..."

while read oldrev newrev refname; do
    # Запрет force-push в защищённые ветки
    if [[ "$refname" == "refs/heads/main" ]] || [[ "$refname" == "refs/heads/master" ]]; then
        # Проверяем, является ли это force-push
        if [ "$oldrev" != "0000000000000000000000000000000000000000" ]; then
            merge_base=$(git merge-base $oldrev $newrev 2>/dev/null)
            if [ "$merge_base" != "$oldrev" ]; then
                echo "ERROR: Force push to $refname is not allowed!"
                exit 1
            fi
        fi
    fi

    # Проверка размера файлов
    max_size=10485760  # 10MB

    for file in $(git diff --name-only $oldrev $newrev); do
        size=$(git cat-file -s "$newrev:$file" 2>/dev/null || echo 0)
        if [ "$size" -gt "$max_size" ]; then
            echo "ERROR: File $file is too large ($(($size/1024/1024))MB > 10MB)"
            exit 1
        fi
    done
done

exit 0
```

### update

Запускается **для каждой ветки** отдельно. Более гранулярный контроль.

```bash
#!/bin/bash
# hooks/update (на сервере)
# Аргументы: $1 = ref name, $2 = old SHA, $3 = new SHA

refname=$1
oldrev=$2
newrev=$3

echo "Updating $refname from $oldrev to $newrev"

# Извлекаем имя ветки
branch=$(echo "$refname" | sed 's|refs/heads/||')

# Правила для разных веток
case "$branch" in
    main|master)
        # Только merge коммиты в main
        for commit in $(git rev-list $oldrev..$newrev); do
            parents=$(git cat-file -p $commit | grep -c "^parent")
            if [ "$parents" -lt 2 ]; then
                # Проверяем, это merge или обычный коммит
                # Разрешаем, если это первый коммит или merge
                echo "WARNING: Non-merge commit $commit detected on $branch"
            fi
        done
        ;;
    release/*)
        # Только определённые пользователи могут пушить в release
        # (требует настройки авторизации)
        ;;
    feature/*|bugfix/*)
        # Обычные правила
        ;;
    *)
        # Неизвестный паттерн ветки
        echo "WARNING: Branch $branch doesn't match any naming convention"
        ;;
esac

exit 0
```

### post-receive

Запускается **после** успешного обновления refs. Не может отменить push.

```bash
#!/bin/bash
# hooks/post-receive (на сервере)
# stdin: <old-value> <new-value> <ref-name>

echo "Push received successfully!"

while read oldrev newrev refname; do
    branch=$(echo "$refname" | sed 's|refs/heads/||')

    # Триггер CI/CD
    if [ "$branch" = "main" ]; then
        echo "Triggering production deployment..."
        # curl -X POST $CI_WEBHOOK_URL
    elif [ "$branch" = "develop" ]; then
        echo "Triggering staging deployment..."
        # curl -X POST $STAGING_WEBHOOK_URL
    fi

    # Отправка уведомлений
    commit_count=$(git rev-list --count $oldrev..$newrev 2>/dev/null || echo "?")
    echo "Branch $branch updated with $commit_count new commits"

    # Email уведомление
    # git log $oldrev..$newrev | mail -s "New commits in $branch" team@example.com
done
```

### post-update

Устаревший хук, используйте post-receive вместо него.

```bash
#!/bin/bash
# hooks/post-update (на сервере)
# Аргументы: список обновлённых refs

# Обновление info/refs для dumb HTTP протокола
exec git update-server-info
```

## Когда какие использовать

### Клиентские хуки подходят для:

| Хук | Использование |
|-----|---------------|
| pre-commit | Линтинг, форматирование, быстрые тесты |
| prepare-commit-msg | Шаблоны сообщений, добавление ticket ID |
| commit-msg | Валидация формата сообщения |
| post-commit | Локальные уведомления, логирование |
| pre-push | Полные тесты, сборка, проверка ветки |
| post-merge | Обновление зависимостей |
| post-checkout | Переключение окружений |

### Серверные хуки подходят для:

| Хук | Использование |
|-----|---------------|
| pre-receive | Глобальные политики, размер файлов, запрет веток |
| update | Политики для конкретных веток |
| post-receive | CI/CD триггеры, уведомления, деплой |

## Ограничения клиентских хуков

### 1. Не гарантируют выполнение

```bash
# Разработчик может пропустить хуки
git commit --no-verify -m "Skip all hooks"
git push --no-verify

# Или удалить файлы хуков
rm .git/hooks/pre-commit
```

### 2. Не синхронизируются автоматически

```bash
# .git/hooks/ не отслеживается Git
# Каждый разработчик должен настроить хуки локально

# Решение 1: Скрипт установки
cat > scripts/install-hooks.sh << 'EOF'
#!/bin/bash
cp hooks/* .git/hooks/
chmod +x .git/hooks/*
EOF

# Решение 2: Изменение пути к хукам
git config core.hooksPath ./hooks

# Решение 3: Использование инструментов (husky, pre-commit)
```

### 3. Зависят от окружения

```bash
#!/bin/bash
# Хук может не работать, если зависимости не установлены

# Плохо - предполагает, что eslint установлен
eslint .

# Лучше - проверяем наличие
if command -v npx &> /dev/null; then
    npx eslint .
else
    echo "Warning: npm not found, skipping lint"
fi
```

### 4. Могут замедлять работу

```bash
# Плохо - полный запуск тестов на каждый коммит
npm test  # 5 минут

# Лучше - быстрые проверки в pre-commit
npm run lint  # 5 секунд

# Полные тесты - в pre-push или CI
```

## Сравнительная таблица

| Аспект | Клиентские | Серверные |
|--------|------------|-----------|
| Где выполняются | Локально у разработчика | На сервере |
| Можно обойти | Да (--no-verify) | Нет |
| Синхронизация | Требует настройки | Настроены централизованно |
| Скорость | Должны быть быстрыми | Могут быть медленнее |
| Доступ к коду | Полный доступ | Только push данные |
| Типичные задачи | Качество кода | Политики, CI/CD |
| Feedback | Мгновенный | После push |

## Best Practices

### 1. Используйте оба типа хуков

```
Клиентские хуки     Серверные хуки
       ↓                  ↓
   Быстрая           Обязательная
   обратная          проверка
   связь             политик
       ↓                  ↓
   Улучшает          Гарантирует
   Developer         качество
   Experience        кода
```

### 2. Клиентские хуки - первая линия защиты

```bash
# pre-commit: быстрые проверки (< 10 секунд)
- Линтинг staged файлов
- Форматирование
- Проверка на секреты

# pre-push: более полные проверки
- Unit тесты
- Type checking
- Build проверка
```

### 3. Серверные хуки - последняя линия защиты

```bash
# pre-receive: критические политики
- Запрет force-push
- Размер файлов
- Защита веток

# post-receive: автоматизация
- CI/CD триггеры
- Уведомления
```

### 4. Делайте хуки идемпотентными

```bash
#!/bin/bash
# Хук можно запустить несколько раз безопасно

# Плохо - может создать дубликаты
echo "[$TICKET]" >> "$COMMIT_MSG_FILE"

# Хорошо - проверяем перед добавлением
if ! grep -q "\\[$TICKET\\]" "$COMMIT_MSG_FILE"; then
    sed -i "1s/^/[$TICKET] /" "$COMMIT_MSG_FILE"
fi
```
