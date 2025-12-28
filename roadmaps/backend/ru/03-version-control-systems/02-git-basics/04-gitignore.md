# .gitignore (Игнорирование файлов)

[prev: 03-committing-changes](./03-committing-changes.md) | [next: 05-viewing-commit-history](./05-viewing-commit-history.md)
---

## Что такое .gitignore?

**`.gitignore`** — это специальный файл, который указывает Git, какие файлы и директории нужно игнорировать. Игнорируемые файлы не отслеживаются и не попадают в репозиторий.

## Зачем нужен .gitignore?

1. **Безопасность** — не коммитить пароли, ключи API, приватные ключи
2. **Чистота репозитория** — не засорять логами, кэшами, временными файлами
3. **Экономия места** — не хранить сгенерированные файлы (build, node_modules)
4. **Независимость** — не хранить настройки IDE конкретного разработчика

## Синтаксис .gitignore

### Базовые правила

```gitignore
# Это комментарий

# Игнорировать конкретный файл
secret.txt

# Игнорировать файл в конкретной директории
config/secrets.json

# Игнорировать все файлы с расширением
*.log
*.pyc
*.tmp

# Игнорировать директорию (слэш в конце)
node_modules/
__pycache__/
.cache/

# Игнорировать всё внутри директории, но сохранить саму директорию
logs/*
!logs/.gitkeep
```

### Шаблоны (Patterns)

```gitignore
# * — любое количество любых символов (кроме /)
*.txt           # all.txt, readme.txt, но НЕ dir/file.txt

# ** — любое количество директорий
**/*.log        # file.log, dir/file.log, dir/subdir/file.log
**/temp/        # temp/, a/temp/, a/b/temp/

# ? — один любой символ
file?.txt       # file1.txt, fileA.txt, но НЕ file10.txt

# [abc] — один символ из набора
file[123].txt   # file1.txt, file2.txt, file3.txt
file[a-z].txt   # filea.txt, fileb.txt, ..., filez.txt

# [!abc] — один символ НЕ из набора
file[!0-9].txt  # filea.txt, но НЕ file1.txt
```

### Отрицание (Negation)

```gitignore
# Игнорировать все .txt файлы
*.txt

# Но НЕ игнорировать important.txt
!important.txt

# ВАЖНО: порядок имеет значение!
# Правила применяются сверху вниз
```

### Слэши и директории

```gitignore
# Без слэша — игнорирует везде
temp            # temp, dir/temp, dir/subdir/temp

# Со слэшем в начале — только в корне репозитория
/temp           # только ./temp, НЕ dir/temp

# Со слэшем в конце — только директории
temp/           # директория temp, но НЕ файл temp

# Путь с директорией
build/output/   # ./build/output/, НЕ ./other/build/output/
```

## Примеры .gitignore для разных проектов

### Python проект

```gitignore
# Байт-код Python
__pycache__/
*.py[cod]
*$py.class
*.so

# Виртуальное окружение
venv/
.venv/
env/
.env/

# Переменные окружения
.env
.env.local
*.env

# Тестирование
.pytest_cache/
.coverage
htmlcov/
.tox/
.nox/

# IDE
.idea/
.vscode/
*.swp
*.swo

# Дистрибутивы
dist/
build/
*.egg-info/
*.egg

# Jupyter Notebooks
.ipynb_checkpoints/

# mypy
.mypy_cache/
```

### Node.js проект

```gitignore
# Зависимости
node_modules/
jspm_packages/

# Сборка
dist/
build/
.next/
out/

# Переменные окружения
.env
.env.local
.env.*.local

# Логи
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# IDE
.idea/
.vscode/
*.swp

# OS файлы
.DS_Store
Thumbs.db

# Тестирование
coverage/
.nyc_output/
```

### Общий .gitignore

```gitignore
# ========== OS ==========
# macOS
.DS_Store
.AppleDouble
.LSOverride
._*

# Windows
Thumbs.db
ehthumbs.db
Desktop.ini

# Linux
*~
.directory

# ========== IDEs ==========
# JetBrains (PyCharm, WebStorm, etc.)
.idea/
*.iml
*.iws
*.ipr

# VS Code
.vscode/
*.code-workspace

# Vim
*.swp
*.swo
*~
.netrwhist

# Emacs
*~
\#*\#
/.emacs.desktop
/.emacs.desktop.lock

# ========== Secrets ==========
.env
.env.*
!.env.example
*.pem
*.key
secrets/
credentials.json

# ========== Logs ==========
*.log
logs/
```

## Глобальный .gitignore

Настройки, которые применяются ко ВСЕМ вашим репозиториям:

```bash
# Создать глобальный .gitignore
touch ~/.gitignore_global

# Настроить Git использовать его
git config --global core.excludesfile ~/.gitignore_global
```

