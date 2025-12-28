# Создание репозиториев на GitHub

[prev: 02-github-interface](./02-github-interface.md) | [next: 04-private-vs-public](./04-private-vs-public.md)
---

## Введение

Репозиторий (repository или repo) — это хранилище вашего проекта, включающее все файлы, историю изменений и метаданные. GitHub предоставляет несколько способов создания репозиториев, каждый из которых подходит для разных сценариев.

---

## 1. Создание через веб-интерфейс

### Способ 1: Кнопка "+" в навигации

```
1. Нажмите "+" в верхней панели
2. Выберите "New repository"
```

### Способ 2: Из Dashboard

```
1. На главной странице найдите секцию "Repositories"
2. Нажмите зелёную кнопку "New"
```

### Способ 3: Прямая ссылка

```
Перейдите на: github.com/new
```

### Форма создания репозитория

```
┌─────────────────────────────────────────────────────────────────┐
│ Create a new repository                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Owner *                Repository name *                        │
│ ┌─────────────┐ /    ┌──────────────────────────────────┐      │
│ │ username  ▼ │      │ my-awesome-project               │      │
│ └─────────────┘      └──────────────────────────────────┘      │
│                      ✓ my-awesome-project is available          │
│                                                                 │
│ Description (optional)                                          │
│ ┌──────────────────────────────────────────────────────────┐   │
│ │ A brief description of your project                      │   │
│ └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│ ○ Public  — Anyone can see this repository                      │
│ ● Private — You choose who can see and commit                   │
│                                                                 │
│ ☐ Add a README file                                             │
│ ☐ Add .gitignore: None ▼                                        │
│ ☐ Choose a license: None ▼                                      │
│                                                                 │
│                              [Create repository]                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Настройки при создании

### Owner (Владелец)

Выбор между личным аккаунтом и организацией:

```
┌─────────────────┐
│ username        │  ← ваш личный аккаунт
│ org-name        │  ← организация, в которой вы состоите
│ another-org     │
└─────────────────┘
```

**Когда использовать организацию:**
- Командный проект
- Корпоративный код
- Open-source проект с несколькими maintainers

### Repository name (Имя репозитория)

Правила именования:
```
Разрешено:
- Буквы (a-z, A-Z)
- Цифры (0-9)
- Дефис (-), подчёркивание (_), точка (.)

Запрещено:
- Пробелы (используйте дефис)
- Специальные символы
- Начинать с точки
```

**Соглашения по именованию:**

```
Хорошие примеры:
my-project              ← kebab-case (рекомендуется)
my_project              ← snake_case
MyProject               ← PascalCase
backend-api
react-todo-app
python-ml-toolkit

Плохие примеры:
my project              ← пробел
my.project.name         ← слишком много точек
a                       ← неинформативно
untitled                ← неинформативно
test123                 ← неинформативно
```

### Description (Описание)

Необязательно, но рекомендуется:
```
Примеры хороших описаний:
- "REST API для управления задачами на Python/FastAPI"
- "CLI инструмент для автоматизации деплоя"
- "React компоненты для дизайн-системы"
- "Учебный проект: клон Twitter на Node.js"
```

### Public vs Private

```
Public:
+ Виден всем в интернете
+ Бесплатный для всех функций
+ Подходит для open-source, портфолио
- Код доступен всем

Private:
+ Виден только вам и приглашённым
+ Подходит для коммерческих проектов
- Некоторые функции ограничены в бесплатном плане
```

---

## 3. README, .gitignore, LICENSE

### README.md

**Что это:** файл с описанием проекта, отображается на главной странице репозитория.

```
☑ Add a README file

Создаст файл README.md с базовым содержимым:
# my-awesome-project
A brief description of your project
```

**Рекомендация:** всегда добавляйте README. Это "лицо" вашего проекта.

**Структура хорошего README:**
```markdown
# Project Name

Brief description

## Installation
How to install

## Usage
How to use

## Contributing
How to contribute

## License
License info
```

### .gitignore

**Что это:** файл со списком файлов/папок, которые Git должен игнорировать.

```
☑ Add .gitignore: Python ▼

GitHub предлагает шаблоны для популярных языков/фреймворков:
- Python
- Node
- Java
- Go
- Ruby
- и многие другие...
```

**Пример .gitignore для Python:**
```gitignore
# Byte-compiled files
__pycache__/
*.py[cod]

