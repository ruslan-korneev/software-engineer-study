# Committing Changes (Создание коммитов)

## Что такое коммит?

**Коммит** (commit) — это снимок (snapshot) вашего проекта в определённый момент времени. Каждый коммит содержит:

- **Изменения файлов** — что именно было добавлено, изменено или удалено
- **Метаданные** — автор, дата, сообщение
- **Ссылку на родительский коммит** — связь с предыдущим состоянием
- **Уникальный идентификатор (SHA-1 hash)** — например, `a1b2c3d4e5f6...`

## Анатомия коммита

```
commit a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0
Author: Ivan Petrov <ivan@example.com>
Date:   Mon Dec 25 10:30:00 2024 +0300

    Add user authentication module

    - Implement JWT token generation
    - Add login/logout endpoints
    - Create user session management

 src/auth/jwt.py       | 45 +++++++++++++++++++++++++++
 src/auth/session.py   | 32 +++++++++++++++++++
 src/api/endpoints.py  | 28 +++++++++++++++++
 3 files changed, 105 insertions(+)
```

## Основные команды

### git commit — создание коммита

```bash
# Базовый коммит с сообщением
git commit -m "Add user authentication"

# Коммит с подробным описанием (откроется редактор)
git commit

# Добавить все изменённые tracked файлы и закоммитить
git commit -a -m "Fix validation bug"
# Эквивалентно:
# git add -u && git commit -m "Fix validation bug"

# Показать diff в редакторе при написании сообщения
git commit -v

# Пустой коммит (для триггера CI/CD)
git commit --allow-empty -m "Trigger build"
```

### Редактирование последнего коммита

```bash
# Изменить сообщение последнего коммита
git commit --amend -m "New message"

# Добавить забытые файлы в последний коммит
git add forgotten_file.py
git commit --amend --no-edit  # сохранить старое сообщение

# Изменить и сообщение, и содержимое
git add new_changes.py
git commit --amend -m "Updated message with new changes"
```

**ВАЖНО**: Не используйте `--amend` для коммитов, которые уже отправлены в удалённый репозиторий!

## Правила хороших сообщений коммитов

### Формат сообщения

```
<тип>: <краткое описание> (до 50 символов)

<пустая строка>

<подробное описание, если нужно> (до 72 символов на строку)

<пустая строка>

<ссылки на issues, если есть>
```

### Типы коммитов (Conventional Commits)

```bash
feat: Add user registration        # новая функциональность
fix: Fix login timeout issue       # исправление бага
docs: Update API documentation     # документация
style: Format code with black      # форматирование (не меняет логику)
refactor: Restructure auth module  # рефакторинг (не меняет поведение)
test: Add unit tests for auth      # тесты
chore: Update dependencies         # служебные задачи
perf: Optimize database queries    # улучшение производительности
```

### Примеры хороших сообщений

```bash
# Плохо
git commit -m "fix"
git commit -m "update"
git commit -m "changes"

# Хорошо
git commit -m "Fix null pointer exception in user validation"
git commit -m "Add email verification to signup flow"
git commit -m "Refactor database connection pooling"
```

### Многострочные сообщения

```bash
# Способ 1: через редактор
git commit
# Откроется редактор (vim, nano, VS Code...)

# Способ 2: несколько -m флагов
git commit -m "Add payment processing" -m "" -m "- Integrate Stripe API" -m "- Add webhook handlers"

# Способ 3: Heredoc
git commit -m "$(cat <<EOF
Add user authentication system

- Implement JWT token generation and validation
- Add login/logout REST endpoints
- Create session management with Redis
- Add rate limiting for auth endpoints

Closes #123
EOF
)"
```

## Просмотр коммитов

```bash
# Информация о последнем коммите
git show

# Информация о конкретном коммите
git show a1b2c3d

# Показать только изменённые файлы
git show --stat

# Показать только имена файлов
git show --name-only

# Показать коммит в одну строку
git show --oneline
```

## Отмена коммитов

### git reset — отмена локальных коммитов

```bash
# Мягкий reset — убрать коммит, но сохранить изменения в staging
git reset --soft HEAD~1

# Смешанный reset (по умолчанию) — убрать коммит, изменения в working directory
git reset HEAD~1
# или
git reset --mixed HEAD~1

# Жёсткий reset — полностью удалить коммит и все изменения
git reset --hard HEAD~1
# ОСТОРОЖНО! Изменения будут потеряны!

# Отменить последние N коммитов
git reset HEAD~3  # отменить 3 коммита
```

