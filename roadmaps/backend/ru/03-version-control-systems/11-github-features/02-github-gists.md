# GitHub Gists

## Что такое GitHub Gists?

**GitHub Gists** - это простой способ делиться фрагментами кода, заметками и другими текстовыми данными. По сути, каждый gist - это мини-репозиторий Git, который можно клонировать, форкать и версионировать.

### Основные особенности

- Каждый gist - это полноценный Git-репозиторий
- Поддержка множества файлов в одном gist
- Подсветка синтаксиса для различных языков
- Возможность комментирования
- История изменений (revisions)
- Встраивание в веб-страницы

## Типы Gists

### 1. Public Gists (Публичные)

**Публичные gists** видны всем и индексируются поисковыми системами.

**Особенности:**
- Отображаются в вашем профиле
- Можно найти через поиск GitHub
- Индексируются Google
- Идеально для sharing полезных сниппетов

### 2. Secret Gists (Секретные)

**Секретные gists** не видны в публичном списке, но доступны по прямой ссылке.

**Особенности:**
- НЕ отображаются в профиле
- НЕ индексируются поисковыми системами
- Доступны по прямой ссылке
- **НЕ являются приватными** - любой с ссылкой может их видеть
- Идеально для черновиков и временного шаринга

> **Важно:** Secret gists не обеспечивают настоящую приватность. Если нужна конфиденциальность - используйте приватные репозитории.

## Создание Gist

### Через веб-интерфейс