# Virtual environment
venv/
.env

# IDE
.idea/
.vscode/

# OS
.DS_Store
Thumbs.db
```

**Пример .gitignore для Node.js:**
```gitignore
# Dependencies
node_modules/

# Environment
.env
.env.local

# Build
dist/
build/

# Logs
*.log
npm-debug.log*
```

### LICENSE (Лицензия)

**Что это:** юридический документ, определяющий права на использование кода.

```
☑ Choose a license: MIT License ▼

Популярные лицензии:
- MIT License          → очень разрешительная
- Apache License 2.0   → разрешительная + патентная защита
- GNU GPLv3            → copyleft, производные должны быть GPL
- BSD 2-Clause         → минималистичная
- None                 → все права защищены (по умолчанию)
```

**Сравнение лицензий:**

| Лицензия | Коммерческое использование | Модификация | Распространение | Copyleft |
|----------|---------------------------|-------------|-----------------|----------|
| MIT | ✓ | ✓ | ✓ | ✗ |
| Apache 2.0 | ✓ | ✓ | ✓ | ✗ |
| GPLv3 | ✓ | ✓ | ✓ | ✓ |
| BSD 2-Clause | ✓ | ✓ | ✓ | ✗ |

**Рекомендации:**
- Open-source: MIT или Apache 2.0
- Хотите, чтобы производные оставались открытыми: GPL
- Коммерческий проект: можно без лицензии или проприетарную

---

## 4. Клонирование на локальную машину

После создания репозитория на GitHub нужно склонировать его локально.

### Получение URL для клонирования

```
┌─────────────────────────────────────────────────────────────────┐
│ <> Code ▼                                                       │
├─────────────────────────────────────────────────────────────────┤
│ Clone                                                           │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ HTTPS │ SSH │ GitHub CLI                                   │ │
│ ├────────────────────────────────────────────────────────────┤ │
│ │ https://github.com/username/repo.git              [📋]     │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│ Open with GitHub Desktop                                        │
│ Download ZIP                                                    │
└─────────────────────────────────────────────────────────────────┘
```

### Клонирование через HTTPS

```bash
# Клонирование
git clone https://github.com/username/repo.git

# Перейти в папку проекта
cd repo

# Проверить статус
git status
```

**Преимущества HTTPS:**
- Работает везде
- Не требует настройки SSH

**Недостатки:**
- Нужно вводить логин/пароль (или токен) при каждом push
- Или настроить credential helper

### Клонирование через SSH

```bash
# Клонирование
git clone git@github.com:username/repo.git

# Перейти в папку проекта
cd repo
```

**Преимущества SSH:**
- Не нужно вводить пароль после настройки ключа
- Более безопасно

**Требования:**
- Настроенный SSH ключ (см. Profile Setup)

### Клонирование через GitHub CLI

```bash
# Установить GitHub CLI (если не установлен)
brew install gh  # macOS
sudo apt install gh  # Ubuntu

# Авторизоваться
gh auth login

# Клонирование
gh repo clone username/repo

# Или с полным именем
gh repo clone github.com/username/repo
```

### Credential Helper для HTTPS

Чтобы не вводить пароль каждый раз:

```bash
# macOS (Keychain)
git config --global credential.helper osxkeychain

# Windows (Credential Manager)
git config --global credential.helper manager

# Linux (cache на 1 час)
git config --global credential.helper 'cache --timeout=3600'

# Linux (сохранить навсегда) — менее безопасно
git config --global credential.helper store
```

**Важно:** вместо пароля GitHub требует Personal Access Token (PAT):
```
Settings → Developer settings → Personal access tokens → Generate new token

Выбрать scopes:
☑ repo — полный доступ к репозиториям
```

---

## 5. Создание репозитория из существующего проекта

Если у вас уже есть локальный проект:

### Шаг 1: Создать пустой репозиторий на GitHub

```
При создании НЕ добавляйте:
☐ README
☐ .gitignore
☐ License

Иначе возникнут конфликты при push
```

### Шаг 2: Инициализировать Git локально

```bash
cd my-project

# Инициализировать Git
git init

# Добавить все файлы
git add .