### git revert — безопасная отмена

```bash
# Создать новый коммит, отменяющий указанный
git revert a1b2c3d

# Отменить последний коммит
git revert HEAD

# Отменить без автоматического коммита
git revert --no-commit a1b2c3d
```

**Когда использовать что:**
- `reset` — для локальных изменений (ещё не push)
- `revert` — для изменений, которые уже в удалённом репозитории

## Практические сценарии

### Сценарий 1: Атомарные коммиты

```bash
# Плохо — один огромный коммит
git add .
git commit -m "Implement user management, fix bugs, update docs"

# Хорошо — разделяем на логические части
git add src/models/user.py src/api/users.py
git commit -m "feat: Add user CRUD operations"

git add tests/test_users.py
git commit -m "test: Add unit tests for user operations"

git add docs/api.md
git commit -m "docs: Document user API endpoints"
```

### Сценарий 2: Забыли добавить файл

```bash
# Сделали коммит, но забыли файл
git commit -m "Add feature X"

# Добавляем забытый файл
git add forgotten_file.py
git commit --amend --no-edit
```

### Сценарий 3: Опечатка в сообщении

```bash
# Опечатка в последнем коммите
git commit -m "Fix bugg in login"

# Исправляем
git commit --amend -m "Fix bug in login"
```

### Сценарий 4: Разбиение большого изменения

```bash
# Используем git add -p для выборочного добавления
git add -p large_file.py
# Выбираем только изменения для первой задачи
git commit -m "Refactor error handling"

git add -p large_file.py
# Выбираем изменения для второй задачи
git commit -m "Add input validation"
```

## Best Practices

### 1. Коммитьте часто
```bash
# Маленькие, частые коммиты лучше больших и редких
# Легче искать баги, делать code review, откатывать
```

### 2. Один коммит — одна задача
```bash
# Каждый коммит должен быть "атомарным"
# Можно откатить без влияния на другие изменения
```

### 3. Пишите осмысленные сообщения
```bash
# Сообщение должно отвечать на вопрос "Что делает этот коммит?"
# А не "Что я делал?"
```

### 4. Используйте императив
```bash
# Хорошо (императив)
"Add user authentication"
"Fix login bug"
"Update dependencies"

# Плохо (прошедшее время)
"Added user authentication"
"Fixed login bug"
"Updated dependencies"
```

### 5. Проверяйте перед коммитом
```bash
# Смотрим, что коммитим
git diff --staged

# Запускаем тесты
pytest

# Только потом коммитим
git commit -m "..."
```

## Типичные ошибки

### 1. Коммит с секретами
```bash
# НИКОГДА не коммитьте:
# - Пароли и ключи API
# - Файлы .env с реальными данными
# - Приватные ключи SSH

# Если случайно закоммитили — история хранит всё!
# Придётся перезаписывать историю (git filter-branch)
```

### 2. Слишком большие коммиты
```bash
# Плохо
git add .
git commit -m "Week of work"

# Сложно делать review, сложно искать баги
```

### 3. amend после push
```bash
# ОСТОРОЖНО!
git push origin main
git commit --amend  # Изменили локально
git push origin main  # ОШИБКА! Истории не совпадают

# Придётся делать force push (плохо для команды)
git push --force origin main
```

### 4. Бессмысленные сообщения
```bash
# Плохо
git commit -m "fix"
git commit -m "wip"
git commit -m "asdf"

# Через месяц не поймёте, что это было
```

## Полезные настройки

```bash
# Шаблон для сообщений коммитов
git config --global commit.template ~/.gitmessage

# Автоматическая подпись коммитов
git config --global commit.gpgsign true

# Редактор для сообщений
git config --global core.editor "code --wait"  # VS Code
git config --global core.editor "vim"           # Vim
```

## Резюме

- **Коммит** — снимок проекта с метаданными и сообщением
- `git commit -m "message"` — создать коммит
- `git commit --amend` — исправить последний коммит (только локально!)
- `git reset` — отменить локальные коммиты
- `git revert` — безопасно отменить коммит (создаёт новый)
- Пишите осмысленные сообщения в императиве
- Делайте атомарные коммиты (один коммит = одна задача)
- Всегда проверяйте `git diff --staged` перед коммитом
