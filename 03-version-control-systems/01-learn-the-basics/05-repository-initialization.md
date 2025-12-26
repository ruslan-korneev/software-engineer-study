# Инициализация репозитория Git

## Два способа начать работу

### Способ 1: Создать новый репозиторий (`git init`)
Для нового проекта или существующей папки без Git.

### Способ 2: Клонировать существующий (`git clone`)
Для работы с уже существующим репозиторием.

---

## git init — Создание нового репозитория

### Базовое использование

```bash
# Создать папку проекта
mkdir my-project
cd my-project

# Инициализировать Git репозиторий
git init
```

**Вывод:**
```
Initialized empty Git repository in /path/to/my-project/.git/
```

### Что создаёт git init?

Команда создаёт скрытую папку `.git/`:

```
my-project/
├── .git/                 # Папка Git (репозиторий)
│   ├── HEAD              # Указатель на текущую ветку
│   ├── config            # Локальная конфигурация
│   ├── description       # Описание (для GitWeb)
│   ├── hooks/            # Скрипты-хуки
│   ├── info/             # Дополнительная информация
│   ├── objects/          # Хранилище объектов (коммиты, файлы)
│   └── refs/             # Ссылки на коммиты (ветки, теги)
└── (ваши файлы)
```

### Инициализация с указанием ветки

```bash
# Создать репозиторий с веткой main (вместо master)
git init -b main

# Или
git init --initial-branch=main
```

### Инициализация в существующей папке

```bash
# Уже есть проект с файлами
cd existing-project

# Инициализировать Git
git init

# Добавить файлы и сделать первый коммит
git add .
git commit -m "Initial commit"
```

### Bare репозиторий

**Bare репозиторий** — репозиторий без рабочей директории. Используется на серверах.

```bash
git init --bare my-repo.git
```

**Структура:**
```
my-repo.git/
├── HEAD
├── config
├── objects/
├── refs/
└── ... (без рабочих файлов)
```

---

## git clone — Клонирование репозитория

### Базовое использование

```bash
# Клонировать по HTTPS
git clone https://github.com/user/repo.git

# Клонировать по SSH (рекомендуется)
git clone git@github.com:user/repo.git
```

### Клонирование в другую папку

```bash
# Клонировать в папку my-folder
git clone https://github.com/user/repo.git my-folder

# Клонировать в текущую папку (должна быть пустой)
git clone https://github.com/user/repo.git .
```

### Частичное клонирование

```bash
# Только последний коммит (shallow clone)
git clone --depth 1 https://github.com/user/repo.git

# Последние N коммитов
git clone --depth 10 https://github.com/user/repo.git

# Клонировать только определённую ветку
git clone --branch develop --single-branch https://github.com/user/repo.git
```

**Когда использовать shallow clone:**
- Большой репозиторий (например, Linux kernel)
- CI/CD pipelines (нужен только последний код)
- Быстрое ознакомление с проектом

### Что делает git clone?

```bash
git clone https://github.com/user/repo.git
```

Эквивалентно:
```bash
mkdir repo
cd repo
git init
git remote add origin https://github.com/user/repo.git
git fetch origin
git checkout main
```

---

## Первые шаги после инициализации

### После git init

```bash
# 1. Создать .gitignore
touch .gitignore

# 2. Добавить файлы
git add .

# 3. Первый коммит
git commit -m "Initial commit"

# 4. Добавить удалённый репозиторий
git remote add origin git@github.com:user/repo.git

# 5. Отправить на сервер
git push -u origin main
```

### После git clone

```bash
# Репозиторий уже настроен, можно работать
git status                    # Проверить состояние
git log                       # Посмотреть историю
git branch -a                 # Посмотреть все ветки
```

---

## Файл .gitignore

### Что это?

`.gitignore` — файл со списком паттернов файлов/папок, которые Git должен игнорировать.

### Создание .gitignore

```bash
# Создать файл
touch .gitignore
```

### Примеры паттернов

```gitignore
# Комментарии начинаются с #

# Игнорировать конкретный файл
secrets.txt

# Игнорировать по расширению
*.log
*.tmp
*.pyc

# Игнорировать папку
node_modules/
__pycache__/
.venv/
dist/
build/

# Игнорировать папку в любом месте
**/logs/

# Исключение из игнорирования
*.log
!important.log

# Игнорировать файлы в корне проекта
/config.local.json

# Игнорировать по паттерну
*.py[cod]        # .pyc, .pyo, .pyd
```

### Типичные .gitignore

#### Python
```gitignore
# Byte-compiled
__pycache__/
*.py[cod]
*$py.class

# Virtual environments
.venv/
venv/
ENV/

# Distribution
dist/
build/
*.egg-info/

# IDE
.idea/
.vscode/
*.swp

# Environment
.env
.env.local

# Testing
.pytest_cache/
.coverage
htmlcov/
```