# Первый коммит
git commit -m "Initial commit"
```

### Шаг 3: Связать с GitHub

```bash
# Добавить remote
git remote add origin https://github.com/username/repo.git

# Переименовать ветку в main (если нужно)
git branch -M main

# Отправить на GitHub
git push -u origin main
```

### Полный пример

```bash
# Допустим, есть проект:
cd ~/projects/my-app

# Инициализация
git init

# Создать .gitignore
echo "node_modules/" > .gitignore
echo ".env" >> .gitignore

# Добавить файлы
git add .

# Коммит
git commit -m "Initial commit: project setup"

# Связать с GitHub
git remote add origin git@github.com:username/my-app.git

# Push
git push -u origin main
```

---

## 6. Шаблоны репозиториев (Templates)

### Создание шаблона

Можно сделать репозиторий шаблоном:

```
Settings → (scroll down) → Template repository → ☑ Check

После этого на странице репозитория появится кнопка:
[Use this template]
```

### Использование шаблона

```
1. Найти репозиторий-шаблон
2. Нажать "Use this template" → "Create a new repository"
3. Указать имя нового репозитория
4. Создать

Результат: новый репозиторий со всеми файлами шаблона,
но без истории коммитов
```

**Отличие от Fork:**
| Template | Fork |
|----------|------|
| Чистая история | Копия с историей |
| Нет связи с оригиналом | Связан с оригиналом |
| Для создания новых проектов | Для контрибуции |

---

## 7. Импорт из других платформ

GitHub поддерживает импорт из:
- GitLab
- Bitbucket
- SVN
- Mercurial
- TFS

### Процесс импорта

```
github.com/new/import

1. Вставить URL репозитория-источника
2. Указать имя нового репозитория
3. Выбрать Public/Private
4. Begin import

GitHub скопирует код и всю историю коммитов
```

---

## 8. Настройки после создания

### Рекомендуемые действия

```bash
# 1. Клонировать репозиторий
git clone git@github.com:username/repo.git
cd repo

# 2. Настроить локальный Git (если не настроен глобально)
git config user.name "Your Name"
git config user.email "your@email.com"

# 3. Создать ветку для разработки
git checkout -b develop

# 4. Добавить .gitignore (если забыли при создании)
# Скачать с gitignore.io или github/gitignore

# 5. Настроить защиту веток (на GitHub)
# Settings → Branches → Add rule
```

### Branch Protection Rules

```
Settings → Branches → Add branch protection rule

Pattern: main

☑ Require a pull request before merging
  ☑ Require approvals: 1
  ☑ Dismiss stale approvals

☑ Require status checks before merging
  ☑ Require branches to be up to date

☑ Require conversation resolution

☑ Restrict who can push
```

---

## 9. Best Practices

### Именование репозиториев

```
✓ Используйте kebab-case: my-awesome-project
✓ Будьте конкретны: react-todo-app, python-api-client
✓ Добавляйте контекст: company-backend, course-homework
✗ Избегайте: test, untitled, project1, aaaa
```

### Структура проекта

```
my-project/
├── .github/
│   ├── workflows/          # GitHub Actions
│   └── ISSUE_TEMPLATE/     # Шаблоны issues
├── src/                    # Исходный код
├── tests/                  # Тесты
├── docs/                   # Документация
├── .gitignore
├── README.md
├── LICENSE
└── requirements.txt / package.json / go.mod
```

### README должен содержать

```markdown
# Project Name

One-paragraph description.

## Features
- Feature 1
- Feature 2

## Requirements
- Python 3.10+
- PostgreSQL

## Installation
git clone ...
pip install -r requirements.txt

## Usage
python main.py

## Configuration
Copy .env.example to .env and fill in values.

## Contributing
See CONTRIBUTING.md

## License
MIT License
```

---

## Заключение

Создание репозитория — первый шаг к организованной разработке. Ключевые моменты:

1. Выбирайте осмысленное имя
2. Добавляйте README с описанием
3. Настраивайте .gitignore сразу
4. Выбирайте подходящую лицензию для open-source
5. Настройте защиту веток для командной работы

Хорошо организованный репозиторий — основа успешного проекта.

---
[prev: 02-github-interface](./02-github-interface.md) | [next: 04-private-vs-public](./04-private-vs-public.md)