Содержимое `~/.gitignore_global`:
```gitignore
# OS файлы
.DS_Store
Thumbs.db

# IDE (ваши личные настройки)
.idea/
.vscode/
*.swp

# Временные файлы
*.tmp
*.bak
```

## Локальный .git/info/exclude

Для игнорирования файлов только в вашей локальной копии (не попадает в репозиторий):

```bash
# Редактировать файл
vim .git/info/exclude
```

```gitignore
# Мои личные файлы, которые не нужно коммитить
my_notes.txt
test_local.py
```

## Команды для работы с .gitignore

### Проверка игнорируемых файлов

```bash
# Проверить, игнорируется ли файл
git check-ignore -v path/to/file

# Показать все игнорируемые файлы
git status --ignored

# Показать какое правило игнорирует файл
git check-ignore -v --no-index file.txt
```

### Игнорирование уже отслеживаемых файлов

```bash
# Если файл уже в Git, добавление в .gitignore не поможет!

# Шаг 1: Убрать файл из отслеживания (но сохранить локально)
git rm --cached secret.txt

# Шаг 2: Добавить в .gitignore
echo "secret.txt" >> .gitignore

# Шаг 3: Закоммитить изменения
git add .gitignore
git commit -m "Stop tracking secret.txt"

# Для директории
git rm -r --cached directory_name/
```

### Временное игнорирование изменений

```bash
# Временно игнорировать изменения в файле
git update-index --assume-unchanged config.py

# Вернуть отслеживание
git update-index --no-assume-unchanged config.py

# Посмотреть все файлы с assume-unchanged
git ls-files -v | grep "^[a-z]"
```

## Шаблоны .gitignore

GitHub поддерживает коллекцию шаблонов: https://github.com/github/gitignore

```bash
# Можно скачать готовый шаблон
curl -o .gitignore https://raw.githubusercontent.com/github/gitignore/main/Python.gitignore

# Или использовать gitignore.io
curl -o .gitignore https://www.toptal.com/developers/gitignore/api/python,django,vscode
```

## Best Practices

### 1. Создавайте .gitignore в начале проекта
```bash
# Первым делом при создании репозитория
git init
touch .gitignore
# Заполнить правилами
git add .gitignore
git commit -m "Initial commit with .gitignore"
```

### 2. Используйте шаблоны
```bash
# Не изобретайте велосипед
# Используйте github/gitignore или gitignore.io
```

### 3. Храните пример конфигурации
```gitignore
# Игнорируем реальный конфиг
.env
config/secrets.yaml

# Но храним пример
!.env.example
!config/secrets.yaml.example
```

### 4. Комментируйте правила
```gitignore
# Python bytecode
__pycache__/
*.pyc

# Virtual environments
venv/
.venv/

# IDE-specific (remove if team uses different IDEs)
.idea/
.vscode/
```

### 5. Проверяйте перед коммитом
```bash
# Убедитесь, что секреты не попадут в репозиторий
git status
git diff --staged
```

## Типичные ошибки

### 1. Добавление .gitignore после коммита секретов

```bash
# ПРОБЛЕМА: файл уже в истории Git!
git add .
git commit -m "Initial commit"  # Включая secrets.py

# Добавление в .gitignore НЕ удалит из истории
echo "secrets.py" >> .gitignore

# РЕШЕНИЕ: нужно удалить из истории
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch secrets.py" \
  --prune-empty --tag-name-filter cat -- --all

# Или использовать BFG Repo-Cleaner (проще)
```

### 2. Игнорирование важных файлов

```gitignore
# ПЛОХО — слишком общее правило
*.json

# ХОРОШО — конкретные файлы
package-lock.json
# Но НЕ package.json (он нужен)
```

### 3. Конфликты в команде

```gitignore
# ПЛОХО — если в команде разные IDE
.idea/       # JetBrains
.vscode/     # VS Code

# Лучше — каждый добавляет в глобальный .gitignore
# Или документируйте в README
```

### 4. Забытый слэш для директорий

```gitignore
# ПЛОХО — игнорирует и файл, и директорию "build"
build

# ХОРОШО — явно указываем, что это директория
build/
```

## Резюме

- **`.gitignore`** — файл с правилами игнорирования файлов и директорий
- Синтаксис: `*` (любые символы), `**` (любые директории), `?` (один символ), `!` (отрицание)
- Слэш в конце `/` означает директорию
- Правила применяются сверху вниз, порядок важен
- Используйте готовые шаблоны (github/gitignore, gitignore.io)
- Файлы, уже добавленные в Git, нужно сначала удалить из отслеживания (`git rm --cached`)
- Глобальный .gitignore — для ваших личных настроек (IDE, OS)
- **Никогда не коммитьте секреты!** Даже если потом добавите в .gitignore

---
[prev: 03-committing-changes](./03-committing-changes.md) | [next: 05-viewing-commit-history](./05-viewing-commit-history.md)