#### Node.js
```gitignore
# Dependencies
node_modules/

# Build
dist/
build/

# Environment
.env
.env.local

# Logs
*.log
npm-debug.log*

# IDE
.idea/
.vscode/

# OS
.DS_Store
Thumbs.db
```

#### Общий шаблон
```gitignore
# IDE
.idea/
.vscode/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Environment
.env
.env.local
.env.*.local

# Logs
*.log

# Secrets
*.pem
*.key
secrets/
```

### Глобальный .gitignore

Для файлов, которые нужно игнорировать во всех проектах:

```bash
# Создать глобальный gitignore
touch ~/.gitignore_global

# Настроить Git
git config --global core.excludesfile ~/.gitignore_global
```

**Содержимое ~/.gitignore_global:**
```gitignore
# OS
.DS_Store
Thumbs.db

# IDE
.idea/
.vscode/
*.swp
```

### Генераторы .gitignore

- https://www.gitignore.io — генератор для разных технологий
- GitHub предлагает шаблоны при создании репозитория

---

## Структура типичного проекта

```
my-project/
├── .git/                # Git репозиторий
├── .gitignore           # Игнорируемые файлы
├── README.md            # Описание проекта
├── LICENSE              # Лицензия
├── src/                 # Исходный код
│   └── main.py
├── tests/               # Тесты
│   └── test_main.py
├── docs/                # Документация
├── requirements.txt     # Зависимости (Python)
└── .env.example         # Пример переменных окружения
```

---

## Проверка состояния репозитория

### git status

```bash
git status
```

**После init (пустой репозиторий):**
```
On branch main

No commits yet

nothing to commit (create/copy files and use "git add" to track)
```

**С неотслеживаемыми файлами:**
```
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        .gitignore
        README.md
        src/

nothing added to commit but untracked files present (use "git add" to track)
```

### Короткий вывод

```bash
git status -s
# или
git status --short
```

**Вывод:**
```
?? .gitignore
?? README.md
?? src/
```

**Значения символов:**
- `??` — неотслеживаемый файл
- `A` — добавлен в staging
- `M` — изменён
- `D` — удалён

---

## Удаление Git из проекта

Если нужно "деинициализировать" Git:

```bash
# Удалить папку .git
rm -rf .git

# Проверить
git status
# fatal: not a git repository
```

**Внимание:** Это удалит ВСЮ историю коммитов!

---

## Типичные ошибки

### 1. Инициализация в неправильной папке

```bash
# Плохо: инициализация в домашней директории
cd ~
git init  # НЕ ДЕЛАЙТЕ ТАК!

# Решение: удалить .git
rm -rf ~/.git
```

### 2. Забыли создать .gitignore

```bash
# Добавили node_modules в репозиторий
git add .
git commit -m "Initial commit"
# 100500 файлов node_modules...

# Решение: добавить .gitignore и удалить из отслеживания
echo "node_modules/" >> .gitignore
git rm -r --cached node_modules/
git commit -m "Remove node_modules from tracking"
```

### 3. Клонирование в непустую папку

```bash
# Ошибка
git clone https://github.com/user/repo.git existing-folder
# fatal: destination path 'existing-folder' already exists

# Решение: указать пустую папку или другое имя
git clone https://github.com/user/repo.git new-folder
```

### 4. Коммит секретов

```bash
# Закоммитили .env с паролями
git add .
git commit -m "Add config"
git push

# УЖЕ ПОЗДНО! Данные в истории Git.
# Решение: сменить все пароли/ключи и использовать .gitignore
```

---

## Best Practices

1. **Всегда создавайте .gitignore первым делом**
2. **Не инициализируйте Git в домашней директории**
3. **Используйте осмысленные имена репозиториев**
4. **Делайте Initial commit минимальным** (README, .gitignore, LICENSE)
5. **Используйте SSH вместо HTTPS для клонирования**
6. **Проверяйте git status перед коммитом**

---

## Резюме

### Создание нового репозитория

```bash
mkdir my-project && cd my-project
git init
echo "node_modules/" > .gitignore
git add .
git commit -m "Initial commit"
git remote add origin git@github.com:user/repo.git
git push -u origin main
```

### Клонирование существующего

```bash
git clone git@github.com:user/repo.git
cd repo
git status
```

### Ключевые команды

| Команда | Описание |
|---------|----------|
| `git init` | Создать новый репозиторий |
| `git clone URL` | Клонировать существующий |
| `git status` | Проверить состояние |
| `git remote add origin URL` | Добавить удалённый репозиторий |

Теперь вы готовы создавать и клонировать репозитории Git!