1. Перейдите на [gist.github.com](https://gist.github.com)
2. Введите описание gist (опционально)
3. Укажите имя файла с расширением (например, `script.py`)
4. Введите содержимое файла
5. Нажмите **Add file** для добавления дополнительных файлов
6. Выберите **Create secret gist** или **Create public gist**

### Через GitHub CLI

```bash
# Создание gist из файла
gh gist create script.py

# Публичный gist
gh gist create --public script.py

# С описанием
gh gist create -d "Полезный скрипт для парсинга" parser.py

# Несколько файлов
gh gist create file1.py file2.py file3.py

# Из stdin
echo "print('Hello')" | gh gist create -f hello.py
```

### Через Git

```bash
# Создание локально и push
git clone https://gist.github.com/YOUR_GIST_ID.git
cd YOUR_GIST_ID
# Редактируем файлы
git add .
git commit -m "Update gist"
git push
```

## Структура URL Gist

```
https://gist.github.com/{username}/{gist_id}
```

**Примеры:**
- `https://gist.github.com/octocat/1234567890abcdef` - полный gist
- `https://gist.github.com/1234567890abcdef` - короткая форма

### Доступ к конкретному файлу

```
https://gist.github.com/{username}/{gist_id}#file-{filename}
```

### Доступ к raw-контенту

```
https://gist.githubusercontent.com/{username}/{gist_id}/raw/{file}
```

## Редактирование Gist

### Через веб-интерфейс

1. Откройте gist
2. Нажмите кнопку **Edit** (карандаш)
3. Внесите изменения
4. Нажмите **Update secret/public gist**

### Через Git

```bash
# Клонирование gist
git clone https://gist.github.com/username/gist_id.git
cd gist_id

# Редактирование
vim script.py

# Commit и push
git add .
git commit -m "Updated script"
git push origin master
```

### Через GitHub CLI

```bash
# Редактирование gist
gh gist edit gist_id

# Добавление файла в существующий gist
gh gist edit gist_id --add newfile.py
```

## Форкинг и клонирование

### Fork Gist

Создает копию чужого gist в вашем аккаунте:

1. Откройте gist
2. Нажмите **Fork** в правом верхнем углу
3. Теперь у вас есть собственная копия

### Clone Gist

```bash
# Клонирование gist
git clone https://gist.github.com/username/gist_id.git

# Или через SSH
git clone git@gist.github.com:gist_id.git

# Через GitHub CLI
gh gist clone gist_id
```

## Встраивание Gist в сайты

### Стандартный embed

GitHub предоставляет готовый скрипт для встраивания:

```html
<script src="https://gist.github.com/username/gist_id.js"></script>
```

### Встраивание конкретного файла

```html
<script src="https://gist.github.com/username/gist_id.js?file=specific_file.py"></script>
```

### Пример в HTML-странице

```html
<!DOCTYPE html>
<html>
<head>
    <title>Code Examples</title>
</head>
<body>
    <h1>My Python Snippet</h1>
    <script src="https://gist.github.com/octocat/1234567890abcdef.js"></script>
</body>
</html>
```

### Встраивание в Markdown (GitHub)

```markdown
В GitHub-flavored Markdown можно использовать прямую ссылку:

https://gist.github.com/username/gist_id

GitHub автоматически отрендерит preview.
```

### Кастомизация встроенного gist

CSS для стилизации embedded gist:

```css
/* Стилизация контейнера gist */
.gist {
    max-width: 800px;
    margin: 20px auto;
}

/* Изменение фона */
.gist .gist-file {
    border: 1px solid #ddd;
    border-radius: 8px;
}

/* Скрытие footer */
.gist .gist-meta {
    display: none;
}
```

## Практические примеры использования

### 1. Сниппеты кода

```python
# gist: python_decorators.py
# Полезные декораторы Python

import functools
import time

def timer(func):
    """Измеряет время выполнения функции"""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        end = time.perf_counter()
        print(f"{func.__name__} выполнена за {end - start:.4f} сек")
        return result
    return wrapper

def retry(max_attempts=3, delay=1):
    """Повторяет вызов функции при ошибке"""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt < max_attempts - 1:
                        time.sleep(delay)
                    else:
                        raise e
        return wrapper
    return decorator
```

### 2. Конфигурационные файлы

```yaml
# gist: docker-compose-dev.yml
# Docker Compose для локальной разработки

version: '3.8'

services:
  app:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - .:/app
    environment:
      - DEBUG=true
      - DATABASE_URL=postgres://user:pass@db:5432/mydb

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### 3. Шпаргалки (Cheatsheets)

```markdown
# gist: git-cheatsheet.md
# Git Cheatsheet

## Базовые команды
- `git init` - инициализация репозитория
- `git clone <url>` - клонирование
- `git status` - статус изменений

## Ветвление
- `git branch <name>` - создать ветку
- `git checkout <name>` - переключиться
- `git merge <branch>` - слияние

## Удаленные репозитории
- `git remote -v` - список remote
- `git push origin <branch>` - отправить
- `git pull` - получить изменения
```

### 4. SQL-запросы

```sql
-- gist: useful_postgres_queries.sql
-- Полезные PostgreSQL запросы

-- Размер всех таблиц
SELECT
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- Активные соединения
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    query
FROM pg_stat_activity
WHERE state = 'active';

-- Медленные запросы (требуется pg_stat_statements)
SELECT
    query,
    calls,
    total_time / 1000 AS total_seconds,
    mean_time / 1000 AS mean_seconds
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;
```

## Управление Gists через GitHub CLI

### Список gists

```bash
# Ваши gists
gh gist list

# Публичные gists
gh gist list --public

# Секретные gists
gh gist list --secret

# Ограничить количество
gh gist list --limit 5
```

### Просмотр gist

```bash
# Просмотр содержимого
gh gist view gist_id

# Просмотр конкретного файла
gh gist view gist_id --filename script.py

# Открыть в браузере
gh gist view gist_id --web
```

### Удаление gist

```bash
gh gist delete gist_id
```

## История изменений (Revisions)

Каждый gist хранит историю всех изменений.

### Просмотр истории

1. Откройте gist
2. Нажмите **Revisions** в правом верхнем углу
3. Просмотрите diff между версиями

### Доступ к конкретной ревизии

```
https://gist.github.com/{username}/{gist_id}/{revision_hash}
```

### Через Git

```bash
cd gist_directory
git log --oneline
git show <commit_hash>
git diff <commit1> <commit2>
```

## Gist API

### Получение gist через API

```bash
# Получить информацию о gist
curl https://api.github.com/gists/gist_id

# Получить ваши gists (с авторизацией)
curl -H "Authorization: token YOUR_TOKEN" \
     https://api.github.com/gists
```

### Создание gist через API

```bash
curl -X POST \
  -H "Authorization: token YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "Example gist",
    "public": false,
    "files": {
      "hello.py": {
        "content": "print(\"Hello, World!\")"
      }
    }
  }' \
  https://api.github.com/gists
```

## Полезные советы

1. **Используйте осмысленные имена файлов** - gists сортируются по имени файла
2. **Добавляйте описания** - помогает найти gist позже
3. **Используйте расширения файлов** - для правильной подсветки синтаксиса
4. **Secret для черновиков** - переключить на public можно, наоборот - нет
5. **Версионируйте через git** - для сложных gists используйте git workflow

## Альтернативы GitHub Gists

| Сервис | Особенности |
|--------|-------------|
| Pastebin | Простой, анонимный |
| GitLab Snippets | Аналог gists для GitLab |
| Carbon | Красивые изображения кода |
| CodePen | HTML/CSS/JS песочница |
| JSFiddle | JavaScript playground |
| Replit | Полноценная IDE |

## Полезные ссылки

- [GitHub Gists](https://gist.github.com)
- [Gists API Documentation](https://docs.github.com/en/rest/gists)
- [GitHub CLI - Gist](https://cli.github.com/manual/gh_gist)
